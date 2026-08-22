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

- [development/architecture.md](development/architecture.md) — the design: event-sourced ingestion,
  provenance model, domain sketch, stack pins, milestones.
- [development/research/2026-07-02/](development/research/2026-07-02/README.md) — twelve web-verified
  research reports the design is grounded in (data sources, schema/storage, firmware data,
  archiving, licensing).
- [development/memory/](development/memory/project.md) — living decision log and data-source access notes.
- [development/WISHLIST.md](development/WISHLIST.md) — requirements draft feeding the event-modeling pass; not commitments.

## Licensing

Everything in this repository today — documentation, architecture, research reports — is
licensed [CC BY-SA 4.0](LICENSE), and the published dataset, when it ships, will be too.
Source code is not part of this repository yet; when it lands it will be **AGPL-3.0**,
declared in its own LICENSE file alongside the code. How the two copylefts coexist (and
why the data never gets compiled into the code) is worked through in
[development/licensing.md](development/licensing.md).

Copyrighted source material (manuals, firmware, raw scraped pages) is never committed here —
it lives in a private archive and is referenced by URL + sha256 + Wayback link only.

---

Personal-first: built for the owner's own use; public sharing is the second tier and a
give-back after 30+ years on the internet. Copy freely within the license.
