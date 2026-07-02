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
- **2026-07-02** Public data will be **CC BY-SA 4.0** (version pinned — 3.0→4.0 ingest is
  one-way, so the catalog must be 4.0) — dissolves share-alike compatibility with
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
- **2026-07-02** **Name decided: HWology** ("hardware-ology"). hwology.com registered;
  .net/.dev consciously skipped (Ivan already pays for too many domains; the .com is what
  matters). Directory renamed `/home/ivan/TOH` → `/home/ivan/hwology` same day.
- **2026-07-02** Project ethos, in Ivan's words: personal use first, public sharing second
  tier, a give-back from 30+ years of internet usage. Copying/forking is welcome — no brand
  defensiveness. Keep decisions biased toward simple and personal-scale.
- **2026-07-02** Repo published publicly: **github.com/HWology/hwology** (org HWology;
  org description + hwology.com URL set). LICENSE added: CC BY-SA 4.0 covering the repo's
  prose and the future published data; future code will carry its own license (open question).
  **This repo — including these memory files — is now public**: keep secrets, personal data,
  and sensitive operational notes out of it.
- **2026-07-02** Porkbun API access verified live from the dev machine (key scoped per
  Porkbun account settings; credentials kept outside this public repo). hwology.com has the
  per-domain API-Access toggle enabled and is manageable via API.
- **2026-07-02** hwology.com zone cut over via Porkbun API (authority stays on Porkbun NS;
  Fastmail's hosted-DNS defaults disabled on their side). Design: apex + wildcard A/AAAA →
  prod VPS (15.204.119.184 / 2604:2dc0:202:300::268b); mail on Fastmail via apex+wildcard
  MX (us1/us2-smtp.messagingengine.com — verified same IPs as legacy in1/in2 names), SPF
  `include:spf.messagingengine.com ?all`, 4 DKIM CNAMEs (fm1-3, mesmtp → *.dkim.fmhosted.com),
  DMARC p=none (no rua yet — add before tightening), full Fastmail SRV autodiscovery set incl.
  `0 0 .` negatives for plaintext services. CAA locks issuance to letsencrypt.org (issue +
  issuewild) — **any ACME client on the VPS must use Let's Encrypt** (acme.sh defaults to
  ZeroSSL — override or amend CAA). No mail.hwology.com records: wildcard sends it to the VPS
  (Fastmail's webmail-redirect A record deliberately skipped). TTLs 600 during setup; raise
  to 3600 once stable. All records verified resolving on Porkbun NS same day (CAA lagged
  sync to the Cloudflare backend by ~2 min — recheck if issuance ever fails).
- **2026-07-02** mail.hwology.com pointed at Fastmail after all (reversing the earlier skip):
  A → 103.168.172.65 (their webmail-redirect host) + MX pair. The explicit A also correctly
  suppresses wildcard AAAA synthesis at that name (empty AAAA, matching Fastmail's default).
  Fastmail's hosted-domain certs are Let's Encrypt (their help: "Secure website support:
  Let's Encrypt") — compatible with the letsencrypt.org-only CAA.
- **2026-07-02** Pre-publication sweep (multi-agent) before first push: reworded bot-wall/ToS
  passages in research docs to neutral, fact-only phrasing; removed a host IP; converted
  remaining relative dates to absolute. Dataset NOTICE/ATTRIBUTION obligation pre-staged in
  architecture.md (triggers at milestone 2's first published record).

## Open / undecided

- License for the future source code (CC BY-SA is unsuitable for software — Creative Commons
  itself recommends against it); decide by milestone 1, declare in its own LICENSE file.

## Naming history (resolved 2026-07-02 → HWology)

"TOH" was consciously borrowed from OpenWrt's Table of Hardware but collides with it;
"HWDB" collides with systemd/udev hwdb. **HWology** collision-checked clear (all registries
free, domains unregistered at RDAP level, no trademarks); hwology.com registered by Ivan.
Other candidates collision-checked against the live web 2026-07-02, kept for the record:
  - **spectrail** — near-clear (unrelated Chrome extension/indie game only; npm/NuGet/PyPI/
    crates all free). Meaning: spec + audit trail.
  - **hwpedigree** — fully clear everywhere; slightly clunky to say.
  - **hardfacts** — ownable; GitHub handle squatted by empty account; reads a bit generic.
  - **silkscreen** — no hard conflict but generic word (font + PCB layer dominate search).
  - **hwledger** — rejected: search engines autocorrect to hledger; reads as LedgerHQ
    hardware wallet.
  - **specledger** — rejected: exact-name active product (specledger.io + GitHub org).
  - **"Tome of Hardware"** — rejected: near-homophone of Tom's Hardware (same subject area,
    trademark-confusion exposure) and the TOH backronym resurrects the OpenWrt collision.

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
