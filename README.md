# Hexpira

Hexpira is an experimental binary 2D code built on a pointy-top hexagonal
lattice. Version **0.4R** focuses on a small, deterministic format: three
payload codecs, three proportional Reed–Solomon profiles, duplicated BCH
format information, spatial interleaving, and CRC-32 validation.

> **Status:** experimental draft. Hexpira is not an ISO standard and is not
> recommended for payments, identity documents, safety-critical systems, or
> long-term archival use.

## Repository contents

```text
hexpira/
├── README.md
├── LICENSE
├── NOTICE
├── CHANGELOG.md
├── CONTRIBUTING.md
├── SECURITY.md
├── specification/
│   ├── HEXPIRA-v0.4R.md       # normative wire-format specification
│   └── ARCHITECTURE.md        # implementation boundaries and roadmap
├── reference/
│   └── web/index.html         # dependency-free reference demo
├── src/
│   ├── core/README.md         # planned portable codec modules
│   └── browser/README.md      # planned camera/image-processing modules
└── tests/
    ├── README.md
    └── vectors.json           # cross-language conformance vectors
```

The specification and test vectors are the interoperability contract. The web
demo is a readable reference implementation, not the normative definition.

## Quick start

Open [`reference/web/index.html`](reference/web/index.html) in a recent
browser. It runs locally and has no external dependency.

The page can:

- encode text as a Hexpira v0.4R symbol;
- export a PNG;
- decode a generated image or photograph;
- simulate rotation and perspective;
- use a webcam when the browser permits camera access from the current origin.

Some browsers block webcams on `file://` pages. Image import remains available.

## Implementing Hexpira

Start with:

1. [`specification/HEXPIRA-v0.4R.md`](specification/HEXPIRA-v0.4R.md)
2. [`tests/vectors.json`](tests/vectors.json)
3. [`reference/web/index.html`](reference/web/index.html)

A conforming decoder does not have to reproduce the reference image-processing
pipeline. It must reproduce the bit-level format after sampling cells.

## Design goals

- fully local encoding and decoding;
- deterministic, documented bitstream;
- robust recovery from localized damage;
- efficient text representation without a mutable language dictionary;
- straightforward implementations in JavaScript, Rust, C, Python, Kotlin,
  Swift, and other languages;
- no network dependency in the core format.

## Non-goals for v0.4R

- compatibility with QR Code, Data Matrix, Aztec, or earlier Hexpira drafts;
- cryptographic authenticity or encryption;
- a stable standards-track promise;
- color or multi-level cells;
- guaranteed webcam support from a local file URL.

## Versioning

Version 0.4R intentionally does **not** decode Hexpira v0.3 or v0.4 symbols.
Until version 1.0, incompatible changes remain possible and will receive a new
format identifier and new conformance vectors.

## Contributing

Issues and pull requests are welcome. Please read
[`CONTRIBUTING.md`](CONTRIBUTING.md). Changes to the wire format must include a
specification update and new test vectors.

## License

Copyright 2026 lucid-forge.

Licensed under the [Apache License 2.0](LICENSE). The license includes a patent
grant from contributors for their contributions. It is not a legal opinion or
a clearance of unknown third-party patent rights.


