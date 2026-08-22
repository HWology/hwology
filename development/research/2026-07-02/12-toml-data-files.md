# TOML as a data-file format for .NET/F#

*Research date: 2026-07-02, parallel web-research agents. Findings marked `verified-live` were confirmed against the live web that day; `reported` means sourced but not directly fetched; treat access details as perishable.*

## Summary

TOML is newly re-energized in 2026: spec 1.1.0 shipped 2025-12-24, and the .NET story is unexpectedly strong because Tomlyn had a full 2.x rewrite (v2.9.0, June 2026) into a System.Text.Json-style serializer targeting TOML 1.1 with net10.0 builds, source generation, and NativeAOT support. I verified Tomlyn 2.9 live on the local .NET 10 SDK from F#: immutable F# records with nested arrays-of-tables ([[boards]] / [[boards.slots]]) deserialize and round-trip cleanly, but F# 'T option and 'T list fail natively, and any missing key on a plain record throws (all constructor params required) — the working recipe is arrays/ResizeArray plus [<CLIMutable>] plus a ~10-line TomlConverter per option type, since no FSharp.SystemTextJson-equivalent exists for TOML; DUs need fully manual converters. TOML's no-null rule maps tolerably onto option-as-absent-key, and first-class dates (DateOnly round-trips unquoted) are a genuine win for a hardware catalog. Prior art for TOML-as-dataset exists at scale (RustSec advisory-db front matter, cargo-vet audits.toml) but the ecosystem lesson is that hand-edited records go in TOML while machine-generated indexes go to JSON-lines (the Cargo index was never TOML); tooling-wise taplo is orphaned and tombi is the active formatter/linter/LSP with JSON-Schema validation. If Ivan wants zero-friction F# type fidelity instead, JSONC via System.Text.Json (comments + trailing commas) with FSharp.SystemTextJson remains the path of least resistance; YAML via YamlDotNet 18.x is healthy but has the same F#-mapping gaps as Tomlyn plus implicit-typing hazards, and JSON5 support in .NET is thin.

## Findings

### TOML 1.1.0 is released (2025-12-24), no longer draft

*Confidence: `verified-live`*

The toml-lang/toml releases page shows Release 1.1.0 dated December 24 (2025), the first spec release since 1.0.0 (Jan 2021). Changes: newlines and trailing commas now allowed in inline tables (the big one — makes inline tables usable for record-like rows), new string escapes \xHH and \e, seconds optional in datetime/time values, plus clarifications on Unicode handling, dotted keys, comments, and parser flexibility. Ecosystem adoption is in progress through 2026 (Alacritty 0.17 added 1.1 syntax in April 2026; TOML-Fortran supports it; Python is discussing tomllib adoption on discuss.python.org). Practical note: 1.1 inline tables with trailing commas make compact one-line-per-slot layouts viable as an alternative to [[slot]] blocks.

Sources:

- <https://github.com/toml-lang/toml/releases>
- <https://discuss.python.org/t/adopting-toml-1-1/105624>

### Tomlyn 2.9.0 is the clear .NET front-runner: active, TOML 1.1, net10.0, STJ-style serializer

*Confidence: `verified-live`*

Tomlyn (xoofx) underwent a major 2.x rewrite. Verified on GitHub and NuGet: v2.9.0 published 2026-06-24, BSD-2-Clause, ~5.0M total downloads, targets netstandard2.0/net8.0/net10.0. Tomlyn 2.x targets TOML 1.1.0 ONLY (no 1.0 mode). Architecture is System.Text.Json-style: static TomlSerializer.Deserialize/Serialize, TomlSerializerOptions literally reuses System.Text.Json types (PropertyNamingPolicy: JsonNamingPolicy, JsonObjectCreationHandling), honors STJ attributes ([JsonPropertyName], [JsonIgnore], [JsonConstructor]), plus TOML-specific attributes (TomlInlineTable, TomlTableArrayStyle, TomlStringStyle, TomlRequired, TomlPolymorphic/TomlDerivedType for class-hierarchy polymorphism). Offers reflection mapping, source generation via TomlSerializerContext/[TomlSerializable] (NativeAOT-ready; reflection auto-disabled under PublishAot), an untyped TomlTable model, a lossless trivia-preserving syntax tree (comment/format-preserving round-trips), and TomlException with precise line/column/span diagnostics. I dumped the full public API surface locally from the 2.9.0 package to confirm all of this.

Sources:

