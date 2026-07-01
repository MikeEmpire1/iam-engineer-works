# Scenario 4: Federation / SSO — Implementation

## Prerequisites

- AWS Management account with IAM Identity Centre enabled
- Microsoft Entra ID P2 licence (for Access Reviews + Conditional Access)
- Global admin access in Entra ID
- AWS CLI configured with Management account access

---

## 1. Configure Entra ID as an External IdP in IAM Identity Centre

### Step 1: Get IAM Identity Centre Metadata

```powershell
# Download IAM Identity Centre SAML metadata
$instanceArn = "arn:aws:sso:::instance/ssoins-1234567890abcdef"
aws sso-admin describe-instance --instance-arn $instanceArn --query "IdentityStoreId"

# Get the AWS access portal URL
aws sso-admin list-instances --query "Instances[0].LoginUrl"
# Returns: https://innogrid.awsapps.com/start
```

### Step 2: Configure Entra ID Gallery App

1. **Azure Portal** → Entra ID → Enterprise applications → **New application** → **Create your own**
2. Search for **AWS IAM Identity Centre (successor to AWS SSO)** in the gallery
3. Name: `InnoGrid AWS SSO`
4. **Single sign-on** → **SAML**
5. Download the **SAML metadata** XML from Entra ID
6. In the AWS access portal URL under **Reply URL**: `https://innogrid.awsapps.com/login/callback`

### Step 3: Configure IAM Identity Centre to Trust Entra ID

```powershell
# Upload Entra ID SAML metadata to IAM Identity Centre
# Via AWS Console: IAM Identity Centre → Settings → Identity source → Change identity source
# Select "External identity provider"
# Upload the SAML metadata XML from Entra ID
```

Console steps:
1. **AWS Console** → IAM Identity Centre → Settings → **Identity source**
2. Click **Change identity source**
3. Select **External identity provider**
4. Upload the Entra ID SAML metadata XML
5. Download the IAM Identity Centre **service provider metadata** XML
6. Upload this to Entra ID SSO configuration

### Step 4: Complete Entra ID SSO Configuration

1. **Azure Portal** → Enterprise applications → `InnoGrid AWS SSO` → **Single sign-on**
2. Upload the IAM Identity Centre SP metadata
3. Map attributes:

| Entra ID Attribute | SAML Attribute |
|---|---|
| `user.userprincipalname` | `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress` |
| `user.displayname` | `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name` |
| `user.givenname` | `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname` |
| `user.surname` | `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname` |

4. **Users and groups** → Assign the corporate groups (`HR-Team`, `Finance-Team`, etc.)

---

## 2. Configure SCIM Provisioning

### Step 1: Generate SCIM Endpoint in IAM Identity Centre

```powershell
# Get the SCIM endpoint URL
aws identitystore describe-identity-store --identity-store-id d-9a7b8c6d5e
# Returns: IdentityStoreId, SCIM endpoint URL

# Generate the SCIM access token (via Console)
# IAM Identity Centre → Settings → Automatic provisioning → Enable
# Copy the SCIM endpoint URL and access token
```

Console Steps:
1. **AWS Console** → IAM Identity Centre → Settings → **Automatic provisioning**
2. Click **Enable** (this enables the SCIM v2.0 endpoint)
3. Copy the **SCIM endpoint URL** (e.g., `https://scim.eu-west-2.amazonaws.com/d-9a7b8c6d5e/scim/v2/`)
4. Copy the **Access token**

### Step 2: Configure Entra ID SCIM Provisioning

1. **Azure Portal** → Enterprise applications → `InnoGrid AWS SSO` → **Provisioning**
2. Set **Provisioning mode** → **Automatic**
3. **Tenant URL** = SCIM endpoint URL from step 1
4. **Secret token** = Access token from step 1
5. Click **Test connection** → verify success
6. **Save**
7. Under **Mappings**, verify attribute mapping:
   - `userPrincipalName` ↔ `userName`
   - `displayName` ↔ `displayName`
   - `givenName` ↔ `name.givenName`
   - `surname` ↔ `name.familyName`
   - `mail` ↔ `emails[type eq "work"].value`
   - `employeeId` ↔ `externalId`
8. Under **Settings**, set:
   - Scope: **Sync only assigned users and groups**
   - Provisioning status: **On**

### Step 3: Test SCIM Sync

```powershell
# Trigger an initial provisioning cycle
# Azure Portal → Provisioning → "Provision on demand"
# Select a test user (e.g., diana.cruz@innogrid.com)
```

Verification:

```powershell
# Verify Diana Cruz now exists in IAM Identity Centre
aws identitystore list-users --identity-store-id d-9a7b8c6d5e `
  --filter AttributePath=UserName,AttributeValue=diana.cruz@innogrid.com

