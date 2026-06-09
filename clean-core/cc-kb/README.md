# cc-kb: remediation knowledge base

Style and idiom guidance the `clean-core-remediator` reads when writing a replacement. It is reference for *how* to write a fix, never the source for *which* successor to use (successors come from SAP via the fix-step cascade).

Copy this directory into your project at `.abapctl/reference/cc-kb/`.

## What ships here

- `clean-core-framework-classifications.md`: Clean Core framework and technology classifications (A/B/C/D reference).

## Optional: add SAP's published guidance

The remediator also benefits from SAP's own ABAP style and cloud-development references. These are Apache-2.0 licensed and maintained upstream, so download the files you want directly into this directory rather than relying on a bundled copy that can go stale:

- [SAP/styleguides](https://github.com/SAP/styleguides): `clean-abap/CleanABAP.md` (Clean ABAP rules) and `clean-abap/sub-sections/ModernABAPLanguageElements.md` (modern-ABAP transforms).
- [SAP-samples/abap-cheat-sheets](https://github.com/SAP-samples/abap-cheat-sheets): e.g. `14_ABAP_Unit_Tests.md` and `19_ABAP_for_Cloud_Development.md`, plus any other cheat sheets relevant to your codebase.

Drop the markdown files into `.abapctl/reference/cc-kb/` and the remediator picks them up on its next run. Add your team's own conventions or domain patterns the same way.
