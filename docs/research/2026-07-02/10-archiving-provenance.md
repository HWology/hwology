# Archiving and provenance tooling (browsertrix, SPN2, blob stores)

*Research date: 2026-07-02, parallel web-research agents. Findings marked `verified-live` were confirmed against the live web that day; `reported` means sourced but not directly fetched; treat access details as perishable.*

## Summary

ArchiveBox is alive but in a long pre-release limbo: stable is still v0.7.4 while the actively-developed v0.9.x RC line (v0.9.31-rc tagged 2026-05-18, docs at 0.9.35rc102) rewrites it around a split-out plugin ecosystem (abx-plugins), a standalone one-shot downloader CLI (abx-dl), and a real REST API — but it is one maintainer who explicitly says "no promises on stable release schedule". For a pipeline that already fetches pages itself, the stronger fits are browsertrix-crawler (v1.13.2, WARC/WACZ-first, browser profiles for bot-protected vendor pages, actively maintained by Webrecorder) plus single-file-cli v2 (can attach to your already-running, interactively-established browser session via --browser-remote-debugging-URL), with Wayback SPN2 as a free third-party "neutral witness" copy. For PDF manuals/firmware, the pragmatic 2026 answer is a plain sha256-named object store (optionally wrapped in BagIt bags for fixity, or managed by git-annex if you want multi-drive location tracking); DVC is the wrong shape. For provenance there is no single standard, but the workable pattern is: per catalog claim store source URL + retrieved-at timestamp + sha256 of the capture + a Memento URI-M (web.archive.org/web/{14-digit-ts}/{url}) and/or WACZ file + pages.jsonl page id; WACZ 1.1.1 with the wacz-auth signing extension is the closest thing to a provenance standard. On copyright: publish facts and links, keep manuals/firmware in the private archive — the Internet Archive hosts manuals under a fair-use/right-to-repair theory backed by DMCA takedown infrastructure you don't have, and Hachette v. Internet Archive (2d Cir. 2024, final Dec 2024) shows wholesale non-transformative rehosting loses; OpenWrt's ToH publishes factual spec tables and links out to vendor firmware rather than rehosting.

## Findings

### ArchiveBox 2026: stable v0.7.4, active v0.9.x RC line, single maintainer, honest 'no promises' status

*Confidence: `verified-live`*

Verified via GitHub API 2026-07-02: latest stable is still v0.7.4 (the v0.7.4 tag only gets Docker dependency refreshes — Chromium, Node, yt-dlp, SingleFile). Latest tagged pre-release is v0.9.31-rc (published 2026-05-18, target branch 'dev'); docs build from v0.9.35rc102 as of mid-June 2026, so dev churn is real and ongoing. v0.9.x is the big rewrite: plugins split into the abx-plugins ecosystem, archiving runs through the standalone abx-dl CLI (v1.11.235, pushed 2026-06-24), and the old pluggy in-process plugin system is replaced with an event-driven append-only-log runtime (stated goals: resumability, auditability, future browser-extension capture and p2p sync). The v0.9.31-rc release note says verbatim: latest stable is still v0.7.4, test on a backup first. Maintainer (Nick Sweeting / 'pirate') noted a new job and a child on the way with 'no promises on stable release schedule anytime soon'. Practical read: fine for a personal archive if you run stable v0.7.4 or accept RC churn; do not build tight coupling against v0.9 internals yet. PyPI stable requires Python >=3.9,<3.12.

Sources:

- <https://github.com/ArchiveBox/ArchiveBox/releases>
- <https://pypi.org/project/archivebox/>
- <https://github.com/ArchiveBox/abx-dl>

### ArchiveBox REST API (v0.8+/v0.9): usable for external-pipeline automation, including pushing your own ArchiveResults

*Confidence: `verified-live`*

