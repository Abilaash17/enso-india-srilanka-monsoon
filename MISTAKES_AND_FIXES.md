# Project Log: Mistakes Made and How They Were Fixed

This document tracks the errors, wrong assumptions, and dead ends encountered while
building the ENSO–rainfall analysis for India and Sri Lanka, and how each was
diagnosed and resolved. Kept as an honest record of the actual research process,
not just the final clean result.

---

## 1. Data source failures (early stage)

**Mistake:** Assumed `downloads.psl.noaa.gov` and other NOAA data-hosting subdomains
would be reachable from a Colab/notebook environment the same way a browser reaches
them.

**What happened:** Repeated `ConnectionError` / `No route to host` / timeout errors
across several NOAA subdomains (`origin.cpc.ncep.noaa.gov`, `downloads.psl.noaa.gov`).
Confirmed via `wget` from multiple angles that some subdomains were reachable
(`www.noaa.gov`, `www.cpc.ncep.noaa.gov`) while others were not — a genuine
network/firewall restriction on the compute environment, not a code bug.

**Fix:** Switched to data sources actually reachable from the notebook environment:
NOAA's ERDDAP server (`coastwatch.pfeg.noaa.gov`) for SST, and eventually `pooch`-based
direct downloads from UC Santa Barbara's CHIRPS server for precipitation, which worked
reliably where the PSL mirror did not.

**Lesson:** Don't assume all subdomains of the same organization share the same
network accessibility. Test each host independently before building a pipeline on it.

---

## 2. ERDDAP request size limits

**Mistake:** Requested 30 years of daily gridded SST data for the full Pacific basin
in a single ERDDAP query.

**What happened:** `HTTP Error 413` (payload too large).

**Fix:** Restructured the download to request only the specific months actually
needed (El Niño event months), rather than the full continuous 30-year daily record —
both fixing the error and reducing download volume substantially.

**Lesson:** Gridded climate data requests should be scoped to exactly what's needed
(time range × spatial extent), not "download everything, filter later."

---

## 3. NetCDF calendar and dimension-naming mismatches

**Mistake:** Assumed all NetCDF climate files use standard `time`/`lat`/`lon`
dimension names and a standard Gregorian calendar.

**What happened:** `ValueError: unable to decode time units 'months since
1960-01-01' with calendar '360'` — the IRI/LDEO mirror used a synthetic 360-day
calendar and single-letter axis names (`T`, `X`, `Y`) instead of `time`/`lon`/`lat`.

**Fix:** Loaded with `decode_times=False`, inspected `sst.dims` directly rather than
assuming, renamed axes explicitly (`.rename({'T':'time','X':'lon','Y':'lat'})`), and
manually rebuilt the time coordinate from the known start date and record length.

**Lesson:** Always inspect a new NetCDF file's actual structure
(`print(ds.dims)`, `print(ds.coords)`) before writing processing code against it —
different providers use different conventions even for the "same" variable.

---

## 4. Missing CRS on rioxarray clipping operations

**Mistake:** Attempted to spatially clip a NetCDF-derived DataArray without first
explicitly setting its coordinate reference system.

**What happened:** `MissingCRS: CRS not found. Please set the CRS with
'rio.write_crs()'`.

**Fix:** Added `.rio.set_spatial_dims(x_dim=..., y_dim=...)` and
`.rio.write_crs("EPSG:4326")` before any `.rio.clip()` call.

**Lesson:** `rioxarray` needs spatial metadata set explicitly on data that didn't
originate from a GeoTIFF/raster source with embedded CRS info — this must be done
once per dataset, early, not just before the first clip operation.

---

## 5. Zone definitions using rectangular slices instead of true polygon clips

**Mistake:** Early zone-averaging code used `.sel(latitude=slice(...),
longitude=slice(...))` — simple rectangular bounding-box slicing — labeled as if it
represented "India" or "Sri Lanka," when it actually just cut a rectangle that could
include ocean, neighboring countries, or exclude parts of the intended zone.

