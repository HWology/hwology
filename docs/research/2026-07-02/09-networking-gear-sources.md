# Networking gear data sources (routers, switches, FCC)

*Research date: 2026-07-02, parallel web-research agents. Findings marked `verified-live` were confirmed against the live web that day; `reported` means sourced but not directly fetched; treat access details as perishable.*

## Summary

Router data is in excellent shape: the OpenWrt Table of Hardware now ships a sanctioned bulk artifact (openwrt.org/toh.json, 3.9 MB, regenerated nightly — pulled 2026-07-02: 2,996 devices x 74 columns incl. CPU, flash/RAM, per-speed ethernet port counts, SFP/SFP+, wifi chips+drivers, FCC ID, target/subtarget, support status; CC BY-SA 4.0), and the WikiDevi successor wikidevi.wi-cat.ru is alive, actively edited (Ubiquiti switch pages edited 2026-07-01), and exposes a full Semantic MediaWiki ask API returning typed facts (chipsets, flash/RAM with units, FCC IDs) under CC BY-SA 3.0. FCC internals are best reached via fcc.report (plain-curl scrapeable per-FCC-ID pages listing internal photos/test reports/manuals as PDFs); the official apps.fcc.gov EAS API exists (KDB 953436) but Akamai-blocks datacenter IPs, and fccid.io bot-blocks. DD-WRT's router database and TechInfoDepot are both behind Anubis anti-bot walls and the DD-WRT DB is community-acknowledged stale — skip both; the armijnhemel/devicecode-data repo is a better pre-parsed JSON aggregate of TechInfoDepot+WikiDevi+OpenWrt+FCC. For switches there is no dedicated structured community DB: coverage comes from ToH (88 realtek-target managed switches), wi-cat WikiDevi, Ubiquiti's live public.json fingerprint file (678 devices), and vendor pages — MikroTik's product pages are fully server-rendered with clean key-value spec blocks and are trivially scrapeable (verified). One gotcha for Ivan's licensing model: ToH and WikiDevi data are share-alike (CC BY-SA), not attribution-only like BuildCores' ODC-By, so his public facts layer needs a compatible license or a facts-only extraction stance.

## Findings

### OpenWrt ToH: toh.json is the canonical bulk export — verified by downloading and parsing it on 2026-07-02

*Confidence: `verified-live`*

