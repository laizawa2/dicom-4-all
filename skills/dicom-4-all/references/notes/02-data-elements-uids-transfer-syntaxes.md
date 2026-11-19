# DICOM Notes 2: Data Elements, UIDs and Transfer Syntaxes

## Data Elements, VRs and Encoding

- Data element = Tag (group,element; two unsigned 16-bit integers, written (gggg,eeee) in hex) + VR + Value Length + Value Field. Elements within a data set (including the data set of each sequence item) are sorted ascending by tag; out-of-order elements are non-conformant.
- Even tag groups are standard; odd groups are private (e.g. (0903,xxxx)). Big Endian syntax is retired [verify].
- Data Dictionary (PS3.6): >2000 attributes; 27 VR types (approximately; the current count is slightly higher [verify]).
- VR = data type/format with allowed characters. Examples: PN person name (all name attributes: Patient Name, Referring Physician Name, etc.), DA date (YYYYMMDD), TM time, SQ sequence, UI UID, OB other byte, UN unknown, ST short text. Example tag: Patient Name (0010,0010); Pixel Spacing (0028,0030).
- Enumerated Values and Defined Terms are constrained in PS3.3. Patient Sex: M, F, O only; "Male"/"male" is invalid. Modality is a Defined Term.
- Value Length: unsigned integer in bytes. Length limits per VR are often overlooked by software and cause incompatibility. Numeric VRs have fixed sizes independent of platform.
- Even-length rule: every value has even length. Odd text values are padded with a trailing space (UI, and byte VRs like OB, are padded with a NUL byte). Example: "Smith^Joe" (9 chars) is stored with a trailing space, length 10.
- Always trim trailing padding (space/NUL) when handing decoded values to non-DICOM systems (RIS, PACS DB, HL7); untrimmed "Smith^Joe " can fail matching or create duplicate patients.
- Implicit VR: no VR in the stream; VR looked up from the dictionary. Explicit VR: VR stored in the element. Which is used is determined by the negotiated transfer syntax.
- Implicit VR Little Endian: the default transfer syntax; every DICOM implementation must support it (UID 1.2.840.10008.1.2). Layout: tag (2+2 bytes) + 4-byte length + value, multi-byte numbers least-significant byte first. Example: (0010,0010) length 0x0000000A "Smith^Joe " encodes as 10 00 10 00 0A 00 00 00 + value. Elements are simply concatenated; fixed-size header fields allow splitting.
- Byte-order mismatch example: 0x007F read as 0x7F00 gives 32,512 instead of 127.
- Pixel Data (7FE0,0010): holds the image; often about 95% of object size.
- Sequences (VR SQ): a sequence is a list of items, each an ordered, non-repeating data set (nesting 5-6 deep occurs in SR and radiotherapy). Length may be explicit or undefined (0xFFFFFFFF) closed by delimitation items ((FFFE,E0DD) sequence delimiter, (FFFE,E00D) item delimiter, items start with (FFFE,E000)). In IOD listings, nesting is shown with ">" prefixes.
- Nested data sets normally use the same encoding as the enclosing set. Exception: a sequence carried as VR UN is encoded Implicit VR Little Endian (arises when an application received implicit data, did not know the true VR of a private/new element, and rewrites in explicit form).
- Data set uses: commands, composite instances, normalized instances, files.
- Message structure: Command Set (command elements, always implicit VR LE) followed by an optional Data Set.
- Example type mismatch: (0040,0244) Performed Procedure Step Start Date is DA; (0040,0245) Performed Procedure Step Start Time is TM.

### Element encoding
- Element = Tag (group, element; 2x uint16) + [VR] + Length + Value. Value length is always even.
- Implicit VR: Tag, 4-byte Length, Value. Receiver finds VR via its own data dictionary (e.g. (0010,0010) = PN). Implicit VR Little Endian is the default Transfer Syntax every implementation must support.
- Explicit VR: Tag, 2-char VR, then length:
  - Short form (most VRs): 2-byte length.
  - Long form (OB, OD, OF, OL, OV, OW, SQ, SV, UC, UN, UR, UT, UV; that is, every VR outside the short-form set AE, AS, AT, CS, DA, DS, DT, FL, FD, IS, LO, LT, PN, SH, SL, SS, ST, TM, UI, UL, US): 2 reserved bytes (0000) then 4-byte length.
- Explicit VR is needed to carry VRs not in the dictionary (e.g. private attributes).
- Example: (0010,0010) PN length 10 value "Smith^Joe " (trailing space pads to even length).
- Odd-length values are padded: space for text VRs, NULL (0x00) for UI and OB.

