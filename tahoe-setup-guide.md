# macOS 26 Tahoe: Simplified Setup for Platform SSO
## Implementation Guide Supplement

**Document Version:** 1.0  
**Date:** 30 October 2025  
**Author:** David Crosby (Bing)
**LinkedIn:** https://www.linkedin.com/in/david-bing-crosby/  

---

## Executive Summary

macOS 26 Tahoe, released on 15 September 2025, introduces **Simplified Setup for Platform Single Sign-On**, the most significant evolution in Mac identity management since Platform SSO was first introduced. This feature fundamentally transforms the onboarding experience by integrating identity authentication directly into the Setup Assistant, creating a true zero-touch deployment where users authenticate with their organisational identity provider before a local account even exists.

This supplement provides detailed implementation guidance for organisations adopting Simplified Setup with Jamf Pro, Microsoft Entra ID, or Okta.

---

## What's New in macOS 26 Tahoe

### Major Platform SSO Enhancements

**1. Simplified Setup for Platform SSO**
- Identity authentication occurs during Setup Assistant
- First local account created automatically from IdP credentials
- Platform SSO registration completed before desktop access
- Eliminates post-setup configuration entirely

**2. Authenticated Guest Mode**
- Temporary IdP-authenticated sessions for shared devices
- NFC badge support for Tap to Login
- Automatic session cleanup on logout
- Ideal for healthcare, education, and retail environments

**3. Enhanced Integration**
- Seamless Managed Apple ID provisioning
- Profile picture synchronisation from IdP
- Improved offline grace period handling
- Better error messaging during setup

### What This Means for Organisations

**Before macOS 26:**
```
Device Setup → Create Local Account → Login → Register Platform SSO → Wait for sync
Total: 15-25 minutes, 8-10 user interactions
```

**With macOS 26 Simplified Setup:**
```
Device Setup → Authenticate with IdP → Desktop (everything configured automatically)
Total: 8-12 minutes, 2-3 user interactions
```

---

## Prerequisites

### Minimum Version Requirements

| Component | Minimum Version | Recommended Version | Notes |
|-----------|----------------|---------------------|-------|
| **macOS** | 26.0 Tahoe | 26.1+ | Must be macOS 26 or later |
| **Jamf Pro** | 11.20.0 | 11.22.0+ | Simplified Setup support added in 11.20 |
| **Company Portal** | 5.2504.0 | Latest | Microsoft Entra ID support |
| **Okta Verify** | 9.52 | Latest | Okta Device Access support |
| **Jamf Connect** | 2.35.0 | Latest | Optional, for complementary features |
| **Jamf Protect** | 5.0.0 | Latest | macOS 26 compatibility |

### Identity Provider Support

**Microsoft Entra ID**
- Platform SSO: Generally Available (GA) since August 2025
- Simplified Setup: Supported with Company Portal 5.2504.0+
- Secure Enclave authentication: Fully supported
- Status: Production ready

**Okta Device Access**
- Platform SSO: Generally Available
- Simplified Setup: Supported with Okta Verify 9.52+
- Secure Enclave authentication: Fully supported
- Status: Production ready

**Google Workspace**
- Platform SSO: Limited availability
- Simplified Setup: Check with Google for current status
- Status: Contact Google for enterprise support

### Network Requirements

**Critical Endpoints (Must Be Accessible)**

Microsoft Entra ID:
```
https://login.microsoftonline.com
https://login.microsoft.com
https://enterpriseregistration.windows.net
https://device.login.microsoftonline.com
https://graph.microsoft.com
```

Okta:
```
https://[your-domain].okta.com
https://[your-domain].oktapreview.com
```

Jamf:
```
https://[your-instance].jamfcloud.com
https://[your-distribution-point]
```

Apple:
```
https://albert.apple.com (ABM)
https://mdmenrollment.apple.com (ADE)
https://deviceenrollment.apple.com (ADE)
```

### Device Compatibility

**Supported Devices for macOS 26 Tahoe:**

✅ **Apple Silicon (Recommended)**
- All M1, M2, M3, M4 series Macs
- Full Simplified Setup support
- Secure Enclave authentication
- Optimal performance

✅ **Intel Macs with T2 Chip (Final Support)**
- Mac Pro (2019)
- iMac (2020)
- MacBook Pro 16-inch (2019)
- MacBook Pro 13-inch (2020, four Thunderbolt 3 ports)

⚠️ **Important Notes:**
- macOS 26 Tahoe is the **final macOS release** to support Intel-based Macs
- MacBook Air and Mac mini Intel models are NOT supported in Tahoe
- Plan hardware refresh for Intel devices before macOS 27

---

## Simplified Setup Architecture

### How It Works

#### Traditional Platform SSO Flow (macOS 13-15)

