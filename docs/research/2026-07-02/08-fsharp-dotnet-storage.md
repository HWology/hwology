# F#/.NET storage ecosystem (SQLite, Dapper.FSharp, serialization)

*Research date: 2026-07-02, parallel web-research agents. Findings marked `verified-live` were confirmed against the live web that day; `reported` means sourced but not directly fetched; treat access details as perishable.*

## Summary

For a 10^3–10^5-entity faceted catalog in F#, plain SQLite via Microsoft.Data.Sqlite 10.0.9 (MIT, actively shipped monthly) plus a thin F# data-access layer is the sweet spot; Dapper.FSharp (4.10.0, Apr 2026) and Dapper (2.1.79, May 2026) are actively maintained, Donald is stable-but-quiet (last release Mar 2024), and SQLProvider is very active (1.5.24, Jun 2026) but its Microsoft.Data.Sqlite backend is still marked experimental. Marten 9.12.0 (Jun 2026, MIT) now defaults to System.Text.Json and works well with FSharp.SystemTextJson, but it drags in PostgreSQL hosting and its multi-case-DU query story is still patchy — overkill for a single-user catalog unless Ivan wants its document/event-store features. LiteDB is alive under litedb-org (v5.0.21 stable Jul 2024, v6 still prerelease as of Feb 2026) but offers no advantage over SQLite here; EF Core remains the highest-friction F# option (CLIMutable, option-type converters, stale EFCore.FSharp for migrations). For the git-versioned public dataset, FSharp.SystemTextJson 1.4.36 + FSharp.Data.JsonSchema 3.0.1 (rewritten Feb 2026, generates JSON Schema directly from F# records/DUs) is the strongest combo, and native STJ discriminated-union support just merged for .NET 11. In Blazor Server, remember Microsoft.Data.Sqlite async methods run synchronously — enable WAL, serialize writes, and pin the SQLitePCLRaw bundle to 3.0.3 (SQLite 3.50.4, jsonb-capable) since the 2.1.11 native lib is deprecated with a high-severity vulnerability.

## Findings

### Microsoft.Data.Sqlite: healthy, MIT, monthly servicing on the 10.x line

*Confidence: `verified-live`*

Latest stable 10.0.9 (2026-06-09), MIT, 112.6M downloads; releases roughly monthly (10.0.5 Mar 2026 → 10.0.9 Jun 2026). Lightweight ADO.NET provider, targets netstandard2.0 through net10.0. JSON support is whatever the bundled SQLite engine provides — json_extract/->>/jsonb all work through plain SQL; there is no special 'JSON column' API, which is fine for Dapper/Donald-style access. Gotcha: its dependency floor is SQLitePCLRaw.bundle_e_sqlite3 >= 2.1.11, and the 2.1.11 native lib package is deprecated with a high-severity vulnerability (see SQLitePCLRaw finding) — add an explicit reference to bundle_e_sqlite3 3.0.3 to lift it.

Sources:

- <https://www.nuget.org/packages/Microsoft.Data.Sqlite>
- <https://www.nuget.org/packages/Microsoft.Data.Sqlite/10.0.9>

### SQLitePCLRaw: 2.x native lib deprecated/vulnerable; 3.0.3 bundles SQLite 3.50.4 (jsonb-era)

*Confidence: `verified-live`*

SQLitePCLRaw.lib.e_sqlite3 2.1.11 (Mar 2025) is now marked deprecated on NuGet 'due to critical bugs' and flagged with a high-severity vulnerability; the replacement native package is SourceGear.sqlite3. SQLitePCLRaw.bundle_e_sqlite3 3.0.3 (2026-05-07, Apache-2.0) depends on SourceGear.sqlite3 >= 3.50.4.5, i.e. SQLite 3.50.4 — comfortably past 3.45, so jsonb() storage and all JSON operators are available. Concrete action for Ivan: reference SQLitePCLRaw.bundle_e_sqlite3 3.0.3 explicitly alongside Microsoft.Data.Sqlite 10.x.

Sources:

