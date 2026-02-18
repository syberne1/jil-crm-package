# JIL Sovereign CRM — Security Audit Report

**Audit Date:** February 16, 2026
**Codebase:** `/Users/Stanley_1/Repos/jil-crm-package`
**Scope:** All 9 custom objects, 3 permission sets, package.xml, validation rules, duplicate rules
**Auditor:** Claude (Technical Lead Review)

---

## Executive Summary

The package has a solid data model foundation with well-designed object relationships, good validation rules (wallet address formats, EIN, email), and sensible duplicate detection. However, **it is NOT deployment-ready.** There are **7 CRITICAL**, **5 HIGH**, and **6 MEDIUM** findings that must be resolved before this touches a production org — especially given the financial transaction and KYC/AML compliance requirements.

**Business impact if deployed as-is:** Regulatory exposure (KYC data unencrypted), financial data leakage (no FLS on currency fields), and audit failures (no sharing rules enforce record isolation).

---

## CRITICAL Findings (Must Fix Before Any Deployment)

### C-1: No Field-Level Security (FLS) on ANY Field

**Severity:** CRITICAL
**Exploit scenario:** Any authenticated user with object access can read/edit ALL fields — including PII (DOB, addresses, EIN), financial amounts (USD_Amount, Total_Funds_Requested), and KYC data (Onfido IDs, Check Results). A portal user with Token Purchaser Portal access could view every purchaser's personal info.

**Current state:** All three permission sets define only object-level permissions (`<objectPermissions>`). Zero `<fieldPermissions>` entries exist in any `.permissionset` file.

**What's exposed:**

| Permission Set | Unprotected Sensitive Fields |
|---|---|
| JIL_Token_Purchaser_Portal | Email, DOB, Address, KYC fields, Onfido IDs, wallet addresses, USD amounts on ALL Token Purchaser records (though OLS is Private, FLS is wide open) |
| JIL_Grant_Recipient_Portal | EIN, Primary Contact PII, Annual Budget, Disbursement Wallet, all financial amounts |
| JIL_CRM_Admin | Appropriate (admin), but still needs explicit FLS for audit trail |

**Fix — effort: ~4 hours:**
Add `<fieldPermissions>` blocks to each permission set. Example for Token Purchaser Portal:

```xml
<!-- Token Purchaser Portal: DENY access to admin-only fields -->
<fieldPermissions>
    <editable>false</editable>
    <field>Token_Purchaser__c.KYC_Risk_Level__c</field>
    <readable>false</readable>
</fieldPermissions>
<fieldPermissions>
    <editable>false</editable>
    <field>Token_Purchaser__c.Onfido_Webhook_Response__c</field>
    <readable>false</readable>
</fieldPermissions>
<!-- ALLOW read-only on own KYC status -->
<fieldPermissions>
    <editable>false</editable>
    <field>Token_Purchaser__c.KYC_Status__c</field>
    <readable>true</readable>
</fieldPermissions>
```

**Fields that need RESTRICTED access (read:false, edit:false) for portal users:**

Token Purchaser Portal:
- `Onfido_Webhook_Response__c`, `Onfido_Applicant_ID__c`, `Onfido_Check_ID__c`
- `Onfido_Document_Report_ID__c`, `Onfido_Facial_Report_ID__c`
- `KYC_Risk_Level__c`, `KYC_Expiration_Date__c`
- `Total_Purchase_Volume__c`, `Total_Transactions__c` (admin rollup only)
- `Notes__c` (admin internal notes)
- `Account_Status__c` (read-only at most, never editable)

Grant Recipient Portal:
- All `Onfido_*` fields
- `KYC_Risk_Level__c` (if present)
- `Vetting_Notes__c`, `Vetting_Status__c` (read-only at most)
- `Annual_Budget__c` (editable by owner, not other portal users)
- `EIN__c` (read-only after submission)
- On Grant_Project: `Review_Notes__c`, `Rejection_Reason__c`, `Approved_By__c`
- On Milestone_Payment: `From_Wallet_Address__c`, `Bank_Reference_Number__c`, `Check_Number__c`


