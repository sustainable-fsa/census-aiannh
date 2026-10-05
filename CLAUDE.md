# census-aiannh

Vintage-matched US Census American Indian / Alaska Native / Native Hawaiian
(AIANNH) Area boundaries. One GeoParquet per TIGER/Line vintage as published,
one more clipped to that same year's cartographic waterline, three derived
display formats (TopoJSON, FlatGeobuf, PMTiles) per vintage, a _cb
TopoJSON/FlatGeobuf pair per vintage sourced from Census's own cb 500k AIANNH
product (where published), and eight stable top-level paths: the newest
vintage of everything, plus an ms_simplify'd _simple pair.

**New repo (2026-09-01).** Cloned from `census-counties`, which is the template
for everything here — every deliberate divergence is listed below. Where this
file is silent, `census-counties/CLAUDE.md` is the record: the mask post-mortem,
the ms_explode/ms_dissolve failure, the ROW_GROUP_SIZE measurement, the
sort-order measurement all live there and were not re-derived.

## Why it exists

`usdm-aiannh` (native-resilience org) downloads TIGER AIANNH itself and commits
the boundaries to git under the old GitHub-Pages pattern. This repo takes the
boundary-archive job, the same refactor that carved `census-counties` out of
`usdm-counties`. It lives in the **sustainable-fsa org** (user decision — it
started life at `~/git/native-resilience/census-aiannh` and was moved before the
first commit) and publishes to the sustainable-fsa bucket under prefix
`census-aiannh`.

- **`data/parquet/<year>-aiannh.parquet` is TIGER/Line as published** — the
  legal boundary, unaltered, the analytical truth.
- **`data/clipped/<year>-aiannh.parquet` is for display and is derived, never
  authoritative.**
- **`data/topojson|flatgeobuf|pmtiles/` are derived from the clipped layer and
  are never read back into the pipeline.** That one-way flow is what makes
  TopoJSON quantization acceptable here — census-counties rejected a TopoJSON
  round trip *as an input* because quantization introduces s2-invalidity, and
  that rejection stands. Display artifacts at the end of the line are the
  exception, not a precedent.

Unlike census-counties ("no tiles and no projection here"), the display formats
live in this archive by explicit user request. TopoJSON and PMTiles are WGS84 by
format convention; FlatGeobuf keeps EPSG:4269 like the parquet.

## Layout

```
census-aiannh.R                      the whole pipeline
R/s3-archive.R                       vendored shared helpers (copy of census-counties')
README.Rmd                           renders README.md; Quick Start reads the CDN
.github/workflows/census-aiannh.yaml weekly cron '30 17 * * 4' (2h after census-counties), OIDC
census-aiannh/                       the BagIt bag (gitignored, S3-hosted)
  data/raw/<census zip name>         verbatim: tl_2010_us_aiannh00.zip (2000),
                                     fe_2007_us_aiannh.zip (2007), tl_<yr>_us_aiannh.zip
  data/parquet/<year>-aiannh.parquet
  data/clipped/<year>-aiannh.parquet
  data/topojson/<year>-aiannh.topojson
  data/topojson/<year>-aiannh_cb.topojson        from cb 500k, where published
  data/flatgeobuf/<year>-aiannh.fgb
  data/flatgeobuf/<year>-aiannh_cb.fgb           from cb 500k, where published
  data/pmtiles/<year>-aiannh.pmtiles
  data/quality/geometry_validation.csv
census-aiannh.{parquet,topojson,fgb,pmtiles}   newest vintage, copied (repo root,
                                               gitignored, uploaded beside the bag)
census-aiannh_cb.{topojson,fgb}                newest cb vintage, copied
census-aiannh_simple.{topojson,fgb}            newest clipped vintage, ms_simplify'd
```

## The schema, and its three traps

User decisions, not defaults — do not "simplify" them away:

- **The R/T split is kept.** TIGER splits some entities into reservation (`R`)
  and off-reservation trust land (`T`) features; the archive's unit is the
  component, keyed by `GEOID` (= `AIANNHCE` + R/T suffix). Do NOT collapse
  components by grouping on `AIANNHCE` or `GNIS` — that was offered and
  declined.
