# DICOM Notes 3: Networking, Associations and DIMSE Services

## Networking, Associations and DIMSE Services

- Association = negotiated connection between two AEs (Upper Layer Protocol over TCP/IP; OSI stack also defined in PS3.8). All DIMSE traffic runs inside an association.
- Two phases: (1) negotiation, (2) exchange. Each operation is a request answered by a response; the exception is C-CANCEL, a request-only message with no response of its own (the cancelled operation ends with its own final response, status FE00). It is not a one-way broadcast.
- Negotiation proposes presentation contexts: abstract syntax (SOP Class), transfer syntax list, plus maximum PDU size and SCU/SCP role. Without negotiation there is no guarantee the peer supports the object, encoding or role.
- Typical flow: A-ASSOCIATE-RQ with presentation contexts (e.g. MR Image Storage with Implicit VR LE, JPEG 2000, JPEG lossless); the SCP accepts and picks one transfer syntax per context (e.g. uncompressed); C-STORE per image with a response each; release.
- The SCU may offer several transfer syntaxes and SOP Classes (e.g. JPEG, RLE; single-frame vs. multi-frame); the SCP may accept several; the sender then chooses, usually per its Conformance Statement (e.g. cine loops as compressed multi-frame).
- Storage service: SCU connects, sends C-STORE-RQ with the dataset, SCP returns C-STORE-RSP with status (success or failure).
- A DICOM service is built from one or more DIMSE commands. Main services: Storage, Query/Retrieve, Modality Worklist, MPPS, Storage Commitment, Print, Verification.
- Verification (C-ECHO): confirms presence/reachability at DICOM level.

### Layers
- DICOM runs over TCP/IP (OSI L4/L3); below that (T1, Wi-Fi, Ethernet) is transparent to DICOM.
- Conceptual OSI mapping (not a strict implementation layering): Upper Layer protocol: association negotiation and PDUs. Association is session-level (L5); presentation context negotiation (abstract syntax + transfer syntax) is presentation-level (L6). DIMSE/SOP sits at the application level (L7).
- TCP/IP alone only moves bytes between IP:port pairs. Ping success does not prove DICOM connectivity.

### PDUs
| PDU | Type | Purpose |
|---|---|---|
| A-ASSOCIATE-RQ | 01 | request association |
| A-ASSOCIATE-AC | 02 | accept |
| A-ASSOCIATE-RJ | 03 | reject |
| P-DATA-TF | 04 | carries command/data fragments |
| A-RELEASE-RQ | 05 | request release |
| A-RELEASE-RP | 06 | release response |
| A-ABORT | 07 | abort |
- Large datasets are fragmented into P-DATA-TF PDUs.

### Command objects (group 0000)
- DIMSE messages are Command Set (always Implicit VR Little Endian) optionally followed by a Data Set.
- Key command elements: Affected SOP Class UID (0000,0002), Command Field (0000,0100), Message ID (0000,0110), Message ID Being Responded To (0000,0120), Move Destination (0000,0600), Priority (0000,0700), Command Data Set Type (0000,0800), Status (0000,0900), Affected SOP Instance UID (0000,1000), sub-operation counts (0000,1020-1023: remaining, completed, failed, warning).
- Command Data Set Type = 0101 means no data set follows; any other value means a data set follows immediately.
- Command Field values: C-STORE-RQ 0001, C-GET-RQ 0010, C-FIND-RQ 0020, C-MOVE-RQ 0021, C-ECHO-RQ 0030, C-CANCEL 0FFF; N-EVENT-REPORT 0100, N-GET 0110, N-SET 0120, N-ACTION 0130, N-CREATE 0140, N-DELETE 0150; response = request value OR 8000 (e.g. 8001).
- Status basics: 0000 success; FF00/FF01 pending; FE00 cancel; others are failure/warning classes per service.

### Status codes (hex)
Cross-checked against the status definitions in the current pynetdicom library (derived from PS3.4 and PS3.7). Ranges such as A7xx and Cxxx are any value in that range.