```
┌─────────────────────────────────────────────────────┐
│ 1. Device enrolls in MDM                            │
├─────────────────────────────────────────────────────┤
│ 2. Setup Assistant creates local account OR         │
│    Jamf Connect creates cloud-synced account        │
├─────────────────────────────────────────────────────┤
│ 3. User logs in to newly created account            │
├─────────────────────────────────────────────────────┤
│ 4. Platform SSO profile deploys to device           │
├─────────────────────────────────────────────────────┤
│ 5. User receives notification to register PSSO      │
├─────────────────────────────────────────────────────┤
│ 6. User manually opens System Settings              │
├─────────────────────────────────────────────────────┤
│ 7. User clicks through Platform SSO registration    │
├─────────────────────────────────────────────────────┤
│ 8. MFA authentication to complete registration      │
├─────────────────────────────────────────────────────┤
│ 9. Platform SSO active after several minutes        │
└─────────────────────────────────────────────────────┘

Issues:
• Multiple steps prone to user error
• Temporary unsynced local account
• Window of non-compliance
• Higher support burden
• 15-25 minute setup time
```

#### Simplified Setup Flow (macOS 26+)

```
┌─────────────────────────────────────────────────────┐
│ 1. Device enrolls in MDM via ADE                    │
├─────────────────────────────────────────────────────┤
│ 2. Setup Assistant pauses automatically             │
├─────────────────────────────────────────────────────┤
│ 3. Jamf Pro deploys IdP app (Company Portal/Okta)   │
├─────────────────────────────────────────────────────┤
│ 4. Setup Assistant detects IdP app ready            │
├─────────────────────────────────────────────────────┤
│ 5. Native IdP authentication window appears         │
├─────────────────────────────────────────────────────┤
│ 6. User authenticates with MFA                      │
├─────────────────────────────────────────────────────┤
│ 7. macOS automatically:                              │
│    • Creates local account from IdP data            │
│    • Registers Platform SSO                         │
│    • Generates Secure Enclave keys                  │
│    • Syncs password                                 │
│    • Downloads profile picture                      │
│    • Applies compliance policies                    │
├─────────────────────────────────────────────────────┤
│ 8. User reaches desktop fully configured            │
└─────────────────────────────────────────────────────┘

Benefits:
• Single authentication step
• No temporary accounts
• Immediate compliance
• Significantly reduced support
• 8-12 minute setup time
```

### Technical Implementation Details

**MDM Response Modification**

When Setup Assistant contacts the MDM during Automated Device Enrolment, Jamf Pro must return a specific HTTP 403 response with Platform SSO configuration data:

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "platform_sso_required": true,
  "platform_sso_app_bundle_id": "com.microsoft.CompanyPortalMac",
  "platform_sso_config_profile_url": "https://[jamf-instance]/profiles/psso_config.mobileconfig",
  "associated_domains": ["login.microsoftonline.com"],
  "wait_for_installation": true
}
```

**Setup Assistant Pause Mechanism**

1. Setup Assistant receives 403 response with Platform SSO requirements
2. Displays "Setting up your Mac" status screen
3. Monitors for Platform SSO app installation
4. Checks for required configuration profiles
5. Once both are present, proceeds to authentication
6. Maximum wait time: 30 minutes (configurable)

**Authentication Token Flow**

```
User enters credentials
    ↓
Company Portal/Okta Verify validates
    ↓
IdP returns OAuth/OIDC token
    ↓
Token includes user claims:
  • givenName
  • surname
  • userPrincipalName
  • mail
  • objectId
  • photo (URL)
    ↓
macOS consumes token and creates account
    ↓
Platform SSO framework stores tokens in Secure Enclave
    ↓
Registration complete
```

---

## Step-by-Step Implementation

### Phase 1: Jamf Pro Configuration

#### Step 1.1: Update Jamf Pro

Ensure Jamf Pro is version 11.20.0 or later:

```
Jamf Pro → Settings → System Settings → About

Current Version: 11.20.0 or higher required
```

If update needed:
- Cloud-hosted: Automatic update or contact Jamf Support
- On-premises: Download from Jamf Nation and apply update

#### Step 1.2: Create Associated Domains Configuration Profile

**Navigation:** `Computers → Configuration Profiles → New`

**General Tab:**
```
Display Name: macOS 26 - Associated Domains for Simplified Setup
Description: Required for Simplified Setup Platform SSO in macOS 26 Tahoe
Category: Identity & Access
Level: Computer Level
Distribution Method: Install Automatically
```

**Associated Domains Payload:**

Click `Configure` → Select `Associated Domains`

**For Microsoft Entra ID:**
```
Add Associated Domains:

webcredentials:login.microsoftonline.com
webcredentials:login.microsoft.com
webcredentials:sts.windows.net
webcredentials:login.partner.microsoftonline.cn
webcredentials:login.chinacloudapi.cn
webcredentials:login.microsoftonline.us
webcredentials:login-us.microsoftonline.com
webcredentials:enterpriseregistration.windows.net
```

**For Okta:**
```
Add Associated Domains:

webcredentials:[your-okta-domain].okta.com
webcredentials:[your-okta-domain].oktapreview.com
```

**Scope Tab:**
```
Targets:
  Computer Groups: All Computers
  OR
  Computer Groups: macOS 26 Tahoe Devices (Smart Group)