### Common VRs
| VR | Name | Max length / size |
|---|---|---|
| CS | Code String | 16 |
| SH | Short String | 16 |
| LO | Long String | 64 |
| ST | Short Text | 1024 |
| LT | Long Text | 10240 |
| UT | Unlimited Text | 4294967294 |
| AE | Application Entity | 16 |
| PN | Person Name | 64 per component group |
| UI | UID | 64 |
| DA | Date | 8 (YYYYMMDD) |
| TM | Time | 14 bytes max; HHMMSS.FFFFFF, with trailing components (minutes, seconds, fraction) omittable from the right |
| DT | Date Time | 26 |
| AS | Age String | 4 (e.g. 018M) |
| IS | Integer String | 12 |
| DS | Decimal String | 16 |
| SS / US | Signed / unsigned short | 16-bit binary |
| SL / UL | Signed / unsigned long | 32-bit binary |
| AT | Attribute Tag | pair of uint16 |
| FL / FD | Float / double | 32 / 64-bit |
| OB / OW / OF | Other Byte / Word / Float | byte, 16-bit, 32-bit streams |
| SQ | Sequence of Items | - |
| UN | Unknown | bytes with unknown encoding |
- Source cites 27 VRs; current standard defines more (adds OD, OL, OV, SV, UV, UC, UR) [verify].
- Examples (synthetic): DA "20050822"; TM "183200.00" = 18:32:00; DT "20050812183000.00"; AS "018M" = 18 months.

### Value Multiplicity (VM)
- VM states how many values an element holds (1, 1-n, 2, ...). Example: Other Patient Names (0010,1001) is 1-n.
- Binary VRs: values concatenated, split by the fixed size of the VR.
- Text VRs: values separated by backslash (\), e.g. "Name1\Name2" gives VM 2. Never use backslash for other purposes inside values (file paths, dates).

### Value-format checks (worked cases)
- Patient Sex defined terms M, F, O: single character padded with a space to length 2.
- Accession Number is SH: max 16 characters; longer content is invalid.
- Person Name uses caret separators: Family^Given^Middle^Prefix^Suffix. "Smith,John" is syntactically valid but parsed as a single family name; trailing empty components (Smith^^^ , Smith^^, Smith^, Smith) are all legal. Limits: up to three component groups (alphabetic=ideographic=phonetic) separated by "=", each at most 64 characters with five components; backslash is not allowed.

### Private attributes
- Odd group numbers; vendor reserves a block via a Private Creator element and uses another block/group if one is taken (e.g. group 0013).
- Acceptable only where the standard has no suitable attribute; receivers from other vendors cannot interpret them. Vendors should document them in the Conformance Statement.
- Devices may also place standard attributes in IODs that do not define them; that makes the SOP Class "Standard Extended".

### Data Dictionary
- PS3.6 is the registry of standard attributes (tag, name, VR, VM, retired flag).

- Data element = tag (group,element), VR (data type/format), value length, value field. (VR is explicit only in Explicit VR syntaxes.)

### Pixel storage attributes
| Attribute | Tag | Meaning |
|---|---|---|
| Samples per Pixel | (0028,0002) | Number of planes/samples per pixel: 1 for monochrome and palette color, 3 for RGB/YBR. Other values allowed but undefined meaning. |
| Photometric Interpretation | (0028,0004) | How samples are interpreted (see below) |
| Rows / Columns | (0028,0010) / (0028,0011) | Matrix size |
| Pixel Aspect Ratio | (0028,0034) | Vertical size \ horizontal size as two integers |
| Bits Allocated | (0028,0100) | Container size per sample (cell), 8 or 16 typical; >= Bits Stored |
| Bits Stored | (0028,0101) | Bits actually carrying the sample value |
| High Bit | (0028,0102) | Position (bit 0 = least significant) of the top stored bit; shall be one less than Bits Stored [NORMATIVE: PS3.5 8.1] |
| Pixel Representation | (0028,0103) | 0 = unsigned, 1 = two's complement |
| Planar Configuration | (0028,0006) | Present only if Samples per Pixel > 1 (shall not be present otherwise). 0 = color-by-pixel (R1G1B1R2G2B2...), 1 = color-by-plane. YBR_FULL_422, YBR_PARTIAL_420, YBR_ICT, YBR_RCT require 0; RGB and YBR_FULL allow 0 or 1; compressed syntaxes that reorganize components may make it ignorable [NORMATIVE: PS3.3 C.7.6.3] |
| Pixel Data | (7FE0,0010) | VR is OB or OW depending on the negotiated Transfer Syntax (PS3.5 8.1 and Annex A); Type 1C: required unless Pixel Data Provider URL (0028,7FE0) is present |

