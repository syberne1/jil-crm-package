# JIL Sovereign CRM Package

A comprehensive Salesforce CRM package for managing Token Purchasers and Grant Recipients with full grant lifecycle management.

## Package Overview

This package provides two separate profile types (per requirements) with full portal authentication support:

### 1. Token Purchaser Module
For individuals purchasing crypto tokens on the JIL platform.

### 2. Grant Recipient Module  
For 501(c)(3) nonprofits applying for and receiving grants with iterative milestone-based funding.

---

## Object Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           TOKEN PURCHASER MODULE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────┐     ┌─────────────────┐     ┌────────────────────┐   │
│  │ Token_Purchaser  │────▶│  Crypto_Wallet  │     │ Crypto_Transaction │   │
│  │                  │     │                 │◀────│                    │   │
│  │ • Personal Info  │     │ • Address       │     │ • TX Hash          │   │
│  │ • Portal Access  │     │ • Network       │     │ • Token/Amount     │   │
│  │ • KYC/Onfido     │     │ • Verification  │     │ • Status           │   │
│  │ • Social Handles │     │ • Is Primary    │     │ • Payment Method   │   │
│  │ • User Tier      │     └─────────────────┘     └────────────────────┘   │
│  └──────────────────┘                                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                           GRANT RECIPIENT MODULE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────┐     ┌─────────────────────┐                          │
│  │ Grant_Recipient  │────▶│ Organization_Document│                          │
│  │                  │     │                     │                          │
│  │ • Org Info       │     │ • IRS Letter        │                          │
│  │ • 501(c)(3)      │     │ • State AP205       │                          │
│  │ • EIN            │     │ • Tax Returns (2yr) │                          │
│  │ • State Reg      │     │ • Prior Projects    │                          │
│  │ • Primary Contact│     │ • Review Status     │                          │
│  │ • Portal/KYC     │     └─────────────────────┘                          │
│  │ • Social Handles │                                                       │
│  │ • Vetting Status │                                                       │
│  └────────┬─────────┘                                                       │
│           │                                                                 │
│           ▼                                                                 │
│  ┌──────────────────┐     ┌─────────────────────┐                          │
│  │  Grant_Project   │────▶│  Project_Milestone  │                          │
│  │                  │     │                     │                          │
│  │ • Project Name   │     │ • Milestone #       │                          │
│  │ • Description    │     │ • Funds Requested   │                          │
│  │ • Funds Request  │     │ • Timeline/Dates    │                          │
│  │ • Use Plan       │     │ • Deliverables      │                          │
│  │ • Timeline       │     │ • Doc Requirements  │                          │
│  │ • Status         │     │ • Payment Status    │                          │
│  │ • Prior Projects │     └──────────┬──────────┘                          │
│  └──────────────────┘                │                                      │
│                                      │                                      │
│                         ┌────────────┴────────────┐                        │
│                         ▼                         ▼                        │
│           ┌─────────────────────┐   ┌─────────────────────┐                │
│           │Milestone_Documentation│ │  Milestone_Payment  │                │
│           │                     │   │                     │                │
│           │ • Progress Reports  │   │ • Amount USD/Crypto │                │
│           │ • Financial Reports │   │ • Payment Method    │                │
│           │ • Receipts          │   │ • TX Hash           │                │
│           │ • Impact Metrics    │   │ • Status            │                │
│           │ • Review Status     │   │ • Approval          │                │
│           └─────────────────────┘   └─────────────────────┘                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Custom Objects

| Object | API Name | Description |
|--------|----------|-------------|
| Token Purchaser | `Token_Purchaser__c` | Individual crypto token buyers with KYC, wallets, social handles |
| Crypto Wallet | `Crypto_Wallet__c` | Wallet addresses linked to Token Purchasers |
| Crypto Transaction | `Crypto_Transaction__c` | All token purchases/transactions on platform |
| Grant Recipient | `Grant_Recipient__c` | 501(c)(3) nonprofits applying for grants |
| Organization Document | `Organization_Document__c` | Uploaded docs (IRS letters, tax returns, AP205) |
| Grant Project | `Grant_Project__c` | Individual funding requests by grant recipients |
| Project Milestone | `Project_Milestone__c` | Iterative funding tranches with deliverables |
| Milestone Documentation | `Milestone_Documentation__c` | Proof docs for milestone completion |
| Milestone Payment | `Milestone_Payment__c` | Actual disbursements with payment tracking |