Exclusions: None
```

**Click Save**

#### Step 1.3: Modify PreStage Enrolment

**Navigation:** `Computers → PreStage Enrolments`

**Option A: Create New PreStage for macOS 26**
```
1. Click "+ New"
2. Name: "macOS 26 Tahoe - Simplified Setup PSSO"
3. Proceed through General settings
```

**Option B: Edit Existing PreStage**
```
1. Select your production PreStage
2. Click "Edit"
3. Proceed to modifications below
```

**General Tab Configuration:**
```
Display Name: Production PreStage - macOS 26 Simplified Setup
Enrolment Site: [Your Site]

☑ Require Authentication
Authentication Type: Identity Provider

Support Information:
  Phone: [Your Support Number]
  Email: bing@bing-bong.co.uk
  Department: IT Services
```

**Enrolment Tab:**
```
☑ Keep existing MDM Profile
☑ Await final configuration
☑ Enable location services
☑ Install profile to allow network access
```

**Account Settings:**
```
☐ Create Administrator Account with JamfConnect
☑ Enable automatic account creation with identity provider
☑ Prevent users from delaying setup of account
☑ Account creation requires network connection

⚠️ IMPORTANT: Do NOT create a separate admin account
   The IdP will create the first account automatically
```

**Setup Assistant Options - Critical Section:**

Scroll down to find the Simplified Setup section:

```
┌───────────────────────────────────────────────────┐
│ Simplified Setup for Platform Single Sign-On     │
├───────────────────────────────────────────────────┤
│ ☑ Enable Simplified Setup for Platform SSO       │
│                                                   │
│ Platform Single Sign-On App Bundle ID:           │
│ ┌───────────────────────────────────────────┐    │
│ │ com.microsoft.CompanyPortalMac            │    │
│ └───────────────────────────────────────────┘    │
│                                                   │
│ For Okta, use: com.okta.macOS.Verify             │
└───────────────────────────────────────────────────┘
```

**Enter the correct Bundle ID:**
- Microsoft Entra ID: `com.microsoft.CompanyPortalMac`
- Okta: `com.okta.macOS.Verify`

**Setup Assistant Screens to Skip:**
```
☑ Location Services
☑ Apple ID Setup
☑ Terms and Conditions
☑ Biometrics (Touch ID/Face ID) - optional
☑ Apple Pay
☑ Siri
☑ Screen Time
☑ Analytics
☑ Appearance (Light/Dark mode)
☑ True Tone

☐ iCloud Account (leave enabled for Managed Apple ID setup)
☐ FileVault (will enable via policy)
```

**Scope Tab:**
```
Assigned Devices:
  Serial Numbers: [Manually added or synced from ABM]
  OR
  Device Enrollment: All Devices
```

**Click Save**

#### Step 1.4: Create IdP Application Deployment Policy

The IdP application must be available during Setup Assistant.

**Navigation:** `Computers → Policies → New`

**General Tab:**
```
Display Name: Deploy Company Portal for Simplified Setup
Enabled: ✅ Yes
Trigger:
  ☑ Enrollment Complete
  ☑ Recurring Check-in (as backup)
Execution Frequency: Once per computer
```

**Packages Tab:**
```
Click "Configure"

Package: Company Portal-5.2504.0.pkg
OR
Package: Okta Verify-9.52.pkg

Action: Install
Priority: 1 (Very high - must install first)
```

**To obtain the package:**

**Microsoft Company Portal:**
```bash
# Download from Jamf Admin or manually:
curl -L -o CompanyPortal.pkg \
  "https://go.microsoft.com/fwlink/?linkid=853070"

# Upload to Jamf Pro via Jamf Admin or web interface
```

**Okta Verify:**
```
Download from Okta Admin Console:
Settings → Downloads → Okta Verify for macOS

Upload to Jamf Pro
```

**Scope Tab:**
```
Targets:
  Smart Groups: New Enrollments - macOS 26 Tahoe
  
  OR
  
  All Computers (if everyone is on macOS 26)

Exclusions: None
```

**Self Service Tab:**
```
☐ Make available in Self Service
(Must install automatically, not by user choice)
```

**Click Save**

### Phase 2: Platform SSO Configuration Profile

#### Step 2.1: Create Platform SSO Profile for Simplified Setup

**Navigation:** `Computers → Configuration Profiles → New`

**General Tab:**
```
Display Name: Platform SSO - Microsoft Entra ID - Simplified Setup (macOS 26)
Description: Platform SSO with Simplified Setup integration for macOS 26 Tahoe
Category: Identity & Access
Level: Computer Level
Distribution Method: Install Automatically
```

**Single Sign-On Extensions Payload:**

Click `Configure` → Add `Single Sign-On Extensions`

**Payload Configuration:**

```
Extension Identifier: com.microsoft.CompanyPortalMac.ssoextension
Team Identifier: UBF8T346G9
Sign-On Type: Redirect

☑ Use Platform SSO (Critical - must be enabled)

URLs:
  Add each of the following:
  • https://login.microsoftonline.com
  • https://login.microsoft.com
  • https://sts.windows.net
  • https://login.partner.microsoftonline.cn
  • https://login.chinacloudapi.cn
  • https://login.microsoftonline.us
  • https://login-us.microsoftonline.com
  • https://enterpriseregistration.windows.net
