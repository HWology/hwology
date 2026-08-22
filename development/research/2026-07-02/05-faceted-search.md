# Faceted search and filter UI options

*Research date: 2026-07-02, parallel web-research agents. Findings marked `verified-live` were confirmed against the live web that day; `reported` means sourced but not directly fetched; treat access details as perishable.*

## Summary

At your scale (thousands of products, single dev, self-hosted), a dedicated search engine is optional, not required — plain SQLite/Postgres facet counts via GROUP BY are milliseconds-fast at this row count, and the real work is modeling specs as structured fields (ints, numerics, JSON arrays) per category. For the MVP, SQLite as source of truth + Datasette gives you a working faceted browser with zero code today (column facets, JSON-array facets for multi-valued specs like video outputs, __gte/__lte column filters), and ItemsJS (client-side faceted search, Apache-2.0, comfortable to ~100k items) gets you a genuinely Newegg-style custom UI by just exporting the catalog to JSON — no search server at all. For the grown-up version, add Meilisearch (v1.48.3, June 2026) or Typesense (v30.2) behind Algolia's open-source InstantSearch UI library with the official adapters; Typesense has the richest faceting (labeled numeric range buckets, facet_query, reference-field faceting) while Meilisearch is simpler to run and MIT-licensed. Sync from SQL should be a dumb full-reindex script on write (seconds at this size) — the CDC tools like meilisync are reportedly unmaintained. Elasticsearch/OpenSearch is overkill on every axis for this project. No existing open-source project delivers PC-hardware spec filtering out of the box (Part-DB is the closest, but it's electronics-inventory-oriented), so the filter UI layer (InstantSearch or ItemsJS) is the piece worth reusing rather than a whole app.

## Findings

### Meilisearch (v1.48.3, June 29 2026): simplest engine, full facet-count support, MIT license

*Confidence: `verified-live`*

Verified current release v1.48.3 (2026-06-29); v1.48.2 (June 24) patched two auth/privilege-escalation CVEs, so pin a recent version. Facet support verified in docs: `facetDistribution` returns value->count maps per facet, `facetStats` returns min/max for numeric facets (feeds range sliders), a dedicated facet-search endpoint lets users type-ahead within a facet's values, and filter syntax supports numeric comparisons (`price < 100`, ranges via AND). Gotcha: facet counts are ESTIMATES by default; exact counts require `exhaustiveFacetCount: true` (slower, but irrelevant at thousands of docs — turn it on). `maxValuesPerFacet` caps returned values (default 100). Single-node only, ~512MB RAM, single binary or Docker — the easiest of the three engines for a solo self-hoster. Official `instant-meilisearch` adapter plugs into InstantSearch.js.

Sources:

- <https://github.com/meilisearch/meilisearch/releases>
- <https://www.meilisearch.com/docs/learn/filtering_and_sorting/search_with_facet_filters>

### Typesense (v30.2 current): richest faceting of the lightweight engines — labeled numeric range buckets

*Confidence: `verified-live`*

Docs landing page verifies v30.2 is the latest version (v30.0 added JOIN improvements incl. faceting on reference fields, global synonyms, MMR diversification; exact release dates could not be reliably confirmed). Faceting verified in v29 API docs: `facet_by` gives counts; numeric range facets with custom labels — `facet_by=price(economy:[0,100], premium:[100,])` — which is exactly the Newegg bucketed-price/spec pattern and something Meilisearch lacks natively; `facet_query` searches within facet values with typo tolerance; `facet_strategy` (exhaustive/top_values/automatic) controls count accuracy; `filter_by` supports `[min..max]` and comparison operators. Trade-offs vs Meilisearch: entire index lives in RAM (a non-issue at thousands of docs, ~256MB floor), GPLv3 (fine for a personal tool), built-in Raft clustering you won't need. Official `typesense-instantsearch-adapter` is maintained on GitHub.

Sources:

- <https://typesense.org/docs/>
- <https://typesense.org/docs/29.0/api/search.html>
- <https://github.com/typesense/typesense-instantsearch-adapter>

### Elasticsearch/OpenSearch: skip it for this project

*Confidence: `reported`*

