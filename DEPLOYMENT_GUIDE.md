# JIL CRM Package - Deployment & Restart Guide

**Last Updated:** February 18, 2026  
**Status:** PAUSED - Waiting on Salesforce account change to info@jilsovereign.com  
**Repository:** https://github.com/syberne1/jil-crm-package  
**Contact for SSH/Admin:** Jeff Mondonca

---

## 📋 Current Status

### ✅ Completed
- [x] All 9 custom Salesforce objects defined and stable
- [x] All 3 permission sets configured
- [x] Package.xml deployment manifest ready (API v59.0)
- [x] Code backed up to GitHub (syberne1@gmail.com account)
- [x] Documentation complete (README.md)
- [x] Object relationships validated (Lookups configured)

### ⏸️ Pending (Resume After Pause)
- [ ] Salesforce account changed to info@jilsovereign.com
- [ ] SSH access coordinated with Jeff Mondonca
- [ ] Salesforce CLI authenticated to new org
- [ ] Package deployed to Salesforce
- [ ] Custom tabs created
- [ ] Experience Cloud portal configured

---

## 🚀 Quick Start - When Resuming

### Step 1: Pull Latest Code from GitHub

```bash
cd /Users/Stanley_1/Repos/jil-crm-package
git pull origin main
```

### Step 2: Verify Salesforce Account Change

- Confirm with Salesforce that account is now **info@jilsovereign.com**
- Verify you can log in at: https://login.salesforce.com
- Account: **Global Hands** (Nonprofit/Charity status)

### Step 3: Install/Update Salesforce CLI

```bash
# Check if SF CLI is installed
sf --version

# If not installed, install it:
brew install sf

# If installed, update it:
brew upgrade sf
```

### Step 4: Authenticate to Salesforce

```bash
# Authenticate using web login
sf org login web --alias globalhands --instance-url https://login.salesforce.com

# This will:
# 1. Open your browser
# 2. Log in with info@jilsovereign.com credentials
# 3. Authorize the CLI
# 4. Save connection as "globalhands"

# Verify connection
sf org display --target-org globalhands

# List current objects (should be empty or default NPSP objects)
sf sobject list --target-org globalhands
```

### Step 5: Deploy Package to Salesforce

```bash
# Make sure you're in the package directory
cd /Users/Stanley_1/Repos/jil-crm-package

# Deploy all metadata
sf project deploy start --source-dir . --target-org globalhands

# Wait for deployment to complete
# You should see: "Deploy Succeeded"
```

### Step 6: Verify Deployment

```bash
# Check if objects were created
sf sobject list --target-org globalhands | grep "__c"

# You should see your 9 custom objects:
# - Token_Purchaser__c
# - Crypto_Wallet__c
# - Crypto_Transaction__c
# - Grant_Recipient__c
# - Organization_Document__c
# - Grant_Project__c
# - Project_Milestone__c
# - Milestone_Documentation__c
# - Milestone_Payment__c
```

---

## 🔧 Post-Deployment Configuration

### 1. Create Custom Tabs

In Salesforce Setup (via web):

1. Go to **Setup → User Interface → Tabs**
2. Click **New** in Custom Object Tabs section
3. Create tabs for each object:
   - Token Purchaser
   - Crypto Wallet
   - Crypto Transaction
   - Grant Recipient
   - Organization Document
   - Grant Project
   - Project Milestone
   - Milestone Documentation
   - Milestone Payment

### 2. Create Lightning App

1. Go to **Setup → Apps → App Manager**
2. Click **New Lightning App**
3. Name: **JIL CRM**
4. Add all custom tabs
5. Assign to relevant profiles

### 3. Configure Sharing Rules

```bash
# Use Salesforce web UI:
# Setup → Security → Sharing Settings

# Set Organization-Wide Defaults:
# - Token_Purchaser__c: Private
# - Grant_Recipient__c: Private
# - All child objects: Controlled by Parent

# Create Sharing Rules for portal users (later step)
```

### 4. Assign Permission Sets

