# DICOM Notes 4: MWL, MPPS, Storage Commitment, Query/Retrieve and Print

## Workflow Services: MWL, MPPS, Storage Commitment, Query/Retrieve, Print

### Modality Worklist (MWL)
- SOP Class: Modality Worklist Information Model - FIND, using C-FIND against the RIS/IS; works like Query/Retrieve (information model + command).
- Purpose: remove re-typing of demographics (name, birth date, sex) and scheduling data at the modality; improves data integrity.
- Modality either queries in real time or refreshes a cached list by polling.
- Matching keys commonly used in practice: Accession Number, Patient Name, Patient ID, plus Scheduled Station AE Title, Modality and Scheduled Procedure Step Start Date. The keys a given SCP supports are defined by PS3.4 and its Conformance Statement; there is no universal minimum. [IMPLEMENTATION]
- Accession Number: links order, images and report. Copied from worklist into image header; the RIS matches the report to the order via it. Order placer/filler numbers carry no meaning for the exam. Some modalities do not allow manual entry of it; an IS interface (MWL) fixes this.
- A combined exam with a single order normally has one Accession Number. Two reports by two readers may need separate sub-orders/accession numbers (or suffix) if the RIS cannot split reports.
- Wrong procedure mapping in the RIS/PACS can leave studies unspecified/unmatched in PACS.
- Service: C-FIND on the Modality Worklist Information Model (1.2.840.10008.5.1.4.31). Entry point is the Scheduled Procedure Step (SPS).
- One C-FIND response per matching SPS; each SPS appears in exactly one response. Requested Procedure, Imaging Service Request (ISR), Visit and Patient data may repeat across responses.
- Hierarchy: Patient -> Visit -> ISR -> Requested Procedure (RP) -> SPS.
- Modules involved (PS3.3): Patient Relationship, Patient Identification, Patient Demographic, Patient Medical, Visit Relationship, Visit Identification, Visit Status, Visit Admission, Scheduled Procedure Step, Requested Procedure, Imaging Service Request.
- Scheduled Procedure Step Description: institution-defined text for the planned step.
- Multiple SPSs under one RP/ISR, e.g. CT with and without contrast: one study, separate SPSs.
- Querying by Accession Number normally returns a single ISR, which avoids picking the wrong patient from a list. An Accession Number is unique only within its issuer (see Issuer of Accession Number Sequence (0008,0051)), so multi-site or merged systems can still collide.
- Slow order databases can delay MWL responses by seconds; modalities may poll and cache the worklist.
- Without MWL, studies reach the PACS with hand-typed demographics: typos, unverified studies, manual reconciliation required.

### Modality Performed Procedure Step (MPPS)
- Modality reports to the MPPS manager (which relays to RIS and PACS): procedure started, number/identity of images, status (in progress, completed, discontinued), what was actually performed versus scheduled (can trigger a change order in the RIS), and optionally radiation dose.

### Storage Commitment
- The SCP commits to safeguarding the referenced SOP Instances and reports that back; the SCU may then decide to delete its own copies. A C-STORE success response alone carries no such commitment. What the commitment means in practice (redundancy, backup, retention) is defined by the SCP's Conformance Statement and operational policy, so it is not a guarantee of permanent storage. [NORMATIVE: PS3.4 Annex J]
- Flow: after C-STORE, SCU sends N-ACTION listing SOP Class/Instance UID pairs; SCP answers with N-EVENT-REPORT listing committed instances and failed ones. The report may be delivered over the original association or a new one; the Transaction UID in the request ties the two together.
- Recommended for all modality software in addition to image transfer.

### Print
- Basic Print Management with Presentation LUT: consistent appearance on printed output. Grayscale Softcopy Presentation State: consistent softcopy appearance.

### QA / relay station (PACS gateway)
- Checks images before they reach PACS: verifies demographics against RIS data, applies shutters/rotation, re-processes, splits or groups studies (e.g. whole-body CT to separate body parts), confirms arrival in PACS. May operate in pass-through mode.
- Orphan images: images missing from, or held outside, the PACS database (e.g. held in a quarantine area at archive/QA station, or present in archive storage with no database entry); they distort study completeness.

- MPPS: modality reports performed procedure step to RIS/PACS (N-CREATE/N-SET); performed details may differ from scheduled, so procedure codes must be filled in by the modality.
- Storage Commitment: N-ACTION request (Push Model instance 1.2.840.10008.1.20.1.1), result by N-EVENT-REPORT from SCP to SCU.
- Print: Film Box created/deleted by N-CREATE/N-DELETE; images placed with N-SET in Image Boxes; N-ACTION prints; Printer status by N-GET and N-EVENT-REPORT.
- Query/Retrieve: C-FIND + C-MOVE/C-GET (see above).

