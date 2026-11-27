# Native bond support in the fork – plan

Living document, last updated 2026-11-27.

The fork gets bonds as an instrument type of their own: a real nominal, prices quoted in percent, and accrued interest, coupons and redemptions as transactions of their own. Files written by the fork no longer need to open in the original Portfolio Performance; an export produces a compatible copy whenever one is needed.

## Goal

- **Real nominal, real percent price:** EUR 10,000 nominal at 98.50 %, not 100 "shares" at 98.50.
- **Clean and dirty value kept apart:** holding value = clean value + accrued interest, day by day. Price gains and interest income are reported separately.
- **Transaction types of their own** for accrued interest, coupons and redemptions.
- **A cash flow engine at the core:** yield to maturity and to call, duration, expected interest over the next 12 months, a maturity ladder, and rate scenarios (±100 basis points on the whole portfolio).
- **Automatic taxonomies:** remaining term buckets, coupon type, seniority.
- **Special cases:** floating rate notes, inflation-linked bonds, callable bonds, amortizing bonds, credit events (default with recovery value).

## Domain model

### Instrument

- `Security` gets a real `InstrumentType` (today only a string, internally `DEBT`).
- Plus optional `BondTerms`: coupon type, rate, frequency, interest accrual start, maturity, day count convention, ex-coupon days, denomination, call and amortization schedule; reference rate and spread for floating rate notes, index and base index for inflation-linked bonds.

### Valuation

