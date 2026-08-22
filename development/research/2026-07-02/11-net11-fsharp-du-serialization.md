# .NET 11 System.Text.Json F# discriminated-union support

*Research date: 2026-07-02, parallel web-research agents. Findings marked `verified-live` were confirmed against the live web that day; `reported` means sourced but not directly fetched; treat access details as perishable.*

## Summary

.NET 11's new System.Text.Json discriminated-union support (dotnet/runtime PR #125610, Eirik Tsarpalis, merged 2026-03-27, closing issue #55744) is specifically for F# discriminated unions via reflection — it is on by default in the reflection-based serializer, shipped in .NET 11 Preview 4 (2026-05-12), and is usable today on the current Preview 5 (2026-06-09). Wire format: fieldless cases serialize as plain JSON strings ("Point"); cases with fields serialize as objects with a $type discriminator ({"$type":"Circle","radius":1.5}); single-case unions are NOT unwrapped ({"$type":"MySingleCaseUnion","Item":"hello"}); struct unions work; option/voption keep their pre-existing unwrap-to-value converters; naming policies, JsonPropertyName, and [JsonPolymorphic(TypeDiscriminatorPropertyName=...)] are all honored. The hard limitation is reflection-only: no source generation and no Native AOT for F# DUs. This differs from FSharp.SystemTextJson's default {"Case":...,"Fields":[...]} adjacent-tag encoding (which also unwraps single-case unions by default); the library has had no release since June 2024 and no public interop/migration statement, though its InternalTag+NamedFields+UnionTagName("$type")+UnwrapFieldlessTags options can approximate the new built-in format. Separately, C# language "type unions" (Type Unions Working Group proposal) did NOT ship in C# 14 — they are a C# 15 preview feature riding .NET 11 (union Pet(Cat, Dog); lowering to a struct implementing IUnion), with their own STJ support (JsonTypeInfoKind.Union, source-gen/AOT capable, C#-only) merged in PR #128162 for Preview 6 (~mid-July 2026).

## Findings

### Scope: PR #125610 / issue #55744 are about F# DUs only — C# unions are a separate track

*Confidence: `verified-live`*

Issue dotnet/runtime#55744 ('System.Text.Json: Consider supporting F# discriminated unions') explicitly targets F# DUs (FSharp.Reflection-based shapes), noting there is no single canonical JSON encoding and distinguishing single-case wrappers, enum-like unions, and complex unions. PR #125610 (author eiriktsarpalis, merged 2026-03-27, .NET 11 milestone) implements F#-DU-only support in the reflection serializer and closed the issue. C# unions are handled separately: runtime scaffolding System.Runtime.CompilerServices.UnionAttribute + IUnion ships in .NET 11, and STJ support for C# unions came later in PR #128162 (merged 2026-05-21) — it does not extend the F# converter.

Sources:

- <https://github.com/dotnet/runtime/issues/55744>
- <https://github.com/dotnet/runtime/pull/125610>
- <https://github.com/dotnet/runtime/pull/128162>
- <https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-11/libraries>

### Wire format: strings for fieldless cases, $type-discriminated objects for cases with fields; single-case DUs are NOT unwrapped

*Confidence: `verified-live`*

Verified from official docs and the in-tree test file (System.Text.Json.FSharp.Tests/UnionTests.fs): fieldless case Point -> "Point" (plain string; [JsonPropertyName] can rename it, e.g. "pt"); Circle 1.5 -> {"$type":"Circle","radius":1.5}; single-case union with an unnamed field -> {"$type":"MySingleCaseUnion","Item":"hello"} (no unwrapping — a notable difference for wrapper types like `type Email = Email of string`); struct DUs serialize identically ({"$type":"StructCircle","radius":2}); F# option/voption are excluded from the union converter and keep their .NET 6-era converters (None -> null, Some x -> x). Generic unions are covered per PR commits ('Fix IL2121 linker error and add T-Gro test coverage'), though the main test file mostly uses concrete instantiations. Both string and object forms are accepted on deserialization. ReferenceHandler.Preserve is supported ($id/$ref; under Preserve a fieldless case becomes {"$id":"1","$type":"Point"}).

Sources:

- <https://github.com/dotnet/runtime/pull/125610>
- <https://raw.githubusercontent.com/dotnet/runtime/main/src/libraries/System.Text.Json/tests/System.Text.Json.FSharp.Tests/UnionTests.fs>
- <https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-11/libraries>

### On by default; respects JsonPolymorphic, naming policies, JsonPropertyName; conflicts throw

*Confidence: `verified-live`*

The support is automatic (no opt-in) whenever the reflection-based serializer encounters an F# union — it replaces the old NotSupportedException ('F# discriminated union serialization is not supported...'). It honors PropertyNamingPolicy (camelCase policy yields {"$type":"circle","radius":1} — applied to case names and field names), PropertyNameCaseInsensitive, JsonPropertyNameAttribute, RespectRequiredConstructorParameters, and [JsonPolymorphic(TypeDiscriminatorPropertyName = "kind")] to rename the discriminator ({"kind":"Beta","x":42,"y":"hello"}). A union case field whose name collides (case-insensitively) with the discriminator property throws InvalidOperationException at contract-build time.

Sources:

- <https://github.com/dotnet/runtime/pull/125610>
- <https://raw.githubusercontent.com/dotnet/runtime/main/src/libraries/System.Text.Json/tests/System.Text.Json.FSharp.Tests/UnionTests.fs>

