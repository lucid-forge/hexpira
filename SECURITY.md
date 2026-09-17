# Security policy

Hexpira v0.4R provides error correction and accidental-corruption detection.
It does **not** provide authenticity, confidentiality, authorization, or
protection against a maliciously generated symbol.

## Supported version

Only the current `0.4R-draft` branch of the format is maintained in this
repository. There is no long-term support promise before version 1.0.

## Reporting a vulnerability

Please use GitHub private vulnerability reporting when enabled for the
`lucid-forge/hexpira` repository. Do not include personal, secret, or production
data in a report.

Useful reports include malformed inputs that cause denial of service, decoder
confusion, out-of-bounds access in ports, or a CRC/BCH validation bypass.

## Decoder requirements

Implementations should enforce input limits before allocating memory, reject
unknown codecs and profiles, cap the radius, use strict UTF-8 decoding, and
validate Reed–Solomon and CRC-32 before returning a payload.