### C-2: No Platform Encryption on PII/KYC Fields

**Severity:** CRITICAL
**Exploit scenario:** Database exports, backup tapes, or compromised storage exposes plaintext PII — DOB, addresses, EIN, Onfido IDs, KYC documents. This is a KYC/AML compliance violation.

**Affected fields requiring Shield Platform Encryption:**

| Object | Field | Data Classification |
|---|---|---|
| Token_Purchaser__c | Date_of_Birth__c | PII |
| Token_Purchaser__c | Email__c | PII |
| Token_Purchaser__c | Street_Address__c | PII |
| Token_Purchaser__c | Onfido_Applicant_ID__c | KYC Sensitive |
| Token_Purchaser__c | Onfido_Check_ID__c | KYC Sensitive |
| Token_Purchaser__c | Onfido_Webhook_Response__c | KYC Sensitive (raw JSON) |
| Grant_Recipient__c | EIN__c | PII/Tax |
| Grant_Recipient__c | Primary_Contact_Email__c | PII |
| Grant_Recipient__c | Disbursement_Wallet_Address__c | Financial |
| Grant_Recipient__c | Onfido_Applicant_ID__c | KYC Sensitive |
| Crypto_Wallet__c | Wallet_Address__c | Financial |

**Fix — effort: ~2 hours (config) + Shield license required:**
1. Enable Shield Platform Encryption in the org
2. Create a tenant secret
3. Apply Deterministic Encryption to fields that need filtering (Email, EIN)
4. Apply Probabilistic Encryption to fields that don't need filtering (DOB, Onfido IDs, Webhook Response)
5. Note: Encrypted fields can't be used in SOQL WHERE clauses with probabilistic encryption — plan queries accordingly

**Note:** The `CLAUDE.md` and `README.md` both acknowledge this is needed (Phase 4 roadmap) but it should be Phase 0, not Phase 4.


### C-3: Grant_Recipient__c OLS is Private but No Sharing Rules Exist

**Severity:** CRITICAL
**Exploit scenario:** The `sharingModel` is `Private` and `externalSharingModel` is `Private` — which is correct. But there are ZERO sharing rules defined in the package. Portal users with Grant Recipient Portal permission set **cannot see their own records** because:
- `viewAllRecords` = false (correct)
- No owner-based or criteria-based sharing rules grant access
- No sharing sets for Experience Cloud portal users

This means the portal is completely broken for grant recipients — they see nothing.

**Similarly affected:** Token_Purchaser__c, Crypto_Wallet__c, Crypto_Transaction__c, all Grant module objects.

**Fix — effort: ~3 hours:**
1. Add `sharingRules/` directory with sharing rule XML for each object
2. For Experience Cloud portal: Configure Sharing Sets that grant portal users access to records where they are the associated Contact/User
3. For internal: Add criteria-based sharing rules (e.g., Grant Reviewers can see records with `Vetting_Status__c = 'Under Review'`)
4. Add to `package.xml`: `<types><members>*</members><n>SharingRules</n></types>`


### C-4: All Lookups Use SetNull — No Referential Integrity

**Severity:** CRITICAL
**Exploit scenario:** Deleting a Grant_Recipient__c record silently orphans all Grant_Project__c, Organization_Document__c, and Milestone_Payment__c records. Financial records lose their parent association — payments become untraceable to recipients. This is an audit catastrophe for a nonprofit handling grant disbursements.

**Every relationship in the package uses:**
```xml
<deleteConstraint>SetNull</deleteConstraint>
<type>Lookup</type>
```

**Relationships that MUST be Master-Detail (or at minimum Restrict delete):**