```

**Extension Data:**

Click `Add Extension Configuration` and add the following key-value pairs:

| Key | Type | Value |
|-----|------|-------|
| `AppPrefixAllowList` | String | `com.microsoft.,com.apple.` |
| `browser_sso_interaction_enabled` | Integer | `1` |
| `disable_explicit_app_prompt` | Integer | `1` |
| `device_registration` | String | `{{DEVICEUDID}}` |
| `authentication_method` | String | `SecureEnclave` |
| `platform_sso_enabled` | Integer | `1` |
| `AccountDisplayName` | String | `$upn$` |
| `LoginFrequency` | Integer | `0` |
| `NewDeviceLoginFrequency` | Integer | `14` |
| `EnableSimplifiedSetup` | Boolean | `true` |

**Additional Keys for Simplified Setup:**

Add these additional extension data keys:

```xml
<key>SetupAssistantIntegration</key>
<dict>
    <key>Enabled</key>
    <true/>
    
    <key>AutoCreateAccount</key>
    <true/>
    
    <key>AccountCustomization</key>
    <dict>
        <key>FullName</key>
        <string>{{givenName}} {{surname}}</string>
        
        <key>UserName</key>
        <string>{{userPrincipalName_prefix}}</string>
        
        <key>HomeDirectory</key>
        <string>/Users/{{userPrincipalName_prefix}}</string>
        
        <key>ProfilePicture</key>
        <true/>
        
        <key>SetAdminStatus</key>
        <false/>
    </dict>
    
    <key>RegistrationRequired</key>
    <true/>
    
    <key>SkipOnExistingAccount</key>
    <false/>
</dict>

<key>Authorization</key>
<dict>
    <key>Enabled</key>
    <true/>
    
    <key>RightForUserAuthentication</key>
    <string>authenticate-session-owner</string>
</dict>
```

**⚠️ Important Notes on Extension Data:**

In Jamf Pro's interface, you may need to upload a custom .plist file for complex nested dictionaries. Create a file named `psso-extension-data.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>AppPrefixAllowList</key>
    <string>com.microsoft.,com.apple.</string>
    
    <key>browser_sso_interaction_enabled</key>
    <integer>1</integer>
    
    <key>disable_explicit_app_prompt</key>
    <integer>1</integer>
    
    <key>device_registration</key>
    <string>{{DEVICEUDID}}</string>
    
    <key>authentication_method</key>
    <string>SecureEnclave</string>
    
    <key>platform_sso_enabled</key>
    <integer>1</integer>
    
    <key>AccountDisplayName</key>
    <string>$upn$</string>
    
    <key>LoginFrequency</key>
    <integer>0</integer>
    
    <key>NewDeviceLoginFrequency</key>
    <integer>14</integer>
    
    <key>EnableSimplifiedSetup</key>
    <true/>
    
    <key>SetupAssistantIntegration</key>
    <dict>
        <key>Enabled</key>
        <true/>
        
        <key>AutoCreateAccount</key>
        <true/>
        
        <key>AccountCustomization</key>
        <dict>
            <key>FullName</key>
            <string>{{givenName}} {{surname}}</string>
            
            <key>UserName</key>
            <string>{{userPrincipalName_prefix}}</string>
            
            <key>ProfilePicture</key>
            <true/>
            
            <key>SetAdminStatus</key>
            <false/>
        </dict>
        
        <key>RegistrationRequired</key>
        <true/>
    </dict>
    
    <key>Authorization</key>
    <dict>
        <key>Enabled</key>
        <true/>
        
        <key>RightForUserAuthentication</key>
        <string>authenticate-session-owner</string>
    </dict>
</dict>
</plist>
```

Upload this file via Jamf Pro's "Upload File" option in Extension Data.

**Scope Tab:**
```
Targets:
  Computer Groups: macOS 26 Tahoe Devices
  
Exclusions:
  Computer Groups: Platform SSO Already Configured (optional)
```

**Click Save**

### Phase 3: Supporting Configuration

#### Step 3.1: SCEP Certificate for Device Identity

Platform SSO requires a device certificate. Create a SCEP profile:

**Navigation:** `Computers → Configuration Profiles → New`

**General:**
```
Display Name: SCEP - Platform SSO Device Certificate
Level: Computer Level
Distribution Method: Install Automatically
```

**Certificate Payload:**
```
Certificate Authority: Jamf Pro Built-in CA

Subject: CN=$UDID,OU=Devices,O=[Your Organisation]

Subject Alternative Name:
  DNS: $SERIALNUMBER.[your-domain].com
  UPN: $UDID@[your-domain].com

Key Size: 2048

Key Usage:
  ☑ Digital Signature
  ☑ Key Encipherment

Extended Key Usage:
  ☑ Client Authentication (1.3.6.1.5.5.7.3.2)
  ☑ Smart Card Logon (1.3.6.1.4.1.311.20.2.2)

Key Is Extractable: ☐ No (Important!)
Allow export from keychain: ☐ No