- Columns: `GEOID, AIANNHCE, GNIS, Name, NameLSAD, LSAD, COMPTYP, year
  (, mask_year), Area`. Dropped by user choice: `CLASSFP, AIANNHR, MTFCC,
  FUNCSTAT, ALAND, AWATER, INTPT*`.

The traps, all verified by ogrinfo on the actual zips (2026-09-01):

1. **The GEOID selector must stay anchored**:
   `dplyr::matches("^(GEOID(10)?|AIANNHID(00)?)$")`. The 5-char id is
   `AIANNHID` in 2007–2009, `AIANNHID00` in 2000, `GEOID10` in 2010, `GEOID`
   from 2011 — and **2025 adds `GEOIDFQ`**, so `starts_with("GEOID")` grabs two
   columns and select() errors (or worse).
2. **2000 has no GNIS, and 2007's is empty.** `tl_2010_us_aiannh00.zip` carries
   no `AIANNHNS00` field — a named selector matching nothing is silently
   dropped, so the select is followed by a backfill branch that adds
   `GNIS = NA_character_`. And `fe_2007_us_aiannh.zip` HAS the column but every
   value is null (verified 2026-09-01); 2008 is the first populated vintage.
   GNIS being all-NA in 2000 and 2007 is the data, not a bug.
3. **COMPTYP's domain is "as published by Census"** — nothing in the pipeline
   assumes it is exactly {R, T}; the README hedges likewise.

Feature counts run 734 (2000) to 867 (2025) per vintage — 16,468
component-vintages, 920 distinct GEOIDs, an 867 MB bag — ~4% of
census-counties' data volume. Everything is faster and smaller here; none of
the county repo's scale workarounds are load-bearing.

## The mask: simpler than census-counties, deliberately

The waterline is still the union of that year's `cb` 500k **counties**
(counties tile the nation; their union is the national landmass), built by the
same repair → `st_union()` → `fill_holes()` path. But AIANNH features carry no
STATEFP and none lie in the four island territories `cb` 2010 misses, so the
composite per-state mask and `mask_year_by_state` plumbing were **deleted, not
ported**. One `mask_year` per vintage, as a scalar.

`mask_years` fallbacks: 2000, 2007, 2008, 2009, 2011 → 2010; 2012 → 2013.

**The `identical(tl$id, clipped$id)` guard is what stands in for the composite
machinery** — mapshaper's `-clean` drops null geometries silently, so a feature
outside the mask would vanish without it. Keep it verbatim.

## Rules carried over that are load-bearing

- **`census_repair()` leaves valid features untouched, byte for byte** — or the
  validity log is meaningless.
- **Keep the area guard** (accept a repair only if s2-valid AND ≥ 99.9% of the
  area it was handed). Clark County WA arrived with inverted winding once.
- **Do NOT clean with `ms_explode %>% ms_dissolve`.** Mapshaper is used once,
  for the clip: `-clip <mask> remove-slivers -clean rewind`, one process.
- **Membership in the S3 listing decides reprocessing**, not local files. Here
  the freshness gate requires all four per-vintage artifacts (clipped parquet
  + 3 display formats), plus the _cb pair where a cb file exists, plus the
  eight top-level latest-vintage files — so a format added later backfills;
  each artifact is also gated individually.
- **GeoJSON for mapshaper (and tippecanoe) goes through
  `geojsonsf::sf_geojson()`**, never `sf::write_sf()` — GDAL's GeoJSON driver
  rewrites NAD83 to CRS84. For the display formats the transform to 4326 is
  explicit and intended.
- `rmapshaper::ms_clip()` silently drops `remove_slivers` on the `sys = TRUE`
  path, which is why the clip is assembled by hand.

## The quality log's `geoid` column

`log_change()`'s id argument/column is `geoid` here (was `fips`). It carries
the AIANNH `GEOID` on feature rows, a **county FIPS on `mask_repair` rows**
(the mask is built from cb counties), and `"<mask>"` on the mask row.

## Display formats

- **TopoJSON**: `mapshaper-xl <geojson> -rename-layers aiannh -o <file>
  format=topojson quantization=1e6 fix-geometry` — the dd17/dd22 option set.