- All planes of an image share Rows, Columns, Bits Allocated, Bits Stored, High Bit, Pixel Representation and Pixel Aspect Ratio.
- Pixel cell = container holding one sample value plus any spare bits; cell width = Bits Allocated.
- Lowest stored bit = High Bit + 1 - Bits Stored. Example: Bits Stored 12, Bits Allocated 16, High Bit 11 gives bits 0-11 used, bits 12-15 spare (must be masked on read).
- Spare bits once carried overlay data; that practice is retired (last described in the 2004 edition of PS3.5). Receivers must mask them.
- Raw size = Rows x Columns x Samples per Pixel x Bits Allocated / 8. Example: 512 x 512 x 1 x 2 = 524,288 bytes; a 200-slice series is about 100 MB.
- Number of gray levels = 2^(Bits Stored).
- Multi-sample "composite pixel code" (interleaved): first-named color in the Photometric Interpretation (e.g. R in RGB) occupies the most significant bits [verify: sample ordering depends on Photometric Interpretation and Planar Configuration].
- Two's complement: the top bit of each stored sample carries the sign. A reader that treats it as unsigned inverts half the range (negative values appear very bright).

### Photometric Interpretation values
| Value | Samples | Meaning |
|---|---|---|
| MONOCHROME1 | 1 | Minimum sample value displays as white (after VOI) |
| MONOCHROME2 | 1 | Minimum sample value displays as black (after VOI) |
| PALETTE COLOR | 1 | Sample is an index into the Red/Green/Blue Palette Color Lookup Tables (0028,1101-1103, 1201-1203), required (1C) for this value |
| RGB | 3 | Red, green, blue planes |
| YBR_FULL | 3 | Y luminance + CB, CR chrominance; black = Y 0; no color = CB/CR at half scale (128 for 8-bit) |
| YBR_FULL_422 | 3 | YBR_FULL with horizontally subsampled chrominance; Rows/Columns describe luminance and must be even for native encoding |
| YBR_PARTIAL_420, YBR_ICT, YBR_RCT | 3 | Other 3-sample values defined for compressed syntaxes (subsampled partial range; JPEG 2000 irreversible/reversible color transforms) |
| HSV, ARGB, CMYK, YBR_PARTIAL_422 | - | Retired |

- Monochrome and PALETTE COLOR require Samples per Pixel 1; RGB and all YBR values require 3 (Samples per Pixel 4 is retired). Other Photometric Interpretation values are permitted only if the Transfer Syntax supports them, with no Standard-defined meaning.

- RGB to YCbCr (8-bit; matches the ITU-R BT.601 luma weights 0.299/0.587/0.114 with the usual Cb = (B-Y)/1.772 and Cr = (R-Y)/1.402 scaling, plus a +128 offset for full-range 8-bit): Y = 0.2990R + 0.5870G + 0.1140B; CB = -0.1687R - 0.3313G + 0.5000B + 128; CR = 0.5000R - 0.4187G - 0.0813B + 128.
- Pixel aspect ratio example: 0.30 mm vertical, 0.25 mm horizontal = "6\5" (any equivalent integer pair). Viewers that ignore it show US, C-arm and some MR images squeezed in one direction.
- Photometric Interpretation is set per instance by the producer, not fixed by modality: many CR/DX/mammography images are MONOCHROME1 and most CT/MR are MONOCHROME2, but vendors vary. A viewer ignoring MONOCHROME1 shows an inverted image.

### Clinical Trial Identification attributes (optional module, group 0012)
| Attribute | Tag | Notes |
|---|---|---|
| Clinical Trial Sponsor Name | (0012,0010) | Entity running the trial and defining the protocol ID |
| Clinical Trial Protocol ID | (0012,0020) | Sponsor-assigned protocol identifier |
| Clinical Trial Protocol Name | (0012,0021) | Protocol title |
| Clinical Trial Site ID | (0012,0030) | Issued by the sponsor; site submitting data |
| Clinical Trial Site Name | (0012,0031) | |
| Clinical Trial Subject ID | (0012,0040) | Subject within the protocol |
| Clinical Trial Subject Reading ID | (0012,0042) | Subject identity for blinded reads |
| Clinical Trial Time Point ID | (0012,0050) | Groups studies into one time point / submission |
| Clinical Trial Time Point Description | (0012,0051) | |
| Clinical Trial Coordinating Center Name | (0012,0060) | Institution collecting images and data |