### Availability: shipped in .NET 11 Preview 4; usable today on Preview 5; no F#-specific regressions found since merge

*Confidence: `verified-live`*

.NET 11 preview cadence (dotnet/core release notes): Preview 1 2026-02-10, P2 2026-03-10, P3 2026-04-14, P4 2026-05-12 (contains the F# DU feature — confirmed in Preview 4 release notes and F# Weekly #20), P5 2026-06-09 (current as of 2026-07-02). GA targeted 2026-11-10 (STS, supported to Nov 2028). So yes — installable and usable today with `dotnet` 11.0 Preview 5. GitHub searches for post-merge issues found no F#-union-specific bug reports; the notable follow-ups are PR #128162 (C# union STJ support, milestone 11.0-preview6, so landing in Preview 6 ~mid-July) and API proposal #129041 (closed-hierarchy polymorphism via InferDerivedTypes, opened 2026-06-05, still open). PR reviewers flagged two low-risk behavioral notes (metadata-skipping breadth; a silent-ignore shift in DefaultJsonTypeInfoResolver exception handling).

Sources:

- <https://github.com/dotnet/core/blob/main/release-notes/11.0/README.md>
- <https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview4/libraries.md>
- <https://sergeytihon.com/2026/05/16/f-weekly-20-2026-net-11-preview-4-and-swaggerprovider-4-0/>
- <https://github.com/dotnet/runtime/issues/129041>

### Limitations: reflection-only — no source generation, no Native AOT for F# DUs; no published perf numbers

*Confidence: `verified-live`*

Per the PR itself: the feature 'works exclusively with the reflection-based serializer' and relies on RequiresUnreferencedCode/RequiresDynamicCode paths — the C# source generator cannot emit metadata for F# unions, so Native AOT and trimmed apps are out. Notably, the later PR #128162 adds source-gen + AOT-compatible union metadata (JsonTypeInfoKind.Union, JsonUnionCaseInfo, JsonUnionAttribute, SYSLIB1227/1228 diagnostics) but explicitly for C# unions only — F# DUs remain reflection-only in .NET 11. No benchmark/performance figures were published in the PR or release notes that I could find; expect typical reflection-converter overhead (contract built once and cached, per the converter-factory design).

Sources:

- <https://github.com/dotnet/runtime/pull/125610>
- <https://github.com/dotnet/runtime/pull/128162>

### vs FSharp.SystemTextJson: incompatible defaults, configurable to match; no interop/migration statement from the library yet

*Confidence: `verified-live`*

FSharp.SystemTextJson's default (JsonFSharpOptions.Default = AdjacentTag + UnwrapOption + UnwrapSingleCaseUnions + AllowUnorderedTag) produces {"Case":"WithArgs","Fields":[123,"Hello, world!"]} / {"Case":"NoArgs"} and unwraps single-case unions (UserId "tarmil" -> "tarmil"). The .NET 11 built-in format differs on all three axes: internal $type tag with named fields, plain strings for fieldless cases, and no single-case unwrapping. The built-in format CAN be approximated in FSharp.SystemTextJson via WithUnionInternalTag().WithUnionNamedFields().WithUnionTagName("$type").WithUnionUnwrapFieldlessTags() — useful for producing .NET 11-compatible files from .NET 8/10 today. As of 2026-07-02 the library shows no release since v1.4.36 (2024-06-13) and no issues/release notes mentioning .NET 11 interop or migration plans; treat 'plans' as: none announced. For Ivan's git-versioned catalog: the built-in format is diff-friendly, but single-case wrapper types will carry an {"$type":...,"Item":...} envelope with no unwrap option, whereas FSharp.SystemTextJson gives full format control.

Sources:

- <https://github.com/Tarmil/FSharp.SystemTextJson/blob/master/docs/Format.md>
- <https://raw.githubusercontent.com/Tarmil/FSharp.SystemTextJson/master/docs/Customizing.md>
- <https://github.com/Tarmil/FSharp.SystemTextJson/releases>

### C# language 'type unions': not in C# 14 — C# 15 preview feature in .NET 11, still evolving; a different thing from F# DUs

*Confidence: `verified-live`*

The csharplang unions proposal (Type Unions Working Group, proposals/unions.md) remains a design document with open questions and no version stamp. It did not ship in C# 14 (.NET 10, Nov 2025). In 2026 it is a preview language feature for C# 15 riding the .NET 11 previews: `public union Pet(Cat, Dog, Bird);` lowers to a compiler-generated struct implementing System.Runtime.CompilerServices.IUnion, with implicit case conversions and exhaustive pattern matching; the union types were introduced around .NET 11 Preview 2 with IDE support improved in Preview 3 (InfoQ), and Maarten Balliauw's June 2026 write-up stresses it 'is still evolving before the final release' (GA Nov 2026). The .NET 11 BCL ships UnionAttribute/IUnion scaffolding, and STJ support for these C# unions (PR #128162, Preview 6) is a separate, source-gen-capable mechanism from the reflection-only F# DU converter — the two features share the 'discriminated union' name but are distinct type systems with distinct serializer paths.

Sources:

- <https://github.com/dotnet/csharplang/blob/main/proposals/unions.md>
- <https://www.infoq.com/news/2026/04/dotnet-11-preview-3/>
- <https://blog.maartenballiauw.be/posts/2026-06-16-discriminated-unions-in-csharp-for-real-this-time/>
- <https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-11/libraries>
