# DICOM Notes: index

Original reference notes on DICOM for PACS, VNA and imaging-integration work, written for the DICOM_4_ALL plugin. They are practical summaries, not the Standard: confirm normative requirements (attribute Types, status codes, UIDs, transfer syntax rules) against the current DICOM Standard (see `../DICOM_PART_LINKS.txt`). Items marked **[verify]** are points the author could not confirm.

Read only the file that matches the question.

- `01-overview-architecture-model.md`: History, Organizations and Versions; Architecture: IODs, Modules, SOP Classes, Service Classes; Information Model and Hierarchies (19 KB)
- `02-data-elements-uids-transfer-syntaxes.md`: Data Elements, VRs and Encoding; UIDs; Compression and Transfer Syntaxes (16 KB)
- `03-networking-dimse.md`: Networking, Associations and DIMSE Services (9 KB)
- `04-workflow-services.md`: Workflow Services: MWL, MPPS, Storage Commitment, Query/Retrieve, Print (12 KB)
- `05-objects-sr-presentation.md`: Objects, Image Pixel Data and Display; Structured Reporting and Presentation States (10 KB)
- `06-media-security-conformance-integration.md`: Media Exchange and DICOMDIR; Security and De-identification; Conformance Statements; DICOMweb; HL7, IHE and System Integration (7 KB)
- `07-troubleshooting-other.md`: Troubleshooting and Tools; Other (4 KB)

## Authority labels
Some statements carry a label saying how much weight they have:
- **[NORMATIVE]**: a requirement of the DICOM Standard (the part is named).
- **[IMPLEMENTATION]**: common product behavior or configuration, not required by DICOM.
- **[verify]**: not confirmed; check the current Standard or the product's Conformance Statement.

Unlabeled statements are practical summaries. Do not present them as DICOM requirements without checking the Standard.

## Not independently verified
Checked against the Standard text (PS3.2 7.3, PS3.3 C.7.6.3, PS3.4 B.4 and J.3.3, PS3.5 7.4, 8.1 and 10.1) and PS3.6 registries: all UIDs and tags, DIMSE command values, attribute Types, SOP Class conformance categories, Storage SCP levels, Image Pixel module rules and Storage Commitment event types. Corrections made: Type 3 zero length equals absence; High Bit is one less than Bits Stored; Pixel Data VR depends on the Transfer Syntax, not Bits Allocated; Planar Configuration, PALETTE COLOR and YBR rows added; transfer syntax table extended.
Also confirmed since: Storage Commitment Failure Reason values (PS3.3 C.14.1.1), the DIMSE status code tables (against pynetdicom's current definitions), and the YBR_FULL conversion coefficients (BT.601 weights).
Still unconfirmed: the HL7 OIDs, the YBR_PARTIAL_420 limited-range formulas, and status codes for services not tabulated (Print, MWL-specific, MPPS). Check these in PS3.3, PS3.4, PS3.7 and PS3.16 before relying on them.