- <https://github.com/xoofx/Tomlyn>
- <https://www.nuget.org/packages/Tomlyn>
- <https://xoofx.github.io/Tomlyn/docs/>

### Live-tested: Tomlyn 2.9 + F# records on .NET 10 — works, with three sharp edges (option, list, missing keys)

*Confidence: `verified-live`*

I ran F# scripts against Tomlyn 2.9.0 on the local .NET 10.0.301 SDK. WORKS: plain immutable F# records map via constructor (no parameterless ctor needed); nested arrays-of-tables ([[boards]] / [[boards.slots]], 3 levels) deserialize into records containing Slot[] and round-trip; System.DateOnly binds from bare TOML dates (released = 2024-09-30); .NET enums bind from strings; snake_case via JsonNamingPolicy.SnakeCaseLower; Nullable<T>/None fields are omitted on serialize (correct TOML no-null behavior). FAILS: 'T option — FSharpOption treated as a table ('Expected StartTable token but was String'); 'T list — FSharpList not recognized as a collection ('Expected StartTable token but was StartArray'; use 'T[] or ResizeArray, which both work); payload-less DUs ('No suitable constructor'); and critically, ANY absent key on a plain record throws 'Missing required constructor parameter' — every record field is a required ctor param, so optional fields break immutable records. VERIFIED WORKAROUND: a ~10-line TomlConverter<string option> (Read = reader.GetString() |> Some; Write = WriteStringValue / omit) registered via TomlSerializerOptions(Converters=...) fixes present-value mapping and None-omission, and combining it with [<CLIMutable>] makes absent keys correctly produce None/null. So: F# is usable with Tomlyn if you accept [<CLIMutable>] records, arrays instead of lists, and hand-written converters per option payload type; DUs require fully custom converters (Tomlyn's TomlPolymorphic/TomlDerivedType attributes exist but are designed for class hierarchies, untested with DU compiled forms). No FSharp.SystemTextJson-equivalent companion package exists for Tomlyn as of July 2026.

Sources:

- <https://github.com/xoofx/Tomlyn>

### CsToml 1.8.4: fastest option, TOML 1.1 opt-in, but its source generator is C#-only — F# gets manual mapping

*Confidence: `verified-live`*

CsToml (prozolic), MIT, v1.8.4 published 2026-06-25, targets net8/net9/net10 (depends on System.IO.Hashing 10.0.9), ~18K downloads (niche but actively maintained, 581 commits). System.Buffers-based UTF-8 parsing, low-allocation; passes the official toml-test suites for both v1.0.0 and v1.1.0 — 1.0 by default, 1.1 opt-in via CsTomlSerializerOptions (TomlSpec.Version110 or per-feature flags like AllowNewlinesInInlineTables). Rich type support: DateOnly/TimeOnly/DateTimeOffset, nullables, immutable/frozen collections, ITomlValueFormatter<T> for custom types. The catch for Ivan: POCO mapping requires CsToml.Generator, a C# source generator using [TomlSerializedObject] on partial classes — F# does not support C# source generators or that partial-class pattern, so from F# you are limited to the TomlDocument DOM plus hand-written mapping (or a C# DTO assembly). Also ships CsToml.Extensions.Configuration for Microsoft.Extensions.Configuration.

Sources:

- <https://github.com/prozolic/CsToml>
- <https://www.nuget.org/packages/CsToml>

### Tomlet 6.2.0: alive but conservative — TOML 1.0 only, reflection, parameterless ctors

*Confidence: `verified-live`*

Tomlet (SamboyCoding), MIT, package Samboy063.Tomlet v6.2.0 updated 2025-12-24, ~375K downloads, targets net6.0/netstandard2.0/net35 (runs on .NET 10 via netstandard2.0 but no modern TFM). Implements the full TOML 1.0.0 spec (no 1.1). Reflection-based mapping only; deserialization requires parameterless constructors, so F# records need [<CLIMutable>]; no option/DU/F# collection awareness; custom mappers via TomletMain.RegisterMapper<T>(). Preserves comments (preceding/inline) but not layout/ordering/whitespace. Fine as a simple dependency-free choice, but strictly weaker than Tomlyn 2.x for this use case. Rest of the field: Nett is dead (no development since ~2020), Tommy is a minimal single-file TOML 1.0 parser, and there is no maintained F#-native TOML library — ionide/fstoml is an experimental TOML project-file format (not a parser you'd use), and Toml.FSharp (FParsec) is long abandoned.

Sources:

- <https://github.com/SamboyCoding/Tomlet>
- <https://www.nuget.org/packages/Samboy063.Tomlet>
- <https://github.com/paiden/Nett>
- <https://github.com/ionide/fstoml>

### Practical fit for a hand-edited catalog: arrays-of-tables are good at 2 levels, tolerable at 3, and null-as-absence suits F# option

*Confidence: `verified-live`*

Verified output shape from Tomlyn: a catalog serializes as top-level scalars, then [[boards]] blocks each followed by their [[boards.slots]] blocks — each record is a clean, git-diff-friendly block, and None fields are simply omitted. The cost is header repetition: every slot carries the full dotted path ([[boards.slots]]), and at 3+ levels ([[boards.slots.subdevices]]) readability degrades because hierarchy is only visible in the key path, not structurally — indentation is voluntary and semantically meaningless. The hitchdev critique (StrictYAML author) quantifies this: TOML spends ~50% more characters than YAML for the same nested data and 'the relationships between data' are hard to perceive; toml-lang issues #309/#353/#1052 are long-running complaints that array-of-tables syntax is the spec's weakest point and nested arrays are less expressive than JSON. Mitigations: TOML 1.1 inline tables with trailing commas/newlines allow one-line-per-slot rows (slots = [ {gen=5, lanes=16}, ... ]) — Tomlyn exposes TomlInlineTableAttribute and InlineTablePolicy for exactly this. Null: TOML has no null by design; 'absent key = None' is actually a clean match for F# option once converters are in place, but you lose any missing-vs-explicitly-null distinction. Datetimes are a real TOML advantage over JSON: native offset/local datetime, local date, local time as unquoted first-class values — verified DateOnly round-trip in Tomlyn, and CsToml maps DateOnly/TimeOnly/DateTimeOffset.

Sources:

- <https://hitchdev.com/strictyaml/why-not/toml/>
- <https://github.com/toml-lang/toml/issues/1052>
- <https://github.com/toml-lang/toml/issues/353>

### Prior art: TOML datasets exist at scale (RustSec, cargo-vet), but machine-generated indexes use JSON-lines — and the Cargo index was never TOML

*Confidence: `verified-live`*

