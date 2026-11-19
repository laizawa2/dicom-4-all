# DICOM Notes 1: Overview, Architecture and Information Model

## History, Organizations and Versions

- DICOM = Digital Imaging and Communications in Medicine: standard for exchange, storage and management of medical images and related data. Defines a file format and a TCP/IP network protocol; integrates modalities, servers, workstations, printers, PACS from different vendors.
- Origin: ACR-NEMA (American College of Radiology + National Electrical Manufacturers Association), versions 1.0 and 2.0. Version 3 (1993) renamed it DICOM and added network support, service classes and Conformance Statements.
- ACR-NEMA 2.0 equipment (50-pin point-to-point) is not interoperable with DICOM; a converter/interface box is needed. "ACR-NEMA" is also a generic label for pre-DICOM proprietary links (e.g. SPI, IDTnet).
- "DICOM 3.0" is a legacy label. There is a single continuously maintained standard: no minor versions, no planned incompatible successor. Changes are intended to be backward compatible, with rare exceptions where an earlier specification was ambiguous or incorrect. Compatibility is handled per association by negotiating SOP Classes, transfer syntaxes and roles, not by a version number.
- Cite the standard without an edition date, except when a retired feature (no longer documented in the current edition) is needed.
- DICOM is recognized by ISO (ISO 12052) and has liaison with CEN. DICOM and HL7 are separate organizations with separate scope (imaging vs. clinical messaging).
- Retired SOP Classes: removed from the Standard and not for use in new implementations. Whether to keep accepting or displaying them is an implementation decision; many PACS do so that historical studies stay viewable. [IMPLEMENTATION]

## Architecture: IODs, Modules, SOP Classes, Service Classes

### Standard parts (PS3.x)
| Part | Content |
|---|---|
| PS3.1 | Introduction and overview (oldest, partly outdated) |
| PS3.2 | Conformance: rules and how to write a Conformance Statement |
| PS3.3 | Information Object Definitions (normalized and composite IODs, modules, attributes) |
| PS3.4 | Service Class Specifications |
| PS3.5 | Data structures and encoding: data sets, VRs, transfer syntaxes, compression, character sets |
| PS3.6 | Data Dictionary: all data elements, plus registry of UIDs |
| PS3.7 | Message exchange: DIMSE, command set structure, association negotiation |
| PS3.8 | Network communication support: Upper Layer Protocol (TCP/IP) |
| PS3.9 | Point-to-point 50-pin connection (ACR-NEMA legacy); retired |
| PS3.10 | Media storage and file format: File Set, DICOM File, DICOMDIR |
| PS3.11 | Media Storage Application Profiles |
| PS3.12 | Physical media and format characteristics referenced by PS3.11 |
| PS3.13 | Point-to-point print management; retired |
| PS3.14 | Grayscale Standard Display Function |
| PS3.15 | Security and system management profiles (TLS, LDAP, DHCP, etc.) |
| PS3.16 | Content Mapping Resource: templates, context groups, coded-term lexicon |
| PS3.17 | Informative annexes (usage explanations) |
| PS3.18 | Web services (URI-based access to objects; WADO origin, now DICOMweb family) |
| PS3.19 | Application Hosting: API between a Hosting System and plug-in Hosted Applications |
| PS3.20 | Imaging reports using HL7 CDA R2, incl. transformation of SR to CDA |
| PS3.21 | Transformations between DICOM and other representations (e.g. NCI AIM annotation) |
| PS3.22 | Real-time communication: SMPTE ST 2110-based RTP transport of DICOM metadata alongside video/audio flows |

