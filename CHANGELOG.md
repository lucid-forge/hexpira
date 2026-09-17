# Changelog

## v0.4R — Hexpira naming

- Renamed the project and symbology from Hexpira to Hexpira.
- Kept the v0.4R wire format unchanged; only project-facing names and test text were updated.

All notable format and repository changes are documented here.

## 0.4R-draft — 2026-09-17

- reduced the payload codecs to segmented text, raw UTF-8, and LZSS;
- reduced Reed–Solomon choices to 10%, 20%, and 35% redundancy profiles;
- added two spatially separated BCH(15,5) format words;
- restored CRC-32 for final frame validation;
- removed exhaustive profile and mask discovery from the decoder;
- retained four finder flowers for camera robustness;
- documented the complete cell ordering and binary format;
- added cross-language conformance vectors.

