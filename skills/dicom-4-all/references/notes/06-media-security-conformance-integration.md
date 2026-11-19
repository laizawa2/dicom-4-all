# DICOM Notes 6: Media, Security, Conformance, DICOMweb and Integration

## Media Exchange and DICOMDIR

- PS3.10: DICOM File Set, DICOM File format, DICOMDIR maintenance policy. Files on DICOM media have no extension; .dcm is conventional elsewhere.
- File structure (standard knowledge): 128-byte preamble, "DICM" prefix, File Meta Information group (0002) in Explicit VR Little Endian, then the data set.
- PS3.11 Media Storage Application Profiles fix medium, format, object types and transfer syntaxes for specific uses (e.g. portable ultrasound CD, cardiology study on CD for patient transfer).
- MIME type application/dicom (RFC 3240); UTI org.nema.dicom.

- File Set roles (media AE roles, listed in Conformance Statement):
| Role | Operations | Rules |
|---|---|---|
| File Set Creator (FSC) | M-WRITE | Creates DICOMDIR (Basic Directory SOP Class with directory records for all image SOP Classes stored) plus zero or more files; must offer finalize, multi-session, or packet-writing if medium supports it |
| File Set Reader (FSR) | M-READ | Reads DICOMDIR and listed SOP Instance files; must not modify any file including DICOMDIR; typical: workstations, archives, modalities receiving outside patient media |
| File Set Updater (FSU) | M-READ, M-WRITE, M-DELETE | May add/delete files and update DICOMDIR; must not modify other DICOM files; typical: workstations that process images |
- Roles are tied to Application Profiles (e.g. STD-GEN); a Conformance Statement states the supported profile and transfer syntaxes.

### Layers (PS3.10/PS3.11/PS3.12)
- Data Format Layer: media storage SOP Classes/IODs, DICOM File Format, Secure DICOM File Format, Media Storage Directory SOP Class, Application Profiles, Security Profiles.
- Media Format Layer: file system / directory structures on the physical medium (OS dependent; PS3.12 selects ISO 9660 and others). Access through the DICOM File Service keeps the data layer independent from medium.
- No negotiation exists on media; Application Profiles (PS3.11) fix the permitted transfer syntaxes. The General Purpose CD/DVD profiles require Explicit VR Little Endian (Implicit VR Little Endian is not permitted there); other profiles add specific compressed syntaxes (JPEG, JPEG 2000, RLE and others, as each profile defines). [NORMATIVE: PS3.11]

### DICOM file format
- 128-byte File Preamble, then 4-byte prefix "DICM", then File Meta Information (group 0002 elements; mandatory in every file), then dataset.
- Group 0002 may appear only in File Meta Information. It carries SOP Class, SOP Instance UID, Transfer Syntax, implementation info, creator.
- File Meta Information is always Explicit VR Little Endian; it names the data set's transfer syntax in Transfer Syntax UID (0002,0010).

### File-set and File IDs
- File ID: up to 8 components (path folders plus file name), each up to 8 characters from A-Z, 0-9, underscore.
- No extension such as .dcm allowed on conformant media.
- DICOMDIR: mandatory single-component File ID in the file-set root; holds the Media Storage Directory SOP Class. Flat index of patient / study / series / image records pointing to files, so readers do not open every file.
- Roles: File-set Creator (FSC), Reader (FSR), Updater (FSU).

### Application Profile structure
- Specifies: clinical context, SOP Classes (required/optional per role), media format, transfer syntaxes, security profile, other interoperability choices (e.g. color encoding).
- Profile document sections: X.1 class and profile IDs; X.2 clinical context and roles; X.3 class-wide parameters (X.3.1 SOP Classes and transfer syntaxes, X.3.2 media/formats, X.3.3 DICOMDIR directory records, X.3.4 other, X.3.5 security); X.4+ specific profiles.
- If X.3.5 is absent, the profile is unsecured and Secure DICOM File Format must not be used; if present it is a Secure Media Storage Application Profile.

