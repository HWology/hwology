# Schema and storage architecture for sparse product specs

*Research date: 2026-07-02, parallel web-research agents. Findings marked `verified-live` were confirmed against the live web that day; `reported` means sourced but not directly fetched; treat access details as perishable.*

> **Note:** an adversarial fact-check pass ran after this research — see [06-claim-verification.md](06-claim-verification.md). Two claims were corrected: ASRock is now Incapsula-blocked from datacenter IPs (was reported open), and pgloader does **not** auto-convert TEXT→JSONB (store JSON as plain TEXT in SQLite, add an explicit CAST rule).

## Summary

The current consensus is clear: classic EAV is a legacy pattern with real join-explosion and typing pain (Magento is the canonical cautionary tale), and the modern default for sparse product specs is a hybrid — a normal relational core with typed columns/child tables for the facets you actually filter on, plus a JSONB (or SQLite JSON) column for the long tail, indexed via GIN and/or generated columns. Every serious commerce platform converged on "typed attribute definitions + flexible storage": Saleor keeps a typed attribute-definition layer in Postgres, Shopify metafields require typed definitions before filtering works, and Medusa's JSONB `metadata` is explicitly acknowledged as awkward to filter — the lesson is that an attribute registry (name, datatype, unit, allowed values) matters more than the physical storage choice. For Ivan's killer query ("3+ x16 slots"), JSON arrays are the wrong tool — containment answers "has at least one matching slot" but counting requires un-indexable array expansion, so PCIe/M.2 slots should be child tables (one row per slot) queried with GROUP BY/HAVING, optionally denormalized into summary count columns. Vocabulary normalization should steal from ETIM: canonical numeric values in base units plus a synonym/controller-part-number mapping table, keeping the raw source string for provenance; board revisions should be first-class rows (product → revision) since Gigabyte-style revs genuinely change specs. SQLite with JSON functions + indexed virtual generated columns is a completely sane start for a single-user tool at this scale (thousands of rows — performance is a non-issue, modeling correctness is the issue), and pgloader gives a one-command migration path to Postgres if it's ever needed.

## Findings

### Consensus: EAV loses to JSONB for dynamic attributes in Postgres — by a lot

*Confidence: `verified-live`*

The widely-cited coussej benchmark (10M entities, 5 properties each): EAV needed 6.43GB across 3 tables vs 2.08GB for one JSONB table (~3x); with a GIN index and the @> containment operator, JSONB multi-attribute filters ran in 0.153ms (~15,000x faster than the EAV equivalent). The structural problem with EAV: every additional filter condition adds another self-join or EXISTS, everything is stored as text (no type safety, no numeric range filters without casts), and the planner can't estimate row counts. Cybertec's well-known 'EAV — don't do it!' post and 2026 write-ups reach the same verdict: JSONB is the default; EAV only if you truly need a runtime attribute-definition system with row-granular locking. Caveats that survive: JSONB updates rewrite the whole document (heavier row locks under concurrent writes — irrelevant for a single-user catalog), and JSONB indexing is not automatic — you must add the right indexes.

Sources:

- <https://coussej.github.io/2016/01/14/Replacing-EAV-with-JSONB-in-PostgreSQL/>
- <https://www.cybertec-postgresql.com/en/entity-attribute-value-eav-design-in-postgresql-dont-do-it/>
- <https://docs.bswen.com/blog/2026-04-24-jsonb-vs-eav-postgresql/>

### Magento's EAV history is the cautionary tale: EAV forces you to build a second read model

*Confidence: `reported`*

Magento's pure-EAV catalog meant a single product load could join 11+ tables. Their fix — 'flat table' indexers that denormalize EAV into wide tables — created its own failure mode: indexing delays, data-sync bugs, extension conflicts; flat catalog was deprecated in Magento 2.3.x, and 2.4 made Elasticsearch/OpenSearch the mandatory filtering engine. The transferable lesson for Ivan: if your write model can't answer filter queries directly, you end up maintaining a derived read model with sync machinery. A hybrid schema that is directly queryable avoids that entire class of infrastructure.

Sources:

- <https://www.mgt-commerce.com/tutorial/magento-2-flat-catalog/>
- <https://www.divante.com/blog/efficient-magento-code-database-flat-eav-part-2>
- <https://yegorshytikov.medium.com/magento-indexers-how-it-works-a2ded38dd971>

### What modern commerce platforms actually do: typed attribute definitions on top of flexible storage

*Confidence: `verified-live`*

Saleor (Postgres/Django): a structured attribute system — Attribute/AttributeValue tables with data types, validation, and per-ProductType assignment (M2M) — i.e. a disciplined, typed EAV for filterable attributes, plus a plain JSONB `metadata` column for untyped key-value extras. Medusa: relational core (products/options/variants) plus a JSONB `metadata` column; community threads confirm filtering on metadata requires hand-rolled JSONB queryBuilder work and is a known pain point. Shopify: metafields are typed key-value pairs, but filtering only works after you create a metafield *definition* with a type and enable the filterable capability; metaobjects map tables→definitions, foreign keys→typed reference fields. Convergent lesson: regardless of physical storage, you need an attribute registry — name, datatype, unit, allowed values, filterable flag — and untyped grab-bags rot ('metafields introduced without a clear definition later produce conflicting analytics').

