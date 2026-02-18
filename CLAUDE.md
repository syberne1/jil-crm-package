# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Salesforce CRM package for **JIL Sovereign** managing two user types: **Token Purchasers** (crypto token buyers) and **Grant Recipients** (501(c)(3) nonprofits receiving milestone-based grants). This is a metadata-only package (no Apex, LWC, or Flows yet) using legacy Metadata API format with API version 59.0.

## File Structure

```
jil-crm-package/
├── CLAUDE.md                    # This file — Claude Code project context
├── README.md                    # Full technical review and documentation
├── package.xml                  # Metadata API manifest (9 objects, 3 permission sets)
├── .swift-sfdc.json             # Swift-SFDC deployment config
├── objects/
│   ├── Token_Purchaser__c.object
│   ├── Crypto_Wallet__c.object
│   ├── Crypto_Transaction__c.object
│   ├── Grant_Recipient__c.object
│   ├── Organization_Document__c.object
│   ├── Grant_Project__c.object
│   ├── Project_Milestone__c.object
│   ├── Milestone_Documentation__c.object
│   └── Milestone_Payment__c.object
└── permissionsets/
    ├── JIL_CRM_Admin.permissionset
    ├── JIL_Grant_Recipient_Portal.permissionset
    └── JIL_Token_Purchaser_Portal.permissionset
```

## Deployment Commands

```bash
# Deploy to a Salesforce org
sf project deploy start --source-dir . --target-org YOUR_ORG_ALIAS

# Legacy alternative
sfdx force:source:deploy -p . -u YOUR_ORG_ALIAS

# Validate only (check deploy without committing)
sf project deploy start --source-dir . --target-org YOUR_ORG_ALIAS --dry-run

# Retrieve latest from org
sf project retrieve start --target-org YOUR_ORG_ALIAS --manifest package.xml
```

There is no build, lint, or test setup — this is pure Salesforce XML metadata.

## Useful SOQL Queries

```bash
# Query token purchasers with KYC status
sf data query -q "SELECT Name, KYC_Status__c, Email__c FROM Token_Purchaser__c" --target-org YOUR_ORG_ALIAS

# Query grant recipients with project count
sf data query -q "SELECT Name, Organization_Name__c, (SELECT Name FROM Grant_Projects__r) FROM Grant_Recipient__c" --target-org YOUR_ORG_ALIAS

# Query milestone payments pending approval
sf data query -q "SELECT Name, Payment_Amount__c, Payment_Status__c FROM Milestone_Payment__c WHERE Payment_Status__c = 'Pending'" --target-org YOUR_ORG_ALIAS
```

## Architecture

### Two-Module Design

**Token Purchaser Module** — 3 objects:
- `Token_Purchaser__c` → `Crypto_Wallet__c` (multi-wallet, multi-network)
- `Token_Purchaser__c` → `Crypto_Transaction__c` (also linked to wallet)

**Grant Recipient Module** — 6 objects in a 4-level hierarchy:
- `Grant_Recipient__c` → `Organization_Document__c` (501(c)(3) vetting docs)
- `Grant_Recipient__c` → `Grant_Project__c` → `Project_Milestone__c` → `Milestone_Documentation__c`
- `Project_Milestone__c` → `Milestone_Payment__c` (also linked to Grant_Recipient__c)

All relationships are **Lookups** (not Master-Detail) with SetNull delete constraint.

### Key Integration Points

- **Onfido KYC**: Both `Token_Purchaser__c` and `Grant_Recipient__c` have identical Onfido field sets (Applicant ID, Check ID, Results, Reports, Webhook Response, KYC Status/Risk/Expiration).
- **Blockchain**: Transaction and payment objects track TX hashes, network, block numbers, confirmations, gas fees. Supported networks: JIL, Ethereum, Bitcoin, Solana, Polygon, and others.

### Permission Sets (3)

- `JIL_CRM_Admin` — full CRUD + Modify All / View All on all 9 objects
- `JIL_Grant_Recipient_Portal` — scoped for Experience Cloud portal (read own record, create docs, read payments)
- `JIL_Token_Purchaser_Portal` — scoped for portal (read/edit own record, manage wallets, read transactions)

## Conventions

- Object files use `.object` extension in `objects/` directory
- Permission sets use `.permissionset` extension in `permissionsets/`
- Auto-number display formats: `TP-{0000}`, `WALLET-{0000}`, `TX-{000000}`, `GR-{0000}`, `DOC-{000000}`, `PROJ-{0000}`, `MS-{00000}`, `MSDOC-{000000}`, `PAY-{000000}`
- Field history tracking is enabled on status, amount, and approval fields
- Currency fields have trending enabled for analytics
- All Metadata API version: 59.0
- All custom fields follow `Field_Name__c` naming convention with clear, descriptive names

## Coding Standards (for future Apex/LWC additions)

- Apex classes: PascalCase (e.g., `OnfidoWebhookHandler.cls`)
- Apex test classes: append `Test` suffix (e.g., `OnfidoWebhookHandlerTest.cls`)
- Minimum 85% code coverage per class; target 90%+
- LWC components: camelCase folder names (e.g., `tokenPurchaserCard/`)
- All Apex must include proper error handling and bulkification
- Use Custom Metadata Types or Custom Settings for configuration (not hardcoded values)

## Security Considerations

- **Critical**: Portal permission sets must enforce record-level access — pair with OWD settings and sharing rules
- Onfido webhook endpoint must validate callback signatures before processing
- Crypto wallet addresses and TX hashes should have format validation rules per network
- KYC data fields (PII) require encryption at rest — consider Salesforce Shield Platform Encryption
- All API integrations (Onfido, blockchain) should use Named Credentials, not hardcoded keys

## Roadmap / TODO

### Phase 1 — Data Integrity (Next)
- [ ] Add validation rules: wallet address format per network, TX hash format, email format
- [ ] Add duplicate rules for Token_Purchaser__c (by email) and Grant_Recipient__c (by EIN/org name)
- [ ] Evaluate converting critical Lookups to Master-Detail (e.g., Milestone_Payment → Project_Milestone)

### Phase 2 — Automation
- [ ] Apex trigger/class for Onfido KYC webhook handling (shared logic for both modules)
- [ ] Flow: Milestone approval workflow (submit → review → approve/reject → trigger payment)
- [ ] Flow: Auto-update Grant_Project status based on child milestone completion percentage
- [ ] Platform Events for blockchain transaction confirmations

### Phase 3 — Portal & UI
- [ ] Experience Cloud site metadata for Token Purchaser and Grant Recipient portals
- [ ] LWC components: KYC status tracker, milestone progress dashboard, wallet management
- [ ] Reports and dashboards for admin oversight

### Phase 4 — Infrastructure
- [ ] Convert from legacy Metadata API format to SFDX source format
- [ ] Set up scratch org definition file for development
- [ ] Add CI/CD pipeline (GitHub Actions + sf CLI)
- [ ] Implement Salesforce Shield Platform Encryption for PII fields
