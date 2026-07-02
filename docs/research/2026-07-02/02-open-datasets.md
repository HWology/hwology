# Existing open datasets (BuildCores OpenDB etc.)

*Research date: 2026-07-02, parallel web-research agents. Findings marked `verified-live` were confirmed against the live web that day; `reported` means sourced but not directly fetched; treat access details as perishable.*

## Summary

BuildCores OpenDB is the standout and should be Ivan's seed dataset: verified alive today (automated sync commits every 6 hours, last push 2026-07-01), ODC-By 1.0 licensed, with 3,693 motherboard JSON records carrying exactly the slot-level detail he needs — per-slot PCIe gen/lanes/quantity, per-M.2-slot size/key/interface, onboard Ethernet speed+controller (2.5GbE filterable via enum), form factor, and back-panel ports including video outputs. Its known weaknesses map precisely to Ivan's differentiator: BIOS version history and CPU support lists are explicitly on BuildCores' "help wanted" list, and I found no other open project anywhere that scrapes vendor BIOS release history — that gap is real and unclaimed. The only other live sources are docyx/pc-part-dataset (a PCPartPicker scrape, last updated July 2025, but motherboard records have just 7 shallow fields — reference-only) and linux-hardware.org's linuxhw GitHub dumps (CC-BY-4.0, 162k dmidecode reports containing field-observed BIOS versions/dates per board — a useful corroboration source, not a spec sheet). Every other GitHub scraper/database project I checked is dead (2018–2021), and Kaggle/HuggingFace offerings are shallow repackages or image/review datasets not worth seeding from. One data-quality caveat on BuildCores: I found LLM-extraction artifacts on older boards (an LGA775/G41 board listed with PCIe 4.0 slots), so pre-DDR4-era records need validation before trusting filters.

## Findings

### BuildCores OpenDB: alive, licensed ODC-By 1.0, best available seed data

*Confidence: `verified-live`*

