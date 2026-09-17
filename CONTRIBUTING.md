# Contributing to Hexpira

Thank you for helping make Hexpira easier to implement and evaluate.

## Before opening a pull request

1. Open an issue for any proposed wire-format change.
2. Keep the reference implementation dependency-free unless a dependency has a
   clear interoperability benefit.
3. Update the specification when observable encoding or decoding behavior
   changes.
4. Add or update a conformance vector for every format change.
5. Preserve decoding of all vectors belonging to the same format version.
6. Separate image-processing improvements from bitstream changes.

## Compatibility rule

An implementation may optimize mask selection, Reed–Solomon arithmetic, or
image processing. It is compatible when it can decode the normative vectors
and emits symbols accepted by a conforming v0.4R decoder.

## Commit and pull-request scope

Prefer small changes with one purpose. Explain:

- what changed;
- whether the wire format changed;
- which tests were added;
- any density or robustness measurement used to justify the change.

## Licensing contributions

Unless explicitly stated otherwise, contributions submitted to this project
are licensed under Apache-2.0 as described in section 5 of the license.


