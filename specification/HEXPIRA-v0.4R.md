# Hexpira 2D Code — binary format specification

**Version:** 0.4R-draft  
**Date:** 2026-09-17  
**Editor:** lucid-forge  
**License:** Apache-2.0

## 1. Status and scope

This document defines the complete logical encoding of a Hexpira v0.4R symbol,
from a Unicode text payload to one binary value per hexagonal cell, and the
inverse decoding operation.

It does not standardize camera hardware, image thresholding, finder detection,
lens correction, printing resolution, or cryptographic authenticity.

Hexpira v0.4R is experimental. It is not compatible with v0.3, v0.4, QR Code,
Data Matrix, or Aztec Code.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are
to be interpreted as interoperability requirements.

## 2. Conventions

- Integers are unsigned unless stated otherwise.
- Byte fields are shown in hexadecimal with the most significant byte first.
- Bits inside bytes are emitted most-significant bit first, except for the
  LZSS token flags explicitly described in section 9.
- Array indices start at zero.
- `floor`, `ceil`, `abs`, `min`, and `max` have their usual mathematical
  meanings.
- `round(x)` for non-negative `x` means `floor(x + 0.5)`.
- `uint32(x)` keeps the low 32 bits and interprets them as 0…2³²−1.
- `imul32(a,b)` is multiplication modulo 2³².

## 3. Symbol geometry

### 3.1 Axial coordinates

Every cell has integer axial coordinates `(q,r)`. Define:

```text
s = -q-r
distance((q1,r1),(q2,r2)) =
  max(abs(q1-q2), abs(r1-r2), abs((q1-q2)+(r1-r2)))
```

A symbol of radius `R` contains every cell whose distance from `(0,0)` is at
most `R`.

```text
cell_count(R) = 1 + 3R(R+1)
```

Version 0.4R requires `R >= 8`. The reference implementation limits `R` to
200. Decoders SHOULD impose an implementation-specific upper bound before
allocating memory.

### 3.2 Normative spiral order

Physical cells are enumerated as follows:

1. emit `(0,0)`;
2. for each ring `k = 1…R`, start at `(-k,k)`;
3. walk `k` steps in each direction, in this order:

```text
(+1, 0)
(+1,-1)
( 0,-1)
(-1, 0)
(-1,+1)
( 0,+1)
```

Emit the current cell before each step. This produces exactly
`1 + 3R(R+1)` coordinates. Test-vector field `symbol_bits_spiral` uses this
order.

### 3.3 Rendering geometry

The normative topology is a pointy-top hexagonal lattice. For circumradius
`u`, a cell center MAY be rendered at:

```text
x = sqrt(3) * u * (q + r/2)
y = 1.5 * u * r
```

The six vertices are at angles `60i - 30` degrees for `i = 0…5`.

A bit value of `1` is dark and `0` is light. Implementations SHOULD provide a
uniform light quiet area of at least two cell circumradii around the outermost
rendered hexagons. This quiet-area recommendation is not yet backed by a final
print-quality standard.

## 4. Finder flowers and reserved cells

Let `k = R - 2`. Four finder centers are defined in this order:

```text
F0 = ( 0,-k)
F1 = ( k,-k)
F2 = ( 0, k)
F3 = (-k, k)
```

Every cell at distance 0, 1, or 2 from a finder center is reserved. Each finder
therefore reserves 19 cells. At `R >= 8`, the four sets do not overlap.

For `F0`, `F1`, and `F2`:

```text
distance 0 or 1 → bit 1
distance 2      → bit 0
```

For orientation finder `F3`:

```text
distance 0      → bit 0
distance 1      → bit 1
distance 2      → bit 0
```

Thus three finders have a filled seven-cell core and the fourth has a light
center surrounded by six dark cells.

`data_cells(R)` is the normative spiral list with all 76 finder cells removed.

## 5. Ordered data-cell mapping

Payload bits are not written to `data_cells` in spiral order. The following
ordering separates format copies and disperses adjacent code bits.

