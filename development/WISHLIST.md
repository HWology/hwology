# WISHLIST / requirements draft — HWology

Informal, and deliberately ahead of the design: this is the requirements draft that feeds the
event-modeling pass, which feeds codegen. Not a parking lot and not commitments.

Evidence convention — HWology has no 13-tool survey behind it the way the transcript viewer does;
the evidence base is the owner's own hardware experience (the "Hard-won requirements" in
[development/memory/project.md](memory/project.md)) plus the twelve verified reports in
[development/research/2026-07-02/](research/2026-07-02/README.md). So:
`why:` names the frustration or evidence that produced the requirement; `open:` marks a genuine
undecided the modeling pass has to resolve.

Settled design does not live here — promote it to [development/architecture.md](architecture.md)
and record the decision in [development/memory/project.md](memory/project.md).

## Mobile capture & identification (owner brain-dump 2026-08-22)

- PWA on the phone; the camera is a first-class input, not an upload form.
  why: the facts this catalog exists to hold — per-slot electrical width, silkscreen labels,
  real port layout — are mostly only knowable with the board in your hands. The capture device
  is a phone at the bench, not a desktop.
- Photograph a device and attach the photos to its record — an owned unit **or** a public catalog
  entry. One flow, both targets, chosen at capture time.
- SCAN TO FIND: shoot a barcode, or the serial sticker, and get back a ranked list of matching
  records — rather than having to pick the device from a list first. The camera is an entry
  point, not only an attachment tool.
- Barcode / MPN → model is the strong identifier path. UPC/EAN on the retail box and the MPN
  silkscreened on the board are structured and public, and they resolve to a catalog model.
  why: works on hardware you have never seen before, including in a shop. It should feed the
  *same* identity/alias resolver as milestone 4, not a second one bolted on.
- Serial → unit is the weak path, and only ever finds your own records.
  why: vendor serial formats are proprietary and do not publicly decode to a model; the endpoints
  that decode them are gated warranty lookups, and several vendors bot-block outright (see
  [development/memory/data-sources.md](memory/data-sources.md)). A serial realistically matches
  "the unit I already logged" — an owned-instance key, not a catalog key. Build it, don't
  over-promise it.
- Ranked candidates that a human confirms. NEVER an automatic bind.
  why: one misread digit silently attaching evidence to the wrong model manufactures exactly the
  kind of unsourced fact the whole provenance model exists to prevent.
- Owner photos rank as `measured` — the top of `measured > correction > manual-PDF > vendor-page
  > aggregator`.
  why: the three-M.2-slot episode in the hard-won requirements (two x1, one x2, discovered only
  by testing) is a photograph of a silkscreen away from being a citable fact.
- Per-photo publish decision, private by default.
  why: unlike scraped HTML, manual PDFs, and firmware, photos the owner takes are the owner's
  copyright — so they *can* go to the public bucket under CC BY-SA 4.0. Public/private becomes a
  flag rather than the blanket prohibition in the licensing tiers.
- Redaction before publish is a human step: serials, MAC labels, service tags, whatever is in the
  background of the shot. No capture auto-publishes.
- EXIF: keep the timestamp, strip GPS. why: this repo and its dataset are public.
- Offline capture queue that syncs later.
  why: capture happens in a rack room or a shop with no signal. The observation's retrieved-at
  must be capture time, never upload time.
- OCR / LLM pre-fill from a spec label or a BIOS version screen arrives as a *suggestion* citing
  the photo as its source, human-confirmed.
  why: identical discipline to the lane-sharing PDF extraction workflow already open in
  architecture.md, and for the same reason — never a silent write.
- A prompted shot list, not a freeform album: board top, rear IO, each PCIe/M.2 slot silkscreen,
  BIOS version screen, box label, chipset area with the heatsink off.
  why: named shots are queryable evidence; an album is just pictures.
- open: platform. `getUserMedia` inside an installed iOS PWA has historically been the fragile
  path; `<input type="file" accept="image/*" capture="environment">` is the boring reliable one.
  Decide whether "PWA" here means installed-app UX or a mobile page that can take pictures.
  Same split governs scanning: `BarcodeDetector` on Chromium/Android, ZXing-WASM fallback.
- open: stack. A Blazor WASM PWA would share the Fun.Blazor GUI codebase (milestone 6) with
  capture as a route — but Fun.Blazor + WASM PWA + camera interop is unverified. That is a
  research doc, not an assumption.

### Seed for the event-modeling pass

- EVENTS vs PROJECTIONS. `PhotoCaptured` is a real event — it happened, at a time, by a person,
  yielding a blob hash. So is `IdentityConfirmed`: the human accepting a candidate match. The
  **candidate list is not an event** — it is a query result over the identity index,
  recomputable, never stored.
- open: does a scan that matches *nothing* deserve an event? It names a device the catalog lacks,
  which is real signal for ingestion priorities — or it is just telemetry. Decide in the modeling
  pass; it is the difference between a demand signal and log noise.
- AGGREGATES, and this is the question that shapes the whole feature: is an **owned unit** an
  aggregate distinct from a **device model**? A photo of my board is evidence about the model,
  but the serial, the purchase date, and "which slot actually carried the x2 link" are facts about
  the unit. Two aggregates with a reference between them, or one model aggregate with an optional
  instance facet — this decides where the personal inventory ends and the public catalog begins.
- VIEW STATE vs DATA: hero-photo selection, display crop, and ordering are presentation state and
  must not round-trip into the normalized observation. The bytes, their sha256, and EXIF capture
  time are data.
- NOT DEFERRABLE: event types are versioned and forever, and milestone 1 is a format freeze. If
  photo and identity events are wanted at all, sketch their shape during that freeze — even if
  nothing implements them for a year. Everything else here can wait for milestones 5–6.

## Later / maybe

- Scan a box in a store and check it against a saved requirement list before buying — the
  project's founding use case, delivered at the point of purchase.
- Photos as a revision-diffing aid: same model, two board revisions, spot the physical delta.
