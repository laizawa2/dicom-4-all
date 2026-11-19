# DICOM_4_ALL

AI-powered technical assistant for healthcare imaging and interoperability professionals: DICOM standards, SOP Classes and IODs, attributes and VRs, transfer syntaxes, DIMSE, association negotiation, DICOMweb (QIDO-RS, WADO-RS, STOW-RS), logs, conformance statements, PACS/VNA integration, modality workflows, HL7, FHIR, and IHE.

It does not provide medical diagnosis, treatment advice, patient-specific clinical interpretation, or clinical image interpretation.

## Privacy: remove all PHI first
You are responsible for removing all PHI, patient identifiers, physician and other healthcare-professional identifiers, and anything else that could identify an individual from DICOM files, headers, datasets, screenshots, logs, and documents before using this plugin. Use synthetic or appropriately de-identified examples only.

- Privacy policy: https://laizawa2.github.io/dicom-4-all-privacy/index.html
- Terms: https://laizawa2.github.io/dicom-4-all-privacy/terms.html
- Support: https://www.everyexpert.com/luizaizawa

## Install

Claude Code:
```
/plugin marketplace add laizawa2/dicom-4-all
/plugin install dicom-4-all@dicom-4-all
```

claude.ai / Claude desktop (Cowork): Customize > Plugins > Add > Add marketplace, then enter `laizawa2/dicom-4-all` (or Add > Upload plugin with the .zip).



## Example prompts
- Explain this DICOM association failure and identify the next diagnostic checks.
- Validate this sanitized DICOM header against its SOP Class and IOD requirements.
- Review this DICOMweb request, including its URL, headers, and Accept values.

## Standard references
The skill consults the current DICOM Standard online; see `skills/dicom-4-all/references/DICOM_PART_LINKS.txt`.

## License

Copyright (c) 2026 Luiz Aizawa. All rights reserved. Free to use, but not to copy, modify or redistribute. See [LICENSE](LICENSE).
