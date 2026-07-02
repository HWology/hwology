# Motherboard spec data sources

*Research date: 2026-07-02, parallel web-research agents. Findings marked `verified-live` were confirmed against the live web that day; `reported` means sourced but not directly fetched; treat access details as perishable.*

> **Note:** an adversarial fact-check pass ran after this research — see [06-claim-verification.md](06-claim-verification.md). Two claims were corrected: ASRock is now Incapsula-blocked from datacenter IPs (was reported open), and pgloader does **not** auto-convert TEXT→JSONB (store JSON as plain TEXT in SQLite, add an explicit CAST rule).

## Summary

Structured motherboard spec data in 2026 comes from four tiers, and no single source is sufficient. Manufacturer sites split cleanly: ASUS and ASRock serve fully server-rendered, parseable spec HTML with permissive robots.txt (ASUS even exposes a live JSON webapi returning complete BIOS version history — directly relevant to the firmware-history feature), while MSI, Gigabyte, and Supermicro sit behind Akamai bot management that 403s datacenter IPs on every path including robots.txt, so they require residential-IP polite crawling. Among aggregators, Geizhals has the best facet taxonomy in existence for this use case but is Cloudflare-hardened with only an application-gated publisher API (price-oriented); Icecat's Open (free) tier is the only legitimately licensed bulk spec source with JSON/XML APIs, covering ASUS and Gigabyte boards as sponsor brands, though datasheet depth is shallower than manufacturer pages; TechPowerUp explicitly prohibits scraping and has no motherboard DB anyway; PCPartPicker has no API and disallows automated collection. Retailer APIs are dead ends for a persistent catalog: Best Buy's Products API does return structured details name/value spec pairs but its ToS caps caching at 72 hours, and Amazon PA-API 5.0's ItemInfo confirms no deep technical spec structure. Practical pipeline: scrape ASUS/ASRock directly, MSI/Gigabyte/Supermicro from a home IP at low rate, use Open Icecat (free registration) for GTIN/EAN identity resolution and baseline datasheets, model your facet schema on Geizhals' taxonomy, and treat ETIM/ECLASS/GS1 as reference vocabularies rather than data sources.

## Findings

### ASUS: server-rendered spec tables + live JSON BIOS/driver API

*Confidence: `verified-live`*

Techspec pages (e.g. /us/motherboards-components/motherboards/prime/prime-b650m-a-ax/techspec/) returned HTTP 200 to plain curl from a datacenter IP; the full spec table is in static HTML (TechSpec__rowTableItems divs with label/value pairs like '4 x DIMM, DDR5 5600(OC)...'), plus a window.__NUXT__ JSON payload and JSON-LD. robots.txt only disallows search/filter URLs, not product/spec pages. Separately, the support webapi returns clean JSON: GetPDBIOS on rog.asus.com/support/webapi/product/ returned 14 BIOS releases with Version, Title, Description, file IDs when I called it — a ready-made firmware-history feed (same GetPDBIOS/GetPDrivers endpoints exist for the main asus.com support site with per-product pdid/hash params, documented in the g-helper community). Cost: free, but no formal API license — it's the site's own frontend API, so rate-limit politely.

Sources:

- <https://www.asus.com/us/motherboards-components/motherboards/prime/prime-b650m-a-ax/techspec/>
- <https://rog.asus.com/support/webapi/product/GetPDBIOS?website=global&model=GA402XY&pdid=22300&cpu=GA402XI&LevelTagId=161533&systemCode=rog>
- <https://github.com/seerge/g-helper/issues/362>
- <https://www.asus.com/robots.txt>

### ASRock: cleanest scrape target — static HTML specs, near-empty robots.txt

*Confidence: `verified-live`*

The old /Specification.asp URLs now 404; specs moved inline into each model's index.asp inside <section id="sSpecification"> as heading + <ul> bullet lists (verified on X870E Taichi White: sections for Unique Feature, CPU, Chipset, Memory, BIOS, Graphics, etc., ~10 KB of spec text per board). Site serves 200 to curl from a datacenter IP; robots.txt disallows only /asrockrack/. The /mb/index.asp listing page links every current model, giving you a free crawl frontier. BIOS pages are separate support pages, also plain HTML. Machine-readability is medium: consistent section labels but free-text bullets need parsing (e.g. '- Marvell (Aquantia) 10GbE LAN').

Sources:

- <https://www.asrock.com/mb/index.asp>
- <https://www.asrock.com/mb/AMD/X870E%20Taichi%20White/index.asp>
- <https://www.asrock.com/robots.txt>

### MSI, Gigabyte, Supermicro: Akamai bot manager blocks datacenter IPs outright

*Confidence: `verified-live`*

