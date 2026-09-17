# Conformance vectors

`vectors.json` contains payloads and complete v0.4R symbols in physical spiral
order. A third-party implementation should:

1. decode every `symbol_bits_spiral` value to the listed text;
2. encode the payload and verify the frame, profile, radius, and format word;
3. reproduce the complete symbol when it uses the reference optimizer and mask
   scoring algorithm;
4. accept alternative valid symbols that use another legal segmentation or
   mask.

`symbol_bits_packed_hex` packs the spiral bit string MSB-first and pads the last
byte with zero bits. `symbol_bits_sha256` hashes those packed bytes.

Complete symbol equality is a reference-encoder test, not a requirement for
all conforming encoders. Successful cross-decoding is the interoperability
requirement.