### MPPS (Modality Performed Procedure Step)
- SOP Class UID 1.2.840.10008.3.1.2.3.3. Modality reports what was actually done to RIS/PACS.
1. N-CREATE at start: Performed Procedure Step Status = IN PROGRESS.
2. N-SET at end: status COMPLETED (or DISCONTINUED). The final set lists created image/series instances so PACS can verify all arrived; may also carry dose information.
- Uses: remove item from modality worklist, show exam as started/completed, trigger billing (technical component), dose reporting, send to reading worklist (which actions a PACS or RIS takes on MPPS messages is implementation-specific).
- Performed procedure may differ from scheduled; modality sends performed codes so the RIS can correct the order.
- IHE: modality sends MPPS to one destination (RIS or PACS); the MPPS manager forwards to the other. RIS/PACS act as SCP; the forwarding party also acts as SCU.
- PACS benefits from MPPS even if RIS lacks support.

### Storage Commitment (Push Model)
- SOP Class UID 1.2.840.10008.1.20.1 (Storage Commitment Push Model).
- Flow: SCU stores instances via C-STORE, then sends N-ACTION-RQ (Request Storage Commitment) listing SOP Class/Instance UID references; SCP answers N-ACTION-RSP; SCP later reports with N-EVENT-REPORT.
- Transaction UID (0008,1195) supplied by SCU identifies each request; echoed in the result. Allows matching of multiple outstanding requests, also across associations.
- Delivery of N-EVENT-REPORT over the original or a separate association, and retry count/interval, are implementation settings. [IMPLEMENTATION]
- Event types: 1 = Storage Commitment Request Successful, 2 = Complete - Failures Exist. Result lists Referenced SOP Sequence (0008,1199) and Failed SOP Sequence (0008,1198) with Failure Reason (0008,1197). Success only if all requested instances are committed. [NORMATIVE: event type IDs 1 and 2, PS3.4 J.3.3; Failure Reason values, PS3.3 C.14.1.1] Failure Reason (0008,1197): 0110H Processing Failure; 0112H No such SOP Instance; 0213H Resource limitation; 0122H Referenced SOP Class not supported; 0119H Class/Instance conflict; 0131H Duplicate Transaction UID (the request's Transaction UID is already in use).
- On failure the SCU retries and must not delete local copies until a positive commitment arrives.
- No Service Class specific status codes for N-ACTION; general PS3.7 statuses apply.
- Commitment is a protocol-level promise only; actual safekeeping depends on administration (backups, access control, auto-delete policies). Workstations that receive images for display only typically do not implement it; archives do.

### Query/Retrieve (Q/R)
- Purpose: simple queries on small key sets and retrieval of composite instances; not general SQL.
- Information model: SOP Class = information model + DIMSE-C group (C-FIND, C-MOVE, C-GET).
- Models: Patient Root, Study Root (identical but top level is Study; patient attributes belong to the study). Patient/Study Only model is retired (PS3.4-2004).

| Model | FIND | MOVE | GET |
|---|---|---|---|
| Patient Root | 1.2.840.10008.5.1.4.1.2.1.1 | ...1.2.1.2 | ...1.2.1.3 |
| Study Root | 1.2.840.10008.5.1.4.1.2.2.1 | ...1.2.2.2 | ...1.2.2.3 |

- Levels and unique keys: PATIENT = Patient ID (0010,0020); STUDY = Study Instance UID (0020,000D); SERIES = Series Instance UID (0020,000E); IMAGE = SOP Instance UID (0008,0018).
- Patient and Study level attributes are modality independent. Series and Instance levels are modality dependent; their optional keys include any series/instance level attribute from any composite IOD.
- Hierarchical queries must supply the higher-level unique keys (cannot search a series without its study/patient keys). Relational (non-hierarchical) queries allow any related key combination; optional, declared in the Conformance Statement, rarely supported.
- Each SCP must support required/unique keys; optional keys vary and are listed in the Conformance Statement.
- Matching types: single value, list of UID, universal, wildcard, range, sequence matching.

**C-FIND**
- Identifier carries keys at the query level. SCP returns one response with status Pending (FF00/FF01) per match with requested attributes; final response Success (0000) with no identifier.
- Failed/Refused = SCP cannot process. SCU may send C-FIND-CANCEL; SCP stops and returns Canceled (FE00).
- Broad queries (e.g. common surname) can flood the SCU, network and database; use narrow keys and cancel/limit.

**C-MOVE**
- SCU supplies unique keys plus Move Destination AE title. SCP performs C-STORE sub-operations on a separate association to the destination (which may not be the Move SCU).
- Pending responses report Remaining, Completed, Failed, Warning sub-operation counts. Final status Success/Warning/Failed/Refused (or Canceled); a Failed sub-operation list is returned as Failed SOP Instance UID List.
- Destination AE must be configured/known at the SCP (AE title, IP, port); static configuration is a typical source of failures. The C-STORE status goes to the destination, so the Move SCU relies on the C-MOVE responses for outcome.
- Initiation of sub-operations is outside DICOM scope; the Q/R SCP and Storage SCU roles may be separate applications.

**C-GET**
- Same as C-MOVE semantically but C-STORE sub-operations run on the same association with roles reversed (requester needs SCP role selection for the storage SOP Classes).
- Single association: no static destination configuration; usually passes firewalls since the connection is outbound. C-MOVE needs an inbound association that firewalls often block.
- May be slightly faster (no extra association).

**Enhanced Multi-Frame Image Conversion extended negotiation**
- Lets SCU view studies as default (as received), Classic single-frame, or Enhanced multi-frame. Conversion applies below IMAGE-level only without duplicates; references (UIDs) are made consistent across the Patient for the chosen view.

### General Purpose Worklist (retired)
- Retired in 2012 (Supplement 158): GP Scheduled Procedure Step, GP Performed Procedure Step, GP Worklist Query, GP Worklist Management Meta SOP Class. Earlier editions remain valid references.
- Purpose: shared pool of tasks (e.g. reading, therapy planning) with status and locking so several workstations avoid duplicate work. Differs from MWL by providing a lock/status back-channel.
- Status model: SCHEDULED, IN PROGRESS, SUSPENDED, COMPLETED, DISCONTINUED. Setting IN PROGRESS via N-ACTION gives implicit exclusive lock; other SCUs' attempts fail. Holder may release by setting SUSPENDED or SCHEDULED.
- N-CREATE/N-SET/N-GET on the GP Performed Procedure Step instance create, update and read performed data.
- Most PACS use proprietary equivalents; matters mostly for multi-vendor viewing.

### Print Management
- Purpose: consolidated networked film printers. Raw image pixel data cannot just be sent to printers; the standard defines hardcopy image objects and print SOP Classes.
- Meta SOP Classes bundle required SOP Classes (Film Session, Film Box, Image Box, Printer) and are negotiated as one abstract syntax; support is listed in the Conformance Statement. Basic Grayscale and Basic Color Print Management Meta SOP Classes exist.
- Procedure:
  1. Associate with print Meta SOP Class.
  2. N-CREATE Basic Film Session (e.g. number of copies).
  3. N-CREATE Basic Film Box (Image Display Format e.g. STANDARD\2,3 = 2 columns x 3 rows; size, background density). SCP creates the Image Boxes and returns references in the N-CREATE response.
  4. N-SET pixel data into each Basic Grayscale/Color Image Box.
  5. N-ACTION to print (film box or session; some printers support only one).
  6. N-DELETE created objects (optional in practice; SCP may clean up on association close, a deviation from other services).
- Printer SOP Class: status and configuration may be requested at any time via N-GET; printer sends N-EVENT-REPORT on status change.
- Additional SOP Classes: Print Job (check queue, prioritize urgent), Presentation LUT, Printer Configuration, Print Annotation (text on film).
- Printable area: most laser printers cannot image to the film edge (about 13.5 x 16.5 in on 14 x 17 in). Image must be cropped or minified (decimate/crop behavior selectable).
- True-size printing: set Requested Image Size (2020,0030) = image width in mm (pixel spacing x columns) on the Image Box; also set printer parameters preventing scaling. Height is derived by the printer.
- Print supports 8 and 12-bit data; 10-bit data must be shifted up or down 2 bits.
- Look consistency: Presentation LUT SOP Class (vendor-neutral table) or Configuration Information attribute (vendor-specific values; consult printer Conformance Statement). Both workstation and printer must support the LUT SOP Class for downloading and referencing.
- Printer Configuration SOP Class lets SCU read printer capabilities (decimate/crop result, pixel spacing, Configuration Information, display formats, rows/columns) instead of maintaining static tables.
- SNMP is separate from DICOM and not part of the print services.