| Code | Category | Meaning |
|---|---|---|
| 0000 | Success | Success |
| FF00 / FF01 | Pending | Operation continuing; FF01 = continuing with an optional-key warning (C-FIND) |
| FE00 | Cancel | Terminated by C-CANCEL |
| 0105, 0106, 0110, 0111, 0112, 0113, 0114, 0115, 0117, 0118, 0119, 0120, 0121, 0123 | Failure | General DIMSE-N failures: No Such Attribute, Invalid Attribute Value, Processing Failure, Duplicate SOP Instance, No Such SOP Instance, No Such Event Type, No Such Argument, Invalid Argument Value, Invalid Object Instance, No Such SOP Class, Class-Instance Conflict, Missing Attribute, Missing Attribute Value, No Such Action |
| 0107 | Warning | Attribute List Error |
| 0116 | Warning | Attribute Value Out of Range |
| 0122 | Failure | Refused: SOP Class Not Supported |
| 0124 | Failure | Refused: Not Authorised |
| 0210 / 0211 / 0212 / 0213 | Failure | Duplicate Invocation / Unrecognised Operation / Mistyped Argument / Resource Limitation |

C-STORE:
| Code | Category | Meaning |
|---|---|---|
| A7xx | Failure | Refused: Out of Resources |
| A9xx | Failure | Data Set Does Not Match SOP Class |
| Cxxx | Failure | Cannot Understand |
| B000 | Warning | Coercion of Data Elements |
| B006 | Warning | Elements Discarded |
| B007 | Warning | Data Set Does Not Match SOP Class |

C-FIND: A700 Out of Resources; A900 Identifier Does Not Match SOP Class; Cxxx Unable to Process; FF00/FF01 Pending; B001 Warning (matching reached response limit); A710 Invalid Prior Record Key (only where that Q/R model uses it).

C-MOVE and C-GET: A701 Out of Resources, unable to calculate number of matches; A702 Out of Resources, unable to perform sub-operations; A900 Identifier does not match SOP class; Cxxx Unable to Process; B000 Warning (sub-operations completed with one or more failures or warnings); FF00 Pending; AA00-AA04 frame-level retrieval failures (none of the frames found, cannot create object, cannot extract frames, time-based request on a non-time-based instance, invalid request). C-MOVE only: A801 Move destination unknown. The final C-MOVE/C-GET response carries the sub-operation counts (0000,1020-1023).

- Practical use: a C-STORE failure in A7xx means the receiver was out of resources (disk, memory); A9xx/Cxxx point at the data; Bxxx means the image was stored but changed or flagged, so compare what was stored with what was sent.
- Status codes are not interchangeable across services: the same code can read differently in a DIMSE-N response than in a C-STORE response. Look it up under the service in use (PS3.4 and PS3.7 Annex C).

### C-ECHO (Verification SOP Class 1.2.840.10008.1.1)
- Simplest DIMSE: C-ECHO-RQ answered by C-ECHO-RSP shows the peer is reachable and accepts a Verification association (AE titles, IP and port correct, SCP listening). It does not show that any other SOP Class, transfer syntax or role is supported.
- Many products also refuse unknown callers: configure peer AEs symmetrically (add A to B and B to A) even if one side never initiates.

### C-STORE (push)
- SCU sends C-STORE-RQ with the dataset; SCP replies C-STORE-RSP with status. One storage SOP Class per data type (CT Image Storage, MR Image Storage, waveform, SR, ...), each with its own UID.

### C-FIND
- Matches request identifier attributes against the SCP database; each match returned as C-FIND-RSP + identifier with status FF00 (pending); final response status 0000 (success) without data set. No matches: single response, Data Set Type 0101, status 0000.
- Query roots: Patient Root, Study Root, Patient/Study Only (retired). Study Root is the most widely supported.
- Not modality-specific (no per-modality SOP Classes), unlike C-STORE.

### C-GET (pull, single association)
- SCU sends identifier with unique level keys; SCP finds matches and sends each via C-STORE sub-operations on the same association, so the C-GET SCU must also act as Storage SCP (role negotiation required).
- Pending C-GET-RSP carry sub-operation counts; final response has no data set. Any failure is reflected in final status.
- Retrieval uses same roots as C-FIND. Matching is by unique keys only (Patient ID at patient level; Study/Series/SOP Instance UIDs below), so a C-FIND is normally run first to obtain UIDs.

### C-MOVE (third-party push)
1. SCP matches level keys in its database.
2. Looks up the Move Destination AE title in its configured destination list (IP/port) and opens a new association to it.
3. Issues one C-STORE sub-operation per instance.
4. Sends pending C-MOVE-RSP with counts (remaining/completed/failed/warning).
5. Sends final response (success or error). C-CANCEL aborts an operation in progress.
- Requires the destination AE to be pre-configured on the SCP and a storage SCP listening on the destination.