**What happened:** Not an error, but a silent correctness issue: results looked
plausible but weren't actually restricted to real country boundaries.

**Fix:** Switched to proper polygon-based clipping (`gpd.overlay()` with real Natural
Earth country boundaries, then `.rio.clip()` using those polygons) so each zone
genuinely respects the national boundary rather than an approximate rectangle.

**Lesson:** A rectangular lat/lon slice is not the same as "this country" — always
verify spatial subsetting against an actual boundary, and sanity-check with a map plot
before trusting the numbers that come out of it.

---

## 6. Combining India and Sri Lanka into single "wet"/"dry" zones

**Mistake:** Initially merged India's and Sri Lanka's wet-zone rainfall into one
combined "wet zone" average (and same for dry), on the assumption they shared a
similar climate regime.

**What happened:** The combined-country correlation was much weaker than expected.
This wasn't caught until directly comparing an India-only test against the combined
version and noticing a large discrepancy.

**Fix:** Split every zone so India and Sri Lanka are always clipped and averaged
**independently** — four zones total (India Wet, India Dry, Sri Lanka Wet, Sri Lanka
Dry), never merged across the border.

**Lesson:** Averaging together two regions with genuinely different underlying
relationships can dilute or completely hide a real signal in either one. When in
doubt, keep sub-populations separate and test whether combining them is actually
justified — don't assume it by default.

---

## 7. Misclassifying Galle as a dry-zone city

**Mistake:** Briefly reclassified Galle (Sri Lanka) from the wet zone into the dry
zone based on a mistaken assumption, without checking it against actual climatological
classification.

**What happened:** Caught and corrected within the same session — Sri Lanka's
Department of Meteorology and standard climate references classify Galle as
southwest **wet zone** (it sits on the monsoon-facing coast alongside Colombo and
Ratnapura).

**Fix:** Reverted the change; kept Galle in the wet-zone bounding box.

**Lesson:** Don't reclassify a location based on a hunch — check it against an
authoritative source (in this case, standard Sri Lankan climatological zoning) before
changing a spatial definition that affects every downstream result.

---

## 8. Notebook cell-ordering bugs after repeated manual edits

**Mistake:** After many rounds of pasting new cells, deleting old ones, and copying
code from the chat into the notebook out of sequence, several cells ended up calling
functions or using variables that were only defined in *later* cells (e.g., an NDVI
correlation cell appeared before the cell that downloaded and built the underlying
NDVI dataset; a plotting cell used `f_oneway` before it was imported).

**What happened:** A string of `NameError`s (`ndvi_files`, `ndvi_masked`,
`f_oneway`, `slope_wet`, `wet_merged`, etc.) that looked like different bugs but were
almost all the same root cause: cell position no longer matched logical dependency
order.

**Fix:** Diagnosed each one by tracing which cell actually defined the missing name
and comparing its position to where it was used. Eventually did a full rebuild of the
notebook from scratch in the correct dependency order rather than continuing to patch
cells one at a time.

**Lesson:** In a notebook that's been edited many times across many sessions, cell
*position* and cell *execution order* can silently diverge. When errors keep
recurring in different forms, it's often faster to rebuild cleanly in dependency order
than to keep patching individual symptoms.

---

## 9. Duplicate/leftover cells from earlier notebook versions

**Mistake:** When the analysis moved from a 2-zone (combined India+Sri Lanka) design
to a 4-zone (separate country) design, several old cells referencing `wet_clip`,
`dry_clip`, `wet_merged`, `dry_merged` (the 2-zone variable names) were left in the
notebook alongside the new 4-zone cells (`india_wet_clip`, `srilanka_wet_clip`, etc.).

**What happened:** Running these stale cells threw `NameError`s for variables that
sounded like they should exist but were actually superseded names from an earlier
design.