Given `all = data_cells(R)`:

1. `sync_cells` are `all[0…31]`.
2. Mark those cells used.
3. `inner_format_cells` are the first 15 still-unused cells in `all` order.
4. From the remaining cells, sort by:
   1. decreasing distance from `(0,0)`;
   2. increasing `q`;
   3. increasing `r`.
5. `outer_format_cells` are the first 15 cells from that sorted list.
6. For every remaining cell `(q,r)`, compute:

```text
h = uint32(
      imul32(q + R + 211,       73856093)
  XOR imul32(r + R + 307,       19349663)
  XOR imul32(q + r + 2R + 401,  83492791)
)
```

7. Sort remaining cells by increasing `h`, then increasing `q`, then
   increasing `r`.

The final ordered list is:

```text
ordered_data_cells =
  sync_cells || inner_format_cells || outer_format_cells || remaining_cells
```

The first 62 positions form the unmasked prefix:

```text
positions  0…31  synchronization word
positions 32…46  first format word
positions 47…61  second format word
positions 62…     masked RS code stream and tail filler
```

## 6. Synchronization word

The 32 synchronization bits are:

```text
11100101100100010110100011101011
```

They are not masked and are not protected by Reed–Solomon. The reference
decoder accepts at most four mismatching synchronization cells after geometry
has been established.

## 7. Format information: duplicated BCH(15,5)

### 7.1 Five data bits

The format value is:

```text
data5 = (profile_id << 3) | mask_id
```

Profiles:

| `profile_id` | Name | Target parity rate |
|---:|---|---:|
| 0 | compact | 0.10 |
| 1 | balanced | 0.20 |
| 2 | robust | 0.35 |
| 3 | reserved | — |

`mask_id` is 0…7 and selects one predicate from section 12.

### 7.2 BCH word generation

The generator polynomial is represented by hexadecimal `0x537`, degree 10.

```text
remainder = polynomial_remainder(data5 << 10, 0x537)
word15    = ((data5 << 10) | remainder) XOR 0x5412
```

`word15` is emitted from bit 14 to bit 0. The same 15-bit word is written to
both format locations.

Polynomial remainder can be calculated with:

```text
v = data5 << 10
while bit_length(v) >= 11:
    v = v XOR (0x537 << (bit_length(v) - 11))
remainder = v
```

### 7.3 Format decoding

A decoder evaluates every legal `(profile_id,mask_id)` pair, computes its BCH
word, and measures Hamming distance independently against both copies.

```text
score(candidate) = min(distance(copy1), distance(copy2))
```

Select the candidate with the lowest score and reject the symbol when the score
is greater than 3. On an exact tie, the reference decoder chooses the smallest
five-bit `data5` value.

Using the best of two spatially separated copies allows either copy to be
destroyed while the other remains readable.

## 8. Text frame

The RS-protected data begins with a variable-length frame:

| Field | Size | Description |
|---|---:|---|
| magic | 1 byte | `0xA5` |
| codec | 1 byte | 0 segmented, 1 raw UTF-8, 2 LZSS |
| original length | varint | decoded UTF-8 byte length |
| payload bit length | varint | exact number of codec payload bits |
| codec payload | variable | zero-padded to a byte boundary |
| CRC-32 | 4 bytes | big-endian CRC over every previous frame byte |

### 8.1 Varint

Varints use unsigned little-endian base-128 encoding. Each byte contains seven
value bits; bit 7 is one when another byte follows.

```text
do:
    byte = value & 0x7F
    value = floor(value / 128)
    if value != 0: byte |= 0x80
    emit byte
while value != 0
```

A v0.4R decoder MUST reject a varint longer than five bytes.

### 8.2 CRC-32

The checksum is CRC-32/ISO-HDLC (commonly called CRC-32 or IEEE CRC-32):

```text
reflected polynomial = 0xEDB88320
initial value        = 0xFFFFFFFF
final XOR            = 0xFFFFFFFF
check("123456789")   = 0xCBF43926
```

