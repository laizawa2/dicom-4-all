# DICOM Notes 5: Image Objects, Display, Structured Reporting and Presentation States

## Objects, Image Pixel Data and Display

- Grayscale Standard Display Function (GSDF, PS3.14): defines display luminance response from a perceptual model (Barten curve) so perceived contrast is uniform across pixel values. Used to calibrate monitors and printers; implemented using a Presentation LUT.
- Key images: images flagged by the interpreting physician. Stored as Key Object Selection Document (an SR object) referencing instances plus a text purpose. IHE profile: Key Image Note (KIN). Standard encoding allows cross-vendor flagging.

### Display consistency
- Grayscale Standard Display Function (GSDF, PS3.14): defines luminance/density versus input value on a perceptually linear scale; applies to monitors and printers. Devices calibrate to it directly or via Presentation LUT.
- Viewing integrity also needs same window width/level, zoom and annotations: conveyed by Presentation State.
- Color consistency: device-independent via ICC profiles (ICC Profile (0028,2000)), transforming to the ICC Profile Connection Space. Color Presentation States must contain an ICC profile; color image SOP Classes allow an optional ICC profile.

### Grayscale pipeline (order of operations)
1. Unpack pixels using Bits Allocated / Bits Stored / High Bit / Pixel Representation.
2. Interpret Photometric Interpretation (MONOCHROME1/2 or color).
3. Modality transform: Rescale Slope (0028,1053) and Rescale Intercept (0028,1052), or a Modality LUT (non-linear table, common for CR/DR). Output = slope x stored + intercept.
4. VOI transform: Window Center (0028,1050) / Window Width (0028,1051) linear window, or VOI LUT Sequence (0028,3010) table.
5. Presentation LUT (application-specific look; also used for print).
6. Display via calibrated device (GSDF).

- CT: output units Hounsfield (water = 0, bone positive, air/lung negative). Stored CT values are often scaled: Rescale Slope/Intercept (commonly 1 and -1024 with unsigned storage, but Pixel Representation and intercept vary by scanner and series) must be applied before display or HU measurement.
- Window: values below WC - WW/2 clip to minimum output, above WC + WW/2 clip to maximum, in between mapped linearly to the output range (typically 0-255).
- An image may carry several window presets (e.g. lung, bone). If none present, the default window is implementation-dependent (often full range or a modality default); viewers should keep a revert-to-stored option.
- "Contrast/brightness" on a viewer are often confused with WW/WL; true monitor brightness/contrast are hardware settings.
- Digital projection radiography has very high dynamic range; the linear detector response lacks the film S-curve. Preferred exchange is unprocessed data plus a VOI LUT in the header, so the receiver can still apply another LUT. A receiver that ignores the LUT shows a very different-looking image.
- A static VOI LUT table cannot be parametrically adjusted and changes cannot be saved except by building a new LUT.
- LUT support belongs in the Conformance Statement; check headers to confirm.

### Display transformations
- Spatial Transformation module: rotation by 90/180/270 degrees clockwise, followed by optional horizontal flip. Arbitrary angles not supported.
- Displayed Area module: selects the region of pixel space to show; magnification/interpolation method is implementation dependent.
- Landscape/portrait mismatches (e.g. CR cassette taken sideways) are corrected by rotation, preferably saved in a Presentation State.
- Zoom, pan, loupe, invert, rotate/flip, window preset are viewer functions; stored state is saved only through a Presentation State.
- Screen annotation (fixed on the screen, unaffected by image operations) versus image annotation (anchored to image coordinates, transforms with it).

### Print-related image attribute
- Requested Decimate/Crop Behavior (2020,0040): DECIMATE (magnification factor < 1, method per Magnification Type (2010,0060) or SCP default), CROP (delete rows/columns; algorithm in SCP Conformance Statement), FAIL (SCP must neither crop nor decimate).
- Presentation LUT Storage SOP Class lets a workstation download a LUT to the printer and reference it in print commands, so hardcopy matches the monitor (GSDF).

## Structured Reporting and Presentation States

- SR: DICOM object for structured, template-based content (observations, reports); same header/body structure as other composite objects with its own SOP Instance UID.
- Uses: key object selection, CAD, routine ultrasound measurements, general reporting (IHE Simple Image and Numeric Report).
- Grayscale Softcopy Presentation State stores zoom, annotations, window/level etc. in a standard way.
- Templates and coded-term context groups live in PS3.16.
- PS3.20 provides CDA R2 templates for imaging reports and maps SR to CDA.

### Coded terminology
- Codes are unambiguous vs free text; needed for software decisions (e.g. hanging protocol selection), payer reimbursement, public-health reporting.
- A code is a triplet: Code Value (0008,0100), Coding Scheme Designator (0008,0102), Code Meaning (0008,0104). The designator is required because a value is only unique within its scheme. Local/private schemes may be defined (e.g. a site-specific designator).
- Schemes: SNOMED CT (clinical terms; designator SCT, legacy SRT; originated at CAP; useful e.g. for specimen anatomical site), LOINC (lab/observation identifiers, from 1994, Regenstrief Institute), CPT (AMA procedure codes for billing), ICD (WHO disease classification; source cites ICD-9, current use is ICD-10/ICD-11 [verify]), DCM (codes defined by DICOM itself, in PS3.16).
- Measurement units: UCUM is the usual DICOM scheme for units (source attributes unit codes to LOINC [verify]).
- PS3.16 = DICOM Content Mapping Resource and controlled terminology (templates, context groups, code definitions).
- Procedure codes describe requested/performed imaging procedures (used in MWL and MPPS).

