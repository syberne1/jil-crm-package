# JIL CRM - Quick Command Reference

**Quick commands for when you restart work on the JIL CRM package**

---

## 🚀 One-Command Resume

```bash
# Pull latest code and authenticate to Salesforce
cd /Users/Stanley_1/Repos/jil-crm-package && \
git pull && \
sf org login web --alias globalhands --instance-url https://login.salesforce.com
```

---

## 📦 Deploy Package

```bash
# From package directory
cd /Users/Stanley_1/Repos/jil-crm-package
sf project deploy start --source-dir . --target-org globalhands
```

---

## ✅ Verify Deployment

```bash
# Check if your 9 custom objects exist
sf sobject list --target-org globalhands | grep "__c"

# Open Salesforce in browser
sf org open --target-org globalhands
```

---

## 🔄 Daily Git Workflow

```bash
# Check status
git status

# Add all changes
git add .

# Commit
git commit -m "Your message here"

# Push to GitHub
git push
```

---

## 🔧 Common Salesforce CLI Commands

```bash
# List authenticated orgs
sf org list

# Display current org info
sf org display --target-org globalhands

# Open Salesforce Setup
sf org open --target-org globalhands --path /lightning/setup/SetupOneHome/home

# Query records
sf data query --query "SELECT Id, Name FROM Token_Purchaser__c" --target-org globalhands

# Assign permission set to current user
sf org assign permset --name JIL_CRM_Admin --target-org globalhands
```

---

## 📞 Quick Contacts

- **Admin:** Jeff Mondonca (SSH/server access)
- **Salesforce:** info@jilsovereign.com (Global Hands nonprofit account)
- **GitHub:** https://github.com/syberne1/jil-crm-package

---

## 📚 Full Documentation

See `DEPLOYMENT_GUIDE.md` for complete instructions.