### C-GET vs C-MOVE
| Aspect | C-GET | C-MOVE |
|---|---|---|
| Associations | one (images return to requester) | two (sub-operations to destination) |
| Destination | always requester | any configured AE |
| Configuration | no pre-configured peer list | static AE table on SCP |
| Firewalls | outbound only; traverses firewalls | inbound association needed; often blocked |
| Security | answers any requester | limited to known destinations |
| Best fit | external/inter-site | internal multi-server PACS |
- Rule of thumb: C-MOVE inside, C-GET outside. Wrong IP/port/AE entries in the destination list are a frequent cause of failed C-MOVE.

### Association
- Association = logical connection between two AEs; lifetime is per exchange, no state retained afterwards (a later association cannot reference an earlier one).
- Sequence: requester (SCU) sends A-ASSOCIATE-RQ with presentation contexts; acceptor replies A-ASSOCIATE-AC (or reject); data exchange; release (A-RELEASE) or abort.
- If no mutually supported context exists, association fails.
- AE Titles are case-sensitive in the Standard and many products reject a mismatched Called or Calling AE Title, while others ignore case or do not validate AE Titles at all [IMPLEMENTATION]. Configure them exactly as the peer's configuration and Conformance Statement expect.
- Idle associations are timed out to free slots. Max simultaneous associations is a configurable parameter and a Conformance Statement item; more concurrent associations consume more memory/swap.
- High-volume senders (CT) may hold one association for hundreds of images; per-image association setup is overhead that is negligible only for slow producers (CR).
- Association count and number of AEs depend on implementation: MWL, Store and Print may share one AE or use separate AEs.
- Roles: SCU/SCP per service. An archive acts as SCP for incoming images and SCU when forwarding to workstations, so it supports both.
- Worklist retrieval needs the SCU role on the modality.
- FSU (File-set Update) role is rarely used; only for updating DICOMDIR, e.g. incremental writing to media.

### Presentation context
- Presentation context = one Abstract Syntax + list of proposed Transfer Syntaxes. Each has a context ID (odd numbers, requester-assigned).
- Abstract Syntax is the SOP Class UID (or Meta SOP Class UID); it is not negotiable. E.g. 1.2.840.10008.5.1.4.1.1.4 = MR Image Storage.
- Acceptor checks the abstract syntax first (unsupported -> context rejected), then picks one transfer syntax from the proposed list. The A-ASSOCIATE-AC returns for each context the result and the single selected transfer syntax.
- Typical context result reasons: acceptance, user-rejection, no-reason, abstract syntax not supported, transfer syntaxes not supported.
- A "not supported" result on a mammography image indicates the PACS lacks that SOP Class.
- Contexts may differ between associations, though a given modality usually proposes the same set each time.
- Compression and byte order belong to the Transfer Syntax, not the Abstract Syntax.
- Private/vendor compression Transfer Syntaxes are a common interoperability problem.

### Extended negotiation
- Optional, rarely used. SOP Class Extended Negotiation carries service-class-specific application info.
- SOP Class Common Extended Negotiation (PS3.7 Annex D): sub-item carries Service Class UID and optionally Related General SOP Class UID(s). At most one sub-item per SOP Class UID in the A-ASSOCIATE-RQ User Data.
- Enhanced Multi-Frame Image Conversion option for Q/R: see Query/Retrieve section.

### Role selection
- SCP/SCU Role Selection is negotiated per SOP Class in the User Information of A-ASSOCIATE-RQ/AC. The requester can act as SCP, the acceptor as SCU, or both; without it the requester is only SCU and the acceptor only SCP. A C-GET SCU must propose the SCP role for each Storage SOP Class it wants returned, and the C-GET SCP must accept it or the C-STORE sub-operations fail. Storage Commitment can use the same mechanism so N-EVENT-REPORT returns on the original association; otherwise the SCP opens a new association. [verify exact requirement per PS3.4 Annex J]

### Supplementary services
- Verification SOP Class (1.2.840.10008.1.1, C-ECHO) tests connectivity at DICOM level only.
- Normalized services (N-CREATE/N-SET/N-GET/N-ACTION/N-DELETE/N-EVENT-REPORT) used by MPPS, Storage Commitment, Print, GP workflow.