### Core definitions
- IOD (Information Object Definition): abstract, object-oriented data model of a real-world object; a list of attributes (no values) giving communicating AEs a common view of exchanged information. Built from modules; IEs group modules.
- Normalized IOD: one real-world entity. Composite IOD: aggregates several real-world entities (e.g. an image carries patient, study, equipment, image data).
- SOP Class = Service-Object Pair: one IOD plus a set of DIMSE services. Example: CT Image Storage. Identified by a SOP Class UID (registered in PS3.6).
- SOP Class UIDs must match exactly between peers; old and new versions of a class can look similar but are different UIDs with different capabilities.
- SOP Instance: concrete occurrence of an IOD managed by an AE; persists beyond the communication.
- Service Class (PS3.4): ties one or more IODs to commands, with requirements for both SCU and SCP. Role terms: SCU (user), SCP (provider).
- DIMSE-C services (composite): C-STORE, C-GET, C-FIND, C-MOVE, C-ECHO. DIMSE-N services (normalized): N-GET, N-SET, N-CREATE, N-ACTION, N-EVENT-REPORT, N-DELETE.

### Functional levels ("dimensions") of DICOM
1. Object exchange: verification (C-ECHO) plus Storage SOP Classes.
2. Information management: Modality Worklist, MPPS, Storage Commitment, General Purpose Worklist (reading lists; the workstation can mark an item as in progress).
3. Image quality: Grayscale Standard Display Function (Part 14) and presentation consistency via Presentation State and Presentation LUT.
4. Structured Reporting: SR objects with templates; used for key image notes, CAD results, measurements.
5. Security: encryption, digital signatures, secure transport; audit and access control are application-level.
6. Exchange media: files plus DICOMDIR under strict Media Application Profiles.
- A limited deployment can implement only some levels (e.g. only Store + Print). Full PACS integration needs the management services too.

### IODs
- IOD = abstract definition of a real-world object as a set of attributes (PS3.3).
- Normalized IOD: describes one real-world entity; holds only attributes inherent to it (e.g. Study IOD: study date/time, not patient name).
- Composite IOD: mixes attributes from several entities (e.g. CT Image: patient + study + series + equipment + image). Used for images and other persistent objects.
- IODs are built from Modules; Modules group attributes shared by many IODs (Patient, General Study, CT Image, ...). Each IOD has a module table.

### Module usage codes (per IOD module table)
| Code | Meaning |
|---|---|
| M | Mandatory; supported as defined in PS3.3 Annex C |
| C | Conditional; mandatory only when condition is met, otherwise must not be sent |
| U | User option; if supported, the attribute types inside the module apply |
- Module table columns: module/functional group name, Annex C section reference, usage.

### Attribute types
| Type | Rule |
|---|---|
| 1 | Must be present with a valid non-empty value; missing or zero length = protocol violation |
| 1C | Type 1 rules when condition true; must NOT be present when condition false |
| 2 | Must be present; zero length (value unknown) allowed |
| 2C | Type 2 rules when condition true; must NOT be present when false |
| 3 | Optional; absence is not a protocol violation and carries no significance; zero length is allowed and means exactly the same as absent [NORMATIVE: PS3.5 7.4.5] |
- Type 2 empty values are intended for exceptional "unknown" cases; always sending empty is legal but defeats the intent.
- Patient Name and Patient ID are Type 2 (may be empty if unknown at acquisition).
- A conditional module may depend on e.g. whether contrast was used (Contrast/Bolus module).

### SOP Class
- SOP Class = Service-Object Pair: DIMSE service group + IOD. Example: CT Image Storage = C-STORE + CT Image IOD.
- Identified by a SOP Class UID (root 1.2.840.10008). The UID, not the modality name, decides compatibility (e.g. several Ultrasound variants: retired, current, multi-frame).
- Examples: Computed Radiography Image Storage 1.2.840.10008.5.1.4.1.1.1; Digital X-Ray Image Storage - For Presentation 1.2.840.10008.5.1.4.1.1.1.1; Digital Mammography X-Ray Image Storage - For Presentation 1.2.840.10008.5.1.4.1.1.1.2.
- CT and MR share modules (General Patient, General Study...) and differ in modality modules (e.g. Magnetic Field Strength (0018,0087) in MR; X-Ray Tube Current (0018,1151) in CT).
- Whether a peer supports a SOP Class is documented in its Conformance Statement and can be proven by attempting a presentation context negotiation.