**Fix:** Searched the full notebook for every reference to the old 2-zone variable
names and deleted the corresponding stale cells entirely, rather than trying to patch
them to use the new names.

**Lesson:** When a core design changes (e.g., 2 zones → 4 zones), old cells built on
the previous design should be deleted, not left "just in case" — they become
confusing, silent failure points later.

---

## 10. A zone-definition cell was accidentally deleted entirely

**Mistake:** During a cleanup pass removing old 2-zone leftover cells, the cell that
defined the *replacement* 4-zone bounding boxes and clips (`india_wet_bbox`,
`srilanka_wet_clip`, etc.) was also accidentally deleted — it looked similar enough to
the things being intentionally removed.

**What happened:** Every downstream cell that depended on `india_wet_clip` /
`srilanka_dry_clip` etc. failed with `NameError`, even though those variables were
never meant to be removed.

**Fix:** Rebuilt the missing cell from scratch and reinserted it at the correct point
in the notebook (after region setup, before precipitation clipping).

**Lesson:** When deleting cells that "look like" leftovers, double-check each one
individually against what it actually defines — visual similarity between old and new
code (e.g., both mention `wet`/`dry`/`clip`) can lead to deleting the wrong one.

---

## 11. Colab session resets silently invalidating all downloaded data

**Mistake:** Assumed variables and downloaded files would persist between work
sessions, and didn't save intermediate results to disk.

**What happened:** Multiple Colab disconnects/restarts wiped all in-memory variables
and the CHIRPS/NDVI files cached in temporary storage, forcing a full ~30+ minute
re-download each time, discovered only after hitting `NameError` on variables that
had clearly worked minutes before.

**Fix:** Built a checkpoint system: save the small, already-processed zone-averaged
monthly time series (not the large raw gridded files) to CSV immediately after the
expensive download/processing steps, and download those CSVs locally. A dedicated
"reload" cell rebuilds all downstream analysis variables from the checkpoint CSVs in
seconds instead of re-running the full pipeline.

**Lesson:** In any environment with an ephemeral/resettable runtime, checkpoint
expensive intermediate results to durable storage (disk/download) as soon as they're
computed — don't wait until something breaks to realize nothing was saved.

---

## 12. Assuming a stale/wrong ENSO index without checking

**Mistake:** Continued pulling from NOAA's classic `oni.ascii.txt` file to check
"current" ENSO conditions without verifying whether that file was still being updated.

**What happened:** The file's most recent entry was frozen at NDJ 2025, which
initially looked like a fetch bug. Investigation revealed NOAA had officially
transitioned primary ENSO monitoring to a new index (RONI) in February 2026, and the
legacy ONI file simply stopped receiving updates around that time.

**Fix:** Diagnosed by explicitly checking `df['YR'].max()` and comparing against
known current-date context, then searching for and confirming the index transition.
For current-event reporting specifically, switched to reading NOAA's live ENSO
Diagnostic Discussion text directly, while keeping the historical 1996–2025 analysis
on the original (valid, complete) classic ONI record.

**Lesson:** When a "live" data feed stops updating, don't assume it's a bug in your
fetch code — check whether the underlying data product itself has changed or been
deprecated, especially for authoritative government/scientific data sources.

---

## 13. Confusing a region name with a data value

**Mistake:** Referred to "ONI is 3.4," conflating the *name* of the Niño 3.4 region
(the standard Pacific monitoring box) with an anomaly *value* of +3.4°C.

**What happened:** This was actually clarified productively — checking real data
showed the current confirmed ONI was much lower (+0.98°C three-month average, +2.3°C
raw weekly), while +3.4°C was NOAA's *forecast* value for the Niño 3.4 monthly mean in
October, not a value already observed.

**Fix:** Explicitly distinguished, in both the code and the report, between (a) the
official smoothed ONI, (b) the raw weekly Niño 3.4 index, and (c) NOAA's forecast
values for future months — using each correctly rather than treating them as
interchangeable.

