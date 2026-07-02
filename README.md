# HWology

*"hardware-ology" — the study of hardware.* — [hwology.com](https://hwology.com)

A public, spec-level hardware catalog with faceted filtering retail sites can't do
("motherboard, mATX/ITX, 3+ PCIe slots, onboard 2.5GbE, onboard video"), plus what retailers
never carry: firmware/BIOS release history, per-slot electrical lane widths, lane-sharing
rules, and reachable provenance for every fact. Motherboards first; wifi routers, switches,
and similar gear after.

**Status:** research/design phase — docs only, no code yet. Planned build: F# on .NET,
event-sourced ingestion.

## What's here

- [docs/architecture.md](docs/architecture.md) — the design: event-sourced ingestion,
  provenance model, domain sketch, stack pins, milestones.
- [docs/research/2026-07-02/](docs/research/2026-07-02/README.md) — twelve web-verified
  research reports the design is grounded in (data sources, schema/storage, firmware data,
  archiving, licensing).
- [docs/memory/](docs/memory/project.md) — living decision log and data-source access notes.

## Licensing

Everything in this repository today — documentation, architecture, research reports — is
licensed [CC BY-SA 4.0](LICENSE), and the published dataset, when it ships, will be too.
Source code is not part of this repository yet; when it lands it will carry its own license,
declared at that time.

Copyrighted source material (manuals, firmware, raw scraped pages) is never committed here —
it lives in a private archive and is referenced by URL + sha256 + Wayback link only.

---

Personal-first: built for the owner's own use; public sharing is the second tier and a
give-back after 30+ years on the internet. Copy freely within the license.