### SOP Class conformance categories
| Category | Rules |
|---|---|
| Standard | Fully per PS3.x, no additions; claimed in Conformance Statement with options/roles |
| Standard Extended | Proper superset of one Standard SOP Class; same UID as the base; no changed semantics; no added/private Type 1/1C/2/2C; no Type 3 promoted to higher type; may add standard or private Type 3 attributes if listed in the Conformance Statement |
| Specialized | Based on a Standard SOP Class; carries all its Type 1/1C/2/2C attributes; semantics unchanged; MUST use a privately defined SOP Class UID; may add Type 1/1C/2/2C and Type 3 attributes; may restrict value sets and permitted templates; Conformance Statement must describe attribute usage, constraints, and the UID |
| Private | Privately defined UID (not under DICOM root); must not alter PS3.6 definitions, existing DIMSE services or PS3.10 file services; may not need default/profile transfer syntax; may apply DIMSE or media storage to private IODs, set any attribute as Type 1/1C/2/2C, define private attributes; Conformance Statement needs a PS3.3/3.4/3.6-style description |
- A SOP Class UID outside 1.2.840.10008 indicates a private SOP Class; other vendors will usually not support it.
- Private SOP Classes are used for objects not yet standardized or for vendor-specific content.

### Service roles and modes
- SCU = requests/uses the service (typically client; modalities, workstations). SCP = provides it (typically server; PACS). Exceptions exist (e.g. Storage Commitment N-EVENT-REPORT goes from SCP to SCU).
- Services in use: Storage, Query/Retrieve, Storage Commitment, MWL, MPPS, Print, Verification.
- Push = C-STORE (automatic routing, e.g. modality to archive). Pull = query (C-FIND) then retrieve (C-MOVE or C-GET); human-driven.
- Typical mapping: query = FIND, retrieval = MOVE, which causes a STORE sub-operation.
- Storage Commitment transfers responsibility for permanent storage (e.g. modality to PACS) after a store.
- MWL delivers scheduled work to a modality (data originates as HL7 orders via RIS).
- Verification (C-ECHO) checks DICOM-level availability of an AE.

### DIMSE families
- Composite (DIMSE-C): C-STORE, C-FIND, C-GET, C-MOVE, C-ECHO (+ C-CANCEL). No update/delete service by design, so stored images are immutable; corrected reports are issued as new documents.
- Normalized (DIMSE-N): N-CREATE, N-SET, N-GET, N-ACTION, N-DELETE, N-EVENT-REPORT. Operate on a single entity (printer, film box, MPPS, ...). ("Normalized" is unrelated to database normalization.)
- N-EVENT-REPORT: notification SCP to SCU (printer status such as film jam; Storage Commitment result).
- N-GET: fetch one instance; requires its SOP Instance UID (may be a Well-known UID).
- N-SET: update an instance (place images in Image Boxes in Print; update MPPS status).
- N-ACTION: request an operation (print film; Storage Commitment request).
- N-CREATE: create an instance in the SCP; UID supplied by SCU or assigned by SCP if omitted.
- N-DELETE: delete an instance (e.g. Film Box after printing).

### Well-known SOP Instance UIDs
| UID | Instance |
|---|---|
| 1.2.840.10008.1.20.1.1 | Storage Commitment Push Model |
| 1.2.840.10008.1.20.2.1 | Storage Commitment Pull Model (retired) |
| 1.2.840.10008.5.1.1.17 | Printer |
| 1.2.840.10008.5.1.1.17.376 | Printer Configuration Retrieval |
| 1.2.840.10008.5.1.1.25 | Print Queue (retired) |