Data pipeline: per-device DokuWiki pages under openwrt.org/toh/hwdata/<brand>/<model> hold 'dataentry' blocks (edited via the wiki's dataentry plugin form; wiki self-registration disabled, access via forum request). A nightly job compiles them into https://openwrt.org/toh.json — Last-Modified header showed 2026-07-02 17:30 UTC, 3,978,260 bytes. Structure: {captions[74], columns[74], entries[2996]}. Columns include: deviceid, brand, model, version, devicetype, availability, cpu, cpucores, cpumhz, flashmb, rammb, ethernet100mports/1g/2_5g/5g/10gports, sfp_ports, sfp_plus_ports, switch (switch chip), wlan24ghz/50ghz/60ghz/600ghz (standards per band), wlanhardware (radio chips), wlandriver, fccid, wikideviurl, oemdevicehomepageurl, target/subtarget, packagearchitecture, supportedsincerel, supportedcurrentrel (values up to 25.12.1 — current), supportedsincecommit, installationmethods, recoverymethods, serial/serialconnectionparameters/jtag/gpios, powersupply, detachableantennas, picture, firmware OEM+OpenWrt install/upgrade/snapshot URLs, forum/git search terms. Device types: 1376 WiFi Router, 435 WiFi AP, 417 SBC, 187 Modem, 140 Router, 113 Range Extender, 88 Switch, 72+41 Travel Router, 52 NAS. Gotchas: (1) server returns 503 to non-browser user agents — needs a browser-ish UA; (2) the fccid column stores http://fcc.io/... shortener links and fcc.io is dead (see FCC finding); (3) many list-valued fields are stringified Python-style lists. License: CC BY-SA 4.0 (wiki footer). Bulk access is clearly intended — toh.json is what the official frontend itself consumes.

Sources:

- <https://openwrt.org/toh.json>
- <https://openwrt.org/toh/start>
- <https://openwrt.org/toh/hwdata/tp-link/tp-link_eap225_v1>

### toh.openwrt.org: new official JS frontend with CSV/JSON export, GPL-2.0 code on GitHub

*Confidence: `verified-live`*

toh.openwrt.org is the enhanced official ToH UI (repo github.com/openwrt/toh-openwrt-org, GPL-2.0, v1.79 released May 2026, ~350 commits, active). It reads openwrt.org/toh.json and offers per-view CSV/JSON export of all or filtered rows (since v1.78b2). For Ivan's pipeline the frontend is irrelevant — consume toh.json directly — but the repo documents the data contract and the nightly generation process.

Sources:

- <https://toh.openwrt.org/>
- <https://github.com/openwrt/toh-openwrt-org>

### WikiDevi successor wikidevi.wi-cat.ru: alive, actively edited, full Semantic MediaWiki API — best structured source for radios/chipsets/FCC IDs

*Confidence: `verified-live`*

Verified via api.php: MediaWiki 1.35.3, 16,269 content articles / 36,313 pages, 10,183 images, edits as recent as 2026-07-01 — notably on switches (Ubiquiti USW Pro XG 8 PoE, USW Aggregation). License: CC BY-SA 3.0 (rightsinfo API). Extensions include SemanticMediaWiki, Semantic Drilldown, SemanticResultFormats, PageForms — the original WikiDevi's semantic schema survived. I ran a live ask-API query ([[Embedded system type::wireless router]] with ?CPU1 model|?FLA1 amount|?RAM1 amount|?FCC ID) and got typed results with units (e.g. 2Wire 2700HG-B: CPU 'Perseus', FLA1 16 MiB, RAM1 32 MiB, FCC ID PGR2W2700RD). Fields carried per device: FCC ID, CPU1/CPU2 brand+model+clock, flash/RAM chips+amounts, WI1/WI2 radio chipsets, antenna config, ethernet ports+switch chip, OUI, default IP/credentials, firmware info, links to OpenWrt/DD-WRT/TechInfoDepot pages. Access: open MediaWiki API (action=ask supports JSON; Special:Ask supports CSV export); no stated bulk/scraping policy and no public dumps of the restored wiki — history was not preserved from the original. It's a small hobbyist-run Russian-hosted server: rate-limit politely and archive what you extract. Sister mirror deviwiki.com is up (HTTP 200) and per wi-cat's About page serves as the read-only archive holding original edit history.

Sources:

- <https://wikidevi.wi-cat.ru/Main_Page>
- <https://wikidevi.wi-cat.ru/api.php>
- <https://deviwiki.com/>

### Original WikiDevi dumps on archive.org: final Oct 2019 wikitext DB + 9 GB image dumps, CC BY-SA 3.0

*Confidence: `verified-live`*

archive.org item 'wikidevi' (verified via metadata API) contains wikidevi_db_2019_10_04.tar.xz (17.7 MB, final MediaWiki export wikitext), earlier snapshots back to 2013, wikidevi_images_2019_09_27.tar.xz (9.1 GB), wikidevi_files_2018_01_18.tar.xz (3.8 GB), plus a torrent. licenseurl on the item: CC BY-SA 3.0. Good as a frozen baseline / provenance archive, but wi-cat.ru was itself restored from these dumps and has 6+ years of newer edits, so prefer the live wiki for current gear.

Sources:

- <https://archive.org/details/wikidevi>
- <https://archive.org/download/wikidevi>

### devicecode + devicecode-data (armijnhemel): pre-parsed unified JSON of TechInfoDepot + WikiDevi + OpenWrt + FCC — the shortcut past all the bot walls

*Confidence: `verified-live`*

github.com/armijnhemel/devicecode (Apache-2.0, ~1,168 commits) parses wiki dumps into normalized per-device JSON; companion repo devicecode-data ships the pre-generated output so you never need to dump the wikis yourself — layout is per-source directories with original/ (raw XML), devices/ (cleaned JSON), overlays/ (corrections). Data licenses are tracked per source: TechInfoDepot & WikiDevi CC BY-SA 3.0, OpenWrt CC BY-SA 4.0, FCC metadata CC0, OUI files GPL-2.0. Last commits on both repos: 2025-05-06 ('add new FCC data') — about 14 months old, so treat as a rich but slightly stale baseline to merge with live toh.json and wi-cat queries. Built for VulnerableCode reuse, includes a TUI browser. This is the closest existing thing to what Ivan is building and worth studying for schema-unification decisions (it already handles the WikiDevi vs TechInfoDepot vs ToH field mapping).

Sources:

- <https://github.com/armijnhemel/devicecode>
- <https://github.com/armijnhemel/devicecode-data>

### FCC official: EAS Web API exists (KDB 953436) but apps.fcc.gov blocks datacenter IPs; data.gov bulk file is grantee-level and stale

*Confidence: `reported`*

The FCC OET publishes an EAS Web API (KDB publication 953436 D01 'OET Laboratory Services API', e.g. getFCCIDList returning matches for full/partial FCC IDs) plus the GenericSearch.cfm advanced search over all grant fields. Every fetch of apps.fcc.gov from this environment returned Akamai 'Access Denied' (403) — this is a datacenter-IP/geo block, so Ivan should test from his residential connection; the API itself requires no key per the docs. Bulk: data.gov hosts 'EAS Equipment Authorization Grantee Registrations' (CSV/JSON/XML/RDF, US-government public domain) but it covers grantee registrations, not per-device grants, and the underlying data hasn't refreshed since March 2021. FCC grant metadata and filings are public records (effectively public domain); note however that attached user manuals and internal photos are manufacturer-copyrighted documents — exactly the material for Ivan's private archive tier, not the public one.

Sources:

- <https://apps.fcc.gov/oetcf/kdb/forms/FTSSearchResultPage.cfm?id=50070&switch=P>
- <https://www.fcc.gov/oet/ea/fccid>
- <https://catalog.data.gov/dataset/eas-equipment-authorization-grantee-registrations>

### FCC third-party mirrors: fcc.report is plainly scrapeable (verified), fccid.io bot-blocks, fcc.io shortener is dead (breaks ToH links)

*Confidence: `verified-live`*

fcc.report: verified by curling https://fcc.report/FCC-ID/TE7EAP225 with a plain UA — clean HTML listing equipment name/grantee/address, per-application frequency ranges and grant dates, and the full document list (Internal Photos, External Photos, Test Reports, RF Exposure, Users Manual) with direct PDF links. No API but the URL scheme is predictable (/FCC-ID/<id>) and pages are static-ish; no license stated (underlying FCC filings are public records). fccid.io: homepage 200 in a browser context but per-ID pages return 403 to non-browser clients (bot protection) — usable manually, poor for automation. fcc.io (the shortener the OpenWrt ToH fccid column links through, e.g. http://fcc.io/TE7EAP225): returns 404 with no redirect — dead, so ToH FCC links must be rewritten to fccid.io/fcc.report/apps.fcc.gov patterns. Chipset identification from internal photos remains a manual/OCR task — the WikiDevi successors already curated that mapping, which is why they stay valuable.

Sources:

- <https://fcc.report/FCC-ID/TE7EAP225>
- <https://fccid.io/>
- <https://fcc.io/>

### DD-WRT router database and TechInfoDepot: both alive but behind Anubis anti-bot walls; DD-WRT DB community-flagged as unmaintained — skip as data sources

*Confidence: `verified-live`*

dd-wrt.com/support/router-database/ returns 200 to a bare HEAD but the actual page is gated by Anubis (Techaro) proof-of-work anti-bot — both curl and headless fetch got challenge/Access-Denied pages. DD-WRT forum consensus (threads through 2025) is that the router database is outdated and recommends known-bad old builds; the wiki's supported-devices list is also stale. No API, no export, no data license. techinfodepot.shoutwiki.com (the third WikiDevi fork, CC BY-SA 3.0, broader scope incl. NAS/tablets): also behind an Anubis challenge now, MediaWiki API unreachable to scripts, no official dumps — the practical route to its content is the devicecode-data pre-parsed JSON. Neither is worth building pipeline against; use wi-cat + ToH + devicecode-data instead.

Sources:

- <https://dd-wrt.com/support/router-database/>
- <https://techinfodepot.shoutwiki.com/wiki/Main_Page>

### Switches: no dedicated structured community DB exists — coverage is ToH (88 realtek-target units) + wi-cat WikiDevi + vendor scraping

*Confidence: `verified-live`*

Searches found no standalone structured switch-spec database (managed/PoE/SFP+). What exists: (1) OpenWrt toh.json has devicetype='Switch' for 88 entries (D-Link DGS-1210 family, Zyxel GS1900, Netgear, TP-Link SG2xxx, Panasonic, ALLNET etc. — the realtek/rtl838x target) with ethernet*ports, sfp_ports, sfp_plus_ports and switch-chip columns; (2) wikidevi.wi-cat.ru actively documents switches including current Ubiquiti USW models (edits 2026-07-01) with chipset-level detail queryable via the ask API; (3) everything else is vendor datasheets. PoE budgets and SFP+ cage counts for the long tail (Netgear/TP-Link/Zyxel/QNAP) live only in per-product datasheet PDFs and spec-tab HTML — Ivan will need per-vendor scrapers, same as his motherboard vendor-page plan. SmallNetBuilder's router/NAS charts still exist (search snippets show activity through May 2026) but the site 403s datacenter IPs and its data is performance benchmarks, not specs, under proprietary copyright.

Sources:

- <https://openwrt.org/toh.json>
- <https://wikidevi.wi-cat.ru/Ubiquiti_Networks_USW_Pro_XG_8_PoE>
- <https://www.smallnetbuilder.com/>

### MikroTik: no public product API, but product pages are fully server-rendered with a clean key-value spec block — trivially scrapeable (verified)

*Confidence: `verified-live`*

mikrotik.com is a Laravel/Livewire app; the /products catalog filter runs over Livewire POSTs (no clean JSON endpoint; /products.json and /api/products both 404). But individual product pages are static server-rendered HTML: verified on https://mikrotik.com/product/CRS326-24G-2SplusRM, which contains a 'Specifications' section as label/value pairs (Product code, Suggested price $209.00, Architecture ARM 32bit, CPU 98DX3236, core count, nominal frequency, Switch chip model, RAM 512 MB, Storage 16 MB FLASH, MTBF, RouterOS license level, OS) plus unique-to-MikroTik L1/L2 throughput test tables (kpps/Mbps at 64/512/1518-byte frames) and a generated PDF datasheet per product. URL slugs derive from product codes ('+' encoded as 'plus'). Content is ordinary proprietary marketing copy — extract facts, don't republish text. This makes MikroTik the easiest switch/router vendor to ingest; enumerate products via the sitemap or the products page HTML.

Sources:

- <https://mikrotik.com/product/CRS326-24G-2SplusRM>
- <https://mikrotik.com/products>

### Ubiquiti: live public device-fingerprint JSON with 678 devices — good canonical product list + images, not full specs

*Confidence: `verified-live`*

https://static.ui.com/fingerprint/ui/public.json verified live (725 KB): 678 devices, each with deviceType, product line, product name, sku, sysid(s), GUIDs, shortnames, icon/image IDs, uisp flag. This is the identification DB the UniFi/UISP apps use — excellent for a canonical Ubiquiti SKU list, naming, and imagery, but it carries no port/PoE/radio specs; those are in ui.com store pages and per-line datasheet PDFs (dl.ui.com/datasheets/...). No license attached — it's vendor data served publicly; treat as facts-extraction only. Pairs well with wi-cat's chipset-level Ubiquiti pages.

Sources:

- <https://static.ui.com/fingerprint/ui/public.json>

### NAS/mini-PC/community sources: thin — devicecode-data and ToH are the only machine-readable options; NASCompares/STH are editorial

*Confidence: `reported`*

toh.json itself covers 52 NAS and 417 SBC entries (OpenWrt-supported subset only). devicecode-data includes TechInfoDepot's NAS coverage in parsed JSON. NASCompares publishes comparisons as articles/videos with no exportable dataset found; ServeTheHome forums (TinyMiniMicro etc.) are knowledge threads, not datasets; SmallNetBuilder NAS charts are benchmark-oriented and bot-walled. Practical takeaway for Ivan: for routers/APs, layer toh.json (nightly, CC BY-SA 4.0) + wi-cat ask API (CC BY-SA 3.0) + devicecode-data (baseline) keyed on FCC ID + brand/model; for switches, ToH+wi-cat+MikroTik/Ubiquiti scrapers. Licensing gotcha for the public/private split: unlike BuildCores' ODC-By (attribution-only), both wiki sources are share-alike — if his published catalog is a derivative of substantial ToH/WikiDevi content, that layer should be CC BY-SA-compatible; individual uncopyrightable facts re-verified against vendor/FCC primary sources sidestep this, and the FCC PDFs (manuals, internal photos) belong strictly in his private ArchiveBox tier.

Sources:

- <https://github.com/armijnhemel/devicecode-data>
- <https://nascompares.com/>
- <https://forums.servethehome.com/>
