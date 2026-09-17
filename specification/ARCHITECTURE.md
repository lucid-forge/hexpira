# Hexpira implementation architecture

This document defines the intended repository boundaries. It is informative;
the normative format is defined in `HEXPIRA-v0.4R.md`.

## Separation of concerns

```text
Application / CLI / browser UI
              │
              ▼
      Encoder and decoder API
              │
     ┌────────┴────────┐
     ▼                 ▼
Logical codec       Image sampler
     │                 │
     └────────┬────────┘
              ▼
        Cell bit array
```

The logical codec must not depend on a canvas, camera, DOM, image library, or
network API. Image processing ends when it produces one bit per physical cell
in normative spiral order. This boundary makes cross-language conformance
practical.

## Planned portable modules

```text
src/core/
├── constants.*        alphabets, sync word, profiles, mask identifiers
├── geometry.*         axial coordinates, rings, finders, cell ordering
├── bits.*             MSB-first packing, varints, CRC-32
├── bch.*              duplicated BCH(15,5) format word
├── gf256.*            finite-field arithmetic over polynomial 0x11D
├── reed-solomon.*     systematic RS encoder and decoder
├── segments.*         numeric, alphanumeric, Latin, and byte segments
├── lzss.*             deterministic 4 KiB-window payload codec
├── frame.*            v0.4R frame serialization and validation
├── masks.*            eight mask predicates and reference scoring
├── encoder.*          payload → physical cell bits
└── decoder.*          physical cell bits → validated payload
```

File extensions are language-specific. Ports should preserve these conceptual
boundaries even when several modules are compiled into one unit.

## Planned browser modules

```text
src/browser/
├── render.*           cell coordinates → Canvas/SVG/PNG
├── threshold.*        global and local luminance classification
├── finders.*          finder candidate detection and orientation
├── homography.*       projective transform and cell sampling
├── camera.*           video loop and temporal frame management
└── demo.*             DOM integration only
```

Browser code may use confidence values and multiple image frames. These are not
part of the v0.4R wire format.

## Public core API target

The future portable library should expose a small API equivalent to:

```js
encodeText(text, {
  profile: "compact" | "balanced" | "robust"
}) -> {
  version,
  radius,
  cells,          // [{ q, r, bit }] in physical spiral order
  profile,
  mask,
  codec
}

decodeCells(bits, radius) -> {
  version,
  text,
  profile,
  mask,
  codec,
  correctedBytes,
  formatDistance
}
```

Binary payload support should be added only under a new documented codec or
frame type. It must not reinterpret the v0.4R text frame.

## Error handling

Core decoders should distinguish at least:

- invalid radius or cell count;
- finder mismatch;
- synchronization mismatch;
- invalid BCH format word;
- unsupported profile or codec;
- uncorrectable Reed–Solomon block;
- malformed varint or segment;
- invalid UTF-8;
- CRC-32 mismatch.

Applications may present a simpler message to users, but library callers need
the structured cause for diagnostics and testing.

## Normative versus replaceable components

Normative:

- coordinates and physical cell order;
- finder values and reserved cells;
- sync and format bits;
- frame, codecs, RS parameters, interleaving, masks, and CRC;
- mapping between the ordered data stream and physical cells.

Replaceable:

- segmentation optimizer, provided the emitted segments are valid;
- selected mask, provided its identifier is encoded correctly;
- Reed–Solomon implementation strategy;
- raster rendering details that preserve the same cell lattice;
- finder detection, thresholding, perspective estimation, and sampling.

## Release discipline

1. Freeze each format version once published.
2. Change the magic/version identifier for incompatible wire changes.
3. Add test vectors before merging a format change.
4. Keep camera changes separate from codec changes.
5. Benchmark density and read success using identical payloads and damage.
6. Do not silently change alphabets, mask formulas, field widths, or tie rules.