### Storage Service Class
- SOP Classes in Storage identify the composite IOD being stored. Each SOP Class UID maps to one IOD in PS3.3.
- Typical persistent objects: images, Structured Reports, Key Object Selection, Presentation States, RT objects, encapsulated documents.
- Vendors often name SOP Classes inconsistently; the UID is the unambiguous identifier.
- C-STORE copies an instance. A Success response only means the SCP accepted the data; it is not a guarantee of safekeeping (see Storage Commitment).

| SOP Class | UID | IOD |
|---|---|---|
| Computed Radiography Image Storage | 1.2.840.10008.5.1.4.1.1.1 | CR Image |
| Digital X-Ray Image Storage - For Presentation | 1.2.840.10008.5.1.4.1.1.1.1 | Digital X-Ray Image |
| Digital X-Ray Image Storage - For Processing | 1.2.840.10008.5.1.4.1.1.1.1.1 | Digital X-Ray Image |
| CT Image Storage | 1.2.840.10008.5.1.4.1.1.2 | CT Image |
| Enhanced CT Image Storage | 1.2.840.10008.5.1.4.1.1.2.1 | Enhanced CT Image |
| MR Image Storage | 1.2.840.10008.5.1.4.1.1.4 | MR Image |
| Secondary Capture Image Storage | 1.2.840.10008.5.1.4.1.1.7 | SC Image (plus derived SC SOP Classes) |

- Secondary Capture: for images not produced by a native modality: digitized film, scanned reports, post-processed/3D renderings, screen captures.

### Storage SCP conformance levels
| Level | Meaning |
|---|---|
| 0 (Local) | Only an implementer-defined subset of attributes stored; rest discarded; subset must be listed in the Conformance Statement |
| 1 (Base) | All Type 1 and 2 attributes of the IOD stored and retrievable; others may be discarded |
| 2 (Full) | All Type 1, 2, 3 attributes plus standard extended and private attributes stored and retrievable; empty (zero-length) Type 3 attributes may be discarded [NORMATIVE: PS3.4 B.4] |
- At Levels 1 and 2 the SCP may, but need not, validate against the IOD.

### Coercion
- Receivers often need a minimum attribute set (Patient Name, Patient ID, etc.) to file an object, and modify (coerce) values instead of rejecting.
- Typical case: modality without Information System connectivity, technologist types demographics, receiver corrects them.
- Rules for allowed coercion are loosely defined and vendor-dependent. Guideline: if a change alters clinical interpretation, create a modified copy as a separate object (new SOP Instance UID) rather than overwriting.

### Multi-frame vs single-frame objects
- Multi-frame: one header plus pixel data for all frames in one instance; one C-STORE per loop. Frame relationships are described in the header (e.g. Frame Increment Pointer (0028,0009)).
- Pros: single transfer, receiver cannot silently miss frames, less repeated header data (minor versus pixel size).
- Cons: receiver must handle one very large object; series cannot be split.
- Enhanced multi-frame IODs allow smaller instances plus defined concatenation of frame groups so senders can chunk large acquisitions.
- Enhanced CT, Enhanced MR (includes MR spectroscopy) etc. must be supported by the PACS and every viewer that may display them.
- DX Image IOD should be used for CR-type devices instead of legacy CR IOD (stricter required header content).
- Enhanced X-Ray RF Image IOD: table-with-column radiofluoroscopy; for systems with patient-based coordinates (cranial/caudal, LAO/RAO), use the Enhanced XA Image IOD.
- Encapsulated PDF IOD wraps a PDF document in a DICOM object.

### Presentation Intent (FOR PRESENTATION / FOR PROCESSING)
- Presentation Intent Type (0008,0068): FOR PRESENTATION = ready for viewing after VOI LUT etc. FOR PROCESSING = detector-corrected raw data intended for further processing.
- FOR PROCESSING separates acquisition from image processing; useful for CAD, QC, research and vendor-neutral processing.
- CAD uses FOR PROCESSING; viewing workstations need FOR PRESENTATION. Archiving both is an institutional policy choice.