1–8GB RAM baseline vs ~256–512MB for Typesense/Meilisearch, JVM ops burden, index-mapping ceremony, and its strengths (distributed scale, log analytics, complex aggregation pipelines) are all irrelevant at thousands of products for one developer. Every 2026 comparison reviewed positions it as the choice only when you need cluster scale or advanced aggregations. If you ever did want it, Searchkit/InstantSearch-compatible layers exist, but there is no scenario in this project where it beats the lighter engines.

Sources:

- <https://ossalt.com/blog/meilisearch-vs-typesense-vs-elasticsearch-search-2026>
- <https://typesense.org/docs/overview/comparison-with-alternatives.html>

### Syncing from SQL source of truth: do dumb full reindexes; CDC tooling is weak

*Confidence: `reported`*

Neither Meilisearch nor Typesense has an official database-CDC pipeline. The community tool meilisync (MySQL/Postgres/Mongo -> Meilisearch via WAL + wal2json) exists and is even referenced in Meilisearch's own guides, but multiple reports (incl. a Meilisearch org discussion) say it is unmaintained and flaky in realtime mode. At your scale this is a non-problem: a script that SELECTs the whole category table, maps rows to JSON docs, and PUTs them to the engine reindexes a few thousand docs in seconds. Trigger it on write, on a cron, or as a CLI step in your ingest pipeline. Treat the search engine as a disposable, rebuildable cache of the SQL database — never a second source of truth.

Sources:

- <https://github.com/orgs/meilisearch/discussions/765>
- <https://www.meilisearch.com/docs/guides/database/meilisync_postgresql>

### Plain Postgres/SQLite is unambiguously fast enough at thousands of rows — no search engine required

*Confidence: `verified-live`*