The 32-bit result is stored most-significant byte first. A decoder MUST verify
the CRC after RS correction and before returning text.

## 9. Payload codecs

An encoder computes all three legal representations and chooses the frame with
the smallest byte length. On an exact tie, the reference encoder prefers:

```text
segmented → raw UTF-8 → LZSS
```

Alternative valid encoder choices remain decodable; identical symbol output is
required only for reference-encoder conformance.

### 9.1 Codec 0: segmented text

Text is processed as Unicode scalar values. Each segment begins with:

| Field | Bits | Meaning |
|---|---:|---|
| mode | 2 | 0 numeric, 1 alphanumeric, 2 Latin, 3 UTF-8 bytes |
| units minus one | 8 | 1…256 characters or bytes |

Segment fields and data are concatenated without byte alignment.

#### Mode 0: numeric

`units` is the number of ASCII digits. Encode three digits as their integer
value in 10 bits. A final pair uses 7 bits; a final single digit uses 4 bits.

#### Mode 1: alphanumeric

The 45-character alphabet is:

```text
0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ $%*+-./:
```

For a pair with indices `a,b`, emit `45a+b` in 11 bits. A final single
character is emitted in 6 bits.

#### Mode 2: Latin

Each character is one 7-bit index. Indices 0…94 are Unicode code points
U+0020…U+007E in ascending order. Indices 95…127 are:

| Index | Character | Code point | Index | Character | Code point |
|---:|:---:|---:|---:|:---:|---:|
| 95 | é | U+00E9 | 112 | Ê | U+00CA |
| 96 | è | U+00E8 | 113 | Ë | U+00CB |
| 97 | ê | U+00EA | 114 | À | U+00C0 |
| 98 | ë | U+00EB | 115 | Â | U+00C2 |
| 99 | à | U+00E0 | 116 | Ù | U+00D9 |
| 100 | â | U+00E2 | 117 | Û | U+00DB |
| 101 | ä | U+00E4 | 118 | Ü | U+00DC |
| 102 | ù | U+00F9 | 119 | Ô | U+00D4 |
| 103 | û | U+00FB | 120 | Î | U+00CE |
| 104 | ü | U+00FC | 121 | Ç | U+00C7 |
| 105 | ô | U+00F4 | 122 | œ | U+0153 |
| 106 | ö | U+00F6 | 123 | Œ | U+0152 |
| 107 | î | U+00EE | 124 | € | U+20AC |
| 108 | ï | U+00EF | 125 | ’ | U+2019 |
| 109 | ç | U+00E7 | 126 | – | U+2013 |
| 110 | É | U+00C9 | 127 | … | U+2026 |
| 111 | È | U+00C8 |  |  |  |

#### Mode 3: UTF-8 byte segment

`units` is the number of UTF-8 bytes, not characters. Emit each byte in 8 bits.
The reference encoder creates segment boundaries only between Unicode scalar
values, so each segment is independently valid UTF-8. Decoders MUST use strict
UTF-8 and reject malformed input.

#### Reference segmentation optimizer

The reference encoder uses dynamic programming over Unicode scalar positions.
For every position it considers every valid run up to 256 units in modes 0, 1,
and 2, plus every scalar-aligned byte run whose UTF-8 length is at most 256.
The cost is the 10-bit segment header plus encoded data bits. A state is updated
only for a strictly smaller cost; modes are considered in order 0, 1, 2, 3.

A third-party encoder MAY use another optimizer or segmentation as long as it
emits valid segments.

### 9.2 Codec 1: raw UTF-8

The codec payload is the original text encoded as strict UTF-8, eight bits per
byte. Its payload bit length is exactly `original_length * 8`.

### 9.3 Codec 2: LZSS

LZSS output is a sequence of groups. Each group begins with one flag byte and
contains up to eight tokens. Flag bit `i` (least-significant bit is token 0) is:

```text
0 → one literal byte follows
1 → a two-byte match follows
```

A match has a 12-bit backward offset and a length from 3 to 18:

