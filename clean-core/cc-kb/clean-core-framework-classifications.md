# Clean Core Framework & Technology Classifications

Source: SAP Note (Clean Core extensibility classification reference)

## Overview

Frameworks, technologies, and development patterns are classified by clean core level (cloud readiness and upgrade stability). Some have conditional/multiple levels depending on usage.

## Conditional Level Classification

A framework or technology may have multiple clean core levels depending on how it is used:

- **Technology vs. SAP object usage**: Creating own CDS views (Technology = Level A) vs. consuming SAP-delivered CDS views (level depends on individual view's classification)
- **Released vs. classic APIs**: If both C1 released APIs and classic APIs exist for the same framework:
  - Released APIs = Level A (upgrade stable + cloud ready)
  - Classic APIs = Level B (upgrade stable only)
  - SAP-internal APIs = Level C
  - Obsolete APIs = Level D

Unless explicitly stated as "Usage of individual SAP..." the classifications refer to usage of the framework/technology in general.

## Clean Core Levels: Extensibility Only

The level concept applies exclusively to **clean core dimension extensibility**, including:

- Calling an API of a framework from within custom/partner code
- Extending SAP standard objects or implementing SAP enhancements (e.g., BAdIs)
- Creating own objects in customer/partner namespace (e.g., own CDS views, Fiori apps)

**Does NOT apply to:**

- Calling standard SAP remote APIs (OData, WebService, IDoc, BAPI) from another system
- Using SAP standard functionality (archiving, ILM, batch jobs) without customer extensions

## Single-Implementation Extension Techniques

Some extension techniques only allow one concurrent active implementation (e.g., Customer Exits, Single Use BAdIs). Multiple implementations can cause short dumps. This does not negatively influence clean core level; it's the customer's responsibility to avoid conflicts.

## API Classification Reference

APIs with their release state and classification are published at:
https://github.com/SAP/abap-atc-cr-cv-s4hc

Local copies in `reference/sap-api-reference/`:
- `objectReleaseInfoLatest.json`: 27,601 objects with release states
- `objectClassifications.json`: 2,731 classified APIs

### Release States → Clean Core Levels

| State | Count | Level | Meaning |
|-------|-------|-------|---------|
| `released` (C1 contract) | 25,982 | A | Cloud ready + upgrade stable |
| `notToBeReleasedStable` (classic API) | 455 | B | Upgrade stable only |
| `notToBeReleased` (SAP internal) | 648 | C | Not intended for customer use |
| `deprecated` (obsolete) | 516 | D | Scheduled for removal |

## References

- Clean Core extensibility white paper
- ABAP Development Tools documentation (C contracts)
- SAP/abap-atc-cr-cv-s4hc GitHub repository