Retries: 3
Retry Delay: 10 seconds
```

**Scope:**
```
Targets: macOS 26 Tahoe Devices
```

#### Step 3.2: FileVault Configuration

Enable FileVault with institutional recovery key:

**Navigation:** `Computers → Configuration Profiles → New`

**General:**
```
Display Name: FileVault 2 - Institutional Recovery Key
Level: Computer Level
```

**FileVault Payload:**
```
☑ Enable FileVault

FileVault Recovery Key Type:
  ● Institutional

Escrow Location: Jamf Pro

☑ Show recovery key to user: No
☑ Personal recovery key rotation: Yes
☑ Defer enabling until logout
☑ Require FileVault on logout

Bypass Attempts: 9999 (effectively never expire)
```

**Scope:**
```
Targets: All Computers
```

---

## Testing and Validation

### Test Device Preparation

**1. Obtain Test Device**
- macOS 26 Tahoe compatible Mac
- Erased or factory fresh
- Assigned to correct PreStage in ABM

**2. Pre-Flight Checklist**

✅ Jamf Pro updated to 11.20+  
✅ PreStage configured with Simplified Setup enabled  
✅ Associated Domains profile deployed  
✅ Platform SSO profile created and scoped  
✅ IdP app package uploaded and policy created  
✅ Test user account exists in Entra ID/Okta  
✅ Test user has MFA configured  
✅ Network connectivity verified  

**3. Enrolment Test Procedure**

```
Time: 0:00 - Power on device
Action: Select language (English - United Kingdom)
Expected: Proceeds to region selection

Time: 0:01 - Select region
Action: Select United Kingdom
Expected: Proceeds to Wi-Fi

Time: 0:02 - Connect to Wi-Fi
Action: Select and connect to network
Expected: Device contacts ABM, enrols in Jamf

Time: 0:03 - Setup Assistant pauses
Expected: "Setting up your Mac" screen appears
Monitor: Jamf Pro logs for enrolment and policy triggers

Time: 0:03-0:07 - IdP app installs
Action: None (automatic)
Expected: Company Portal or Okta Verify installs silently
Monitor: Check policy logs

Time: 0:07 - Authentication window appears
Expected: Native sign-in window with company branding
Action: Enter test user email: testuser@yourdomain.com

Time: 0:08 - Redirect to IdP
Expected: Microsoft/Okta authentication page loads
Action: Enter password

Time: 0:09 - MFA challenge
Expected: Authenticator app prompt / SMS / etc.
Action: Complete MFA

Time: 0:10 - Account creation
Expected: "Creating your account..." message
Automatic: Local account created with username from UPN
Automatic: Platform SSO registered
Automatic: Secure Enclave keys generated

Time: 0:11 - Final setup
Expected: Optional iCloud/Managed Apple ID screen
Action: Skip or sign in as preferred

Time: 0:12 - Desktop
Expected: User reaches desktop
Verification steps below
```

**4. Post-Enrolment Verification**

On the test device, run these commands:

```bash
# 1. Verify Platform SSO is registered
/usr/bin/app-sso platform -s

# Expected output should include:
# State: Registered
# Registration State: Registered
# Authentication Method: SecureEnclave

# 2. Check local account was created correctly
dscl . -read /Users/$(whoami)

# Should show:
# Full Name matching IdP
# Username from UPN
# Home directory correct

# 3. Verify Secure Enclave key
/usr/bin/app-sso platform -L

# Should list Platform SSO configuration with Secure Enclave

# 4. Check Company Portal / Okta Verify
ls -la "/Applications/Company Portal.app"
# OR
ls -la "/Applications/Okta Verify.app"

# 5. Verify in Jamf Pro
# - Device appears in inventory
# - Platform SSO Registered extension attribute = Yes
# - Last check-in is recent
# - Configuration profiles installed

# 6. Check Entra ID / Okta
# - Device appears in Devices
# - Registration method: Microsoft Entra joined (Entra) or Registered (Okta)
# - Compliance status: Compliant
# - Last sync recent
```

**5. Functional Testing**

Test SSO functionality:

```bash
# Open Microsoft Office app (if Entra ID)
open -a "Microsoft Word"
# Should open without authentication prompt

# Open browser and navigate to:
https://portal.office.com
# Should sign in automatically

# Test Conditional Access (if configured)
# Navigate to protected resource
# Should be granted or denied based on compliance