| Parent → Child | Reason |
|---|---|
| Token_Purchaser → Crypto_Wallet | Wallet is meaningless without owner |
| Token_Purchaser → Crypto_Transaction | Transaction must always trace to purchaser |
| Crypto_Wallet → Crypto_Transaction (via Wallet__c) | TX must trace to wallet |
| Grant_Recipient → Grant_Project | Project can't exist without recipient |
| Grant_Recipient → Organization_Document | Doc is evidence for specific recipient |
| Grant_Project → Project_Milestone | Milestone belongs to project |
| Project_Milestone → Milestone_Payment | Payment must always trace to milestone |
| Project_Milestone → Milestone_Documentation | Doc proves milestone completion |

**Fix — effort: ~3 hours:**
Convert critical lookups to Master-Detail. This also gives you free rollup summary fields (replace the manual `Total_Milestones__c`, `Milestones_Completed__c` number fields with real rollups) and cascading sharing (solves C-3 partially).

**Caveat:** Master-Detail conversion requires the child's lookup field to be populated on ALL existing records. If you deploy to an empty org first (which is the plan), convert before loading any data.


### C-5: No Named Credentials for Onfido/External Integrations

**Severity:** CRITICAL
**Current state:** No `namedCredentials/` directory exists. The CLAUDE.md mentions "API integrations should use Named Credentials, not hardcoded keys" but nothing is implemented. The README references Onfido webhook integration but no credential management exists.

**Fix — effort: ~2 hours:**
1. Create `namedCredentials/` directory
2. Define `JIL_Onfido.namedCredential` with External Credential + Principal
3. Reference in any future Apex callout classes
4. Add to `package.xml`

```xml
<!-- namedCredentials/JIL_Onfido.namedCredential -->
<?xml version="1.0" encoding="UTF-8"?>
<NamedCredential xmlns="http://soap.sforce.com/2006/04/metadata">
    <label>JIL Onfido</label>
    <type>SecuredEndpoint</type>
    <endpoint>https://api.onfido.com/v3.6</endpoint>
    <principalType>NamedUser</principalType>
    <protocol>NoAuthentication</protocol>
</NamedCredential>
```


### C-6: Missing Field History Tracking on Financial Amount Fields

**Severity:** CRITICAL
**Exploit scenario:** Someone edits `Payment_Amount_USD__c` on a Milestone_Payment from $50,000 to $500,000. With no field history, there's zero audit trail. For a nonprofit handling grant disbursements, this is a compliance and fraud risk.

**Fields with `trackHistory: false` that MUST be `true`:**

| Object | Field | Why |
|---|---|---|
| Crypto_Transaction__c | Price_Per_Token__c | Price manipulation detection |
| Crypto_Transaction__c | Transaction_Fee__c | Fee tampering |
| Crypto_Transaction__c | Payment_Method__c | Payment method switching |
| Milestone_Payment__c | Payment_Amount_Crypto__c | Crypto amount alteration |
| Milestone_Payment__c | Exchange_Rate__c | Rate manipulation |
| Milestone_Payment__c | From_Wallet_Address__c | Source wallet change |
| Milestone_Payment__c | To_Wallet_Address__c | Destination wallet change |
| Milestone_Payment__c | Bank_Reference_Number__c | Bank ref alteration |
| Grant_Recipient__c | Annual_Budget__c | Budget inflation |
| Grant_Recipient__c | Disbursement_Wallet_Address__c | Wallet swap fraud |


### C-7: package.xml Uses Wildcards — Uncontrolled Deployment

**Severity:** CRITICAL
**Exploit scenario:** `<members>*</members>` on CustomObject, CustomField, Layout, etc. means a deploy will push EVERY matching metadata type from the directory — including anything accidentally added. In a CI/CD pipeline, this could deploy untested or malicious metadata.

**Current package.xml:**
```xml
<types><members>*</members><n>CustomObject</n></types>
<types><members>*</members><n>CustomField</n></types>
<!-- ... all wildcards ... -->
```

