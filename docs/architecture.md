# Architecture

*Status: design phase, no code yet. Last updated 2026-07-02. Evidence for every claim here is
in [docs/research/2026-07-02/](research/2026-07-02/README.md).*

## What this is

A public, spec-level hardware catalog with faceted filtering retail sites can't do
("motherboard, mATX/ITX, 3+ PCIe slots, onboard 2.5GbE, onboard video"), plus data retailers
never carry: firmware/BIOS release history, per-slot electrical lane widths, lane-sharing
rules, and provenance behind every fact. Categories: motherboards first, then wifi routers,
switches, and similar gear. Owner-operated, published openly.

## Core principles

1. **Facts are observations, not truths.** Everything is ingested as an append-only event with
   provenance; the catalog is a projection that resolves conflicts by trust rank:
   `measured > correction > manual-PDF > vendor-page > aggregator`.
2. **Never delete what a vendor deletes.** Pulled BIOS releases get flagged `pulled`, not
   removed — a pulled BIOS is high-value signal. (Vendors silently remove releases; history
   not captured today is gone tomorrow.)
3. **Provenance is user-reachable.** Any published fact must resolve to its evidence chain:
   source URL + retrieved-at + sha256 + archive reference. Public users get URL/hash/Wayback
   link; the private archive holds the actual bytes.
4. **Public facts, private bytes.** Published data is CC BY-SA. Scraped raw pages, manual
   PDFs, and firmware binaries live in the private tier and are referenced publicly by
   hash/URL only (the OpenWrt-ToH legal model; see research doc 10).

## Layers

```
                    ┌─────────────────────────────────────────────┐
 sources ──────────▶│ EVENT LOG (append-only JSONL, git-versioned)│
 (scrapers,         │ SnapshotCaptured / SpecObserved /           │
  imports,          │ BiosReleaseObserved / BiosReleasePulled /   │
  hand-edited TOML, │ CorrectionAsserted / MeasurementRecorded    │
  measurements)     └──────────────┬──────────────────────────────┘
                                   │ replay/project (F# build step)
              ┌────────────────────┼───────────────────────┐
              ▼                    ▼                       ▼
   published dataset        SQLite read model       in-memory F# model
   (JSON per device,        (facet columns, slot    (Fun.Blazor GUI queries,
    CC BY-SA, git)           child tables; QA via    incl. slot-requirement
                             Datasette/DuckDB)       matching)

   blobs: Garage (S3) — sha256-keyed buckets: private archive (WARCs/WACZ, PDFs,
   firmware) + public bucket (published exports, redistributable captures)
   witness: Wayback SPN2 push per scraped URL → public citation link
```

- **Canonical layer**: F# domain types are the schema. Machine-sourced records serialize to
  JSON (one file per device); hand-authored records (corrections, measurements, notes) are
  TOML. Both validate against JSON Schema generated from the F# types
  (FSharp.Data.JsonSchema; tombi validates the TOML side in editors/CI).
- **Projections are disposable.** SQLite, the published dataset, and the in-memory model are
  all rebuildable from the event log. Upstream syncs (BuildCores, toh.json) never conflict
  with local corrections — they're just events at different trust ranks.
- **Event discipline**: event types are versioned (`SpecObserved.v1`) and forever; replay of
  five-year-old events must keep working.

## Domain model (sketch)

- `product` → `board_revision` (revisions are first-class: Gigabyte revs differ in real
  hardware; BIOS files are per-revision — flashing across revisions can brick).
- `slot` child rows, not JSON arrays (counting is the dealbreaker for JSON):
  `pcie_slot(rev_id, label, physical_size, electrical_lanes, gen, attached_to CPU|chipset,
  sharing_group_id)`, `m2_slot(...)`, `port(...)`. `label` = silkscreen name (M2_1, PCIEX16_2)
  — "which M.2 is the x2 one" must be answerable.
- `sharing_group(id, rule)` — lanes contended between members. Slot queries are requirement
  lists ("2× ≥x16-phys/≥x8-elec/≥Gen4 AND 2× ≥x1/≥Gen3, simultaneously") evaluated as a small
  matching problem in F# — trivially fast at catalog scale, unwritable in SQL.
- `identity` alias table per product: gtin_ean, gtin_upc, vendor_sku, dmi_name, buildcores_id,
  icecat_id, marketing_name+region. Every cross-source join depends on this (research doc 07,
  gap 3).
- `bios_release(rev_id, version, date, changelog, agesa/me_version, sha256, download_url,
  status: live|pulled)`.
- Attribute registry for the long tail: key, datatype, canonical unit, allowed values,
  filterable flag. Store canonical values (`lan_speed_mbps = 2500`), keep the raw source
  string alongside; controller reference table (RTL8125BG, I226-V → vendor, class).
- Conditional specs are separate attributes (`max_mem_speed_native` vs `_oc`), date/BIOS-
  qualified where they changed over time.

## Provenance schema

Per fact: `{source_url, retrieved_at (ISO-8601 UTC), capture_sha256, blob_ref (garage
bucket/key or wacz+page_id), wayback_urim (nullable), tool+version}`.
Published subset: `source_url, retrieved_at, wayback_urim, sha256`. UI principle: click any
spec value → evidence chain.

