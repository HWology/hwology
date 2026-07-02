# Adversarial claim verification

*A fact-check agent re-verified the most load-bearing claims from the 2026-07-02 research against the live web.*

## BuildCores OpenDB is alive, licensed ODC-By 1.0, has ~3,693 Motherboard JSONs across 30 categories, and its README explicitly solicits motherboard BIOS versioning data (confirming the BIOS-history gap)

**Status: CONFIRMED**

Verified-live via GitHub API and raw files. Repo pushed_at 2026-07-01T18:07Z, 320 stars, 150 forks, 117 open issues. README 'License' section states Open Data Commons Attribution (ODC-By) v1.0 verbatim (license file is LICENSE.txt, which is why GitHub API reports 'Other'). git-trees API: open-db/ has exactly 30 category dirs and Motherboard/ contains exactly 3,693 files, untruncated. README 'Help Wanted / Bounties' lists 'collect motherboard BIOS versioning data along with CPU support lists' and product-page URLs/PDFs as unmet goals, and 'Limitations' confirms no price/retailer data. Seeding recommendation stands.

## BuildCores Motherboard.schema.json has structured pcie_slots (gen enum 1.1–5.0, lanes, quantity), m2_slots (size, key enum M/E/B/B+M, interface), onboard_ethernet (speed enum incl. 2.5 Gb/s + controller), but onboard video out exists only as free-text strings in back_panel_ports

**Status: CONFIRMED**

Verified-live from raw schema (16,944 bytes). pcie_slots items: gen enum ['5.0','4.0','3.0','2.0','1.1',null], quantity (number), lanes (number). m2_slots items: size, key enum ['M','E','B','B+M',null], interface. onboard_ethernet items: speed enum ['10 Gb/s','5 Gb/s','2.5 Gb/s','1 Gb/s','100 Mb/s',null] plus controller string. Top-level properties contain NO video-output field; back_panel_ports is 'array of string' — so the '3+ PCIe slots' and '2.5GbE' facets map directly, and the video-out facet does require parsing back_panel_ports strings, exactly as claimed.

## ASUS undocumented JSON API GetPDBIOS returns full BIOS history (Version, ReleaseDate, changelog with AGESA, SHA256, DownloadUrl.Global) and works from a datacenter IP with no auth

**Status: CONFIRMED**

Verified live 2026-07-02: curl to https://www.asus.com/support/api/product.asmx/GetPDBIOS?website=us&model=ROG%20STRIX%20B650E-I%20GAMING%20WIFI from this datacenter host returned HTTP 200, 35 KB JSON, 32 BIOS releases; newest Version 3854, ReleaseDate 2026/04/23, Description begins 'Update AGESA ComboAM5 PI 1.3.0.1', DownloadUrl has Global+China keys. The BIOS-history pipeline's most reliable vendor source is real and currently open.

## ASRock is the cleanest scrape target: static HTML specs and per-board BIOS pages plus a sitewide same-day BIOS feed, served 200 to datacenter curl with no bot-blocking (robots.txt disallows only /asrockrack/)

**Status: CORRECTED**

Materially changed since the original check. On 2026-07-02 every ASRock content page tested — /support/index.asp?cat=BIOS, /mb/amd/B650M%20Pro%20RS/bios.html, /mb/index.asp — returned an ~840-byte Imperva Incapsula challenge iframe ('Request unsuccessful. Incapsula incident ID...') to curl-with-Chrome-UA from a datacenter IP; Anthropic's fetcher also got empty content for the X870E Taichi White spec page. robots.txt IS as claimed (only /asrockrack/ disallowed). So the data structures may still exist, but ASRock must be moved into the same bucket as MSI/Gigabyte for access planning: residential IP or headless browser, not 'clean datacenter curl'. This weakens the 'daily poll of ASRock sitewide feed is cheap and reliable' part of the architecture; ASUS becomes the only vendor confirmed scrape-friendly from cloud infra.

## Open Icecat sponsoring brands include ASUS (vendor 120) and Gigabyte (480), but NOT MSI, ASRock, or Supermicro (those require paid Full Icecat)

**Status: CONFIRMED**

Verified-live against the ITscope Open Icecat sponsors list (guide.itscope.com, list dated 2025-08): '120 ASUSTeK COMPUTER' and '480 Gigabyte Technology' present; MSI, ASRock, and Supermicro absent from the sponsor list. Free-tier Icecat coverage of exactly two of the five major board vendors holds, supporting the 'baseline + manufacturer enrichment' plan.

## InstantSearch is viable self-hosted because both engines maintain official adapters: meilisearch 'instant-meilisearch' and 'typesense-instantsearch-adapter'

**Status: CONFIRMED**

Verified-live via GitHub/npm APIs. instant-meilisearch now lives in the meilisearch/meilisearch-js-plugins monorepo (pushed 2026-07-01, not archived; npm @meilisearch/instant-meilisearch latest 0.31.2 published 2026-06-18). typesense/typesense-instantsearch-adapter: pushed 2026-03-26, 518 stars, not archived. Both officially maintained; the standalone meilisearch/meilisearch-angular repo is archived but irrelevant. UI-layer recommendation stands.

## Newegg has no public product/spec API — developer.newegg.com documents only the seller-credential Marketplace API; no public catalog reads

**Status: CONFIRMED**

Direct fetch of developer.newegg.com was Akamai-blocked from this host (403 edgesuite), so confirmed via live search over the portal's indexed pages: every documented API is Marketplace-scoped (item/order/datafeed management, api.newegg.com/marketplace/ base URLs, auth requires seller Secret Key + developer API key per Seller Portal academy page). No product-catalog/spec read API appears anywhere; only ancient unofficial wrappers (e.g. a 2013-era Ruby 'Newegg Mobile API' shim) exist. 'Newegg is a UX benchmark, not a data source' stands.

## pgloader migrates SQLite→Postgres in one command and notably auto-converts TEXT columns to JSONB when content is valid JSON, landing the SQLite 'JSON in TEXT' pattern directly on JSONB

**Status: CORRECTED**

Single-command migration (schema+data+indexes via default 'create tables, create indexes, reset sequences'), default type mappings, and custom CAST rules: all confirmed in the official SQLite reference (pgloader.readthedocs.io/en/latest/ref/sqlite.html). But the auto TEXT→JSONB claim is NOT in the official docs — the SQLite cast rules list only numeric/text/binary/date mappings, no JSON. It traces to a render.com tutorial, and pgloader issue #1607 shows SQLite's NATIVE JSONB binary type actually fails ('invalid input syntax for type json', closed via PR #1705). Correct plan: store JSON as plain TEXT in SQLite (never SQLite-native JSONB) and add an explicit CAST rule (or post-migration ALTER ... TYPE jsonb USING ...) — a minor, mechanical addition, but don't count on automatic conversion. Migration path remains viable.