**Fix — effort: ~1 hour:**
Replace wildcards with explicit members:

```xml
<types>
    <members>Token_Purchaser__c</members>
    <members>Crypto_Wallet__c</members>
    <members>Crypto_Transaction__c</members>
    <!-- ... enumerate all 9 ... -->
    <n>CustomObject</n>
</types>
```

---

## HIGH Findings

### H-1: Token_Purchaser Lookup on Crypto_Wallet Is Not Required

**Severity:** HIGH
**Issue:** `Crypto_Wallet__c.Token_Purchaser__c` has `<required>false</required>`. A wallet can be created with no owner.

**Fix:** Set `<required>true</required>` (or convert to Master-Detail per C-4).


### H-2: No Indexes Defined on High-Query Fields

**Severity:** HIGH
**Issue:** Per your requirements, `Status__c`, `CreatedDate`, and `ContactId` should be indexed for selective SOQL. No custom indexes are declared. On Token_Purchaser: `KYC_Status__c`, `Account_Status__c`, `Email__c` will be in WHERE clauses. On Crypto_Transaction: `Transaction_Status__c`, `Transaction_DateTime__c`, `Payment_Method__c`.

**Fix — effort: Post-deployment (Setup → Custom Indexes or contact Salesforce Support):**
Custom indexes can't be declared in metadata — they must be requested via Salesforce Support or configured in Setup for custom objects. Document the required indexes and request them immediately after deployment.


### H-3: Grant Recipient Portal Can Edit Grant_Project But Shouldn't Edit Approved Projects

**Severity:** HIGH
**Issue:** The Grant Recipient Portal permission set grants `allowEdit: true` on Grant_Project__c. There's no validation rule preventing edits after a project is Approved/In Progress/Completed. A recipient could change `Total_Funds_Requested__c` on an already-approved project.

**Fix — effort: ~30 min:**
Add a validation rule on Grant_Project__c:

```xml
<validationRules>
    <fullName>Prevent_Edit_After_Approval</fullName>
    <active>true</active>
    <description>Prevents changes to key fields after project approval</description>
    <errorConditionFormula>AND(
    OR(
        ISPICKVAL(PRIORVALUE(Project_Status__c), "Approved"),
        ISPICKVAL(PRIORVALUE(Project_Status__c), "In Progress"),
        ISPICKVAL(PRIORVALUE(Project_Status__c), "Completed")
    ),
    OR(
        ISCHANGED(Total_Funds_Requested__c),
        ISCHANGED(Funds_Use_Plan__c),
        ISCHANGED(Project_Name__c)
    ),
    NOT($Permission.JIL_CRM_Admin)
)</errorConditionFormula>
    <errorMessage>Cannot modify funding or project details after approval. Contact admin.</errorMessage>
</validationRules>
```


### H-4: Milestone_Payment Has No Idempotency Key

**Severity:** HIGH
**Issue:** `Payment_Transaction_ID__c` on Project_Milestone__c exists but has `<unique>false</unique>`. On Milestone_Payment__c, `Transaction_Hash__c` also has `<unique>false</unique>`. Duplicate payments can be recorded with the same TX hash.

**Fix:** Set `<unique>true</unique>` on `Milestone_Payment__c.Transaction_Hash__c` and add a `Payment_Idempotency_Key__c` field with unique constraint.


### H-5: No Approval Process Metadata for Grant Projects

**Severity:** HIGH
**Issue:** The README describes an approval workflow (single approver <$50K, dual $50K-$250K, triple >$250K) but there's no `approvalProcesses/` directory in the package. The `Approved_By__c` lookup on Grant_Project__c is manually populated — anyone with edit access can set it.

**Fix — effort: ~3 hours:**
Create approval process metadata or Flows with approval steps. Add `<types><members>*</members><n>ApprovalProcess</n></types>` to package.xml.

---