- Typical flow: images acquired at sites, collected by the coordinating center (CTCC), identifying attributes removed, CTI added, then blinded reading; results go to the sponsor and onward to the regulator.
- CTI can be added at the scanner, PACS, a site workstation, a CTCC-supplied workstation, or by the CTCC after receipt.
- Adding CTI attributes does not by itself require a new SOP Instance UID; the protocol or the de-identification process may.
- Keeping trial data in the header (instead of an external database) avoids loss or mix-up of the association with the images.
- Related attributes: Patient Identity Removed (0012,0062), De-identification Method (0012,0063) [verify tags].

### Overlays (group 60xx)
- Overlay planes are 1 bit/pixel bitmaps in repeating groups 6000-601E (even groups only): at most 16 per image.
- Historical forms: bits embedded in the high spare bits of Pixel Data (retired 2004), separate group 60xx elements (still seen), stand-alone Overlay object (retired). Preferred modern method is a Presentation State.
- Most vendors support only group 6000. Test overlay support by opening an image containing group 6000.
- Burned-in annotation (text drawn into pixels) is discouraged: destroys original pixels, can hide findings, is transformed by zoom/rotate like image data, cannot be corrected (e.g. wrong patient name), and may force conversion to Secondary Capture for color marks.
- Proprietary annotation formats vanish when images are viewed on another vendor's workstation.

## UIDs

- UIDs (VR UI) identify SOP Classes, transfer syntaxes, SOP Instances, Study/Series Instance, etc. SOP Class and transfer syntax UIDs are registered in PS3.6 under the 1.2.840.10008 root.
- Compare UIDs exactly; similar-looking old/new SOP Classes differ only by UID.
- Retrieval by URI (WADO/DICOMweb) requires the Study, Series and SOP Instance UIDs.
- Well-known UIDs: Verification SOP Class 1.2.840.10008.1.1; Implicit VR Little Endian 1.2.840.10008.1.2; Storage Commitment Push Model SOP Class 1.2.840.10008.1.20.1.

- Used for SOP Classes, SOP Instances, studies, series, transfer syntaxes, implementations, devices, coding schemes.
- Structure: `<org root>.<suffix>`; digits and periods only, max 64 characters, components numeric.
- 1.2.840.10008 is reserved for DICOM-defined UIDs (SOP Classes, transfer syntaxes, well-known instances); nobody else may use it as a root.
- Org roots are registered; suffix must be unique within that root. Suffix schemes: device/software id + date/time stamp + counter (more characters = lower collision risk). Collisions arise from clock resets or DST rollback, causing exceptions or database integrity damage. Alternative: 2.25.<UUID written as a decimal integer> is a valid UID generation method that needs no registered organization root.
- SOP Instance UID (0008,0018) uniquely identifies an object; often used as file name (names such as 1.2.804.114118.2.20040909... are typically DICOM files).
- SOP Class UID = type of object/service (assigned by standard). SOP Instance UID = this specific object (generated by the creating device software).
- Implementation Class UID: identifies the implementation/software release; sent in the User Information item (item type 52H) of A-ASSOCIATE-RQ/AC; max 64 chars; exactly one per A-ASSOCIATE-RQ and per A-ASSOCIATE-AC; the vendor chooses what it identifies (typically its implementation or software line); listed in the Conformance Statement. Optional companion: Implementation Version Name (55H).
- Archives (PACS/VNA) typically refuse or flag a Study Instance UID that already exists to avoid duplicates (behavior is product-specific; conflicting content under same UID is handled by merge/reject policies).

## Compression and Transfer Syntaxes

- Transfer syntax defines VR explicit/implicit, byte order and pixel compression; negotiated per presentation context.
- Part 5 specifies standard compression (JPEG lossless and lossy, among others). Media profiles fix the allowed transfer syntax because there is no negotiation on media.
- Implicit VR Little Endian must be supported by all (default). See Data Elements section.

- Default transfer syntax is uncompressed; Implicit VR Little Endian is mandatory for every DICOM device.
- Transfer syntax is negotiated per presentation context. The requester must offer Implicit VR Little Endian (the Default Transfer Syntax, which every conformant implementation must support) in at least one presentation context for each Abstract Syntax, usually alongside compressed or Explicit VR options. The acceptor selects exactly one offered syntax per accepted context. There is no automatic fallback: only offered syntaxes can be chosen, so an unsupported compressed syntax works out only because another offered context is accepted. The Default Transfer Syntax itself may be declined if another context for the same Abstract Syntax is accepted. [NORMATIVE: PS3.5 section 10.1]