- <https://www.nuget.org/packages/SQLitePCLRaw.lib.e_sqlite3/2.1.11>
- <https://www.nuget.org/packages/sqlitepclraw.bundle_e_sqlite3/>
- <https://github.com/ericsink/SQLitePCL.raw>

### Dapper + Dapper.FSharp: both actively maintained; good default for this catalog

*Confidence: `verified-live`*

Dapper 2.1.79 (2026-05-16, Apache-2.0, no deps on modern .NET). Dapper.FSharp 4.10.0 (2026-04-01, MIT, author dzoukr) targets net6.0–net10.0 and explicitly supports SQLite (including InsertOrReplaceAsync for INSERT OR REPLACE). Design philosophy: covers ~90% of repetitive CRUD; max 4 joined tables, weak aggregates — for faceted-count SQL (GROUP BY, json_each, generated columns) you drop to raw Dapper strings, which is idiomatic anyway. Handles F# records and option types natively (that is its raison d'être). This is the most F#-ergonomic actively-maintained option for SQLite.

Sources:

- <https://www.nuget.org/packages/Dapper>
- <https://www.nuget.org/packages/Dapper.FSharp>
- <https://github.com/Dzoukr/Dapper.FSharp>

### Donald: stable and pleasant but effectively dormant since 2024

*Confidence: `verified-live`*

Donald 10.1.0 (NuGet, 2024-03-20, Apache-2.0, Pim Brouwers); last repo commits are README touch-ups from Oct–Dec 2024. Pipe-forward F# API (conn |> Db.newCommand sql |> Db.setParams ... |> Db.query mapper) over any ADO.NET provider. It's small and 'finished' rather than abandoned, but if Ivan wants a package with an active issue tracker, Dapper.FSharp is the safer bet. No JSON-specific features — you map strings and deserialize yourself, which composes fine with FSharp.SystemTextJson.

Sources:

- <https://www.nuget.org/packages/Donald>
- <https://github.com/pimbrouwers/Donald>

### SQLProvider: very actively maintained, but SQLite-on-.NET-Core path is its weak spot

*Confidence: `verified-live`*

SQLProvider 1.5.24 released 2026-06-14 (Apache-2.0, fsprojects; requires FSharp.Core >= 8.0.301). Erasing type provider giving compile-time-checked LINQ against live schema. Docs state the Microsoft.Data.Sqlite backend is 'experimental' because it lacks GetSchema() support; the recommended System.Data.SQLite path needs manual interop-library placement (libSQLite.Interop.so on Linux). Also needs a design-time DB connection, which fights a JSON-seeded, migration-from-code workflow, and JSON columns are just TEXT to it. Great tech, awkward fit for this specific project.

Sources:

- <https://www.nuget.org/packages/SQLProvider>
- <https://fsprojects.github.io/SQLProvider/core/sqlite.html>
- <https://raw.githubusercontent.com/fsprojects/SQLProvider/master/LICENSE.txt>

### Marten 9.12.0: System.Text.Json is now the default serializer; F# workable but Postgres-only

*Confidence: `verified-live`*

Marten 9.12.0 (2026-06-29, MIT, JasperFx stack: JasperFx 2.18.x, Weasel.Postgresql 9.3). As of Marten 9, System.Text.Json is the default and Newtonsoft moved to a separate Marten.Newtonsoft package — so plugging JsonFSharpConverter into UseSystemTextJsonForSerialization gives records/DUs/options in stored documents. Marten docs support single-case DU document identities (Guid/string inner types only; you set the id yourself since records are immutable).

Sources:

- <https://www.nuget.org/packages/Marten>
- <https://martendb.io/configuration/json.html>
- <https://martendb.io/documents/identity>

### Marten F# gaps + verdict: overkill for a single-user catalog

*Confidence: `reported`*