Verified from live dev apidocs (built from v0.9.35rc102): Bearer-token auth (get token from /api/v1/auth/get_api_token or the admin UI, send Authorization: Bearer <token>). Core routes under /api/v1/core: GET/POST/PATCH/DELETE /snapshots, GET /snapshots/{id}, GET /snapshots/rss, tag CRUD, GET /any/{id} (resolve any object by id), and notably GET/POST/PATCH /archiveresults — POST 'creates or updates an ArchiveResult with output files' and PATCH 'appends/replaces files on existing results', i.e. an external scraper can register captures it made itself. Snapshots carry UUID/ABID identifiers plus url+timestamp. Caveat: API was labeled ALPHA on the stable-line PyPI page; it only exists in v0.8+/v0.9 RCs, not stable v0.7.4. Output formats per snapshot (unchanged concept across versions): SingleFile HTML, wget clone + WARC, Chrome PDF, PNG screenshot, DOM dump, readability/mercury article text, yt-dlp media, git clone, headers, favicon.

Sources:

- <https://docs.archivebox.io/dev/apidocs/archivebox/archivebox.api.v1_core.html>
- <https://docs.archivebox.io/dev/apidocs/index.html>

### ArchiveBox dedup and bot-protection behavior: URL-level dedup only; 'personas' for auth/cookies; still weak against hard bot walls

*Confidence: `reported`*