- Today about 30 core files and 19 UI files compute shares × price directly via `Values.Share` / `Values.Quote`.
- These call sites move to a `PricingModel` on the security with a single method `long value(long shares, long price)`. No date, nothing bond-specific: this half is offered upstream as well (see [Upstream strategy](#upstream-strategy)).
- `PerUnitPricing`: today's formula, the default for everything except bonds.
- `PercentOfNominalPricing` (fork only): the clean value, nominal × price / 100 × index ratio. It reads the bond data (`BondTerms`, index fixings) from the security itself; nothing bond-specific passes through the interface.
- `value()` needs no valuation date:
    - **Pool factor:** booked as a partial redemption. The quantity held is always the outstanding nominal, and the repayment shows up as a cash flow.
    - **Index ratio** (inflation-linked bonds only): taken at the date of the price, because the price applies to that date. `SecurityPrice` carries its date already.
    - **Accrued interest:** a separate amount, not part of `value()`. It is added only where a holding is valued at a date. `PortfolioSnapshot.create(portfolio, converter, date)` knows that date already and creates every `SecurityPosition`; the fork passes the date on, and `calculateValue()` returns clean value + accrued interest at that date. The statement of assets, performance and charts all run through snapshots, so only a handful of files change. Buys and sells carry their own date and their own accrued interest amount.

  ```
  Market value (dirty) = nominal × price / 100 × index ratio + accrued interest(t)
  Accrued interest(t)  = nominal × index ratio × coupon × day count fraction(t)
  ```

  The nominal is already the outstanding nominal after partial redemptions, so the pool factor does not appear in the formulas.
- Accrued interest is shown as a figure of its own, as is common in bond trading, for example as a column in the statement of assets.

### Transactions

- `Unit.Type.ACCRUED_INTEREST` as a unit on buys, sells and inbound deliveries.
- `AccountTransaction.Type.COUPON` with a security reference and tax units, separate from `DIVIDENDS`. Today only `DIVIDENDS` can reference a security, `INTEREST` cannot.
- `PortfolioTransaction.Type.REDEMPTION` with an account counterpart, like `BuySellEntry`. Covers final maturity, calls and partial redemptions, including pool factor changes.

### Events (`SecurityEvent`)

- `RATE_FIXING` (floating rate notes), `INDEX_FIXING` (inflation-linked bonds), `POOL_FACTOR` (source for proposed partial-redemption bookings), `CALL_NOTICE`, `CREDIT_EVENT`.

### Performance

- A new category for accrued interest; `EARNINGS` is split into dividends and coupons.
- Capital gains measure only the change in clean value, so the pull to par becomes visible.
- The cost basis is clean: accrued interest paid on a buy is not acquisition cost, accrued interest received on a sale is income.
- On the coupon date the dirty value drops by exactly the coupon; the payment is neutral to the valuation.
- Optional second view: amortized cost using the effective interest method.

## File format and parallel use

Fork files no longer open in the original Portfolio Performance; an export keeps a compatible copy available at any time.

- **A version counter of its own,** `forkVersion`, in a separate field. Upstream's `CURRENT_VERSION` (currently 70) keeps growing, so upstream migrations are merged as they are instead of being renumbered.
- **Number ranges of its own:** new protobuf fields from 1000, new enum values from 100, so they never collide with later upstream additions.
- **Fork files are marked** so that the original Portfolio Performance rejects them with a clear message. Unmarked, it would crash halfway through reading on an unknown enum value (`ProtobufWriter.java:592` throws `UnsupportedOperationException`).
- **Export for the original Portfolio Performance:** one click writes a file in today's workaround format: nominal / 100 as shares, accrued interest inside the amount, coupons as dividends, redemptions as sales at 100. Keeping both apps in use means running this export, not entering anything twice. If the fork fails, an up-to-date file for the original remains.
- **Upgrade import:** a wizard takes over existing files. It detects bonds (type `DEBT`, prices around 100, percentages in the name), attaches terms and separates the accrued interest, either from the note (some importers write "Stückzins … 25,60 EUR" into it) or as the remainder of amount − clean value − fees − taxes.
- Export and import test each other: fork → export → upgrade import must give back the same file, as far as the workaround format carries the information.

## Upstream strategy

The fork's diff will be large, so upstream's importers stay untouched and every merge runs through the same tests. In the three months up to 2026-11-17 upstream had 374 commits; 181 changed files were in `datatransfer`, 55 in `online`, only 15 in `snapshot` and 7 in `model`.

1. **Importers stay unchanged.** A step after extraction converts the workaround output into the new model. `datatransfer` carries about half of upstream's churn; staying out of it avoids most conflicts.
2. **Refactor first, then features.** `PricingModel.value(shares, price)` arrives as a pure refactoring with no change in behaviour; all upstream tests stay green. It is offered upstream as a PR that computes the value of a position in one place.
    - **Accepted:** the roughly 50 call sites are identical in fork and upstream and stop causing conflicts. New upstream code calling `value()` is right for bonds in the fork, except for the accrued interest, which is small and caught by the golden tests.
    - **Rejected (plan B):** the refactoring stays in the fork, and the arithmetic guard below catches new upstream code.
3. **Arithmetic guard in CI:** a test that fails the build on direct shares × price arithmetic (`Values.Share` / `Values.Quote` factor math) outside `PricingModel`. Without it, a new upstream view written with the old formula would show bonds wrong (off by a factor of 100 with real nominal semantics) without a merge conflict or a compiler error.
4. **Switch inventory:** older Java `switch` statements over enums are not checked for exhaustiveness, so new values such as `COUPON` or `REDEMPTION` would fall into `default` without a warning. Roughly 41 files handle dividends, 50 sales and 44 fee units. The relevant switches become exhaustive switch expressions or get a test; on every merge an agent looks for newly added switches.
5. **Golden tests** (see [Testing](#testing)) run on every merge, so upstream logic that drifts silently (everything compiles, the numbers are wrong) turns the build red.
6. **Patch inventory** in `docs/fork-patches.md`: what changed where and why, with the covering test. It is the context the merge agent works from.
7. **Weekly merge run:** fetch upstream, merge, run upstream tests, fork tests and golden tests; an agent resolves conflicts and the result arrives as a PR for review. Merging promptly instead of in batches means a large upstream refactoring (for example of the performance calculation) arrives in small pieces.

## Testing

There is no real depot statement to compare against, so all reference values are synthetic or taken from documents that are already in the repository.

- **Reference portfolio** with one case each: German federal bond, corporate bond, zero coupon bond, floating rate note, inflation-linked bond, US Treasury in USD, a buy across a coupon date, a buy in the ex-coupon period (negative accrued interest), a partial redemption, final maturity.
- **Expected values from independent sources:**
    - QuantLib (Python, offline) for accrued interest, yields and duration, stored as JSON test vectors.
    - The sanitized bank statements already in `name.abuchen.portfolio.tests`: 80 samples from 15 banks show accrued interest (Baader Bank, comdirect, Consorsbank, DAB, DADAT Bankenhaus, DekaBank, Deutsche Bank, DKB, easybank, ING, onvista, quirin bank, Trade Republic, UniCredit, Weberbank). They list nominal, price, accrued interest and total as the bank calculated them.
    - The issuers' final terms for coupon dates and day count conventions.
- **Golden tests:** end to end on the reference portfolio, with fixed expected figures for the statement of assets, performance per period and accrued interest.
- **Round trip:** export for the original and upgrade import must reproduce the fork file.

## Phases

| Phase | Name | Content |
| --- | --- | --- |
| 0 | Foundation | `upstream` remote, CI, reference portfolio and golden tests, QuantLib test vectors |
| 1 | Engine | Terms, day count conventions (ACT/ACT ICMA, ACT/360, ACT/365F, 30/360, 30E/360), payment schedules with holiday calendars, accrued interest, yield, duration, convexity, fixings; pure functions in a package of their own |
| 2 | Refactoring | `PricingModel.value(shares, price)` at every valuation site, arithmetic guard, switch inventory, no change in behaviour; upstream PR |
| **E1** | Decision | Are all upstream tests green, and is a first upstream merge onto the refactored code workable? |
| 3 | Model | `InstrumentType`, `BondTerms`, nominal semantics, new transaction types, units and events, persistence, `forkVersion`, migration |
| 4 | Valuation | Valuation date in `SecurityPosition`, accrued interest, performance categories, cost basis and trades, IRR, maturity, foreign currency |
| **E2** | Decision | Does the fork reproduce the synthetic reference portfolio and the bank statement samples to the cent? |
| 5 | Bridges | Export for the original, upgrade import, importer post-processing, CSV columns |
| 6 | UI | Security editor with bond data, dialogs with nominal, price and accrued interest, new columns, translations into all languages |
| 7 | Analytics | Payment calendar, maturity ladder, portfolio yield and duration, rate scenarios, amortized cost, automatic taxonomies |
| 8 | Master data | ESMA FIRDS import (maturity, fixed rate or reference rate with spread, denomination, seniority), optionally extraction from the final terms |
| 9 | Special cases | Floating rate notes (€STR is free from the ECB, Euribor is not), inflation-linked bonds (HICP through the existing Eurostat feed), callable bonds, amortizing bonds, credit events |

The usable core is phases 0–5 plus the essentials of phase 6.

## Infrastructure

- **Snapshot builds:** `.github/workflows/fork-snapshot.yml` builds every push to `master` with all tests and publishes self-contained archives (bundled Java runtime) for Windows, macOS and Linux as the pre-release "snapshot", with SHA-256 checksums and build provenance (`gh attestation verify`).
- **No update to the original:** fork builds have automatic updates turned off. Otherwise the next official release would replace the fork's bundles and fork files would no longer open.
- **Planned:** the weekly merge run, an update site of the fork's own, optional code signing for Windows and macOS.

## Open questions

The plan keeps evolving; these are the decisions still open.

**Modelling**

- Do coupons get their own performance category, or are they part of earnings?
- Where is the original nominal shown next to the outstanding nominal after partial redemptions?
- Inflation-linked bonds take the index ratio at the price date. Is that approximation good enough with stale prices, or does the valuation date have to reach `value()` after all?
- Zero coupon bonds: valued through the price only (pull to par), with no accrued interest. Does that hold for every view?
- Ex-coupon period: is the coupon entitlement booked at the record date or the payment date?
- Matured bond without a redemption booking: book the redemption at 100 % automatically, or only warn?
- Foreign currency bonds: is accrued interest converted at the valuation date's rate, consistent with the existing currency handling?
- Is the German tax treatment of accrued interest (negative capital income) in scope?

**Master data**

- Which day count conventions and holiday calendars are needed for the first step?
- How well does ESMA FIRDS cover bonds outside the EU, such as US Treasuries, and what is the fallback?
- Should terms be extracted from the final terms with an LLM API, and where would its API key live?

**File format**

- Does the fork keep writing XML, or only the binary format?
- How exactly are fork files marked so that the original rejects them cleanly?

**Upstream**

- Is the `PricingModel` PR offered before the fork's phase 2, or after the refactoring has proven itself in the fork? One PR or several slices?
- How does the arithmetic guard recognise shares × price: a pattern on `Values.Share` / `Values.Quote`, or static analysis?

**Testing**

- Which of the 80 bank statement samples become reference cases, and how are expected values derived where a bank shows only totals?

## First step

A spike on the branch `bonds/spike` shows whether the approach holds before the large refactorings start.

- [ ] Phase 0: `upstream` remote, CI, reference portfolio with golden tests
- [ ] Minimal engine: fixed coupon, ACT/ACT, accrued interest, yield to maturity
- [ ] `PricingModel.value(shares, price)` in `SecurityPosition` only, plus the valuation date from `PortfolioSnapshot`
- [ ] One bond end to end: buy with accrued interest, coupon, redemption, with a golden test against QuantLib values
