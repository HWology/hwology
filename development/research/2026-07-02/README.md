# Research reports — 2026-07-02

Web-verified research produced by parallel agent runs on 2026-07-02, backing the design in
[../../architecture.md](../../architecture.md). Access details (bot-blocking, API availability,
package versions) are perishable — re-verify before relying on them months later.

| Doc | Covers |
|---|---|
| [01-motherboard-spec-sources.md](01-motherboard-spec-sources.md) | Vendor pages, Icecat, Geizhals, retailer APIs — what's accessible and licensed |
| [02-open-datasets.md](02-open-datasets.md) | BuildCores OpenDB (the seed dataset), linuxhw/DMI, dead projects |
| [03-schema-storage.md](03-schema-storage.md) | EAV vs JSONB vs hybrid, slot child tables, vocabulary normalization, revisions |
| [04-firmware-bios-data.md](04-firmware-bios-data.md) | ASUS BIOS API, vendor BIOS pages, AGESA tracking, LVFS, Wayback backfill |
| [05-faceted-search.md](05-faceted-search.md) | Datasette, ItemsJS, Meilisearch/Typesense, plain-SQL facets |
| [06-claim-verification.md](06-claim-verification.md) | Adversarial fact-check verdicts (2 corrections: ASRock access, pgloader) |
| [07-gaps-and-risks.md](07-gaps-and-risks.md) | 13 design risks: ToS/legal, BIOS-history lossiness, product identity, lane-sharing PDFs, revision chaos, conditional specs… |
| [08-fsharp-dotnet-storage.md](08-fsharp-dotnet-storage.md) | Microsoft.Data.Sqlite, Dapper.FSharp, Marten/LiteDB/EF verdicts, FSharp.SystemTextJson, JSON Schema from F# types |
| [09-networking-gear-sources.md](09-networking-gear-sources.md) | OpenWrt toh.json, wikidevi.wi-cat.ru, FCC sources, MikroTik/Ubiquiti, devicecode-data |
| [10-archiving-provenance.md](10-archiving-provenance.md) | ArchiveBox state, browsertrix-crawler, single-file-cli, SPN2, sha256 stores, WACZ, copyright posture |
| [11-net11-fsharp-du-serialization.md](11-net11-fsharp-du-serialization.md) | .NET 11 native F# DU support in System.Text.Json (vs the separate C# unions feature) |
| [12-toml-data-files.md](12-toml-data-files.md) | TOML 1.1, Tomlyn 2.9 live-tested with F#, tombi, TOML-as-dataset prior art |
