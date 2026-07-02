# TOH — hardware spec catalog (working name; see naming note in docs/memory/project.md)

A public, spec-level hardware catalog with faceted filtering retail sites can't do, plus
data retailers never carry: firmware/BIOS release history, per-slot electrical lane widths,
lane-sharing rules, and reachable provenance for every fact. Motherboards first, then wifi
routers, switches, and similar gear.

Owner: Ivan. Status: research/design phase — no code yet.

## Session bootstrap (do this first)

1. Read `docs/memory/project.md` — living project memory: decisions log, open questions,
   hard-won requirements. **Update it whenever a decision is made or a fact changes**; date
   volatile entries. This is the shared memory for all agents (Claude, Codex, humans).
2. Read `docs/memory/data-sources.md` — verified data sources and access status (dated;
   perishable — re-verify old entries before building on them).
3. Read `docs/architecture.md` before proposing or making design changes.
4. `docs/research/<date>/` holds web-verified research reports backing the architecture.
   Cite them; don't re-research what's already verified.

## Ground rules

- **Stack**: F# on .NET 11 previews (owner deliberately runs newest). GUI will be the owner's
  own fork of Fun.Blazor. No C#-first frameworks; check `docs/architecture.md` stack pins
  before adding packages.
- **Architecture**: event-sourced ingestion (append-only observations with provenance);
  SQLite and the published dataset are rebuildable projections, never sources of truth.
  Machine-written records are JSON (.NET 11 built-in STJ F# DU format); hand-authored files
  are TOML; both validate against JSON Schema generated from the F# types.
- **Provenance**: every fact carries source URL + retrieved-at (ISO-8601 UTC) + sha256 +
  archive ref. Trust rank: `measured > correction > manual-PDF > vendor-page > aggregator`.
  Never delete vendor-pulled data; flag it `pulled`.
- **Licensing tiers**: published data is CC BY-SA. Raw scraped HTML, manual PDFs, and
  firmware binaries are private (Garage S3, sha256-keyed) and referenced publicly by
  URL + hash + Wayback link only. Do not commit copyrighted material to this repo.
- **Scraping**: be polite (low rate, identify honestly where sensible). Several vendors
  bot-block datacenter IPs — see data-sources memory before pointing a scraper anywhere.
- Convert relative dates to absolute in all docs and memory files.
