# DICOM Notes 7: Troubleshooting, Tools and Other

## Troubleshooting and Tools

Checklist from the concepts above:
1. C-ECHO failing: check AE titles, ports, firewall, IP/DNS before suspecting services.
2. Association rejected or SOP Class not accepted: compare presentation contexts against Conformance Statements (SOP Class UID, transfer syntax, role).
3. Images missing in PACS but sent OK: check Storage Commitment, QA station queue, orphan images (archive files without DB entries).
4. Study unmatched or wrong patient: trim padding in strings, check Accession Number and MWL keys, procedure code mapping.
5. Query by series fails: supply parent-level keys; confirm whether relational queries are supported.

- Ping works but DICOM fails: C-ECHO to verify AE title/IP/port; configure both peers symmetrically.
- Stored object rejected: compare SOP Class and transfer syntax with the accepted presentation contexts in A-ASSOCIATE-AC.
- Private SOP Class UIDs (outside 1.2.840.10008) are usually not understood by other products, so those presentation contexts are typically rejected.
- C-GET: confirm Storage SCP role negotiation. C-MOVE: destination AE table, inbound firewall rules.
- Duplicate UIDs from clock resets or DST corrupt archives.
- Field errors: odd-length padding, SH over 16 chars, backslash misuse, "Last,First" instead of "Last^First".

| Symptom | Likely cause |
|---|---|
| Association not established | AE title case mismatch; no common abstract syntax; no common transfer syntax; max associations reached |
| "Not supported" for an image type | Peer lacks that SOP Class (e.g. mammography, Enhanced CT) |
| Images unreadable after transfer | Compression/private transfer syntax mismatch; viewer lacks Enhanced IOD support |
| C-MOVE reports success but nothing arrives / fails | Destination AE not configured at SCP; firewall blocks inbound association; consider C-GET |
| Query returns huge result set | Overly broad keys; cancel with C-FIND-CANCEL |
| Series query fails | Missing parent Patient/Study keys in hierarchical model |
| Print too small/cropped | Printable-area limits; set Requested Image Size and no-scale options |
| Print contrast differs from monitor | GSDF calibration or Presentation LUT/Configuration Information mismatch |
| Image deleted at modality but lost at archive | Relied on C-STORE success instead of Storage Commitment; or archive auto-purge |
| Mismatched demographics in PACS | No MWL, manual entry, or receiver coercion |

| Symptom | Likely cause |
|---|---|
| Image appears inverted | Photometric Interpretation (MONOCHROME1 vs MONOCHROME2) not honored |
| Half of tonal range inverted, odd bright/dark noise | Pixel Representation (signed) ignored, or spare high bits not masked |
| CT values shifted by ~1024 | Rescale Intercept/Slope not applied |
| Image looks flat/very different after receipt | Embedded VOI/Modality LUT not supported by receiver |
| Image squeezed in one axis | Pixel Aspect Ratio ignored |
| Annotations or shutters missing at other workstation | Stored in proprietary form, or PR SOP Class not supported by PACS/viewer |
| Overlays missing | Receiver ignores group 60xx or only supports group 6000 |
| White borders on collimated images | No shutter applied/PR not retrieved |
| Landscape-rotated image | Rotation not applied; no Spatial Transformation |
| Generation loss after repeated lossy compression | Lossy compression history not honored; avoid recompressing lossy images |
| Disc will not import | Non-conformant file IDs, missing DICOMDIR, proprietary format, non-Explicit VR |

- To test overlay or LUT support, inspect headers for group 60xx, (0028,3010), (0028,1050)/(0028,1051).

## Other

### Display calibration (Grayscale Standard Display Function, PS3.14)
- GSDF maps digital driving levels to luminance on a perceptually linear scale built from just noticeable differences (JNDs); based on human vision, not on a specific device, modality or preference. Eye is less sensitive in dark regions, so dark levels are compressed and bright levels stretched.
- Applies to monitors and to hardcopy (film/paper); both must be GSDF-calibrated for matching appearance.
- Calibration: measure luminance at a series of levels from black to white and compare to the GSDF curve; best with a front-of-screen photometer. Built-in sensors can measure continuously; manual calibration is a periodic cost trade-off. Backlights age, so luminance drifts.
- Diagnostic displays: calibrated internally with a LUT converting graphic RGB to 10-12-bit driving levels; software sets Lmax, luminance ratio/Lmin and ambient compensation. Typical resolution 1536x2048 or higher; many are color IPS-TFT LCD that still provide calibrated gray. Luminance of 350 cd/m2 minimum is cited for diagnostic use; medical panels reach 1000 cd/m2 or more.
- ICC profiles characterize color devices via a profile connection space (CIELAB or CIEXYZ); current ICC spec version 4.3 [verify].