```text
first byte  = offset >> 4
second byte = ((offset & 0x0F) << 4) | (length - 3)
```

Valid offsets are 1…4095 and MUST NOT exceed the number of bytes already
decoded. Match copying permits overlap.

The decoder stops when it has produced `original_length` bytes and then applies
strict UTF-8 decoding.

The canonical compressor uses a three-byte hash key, remembers at most the 64
most recent positions per key, searches newest to oldest, selects the longest
match up to 18 bytes, and emits a match only when its length is at least 3.
Other valid LZSS tokenizations are permitted.

## 10. Reed–Solomon layout

Let:

```text
N       = number of data cells after removing finders
prefix  = 62 bits
total   = floor((N - prefix) / 8)       // total RS code bytes
blocks  = ceil(total / 255)
base    = floor(total / blocks)
extra   = total mod blocks
```

Block `i` has total length:

```text
length[i] = base + 1, if i < extra
            base,     otherwise
```

For profile parity rate `rate` from section 7:

```text
nsym = max(4, min(base - 1, round(base * rate)))
```

Every block uses the same `nsym`. A layout is invalid if any block length is
less than or equal to `nsym`.

Data capacity is:

```text
sum(length[i] - nsym)
```

The encoder chooses the smallest radius `R >= 8` whose data capacity can hold
the complete frame.

### 10.1 GF(256)

Reed–Solomon arithmetic uses GF(256) with:

```text
primitive polynomial = 0x11D
primitive element    = 0x02
```

For `nsym` parity bytes, the generator polynomial is:

```text
g(x) = product for i=0…nsym-1 of (x + α^i)
```

Encoding is systematic: each block is `data || parity`.

### 10.2 Frame padding

The frame is padded to the total RS data capacity with deterministic bytes.
Initialize:

```text
state = CRC32(frame)
if state == 0: state = 1
```

For each required padding byte:

```text
state = uint32((state >>> 1) XOR (0xEDB88320 if (state & 1) else 0))
emit state & 0xFF
```

The decoder ignores these bytes after the frame length and CRC identify the end
of the frame.

### 10.3 Block construction and interleaving

Consume `length[i] - nsym` padded data bytes for each block, encode each block,
then interleave:

```text
for column = 0 … max(length)-1:
    for block = 0 … blocks-1:
        if column < length[block]:
            emit block[block][column]
```

Convert the resulting bytes to bits, most-significant bit first.

The decoder reverses this operation before RS correction.

## 11. Tail filler

Construct the pre-mask stream as:

```text
SYNC || 30 zero placeholders || interleaved_RS_bits
```

Continue the same `state` left after frame-byte padding. Until the stream has
one bit for every ordered data cell, update the state using the recurrence in
section 10.2 and append `state & 1`.

Tail filler is not part of an RS codeword and is ignored by the decoder.

## 12. Masks

Positions 0…61 are never masked. For every later ordered data position `i`
with cell `(q,r)`, XOR the bit with `mask(mask_id,q,r,i)`:

| ID | Predicate producing 1 |
|---:|---|
| 0 | never |
| 1 | `(q+r) mod 2 != 0` |
| 2 | `abs(q) mod 3 == 0` |
| 3 | `abs(r) mod 3 == 0` |
| 4 | `(abs(q*r)+i) mod 2 != 0` |
| 5 | `(abs(q)+2*abs(r)+i) mod 3 == 0` |
| 6 | `(i+abs(q+r)) mod 2 != 0` |
| 7 | `((i mod 3)+(abs(q*r) mod 2)) mod 2 != 0` |

After masking, replace prefix positions 32…61 with the two copies of the BCH
format word for the selected profile and mask.

Any mask is legal. The decoder MUST use the ID recovered from the format word.

### 12.1 Reference mask selection

The reference encoder evaluates all eight complete candidate streams and
selects the lowest score, choosing the lower mask ID on a tie. The score is:

1. `5 * abs(total_ones - total_bits/2)`.
2. For lines of constant `q`, constant `r`, and constant `s=-q-r`, sort cells
   along the line. Every same-bit run longer than 5 adds `4*(run-5)`.
3. For each cell, inspect neighbors `(q+1,r)`, `(q,r+1)`, `(q-1,r+1)`. If all
   three exist and equal the cell, add 2.
4. For data positions from 62 onward, divide cell centers into six angular
   sectors using the rendering coordinates in section 3.3. Each sector adds
   `1.5 * abs(sector_ones - sector_count/2)`.

This selection algorithm is recommended, not required for interoperability.

## 13. Final physical symbol

1. Initialize a map with all 76 finder bits from section 4.
2. Map the selected complete data stream, in order, onto
   `ordered_data_cells` from section 5.
3. Emit the bit of every physical cell in normative spiral order.

Every physical cell is assigned exactly one bit.

## 14. Encoding procedure

```text
input: Unicode text, profile name

1. Encode the text using codecs 0, 1, and 2.
2. Choose the candidate with the smallest complete frame size.
3. Serialize the frame and append CRC-32.
4. For R = 8 upward, derive the RS layout for the selected profile.
5. Select the first R whose RS data capacity holds the frame.
6. Deterministically pad the frame to RS data capacity.
7. Encode and interleave the RS blocks.
8. Create sync, format placeholders, RS bits, and tail filler.
9. Evaluate legal masks and choose one.
10. Insert two copies of the BCH format word.
11. Map data and finder bits to physical cells.
12. Render dark and light hexagonal cells with a quiet area.
```

## 15. Decoding procedure

This procedure begins after image processing has estimated `R` and sampled one
binary value per physical cell.

```text
input: physical bits in normative spiral order, radius R

1. Verify bit count equals 1 + 3R(R+1).
2. Reconstruct the coordinate → bit map.
3. Check finder patterns. The reference allows at most 5 mismatches.
4. Build ordered_data_cells and read the 32-bit sync word.
5. Reject when sync has more than 4 mismatches.
6. Decode the two BCH format copies; recover profile and mask.
7. Derive the unique RS layout from R and profile.
8. Unmask positions 62 onward.
9. Read exactly total*8 RS bits and pack them MSB-first.
10. Reverse interleaving and RS-decode each block.
11. Concatenate corrected data portions.
12. Parse magic, codec, original length, and payload bit length.
13. Locate and verify CRC-32.
14. Decode the selected payload codec.
15. Verify decoded UTF-8 byte length equals original length.
16. Return text only after every validation succeeds.
```

A decoder MUST reject unknown codecs, reserved profiles, impossible lengths,
invalid UTF-8, uncorrectable RS blocks, and CRC mismatches.

## 16. Image-processing guidance (informative)

The reference demo:

- thresholds luminance;
- finds three filled flowers and one ring-centered orientation flower;
- estimates a four-point projective transform;
- samples multiple pixels around each expected cell center;
- tests nearby radii when geometry is uncertain.

Future readers may use adaptive local thresholds, confidence values, temporal
fusion, or RS erasures. Such improvements do not alter this bitstream.

## 17. Conformance

A v0.4R decoder is bitstream-conforming when it decodes every normative vector
in `tests/vectors.json` and rejects corrupted frames that fail required checks.

An encoder is interoperable when its output is accepted by a conforming
decoder. It need not reproduce the reference segmentation or selected mask.

An encoder is reference-canonical when it also reproduces the complete symbol
bits and hashes in `tests/vectors.json`.

## 18. Security considerations

- CRC, BCH, and Reed–Solomon are not cryptographic mechanisms.
- A valid symbol may have been created by anyone.
- Decoders must validate lengths before allocation and cap radius and work.
- Text must be decoded with strict UTF-8.
- Applications must not execute decoded text as code or trust it as a URL.
- Authenticity requires a separate, standardized digital-signature layer.

## 19. IANA, GS1, and standards status

Hexpira has no IANA media type, GS1 allocation, ISO standard number, or official
industry identifier. Implementations must not claim such status.