```bash
# Via CLI:
sf org assign permset --name JIL_CRM_Admin --target-org globalhands

# Or via web:
# Setup → Users → Permission Sets
# Assign "JIL_CRM_Admin" to admin users
```

---

## 🌐 Experience Cloud Portal Setup

### Prerequisites
- Experience Cloud must be enabled in your org
- Custom domain configured (e.g., globalhands.force.com)

### Steps

1. **Enable Experience Cloud**
   - Setup → Feature Settings → Digital Experiences → Settings
   - Enable Digital Experiences

2. **Create Experience**
   - Setup → Digital Experiences → All Sites
   - New → "Customer Account Portal" template
   - Name: "JIL Portal"
   - URL: `globalhands.force.com/portal`

3. **Configure Portal User Profiles**
   - Create profiles based on:
     - `JIL_Token_Purchaser_Portal` permission set
     - `JIL_Grant_Recipient_Portal` permission set

4. **Set Up Portal Pages**
   - Token Purchaser Dashboard
   - Grant Recipient Dashboard
   - Document Upload pages
   - Milestone tracking pages

---

## 🔐 Security Configuration (CRITICAL)

### Known Security Gaps to Address

From security audit (Feb 16, 2026), these must be fixed AFTER deployment:

#### 1. Field-Level Security (FLS)

```bash
# Add FLS restrictions via Permission Sets
# Priority fields needing FLS:
# - TokenSale__c.Amount__c
# - TokenSale__c.PaymentDetails__c
# - All financial fields
# - All PII fields (KYC data)
```

**Action Required:**
- Update permission sets with proper FLS
- Re-deploy: `sf project deploy start --source-dir permissionsets/`

#### 2. Platform Encryption

**Fields requiring encryption:**
- KYCVerification__c.DocumentID__c
- Any SSN/Passport fields
- Payment credential fields

**Action Required:**
- Setup → Platform Encryption
- Enable Shield Platform Encryption
- Encrypt sensitive fields

#### 3. Named Credentials (Remove Hardcoded Keys)

**Current Issue:** Hardcoded Onfido API key in code

**Action Required:**
- Setup → Named Credentials
- Create: `Onfido_API`
- Store API key securely
- Update integration code to use Named Credentials

#### 4. Field History Tracking

**Enable on:**
- Status fields
- Amount fields
- Approval fields

**Action Required:**
- Each object → Fields → Set History Tracking
- Enable on critical fields

---

## 🔗 Onfido Integration Setup

### Webhook Configuration

1. **Create Apex Webhook Handler** (to be developed)
   - Endpoint: `/services/apexrest/onfido/webhook`
   - Handles: KYC verification callbacks

2. **Register Webhook in Onfido Dashboard**
   - URL: `https://globalhands.my.salesforce.com/services/apexrest/onfido/webhook`
   - Events: `check.completed`, `check.form_completed`

3. **Field Mapping**
   - Onfido fields already defined in objects
   - Auto-populate on webhook receipt

---

## 📊 Testing Checklist

After deployment, test:

### Token Purchaser Module
- [ ] Create Token Purchaser record
- [ ] Add Crypto Wallet
- [ ] Create Transaction
- [ ] Verify relationships display correctly
- [ ] Test KYC field updates (manual for now)

### Grant Recipient Module
- [ ] Create Grant Recipient record
- [ ] Upload Organization Documents
- [ ] Create Grant Project
- [ ] Add Project Milestones
- [ ] Upload Milestone Documentation
- [ ] Create Milestone Payment
- [ ] Verify approval workflow

### Portal Access (After Experience Cloud setup)
- [ ] Create portal user (Token Purchaser)
- [ ] Create portal user (Grant Recipient)
- [ ] Test login and data visibility
- [ ] Test document upload
- [ ] Verify sharing rules work

---

## 🆘 Troubleshooting

### Issue: "This component has a problem: Invalid setup context"

**Solution:**
```bash
sf org open --target-org globalhands
# Go to Setup → Lightning App Builder
# Refresh all pages
```