### SR IODs
| IOD | Notes |
|---|---|
| Basic Text SR | Most constrained: heading/text tree, minimal coding; SOP Instance references only at leaf level |
| Enhanced SR | Superset of Basic Text; adds numeric measurements with coded names/units and spatial/temporal ROI references |
| Comprehensive SR | Superset of both; allows by-reference relationships between content items and broader value types |
| Comprehensive 3D SR | 3D spatial coordinates |
| Procedure Log | Time-stamped event log of extended procedures (e.g. cath lab) |
| Key Object Selection Document | Key image flagging |

SR SOP Class UIDs (from standard knowledge): Basic Text 1.2.840.10008.5.1.4.1.1.88.11, Enhanced .88.22, Comprehensive .88.33, Comprehensive 3D .88.34, Procedure Log .88.40, Mammography CAD .88.50, Key Object Selection .88.59, Chest CAD .88.65, X-Ray Radiation Dose .88.67, Colon CAD .88.69, Implantation Plan .88.70; Encapsulated PDF 1.2.840.10008.5.1.4.1.1.104.1. [UIDs checked against the PS3.6 registry]
- Also: dedicated SOP Classes bind specific templates (CAD per template, US vascular/OB-GYN/echo, breast imaging, cardiology, dose).

### SR content model
- Single tree, one root content item = document title; may represent flat lists or DAG-like structures via by-reference links.
- Value types: text, numeric (value + unit), code, date/time/date/time, person name, UID reference, image/waveform/composite references, spatial and temporal coordinates, container.
- Images/waveforms are referenced, never embedded (thumbnails may be included).
- Relationship types: CONTAINS, HAS PROPERTIES, INFERRED FROM, SELECTED FROM, HAS OBS CONTEXT, HAS ACQ CONTEXT, HAS CONCEPT MOD.
  - CONTAINS: child is part of parent (headings/sections).
  - HAS PROPERTIES: child describes characteristics of parent (e.g. size, margin).
  - INFERRED FROM: parent is a conclusion derived from child content (e.g. measurement/finding).
- Numeric measurement: value + coded unit + concept name, linkable to an image via INFERRED FROM.
- Templates (TIDs) constrain nodes, relationships and order; either conveyed within generic SR SOP Classes or fixed by a dedicated SOP Class. IHE simple report: text, image reference, measurement.
- Procedure Log (cath lab): comprehensive time-stamped record of events to document acquisition context for images, waveforms and measurements.

### Key Object Selection (Key Image Note)
- Stored as SR: Key Object Selection Document (KO) referencing images with a title (purpose) and optional text comment.
- IHE Key Image Note (KIN) profile: mark significant images, with title and user comment. Uses: referring physician access, teaching files, consultation, image-quality problems.
- Vendor-specific flagging is not portable; KO is.

### Presentation State (PR)
- Independent SOP Instance (modality PR, its own series) holding display instructions; contains no Pixel Data and references the image(s)/series it applies to.
- Contents: window/level, zoom, pan, rotation/flip, shutters, annotations (text, graphics), displayed areas, overlays/masks, image orientation, LUTs. Grayscale and color variants exist.
- Original pixels remain untouched; presentation can be reverted. Must be stored in a PACS that supports the PR SOP Classes, else it is dropped or not retrievable.
- Preferred over burned-in annotations and Group 6000 overlay bits for storing annotations and display state, and the mechanism used by IHE Consistent Presentation of Images. It can activate or hide overlays but does not replace the Overlay Plane attributes themselves.
- Use cases: measurements drawn as graphics, technologist notes as screen annotation, shutter after collimation, rotation correction.

### Display shutters
- Shape defined in row\column coordinates, origin = upper-left pixel = 1\1. Up to three shapes may be combined.
- Shutter Shape (0018,1600): RECTANGULAR, CIRCULAR, POLYGONAL (shapes may be combined). Rectangular edges: left (0018,1602), right (0018,1604), upper (0018,1606), lower (0018,1608). Circular: center (0018,1610), radius (0018,1612). Polygonal: vertices (0018,1620).
- Polygonal: Vertices of the Polygonal Shutter (0018,1620) = row\column pairs, first pair is origin vertex; implicitly closed to the origin; edges must not intersect except at vertices.
- Shutter Presentation Value (0018,1622): gray value used to blank the outside area; behavior undefined if absent or empty.
- Rectangular: hides collimator white borders (CR/DR). Circular: hides bright rim of round images. Shutters in a PR can be toggled so the masked area is still viewable.

### SR, CDA and HL7 references
- CDA (Clinical Document Architecture): HL7 XML document standard for longitudinal records; may reference images. Transport-neutral (DICOM, MIME, FTP, HTTP, HL7 v2/v3). CDA plus CCR underlie the CCD. FHIR is a separate, broader HL7 standard.
- XML only supplies syntax; interoperability still needs shared semantic definitions (comparable to DICOM tag/length/value without a shared dictionary).
- SR objects can be mapped to/from XML/CDA for enterprise use; an SR can reference its equivalent CDA document.
- Referencing an unencapsulated HL7 Structured Document from a DICOM SOP Instance: SOP Class UID = OID of its Hierarchical Message Description; SOP Instance UID = mapped from the HL7 instance identifier (II). OIDs do not imply a storage/network service but identify the abstract syntax.
- HL7 OIDs: CDA R1 2.16.840.1.113883.1.7.1; CDA R2 2.16.840.1.113883.1.7.2; SPL R1 2.16.840.1.113883.1.7.3.
- DICOM UIDs use ISO OID syntax, so the same value can be carried as an OID in HL7 identifiers; how identifiers map between the two standards is defined per use case (as above), not automatically. CDA uses SNOMED CT and LOINC coding.
