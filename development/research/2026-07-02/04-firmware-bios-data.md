# Firmware / BIOS data sources

*Research date: 2026-07-02, parallel web-research agents. Findings marked `verified-live` were confirmed against the live web that day; `reported` means sourced but not directly fetched; treat access details as perishable.*

> **Note:** an adversarial fact-check pass ran after this research — see [06-claim-verification.md](06-claim-verification.md). Two claims were corrected: ASRock is now Incapsula-blocked from datacenter IPs (was reported open), and pgloader does **not** auto-convert TEXT→JSONB (store JSON as plain TEXT in SQLite, add an explicit CAST rule).

## Summary

Firmware/BIOS data is very gettable, but access difficulty splits cleanly by vendor. ASUS has a clean undocumented JSON API (verified live: full version/date/changelog/SHA256/download-URL history, 32 releases for one AM5 board) and ASRock serves plain static HTML tables plus a sitewide "Latest BIOS Update" feed that was current to 2026-07-02. MSI and Gigabyte hard-block non-browser clients via Akamai (403 on both HTML pages and MSI's known JSON API from datacenter IPs) — they need a real browser session on a non-datacenter connection. Changelogs reliably encode AGESA versions on AMD boards ("Update AGESA to ComboAM5 1.3.0.1b Patch A") and even Intel microcode revisions on ASRock ("Update CPU microcode to 0x133"), and the community Reous AM5 Google Sheet cross-referencing board×BIOS×AGESA×SMU exports as CSV. LVFS has no REST API but its AppStream XML metadata is one download — however, I parsed it live: zero consumer desktop motherboards from ASUS/ASRock/Gigabyte/MSI (their few LVFS entries are NVIDIA GX10/DGX-Spark-class mini-PCs), so LVFS is irrelevant for retail boards but a good schema model. CPU support lists ("since BIOS X") are structured HTML at ASRock and API-backed at ASUS; MSI/Gigabyte equivalents sit behind the same bot wall.

## Findings

### ASUS: undocumented JSON API with full BIOS history — best-in-class source

*Confidence: `verified-live`*

GET https://www.asus.com/support/api/product.asmx/GetPDBIOS?website=us&model=<MODEL-NAME> returns JSON: Result.Obj[] with Version, ReleaseDate (YYYY/MM/DD), Description (full changelog text, e.g. 'Update AGESA ComboAM5 PI 1.3.0.1...'), FileSize, SHA256, DownloadUrl.Global. Verified 32 releases spanning Apr 2023–Apr 2026 for ROG STRIX B650E-I. Works with plain HTTP from a datacenter IP, no auth. A ROG-site variant exists (rog.asus.com/support/webapi/product/GetPDBIOS with pdid/systemCode params, per g-helper GitHub issue #362). CPU QVL endpoint GetPDQVL exists and returns SUCCESS but needs additional params (likely pdid/hashedid) to return rows — reverse-engineer from the product 'helpdesk_cpu' page. Reliability: excellent; this API is what ASUS's own support pages consume.

Sources:

- <https://www.asus.com/support/api/product.asmx/GetPDBIOS?website=us&model=ROG%20STRIX%20B650E-I%20GAMING%20WIFI>
- <https://github.com/seerge/g-helper/issues/362>

### ASRock: static HTML tables + a sitewide same-day BIOS feed — easiest to scrape

*Confidence: `verified-live`*

Per-board BIOS page at https://www.asrock.com/mb/amd/<Model>/bios.html (note: old bios.asp URLs 404) is a plain HTML table: Version | Date | Size | Update method | Description | Global+China download + SHA256. Descriptions cite AGESA ('4.41 | 2026/6/23 | Update AGESA to ComboAM5 1.3.0.1b'). Crucially, https://www.asrock.com/support/index.asp?cat=BIOS is a sitewide recent-BIOS feed (Model/Version/Date/Description) that contained an entry dated 2026-07-02 (same day as the check), making it ideal for change-detection polling instead of crawling every board. Intel-side entries even cite microcode revs ('Z790 Lightning WiFi 13.02: Update CPU microcode to 0x133. Update Secure Boot Key'). No bot-blocking observed from a datacenter IP.

Sources:

- <https://www.asrock.com/mb/amd/B650M%20Pro%20RS/bios.html>
- <https://www.asrock.com/support/index.asp?cat=BIOS>

### MSI and Gigabyte: data exists but sits behind Akamai bot protection — plan for headless browser

*Confidence: `verified-live`*

MSI's JSON API pattern (www.msi.com/api/v1/product/support/panel?product=<slug>&type=bios) and its HTML support pages, plus Gigabyte's support pages (gigabyte.com/Motherboard/<model>/support), ALL returned 403 Access Denied (Akamai edgesuite reference IDs) from both curl-with-browser-UA and Anthropic's fetch infrastructure. This is IP/JS-challenge based, not UA-based. Both sites work fine in real browsers, so access requires a real browser session on a non-datacenter connection; in such a session, MSI's api/v1 panel endpoint returns JSON and Gigabyte's support pages carry BIOS version/date/description lists. Budget for this being the flakiest part of the pipeline; treat MSI/Gigabyte freshness as best-effort and consider the Reous sheet (below) as a cross-check. Third-party archive msi-bios.com also bot-blocked (HTTP 466) from this host.

Sources:

- <https://www.msi.com/api/v1/product/support/panel?product=MAG-B650-TOMAHAWK-WIFI&type=bios>
- <https://www.gigabyte.com/Motherboard/B650M-AORUS-ELITE-AX-rev-10/support>

### AGESA tracking: vendor changelogs are reliable carriers; Reous Google Sheet is the community source of truth

*Confidence: `verified-live`*

AMD publishes no official AGESA changelog/database — AGESA versions surface only through board-vendor BIOS descriptions, and those are reliably present: verified ASUS ('Update AGESA ComboAM5 PI 1.3.0.1') and ASRock ('Update AGESA to ComboAM5 1.3.0.1b Patch A') examples; parse with a regex over Description fields (patterns: ComboAM5[ ]?PI x.y.z.w[letter][ PatchA], ComboV2PI for AM4). The community tracker is Reous's 'AM5 AGESA/UEFI/BIOS List | SMU AM5/AM4 Table' Google Sheet (maintained by hardwareluxx user Reous Innox): a board×BIOS-version grid per vendor/chipset with AGESA and SMU version columns, including beta/test BIOSes vendors later pull. VERIFIED: exports via the standard Google Sheets CSV endpoint (export?format=csv&gid=937453961), current through ComboAM5Pi 1.3.0.1b PatchA. Layout is a human-oriented pivot grid (messy multi-header CSV) — parseable but brittle; treat as enrichment/cross-check, not primary. Companion tool: github.com/ReousI/AM5-SMU-Checker. Related hardwareluxx thread hosts discussion/updates.

Sources:

- <https://docs.google.com/spreadsheets/d/12zg6yT_H7H-W1voyw1ZoIrj0GSE7WI4Ug-uLlv-Asa8/edit>
- <https://github.com/ReousI/AM5-SMU-Checker>
- <https://www.hardwareluxx.de/community/threads/am5-agesa-uefi-bios-info-laberthread.1323294/>

### Intel microcode: two excellent structured GitHub sources

*Confidence: `verified-live`*

(1) intel/Intel-Linux-Processor-Microcode-Data-Files — official Intel repo; releases verified live through microcode-20260512, each with markdown release notes containing security advisories (INTEL-SA-xxxxx) and an 'Updated Platforms' table (processor family, CPUID, old→new microcode revision). Consumable via GitHub API (gh api repos/.../releases) — fully structured, authoritative for the microcode dimension of Intel boards. (2) platomav/CPUMicrocodes — community repo of Intel/AMD/VIA/Freescale microcode binaries named with CPUID+revision+date; verified actively maintained (last update 2026-06-30, 1071 stars). Caveat: microcode revision inside a given vendor BIOS is NOT usually stated in changelogs (ASRock is the notable exception, e.g. '0x133'); mapping BIOS→microcode generally requires extracting the BIOS image (UBU/MCExtractor tooling from the Win-Raid community).

Sources:

- <https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files>
- <https://github.com/platomav/CPUMicrocodes>

### LVFS/fwupd: one-file machine-readable metadata, but ZERO consumer desktop motherboards in 2026

*Confidence: `verified-live`*

No REST/JSON query API (docs at lvfs.readthedocs.io are vendor-upload oriented; the /lvfs/search endpoint even 403s non-browser clients). The canonical machine-readable artifact is the AppStream XML metadata: https://cdn.fwupd.org/downloads/firmware.xml.gz (also .zst/.xz) — verified: 1.86 MB, 799 components, one download refreshes everything. Live parse of vendor breakdown: Lenovo 310, Dell 233, HP 87, Intel 76, Star Labs 17, Framework 8 ... vs Gigabyte 4, MSI 3, Asus 2, ASRock Industrial 1. Component-level inspection shows those Gigabyte/MSI/Asus entries are ALL NVIDIA GB10/GX10 'DGX Spark'-class mini-PC firmware (ASUS GX10, Gigabyte AITOP ATOM, MSI MS_C931) and one ASRock Industrial NUC — not one retail desktop motherboard. Verdict: LVFS is not a data source for Ivan's catalog, but its metainfo schema (GUIDs, <release> with version/date/description) is a good model for his own firmware table, and it IS the right source if he later covers Framework/System76/Star Labs-style machines.

Sources:

- <https://cdn.fwupd.org/downloads/firmware.xml.gz>
- <https://fwupd.org/lvfs/vendors/>
- <https://lvfs.readthedocs.io/en/latest/>

### CPU support lists ('since BIOS X'): structured at ASRock, API-backed at ASUS, browser-gated at MSI/Gigabyte

*Confidence: `verified-live`*

ASRock: HTML tables at asrock.com/support/cpu.asp?s=<socket>&u=<cpu-id> with columns Family/Model(+OPN)/Core/Frequency/Cache/CPU Rev/Power and a 'Support BIOS' column giving the minimum BIOS per board — verified live (e.g. Ryzen 7 8700G: X870E Taichi since BIOS 3.04). Note it's organized per-CPU-across-boards; per-board views exist via product pages. Old per-board cpu.asp/cpu.html URLs 404 — enumerate via the s= socket parameter and u= ids. ASUS: CPU QVL served by the same support API family (GetPDQVL returns valid JSON envelope; needs extra params from the product helpdesk_cpu page). MSI ('Compatibility' tab) and Gigabyte ('Support CPU' list) publish equivalent since-BIOS tables but sit behind the same Akamai wall — headless browser required. Reliability of the data itself is high across all four vendors (these lists are legally load-bearing for RMA), though beta-BIOS rows sometimes appear/disappear.

Sources:

- <https://www.asrock.com/support/cpu.asp?s=AM5&u=746>

### Community/coreboot sources: Win-Raid lives at Level1Techs; coreboot board support is docs-generated; no good third-party BIOS-version DB exists

*Confidence: `verified-live`*

Win-Raid forum (the BIOS-modding community behind UBU, MCExtractor, MMTool guides) migrated to Level1Techs hosting — verified live at winraid.level1techs.com (HTTP 200). It's unstructured forum content: valuable for methodology (extracting AGESA/microcode/RAID-ROM versions from BIOS images) and for archived/pulled BIOSes, not as a scrapeable feed. coreboot: the supported-mainboards list is auto-generated documentation at doc.coreboot.org/mainboard/ (verified 200; source of truth is per-board directories in the coreboot git tree — cleanly parseable from the repo). The legacy board-status reporting system is in flux: coreboot.org/status/board-status.html 404s and the Jan-2025 leadership meeting discussed replacing it (JSON format proposed); treat board-status as historical. Notable gap = opportunity: there is NO existing structured cross-vendor BIOS-version database (msi-bios.com is a bot-blocked unofficial archive; Reous covers AMD only). Ivan's changelog-per-board table would be genuinely novel. Suggested architecture from these findings: daily poll of ASUS API + ASRock sitewide feed (cheap, reliable), weekly Playwright run for MSI/Gigabyte, Reous CSV + Intel microcode GitHub as enrichment layers, and store raw changelog text + parsed AGESA/microcode fields separately.

Sources:

- <https://winraid.level1techs.com/>
- <https://doc.coreboot.org/mainboard/index.html>
- <https://github.com/coreboot/coreboot>