Dedup (reported from official docs/wiki): a collection is keyed on URL — re-adding an already-seen URL is ignored (earliest kept) unless --overwrite (re-archive in place) or a 're-snapshot' (new snapshot of same URL at new timestamp; the classic workaround was appending a #2020-10-24-style fragment). There is no content-level/cross-snapshot dedup — every snapshot dir stores full copies, so N snapshots of one vendor page = N copies. For bot-protected/JS-heavy pages: capture is headless-Chrome-based; v0.9 adds 'personas' (archivebox persona create --import=chrome|chromium|brave|edge) bundling a CHROME_USER_DATA_DIR, cookies.txt, user-agent and auth state per identity, which handles logins and simple cookie walls. But it does nothing special about Cloudflare-style fingerprinting (headless signals, TLS fingerprint), and ArchiveBox's own docs punt to ArchiveWeb.page/ReplayWeb.page for 'very complex interactive pages with heavy JS'. Expect Cloudflare-fronted vendor pages (TP-Link, ASUS support portals) to intermittently fail; docs also warn archived pages can preserve reflected session tokens, so use burner credentials in personas.

Sources:

- <https://github.com/ArchiveBox/ArchiveBox/wiki/Usage>
- <https://docs.archivebox.io/dev/Configuration.html>
- <https://docs.archivebox.io/dev/README.html>

### browsertrix-crawler v1.13.2: the WARC/WACZ-first choice, actively maintained, best story for bot-protected vendor pages

*Confidence: `verified-live`*

Verified via GitHub API: stable v1.13.2 (2026-06-20), v1.14.0-beta.2 published 2026-07-02 — very active, backed by Webrecorder (a real company, not one volunteer). Single Docker container (docker pull webrecorder/browsertrix-crawler), Puppeteer driving Brave (a real, non-headless-fingerprinting browser), writes standards-grade WARCs designed to stay valid even on interruption, packages to WACZ with --generateWACZ (WACZ = WARCs + CDXJ index + pages.jsonl + datapackage.json with sha256 per resource). Supports screenshots, text extraction to pages.jsonl or WARC resource records, proxies, per-page limits, and crucially reusable browser profiles (create-login-profile workflow) so you can pass an interactively-established, logged-in browser profile to unattended crawls. Fit for Ivan: even though his pipeline parses pages itself, running browsertrix as the archival capture path (parse from its WARC/WACZ output, or run it per-URL with --url and page limit 1) gives him preservation-grade artifacts plus replay via ReplayWeb.page. This is what I'd recommend as the primary archiver over ArchiveBox for a pipeline-driven system.

Sources:

- <https://github.com/webrecorder/browsertrix-crawler/releases>
- <https://crawler.docs.browsertrix.com/user-guide/>

### single-file-cli v2.0.83: attaches to your existing browser session over CDP — the best match for a scraper that already fetches pages

*Confidence: `verified-live`*

Verified live: latest release v2.0.83 (2025-11-29); README and options.js fetched from master. v2 runs on Deno (standalone compiled binaries per platform, or npx single-file-cli, or Docker capsulecode/singlefile); it injects SingleFile into the page via Chrome DevTools Protocol rather than a WebExtension. Key options confirmed in options.js: --browser-remote-debugging-URL (attach to an ALREADY-RUNNING browser — i.e., the same real browser session your scraper already uses, logins and cookies intact), --browser-server, --browser-executable-path, --browser-cookies-file (JSON or Netscape cookies.txt), --http-proxy-server, --crawl-links/--crawl-max-depth, --urls-file, --filename-template. Output is one self-contained .html per page — great for human-readable offline snapshots, sha256-friendly (single file), but NOT a WARC (no raw HTTP transactions/headers preserved). Ideal pattern for Ivan: scraper drives a headed Chromium with --remote-debugging-port, extracts specs, then invokes single-file against the same tab/session for the archival snapshot; pair with wget --warc-file or warcprox only if he also wants raw-HTTP provenance. monolith (Rust, similar single-HTML output) is at v2.10.1 (2025-03-30) but executes no JavaScript, so it's unsuitable for JS-heavy vendor spec pages.

Sources:

- <https://github.com/gildas-lormeau/single-file-cli/releases>
- <https://github.com/gildas-lormeau/single-file-cli>
- <https://github.com/Y2Z/monolith/releases>

### Wayback Machine SPN2 API: free third-party witness copy; authenticated = 'Authorization: LOW <S3-access>:<S3-secret>'; ~12 concurrent / 100k daily

*Confidence: `verified-live`*

Details verified from the widely-used community gist (IA has never published official SPN2 docs, so treat numbers as best-available, not contractual): POST https://web.archive.org/save with Authorization: LOW accesskey:secret (keys free from archive.org/account/s3.php); poll https://web.archive.org/save/status/{job_id}. Useful params: if_not_archived_within=<timedelta> (skip if a recent capture exists — natural dedup), capture_screenshot=1, capture_all=1 (keep 4xx/5xx), capture_outlinks=1, js_behavior_timeout (max 30s), skip_first_archive=1. Limits per gist: authenticated 12 concurrent captures / 100k daily; anonymous 6 concurrent / 4k daily; ~10 captures per URL per day; artificial delays kick in past ~20 concurrent captures on one host. Reliability caveat: community reports through March 2026 of SPN throttling/failed saves (429s, '1 success in 16 attempts' style complaints), so use it as a secondary, fire-and-forget provenance anchor (and public citation URL), never the primary archive. Bonus: pushing vendor pages to SPN2 means IA hosts the public copy — your catalog can cite a web.archive.org URL instead of rehosting anything.

Sources:

- <https://gist.github.com/regstuff/82e690db2f1d91ba59f6681c1abad6cf>
- <https://github.com/overcast07/wayback-machine-spn-scripts>

### Binary archive store (manuals PDFs, firmware images): plain sha256 store or git-annex; BagIt for interop; DVC is the wrong tool

*Confidence: `verified-live`*

What preservation/hoarding communities actually use in 2026: (1) Libraries/archives use BagIt (RFC 8493, Oct 2018, still current) — a directory with data/ payload + manifest-sha256.txt fixity manifests; still actively adopted (e.g. Harvard Library Innovation Lab's 2025 data.gov archive used BagIt for authenticity/provenance). It's a packaging/fixity convention, not a dedup store. (2) git-annex remains actively maintained (Debian sid ships 10.20251029; date-versioned releases continue) — SHA256E backend gives content-addressed keys with human-readable symlink/pointer names, plus location tracking across drives/remotes ('which of my 3 disks has this firmware'); it's the datahoarder/DataLad choice when collections span media. (3) DVC (3.67.1 on PyPI, 2026-03-31, verified) is content-addressed but built around ML pipeline reproducibility; community comparisons consistently frame it as workflow-tracking, with less flexibility for pure data management than git-annex — skip it. Pragmatic recommendation for a SQLite-backed catalog: a plain objects/<first2>/<sha256> file store with the sha256, original filename, source URL, and retrieved-at recorded in SQLite (this is exactly what most one-machine hoarders do), upgrade to git-annex only when it outgrows one disk, and export BagIt bags if it's ever handed to a real archive. Firmware-specific note: keep vendor-published checksums alongside your own sha256 when the vendor provides them — that's provenance evidence, not just integrity.

Sources:

- <https://www.rfc-editor.org/rfc/rfc8493.html>
- <https://sources.debian.org/api/src/git-annex/>
- <https://pypi.org/pypi/dvc/json>
- <https://discuss.dvc.org/t/comparison-to-datalad-and-git-annex/92>

### Provenance linking catalog→snapshot: no single standard; the working pattern is Memento URI-M + sha256 + WACZ page id; WACZ signing is the closest formal spec

*Confidence: `verified-live`*

Available identifier systems, all verified in specs/docs: (a) Memento (RFC 7089) URI-M form — https://web.archive.org/web/{YYYYMMDDhhmmss}/{url} — the de-facto universal citation for any Wayback-compatible archive (pywb can serve your own WARCs the same way locally). (b) WARC records each carry WARC-Record-ID (urn:uuid:...), WARC-Date and WARC-Payload-Digest — stable but only meaningful with the WARC file in hand. (c) WACZ 1.1.1 (spec fetched live): datapackage.json lists every resource with sha256; pages.jsonl entries have id, url, ts (RFC3339) — so 'wacz sha256 + page id' is a precise, portable snapshot reference; the WACZ Signing and Verification extension (wacz-auth 0.1.0, used by Starling Lab/Harvard LIL/Perma tools) adds datapackage-digest.json with a cryptographic signature + RFC 3161 timestamp — the closest thing to a provenance standard in this space. (d) ArchiveBox snapshots have UUIDs/ABIDs addressable via GET /api/v1/core/snapshots/{id} — but they're instance-local; don't make them your canonical provenance key while the project is in RC flux. Recommended schema for Ivan's catalog: per fact/spec-field, a source row {source_url, retrieved_at ISO-8601 UTC, capture_sha256, local_store_path or wacz_file+page_id, wayback_urim (nullable), tool+version} — publishable subset: source_url, retrieved_at, wayback_urim, sha256.

Sources:

- <https://specs.webrecorder.net/wacz/1.1.1/>
- <https://github.com/webrecorder/wacz-auth-spec/blob/main/spec.md>
- <https://www.rfc-editor.org/rfc/rfc7089.html>

### Copyright posture: established projects either (a) host under fair-use + DMCA machinery at institutional scale, or (b) publish facts and link out — do (b) publicly, archive privately

*Confidence: `reported`*

What established projects actually do: Internet Archive's Manuals collection (1M+ items) openly hosts full copyrighted manuals, explicitly justified in their 2024 'Fair Use in Action' post via right-to-repair framing, backed by a formal DMCA takedown process and institutional legal capacity — and they still lost Hachette v. Internet Archive (2d Cir. affirmed no fair use for wholesale non-transformative copying, Sept 2024; final after IA declined cert, Dec 2024; permanent injunction + confidential payment). Bitsavers and retro-computing libraries host scans of decades-old orphaned documentation — tolerated largely because rightsholders are gone or indifferent, which does NOT extend to current TP-Link/ASUS/Netgear manuals. OpenWrt's Table of Hardware (toh.openwrt.org, nightly toh.json from user-maintained wiki dataentry pages) is the model closest to Ivan's project: it publishes factual spec tables (facts aren't copyrightable — Feist) and firmware DOWNLOAD URLS pointing at vendor sites, hosting only OpenWrt's own built images, never vendor stock firmware or manuals. Recommended posture: public catalog = facts + vendor links + Wayback URI-Ms + sha256 hashes of manuals ('we verified against manual X, hash Y, archived at Wayback Z'); private archive = full PDFs/firmware/WARCs, unpublished (private copying for reference is low-risk; redistribution is where liability lives, and firmware adds EULA/blob issues beyond copyright — only GPL components are safely redistributable). Pushing source pages through SPN2 cleverly outsources public hosting of the copyrighted originals to IA.

Sources:

- <https://blog.archive.org/2024/03/01/fair-use-in-action-at-the-internet-archive/>
- <https://blog.archive.org/2024/12/04/end-of-hachette-v-internet-archive/>
- <https://en.wikipedia.org/wiki/Hachette_v._Internet_Archive>
- <https://openwrt.org/toh/views/toh_fwdownload>
- <https://toh.openwrt.org/>