# Expected: User exists with attributes synced from Entra ID
```

---

## 3. Configure Conditional Access Policies

### Step 1: Require MFA for AWS

```powershell
# Via Azure Portal (these must be configured in the Entra ID admin centre)
```

1. **Azure Portal** → Entra ID → Security → Conditional Access → **New policy**
2. Name: `InnoGrid AWS - Require MFA`
3. **Users**: `InnoGrid AWS SSO` app assignment
4. **Cloud apps**: `AWS IAM Identity Centre`
5. **Conditions**: —
6. **Grant**: `Require multifactor authentication`
7. **Enable policy**: On

### Step 2: Block Legacy Authentication

1. **New policy** → Name: `Block Legacy Auth for AWS`
2. **Users**: All
3. **Cloud apps**: `AWS IAM Identity Centre`
4. **Conditions**: Client apps → `Exchange ActiveSync`, `Other clients`
5. **Grant**: Block access
6. **Enable policy**: On

### Step 3: Device Compliance for Production Access

1. **New policy** → Name: `Prod Access - Require Compliant Device`
2. **Users**: Select `Finance-Team`, `Legal-Team`, `Operations-Team` groups
3. **Cloud apps**: `AWS IAM Identity Centre`
4. **Conditions**: —
5. **Grant**: `Require device to be marked as compliant` + `Require multifactor authentication`
6. **Enable policy**: On

### Step 4: Session Timeout

Configured in IAM Identity Centre:

```powershell
# Set the default session duration for federated users
# Via Console: IAM Identity Centre → Settings → Session settings
# Set: "Access portal session duration" = 8 hours
```

---

## 4. Test Federation Flows

### Test 1: Corporate User SSO

```powershell
# Simulate Diana Cruz (Finance) login flow:
# 1. Browse to https://myapps.microsoft.com
# 2. Sign in as diana.cruz@innogrid.com
# 3. Complete MFA prompt
# 4. Click the "AWS" app tile
# 5. Verify redirect to AWS access portal
# 6. Verify visible account: inno-prod with Prod-ReadOnly permission set
```

### Test 2: SCIM Deprovisioning

```powershell
# 1. In Entra ID, disable a test corporate user
# 2. Wait for SCIM sync (runs every ~40 minutes by default, or trigger on-demand)
# 3. Verify user is disabled in IAM Identity Centre:
aws identitystore describe-user --identity-store-id d-9a7b8c6d5e --user-id "<user-id>"
# Expected: status = DISABLED (or user not found if deleted)
```

### Test 3: Engineering User Direct Auth

```powershell
# Verify Alex Rivera (Engineering, IAM Identity Centre-native) can still log in
# directly via the AWS access portal without Entra ID:
# Browse to https://innogrid.awsapps.com/start
# Sign in with Alex's IAM Identity Centre credentials
# Expected: successful login with Nonprod-Dev + Prod-ReadOnly access
```

---

## 5. SSO Portal Access

### Entra ID My Apps Portal Configuration

1. **Azure Portal** → Enterprise applications → `InnoGrid AWS SSO` → **Properties**
   - Visible to users: **Yes**
2. **Users and groups** → Add corporate users/groups
3. Users browse to `https://myapps.microsoft.com`
4. They see the `InnoGrid AWS SSO` tile
5. Click → SAML redirect → IAM Identity Centre → AWS account selection

### Direct AWS Access Portal URL

Users can also bookmark the AWS access portal directly:
```
https://innogrid.awsapps.com/start
```

---

## 6. Monitoring & Troubleshooting

### SCIM Provisioning Logs

```powershell
# View SCIM sync logs in Entra ID
# Azure Portal → Enterprise applications → InnoGrid AWS SSO → Provisioning → View provisioning logs
```

### CloudTrail Federation Events

```sql
-- Athena: Query federation events
SELECT
    eventtime,
    eventname,
    useridentity.arn,
    JSON_EXTRACT_SCALAR(responseelements, '$.IdentityStoreId') AS store_id
FROM cloudtrail_logs
WHERE
    eventname IN ('Federate', 'CreateUser', 'UpdateUser', 'DeleteUser')
    AND eventtime >= '2026-07-01T00:00:00Z'
ORDER BY eventtime;
```

### Conditional Access Sign-in Logs

```powershell
# Azure Portal → Entra ID → Sign-in logs
# Filter by: Application = "AWS IAM Identity Centre"
# Check: "Conditional Access" tab for each sign-in to see which policies were applied
```

---

## 7. Fallback Procedure

If Entra ID is unavailable:

1. Engineering users continue authenticating directly via IAM Identity Centre (unaffected)
2. Corporate users cannot authenticate until Entra ID is restored
3. If extended outage (>4 hours) and critical access is needed:
   - IAM team creates temporary IAM Identity Centre-native users for essential corporate personnel
   - Users are disabled/removed when Entra ID returns
   - CISO approves all temporary accounts

---

## Summary

| Component | Status | Method |
|---|---|---|
| Entra ID ↔ IAM Identity Centre SAML federation | Configured | SAML 2.0 metadata exchange |
| SCIM provisioning | Enabled | Automatic sync every ~40 minutes |
| Conditional Access - MFA | Enforced | All AWS access requires MFA |
| Conditional Access - Device compliance | Enforced | Production access requires compliant device |
| SCIM attribute mapping | Verified | department, employeeId, name, email |
| Engineering fallback auth | Unchanged | Direct IAM Identity Centre login |