## Storage & tooling roles

| Thing | Role |
|---|---|
| git repo | canon: event log JSONL, device files, schemas, docs |
| SQLite | rebuildable read model; STRICT tables; facet columns + slot child tables |
| Garage (S3) | sha256-keyed blob buckets, private (archive) + public (exports). WACZ replays over HTTP range requests from S3 |
| DuckDB | analysis/ingest tool: query JSON/JSONL/Parquet directly (`read_json_auto`), attach SQLite, publish Parquet exports to Garage for remote querying; `duckdb -ui` for local exploration. Not the operational store |
| ~~Datasette~~ | skipped (decision 2026-07-02) — DuckDB's local UI covers exploration |
| browsertrix-crawler | archival capture (WARC/WACZ) of scraped pages |
| single-file-cli v2 | human-readable snapshot via CDP attach to the scraper's browser session |
| Wayback SPN2 | fire-and-forget public witness copy per scraped URL (`if_not_archived_within=30d`) |

## Stack pins (verified 2026-07-02 — see research docs 08, 11, 12)

- .NET 11 previews (P5 current; GA 2026-11-10). F# everywhere; GUI = owner's Fun.Blazor fork.
- Serialization: **.NET 11 built-in STJ F# DU support** (on by default, reflection-only;
  fieldless case → string, fields → `{"$type":...}`; single-case DUs NOT unwrapped — keep
  wrapper DUs out of file-facing types). Options unwrap as before. Pin the format with
  round-trip tests; it freezes at first publish.
- SQLite: Microsoft.Data.Sqlite 10.x + explicit SQLitePCLRaw.bundle_e_sqlite3 ≥ 3.0.3
  (2.1.11 native lib is deprecated with a high-sev vuln). WAL; writes serialized; async is
  fake in SQLite — fine.
- Data access: Dapper.FSharp + raw Dapper for facet SQL. Skip Marten / LiteDB / EF Core.
- TOML: Tomlyn 2.9 (TOML 1.1-only; needs `[<CLIMutable>]`, arrays not F# lists, small
  converters per option type). Keep files 1.0-clean syntax for cross-tool compat. tombi for
  editor/CI schema validation.
- JSON Schema from F# types: FSharp.Data.JsonSchema 3.x.

## Data sources plan (details + access status: docs/memory/data-sources.md)

- **Motherboards**: seed BuildCores OpenDB (ODC-By) → enrich Open Icecat (ASUS+Gigabyte,
  GTIN identity) → vendor scrapes (ASUS open incl. BIOS JSON API; MSI/Gigabyte/ASRock/
  Supermicro need residential IP) → linuxhw/DMI corroboration → manual PDFs for lane-sharing
  (semi-manual, LLM-assisted + human confirm flag).
- **Routers/APs**: openwrt.org/toh.json nightly (CC BY-SA 4.0) + wikidevi.wi-cat.ru ask API
  (CC BY-SA 3.0) + fcc.report per FCC ID + devicecode-data baseline.
- **Switches**: no community DB exists (unclaimed niche, like BIOS history). ToH's 88
  realtek-target units + wi-cat + MikroTik (trivially scrapeable) + Ubiquiti public.json +
  vendor datasheets.
- **BIOS history**: build the ASUS GetPDBIOS poller early (append-only; Wayback CDX backfill;
  contract tests for API shape). Nobody else has this data; BuildCores solicits it upstream.
- Planned public artifact: an AI/bot-readable mirror of ToH data (+ accumulated change
  history), CC BY-SA with attribution.

## Licensing tiers

| Tier | License | Examples |
|---|---|---|
| Own published catalog | CC BY-SA | normalized facts, event-derived history |
| Share-alike ingested | CC BY-SA 4.0/3.0 | toh.json, wi-cat (3.0→4.0 is one-way OK) |
| Attribution ingested | ODC-By | BuildCores |
| Facts-only extraction | n/a (facts uncopyrightable) | vendor spec pages, MikroTik/Ubiquiti |
| Private archive | none — do not redistribute | raw scraped HTML, manual PDFs, firmware binaries |

## Milestones (proposed)

1. Domain model + event types in F# on .NET 11; round-trip serialization tests (format freeze).
2. BuildCores importer → events → SQLite projection; DuckDB over it (working
   mATX/ITX + slots + 2.5GbE filter).
3. ASUS BIOS poller (append-only, SPN2 push, Wayback CDX backfill) — the perishable data first.
4. toh.json + wi-cat ingest; identity/alias resolution.
5. Garage blob store + browsertrix capture path; provenance end-to-end.
6. Fun.Blazor GUI with requirement-matching queries.

## Open questions

- **Project name.** "TOH" collides with OpenWrt's Table of Hardware (confusing once we mirror
  their data); "HWDB" collides with systemd/udev's hwdb. Undecided; rename is cheap until
  publication.
- Meilisearch/Typesense: only if the public site later needs typo-tolerant search; not needed
  at current scale.
- lane-sharing extraction workflow (PDF → LLM-assisted → human-verified flag): tooling TBD.
