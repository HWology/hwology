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
  prose and the future published data; future code will carry its own license (decided later
  same day: AGPL-3.0, see below).
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
  Wildcard behavior dig-verified on Porkbun/Cloudflare NS 2026-07-02: any explicit record at
  a name suppresses ALL wildcard synthesis there (name exists → NODATA for other types) and
  names *under* a record-bearing node go true NXDOMAIN; but names under a mere empty
  non-terminal (e.g. `anything._tcp.hwology.com`) still get wildcard answers — Cloudflare
  matches the apex wildcard through ENTs, deviating from RFC 4592. Blocker idiom: one inert
  TXT at the name (use the record's `notes` field to say why); no native "block" type exists.
- **2026-07-02** Mail auth hardened to maximum (Ivan's call: fresh domain, Fastmail is the
  only sender, no ramp needed): apex SPF → `-all`; wildcard `*.hwology.com` TXT SPF `-all`
  (covers subdomain-addressing sends and spoofing); DMARC → `p=reject;
  rua=mailto:dmarc@hwology.com` (mailbox live). Alignment stays RELAXED (defaults — do NOT
  add `adkim=s`): Fastmail signs d=hwology.com, and strict alignment would break sending
  from Ivan's per-service `user@sub.hwology.com` addresses; receiving via wildcard MX is
  unaffected by all of this. Also added `*._report._dmarc` TXT `v=DMARC1` so hwology.com can
  receive DMARC reports for OTHER domains (ivanthegeek.com etc. planned) — required because
  the wildcard SPF TXT would otherwise be synthesized for `<domain>._report._dmarc` lookups
  and receivers would refuse cross-domain rua. If mail must ever originate outside Fastmail,
  revisit SPF (route via Fastmail SMTP preferred).
  A → 103.168.172.65 (their webmail-redirect host) + MX pair. The explicit A also correctly
  suppresses wildcard AAAA synthesis at that name (empty AAAA, matching Fastmail's default).
  Fastmail's hosted-domain certs are Let's Encrypt (their help: "Secure website support:
  Let's Encrypt") — compatible with the letsencrypt.org-only CAA.
- **2026-07-02** Pre-publication sweep (multi-agent) before first push: reworded bot-wall/ToS
  passages in research docs to neutral, fact-only phrasing; removed a host IP; converted
  remaining relative dates to absolute. Dataset NOTICE/ATTRIBUTION obligation pre-staged in
  architecture.md (triggers at milestone 2's first published record).

- **2026-07-02** **Code license decided: AGPL-3.0** for the F# source (Ivan's call). Own
  LICENSE file lands with the code; pin the only/or-later variant at that point. Caveat
  recorded in architecture.md: CC BY-SA 4.0's one-way compatibility covers GPLv3 only, not
  AGPL — data and code stay separate artifacts, never fold BY-SA material into the code tree.

## Open / undecided

- DMARC report processor for dmarc@hwology.com (multi-domain: ivanthegeek.com etc. will
  point rua here too; `*._report._dmarc` authorization already published). Shortlist
  (multi-agent verified 2026-07-02): **cry-inc/dmarc-report-viewer** (recommended — Rust,
  single ~10 MB container, built-in IMAP+UI, per-domain filters, monthly releases; stateless:
  the mailbox IS the datastore, so keep report mail) vs **dmarcguardhq/dmarcguard** (Go+SQLite
  persistence, single 14 MB container, very active but only ~8 months old) vs **liuch/dmarc-srg**
  (most mature viewer; PHP+MariaDB, 2 containers on third-party images, and each report domain
  must be added to its allow-list). parsedmarc itself is healthy (10.2.0, 2026-06; v10 added a
  PostgreSQL backend) but its lightest UI path is still 3 containers.
  Trial-plan facts (source-verified 2026-07-02): the two shortlisted tools must NOT share one
  IMAP folder — cry-inc's fetch is always non-PEEK (marks everything `\Seen`) while dmarcguard
  discovers work solely via SEARCH UNSEEN, so it gets silently starved. Setup: two Fastmail
  folders + Sieve `fileinto :copy` rule, one app password per tool; keep dmarcguard's
  `IMAP_PROCESSED_MAILBOX` disabled (its move path expunges ALL `\Deleted` in the folder and
  uses sequence numbers, not UIDs). dmarcguard's dashboard/API has NO auth and CORS `*` —
  reverse-proxy auth mandatory; cry-inc has built-in Basic Auth. dmarcguard.io SaaS ≠ the OSS
  repo (Apache-2.0, no telemetry, same author): of its marketed "9 protocols" only DMARC (+
  embedded SPF/DKIM results) is in the OSS; the rest is closed or ROADMAP-planned.
- Auth in front of self-hosted web UIs (candidate, not yet deployed): **Pocket ID**
  (passkey-only OIDC, 1 container) + **Tinyauth** (Caddy forward_auth middleware, OIDC client
  w/ auto-redirect) — mutually documented pairing, both very active 2026-07. Single-container
  alternative: Authelia ≥4.39 (`webauthn.enable_passkey_login`). Avoid caddy-security
  (WebAuthn is password+U2F 2FA only; 10 unpatched 2024 GHSAs). Pocket ID's WebAuthn rpID
  binds to its hostname — choose e.g. id.&lt;domain&gt; deliberately, renaming re-registers
  passkeys.

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