github.com/buildcores/buildcores-open-db. Created 2025-04, last push 2026-07-01 (yesterday) via '[skip ci] Automated opendb sync' commits every ~6 hours plus human PRs (PR #243 merged 2026-06-29). 320 stars, 150 forks, 117 open issues, 1,935 commits. License: Open Data Commons Attribution (ODC-By) v1.0 — free to use/modify/build on with attribution; note their README says they cannot include price or retailer-specific data. 30 categories, ~42,000 UUID-named JSON files under /open-db/ (RAM 4,478; Mouse 4,423; Keyboard 4,036; Monitor 3,848; GPU 3,832; PCCase 3,725; Motherboard 3,693; Storage 3,496; PSU 3,286; CPU 788; etc.). Per-category JSON Schemas in /schemas/ with CI validation on PRs; merged changes auto-sync to their API. GitHub git-trees API returns the full file list untruncated, so a clone + local import into SQLite is trivial.

Sources:

- <https://github.com/buildcores/buildcores-open-db>
- <https://github.com/buildcores/buildcores-open-db/blob/main/README.md>

### BuildCores Motherboard schema has the slot-level detail Ivan's filters need

*Confidence: `verified-live`*

Motherboard.schema.json (~17KB) fields: socket, chipset, form_factor; pcie_slots as array of {gen enum 1.1-5.0, lanes, quantity} — directly supports '3+ PCIe slots' filtering; m2_slots as array of {size, key enum M/E/B/B+M, interface e.g. 'PCIe 4.0 x4'}; onboard_ethernet as array of {speed enum 10/5/2.5/1 Gb/s/100Mb, controller model} — '2.5GbE' is a clean enum filter; memory {max GB, ram_type enum, slots}; usb_headers, fan_headers, rgb_headers, front_panel_headers, power_connectors, ecc_support, raid_support, back_connect_connectors, audio, wireless_networking, bios_features {flashback, clear_cmos}. Caveat: onboard video out is NOT a structured field — it only appears inside back_panel_ports, an array of free-text strings like '1 x HDMI port', so Ivan would need to parse those strings (or infer from chipset/socket) to build an 'onboard video out' facet. Verified sample record ASUS B760M-AYW WIFI D4 was complete and accurate.

Sources:

- <https://github.com/buildcores/buildcores-open-db/blob/main/schemas/Motherboard.schema.json>

### BuildCores data-quality gotcha: extraction errors on older boards

*Confidence: `verified-live`*

Sample record open-db/Motherboard/0013d0d2-024d-4153-92a7-9beec572b52b.json (Gigabyte GA-G41MT-S2PT, LGA775/Intel G41, DDR3-era) lists pcie_slots as gen '4.0' — impossible for that platform (G41 is PCIe 1.x) — and its back_panel_ports contains only the Ethernet entry, missing VGA/DVI/USB. Records carry Mongo-style '_id' fields suggesting automated (likely LLM) extraction pipelines. Recommendation if seeding: import everything, but run sanity rules (socket-era vs PCIe-gen consistency, empty back_panel_ports) and prioritize manual verification for pre-2015 boards; modern boards (the ASUS B760 sample) looked accurate.

Sources:

- <https://raw.githubusercontent.com/buildcores/buildcores-open-db/main/open-db/Motherboard/0013d0d2-024d-4153-92a7-9beec572b52b.json>

### BIOS/firmware version history: BuildCores doesn't have it, and nobody else does either — Ivan's differentiator is a confirmed gap

*Confidence: `verified-live`*

BuildCores' README 'Help Wanted / Bounties' section explicitly lists as unmet goals: collecting manufacturer product-page URLs, product PDFs (motherboard manuals), and 'motherboard BIOS versioning data along with CPU support lists'. GitHub searches for BIOS-history scrapers surfaced only BIOS modding/extraction tools (BIOSUtilities, 86Box/bios-tools, dreamwhite/bios-extraction-guide) — no project scrapes ASUS/MSI/Gigabyte/ASRock BIOS release pages into a dataset. This means (a) Ivan must build the BIOS-history scraper himself against vendor support pages, and (b) if he does, BuildCores is actively soliciting exactly that data, so contributing it upstream is an option.

Sources:

- <https://github.com/buildcores/buildcores-open-db/blob/main/README.md>

### docyx/pc-part-dataset (PCPartPicker scrape): fresh-ish but motherboard data too shallow to seed from

*Confidence: `verified-live`*

github.com/docyx/pc-part-dataset. MIT-licensed (code; the data itself is scraped from PCPartPicker, which is a ToS gray area — treat as reference, not something to redistribute). Last push 2025-07-23, 296 stars. 66,778 parts total in JSON/JSONL/CSV; data/json/motherboard.json has 4,973 boards but only 7 fields each: name, price, socket, form_factor, max_memory, memory_slots, color. It scrapes PCPP's list view, not detail pages — no PCIe slots, no LAN, no M.2, no video out. Useful only as a cross-check for model-name coverage (it has ~1,300 more board names than BuildCores) and a stale US price snapshot. The included scraper needs a VPN and ~5-10 min per run if Ivan wanted to refresh it.

Sources:

- <https://github.com/docyx/pc-part-dataset>

### linux-hardware.org / linuxhw GitHub dumps: CC-BY-4.0 field data, best open source of observed BIOS versions

*Confidence: `verified-live`*

linux-hardware.org reports 352,511 tested computers / 597,002 tested parts (verified on homepage today). The hw-probe tool repo (github.com/linuxhw/hw-probe, 914 stars) was last pushed 2025-05-03, but the data-dump repos in the linuxhw org are actively updated (TestDays and Trends pushed 2026-01; DMI, SMART, EDID, LsPCI pushed 2025-01/2026-01) and are licensed CC-BY-4.0. The key one for Ivan: github.com/linuxhw/DMI — 162,480 raw dmidecode reports organized as {Type}/{Vendor}/{Model}/{HWID} (e.g. Desktop/MSI/MS-7/MS-7C02/...). Each report contains motherboard vendor/model plus BIOS vendor, version, and release date, and dmidecode's DMI type-9 records also list physical system slots. So for any board Linux users own, you can extract a crowd-sourced timeline of BIOS versions actually deployed — corroboration for a BIOS-history feature, though not authoritative (vendor download pages remain the source of truth; coverage skews to Linux-popular boards; data is raw text needing parsing). Reference/enrichment source, not primary seed.

Sources:

- <https://linux-hardware.org/>
- <https://github.com/linuxhw/DMI>
- <https://github.com/linuxhw/hw-probe>

### Other GitHub scraper/database projects: effectively all dead

*Confidence: `verified-live`*

Checked via GitHub API: JonathanVusich/pcpartpicker-scraper (last push 2021-07, 12 stars, no license), nynhex/PCPartPicker-API (2018-11, MIT), niamtokik/friendly-router (router/firewall board DB in CSV/SQLite/JSON — conceptually adjacent to Ivan's ITX/2.5GbE use case but last push 2020-09, 2 stars). A GitHub search for recently-updated 'pc parts database' repos returns only student e-commerce schemas and toy projects. felixsteinke/cpu-spec-dataset covers CPUs only. Antmicro's Hardware Component Database targets embedded/electronic components, not consumer motherboards. Conclusion: BuildCores has consolidated this niche; no second maintained option exists.

Sources:

- <https://github.com/JonathanVusich/pcpartpicker-scraper>
- <https://github.com/nynhex/PCPartPicker-API>
- <https://github.com/niamtokik/friendly-router>

### Kaggle / HuggingFace: nothing worth seeding from

*Confidence: `reported`*

Candidates found: kaggle.com/datasets/warcoder/pc-parts (appears to be a repackage of the docyx PCPartPicker scrape; Kaggle page content couldn't be fetched directly to confirm update date/license), sudhanshuy17/pccomponents (price-focused), iliassekkaf/computerparts (CPUs/GPUs only, years old), asaniczka/pc-parts-images-dataset-classification (images for ML classification, no specs), and huggingface.co/datasets/argilla/pc-components-reviews (review text for NLP, not specs). None offers motherboard spec depth beyond what BuildCores already provides with better licensing and freshness. Recommendation: skip this tier entirely; seed from BuildCores, enrich from linuxhw/DMI and vendor pages.

Sources:

- <https://www.kaggle.com/datasets/warcoder/pc-parts>
- <https://huggingface.co/datasets/argilla/pc-components-reviews>

### Practical seeding recommendation

*Confidence: `verified-live`*

Seed pipeline: (1) git clone buildcores/buildcores-open-db (~97MB), import the 3,693 Motherboard JSONs into SQLite keyed on opendb_id, keeping their JSON Schemas as import validators — the schema's pcie_slots/m2_slots/onboard_ethernet structures map almost 1:1 to Ivan's target facets; (2) add a derived 'video_out' facet by parsing back_panel_ports strings (HDMI/DisplayPort/DVI/VGA/USB-C DP-alt); (3) add sanity-check rules to flag suspect records (PCIe gen vs socket era); (4) build the BIOS-history layer himself from vendor support pages, optionally cross-referencing linuxhw/DMI for observed-in-the-wild versions; (5) periodically 'git pull' upstream — ODC-By only requires attribution. The metadata.variant/series structure also gives him variant grouping for free, and the same repo covers 29 other categories for when he generalizes beyond motherboards.

Sources:

- <https://github.com/buildcores/buildcores-open-db>