- **FlatGeobuf**: `sf::write_sf(driver = "FlatGeobuf")` straight from the
  clipped sf object, EPSG:4269, GDAL spatial index by default.
- **PMTiles**: `tippecanoe -o <file> -l aiannh -zg
  --coalesce-densest-as-needed --extend-zooms-if-still-dropping
  --detect-shared-borders --force <geojson>`. The layer is always `aiannh` so
  one map style serves every vintage. tippecanoe comes from conda-forge via
  `setup-geospatial@v1` in CI (`shell: bash -l {0}` puts it on PATH); locally
  `brew install tippecanoe`.
- `Area` is cast `as.numeric()` on the way to GeoJSON: properties cannot carry
  a units class.
- PMTiles serve over plain HTTPS range requests — no CloudFront change needed.
  Browser use from third-party origins depends on the bucket/distribution CORS
  policy, which is org infrastructure, not this repo.

## The _cb layer is Census's product; the _simple layer is ours

Two generalized layers, deliberately both (user decision, 2026-09-01 — a
cb-only "_simple" was tried and rejected the same day because the cb product
turned out to be entity-level):

**`<year>-aiannh_cb.{topojson,fgb}`** come from
`cb_<year>_us_aiannh_500k.zip` (2010: `gz_2010_us_250_00_500k.zip`, summary
level 250) — Census's own 1:500k cartographic generalization.

- **The unit is the ENTITY.** cb AIANNH files have a 4-char `GEOID`
  (= `AIANNHCE`), no COMPTYP, R/T dissolved by Census: 692–704 features vs
  tl's 734–867. That does not soften the "keep the R/T split" rule for
  everything tl-derived — this layer is different because its *source* is.
  Joins to the component-level files go through `AIANNHCE`.
- **Availability is 2010, 2013, 2014+** (the cb-counties pattern). No
  fallback for missing years: borrowing a neighboring year's waterline is
  defensible, borrowing its boundaries as THE boundaries is not.
- Schema drift (ogrinfo-verified 2026-09-01): 2010 is `GEO_ID`/`AIANHH`, no
  GNIS, textual LSAD; `AIANNHNS` from 2013; `NAMELSAD` from 2021; 2025 adds
  `GEOIDFQ` — selectors stay anchored, same trap as the tl reader.
- Display-only: no repair, no quality-log rows. `Area` is computed from the
  generalized geometry — an approximation, unlike the parquet layers.

**`census-aiannh_simple.{topojson,fgb}`** (top-level only, no per-vintage
form) are the newest clipped vintage through
`ms_simplify(keep = 0.05, keep_shapes = TRUE, sys = TRUE)`, keeping the
component-level schema. Two user decisions (2026-09-01) diverge from the
dd17/dd22 precedent, both because keep_shapes protects FEATURES, not parts —
at dd's keep = 0.008 all 867 features "survived" while 72% of polygon parts
vanished and 196 components lost over half their area (trust-land
checkerboards worst: Crow T went 102 parts → 1 at 6% area):

