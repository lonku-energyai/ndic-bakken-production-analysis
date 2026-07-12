# Cleaning Log — NDIC Bakken 2025

A record of the data-quality issues encountered in the raw NDIC monthly production files and how each was resolved. Raw public data is rarely analysis-ready; this log documents the distance between "downloaded" and "trustworthy."

---

## Issue 1 — API well numbers arrived as scientific notation

**Symptom:** the 14-digit API well identifier (`33053039010000`) displayed as `3.3053E+13` after import.

**Cause:** Power BI auto-typed the column as a number; 14-digit integers exceed clean display precision and identifiers should never be numeric anyway (they are never summed, and leading zeros matter in other datasets).

**Fix:** re-typed `API_WELLNO` and `FileNo` as **Text** immediately after import, before any other transformation.

**Rule adopted:** *identifiers are text, always, first.*

---

## Issue 2 — Missing values encoded as the literal text "NULL"

**Symptom:** volume columns (oil, water, gas, days) could not be typed as numbers; the custom quality-flag formula later failed with `Expression.Error: We cannot apply operator < to types Number and Text`.

**Cause:** the source files encode missing values as the four-character **string** `"NULL"`, not as empty cells. A text `"NULL"` is not a real null — null checks pass right over it.

**Fix (three ordered steps):**
1. Convert all numeric columns to Decimal Number in a single type step - the `"NULL"` strings become explicit `Error` values.
2. Immediately apply **Replace Errors → null** on the same columns — converting failed casts into true nulls.
3. Only then run the quality-flag formula, written null-safe (`if [col] = null then "Missing data" else ...`).

**Rule adopted:** *types first → error traps second → formulas last.* Power Query steps execute top-to-bottom; a formula can only trust columns prepared above it. (Discovered the hard way: an early version had the type-change step *after* the formula step, which silently broke ~2,000 rows.)

---

## Issue 3 — Naive flare-rate ranking was technically correct and completely misleading

**Symptom:** the first "Top 10 operators by flare rate" chart showed ten unknown names, all at ~100%.

**Cause:** micro-operators with one or two wells and no gas-gathering connection flare essentially all of their (trivial) gas volume. Ranked purely by rate, they bury every operator of consequence. A second trap: Power BI's Top-N filter computes independently of other visual filters and intersects the results — combining "Top 10 by flare rate" with "gas > 1M Mcf" returned an empty set, because the top-10-by-rate list contained no one above the threshold.

**Fix:** dropped Top N; applied a volume threshold (**Total Gas > 1,000,000 Mcf**) and sorted descending by flare rate. Result: a complete ranking of significant operators, from KODA Resources (~45%) down to majors near zero. The threshold is disclosed in the chart title.

**Rule adopted:** *a ranking without a denominator check is a trap; disclose filtering choices in the visual itself.*

---

## Issue 4 — Confidential ("tight hole") wells

**Symptom:** ~0.9% of records (≈2,000 of 261,683) carry no reported volumes.

**Cause:** North Dakota grants new wells a confidentiality period during which production details are withheld.

**Fix:** flagged as `"Missing data"` via the QualityFlag column rather than dropped or zero-filled — unreported is not the same as zero, and zero-filling would distort per-well averages.

**Rule adopted:** *null ≠ zero. Label gaps; don't paper over them.*

---

## Residual quality profile

| Category | Records | Share |
|---|---|---|
| OK | ~259,700 | 99.1% |
| Missing data (confidential wells) | ~2,000 | ~0.8% |
| Volume w/o producing days | small | <0.1% |
| Days w/o volume | small | <0.1% |

Data Quality Score (share of OK records): **99.1%** — computed as a DAX measure and displayed on the report's Data Quality page.