---

## Key Features

### Token Purchaser Features
- **KYC/Onfido Integration**: Full Onfido field set (Applicant ID, Check ID, Results, Reports)
- **Multi-Wallet Support**: Multiple wallets per user across networks (JIL, ETH, BTC, etc.)
- **Transaction History**: Complete purchase history with TX hashes, amounts, status
- **Social Profiles**: Twitter, Discord, Telegram, LinkedIn, GitHub, Instagram
- **User Tiering**: Bronze → Silver → Gold → Platinum → Institutional
- **Portal Access**: Username, enabled flag, last login tracking

### Grant Recipient Features
- **501(c)(3) Vetting**: 
  - IRS Determination Letter checkbox
  - State AP205/equivalent checkbox  
  - 2 years tax returns checkboxes
  - EIN capture
- **Multi-Project Support**: Recipients can have multiple grant projects
- **Milestone-Based Funding**: 
  - Projects broken into numbered milestones
  - Each milestone has its own funding amount and timeline
  - Documentation requirements per milestone
- **Iterative Disbursement**:
  - Documentation upload triggers review
  - Admin approval required before payment
  - Payment records with crypto TX hashes
- **Prior Project Tracking**: Evidence of past successful projects

### Grant Workflow
1. **Application**: Recipient creates profile, uploads 501(c)(3) docs
2. **Vetting**: Admin reviews IRS letter, state reg, tax returns
3. **Project Submission**: Recipient creates project with use plan, milestones
4. **Approval**: Admin approves project and funding amounts
5. **Milestone Execution**: Recipient completes work, uploads documentation
6. **Review**: Admin reviews milestone documentation
7. **Payment**: On approval, payment is processed and recorded
8. **Iteration**: Process repeats for subsequent milestones

---

## Permission Sets

| Permission Set | API Name | Use Case |
|----------------|----------|----------|
| JIL CRM Admin | `JIL_CRM_Admin` | Full admin access to all objects |
| Grant Recipient Portal | `JIL_Grant_Recipient_Portal` | Portal users - grant recipients |
| Token Purchaser Portal | `JIL_Token_Purchaser_Portal` | Portal users - token buyers |

---

## Deployment Instructions

### Prerequisites
- Salesforce CLI (sf or sfdx)
- Authenticated to target org

### Deploy Package

```bash
# Navigate to package directory
cd jil-crm-package

# Deploy to org
sf project deploy start --source-dir . --target-org YOUR_ORG_ALIAS

# Or using older sfdx command
sfdx force:source:deploy -p . -u YOUR_ORG_ALIAS
```

### Post-Deployment Steps

1. **Create Custom Tabs** for each custom object
2. **Add to App** - Create a Lightning App with relevant tabs
3. **Configure Portal** - Set up Experience Cloud for external users
4. **Onfido Integration** - Configure webhook endpoint to update KYC fields
5. **Sharing Rules** - Set up OWD and sharing rules for portal access
6. **Validation Rules** - Add business logic validation as needed
7. **Flows/Triggers** - Build automation for:
   - Auto-populate KYC Expiration Date
   - Roll-up milestone counts on projects
   - Notification on documentation submission
   - Payment status sync from blockchain

---

## Onfido Integration Points

The following fields are designed for Onfido webhook integration:

**On Token_Purchaser__c and Grant_Recipient__c:**
- `Onfido_Applicant_ID__c` - Applicant ID from Onfido
- `Onfido_Check_ID__c` - Check ID from verification
- `Onfido_Check_Result__c` - Clear/Consider/Rejected
- `Onfido_Check_Date__c` - When check completed
- `Onfido_Document_Report_ID__c` - Document verification report
- `Onfido_Facial_Report_ID__c` - Facial similarity report
- `Onfido_Webhook_Response__c` - Raw JSON for debugging
- `KYC_Status__c` - Overall status (Not Started → Approved)
- `KYC_Risk_Level__c` - Low/Medium/High
- `KYC_Expiration_Date__c` - When re-verification needed

---

## File Structure

```
jil-crm-package/
├── package.xml
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
├── permissionsets/
│   ├── JIL_CRM_Admin.permissionset
│   ├── JIL_Grant_Recipient_Portal.permissionset
│   └── JIL_Token_Purchaser_Portal.permissionset
└── README.md
```

---

## Support

For questions or customization needs, contact the JIL Sovereign development team.