### Gated NM / R-R data
- Gated Information Sequence (0054,0062), Type 2C: required if Frame Increment Pointer (0028,0009) contains R-R Interval Vector (0054,0060).
- Item count equals Number of R-R Intervals (0054,0061). Item n describes frames whose R-R Interval Vector value is n. Each item gives an upper/lower heart-beat duration range.

## Information Model and Hierarchies

- Entity-Relationship (E/R) model in PS3.3: Information Entities (IEs) are real-world concepts; relationships link them. Vendor deviation from the model is a recurrent source of interoperability problems.
- Main hierarchy: Patient 1..n Study; Study 1..n Series; Series 0..n (typically 1..n) Instances (images, waveforms, SR, etc.). Around 20 IEs exist (also Frame of Reference, Equipment, etc.).
- Modules group related attributes; IEs are built from modules. Examples: Patient IE = Patient, Specimen Identification, Clinical Trial Subject modules. Study IE = General Study, Patient Study, Clinical Trial Study modules. Series/image levels hold more modality-specific modules.
- Series-level attribute Modality (General Series Module) is single-valued, so a CT and a PET acquisition cannot share one series. Combined exams use separate series (usually described by Series Description).
- A series generally corresponds to one modality/acquisition or patient position.
- Query/Retrieve hierarchical model: Patient (optionally omitted in Study Root) > Study > Series > Image. Hierarchical queries/retrievals fail unless higher-level keys (e.g. Patient ID, Study Instance UID) are supplied. Support for relational (non-hierarchical) queries is optional; the Conformance Statement says whether it exists. Hierarchical is mandatory.
- Hanging protocols rely on series identification, laterality/view codes, orientation, body part, description, and so on. Studies may need splitting so that body regions go to different readers. Standard objects: Hanging Protocol Storage and Query/Retrieve SOP Classes (the source describes this as a "Hanging Protocol Service Class").

### Procedure-related entities (workflow model)
- Imaging Service Request: set of Requested Procedures sent by one requester to one provider within one service episode; carries common information. In MWL it drives acquisition; in General Purpose Worklist it can drive reporting or post-processing.
- Requested Procedure (RP): instance of a procedure type; the unit an order asks for and a report answers; maps to one or more Scheduled Procedure Steps.
- Scheduled Procedure Step (SPS): unit of scheduled work with protocol, equipment, staff, location and time. Performed on exactly one modality device. May yield zero or more Performed Procedure Steps. An SPS may belong to several RPs (e.g. shared contrast injection), but for billing it normally counts in one.
- Performed Procedure Step (PPS) / Modality PPS: what was actually done; has a state (in progress, discontinued, completed). Contains performed protocol, billing and material data, and references to the series/instances created. One series belongs to exactly one MPPS. MPPS only reports; it implies no storage semantics and does not control later workflow.
- One PPS can satisfy several SPS/RP/requests; one SPS can need several PPS (equipment failure, multiple devices, long duration).
- MWL data model (two levels): Requested Procedure (patient identity, Accession Number, Study Instance UID, procedure via (0032,1060) Requested Procedure Description or (0032,1064) Requested Procedure Code Sequence) as parent; Scheduled Procedure Step (modality-specific details) as child. An RP can hold several SPS (multi-modality, or several protocols such as chest + abdomen; with/without contrast are SPS of one RP).
- Content items in an SR: each parent (source) item relates to child (target) items via a Relationship Type.

- All real-world data (patient, study, device) is modelled as objects with attributes (IODs).
- A DICOM file packs header and pixel data into one file; header tags carry patient, study and acquisition data and must stay bound to the image data. A detached header leaves the image unidentified and mis-displayed (patient safety/medico-legal risk).

- Real-world model (PS3.3 section 7) extends Patient-Study-Series-Image without changing it; most DICOM devices operate on Patient > Study > Series > Image.
- Relationships: patient has many studies; a study belongs to one patient; a series belongs to one study; an instance belongs to one series. Frame of Reference and Equipment IEs sit at series level.
- Unique key per level selects the single path down the tree (see Q/R section).