- **keep = 0.05**, not 0.008.
- **Explode-protect-regroup**: st_cast to one row per parcel before
  ms_simplify (each parcel then its own feature, so none can vanish),
  group_by + summarise(do_union = FALSE) back to components after. sf-side
  on purpose — NOT mapshaper's ms_explode/ms_dissolve, which stays banned
  (its snapping broke census-counties' mask). do_union = FALSE because a
  spherical union would gate on s2 validity, which simplified geometry does
  not promise. A stopifnot guards that no GEOID is lost.

Their territory filter and shift_geometry() are also not ported. `Area`
stays the pre-simplification clipped area.

## Top-level latest-vintage artifacts (the stack is gone)

The all-vintage stack was **replaced by user decision (2026-09-01)**:
`census-aiannh.{parquet,topojson,fgb,pmtiles}` are byte-identical `file.copy`s
of the newest vintage's artifacts, `census-aiannh_cb.{topojson,fgb}` of the
newest cb vintage's, and `census-aiannh_simple.{topojson,fgb}` are built
fresh from the newest clipped vintage. Consequences worth keeping straight:

- `latest_vintage` and `latest_cb_vintage` are pinned **before** VINTAGES
  narrowing, so a backfill run cannot repoint the top-level files; they are
  pinned **separately** because a year's tl and cb need not land together.
- The eight files are rebuilt unconditionally on any run that gets past the
  freshness gate, and they are part of `required_keys`, so adding one later
  backfills. They are the mutable half of the archive: uploaded via `s3_put`
  outside the append-only bag push, always invalidated.
- `ensure_clipped()` was generalized to `ensure_archived()` (any bag-relative
  path; `ensure_clipped` survives as an alias) so the copies work on a fresh
  CI runner where the newest vintage was never built locally.
- The stack's `arrange(GEOID, year)` / `ROW_GROUP_SIZE=2000` lore now lives
  only in census-counties; nothing here depends on it anymore.

## `ensure_clipped()`, and a census-counties bug

Any stage that reads a clipped parquet calls `ensure_clipped()`, which pulls it
from S3 when the local file is missing — on a CI runner, an already-archived
vintage was never built locally. **census-counties' CLAUDE.md documents this
fix but its shipped `census-counties.R` does not contain it** (the stack at
~line 925 reads local paths blind); it will fail there the week a new vintage
lands on a fresh runner. Back-port it.

## Publishing

`Rscript census-aiannh.R` builds and publishes; `PUBLISH=0` builds locally;
`VINTAGES=2024,2025` narrows (it narrows `cb_aiannh_urls` too, by
intersection — a cb-less vintage in VINTAGES is fine). Append-only
(`s3_push(delete = FALSE)`), then `s3_verify`, eight `s3_put`s for the
top-level artifacts (TopoJSONs as `application/json` or CloudFront will not
compress them — measured in fsa-counties-dd22), `generate_tree_flat()`,
`s3_write_manifest`, `cf_invalidate` (the twelve mutable paths: eight
top-level artifacts plus the two manifests, the quality log and
`_manifest.txt` — per-vintage files are immutable once written),
`cf_wait_manifest`, then
`rmarkdown::render("README.Rmd")` — after publish, because Quick Start reads
the CDN. AWS profile `mco` locally; OIDC in CI.

Vintages discovered by `url_exists()` out to next year; 20 tl resolve as of
2026-09-01 (2000, 2007–2025). URL shapes: 2000 and 2010 live inside
`TIGER2010/AIANNH/<2000|2010>/`, 2007 is `TIGER2007FE/fe_2007_us_aiannh.zip`,
2008–2009 sit at the TIGER root, 2011+ are `TIGER<yr>/AIANNH/`. The cb files
for the _cb layer: 14 resolve (2010, 2013–2025) — `GENZ2010/gz_…`,
`GENZ2013/cb_…` (no `shp/`), `GENZ<yr>/shp/cb_…` from 2014.

## Open threads

1. ~~First publish and `gh repo create sustainable-fsa/census-aiannh`.~~ Done
   2026-09-01: the repo exists and S3 holds the full archive (verified against
   the live listing — all 20 vintages × 5 artifact families + quality log).
   Still owner steps: the Zenodo GitHub integration and first release; add the
   DOI badge and citation DOI, plus `doi:`/`date-released:` in CITATION.cff,
   once minted. Note the first CI run after the 2026-09-01 refactor overwrites
   the published `census-aiannh.parquet` — the 15 MB all-vintage stack becomes
   the ~7 MB latest-vintage copy — so anything reading that URL for stacked
   vintages breaks then.
2. Back-port `ensure_clipped()` to census-counties (see above).
3. ~~Repoint `usdm-aiannh` to read its boundaries from this archive, the way
   `usdm-counties` reads from census-counties.~~ Done 2026-10-04: usdm-aiannh
   reads `data/parquet/<year>-aiannh.parquet` (vintage V for USDM year V+1),
   keyed by GEOID component, in the published NAD83 so this archive's `Area`
   is the exact denominator. Anything here that changes `data/parquet/`
   paths, columns, CRS or `Area` breaks it.
4. ~~README Extent/size numbers should be refreshed from the first full
   build.~~ Done 2026-09-01: full local build (`PUBLISH=0`) gave 16,468 rows /
   920 GEOIDs / 20 vintages, 0 clip damage, quality log 20 `mask_fill_holes` +
   20 `raw_repair` and nothing else.