Sources:

- <https://docs.saleor.io/docs/3.x/developer/attributes>
- <https://shopify.dev/docs/apps/build/metaobjects/data-modeling-with-metafields-and-metaobjects>
- <https://help.shopify.com/en/manual/custom-data/metafields/filtering-products>
- <https://github.com/medusajs/medusa/discussions/4261>

### The hybrid pattern and how to index it: GIN for containment, generated columns for equality/range facets

*Confidence: `verified-live`*

Pattern: typed columns for hot filterable facets (form_factor, socket, chipset, lan_speed_mbps) + one `specs` JSONB for the long tail. Indexing: (a) GIN with jsonb_path_ops for containment (@>) queries across arbitrary keys; (b) for known hot keys, STORED generated columns extracting typed scalars with plain btree indexes — a May 2026 benchmark (PG 18.2, 50k rows) measured generated columns at ~0.11ms vs 0.19ms GIN for equality and 0.698ms vs 6.6ms for range+equality, because typed scalars give the planner real statistics while GIN treats JSONB as opaque and only supports @>, not = or <. Crunchy Data's rule of thumb: if a key appears in most rows and you filter on it, promote it to a column. PG18 note: VIRTUAL is now the default for generated columns (compute-on-read, no disk); use explicit STORED for anything you want indexed. Limits of the hybrid: no FKs/constraints inside JSONB, whole-document rewrite on update, and facet *counts* (Newegg's '(123) per value') are the genuinely hard part at scale — though at a personal catalog's size (10^3–10^4 rows) even seq scans are instant, so choose this architecture for modeling correctness, not speed.

Sources:

- <https://richyen.com/postgres/2026/05/11/generated_columns_jsonb.html>
- <https://www.crunchydata.com/blog/indexing-jsonb-in-postgres>
- <https://www.postgresql.org/about/news/postgresql-18-released-3142/>
- <https://www.paradedb.com/blog/faceting>

### PCIe/M.2 slots must be child tables, not JSON arrays — counting is the dealbreaker

*Confidence: `reported`*

JSONB containment handles 'has at least one x16 Gen5 slot' (specs @> '{"pcie_slots":[{"size":"x16","gen":5}]}') and is GIN-indexable. But '3 or more x16 slots' cannot be expressed as containment: it requires jsonb_array_elements() lateral expansion (or jsonb_path_query with a count), which expands every row's array and gets no index help. With a child table — pcie_slot(board_id, physical_size, electrical_lanes, gen, open_ended, shared_with) and m2_slot(board_id, key, lengths, gen, lanes, interface_modes) — the query is a plain GROUP BY board_id HAVING count(*) FILTER (WHERE physical_size='x16') >= 3, fully btree-indexed, and you also get CHECK constraints/enums on slot fields plus a natural place for real-world messiness (electrical vs physical lanes, lane-sharing/bifurcation notes, chipset-vs-CPU attachment). Slots are genuine entities with 4-8 typed fields each, few per board — exactly the case where relational wins. Pragmatic addition: denormalize hot counts (pcie_x16_count, m2_count) as generated/maintained summary columns on the board row so faceted UI queries stay one-table.

Sources:

- <https://hevodata.com/learn/query-jsonb-array-of-objects/>
- <https://oneuptime.com/blog/post/2026-01-25-postgresql-json-array-elements/view>
- <https://sqlite.org/json1.html>

### Vocabulary/unit normalization: steal ETIM's design — canonical typed features + synonym mapping, keep the raw string

*Confidence: `reported`*

ETIM (the dominant classification standard for electrical/technical products) is exactly this problem solved at industry scale: per-class feature lists with four datatypes (Alphanumeric with predefined value lists, Logical, Numeric with standardized units, Range), so 'Rated Voltage' vs 'Max Operating Voltage' collapses to one canonical feature. Ivan's version: (1) attribute dictionary with canonical unit — store lan_speed_mbps=2500, never the string '2.5G LAN'; (2) a synonyms/aliases table mapping source strings ('2.5GbE', '2.5G LAN', '2.5 Gigabit') → canonical attribute+value, grown incrementally as scrapers hit new phrasings; (3) controllers as a reference table (part_no 'RTL8125BG'/'I226-V' → vendor, speed class, known quirks) with board rows FK-ing to it — this is higher-fidelity than the marketing string and enables queries like 'Intel NIC only'; (4) always keep the raw scraped string alongside the normalized value for provenance/re-normalization. Icecat's per-category taxonomy (standardized specs per category ID) is the same idea and a possible data source. BMEcat/ETIM themselves are overkill to adopt wholesale for one person, but the structure is the proven template.