# Test password sync (with Jamf Connect if deployed)
# Change password via IdP portal
# Wait 5 minutes
# Verify local password synced
```

---

## Troubleshooting Common Issues

### Issue 1: Setup Assistant Stuck on "Setting up your Mac"

**Symptoms:**
- Setup Assistant shows "Setting up your Mac" for > 10 minutes
- Progress indicator spinning indefinitely
- No authentication window appears

**Cause Analysis:**

1. **IdP app not installing**
   ```bash
   # SSH into device or use recovery mode
   # Check if app is present
   ls -la "/Applications/Company Portal.app"
   ls -la "/Applications/Okta Verify.app"
   ```

2. **Policy not triggering**
   ```bash
   # Check Jamf logs
   tail -f /var/log/jamf.log
   # Look for policy execution
   ```

3. **Network connectivity issue**
   ```bash
   # Test connectivity
   ping -c 4 jamfcloud.com
   curl -I https://login.microsoftonline.com
   ```

**Resolution Steps:**

**Option A: Wait and Monitor**
- Wait up to 30 minutes (default timeout)
- Monitor Jamf Pro policy logs
- Check device's last contact time

**Option B: Force Policy Execution (Advanced)**
```bash
# In recovery mode or via SSH:
sudo /usr/local/bin/jamf policy -id [policy-id]
```

**Option C: Restart Enrolment**
1. Power off device
2. Boot into recovery mode (Cmd+R)
3. Erase disk
4. Restart enrolment process

**Prevention:**
- Pre-stage IdP app package in Distribution Point
- Increase package priority to 1
- Test network connectivity beforehand
- Use wired connection if Wi-Fi unreliable

### Issue 2: Authentication Window Appears But Fails

**Symptoms:**
- Sign-in window loads
- User enters credentials
- Error: "Unable to authenticate" or "Registration failed"

**Diagnosis:**

```bash
# Check Platform SSO logs
log show --predicate 'subsystem == "com.apple.extensibleSingleSignOn"' \
  --last 30m --style syslog

# Check for specific errors
log show --predicate 'process == "Company Portal"' --last 30m

# Look for:
# - Network errors
# - Certificate errors
# - Token validation failures
```

**Common Causes and Fixes:**

1. **Missing Associated Domains**
   ```bash
   # Check if profile installed
   profiles show | grep -A 10 "Associated Domains"
   
   # If missing, deploy profile then restart Setup Assistant
   ```

2. **Incorrect Bundle ID in PreStage**
   - Verify Bundle ID matches installed app exactly
   - Check for typos or extra spaces
   - Re-create PreStage if necessary

3. **IdP Configuration Error**
   - Verify redirect URIs in IdP app configuration
   - Confirm user has permissions
   - Check Conditional Access policies aren't blocking

4. **Certificate Issues**
   ```bash
   # Check if SCEP cert issued
   security find-certificate -a -p /Library/Keychains/System.keychain
   
   # Look for cert with Subject containing UDID
   ```

**Resolution:**
1. Fix the underlying issue (profile, Bundle ID, IdP config)
2. Restart device
3. Re-attempt enrolment

### Issue 3: Account Created But Not Admin When Expected

**Symptom:**
- Account created successfully
- User is Standard instead of Administrator
- Need to elevate privileges

**Cause:**
- `SetAdminStatus` is `false` in Platform SSO profile
- By design for security

**Resolution:**

**Option A: Use Privilege Elevation Tool**
```
Deploy Jamf Connect (with Privileges feature)
OR
Deploy SAP Privileges
OR
Create Self Service policy for temporary admin
```

**Option B: Modify Account Post-Creation**
```bash
# Via policy or script
sudo dseditgroup -o edit -a username -t user admin
```

**Option C: Pre-Configure Admin Status**
```xml
<!-- In Platform SSO profile Extension Data -->
<key>SetAdminStatus</key>
<true/>  <!-- Makes user admin -->

⚠️ Security Note: Only use for specific roles/users
Consider using Smart Groups to scope differently
```

### Issue 4: Simplified Setup Works But Legacy Devices Break

**Symptom:**
- macOS 26 devices enrol perfectly with Simplified Setup
- macOS 15 and earlier devices fail to enrol
- Error during Setup Assistant

**Cause:**
- PreStage enabled Simplified Setup
- Older macOS versions don't support this feature
- Setup Assistant fails on incompatible OS

**Resolution:**

**Create Separate PreStages:**

**PreStage 1: macOS 26 Tahoe (Simplified Setup)**
```
Name: Production - macOS 26 Simplified Setup
Simplified Setup: ☑ Enabled
Bundle ID: com.microsoft.CompanyPortalMac
Scope: Devices with macOS 26+
```

**PreStage 2: macOS 15 and Earlier (Traditional)**
```
Name: Production - Legacy Platform SSO
Simplified Setup: ☐ Disabled
Use Jamf Connect for account creation
Scope: Devices with macOS 15 and earlier
```

**Assign Devices Appropriately:**
- In Apple Business Manager, assign devices to correct PreStage
- Use model year or manual assignment
- Monitor and migrate as devices are replaced

---

## Best Practices

### Deployment Strategy

**Phase 1: Pilot (2-4 weeks)**
```
Participants: 10-20 IT staff members
Devices: Mix of Intel T2 and Apple Silicon
Goal: Validate configuration and identify issues
Success Criteria:
  • 100% successful enrolments
  • < 15 minute setup time
  • Zero support tickets
```

**Phase 2: Early Adopters (4-6 weeks)**
```
Participants: 50-100 volunteer users across departments
Devices: Representative mix of roles and locations
Goal: Test at moderate scale with diverse scenarios
Success Criteria:
  • > 95% successful enrolments
  • < 3 support tickets per 100 enrolments
  • Positive user feedback
```

**Phase 3: Departmental Rollout (8-12 weeks)**
```
Approach: One or two departments at a time
Size: 100-500 users per department
Goal: Manage support burden and refine processes
Success Criteria:
  • > 98% successful enrolments
  • Established support runbooks
  • Automated monitoring in place