Facet counts are just `SELECT col, count(*) ... WHERE <current filters> GROUP BY col` per facet (or one pass with lateral joins / a UNION of grouped queries). The known pain points are at MILLIONS of rows: cybertec's pgfaceting (roaring-bitmap extension) exists to count facets over 100M-row tables (155ms over 60M matched rows), and ParadeDB's pg_search added a `pdb.agg()` window function (verified live on their blog) returning results + facet counts in one index pass because 'manual faceting performance degrades dramatically with larger result sets' — their benchmark dataset is 46M rows. At 10^3–10^4 rows with btree indexes on facet columns, naive GROUP BY faceting is low single-digit milliseconds; the same holds for SQLite (Datasette's whole facet UI runs suggested-facet queries under a 50ms budget). What SQL alone doesn't give you: typo-tolerant text search (acceptable: FTS5/tsvector), and a prebuilt facet UI — that's the actual gap a search engine fills via InstantSearch adapters.

Sources:

- <https://www.paradedb.com/blog/faceting>
- <https://github.com/cybertec-postgresql/pgfaceting>
- <https://www.cybertec-postgresql.com/en/faceting-large-result-sets/>

### Datasette: a real zero-code faceted MVP over SQLite, with known UI ceilings

*Confidence: `verified-live`*

Verified from stable docs: add `_facet=COLUMN` to any table URL (or configure in metadata) and you get value counts with click-to-filter; three facet types — plain columns, JSON-array columns (perfect for multi-valued specs like video outputs or M.2 slots without join tables), and auto-detected date facets. Facet size defaults to 30 values (`_facet_size`, `default_facet_size` configurable), suggested facets run under a 50ms/query budget, and indexing facet columns is the documented perf lever. Column filters support gte/lte-style operators in the URL for numeric ranges (no slider widget — it's a data-explorer UI, not a storefront). Version status verified on PyPI: stable is 0.65.2 (Nov 5 2025); the 1.0 alpha series is very active (1.0a34 June 16 2026 added in-browser row insert/edit/delete; 1.0a35 June 23 2026) with a large plugin ecosystem (datasette.io/tools, dogsheep-beta for cross-table faceted search). Verdict: ideal week-one MVP and permanent admin/QA view of your catalog, but you will outgrow its browsing UX for the 'shopping' experience — it can't do dependent facet counts styled like Newegg without heavy plugin/template work.

Sources:

- <https://docs.datasette.io/en/stable/facets.html>
- <https://pypi.org/project/datasette/>
- <https://datasette.io/>

### ItemsJS: client-side faceted search engine — the sleeper pick for this exact scale

*Confidence: `verified-live`*

Verified on GitHub: Apache-2.0, v2.4.3 released Nov 25 2025 (actively maintained), explicitly designed for JSON datasets up to ~100K items, runs in browser or Node. Gives you facet aggregations with per-facet AND/OR logic, counts, pagination, sorting, boolean filter combinations, and pluggable full-text (MiniSearch/Lunr) — i.e., the entire Newegg filter-sidebar data layer with zero servers: export your SQLite catalog to a single JSON file at build time, ship a static page. Numeric range filters are done via custom filter functions rather than declarative buckets — fine at this data size since it just scans. This is the least infrastructure that still produces a real product-filtering UX; pair it with your own sidebar components or a small Vue/React/HTMX frontend. Related: stereobooster/facets is a newer TypeScript alternative in the same niche if ItemsJS feels dated.

Sources:

- <https://github.com/itemsapi/itemsjs>
- <https://github.com/stereobooster/facets>

### Reusable UI layer: InstantSearch + official adapters is the only mature off-the-shelf facet UI

*Confidence: `reported`*

Algolia's InstantSearch (MIT, vanilla JS/React/Vue flavors) provides prebuilt refinement-list, numeric-range, hierarchical-menu, and current-refinements widgets — the whole Newegg sidebar — and both Meilisearch (`instant-meilisearch`) and Typesense (`typesense-instantsearch-adapter`) maintain official adapters so it runs against your self-hosted engine. This is the component to reuse instead of building facet widgets from scratch. Whole-app candidates checked and mostly rejected: Part-DB (open-source parts inventory, SQLite/MySQL/Postgres backends, 'parametric search' filters, pulls specs/datasheets from Octopart/DigiKey/LCSC) is the closest existing self-hosted system and worth a look for its parametric-attribute data model, but it targets electronic-component inventory, not PC-hardware shopping-spec comparison; Binner is similar but narrower. No open-source PCPartPicker equivalent with a spec-filter UX exists (PCPartPicker itself is closed). E-commerce platforms (Saleor, Sylius, etc.) are far too heavy for a single-user catalog.

Sources:

- <https://github.com/typesense/typesense-instantsearch-adapter>
- <https://github.com/meilisearch/instant-meilisearch>
- <https://github.com/Part-DB/Part-DB-server>
- <https://github.com/replaysMike/Binner>

### Recommended MVP stack

*Confidence: `verified-live`*

SQLite as the single source of truth (one table per category with typed spec columns; multi-valued specs as JSON arrays — e.g. form_factor TEXT, pcie_x16_slots INT, lan_speed_gbe REAL, video_outputs JSON, plus a bios_releases child table for firmware history). Layer 1 (day one): Datasette over that file — instant faceted browsing, JSON API for free, add indexes on facet columns. Layer 2 (when you want the Newegg feel): a static page with ItemsJS fed by a build-step JSON export from the same SQLite DB. This stack is zero-daemon (Datasette optional), fully self-hosted, and your example query — mATX/ITX + pcie_slots >= 3 + lan >= 2.5 + video_out present — works in both layers immediately. Critical success factor is spec normalization discipline, not engine choice: every filterable spec must be a typed field, never free text.

Sources:

- <https://datasette.io/>
- <https://github.com/itemsapi/itemsjs>

### Recommended grown-up stack

*Confidence: `verified-live`*

Keep SQL as source of truth (stay on SQLite until multi-writer needs appear; then Postgres). Add Typesense v30.x if you want the best faceting semantics (labeled numeric range buckets in facet_by, facet_query type-ahead, exhaustive counts) or Meilisearch 1.48+ if you value MIT license and the lowest-friction ops — both are a single Docker container. Frontend: InstantSearch with the engine's official adapter, which supplies dependent facet counts, range widgets, and URL-synced state out of the box. Sync: a ~50-line full-reindex script run on ingest (idempotent, rebuildable). Escape hatch if you'd rather run zero extra services: thin API over Postgres doing GROUP BY facet counts + FTS — verified to be well within performance budget at this scale, with ParadeDB pg_search available later if the catalog somehow reaches millions of rows. Datasette stays alive permanently as the admin/debugging view over the source database.

Sources:

- <https://typesense.org/docs/29.0/api/search.html>
- <https://github.com/meilisearch/meilisearch/releases>
- <https://www.paradedb.com/blog/faceting>