| Profile | Use |
|---|---|
| STD-XABC-CD | Basic cardiac X-ray angiography on CD-R; single/multi-frame up to 512x512x8; biplane stored as two single-plane objects; JPEG lossless |
| STD-CTMR-CD, -DVD, -DVD-RAM, -MOD41 | CT/MR; single-frame 8/12/16-bit grayscale and 8-bit palette color; uncompressed or lossless |
| STD-DEN-CD | Dental radiographs on CD |
| STD-GEN-CD / STD-GEN-SEC-CD | General purpose on 120 mm CD-R, ISO/IEC 9660; any SOP Class |
| STD-GEN-DVD-RAM / -SEC-DVD-RAM | General purpose, 120 mm DVD-RAM |
| STD-GEN-BD / -SEC-BD | General purpose, 120 mm Blu-ray |

- US/cardiology studies often need compression to fit one disc.
- IHE Portable Data for Imaging (PDI): restates media requirements and common mistakes for CD/DVD/USB, with import, print and optional browser display.
- Frequent failure: discs with proprietary image formats or viewers only, which cannot be loaded into a PACS for comparison.

## Security and De-identification

- Application-level: access control, authentication (passwords, biometrics), audit logging of who accessed data.
- DICOM-level: secure transport (TLS profiles in PS3.15), digital signatures for integrity and origin, and attribute-level or whole-object encryption for confidentiality: three separate mechanisms.
- Protects against alteration unnoticed by the receiver (signatures) and exposure over untrusted networks (encryption). IHE also defines security profiles.

## Conformance Statements

- PS3.2 defines conformance; an application may claim DICOM conformance only with an accurate Conformance Statement.
- Statement lists supported SOP Classes (as SCU/SCP), transfer syntaxes, roles, optional features (e.g. relational query support), extended negotiation, and PDU sizes. Network administrators use it to plan interfaces.

- Defined by PS3.2; a product claiming DICOM conformance must supply one, organized as PS3.2 prescribes: application descriptions, SOP Classes, roles, transfer syntaxes, association limits, network/media profiles, extensions. (HL7 has no comparably strict mechanism.)
- Interoperability check, in order:
  1. Compare application descriptions.
  2. Match SOP Classes (SOP Class UIDs).
  3. Match roles (SCU/SCP, FSC/FSR/FSU).
  4. Check number of simultaneous associations.
  5. Compare presentation contexts (abstract and transfer syntaxes).
  6. Compare communication profiles.
  7. Check special attribute requirements (private attributes, required values).
- Any SOP Class under a root other than 1.2.840.10008 is private and must be described fully in the statement.

- Statement lists: SOP Classes and roles (SCU/SCP), transfer syntaxes selected when several proposed, association length/timeouts, maximum simultaneous associations, optional Q/R keys, relational query support, Meta SOP Class support, print attribute values (e.g. Configuration Information), coercion behavior and storage conformance level.

## DICOMweb

- WADO (Web Access to DICOM Objects) originated jointly with the European standards body: retrieve a DICOM persistent object by HTTP URL/URI using its UIDs (study, series, SOP Instance).
- Request chooses result format via MIME content type: application/dicom or image/jpeg for images; application/dicom, RTF, XML (or HL7 CDA) for reports; content-encoding options.
- Implementations vary: a gateway may translate requests into DICOM SCU operations against an archive, while native DICOMweb servers serve objects directly without DICOM networking.
- UIDs are found from a referencing document message or from a query of other systems.
- Described in PS3.18; the current form is the DICOMweb family (WADO-RS, STOW-RS, QIDO-RS plus legacy WADO-URI).

## HL7, IHE and System Integration

- HL7 messages in imaging workflow: ORM (general order; new, update, cancel orders and patient demographics) and SIU (scheduling information unsolicited; appointment info from scheduling system to practice systems).
- RIS/HIS to modality via MWL; modality to RIS/PACS via MPPS; modality to archive via C-STORE then Storage Commitment.

- RIS: registration, tracking, results management and distribution, image tracking; may include scheduling, billing, HIS/PACS interfaces.
- HIS: admission, billing, sometimes EHR.
- Workflow loop: HIS/RIS order -> MWL to modality -> images (C-STORE) -> MPPS to RIS/PACS -> Storage Commitment -> modality may delete -> viewing via Q/R.
- IHE: MPPS forwarded by manager to both RIS and PACS; Key Image Note profile.