```

**Phase 4: Organisation-Wide (Ongoing)**
```
Approach: All new devices and hardware refresh
Timeline: Ongoing
Goal: 100% adoption for macOS 26+ devices
Success Criteria:
  • Simplified Setup is standard onboarding
  • Legacy methods deprecated
  • Continuous improvement cycle
```

### User Communication

**Pre-Deployment Announcement (2 weeks before)**
```
Subject: New Mac Setup Experience Coming Soon

Dear Team,

We're excited to announce an improved setup experience for new Mac devices. 
Starting [date], when you receive a new Mac, you'll notice a streamlined 
setup process that's faster and simpler.

What's New:
• Single sign-in with your work credentials
• Automatic configuration of your account
• Faster time to productivity (under 15 minutes!)

What You Need to Know:
• Have your work email and password ready
• MFA device required during setup
• Network connection needed throughout setup

Questions? Contact IT Support: bing@bing-bong.co.uk

Regards,
IT Team
```

**Setup Guide for Users**
```
# Your New Mac: Quick Setup Guide

## What You'll Need
✅ Your work email (user@company.com)
✅ Your work password
✅ Your MFA device (phone with Authenticator app)
✅ Wi-Fi network name and password

## Setup Steps (8-12 minutes)

1. **Power On**
   - Press the power button
   - Select your language and region

2. **Connect to Wi-Fi**
   - Choose your Wi-Fi network
   - Enter the password

3. **Wait for Setup**
   - Your Mac will configure automatically
   - This takes 3-5 minutes
   - Do not turn off or restart

4. **Sign In**
   - Enter your work email
   - Enter your work password
   - Complete MFA when prompted

5. **Finish Setup**
   - Your account is created automatically
   - You'll reach the desktop fully configured
   - All apps are ready to use!

## Troubleshooting

Setup stuck for > 15 minutes?
→ Contact IT Support immediately

Authentication error?
→ Verify your email and password are correct
→ Ensure MFA device is working

Need help?
→ Email: bing@bing-bong.co.uk
→ Phone: [support number]
→ Self Service app on your desktop
```

### Monitoring and Metrics

**Key Performance Indicators:**

1. **Enrolment Success Rate**
   ```
   Target: > 98%
   Measurement: Completed enrolments ÷ Attempted enrolments
   Alert Threshold: < 95%
   ```

2. **Average Setup Time**
   ```
   Target: < 12 minutes
   Measurement: Time from power-on to desktop
   Alert Threshold: > 20 minutes
   ```

3. **Platform SSO Registration Rate**
   ```
   Target: 100% (automatic with Simplified Setup)
   Measurement: Devices with PSSO registered ÷ Total enrolled
   Alert Threshold: < 98%
   ```

4. **Support Ticket Volume**
   ```
   Target: < 2 tickets per 100 enrolments
   Measurement: Setup-related tickets
   Alert Threshold: > 5 per 100
   ```

**Monitoring Dashboards:**

Create a Jamf Pro dashboard with:
- Pie chart: Enrolment success vs. failures (last 30 days)
- Line graph: Daily enrolment trend
- Table: Failed enrolments with error details
- Gauge: Platform SSO registration percentage
- Bar chart: Support tickets by category

**Automated Alerts:**

Configure alerts for:
- Enrolment failure rate > 5%
- Device stuck in Setup Assistant > 30 minutes
- Platform SSO registration pending > 24 hours
- Compliance rate drops below threshold

### Security Hardening

**1. Enforce Secure Enclave Authentication**
```xml
<!-- In Platform SSO profile, ensure: -->
<key>authentication_method</key>
<string>SecureEnclave</string>

<!-- Never use Password method for 1:1 devices -->
```

**2. Mandatory Compliance Before Access**
```
Configure Conditional Access to:
• Block non-compliant devices
• Require Secure Enclave registration
• Enforce FileVault encryption
• Verify device is MDM-managed
```

**3. Network Access Restrictions**
```
During Setup Assistant:
• Restrict network access to required endpoints only
• Block consumer email domains
• Monitor for suspicious authentication attempts
```

**4. Audit and Logging**
```
Enable comprehensive logging:
• Platform SSO registration events
• Authentication failures
• Account creation actions
• Configuration profile installations
```

**5. Incident Response Plan**
```
If authentication credentials compromised:
1. Immediately revoke access in IdP
2. Remote wipe compromised devices
3. Force Platform SSO re-registration
4. Reset Secure Enclave keys
5. Conduct security review
```

---

## Authenticated Guest Mode (Bonus Feature)

macOS 26 Tahoe also introduces **Authenticated Guest Mode** for shared device scenarios.

### Use Cases
- Healthcare: Nurses/doctors accessing shared workstations
- Education: Students using lab Macs
- Retail: Staff using shared POS devices
- Libraries: Public access terminals with authentication

### How It Works

**Traditional Shared Device:**
```
Multiple permanent accounts on one Mac
→ Accounts accumulate over time
→ Disk space consumed
→ Manual cleanup required
→ Privacy concerns
```

**Authenticated Guest Mode:**
```
User approaches Mac with NFC badge or credentials
→ Taps badge or enters credentials
→ Temporary session created
→ User works in authenticated session
→ User logs out
→ Session automatically deleted
→ Mac ready for next user
```

### Configuration

**Prerequisites:**
- macOS 26 Tahoe
- Platform SSO configured
- NFC reader (optional, for badge tap)

**Setup in Jamf Pro:**

**Platform SSO Profile Extension Data:**
```xml
<key>AuthenticatedGuestMode</key>
<dict>
    <key>Enabled</key>
    <true/>
    
    <key>SessionTimeout</key>
    <integer>480</integer>  <!-- 8 hours -->
    
    <key>AutoDeleteOnLogout</key>
    <true/>
    
    <key>NFCEnabled</key>
    <true/>
    
    <key>AllowedIdentityProviders</key>
    <array>
        <string>com.microsoft.CompanyPortalMac</string>
    </array>