All three returned Akamai 'Access Denied' (errors.edgesuite.net reference) for every path tested — including robots.txt — from this environment, even with a full Chrome user-agent. This is IP-reputation blocking, not UA sniffing; community scrapers report the sites work from residential IPs. MSI's site is an SPA backed by JSON endpoints under www.msi.com/api/v1/product/ (getProductList, getProductDetail — the 403 came from Akamai, not a 404, confirming the path exists). Gigabyte spec pages (/Motherboard/<model>/sp) are server-rendered spec tables per community scrapers; Supermicro uses classic HTML spec tables. Practical takeaway: for a self-hosted single-dev tool, crawl these three from a home connection at low rate; cloud-hosted crawling is not viable for these vendors.

Sources:

- <https://www.msi.com/api/v1/product/getProductList>
- <https://www.gigabyte.com/Motherboard/B650M-DS3H-rev-10/sp>
- <https://www.supermicro.com/en/products/motherboard/X13SAE-F>

### Geizhals/Skinflint: best facet taxonomy, hardest access — partner API by application only

*Confidence: `verified-live`*

Category pages (geizhals.de/?cat=mbam5, skinflint.co.uk redirecting to geizhals.eu) returned 403 from datacenter IPs; Geizhals is a published Cloudflare bot-management customer, and robots.txt disallows all /hardware/ category paths. Official access: a publisher/partner API granted case-by-case after a quality review of your site (username/password credentials; documented via the affiliate-toolkit integration; covers AT/DE/PL/UK). It is affiliate/price-oriented — offers, products, EAN matching, shops — and it's unclear whether full spec sheets are exposed (their motherboard facets cover form factor, chipset, LAN speed/count, PCIe slot versions/counts, M.2 slots by key, video outputs, VRM, BIOS features — the exact gold standard for your filter UX, worth mirroring as a schema even if you never get the data). Third-party paid route: PriceAPI resells Geizhals data (offers, products, matchings, search, shops) without an approval process. A hobbyist personal-catalog site may not pass their publisher review; treat Geizhals as taxonomy inspiration first, data source second.

Sources:

- <https://geizhals.de/robots.txt>
- <https://www.affiliate-toolkit.com/kb/set-up-geizhals-interface/>
- <https://readme.priceapi.com/reference/geizhals-1>
- <https://www.cloudflare.com/case-studies/geizhals-at/>

### Icecat / Open Icecat: the only licensed free bulk source; covers ASUS+Gigabyte boards; JSON API

*Confidence: `verified-live`*

Open Icecat (free registration) gives JSON/XML/CSV access at live.icecat.biz/api with api_token + content_token, but only for ~450 'sponsoring brands'. ASUS (vendor ID 120) and Gigabyte (ID 480) ARE Open Icecat sponsors; MSI, ASRock, Supermicro are not (their data needs paid Full Icecat). JSON responses include GeneralInfo (brand, category, EAN/GTIN, family), Features grouped in FeatureGroups with RawValue/Value/PresentationValue — genuinely machine-readable and ideal for facet filtering. License: Open Content License — attribution + 'AS IS' disclaimer required, no ad-network use, fair use ~1M datasheet downloads/month then EUR 0.30/1000. Caveat verified on a live ASUS PRIME B560M-A datasheet: motherboard entries exist in quantity but can be shallow (chipset/socket/form factor solid; PCIe/M.2/LAN detail thinner than manufacturer pages) and 'created by Icecat' entries lag new launches. Best used for GTIN/identifier resolution and baseline data, enriched by manufacturer scrapes.

Sources:

- <https://iceclog.com/manual-for-icecat-json-product-requests/>
- <https://iceclog.com/open-content-license/>
- <https://guide.itscope.com/en/kb/open-icecat-sponsors/>
- <https://icecat.biz/en/p/asus/90mb17a0-m0eay0/motherboards-4711081130376-prime+b560m-a-89217358.html>
- <https://iceclog.com/open-icecat-fair-use-policy/>

### TechPowerUp: scraping explicitly prohibited, and no motherboard spec database anyway

*Confidence: `verified-live`*

robots.txt (fetched live) opens with a legal notice: content is personal non-commercial use only; automated data mining/scraping prohibited without written permission, explicitly reserving EU DSM Art. 4 TDM rights and banning AI/LLM use, dataset redistribution, and commercial purposes (contact w1zzard@techpowerup.com for licensing). ClaudeBot is specifically disallowed from /gpu-specs/, /cpu-specs/, /ssd-specs/. Their spec databases cover GPUs, CPUs, and SSDs — there is no motherboard spec DB, so it's largely irrelevant to your v1 anyway, but relevant if you extend to GPUs/CPUs: budget for asking permission or skip it.

Sources:

- <https://www.techpowerup.com/robots.txt>
- <https://www.techpowerup.com/gpu-specs/>

### PCPartPicker: no API, scraping against ToS, Cloudflare content signals