| Transfer Syntax | UID |
|---|---|
| Implicit VR Little Endian (default) | 1.2.840.10008.1.2 |
| Explicit VR Little Endian | 1.2.840.10008.1.2.1 |
| Explicit VR Big Endian (retired in current Standard) | 1.2.840.10008.1.2.2 |
| JPEG Baseline (Process 1), 8-bit lossy | 1.2.840.10008.1.2.4.50 |
| JPEG Extended (Process 2 & 4), 12-bit lossy | 1.2.840.10008.1.2.4.51 |
| JPEG Lossless, Non-Hierarchical (Process 14) | 1.2.840.10008.1.2.4.57 |
| Deflated Explicit VR Little Endian | 1.2.840.10008.1.2.1.99 |
| JPEG Lossless, Non-Hierarchical, First-Order Prediction (Process 14 SV1) | 1.2.840.10008.1.2.4.70 |
| JPEG-LS Lossless | 1.2.840.10008.1.2.4.80 |
| JPEG-LS Lossy (Near-Lossless) | 1.2.840.10008.1.2.4.81 |
| JPEG 2000 Lossless Only | 1.2.840.10008.1.2.4.90 |
| JPEG 2000 | 1.2.840.10008.1.2.4.91 |
| High-Throughput JPEG 2000 Lossless Only | 1.2.840.10008.1.2.4.201 |
| HTJ2K Lossless RPCL | 1.2.840.10008.1.2.4.202 |
| High-Throughput JPEG 2000 | 1.2.840.10008.1.2.4.203 |
| RLE Lossless | 1.2.840.10008.1.2.5 |
| MPEG2 Main Profile / Main Level | 1.2.840.10008.1.2.4.100 |
| MPEG-4 AVC/H.264 High Profile / Level 4.1 | 1.2.840.10008.1.2.4.102 |

- .4.57 is JPEG Lossless Non-Hierarchical (Process 14), not "baseline lossless"; .4.70 is the first-order-prediction (SV1) variant, the one most lossless-JPEG products use. Transfer syntax names and UIDs checked against pydicom's registry of the Standard.
- Compressed syntaxes use Explicit VR Little Endian rules for the dataset.
- Byte order: receivers byte-swap when sender and receiver endianness differ. Big-endian support is optional and now retired.
- Why compress: archive size; cine runs (cardiology, ultrasound) fitting on one CD/MOD; WAN bandwidth (uncompressed CR or CT can take very long on slow links).
- Vendor private compression Transfer Syntaxes create interoperability problems.

- Presentation context = abstract syntax (SOP Class) + transfer syntax (byte order, implicit/explicit VR, compression). Compression is negotiated at association setup.
- Compressed Pixel Data uses encapsulated transfer syntaxes, always Explicit VR Little Endian for the dataset.
- Pixel Data compression (encapsulated Pixel Data): RLE, JPEG (lossless and lossy), JPEG-LS, JPEG 2000. Deflated Explicit VR Little Endian (1.2.840.10008.1.2.1.99) is different: it deflates the entire encoded Data Set, not only Pixel Data; File Meta Information stays uncompressed Explicit VR Little Endian.
- Private/vendor compression transfer syntaxes cause interoperability failures.
- Reasons: storage reduction, fitting cine/US/cardiology on CD, WAN bandwidth.
- Costs: CPU/decode latency, possible quality loss.
- Compression ratio = original size / compressed size. Lossy typical for medical images about 10:1 up to 20:1; extremes near 100:1 possible but artifacts become visible.
- Lossless: bit-exact reconstruction (variable-length/entropy coding of frequent values). Lossy: quantizes near-equal values within an error margin, then entropy codes.
- Lossy-compressed instances must be flagged in the header (Lossy Image Compression (0028,2110) = 01, with ratio (0028,2112) and method (0028,2114)). DICOM does not forbid recompressing lossy data; each generation must keep the flag and extend the ratio/method history so the image's lossy history stays accurate.
- RLE (1.2.840.10008.1.2.5): lossless, PackBits-style byte-oriented run-length coding. Pixel bytes are first split into byte segments (one per byte of each sample, most significant byte first); each segment is run-length encoded and an RLE header with segment offsets precedes the segments. Not tied to a particular modality or single-frame use.