## MEDIUM Findings

### M-1: Crypto_Transaction Lookups Allow Orphaned Transactions

Token_Purchaser__c and Wallet__c lookups on Crypto_Transaction__c both have `required: false`. A transaction can exist with no purchaser and no wallet — making it untraceable.

### M-2: No Record Type Separation

Token Purchaser module and Grant Recipient module are separate objects (good), but within objects like Organization_Document__c, there's no record type to distinguish IRS letters from tax returns for permission/layout purposes.

### M-3: Roll-up Fields Are Manual Number Fields

`Total_Milestones__c`, `Milestones_Completed__c`, `Active_Projects_Count__c`, `Total_Grants_Received__c`, `Total_Purchase_Volume__c`, `Total_Transactions__c` are all manually maintained number/currency fields. They'll drift from reality without triggers or flows. Converting to Master-Detail (C-4) would enable native Rollup Summary fields.

### M-4: No Data Classification Labels

None of the fields have `<complianceGroup>` or `<securityClassification>` metadata tags. For KYC/AML compliance, fields should be classified (PII, Financial, Confidential, etc.) to support data governance policies.

### M-5: Onfido Webhook Response Stores Raw JSON in LongTextArea

`Onfido_Webhook_Response__c` is a 32KB LongTextArea storing raw Onfido JSON. This likely contains PII (document images, facial recognition data, check details). It should either be encrypted, moved to a Big Object with limited access, or stored in Salesforce Files with restricted sharing.

### M-6: No Error/Audit Log Object

The package has no `ErrorLog__c` or `AuditLog__c` object for tracking integration failures, webhook processing, or admin actions. Per your requirements, all callouts should log to ErrorLog__c.

---

## What's Working Well

- **OLS defaults are correct:** All 9 objects have `sharingModel: Private` and `externalSharingModel: Private`
- **Validation rules are solid:** Wallet address format validation per blockchain network (EVM, Bitcoin, Solana), TX hash format validation, EIN format validation, email format validation
- **Duplicate detection specs:** Well-designed matching rules for Token_Purchaser (email) and Grant_Recipient (EIN + fuzzy org name)
- **Field history tracking** is enabled on most status and approval fields (with gaps noted in C-6)
- **Auto-number naming** is clean and consistent across all objects
- **Portal permission sets** have appropriate object-level restrictions (no delete, no viewAll/modifyAll)
- **Picklist values** are restricted (no free-text additions) on all status/type fields

---

## Recommended Fix Order (Priority Sequence)

| # | Finding | Effort | Blocker? |
|---|---------|--------|----------|
| 1 | C-4: Convert Lookups → Master-Detail | 3h | Yes — do first (before data load) |
| 2 | C-1: Add FLS to permission sets | 4h | Yes — deployment blocker |
| 3 | C-6: Enable field history on financial fields | 1h | Yes — compliance blocker |
| 4 | C-3: Add sharing rules / sharing sets | 3h | Yes — portal broken without this |
| 5 | H-3: Add post-approval validation rule | 30m | Yes — fraud prevention |
| 6 | H-4: Add unique constraint on TX hash | 30m | Yes — duplicate payment prevention |
| 7 | C-7: Replace package.xml wildcards | 1h | Yes — CI/CD safety |
| 8 | C-5: Create Named Credentials | 2h | Yes — before any Apex integration |
| 9 | C-2: Platform Encryption | 2h | Yes — but requires Shield license |
| 10 | H-5: Approval process metadata | 3h | Before grant workflows go live |
| 11 | H-2: Request custom indexes | 1h | Before high-volume usage |
| 12 | M-6: Create ErrorLog__c object | 2h | Before any Apex callouts |

**Total estimated effort: ~23 hours**

---

## Next Step

Begin implementing fixes in priority order, starting with C-4 (converting Lookups to Master-Detail) since that's the foundation everything else builds on and must happen before any data exists in the org.