### Issue: "Object not found" when deploying

**Solution:**
```bash
# Check package.xml includes CustomObject
cat package.xml | grep CustomObject

# Re-deploy specific object
sf project deploy start --source-dir objects/Token_Purchaser__c.object --target-org globalhands
```

### Issue: Permission Set assignment fails

**Solution:**
```bash
# Check if permission set exists
sf data query --query "SELECT Id, Name FROM PermissionSet WHERE Name LIKE 'JIL%'" --target-org globalhands

# If missing, deploy permission sets again
sf project deploy start --source-dir permissionsets/ --target-org globalhands
```

### Issue: Portal users can't see records

**Solution:**
- Check Organization-Wide Defaults (OWD)
- Verify Sharing Rules are created
- Check Profile/Permission Set assignments
- Ensure Role Hierarchy is configured

---

## 📞 Key Contacts

| Role | Name | Contact | Purpose |
|------|------|---------|---------|
| **Admin/SSH Access** | Jeff Mondonca | TBD | Coordinate server access |
| **Salesforce Org Owner** | TBD | info@jilsovereign.com | Primary org admin |
| **Developer** | Stanley | syberne1@gmail.com | Package maintainer |

---

## 🔄 Git Workflow (For Future Updates)

### Making Changes

```bash
cd /Users/Stanley_1/Repos/jil-crm-package

# Pull latest
git pull

# Make your changes to object files, permission sets, etc.

# Check what changed
git status

# Add all changes
git add .

# Commit with descriptive message
git commit -m "Added validation rule for Grant amount minimum"

# Push to GitHub
git push
```

### Deploying Changes

```bash
# Deploy everything
sf project deploy start --source-dir . --target-org globalhands

# OR deploy specific metadata
sf project deploy start --source-dir objects/Grant_Project__c.object --target-org globalhands
```

---

## 📚 Reference Documentation

### Salesforce CLI Commands

```bash
# List all orgs
sf org list

# Open Salesforce in browser
sf org open --target-org globalhands

# Retrieve metadata from org (if needed)
sf project retrieve start --source-dir . --target-org globalhands

# Check deployment status
sf project deploy report --target-org globalhands

# Run Apex tests
sf apex test run --target-org globalhands --wait 10
```

### Useful Salesforce Setup Paths

- **Objects:** Setup → Object Manager
- **Permission Sets:** Setup → Users → Permission Sets
- **Sharing:** Setup → Security → Sharing Settings
- **Tabs:** Setup → User Interface → Tabs
- **Experience Cloud:** Setup → Digital Experiences
- **Platform Encryption:** Setup → Platform Encryption

---

## 🎯 Success Criteria

Deployment is successful when:

1. ✅ All 9 custom objects visible in Object Manager
2. ✅ All 3 permission sets assignable to users
3. ✅ Custom tabs created and functional
4. ✅ Test records can be created in all objects
5. ✅ Relationships between objects work correctly
6. ✅ Portal users can log in (after portal setup)
7. ✅ Security rules properly restrict data access

---

## 📝 Notes

- **Average Crypto Wallet Balance:** ~$3,560 (2025 data) - useful for analytics
- **User Tier System:** Bronze → Silver → Gold → Platinum → Institutional
- **Milestone-Based Funding:** Projects broken into numbered milestones with documentation requirements
- **KYC Expiration:** Plan for re-verification workflow (typically annual)

---

## 🔜 Future Enhancements (Post-MVP)

- [ ] Apex triggers for rollup summaries
- [ ] Flows for automatic notifications
- [ ] Validation rules for business logic
- [ ] Custom reports and dashboards
- [ ] Onfido integration automation
- [ ] Blockchain transaction verification
- [ ] Payment gateway integration (Stripe/crypto)
- [ ] Mobile app for portal users

---

**Last Updated By:** Claude (AI Assistant)  
**Date:** February 18, 2026  
**Status:** Ready for deployment after Salesforce account change