Real many-record TOML catalogs in git: (1) RustSec advisory-db — thousands of security advisories, each a file with TOML front matter ([advisory] table with id, package, dates, CVSS, version ranges); their V3 format deliberately moved TO Markdown+TOML front matter (issue #240), and the repo remains the canonical Rust vulnerability database. (2) cargo-vet — mozilla/supply-chain aggregates Mozilla's crate audit records into a large audits.toml (arrays of tables, machine-appended + human-reviewed), actively maintained and consumed cross-organization. (3) Hugo officially supports TOML files in the data/ directory for template datasets. Counter-lesson: the crates.io/Cargo registry index is newline-delimited JSON (one JSON object per line per version) and per the Cargo Book always has been — the folk claim that it 'moved from TOML to JSON' is wrong; the actual migration was transport (git clone to sparse HTTP), not format. The Rust ecosystem's revealed preference: TOML for files humans write and review (Cargo.toml, audits, advisories), JSON-lines for high-volume machine-generated data. For a hand-edited spec catalog, Ivan's use case sits on the TOML-appropriate side, with the caveat that nobody reports using TOML for deeply nested (3+ level) record hierarchies at scale — RustSec and cargo-vet records are 1-2 levels deep.

Sources:

- <https://github.com/rustsec/advisory-db>
- <https://github.com/RustSec/advisory-db/issues/240>
- <https://github.com/mozilla/supply-chain>
- <https://doc.rust-lang.org/cargo/reference/registry-index.html>

### Tooling 2026: taplo is orphaned, tombi is the successor (JSON Schema validation included); .NET-native conversion is半 DIY

*Confidence: `verified-live`*

taplo (Rust: formatter/linter/LSP/CLI with JSON-Schema-based validation) lost its maintainer — the author formally stepped down in issue #715 and the project's future is uncertain, though binaries still work. tombi is the actively maintained replacement (formatter + linter + LSP + VS Code extension): JSON Schema validation with SchemaStore distribution, comment directives to pin TOML version/schema/lint rules per file, schema-driven key sorting, and 'go to type definition' into the schema; major Rust projects (ratatui, wgpu) migrated taplo-to-tombi in 2025-2026. Schema story for TOML generally: there is no native TOML schema language — the ecosystem standard is JSON Schema applied to the parsed document (taplo/tombi in CI and editors), so Ivan can generate one JSON Schema from his F# types and get both editor completion and CI validation. Generic CLI converters: yj (sclevine) converts YAML/TOML/JSON/HCL preserving map order; dasel is actively maintained for query/convert across JSON/TOML/YAML/XML/CSV/KDL. .NET-native conversion, live-tested: Tomlyn TomlSerializer.Deserialize<TomlTable> then System.Text.Json serialize gives TOML-to-JSON in ~3 lines (caveat: TomlDateTime serializes as a structured object unless you add a small STJ converter); the reverse direction is not free — Tomlyn rejects raw JsonElement values ('Unsupported untyped TOML value'), so JSON-to-TOML needs a JsonElement-to-CLR walk first. No polished off-the-shelf .NET TOML/JSON converter package exists; for CI pipelines the Rust CLIs are the pragmatic choice.

Sources:

- <https://github.com/tamasfe/taplo/issues/715>
- <https://tombi-toml.github.io/tombi/docs/reference/difference-taplo/>
- <https://github.com/sclevine/yj>
- <https://github.com/tomwright/dasel>

### Comparison: JSONC via System.Text.Json + FSharp.SystemTextJson is the F#-fidelity baseline; YAML healthy but same F# gaps; JSON5 thin in .NET

*Confidence: `verified-live`*

JSONC: System.Text.Json natively reads comments and trailing commas (JsonSerializerOptions.ReadCommentHandling = Skip, AllowTrailingCommas = true) — no extra dependency on .NET 10 — and FSharp.SystemTextJson (Tarmil, v1.4.36, maintained) supplies complete F# type fidelity: records with missing-field handling, options (None as omitted field), lists/maps/sets, and DUs with four configurable encodings (AdjacentTag/ExternalTag/InternalTag/Untagged). That combination has no TOML equivalent and is the strongest argument for JSONC if F#-type roundtripping of DUs matters more than syntax niceness; main caveats are that STJ cannot WRITE comments (any tool that rewrites files destroys them) and .jsonc has no dates. YAML: YamlDotNet is healthy — v18.1.0 released 2026-06-26, active repo, explicit .NET 10 support; but like Tomlyn it has no native F# option/list/DU mapping (community converters or CLIMutable required), and YAML brings implicit-typing hazards (the Norway problem), anchors, and a much larger spec surface for hand-editors to hold; GitHub renders YAML front matter nicely which TOML lacks. JSON5: the weakest .NET story — json5-dotnet exists under the official json5 org but with low activity, and Json5Core (last update June 2025) is a small single-maintainer package depending on STJ 9.x; no first-party or high-download option, and JSON5 has no editor/schema ecosystem in .NET comparable to tombi-for-TOML or SchemaStore-for-JSON. Net assessment for a git-versioned, hand-edited, nested catalog: TOML (Tomlyn + tombi + JSON Schema) and JSONC (STJ + FSharp.SystemTextJson) are the two credible finalists; TOML wins on comment-preserving round-trips, dates, and diff-friendly [[record]] blocks; JSONC wins on zero-friction F# DU/option mapping and unlimited nesting depth.

Sources:

- <https://github.com/aaubry/YamlDotNet>
- <https://github.com/Tarmil/FSharp.SystemTextJson>
- <https://www.nuget.org/packages/FSharp.SystemTextJson>
- <https://github.com/json5/json5-dotnet>
- <https://www.nuget.org/packages/Json5Core>

### Ecosystem lag caveat on TOML 1.1 (reported, not fully verified)

*Confidence: `reported`*

Mixed-version risk through 2026: Tomlyn 2.x parses TOML 1.1 ONLY, CsToml defaults to 1.0 with 1.1 opt-in, Tomlet is 1.0-only, and Python's tomllib plus many CI tools still target 1.0 (adoption discussion ongoing on discuss.python.org). If Ivan's catalog files use 1.1 features (trailing commas in inline tables especially), non-Tomlyn consumers and older editor tooling may reject them; conversely Tomlyn's lack of a 1.0-strict mode means it will accept 1.1 syntax that other tools then choke on. Recommendation if cross-tool compatibility matters: keep files 1.0-clean for now (avoid inline-table trailing commas/newlines and \e/\x escapes) and enforce the version with a tombi comment directive.

Sources:

- <https://discuss.python.org/t/adopting-toml-1-1/105624>
- <https://github.com/toml-lang/toml/releases>