Sources:

- <https://www.atropim.com/en/blog/etim>
- <https://en.wikipedia.org/wiki/ETIM_(standard)>
- <https://icecat.com/product-taxonomy/>

### Board revisions: model as first-class child rows with spec overrides, not as attributes

*Confidence: `uncertain`*

Gigabyte-style revisions (rev 1.0 vs 1.1) can change LAN controllers, VRMs, even M.2 layouts, and each revision has its own BIOS branch — so revision must be an entity, not a text field. Recommended shape: product (the family, e.g. 'B550I AORUS PRO AX') → board_revision (rev label, dates, its own spec rows/overrides) with specs resolved as revision-level value if present, else product-level default; BIOS releases (version, date, changelog, AGESA/microcode, download URL) FK to board_revision, since firmware history — Ivan's differentiator — is inherently per-revision. Icecat's data model similarly carries 'product variants' as distinct linked records. This finding is design synthesis grounded in how vendor sites structure rev-specific spec/BIOS pages; not verified against a reference implementation (as of 2026-07-02).

Sources:

- <https://iceclog.com/open-catalog-interface-oci-open-icecat-xml-and-full-icecat-xml-repositories/>

### SQLite (JSON + generated columns) is a sane v1 for a single-user tool — verified mechanics

*Confidence: `verified-live`*

JSON functions are built into SQLite since 3.38 (Feb 2022; JSON1 extension before that), and generated columns since 3.31 (Jan 2020). The exact hybrid pattern ports over: store the long-tail spec document in a TEXT JSON column, add VIRTUAL generated columns via json_extract for hot facets, and index them (verified from sqlite.org: both VIRTUAL and STORED generated columns are indexable — VIRTUAL ones become expression indexes; VIRTUAL columns can be added later via ALTER TABLE, STORED cannot). json_each() handles array queries (slot child tables are still the better model, and they're plain tables anyway). Two SQLite-specific recommendations: declare tables STRICT to suppress type affinity (SQLite happily stores 'abc' in an INTEGER column otherwise — the main source of migration bugs later), and avoid SQLite-only SQL in views so the DDL stays portable. Known limits vs Postgres: no GIN/containment operator (arbitrary-key JSON queries scan), single-writer concurrency (irrelevant here), weaker ALTER TABLE. At catalog scale (thousands of boards) none of these bind.

Sources:

- <https://sqlite.org/gencol.html>
- <https://sqlite.org/json1.html>
- <https://www.dbpro.app/blog/sqlite-json-virtual-columns-indexing>

### Migration path SQLite → Postgres: pgloader, one command, with two caveats

*Confidence: `reported`*

pgloader migrates schema + data + indexes in a single command (pgloader sqlite:///file.db pgsql://...), does standard type mapping (INTEGER→BIGINT, TEXT→VARCHAR), supports custom CAST rules, and notably auto-converts TEXT columns to JSONB when content is valid JSON — which means the SQLite 'JSON in TEXT' pattern lands directly on JSONB. Caveats: (1) SQLite type affinity means dirty data (strings in numeric columns) will fail Postgres's strictness at migration time — STRICT tables from day one prevent this; (2) generated columns and expression indexes won't transfer meaningfully — plan to re-declare facet columns/GIN indexes as Postgres-native after the data lands (a small, mechanical script). Alternative worth noting: if v1 stays single-user forever, there may simply never be a reason to migrate; the schema design (attribute registry, child tables for slots, revision entities) is the durable asset, and it is engine-agnostic.

Sources:

- <https://pgloader.readthedocs.io/en/latest/ref/sqlite.html>
- <https://render.com/articles/how-to-migrate-from-sqlite-to-postgresql>

### Recommended concrete schema for Ivan (synthesis of all of the above)

*Confidence: `uncertain`*

Core tables: category; product (brand, model, category FK); board_revision (product FK, rev label — specs hang here); attribute_def registry (key, display name, datatype, canonical unit, enum values, filterable flag, per-category applicability); spec values split three ways: (1) hot scalar facets as real typed columns on board_revision or a per-category summary table (form_factor, socket, chipset, lan_speed_mbps, has_video_out, m2_count, pcie_x16_count), (2) long-tail scalars in one specs JSON(B) column validated against attribute_def, (3) multi-valued structures as child tables: pcie_slot, m2_slot, port (rear I/O), memory_config. Reference tables: controller (NIC/audio/USB4 part numbers → capabilities), cpu_socket, chipset. Provenance: source table (URL, scrape date, raw payload) FK'd from every spec value or at minimum per revision; bios_release table per revision. This starts in SQLite STRICT tables with virtual generated columns for any facet you later want out of JSON, and maps 1:1 onto the Postgres hybrid pattern. It generalizes to GPUs/cases/PSUs by adding categories + attribute_def rows + new child tables only where a category has genuinely structured multi-valued specs (e.g. PSU connectors, case drive bays).
