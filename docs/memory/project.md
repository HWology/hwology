# Project memory — living notes

*Agents (Claude, Codex, humans): read this at session start, update it whenever a decision is
made, reversed, or a fact changes. Date volatile entries. Durable design belongs in
[../architecture.md](../architecture.md); this file is for context, decisions, and state.*

## Why this project exists

Ivan's long-standing frustration: no retailer supports precise hardware spec filtering
(e.g. "motherboard, mATX/ITX, 3+ PCIe slots, onboard 2.5GbE, onboard video"). Newegg is the
least-bad filter-UX reference; Amazon is the anti-goal (returns non-matching results).
Firmware/BIOS detail — which retailers never carry — matters as much as specs.

## Decisions log

- **2026-07-02** Stack: F# / .NET 11 previews (Ivan runs newest happily; GA Nov 2026). GUI
  will be his own fork of Fun.Blazor. Storage: git-versioned text canon + SQLite read model +
  in-memory app model. Event-sourced ingestion (Ivan's framing: SQLite is a "view"/read model).
- **2026-07-02** Public data will be **CC BY-SA** — dissolves share-alike compatibility with
  OpenWrt ToH / WikiDevi sources.
- **2026-07-02** File formats: JSON for machine-written records (.NET 11 built-in STJ F# DU
  format), TOML for hand-authored files (Ivan dislikes JSON bracket-editing). Both validated
  against JSON Schema generated from the F# types.
- **2026-07-02** Blob storage: Ivan runs a **Garage** (S3-compatible) instance — PDFs,
  firmware images, WARCs/WACZ go there, sha256-keyed, private + public buckets.
- **2026-07-02** Provenance must be end-user-reachable: any published fact resolves to
  source URL + retrieved-at + sha256 + Wayback link (bytes stay private where copyrighted).
- **2026-07-02** Archiving: browsertrix-crawler + single-file-cli + Wayback SPN2 (not
  ArchiveBox — v0.7.4/v0.9-RC limbo, single maintainer).
- **2026-07-02** Planned public artifact: AI/bot-readable mirror of OpenWrt toh.json with
  accumulated change history (the wiki HTML bot-blocks; toh.json itself is open, CC BY-SA 4.0).
- **2026-07-02** Query/analysis tooling: **SQLite (operational read model) + DuckDB
  (exploration, ingest checks, Parquet publishing to Garage). Datasette skipped** — DuckDB's
  local UI covers exploration; no Python sidecar wanted.
- **2026-07-02** Repo created (git init, main branch) with docs, AGENTS.md/CLAUDE.md, memory.

## Open / undecided

- **Name**: "TOH" was consciously borrowed from OpenWrt's Table of Hardware but collides with
  it (awkward once we mirror their data); "HWDB" collides with systemd/udev hwdb. Scope has
  outgrown "table" anyway (catalog + archive + firmware ledger). Rename is cheap until
  publication. Candidate direction: ledger/history-flavored names fit the event-sourced,
  append-only identity.

## Hard-won requirements (Ivan's own experience)

- PCIe slot queries are **requirement lists with min-constraints**: e.g. "2× x16-physical /
  x8-electrical / Gen4+, plus 2× x1 Gen3+" — physical size, electrical lanes, and generation
  are separate queryable dimensions, evaluated with lane-sharing awareness (matching problem,
  done in F#, not SQL).
- M.2 electrical widths are real-world trial-and-error: Ivan owned a board with 3 M.2 slots —
  two x1-electrical, one x2-electrical — and had to discover which was which by testing.
  Capture per-slot electrical width + silkscreen label; vendor pages often omit this; owner
  measurements are first-class, highest-trust data.
- Spreadsheets ruled out (his instinct, confirmed: sparse heterogeneous multi-valued specs
  need child tables + attribute registry).

## Session bootstrap for agents

1. Read this file and [data-sources.md](data-sources.md).
2. Read [../architecture.md](../architecture.md) before proposing design changes.
3. Research evidence lives in [../research/](../research/) (dated dirs) — cite, don't re-research
   what's already verified; do re-verify perishable access details if months old.