Querying on multi-case DU properties has been a long-standing gap (JasperFx/marten issues #1283, #3286 — improving but not fully solved as far as I could confirm), and the community wrapper Marten.FSharp is dead (v0.6.2, Jun 2023). Marten's compelling features (event sourcing, projections, LINQ over JSONB with GIN indexes) come at the cost of running PostgreSQL and living with C#-first APIs. For ~10^3–10^5 entities, single user, SQLite with JSON1 + a few generated columns gives equivalent faceted-filter power with zero server footprint. Marten only wins if Ivan later wants multi-user hosting on Postgres or an audit/event history of spec edits.

Sources:

- <https://github.com/JasperFx/marten/issues/3286>
- <https://github.com/JasperFx/marten/issues/1283>
- <https://github.com/TheAngryByrd/Marten.FSharp>

### LiteDB: alive under litedb-org, but no reason to prefer it here

*Confidence: `verified-live`*

Stable is still 5.0.21 (2024-07-05, MIT); development continues as 6.0.0 prereleases (latest -prerelease.77, 2026-02-27) under the litedb-org organization (moved from mbdavid's personal account). BsonDocument model, no SQL JSON functions, weaker tooling than SQLite (no sqlite3 CLI/Datasette ecosystem for a published dataset), and F# support relies on its own BsonMapper which doesn't understand DUs/options without custom mapping. Only advantage is a pure-managed single DLL with no native dependency — irrelevant on a Linux dev box. Skip.

Sources:

- <https://www.nuget.org/packages/LiteDB>
- <https://github.com/litedb-org/LiteDB>

### EF Core + F#: still the highest-friction option in 2026

*Confidence: `reported`*

EF Core 10 (LTS, Nov 2025, support to Nov 2028) works with F# but the classic pain points persist: entities want [<CLIMutable>] records; option types need value converters (or you model with Nullable/allowNullLiteral types); DUs are unmapped; migration files are generated in C# unless you use EFCore.FSharp, whose last release is v6.0.7 (June 2022) targeting EF Core 6 — effectively stale; and generated migration files must be manually added to the .fsproj compile list. Fine if Ivan already knew EF, but for a read-heavy faceted catalog it buys little over Dapper.FSharp + hand SQL.

Sources:

- <https://github.com/efcore/EFCore.FSharp>
- <https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/whatsnew>
- <https://hamy.xyz/blog/2023-11-fsharp-entity-framework>

### FSharp.SystemTextJson: the default choice for the public dataset serializer

*Confidence: `verified-live`*

v1.4.36 (2025-06-13, MIT, Tarmil), 1.3M+ downloads of current version. Adds records/DUs/options/maps/lists to System.Text.Json with configurable, deterministic union encodings (AdjacentTag/ExternalTag/InternalTag/Untagged + UnionNamedFields), so JSON shape is an explicit, documented choice — key for a stable published format. Baseline STJ (since .NET 6) already handles F# records, options, lists, sets, maps natively; only DUs need this package today. For diff-friendliness: STJ preserves declaration order, and .NET 9+ JsonSerializerOptions exposes IndentCharacter/IndentSize/NewLine for byte-stable, git-friendly output across platforms.

Sources:

- <https://www.nuget.org/packages/FSharp.SystemTextJson>
- <https://github.com/Tarmil/FSharp.SystemTextJson>

### Native F# DU support merged into System.Text.Json for .NET 11

*Confidence: `verified-live`*

dotnet/runtime PR #125610 by Eirik Tsarpalis merged 2026-03-27, milestone .NET 11.0.0, closing the 2021-era issue #55744. Fieldless cases serialize as strings ('Point'), cases with fields as {"$type":"Circle","radius":3.14}; honors naming policies and [JsonPolymorphic]; reflection-only (no source-gen/Native AOT). Relevant to Ivan: from .NET 11 (Nov 2026) the public dataset could be plain STJ with zero extra packages, but the built-in encoding differs from FSharp.SystemTextJson's defaults — whichever he ships first becomes the frozen format, so pick deliberately now.

Sources:

- <https://github.com/dotnet/runtime/pull/125610>
- <https://github.com/dotnet/runtime/issues/55744>

### Thoth.Json: actively maintained, maximal control, but more boilerplate

*Confidence: `verified-live`*

The modern ecosystem is Thoth.Json.Core 0.9.1 (2026-06-03, MangelMaxime) with backend packages Thoth.Json.Newtonsoft, Thoth.Json.System.Text.Json, Thoth.Json.JavaScript, Thoth.Json.Python; legacy Thoth.Json.Net is at 12.0.0 (May 2024). Elm-style explicit encoders/decoders give the most stable wire format possible (nothing changes unless you edit the encoder) and graceful versioning — but for a spec catalog with dozens of record types the hand-written encoder maintenance is real. Best reserved for the public API boundary if the schema churns; otherwise FSharp.SystemTextJson with pinned options is less work. Note Thoth.Json.Core 0.9.1 requires a very recent FSharp.Core (>= 10.1.203).

Sources:

- <https://www.nuget.org/packages/Thoth.Json.Core>
- <https://www.nuget.org/packages/Thoth.Json.Net>

### JSON Schema from F# types: FSharp.Data.JsonSchema 3.x is maintained and purpose-built

*Confidence: `verified-live`*

fsprojects/FSharp.Data.JsonSchema was restructured in v3.0.1 (2026-02-10, MIT, active): FSharp.Data.JsonSchema.Core is a target-agnostic schema IR + F# type analyzer depending only on FSharp.Core and FSharp.SystemTextJson; FSharp.Data.JsonSchema.NJsonSchema is the recommended generator/validator package (the old unqualified package name is now a deprecated shim). Mapping: records → object+required, DUs → anyOf with discriminator, options → nullable, fieldless DUs → string enums — i.e. schemas that match FSharp.SystemTextJson output, exactly what a documented git-versioned dataset needs. The BCL alternative, System.Text.Json.Schema.JsonSchemaExporter (.NET 9+), works from serializer contracts but emits opaque/unknown schemas for types handled by custom converters like JsonFSharpConverter, so it can't describe DUs today (confidence on that specific limitation: reported).

Sources:

- <https://github.com/fsprojects/FSharp.Data.JsonSchema>
- <https://www.nuget.org/packages/FSharp.Data.JsonSchema>
- <https://learn.microsoft.com/en-us/dotnet/api/system.text.json.schema.jsonschemaexporter?view=net-9.0>

### SQLite in-process with Blazor Server: async is fake; use WAL and serialize writes

*Confidence: `verified-live`*

Official Microsoft docs: 'SQLite doesn't support asynchronous I/O. Async ADO.NET methods will execute synchronously in Microsoft.Data.Sqlite. Avoid calling them,' with the recommendation to enable PRAGMA journal_mode='wal' instead. For a Blazor Server (Fun.Blazor) app this means: DB calls block the circuit's thread-pool thread briefly (harmless at this data size), WAL lets concurrent circuit reads proceed while one writer works, writes should go through a single serialized path (BEGIN IMMEDIATE for write transactions avoids SQLITE_BUSY upgrades), and Microsoft.Data.Sqlite's built-in connection pooling (on by default) makes open-per-operation cheap. Practical pattern for faceted filtering: promote hot facet fields to generated columns (ALTER TABLE ... ADD COLUMN x AS (json_extract(specs,'$.x'))) with indexes, keep the long tail in a JSON blob. At 10^4 rows, even full-scan json_each queries are sub-millisecond-to-few-ms, so this is comfortably in-process.

Sources:

- <https://learn.microsoft.com/en-us/dotnet/standard/data/sqlite/async>
- <https://berthub.eu/articles/posts/a-brief-post-on-sqlite3-database-locked-despite-timeout/>

### Recommended stack for Ivan's catalog

*Confidence: `reported`*

Storage: SQLite via Microsoft.Data.Sqlite 10.0.9 + explicit SQLitePCLRaw.bundle_e_sqlite3 3.0.3; one file for public facts, a separate DB/file tree for the private archive (clean licensing separation, matches the ODC-By vs copyrighted-manuals split). Access: Dapper.FSharp for CRUD + raw Dapper for facet-count SQL; Donald as a lighter alternative if he prefers zero query-DSL. Public dataset: F# domain types serialized with FSharp.SystemTextJson (pinned JsonFSharpOptions + WriteIndented + fixed NewLine) committed to git, with schemas generated by FSharp.Data.JsonSchema.NJsonSchema and published alongside; SQLite DB treated as a rebuildable index over the JSON, not the source of truth. Skip Marten, LiteDB, and EF Core for this project; revisit plain STJ DU support when .NET 11 lands (Nov 2026).