</dict>
```

**User Experience:**

**With NFC Badge:**
```
1. User approaches locked Mac
2. Taps NFC badge on reader
3. Mac authenticates with IdP
4. Temporary session created
5. User works
6. User taps badge again to log out (or walks away)
7. Session ends and deletes automatically
```

**Without NFC (Manual Login):**
```
1. User sees login screen
2. Clicks "Other" or "Guest"
3. Enters work credentials
4. MFA if required
5. Temporary session created
6. User works
7. User logs out
8. Session ends and deletes
```

This feature is particularly powerful for organisations with shared device pools, eliminating account management overhead whilst maintaining security and auditability.

---

## Conclusion

macOS 26 Tahoe's Simplified Setup for Platform SSO represents the culmination of Apple's vision for enterprise identity integration. By moving identity authentication to the very beginning of the device setup process, Apple has created a truly modern, zero-touch deployment experience that benefits both IT administrators and end users.

**Key Takeaways:**

✅ **Simplified Setup transforms the onboarding experience** - reducing setup time by nearly 50% and eliminating manual registration steps

✅ **Security is enhanced from day one** - Secure Enclave-backed credentials and immediate compliance enforcement

✅ **IT overhead is significantly reduced** - fewer support tickets, automated provisioning, streamlined troubleshooting

✅ **The future is identity-first** - macOS 26 sets the standard for how enterprise devices should be provisioned

**Next Steps:**

1. **Evaluate your readiness** - Review prerequisites and update Jamf Pro if needed
2. **Plan your pilot** - Start with a small group to validate your configuration
3. **Communicate the change** - Prepare users for the new experience
4. **Deploy incrementally** - Use a phased approach to manage risk
5. **Monitor and optimise** - Track metrics and continuously improve

**Important Note on Intel Macs:**

Remember that macOS 26 Tahoe is the **final release supporting Intel-based Macs**. Begin planning your hardware refresh strategy now to transition to Apple Silicon before the next major macOS release.

---

## Additional Resources

**Apple Documentation:**
- [What's New for Enterprise in macOS Tahoe 26](https://support.apple.com/en-us/124963)
- [Platform SSO Overview](https://support.apple.com/guide/deployment/platform-single-sign-on-dep7b4b5a124/)
- [Automated Device Enrollment](https://support.apple.com/guide/deployment/automated-device-enrollment-dep23db2037d/)

**Jamf Resources:**
- [Jamf Pro 11.20 Release Notes](https://learn.jamf.com/bundle/jamf-pro-release-notes/page/Jamf_Pro_11.20.0.html)
- [Platform SSO Simplified Setup Blog](https://www.jamf.com/blog/macos-26-platform-sso-simplified-setup/)
- [JNUC 2025 Session: Identity Management with Platform SSO](https://www.jamf.com/resources/videos/jnuc-2025-identity-management/)

**Microsoft Documentation:**
- [Platform SSO for macOS with Microsoft Entra ID (GA)](https://techcommunity.microsoft.com/blog/microsoft-entra-blog/now-generally-available-platform-sso-for-macos-with-microsoft-entra-id/4437424)
- [Configure Platform SSO in Intune](https://learn.microsoft.com/en-us/intune/configuration/platform-sso-macos)

**Okta Documentation:**
- [Merging Identity with macOS Onboarding](https://www.okta.com/blog/product-innovation/merging-identity-with-macos-onboarding/)
- [Okta Device Access for macOS](https://help.okta.com/en-us/content/topics/device-access/macos-psso.htm)

**Community Resources:**
- [MacAdmins Slack](https://macadmins.org) - #platform-sso channel
- [Jamf Nation Community](https://community.jamf.com)
- [r/macsysadmin Reddit](https://reddit.com/r/macsysadmin)

---

## About the Author

**David Crosby (Bing)**  
https://www.linkedin.com/in/david-bing-crosby/

Specialising in Apple enterprise management, identity integration, and zero-touch deployment solutions. Advocate for automated, user-friendly IT experiences that enhance security without compromising productivity.

This document is provided for informational purposes. Implementation details may vary based on your specific environment and identity provider. Always test thoroughly in a non-production environment before deploying to production devices.