**Lesson:** ENSO indices have several related-but-different variants (raw weekly
index, 3-month smoothed ONI, region-name shorthand, forecast vs. observed values).
Confusing them can lead to using the wrong number, or misreading a forecast as an
already-realized observation.

---

## 14. Testing only one calendar season (JJAS) for both countries

**Mistake:** Applied India's conventional JJAS (June–September) monsoon definition
uniformly to Sri Lanka as well, based on the assumption that "the regional monsoon
season" was the same for both countries.

**What happened:** This produced a seemingly clean but misleading headline finding —
"India is ENSO-sensitive, Sri Lanka is not" — that was reported and wired into an
entire draft of the paper (report text, abstract, conclusion) before being questioned.

**Fix:** On request, re-ran the correlation analysis at full monthly resolution
(all 12 months, all 4 zones) instead of a single assumed season. This revealed Sri
Lanka *does* have strong, statistically significant ENSO sensitivity — just
concentrated in October (wet) and February–April (dry), tied to its own Maha monsoon
cycle rather than India's JJAS window. The entire abstract, results, discussion, and
conclusion were rewritten around this corrected, more complete finding.

**Lesson:** This was the single most consequential mistake in the whole project.
Applying one region's seasonal convention to a climatically different region can
produce a confident-looking but wrong conclusion ("no relationship") when the real
issue is simply "wrong test window." When a negative/null result seems surprising
given known physical context, testing at finer temporal resolution before accepting
the null result is worth the extra effort.

---

## 15. Naive linear extrapolation beyond the observed data range

**Mistake:** Applied a regression fit on historically observed ONI values (max ~2.3
for the relevant month) directly to a forecast ONI of +3.4°C, without checking whether
that input fell within the range the regression was actually fit on.

**What happened:** Produced an implausible predicted rainfall anomaly (+11.8 mm/day)
for Sri Lanka's October wet-zone projection — a number driven by extrapolating a
linear fit well past the data it was built from, rather than a credible estimate.

**Fix:** Explicitly checked the observed ONI range for the relevant regression,
capped the prediction input at the historically observed maximum for cases where the
forecast value exceeded it, and reported both the naive extrapolation and the capped,
more defensible estimate side by side with clear caveats.

**Lesson:** A regression line has no way of "knowing" a relationship might not stay
linear beyond the range it was trained on. Before using a fitted model to predict at a
new input value, always check whether that value falls inside or outside the range of
data the model actually saw.

---

## 16. Multiple-comparisons risk in the 48-test monthly sweep

**Mistake (avoided, but worth recording):** Initially treated every "statistically
significant" result from the 12-month × 4-zone sweep (48 total tests) as equally
trustworthy.

**Fix:** Explicitly noted that with 48 tests at p < 0.05, roughly 2–3 "significant"
results would be expected by chance alone even if no real relationship existed
anywhere — and used this to distinguish the very strong, highly unlikely-to-be-chance
findings (e.g., p = 0.0005, p = 0.001) from weaker single-star results (p ≈ 0.02–0.05)
that deserve more caution, rather than presenting all significant results with equal
confidence.

**Lesson:** Running many similar statistical tests increases the chance of false
positives by pure chance. When reporting results from a large batch of tests, say so
explicitly, and weight confidence by how far below the significance threshold each
result actually falls, not just whether it crosses it.

---

## Summary

Most of the serious issues in this project fell into three categories: (1) **network
and data-format assumptions** that didn't hold across different servers/providers
(Sections 1–4), (2) **notebook state management** problems from iterating quickly
across many sessions (Sections 8–11), and (3) **methodological assumptions**
that quietly shaped the conclusion before being questioned (Sections 6, 14, 15) — the
last category being the most consequential, since it affected not just whether code
ran, but whether the actual scientific conclusion was correct.
