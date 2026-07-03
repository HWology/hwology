# Data sources — verified access status

*All entries verified live 2026-07-02 unless noted. Access details are perishable — re-verify
before building on them if this file has aged. Full evidence:
[../research/2026-07-02/](../research/2026-07-02/README.md).*

## Motherboards

- **BuildCores OpenDB** (github.com/buildcores/buildcores-open-db) — the seed dataset.
  ODC-By 1.0, auto-syncs every 6h, 3,693 motherboard JSONs with per-slot PCIe gen/lanes,
  M.2 key/interface, onboard-Ethernet speed+controller enums. Gotchas: video-out is free text
  inside `back_panel_ports`; pre-DDR4 records have LLM-extraction errors (a G41 board tagged
  PCIe 4.0). README solicits BIOS-history data — upstream contribution opportunity.
- **ASUS undocumented JSON API** —
  `https://www.asus.com/support/api/product.asmx/GetPDBIOS?website=us&model=<name>` returns
  full BIOS history (version, date, changelog w/ AGESA, SHA256, download URL), no auth, works
  from datacenter IPs. Fragile: needs model-name enumeration + contract tests; Wayback CDX
  holds archived responses for backfill.
- **Open Icecat** (free tier): covers ASUS + Gigabyte sponsor brands (not MSI/ASRock/
  Supermicro); best for GTIN/EAN identity resolution.
- **Scrape access**: MSI/Gigabyte/Supermicro Akamai-block datacenter IPs; **ASRock went
  behind Imperva Incapsula as of 2026-07-02** (was open shortly before). Home/residential IP
  needed for all but ASUS.
- **linuxhw/DMI** (CC-BY-4.0, 162k dmidecode dumps) — field-observed BIOS versions and
  as-shipped hardware (NIC/WiFi component swaps within one revision are real).
- Dead ends: PCPartPicker (no API, ToS), TechPowerUp (prohibits scraping, no mobo DB),
  Newegg (seller-only API), Geizhals (Cloudflare + gated partner API — but its facet taxonomy
  is the schema gold standard to mirror).
- ASUS ToS §1.8.4 prohibits scraping; publish extracted facts only, never raw pages.

## Networking gear

- **OpenWrt ToH**: `https://openwrt.org/toh.json` — nightly, 2,996 devices × 74 columns
  (CPU, flash/RAM, per-speed ethernet ports, SFP/SFP+, wifi chips/drivers, FCC ID, support
  status), CC BY-SA 4.0. Includes 88 managed switches. The `fcc.io` shortener its FCC links
  use is dead. Wiki HTML bot-blocks; the JSON export is open.
- **wikidevi.wi-cat.ru**: live WikiDevi successor, actively edited, Semantic MediaWiki ask
  API (typed chipset/flash/RAM/FCC facts), CC BY-SA 3.0. Original WikiDevi 2019 dumps on
  archive.org.
- **armijnhemel/devicecode-data** (GitHub): pre-parsed JSON aggregate of TechInfoDepot +
  WikiDevi + OpenWrt + FCC with per-source license tracking; last updated ~2025-05; closest prior
  art to this project — study its schema unification.
- **fcc.report**: plain-curl scrapeable per-FCC-ID pages (internal photos, manual PDFs).
  fccid.io bot-blocks; apps.fcc.gov API Akamai-blocks datacenter IPs.
- **MikroTik**: server-rendered key-value spec blocks per product page + L1/L2 throughput
  tables — easiest vendor to ingest. **Ubiquiti**: `static.ui.com/fingerprint/ui/public.json`
  (678 devices, SKU/naming/images, no port/PoE specs).
- Skip: DD-WRT router DB + TechInfoDepot (Anubis-walled; DD-WRT DB community-flagged stale).
- Switches have **no dedicated community DB anywhere** — unclaimed niche.

## Firmware/BIOS notes

- Vendors silently pull BIOS releases (betas, bricked versions) — capture is perishable,
  append-only model required, `pulled` is a status not a deletion.
- soggi.org archives MSI BIOSes incl. pulled betas. Reous spreadsheet tracks AMD AGESA;
  no Intel-side equivalent found — parse ME versions from changelogs for parity.