*Confidence: `verified-live`*

No official API exists. Live robots.txt: Cloudflare-managed block list denying ClaudeBot/GPTBot/CCBot etc. entirely; for all agents Content-Signal 'search=yes, ai-train=no, ai-input=no', Disallow /api/ and /search/, Crawl-delay 60. Staff have stated in their forums that scraping violates ToS. Unofficial scrapers exist (pypartpicker on PyPI, several GitHub projects, Apify actors) but sit behind Cloudflare and break regularly. Their spec depth for motherboards is also mediocre (fewer facets than Geizhals). Recommendation: not a viable primary source; at most a manual cross-check.

Sources:

- <https://pcpartpicker.com/robots.txt>
- <https://pcpartpicker.com/forums/topic/315918-is-scraping-against-tos>
- <https://pypi.org/project/pypartpicker/>

### Best Buy Products API: real structured specs but a 72-hour cache limit kills catalog use

*Confidence: `verified-live`*

Free API key by email signup (portal live and open); Products API returns structured details.name/details.value spec pairs plus features, UPC, model number, pricing; limits 50,000 calls/day, 5/sec. However the legal terms (fetched live) permit storing/caching content only 'on a temporary basis not to exceed seventy-two (72) hours', require conspicuous Best Buy attribution/logo, and prohibit use benefiting other retailers. A persistent self-hosted spec catalog built on stored Best Buy data would violate the ToS, and their motherboard assortment is thin regardless. Useful only as a live price/availability overlay, not a spec store.

Sources:

- <https://developer.bestbuy.com/>
- <https://developer.bestbuy.com/legal>
- <https://bestbuyapis.github.io/api-documentation/>

### Amazon PA-API 5.0: confirmed no deep technical spec structure

*Confidence: `verified-live`*

Fetched the ItemInfo documentation live: TechnicalInfo holds only format/energy-class style fields, ProductInfo holds dimensions/color/size/release date, Features is a flat list of marketing bullet strings. No structured chipset/PCIe/LAN attributes exist; attribute availability 'varies with locale and category'. Access also requires an Amazon Associates account with qualifying sales (3 sales in 180 days) and throttles scale with revenue. Verdict: unusable for spec filtering; marginal value for price links only.

Sources:

- <https://webservices.amazon.com/paapi5/documentation/item-info.html>

### Newegg: no public product/spec API — Marketplace API is seller-only; affiliate feeds are shallow

*Confidence: `reported`*

developer.newegg.com documents only the Marketplace API, which requires seller-account credentials (developer API key + seller secret key requested through Seller Portal) and is for listing management, not public catalog reads — though notably its item-creation feeds use per-category industry spec templates, proving Newegg holds structured spec data internally that it simply doesn't expose. Public-facing filtering data (their faceted 'Power Search') has no API; the affiliate program (via CJ/Rakuten networks) provides standard shopping feeds — title, price, image, category — not technical specs. Third parties (Piloterr etc.) sell Newegg page-scraping APIs. Verdict: Newegg is a UX benchmark, not a data source.

Sources:

- <https://developer.newegg.com/newegg_marketplace_api/>
- <https://sellerportal.newegg.com/selleracademy/knowledge-base/api-access-authorization-requesting-api-credentials/>
- <https://developers.cj.com/docs/data-imports/product-feeds>
- <https://promotions.newegg.com/affiliate_program/affiliate.html>

### Standards: Icecat taxonomy is the practical one; ETIM weak for IT; ECLASS/GS1 exist but are B2B-gated

*Confidence: `reported`*

Icecat's category/feature taxonomy (downloadable category feature lists with the free tier) is the de-facto standard for consumer electronics attribute schemas and directly maps to faceted filtering — the pragmatic choice to borrow feature IDs from. ETIM (2-level EG group / EC class model with typed features) is excellent in concept but focused on electrical/HVAC/building trades; IT-hardware coverage like motherboards is limited. ECLASS does cover IT hardware broadly but the dictionary is paid-membership licensed. GS1's GPC classification is far too coarse for spec filtering (brick-level categories) and GDSN data pool access is enterprise B2B — both irrelevant except for using GTIN/EAN as your canonical product identifier, which IS worth adopting (Icecat and Geizhals both key on EAN/GTIN). Suggested schema approach: your own attribute model, GTIN-keyed, with Icecat feature IDs as an interchange mapping and ETIM-style typed features (numeric+unit, boolean, enum) so facets stay queryable.

Sources:

- <https://viewer.etim-international.com/>
- <https://wisepim.com/guides/product-taxonomy/etim>
- <https://www.rastro.ai/resources/comparisons/eclass-vs-etim-for-distributors>
- <https://iceclog.com/open-catalog-interface-oci-open-icecat-xml-and-full-icecat-xml-repositories/>