- BIOS binaries are copyrighted (AGESA/Intel reference code inside): archive privately,
  publish metadata + sha256 + vendor link only.

## Stack facts (verified 2026-07-02)

- **.NET 11 STJ F# DU support**: PR dotnet/runtime#125610 (closes #55744) is genuinely for
  F# DUs — NOT the separate C# 15 `union` preview feature (own STJ path, PR #128162).
  Shipped Preview 4, on by default, reflection-only (no source-gen/AOT). Fieldless case →
  string; fields → `{"$type":"Case",...}`; **single-case DUs not unwrapped**; options unwrap.
- **SQLite**: Microsoft.Data.Sqlite 10.x + explicit SQLitePCLRaw.bundle_e_sqlite3 ≥ 3.0.3
  (2.1.11 native lib deprecated w/ high-sev vuln). Dapper.FSharp active; skip
  Marten/LiteDB/EF Core. SQLite async is fake — WAL + serialized writes.
- **TOML**: spec 1.1.0 released 2025-12-24. Tomlyn 2.9 (June 2026, STJ-style, TOML 1.1-only)
  live-tested with F# records: needs `[<CLIMutable>]`, arrays not F# lists, ~10-line
  converter per option type; DUs manual. taplo orphaned → **tombi** for schema validation.
  Prior art for TOML datasets: RustSec advisory-db, mozilla cargo-vet.
- **Porkbun API** (registrar + DNS host for hwology.com): v3.7 verified live 2026-07-02.
  Full DNS CRUD (by record id and by name+type, dryRun on writes), NS/glue/DNSSEC/URL-forward
  management, domain register/renew/transfer via account credit (cost-in-pennies confirm,
  dryRun, spend controls), HMAC-signed webhooks, Let's Encrypt bundle retrieval. Base
  `https://api.porkbun.com/api/json/v3` (hostname moved off porkbun.com in 2025 — pre-2025
  clients broke); OpenAPI 3.0 spec at `porkbun.com/api/json/v3/spec`, markdown reference at
  `porkbun.com/llms-full.txt` — **docs bot-block datacenter IPs** (one agent got 403s).
  Gotchas: per-domain "API Access" toggle, off by default, dashboard-only; account-wide
  key+secret (per-key IP/domain allowlists exist, no read-only keys); min TTL 600 s;
  ~200 records/zone; no zone-file export or bulk endpoints; documented 429 rate limits but
  serialize + back off anyway. DNS resolves on Cloudflare anycast (Porkbun-branded NS);
  set CAA + watch CT logs — two independent reports (2024, 2026) of Cloudflare-side cert
  issuance for Porkbun-hosted zones. No real .NET client on NuGet — plain JSON-over-HTTPS,
  call from F# directly or codegen from the OpenAPI spec. ACME/IaC tooling mature:
  acme.sh/lego/Caddy/certbot, Terraform `marcfrederick/porkbun` (active; cullenmcdermott
  archived), DNSControl, octoDNS, ddns-updater, official MCP server (`@porkbunllc/mcp-server`).
  DNSSEC (KB 216, reviewed 2026-07-02): managed signing ("Porkbun DNSSEC", Cloudflare-backed)
  is a dashboard-only toggle — Domain Management → Details → "Porkbun DNSSEC" (red→green);
  Porkbun-NS domains only; if it won't activate, delete pre-existing registry DNSSEC records
  first; verify at dnsviz.net. The API's `/dns/{get,create,delete}DnssecRecord` endpoints
  manage raw registry DS/key data only (the "Registry DNSSEC" manual path for external signed
  NS, e.g. deSEC) — they do NOT trigger zone signing. KB documents no disable/migration
  procedure: before ever moving NS, delete the DS and wait out TTLs. hwology.com unsigned as
  of 2026-07-02 (no DS, no DNSKEY, API records null).
- **Archiving**: browsertrix-crawler v1.13.x (WARC/WACZ, browser profiles beat bot walls);
  single-file-cli v2 attaches to a running browser via CDP; Wayback SPN2
  (`POST web.archive.org/save`, `Authorization: LOW key:secret`, `if_not_archived_within`)
  as public witness — IA fetches the URL itself, so vendor bot-walls can block it too.
