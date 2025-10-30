# Platform SSO Implementation Guide for Jamf Pro
## Zero-Touch Deployment with Jamf Connect and Jamf Protect

**Document Version:** 1.1  
**Date:** 30 October 2025  
**Author:** David Crosby (Bing)  
**LinkedIn:** https://www.linkedin.com/in/david-bing-crosby/  

**Target macOS Version:** macOS 13.0 Ventura and later (Recommended: macOS 15 Sequoia or later)  
**Simple Setup macOS Version:** macOS 26 Tahoe, see seperate guide (https://github.com/david-crosby/jamf-pro-psso-guide/blob/main/tahoe-setup-guide.md)  

---

## Executive Summary

This guide provides comprehensive instructions for implementing Apple Platform Single Sign-On (Platform SSO) with Jamf Pro, enabling zero-touch deployment whilst integrating Jamf Connect for identity management and Jamf Protect for endpoint security. The implementation leverages Apple's Secure Enclave to create hardware-backed, phishing-resistant device identities that enhance your organisation's security posture.

Platform SSO represents a significant evolution in macOS identity management, providing seamless authentication to cloud resources whilst maintaining local account security. When properly configured with Jamf Pro, this creates a true zero-trust environment with minimal user friction.

---

## Table of Contents

1. [Overview of Platform SSO](#overview-of-platform-sso)
2. [Prerequisites](#prerequisites)
3. [Architecture Overview](#architecture-overview)
4. [Understanding Secure Enclave](#understanding-secure-enclave)
5. [Zero-Touch Deployment Configuration](#zero-touch-deployment-configuration)
6. [Jamf Connect Integration](#jamf-connect-integration)
7. [Platform SSO Configuration](#platform-sso-configuration)
8. [Jamf Protect Integration](#jamf-protect-integration)
9. [Device Compliance Setup](#device-compliance-setup)
10. [Configuration Profiles](#configuration-profiles)
11. [Deployment Workflow](#deployment-workflow)
12. [User Experience](#user-experience)
13. [Troubleshooting](#troubleshooting)
14. [Best Practices](#best-practices)

---

## Overview of Platform SSO

### What is Platform SSO?

Platform Single Sign-On (Platform SSO) is a framework introduced by Apple in macOS 13.0 Ventura that extends the Extensible Single Sign-On (SSO-E) framework. It provides:

- **Unified Authentication**: Links macOS login with cloud identity providers (Microsoft Entra ID, Okta)
- **Password Synchronisation**: Optionally syncs local account passwords with cloud identity passwords
- **Hardware-Backed Security**: Uses Secure Enclave for phishing-resistant authentication
- **Seamless Access**: Eliminates repeated authentication prompts for cloud resources

### Platform SSO Evolution

**macOS 13.0 Ventura (2022)**
- Initial Platform SSO framework release
- Basic password sync capabilities
- Foundation for cloud identity integration

**macOS 14.0 Sonoma (2023)**
- Cloud identity provider passwords at login window
- Secure Enclave key authentication method
- Smart card (PIV/CAC) support
- Enhanced authorisation prompt integration

**macOS 15.0 Sequoia (2024)**
- FileVault integration with Platform SSO
- Improved offline authentication handling
- Enhanced network authentication grace periods

**macOS 26 Tahoe (Released September 2025)**
- **Simplified Setup for Platform SSO** - Identity authentication during Setup Assistant
- Native provisioning of first local account from IdP credentials
- **Authenticated Guest Mode** - Temporary IdP-authenticated sessions for shared devices
- Tap to Login with NFC-enabled badges
- Final macOS release supporting Intel-based Macs
- New Liquid Glass design language across the platform

### Why Implement Platform SSO?

1. **Enhanced Security**
   - Hardware-backed authentication via Secure Enclave
   - Phishing-resistant credentials
   - Non-exportable cryptographic keys
   - Compliance with zero-trust frameworks

2. **Improved User Experience**
   - Single sign-on across macOS and cloud services
   - Reduced password fatigue
   - Seamless Microsoft 365 / Google Workspace access
   - Transparent background authentication

3. **Simplified Management**
   - Centralised identity management
   - Automated credential lifecycle
   - Enhanced visibility and reporting
   - Integration with Conditional Access policies

---

## Prerequisites

### Apple Requirements

- **macOS Version**: 13.0 Ventura or later
  - **macOS 15 Sequoia** - Current stable release with full Platform SSO support
  - **macOS 26 Tahoe** - Latest release with Simplified Setup for Platform SSO (recommended for new deployments)
  - **Note**: macOS 26 Tahoe is the final release supporting Intel-based Macs
- **Apple Business Manager**: Active ABM account with MDM server token
- **Automated Device Enrolment**: Devices enrolled via ADE (formerly DEP)
- **Device Hardware**: 
  - **Apple Silicon** (M-series): All Macs with M1 or later (recommended)
  - **Intel Macs with T2 chip**: Mac Pro (2019), MacBook Pro 16-inch (2019), MacBook Pro 13-inch (2020, four Thunderbolt 3 ports), iMac (2020)
  - **Important**: No further macOS releases beyond Tahoe will support Intel-based Macs

### Jamf Requirements

#### Jamf Pro
- **Version**: 11.20.0 or later (for macOS 26 Tahoe Simplified Setup support)
  - **11.0+** - Basic Platform SSO support
  - **11.20+** - Full Simplified Setup for Platform SSO support
- **Hosting**: Cloud or on-premises with external accessibility
- **Licence**: Active Jamf Pro licence
- **Integration**: Connected to Apple Business Manager
- **SCEP Certificate Authority**: Configured for device certificates

#### Jamf Connect
- **Version**: 2.31.0 or later (2.35.0+ recommended for macOS 26)
- **Licence**: Active Jamf Connect licence
- **Identity Provider**: Configured integration (Entra ID, Okta, Google)
- **Purpose**: Account creation and password synchronisation
- **Note**: Works alongside Platform SSO for complementary functionality

#### Jamf Protect
- **Version**: 5.0.0 or later (for macOS 26 support)
- **Licence**: Active Jamf Protect licence
- **Purpose**: Endpoint protection and compliance monitoring

### Identity Provider Requirements

#### Microsoft Entra ID (Azure AD)
- **Licence**: Microsoft Entra ID P1 or P2
- **Permissions**: Global Administrator or Application Administrator
- **Applications**:
  - Enterprise SSO app configured
  - Microsoft Company Portal app (version 5.2504.0 or later for full Platform SSO GA support)
  - Conditional Access policies (optional but recommended)
- **Features**:
  - Device registration enabled
  - Conditional Access (for compliance policies)
  - **Platform SSO**: Generally Available (GA) as of August 2025
  - **Simplified Setup**: Supported in macOS 26 Tahoe with latest Company Portal

#### Okta (Alternative)
- **Licence**: Okta Identity Engine
- **Applications**:
  - Okta Device Access configured
  - Okta Verify application (version 9.52 or later for macOS 26 Simplified Setup support)
- **Features**:
  - Device Trust enabled
  - SCEP certificate distribution
  - **Simplified Setup**: Supported in macOS 26 Tahoe with Okta Verify 9.52+

### Network Requirements

- **Connectivity**: Devices must reach identity provider endpoints
- **URLs to Allowlist**:
  ```
  # Microsoft Entra ID
  login.microsoftonline.com
  graph.microsoft.com
  enterpriseregistration.windows.net
  device.login.microsoftonline.com
  
  # Okta
  {your-domain}.okta.com
  {your-domain}.oktapreview.com
  
  # Jamf
  {your-instance}.jamfcloud.com
  ```
- **Ports**: 443 (HTTPS), 5223 (APNs)
- **Certificate**: Valid SSL certificates on all endpoints

---

## Architecture Overview

### Component Interaction

```
┌─────────────────────────────────────────────────────────────┐
│                     Apple Business Manager                   │
│                  (Device Assignment & ADE)                   │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                        Jamf Pro                              │
│  • PreStage Enrolment Configuration                          │
│  • Configuration Profiles (Platform SSO)                     │
│  • Smart Groups & Policies                                   │
│  • Device Compliance Reporting                               │
└────┬────────────────┬─────────────────┬────────────────┬────┘
     │                │                 │                │
     ▼                ▼                 ▼                ▼
┌─────────┐   ┌──────────────┐   ┌─────────┐   ┌──────────────┐
│  Jamf   │   │  Platform    │   │  Jamf   │   │  Microsoft   │
│ Connect │   │     SSO      │   │ Protect │   │  Entra ID /  │
│         │   │  Extension   │   │         │   │     Okta     │
└─────────┘   └──────────────┘   └─────────┘   └──────────────┘
     │                │                 │                │
     └────────────────┴─────────────────┴────────────────┘
                          │
                          ▼
              ┌──────────────────────┐
              │    macOS Device      │
              │  • Secure Enclave    │
              │  • Local Account     │
              │  • Cloud Identity    │
              └──────────────────────┘
```

### Authentication Flow

1. **Initial Enrolment**
   - Device powers on and connects to Wi-Fi
   - Contacts Apple Business Manager via serial number
   - Automatically enrolls in Jamf Pro
   - Jamf Connect creates first local account
   - User authenticates with cloud identity

2. **Platform SSO Registration**
   - Configuration profile deploys Platform SSO extension
   - User prompted to register device
   - Authentication with MFA to identity provider
   - Secure Enclave generates cryptographic key pair
   - Public key registered with identity provider
   - Device registered for Conditional Access

3. **Ongoing Authentication**
   - User logs in with local password (synced by Jamf Connect)
   - Platform SSO automatically obtains cloud tokens
   - Microsoft 365 / Google Workspace apps authenticate silently
   - Secure Enclave key used for device attestation
   - Conditional Access policies evaluated continuously

---

## Understanding Secure Enclave

### What is Secure Enclave?

The Secure Enclave is a dedicated secure coprocessor integrated into Apple Silicon (M-series) and Intel Macs with T2 chips. It provides:

- **Hardware-Isolated Security**: Separate processor with encrypted memory
- **Cryptographic Operations**: Key generation and storage
- **Biometric Data**: Touch ID and Face ID processing
- **Secure Boot**: Verified boot chain

### Secure Enclave in Platform SSO

When Platform SSO uses the Secure Enclave authentication method:

1. **Key Generation**
   - Private key generated within Secure Enclave
   - Key is **non-exportable** and bound to the hardware
   - Cannot be extracted, even with root access
   - Destroyed if Secure Enclave is tampered with

2. **Authentication**
   - Challenge-response using private key
   - Private key never leaves Secure Enclave
   - Public key registered with identity provider
   - Phishing-resistant by design (physical device required)

3. **Device Identity**
   - Hardware-backed device attestation
   - Tied to specific physical Mac
   - Meets FIDO2 specifications
   - Supports Conditional Access device policies

### Authentication Methods Comparison

| Method | Password Sync | Phishing Resistant | Offline Access | Secure Enclave |
|--------|--------------|-------------------|----------------|----------------|
| **Password** | ✅ Yes | ❌ No | ✅ Yes | ❌ No |
| **Secure Enclave Key** | ❌ No | ✅ Yes | ⚠️ Limited* | ✅ Yes |
| **Smart Card** | ❌ No | ✅ Yes | ❌ No | N/A |

*Secure Enclave Key method requires network connectivity for initial authentication, but grace periods can be configured.

### Recommended Configuration

**For 1:1 Devices (Assigned to Individual Users)**
- **Authentication Method**: Secure Enclave Key
- **Account Management**: Jamf Connect
- **Benefit**: Maximum security with hardware-backed credentials whilst Jamf Connect maintains local password sync

**For Shared Devices**
- **Authentication Method**: Password
- **Account Management**: Native macOS Setup Assistant or Jamf Connect
- **Benefit**: Multiple users can authenticate with cloud passwords

---

## Zero-Touch Deployment Configuration

### Apple Business Manager Setup

1. **Configure ABM Integration**
   ```
   Settings → Device Management Settings → MDM Server
   • Add Jamf Pro as MDM server
   • Download and upload MDM server token
   • Assign devices to Jamf Pro
   ```

2. **Device Assignment**
   - Purchase devices through Apple or authorised resellers
   - Devices automatically appear in ABM
   - Assign to appropriate MDM server (Jamf Pro)

### Jamf Pro PreStage Enrolment

A PreStage Enrolment defines the automated setup experience for devices.

#### PreStage Configuration

**Navigation**: `Computers → PreStage Enrolments → New`

**General Settings**
```
Display Name: Production Mac PreStage - Platform SSO
Enrolment Site: [Your Site]
Support Information:
  • Phone: +44 [Your Support Number]
  • Email: bing@bing-bong.co.uk
  • Department: IT Support
```

**Enrolment Settings**
```
☑ Require Authentication
Authentication Type: Identity Provider (Jamf Connect)
Identity Provider: [Your Entra ID / Okta Configuration]

☑ Make MDM Profile Removable: Never
☑ Await Final Configuration: Enabled
☑ Enable Setup Assistant Options

Account Settings:
☑ Create local administrator account
  Username: localadmin
  Full Name: Local Administrator
  Password: [Secure Password - Store in password manager]
```

**Setup Assistant Options**
```
Skip the Following Screens:
☑ Location Services
☑ Apple ID
☑ Terms and Conditions
☑ Touch ID / Biometrics (if desired)
☑ Display Appearance
☑ Screen Time
☑ Siri
☑ Analytics

Do Not Skip:
☐ Computer Account (Jamf Connect will handle this)
☐ FileVault (Enable via policy post-enrolment)
```

**Account Settings (with Jamf Connect)**
```
☑ Enable Account Creation with Jamf Connect

Required Information:
• Full Name: From Identity Provider
• Username: From Identity Provider
• Email Address: From Identity Provider

Optional Settings:
☑ Skip primary account setup
☐ Create additional administrator account
```

#### Smart Groups for Zero-Touch

Create Smart Groups to manage deployment phases.

**1. New Enrolments Smart Group**
```
Name: 🆕 New Enrolments - Last 3 Days
Criteria:
  Enrolment Date: less than 3 days ago
Update Frequency: 15 minutes
```

**2. Platform SSO Eligible Smart Group**
```
Name: ✅ Platform SSO Eligible
Criteria:
  AND Operating System Version: greater than or equal to 14.0
  AND Processor Architecture: arm64 OR x86_64 with T2
  AND Enrolment Method: PreStage Enrolment
Update Frequency: 30 minutes
```

**3. Platform SSO Registered Smart Group**
```
Name: 🔐 Platform SSO Registered
Criteria:
  Extension Attribute: PSSORegistered equals "Yes"
Update Frequency: 1 hour
```

**4. Platform SSO Not Registered Smart Group**
```
Name: ⏳ Platform SSO Pending Registration
Criteria:
  Computer Group: Platform SSO Eligible
  AND NOT Computer Group: Platform SSO Registered
```

#### Extension Attributes for Monitoring

**Platform SSO Registration Status**
```bash
#!/bin/bash

# Check Platform SSO registration status
pssostatus=$(/usr/bin/app-sso platform -s 2>/dev/null | grep -i "registration" | awk '{print $NF}')

if [[ "$pssostatus" == "Registered" ]]; then
    echo "<result>Yes</result>"
elif [[ "$pssostatus" == "NotRegistered" ]]; then
    echo "<result>No</result>"
else
    echo "<result>Unknown</result>"
fi
```

**Secure Enclave Authentication Method**
```bash
#!/bin/bash

# Check if Secure Enclave key authentication is enabled
seauth=$(/usr/bin/app-sso platform -s 2>/dev/null | grep -i "authentication" | grep -i "SecureEnclave")

if [[ -n "$seauth" ]]; then
    echo "<result>Secure Enclave</result>"
else
    echo "<result>Other</result>"
fi
```

---

## Jamf Connect Integration

Jamf Connect serves as the identity bridge between your cloud identity provider and macOS. It creates local accounts, synchronises passwords, and provides a customisable login experience.

### Installation and Configuration

#### 1. Create Identity Provider Application

**Microsoft Entra ID**
```
Azure Portal → Entra ID → Enterprise Applications → New Application

Application Type: Custom Application
Name: Jamf Connect

Authentication:
  Redirect URIs: 
    • jamfconnect://auth
    • https://127.0.0.1/jamfconnect
  
  Token Configuration:
    • Add optional claim: given_name
    • Add optional claim: family_name
    • Add optional claim: upn

API Permissions:
  • Microsoft Graph: User.Read (Delegated)
  • Microsoft Graph: User.ReadBasic.All (Delegated)

Client Secret: [Generate and securely store]
Application (client) ID: [Note for configuration]
Directory (tenant) ID: [Note for configuration]
```

**Okta**
```
Okta Admin Console → Applications → Create App Integration

Sign-in method: OIDC
Application type: Native Application
Name: Jamf Connect

Grant Types:
  ☑ Authorization Code
  ☑ Refresh Token

Login redirect URIs:
  • jamfconnect://auth
  • https://127.0.0.1/jamfconnect

Assignments: [Assign appropriate groups]

Note: Client ID and Okta Domain
```

#### 2. Jamf Connect Configuration Profile

Create a configuration profile in Jamf Pro for Jamf Connect.

**Navigation**: `Computers → Configuration Profiles → New`

**General**
```
Display Name: Jamf Connect - Login and Menu Bar
Level: Computer Level
Distribution Method: Install Automatically
```

**Jamf Connect Login Payload**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>OIDCProvider</key>
    <string>Azure</string>
    
    <key>OIDCTenantID</key>
    <string>[YOUR-TENANT-ID]</string>
    
    <key>OIDCClientID</key>
    <string>[YOUR-CLIENT-ID]</string>
    
    <key>OIDCRedirectURI</key>
    <string>jamfconnect://auth</string>
    
    <key>CreateAdminUser</key>
    <false/>
    
    <key>LocalUsernameFormat</key>
    <string>upn</string>
    
    <key>DenyLocal</key>
    <true/>
    
    <key>DenyLocalExcluded</key>
    <array>
        <string>localadmin</string>
    </array>
    
    <key>EnableFDE</key>
    <true/>
    
    <key>EnableFDERecoveryKey</key>
    <true/>
    
    <key>RotateUsedKey</key>
    <true/>
</dict>
</plist>
```

**Key Settings Explained:**
- `OIDCProvider`: Identity provider type (Azure, Okta, Google)
- `CreateAdminUser`: False (users created as standard, elevated via policy)
- `DenyLocal`: Prevents local account authentication
- `DenyLocalExcluded`: Allows emergency account access
- `EnableFDE`: Automatically enables FileVault
- `LocalUsernameFormat`: Uses UPN for consistent usernames

**Jamf Connect Menu Bar Payload**

```xml
<dict>
    <key>Branding</key>
    <dict>
        <key>BackgroundImage</key>
        <string>/usr/local/shared/wallpaper.png</string>
        
        <key>IconImage</key>
        <string>/usr/local/shared/company-logo.png</string>
    </dict>
    
    <key>PasswordChangeURL</key>
    <string>https://account.activedirectory.windowsazure.com/ChangePassword.aspx</string>
    
    <key>PasswordExpiry</key>
    <dict>
        <key>NotificationTime</key>
        <integer>14</integer>
    </dict>
    
    <key>RunScript</key>
    <string>/usr/local/scripts/jamfconnect-post-login.sh</string>
</dict>
```

#### 3. Post-Login Script

Create a script to run after successful Jamf Connect authentication.

**Script: `jamfconnect-post-login.sh`**

```bash
#!/bin/bash

###############################################################################
# Jamf Connect Post-Login Script
# Executes after successful cloud identity authentication
# Triggers Jamf Pro policies for software installation and configuration
###############################################################################

# Logging
LOG_FILE="/var/log/jamfconnect-postlogin.log"
exec 1>> "$LOG_FILE"
exec 2>&1

echo "$(date '+%Y-%m-%d %H:%M:%S') - Post-login script started"

# Wait for Jamf binary
echo "Waiting for Jamf Pro enrolment to complete..."
until [[ -f "/usr/local/bin/jamf" ]]; do
    sleep 2
done

echo "Jamf binary detected. Waiting for full initialisation..."
sleep 5

# Update inventory
echo "Updating inventory..."
/usr/local/bin/jamf recon

# Trigger essential software installation
echo "Triggering essential software installation..."
/usr/local/bin/jamf policy -event provisioning

# Trigger Platform SSO configuration
echo "Triggering Platform SSO setup..."
/usr/local/bin/jamf policy -event platformSSO

# Trigger Jamf Protect installation
echo "Triggering Jamf Protect installation..."
/usr/local/bin/jamf policy -event installProtect

echo "$(date '+%Y-%m-%d %H:%M:%S') - Post-login script completed"

exit 0
```

**Deploy Script via Package**
1. Create package using Jamf Composer or `pkgbuild`
2. Include script at `/usr/local/scripts/jamfconnect-post-login.sh`
3. Set permissions: `chmod +x`
4. Upload to Jamf Pro
5. Scope to PreStage devices

#### 4. Branding Assets

**Wallpaper and Logo**
- **Wallpaper**: 2560x1600 PNG (or higher for Retina)
- **Logo**: 250x250 PNG with transparent background
- **Location**: `/usr/local/shared/`
- **Deployment**: Package or policy

---

## Platform SSO Configuration

### Configuration Profile Creation

Platform SSO is deployed via an Extensible Single Sign-On configuration profile with Platform SSO enabled.

#### Microsoft Entra ID Platform SSO

**Navigation**: `Computers → Configuration Profiles → New`

**General Settings**
```
Display Name: Platform SSO - Microsoft Entra ID (Secure Enclave)
Level: Computer Level
Distribution Method: Install Automatically
```

**Single Sign-On Extensions Payload**

**Payload Type**: Single Sign-On Extensions

**SSO Extension Configuration**

```
Extension Identifier: com.microsoft.CompanyPortalMac.ssoextension
Team Identifier: UBF8T346G9
Sign-On Type: Redirect

☑ Use Platform SSO (Critical setting)

URLs:
  • https://login.microsoftonline.com
  • https://login.microsoft.com
  • https://sts.windows.net
  • https://login.partner.microsoftonline.cn
  • https://login.chinacloudapi.cn
  • https://login.microsoftonline.us
  • https://login-us.microsoftonline.com
  • https://enterpriseregistration.windows.net

Extension Data:
```

**Extension Data Keys**

```xml
<dict>
    <key>AppPrefixAllowList</key>
    <string>com.microsoft.,com.apple.</string>
    
    <key>browser_sso_interaction_enabled</key>
    <integer>1</integer>
    
    <key>disable_explicit_app_prompt</key>
    <integer>1</integer>
    
    <key>device_registration</key>
    <string>{{DEVICEUDID}}</string>
    
    <!-- Secure Enclave Configuration -->
    <key>authentication_method</key>
    <string>SecureEnclave</string>
    
    <!-- Enable offline access with 14-day grace period -->
    <key>AccountDisplayName</key>
    <string>$upn$</string>
    
    <key>LoginFrequency</key>
    <integer>0</integer>
    
    <key>NewDeviceLoginFrequency</key>
    <integer>14</integer>
    
    <!-- Platform SSO Specific Settings -->
    <key>platform_sso_enabled</key>
    <integer>1</integer>
    
    <key>Authorization</key>
    <dict>
        <key>Enabled</key>
        <true/>
        <key>RightForUserAuthentication</key>
        <string>authenticate-session-owner</string>
    </dict>
    
    <key>RegistrationToken</key>
    <dict>
        <key>Enabled</key>
        <true/>
        <key>ExpirationInDays</key>
        <integer>14</integer>
    </dict>
</dict>
```

**Critical Settings Explained:**

- **`authentication_method: SecureEnclave`**: Uses hardware-backed keys
- **`LoginFrequency: 0`**: Never requires re-authentication (rely on Conditional Access)
- **`NewDeviceLoginFrequency: 14`**: 14-day offline grace period for new devices
- **`platform_sso_enabled: 1`**: Enables Platform SSO features
- **`Authorization: Enabled`**: Allows Platform SSO for authorisation prompts
- **`RegistrationToken`**: 14-day token for device registration

#### Okta Platform SSO Configuration

```xml
Extension Identifier: com.okta.macos.login-extension
Team Identifier: OKTA_TEAM_ID
Sign-On Type: Redirect

☑ Use Platform SSO

URLs:
  • https://[your-domain].okta.com
  • https://[your-domain].oktapreview.com

Extension Data:
<dict>
    <key>AuthMode</key>
    <string>OIDC</string>
    
    <key>IncludedDomains</key>
    <array>
        <string>[your-domain].okta.com</string>
    </array>
    
    <key>IdpHost</key>
    <string>[your-domain].okta.com</string>
    
    <key>ClientId</key>
    <string>[YOUR-CLIENT-ID]</string>
    
    <key>DeviceAccessClientId</key>
    <string>[DEVICE-ACCESS-CLIENT-ID]</string>
    
    <key>RegistrationMethod</key>
    <string>SecureEnclave</string>
    
    <key>LocalUsernameFormat</key>
    <string>email</string>
</dict>
```

### SCEP Certificate Configuration

Platform SSO requires a device certificate for authentication. Configure SCEP to issue certificates automatically.

#### Create SCEP Configuration Profile

**Navigation**: `Computers → Configuration Profiles → New`

**General**
```
Display Name: SCEP Certificate - Platform SSO Device Identity
Level: Computer Level
```

**Certificate Payload**

```
Certificate Authority: Jamf Pro Built-in CA
Subject: CN=$UDID,OU=Devices,O=Your Organisation
Subject Alternative Name:
  • DNS: $SERIALNUMBER.your-domain.com
  • UPN: $UDID@your-domain.com

Key Size: 2048
Key Usage: Digital Signature, Key Encipherment
Extended Key Usage:
  • Client Authentication (1.3.6.1.5.5.7.3.2)
  • Smart Card Logon (1.3.6.1.4.1.311.20.2.2)

Key Is Extractable: No (Important for security)
Allow export from keychain: No

Retries: 3
Retry Delay: 10 seconds
```

**Scope**
```
Targets: Platform SSO Eligible (Smart Group)
Exclusions: None
```

---

## Jamf Protect Integration

Jamf Protect provides endpoint detection and response (EDR), threat prevention, and compliance monitoring. Integration with Platform SSO enables comprehensive security visibility.

### Jamf Protect Deployment

#### 1. Prerequisites

- **Jamf Protect Cloud**: Active subscription and tenant
- **System Extensions**: Approved via configuration profile
- **Full Disk Access**: Granted to Jamf Protect
- **Network Access**: Jamf Protect endpoints accessible

#### 2. Configuration Profile for System Extensions

**Navigation**: `Computers → Configuration Profiles → New`

**General**
```
Display Name: Jamf Protect - System Extensions and Privacy
Level: Computer Level
Distribution Method: Install Automatically
```

**System Extensions Payload**

```
Allowed System Extensions:
Team Identifier: 483DWKW443 (Jamf)
Allowed Extensions:
  • com.jamf.protect.agent
  • com.jamf.protect.shield
```

**Privacy Preferences Policy Control Payload**

```
Identifier: com.jamf.protect.agent
Identifier Type: Bundle ID
Code Requirement: 
  anchor apple generic and identifier "com.jamf.protect.agent" 
  and certificate leaf[subject.OU] = "483DWKW443"

Privacy Preferences:
☑ Full Disk Access
☑ System Policy All Files
☑ Accessibility (if needed)
```

#### 3. Jamf Protect Installation Policy

**Navigation**: `Computers → Policies → New`

**General**
```
Display Name: Install Jamf Protect - Zero Touch
Trigger: Custom Event (installProtect)
Execution Frequency: Once per computer
```

**Packages**
```
Package: Jamf Protect Installer.pkg
Action: Install
Priority: During next check-in
```

**Scripts** (Post-Installation Configuration)

```bash
#!/bin/bash

###############################################################################
# Jamf Protect Post-Installation Configuration
# Configures Jamf Protect with tenant-specific settings
###############################################################################

PROTECT_BINARY="/usr/local/bin/protectctl"
TENANT_ID="[YOUR-TENANT-ID]"
REGISTRATION_KEY="[YOUR-REGISTRATION-KEY]"

# Wait for Jamf Protect installation
echo "Waiting for Jamf Protect installation..."
until [[ -f "$PROTECT_BINARY" ]]; do
    sleep 5
done

echo "Jamf Protect detected. Configuring tenant registration..."
sleep 3

# Register with Jamf Protect cloud
"$PROTECT_BINARY" register \
    --tenant-id "$TENANT_ID" \
    --registration-key "$REGISTRATION_KEY"

if [[ $? -eq 0 ]]; then
    echo "Jamf Protect registered successfully"
else
    echo "ERROR: Jamf Protect registration failed"
    exit 1
fi

# Enable Jamf Protect agent
"$PROTECT_BINARY" start

echo "Jamf Protect configuration complete"
exit 0
```

**Scope**
```
Targets: New Enrolments - Last 3 Days
Exclusions: None
```

#### 4. Jamf Protect Plans and Configuration

Within Jamf Protect Cloud console, configure detection and prevention plans.

**Threat Prevention Plan**
```
Name: Production Macs - Standard Protection

Prevention:
☑ Malware Prevention
☑ Ransomware Protection
☑ Fileless Malware Protection
☑ Suspicious Script Prevention

Actions:
  • Block and Quarantine
  • Generate alert
  • Upload suspicious files (< 50MB)
```

**Analytic Plan**
```
Name: Platform SSO Security Monitoring

Key Analytics:
☑ Credential Access Attempts
☑ Unauthorised Privilege Escalation
☑ Suspicious Authentication Activity
☑ Token Theft Indicators
☑ Keychain Access Monitoring

Custom Analytic: Platform SSO Registration Status
  • Monitor: /usr/bin/app-sso
  • Alert on: Unexpected registration changes
```

**Unified Log Collection**
```
Name: Platform SSO and Identity Events

Predicates:
  • eventMessage CONTAINS "Platform SSO"
  • subsystem == "com.apple.extensibleSingleSignOn"
  • subsystem == "com.microsoft.CompanyPortalMac"
  • category == "ssoextension"
  
Retention: 90 days
```

### Integration with Compliance

Jamf Protect compliance data flows to Jamf Pro and can be used in Smart Groups.

**Extension Attribute: Jamf Protect Status**

```bash
#!/bin/bash

# Check if Jamf Protect is installed and running
if [[ -f "/usr/local/bin/protectctl" ]]; then
    protect_status=$(/usr/local/bin/protectctl status 2>/dev/null | grep "Status:" | awk '{print $2}')
    
    if [[ "$protect_status" == "Running" ]]; then
        echo "<result>Running</result>"
    else
        echo "<result>Installed - Not Running</result>"
    fi
else
    echo "<result>Not Installed</result>"
fi
```

**Smart Group: Non-Compliant Devices**

```
Name: ⚠️ Non-Compliant - Jamf Protect Issues
Criteria:
  Extension Attribute: Jamf Protect Status is not Running
  OR
  Last Check-in: more than 7 days ago
```

---

## Device Compliance Setup

Device Compliance integrates Jamf Pro with identity providers (Microsoft Entra ID) to enforce Conditional Access policies based on device state.

### Microsoft Entra ID Compliance Integration

#### 1. Configure Jamf Pro - Microsoft Entra Integration

**Jamf Pro Navigation**: `Settings → Global → Cloud Services`

**Microsoft Entra ID Integration**
```
☑ Enable Microsoft Entra ID Integration

Tenant ID: [YOUR-ENTRA-TENANT-ID]
Application (Client) ID: [YOUR-CLIENT-ID]
Client Secret: [YOUR-CLIENT-SECRET]

Test Connection → Success

Device Compliance:
☑ Enable Device Compliance
☑ Send compliance data to Microsoft Entra ID
Compliance Update Frequency: Every 2 hours

Attributes to Send:
☑ Device Name
☑ Serial Number
☑ Operating System Version
☑ MDM Enrolment Status
☑ Last Check-in
☑ Compliance Status
```

#### 2. Create Compliance Smart Groups

Jamf Pro uses Smart Groups to determine device compliance.

**Compliant Devices Smart Group**

```
Name: ✅ Compliant Devices - All Requirements Met
Criteria:
  AND Operating System Version: 14.0 or greater
  AND Last Check-in: less than 7 days ago
  AND FileVault: Enabled
  AND Computer Group is: Platform SSO Registered
  AND Computer Group is: Jamf Protect Running
  AND Computer Group is not: Known Security Issues
```

**Non-Compliant Devices Smart Group**

```
Name: ❌ Non-Compliant Devices
Criteria:
  NOT Computer Group: Compliant Devices - All Requirements Met
```

#### 3. Configure Compliance in Jamf Pro

**Navigation**: `Settings → Device Compliance`

```
Compliance Method: Smart Group Based

Compliant Smart Group: ✅ Compliant Devices - All Requirements Met
Non-Compliant Smart Group: ❌ Non-Compliant Devices

Compliance Evaluation Frequency: Every 2 hours

Actions on Non-Compliance:
☑ Notify user via Self Service
☑ Send push notification
☑ Generate report for IT
☐ Revoke access (Handled by Conditional Access)
```

#### 4. Microsoft Entra Conditional Access Policies

**Microsoft Entra Admin Centre**: `Protection → Conditional Access → Policies`

**Policy 1: Require Compliant Device for All Cloud Apps**

```
Name: CA001 - Require Compliant or Hybrid Joined Device

Assignments:
  Users: All Users
  Exclusions: Break Glass Accounts
  
  Target Resources:
    Cloud Apps: All cloud apps
    User Actions: None
  
  Conditions:
    Platforms: macOS
    Client Apps: Browser, Mobile apps and desktop clients

Access Controls:
  Grant:
    ☑ Require device to be marked as compliant
    OR
    ☑ Require Microsoft Entra hybrid joined device
  
  Session: None

Enable Policy: Report-only (initially), then On

```

**Policy 2: Block Access from Non-Compliant Devices**

```
Name: CA002 - Block Non-Compliant macOS Devices

Assignments:
  Users: All Users
  Exclusions: Break Glass Accounts
  
  Target Resources:
    Cloud Apps: Office 365, SharePoint, Exchange
  
  Conditions:
    Platforms: macOS
    Device State:
      Include: All device state
      Exclude: Device marked compliant

Access Controls:
  Grant: Block Access
  
Enable Policy: On
```

**Policy 3: Require MFA for Non-Compliant Devices**

```
Name: CA003 - Step-Up MFA for Non-Compliant

Assignments:
  Users: All Users
  
  Target Resources:
    Cloud Apps: All cloud apps
  
  Conditions:
    Platforms: macOS
    Device State: Device NOT marked compliant

Access Controls:
  Grant:
    ☑ Require multifactor authentication
  
  Session:
    Sign-in frequency: Every time

Enable Policy: On
```

### Compliance Monitoring and Reporting

#### Jamf Pro Dashboard

Create a dashboard for compliance visibility.

**Navigation**: `Dashboard → New Dashboard`

```
Dashboard Name: Platform SSO & Compliance Overview

Widgets:
1. Pie Chart: Compliance Status
   Data Source: Smart Groups
   Groups:
     • Compliant Devices
     • Non-Compliant Devices
     • Not Evaluated

2. Table: Non-Compliant Devices Details
   Columns:
     • Computer Name
     • Username
     • Last Check-in
     • Compliance Issue
     • Action Required

3. Line Chart: Compliance Trend (30 Days)
   Y-Axis: Number of Devices
   Lines:
     • Compliant
     • Non-Compliant

4. Table: Platform SSO Registration Status
   Columns:
     • Computer Name
     • PSSORegistered
     • Last Registration
     • Authentication Method
```

#### Extension Attribute: Compliance Status

```bash
#!/bin/bash

###############################################################################
# Compliance Status Check
# Determines overall device compliance based on multiple factors
###############################################################################

compliance_issues=()

# Check OS version
os_version=$(sw_vers -productVersion)
required_version="14.0"
if [[ $(echo "$os_version >= $required_version" | bc) -eq 0 ]]; then
    compliance_issues+=("OS_VERSION")
fi

# Check FileVault
fv_status=$(fdesetup status | grep -o "FileVault is On")
if [[ -z "$fv_status" ]]; then
    compliance_issues+=("FILEVAULT_DISABLED")
fi

# Check Platform SSO
psso_status=$(/usr/bin/app-sso platform -s 2>/dev/null | grep -o "Registered")
if [[ -z "$psso_status" ]]; then
    compliance_issues+=("PSSO_NOT_REGISTERED")
fi

# Check Jamf Protect
if [[ ! -f "/usr/local/bin/protectctl" ]]; then
    compliance_issues+=("PROTECT_NOT_INSTALLED")
else
    protect_running=$(/usr/local/bin/protectctl status 2>/dev/null | grep -o "Running")
    if [[ -z "$protect_running" ]]; then
        compliance_issues+=("PROTECT_NOT_RUNNING")
    fi
fi

# Check last recon (Jamf check-in)
last_recon=$(defaults read /Library/Preferences/com.jamfsoftware.jamf.plist lastReconDate 2>/dev/null)
current_epoch=$(date +%s)
recon_epoch=$(date -j -f "%Y-%m-%d %H:%M:%S %z" "$last_recon" +%s 2>/dev/null)
days_since_recon=$(( ($current_epoch - $recon_epoch) / 86400 ))
if [[ $days_since_recon -gt 7 ]]; then
    compliance_issues+=("LAST_CHECKIN_OLD")
fi

# Output result
if [[ ${#compliance_issues[@]} -eq 0 ]]; then
    echo "<result>Compliant</result>"
else
    issues_string=$(IFS=,; echo "${compliance_issues[*]}")
    echo "<result>Non-Compliant: $issues_string</result>"
fi
```

---

## Configuration Profiles

### Summary of Required Configuration Profiles

| Profile Name | Payload Type | Purpose | Scope |
|-------------|--------------|---------|-------|
| **Jamf Connect - Login** | Jamf Connect | Account creation and authentication | PreStage Devices |
| **Jamf Connect - Menu Bar** | Jamf Connect | Password sync and management | All Devices |
| **Platform SSO - Microsoft Entra ID** | SSO Extensions (with Platform SSO) | Enable Platform SSO with Secure Enclave | Platform SSO Eligible |
| **SCEP Certificate - Device Identity** | SCEP | Device authentication certificate | Platform SSO Eligible |
| **Jamf Protect - System Extensions** | System Extensions, Privacy | Approve Jamf Protect extensions | All Devices |
| **Microsoft SSO Extension** | SSO Extensions (Redirect) | Cloud resource SSO | All Devices |
| **FileVault** | FileVault | Disk encryption | All Devices |
| **Security & Privacy** | Restrictions | Enforce security settings | All Devices |

### Profile Deployment Order

The order of profile deployment is critical for successful zero-touch deployment.

#### Phase 1: Enrolment (Automatic via PreStage)

1. **Management Profile** (Automatic)
   - Delivered during ADE enrolment
   - Establishes MDM communication

2. **Jamf Connect - Login** (PreStage)
   - Enables cloud identity authentication
   - Creates first local account

#### Phase 2: Initial Configuration (Immediate Post-Enrolment)

3. **Jamf Connect - Menu Bar**
   - Provides password management
   - Enables post-login script execution

4. **SCEP Certificate - Device Identity**
   - Issues device certificate
   - Required before Platform SSO

5. **Microsoft SSO Extension** (Redirect)
   - Enables browser and app SSO
   - Foundation for Platform SSO

#### Phase 3: Platform SSO (Post Account Creation)

6. **Platform SSO - Microsoft Entra ID**
   - Deploys Platform SSO extension
   - Triggers user registration workflow

#### Phase 4: Security and Compliance (Ongoing)

7. **Jamf Protect - System Extensions**
   - Approves security agents
   - Enables threat protection

8. **FileVault**
   - Encrypts disk
   - Stores recovery key in Jamf Pro

9. **Security & Privacy**
   - Enforces security baselines
   - Configures privacy settings

### Profile Scoping Strategy

Use Smart Groups to automatically scope profiles appropriately.

**Example Scoping**

```
Profile: Platform SSO - Microsoft Entra ID

Targets:
  Computer Group: Platform SSO Eligible
  AND
  Computer Group: Jamf Connect Configured
  AND NOT
  Computer Group: Platform SSO Registered

Exclusions:
  Computer Group: Shared Devices
```

---

## Deployment Workflow

### End-to-End Deployment Process

#### Timeline Overview

```
Day 0-1: Device Ships from Apple
  ↓
Day 1: User Receives Device
  ↓
  0-5 min: Power On & Wi-Fi Connection
  ↓
  5-10 min: Automated Enrolment & Setup Assistant
  ↓
  10-15 min: Jamf Connect Account Creation
  ↓
  15-25 min: Software Installation (Background)
  ↓
  25-30 min: Platform SSO Registration (User Action Required)
  ↓
  30-35 min: Final Configuration & Verification
  ↓
Day 1 Complete: Device Fully Configured and Compliant
```

#### Detailed Steps

**Step 1: Device Power-On (0-2 minutes)**

User Actions:
1. Unbox device
2. Press power button
3. Select language and region
4. Connect to Wi-Fi

Background Process:
- Device contacts Apple Activation servers
- Serial number lookup in Apple Business Manager
- MDM assignment to Jamf Pro identified
- MDM profile download initiated

**Step 2: Automated Enrolment (2-5 minutes)**

User Actions:
- Minimal (most screens skipped via PreStage)
- May see: Terms & Conditions (if not skipped)

Background Process:
- MDM profile installs
- Device enrols in Jamf Pro
- PreStage configurations apply
- Smart Groups evaluate
- Configuration profiles begin deploying
- Policies trigger: `enrollmentComplete`

**Step 3: Jamf Connect Authentication (5-10 minutes)**

User Actions:
1. Jamf Connect login screen appears
2. Enter corporate email address
3. Redirect to Microsoft / Okta sign-in
4. Authenticate with MFA
5. Local account created automatically

Background Process:
- Jamf Connect validates identity with IdP
- Token exchange and validation
- Local account created with cloud username
- Password hash stored
- Post-login script executes:
  - `jamf recon` (inventory update)
  - Triggers: `provisioning` custom event
  - Triggers: `platformSSO` custom event
  - Triggers: `installProtect` custom event

**Step 4: Software Installation (10-25 minutes)**

User Experience:
- Status window (DEPNotify or Jamf Connect Notify) shows progress
- "Setting up your Mac..."
- Progress bar with installation stages

Background Process:
```
Priority 10: Essential Software
  • Microsoft Company Portal
  • Microsoft Office Suite
  • Web Browsers (Chrome, Edge)

Priority 20: Security Software
  • Jamf Protect
  • Jamf Trust (if applicable)

Priority 30: Productivity Tools
  • Slack, Zoom, etc.
  • Department-specific applications

Priority 40: Configuration
  • SCEP Certificate installation
  • Printer configurations
  • Network settings

Priority 50: Final Tasks
  • Enable FileVault
  • Computer rename based on user
  • Inventory update
```

**Step 5: Platform SSO Registration (25-30 minutes)**

User Actions:
1. Notification appears: "Register your device for seamless access"
2. Click notification or open System Settings → Privacy & Security
3. Platform SSO registration prompt appears
4. Click "Continue"
5. Authenticate with MFA
6. Registration completes

Technical Process:
- Platform SSO extension activates
- Registration request sent to Microsoft Entra ID
- Secure Enclave generates key pair
- Public key uploaded to Azure
- Device registered in Entra ID
- Conditional Access policies apply
- Compliance evaluation begins

**Step 6: Verification and Handoff (30-35 minutes)**

User Actions:
- Final status message: "Your Mac is ready!"
- Prompted to restart (if needed)
- Can begin using device

Background Process:
- Final inventory update (`jamf recon`)
- Verification scripts run
- Compliance status sent to Entra ID
- Device marked as compliant
- User granted access to corporate resources

#### IT Administrator View

**Jamf Pro Activity Log**

```
[10:00:00] Device serial ABC123 enrolled via PreStage
[10:00:30] Configuration profiles deployed (8 profiles)
[10:01:00] Jamf Connect Login successful for user: bing@bing-bong.co.uk
[10:01:15] Local account created: bing
[10:01:30] Policy triggered: Essential Software Installation
[10:05:00] Microsoft Company Portal installed
[10:06:00] Jamf Protect installed and registered
[10:08:00] SCEP certificate issued
[10:10:00] Platform SSO profile deployed
[10:25:00] Platform SSO registration completed
[10:25:30] Device compliance status: Compliant
[10:26:00] Inventory updated - all software installed
[10:26:15] Device: ABC123 ready for production use
```

---

## User Experience

### User Journey Map

#### First Login Experience

**Jamf Connect Login Screen**
```
┌─────────────────────────────────────┐
│         [Company Logo]              │
│                                     │
│   Welcome to Your New Mac          │
│                                     │
│   Sign in with your work email:    │
│   ┌─────────────────────────────┐  │
│   │ bing@bing-bong.co.uk       │  │
│   └─────────────────────────────┘  │
│                                     │
│         [Continue]                  │
│                                     │
│   Need Help? support@company.com   │
└─────────────────────────────────────┘
```

**Microsoft / Okta Authentication**
- Redirect to identity provider
- Standard MFA prompt (authenticator app, SMS, etc.)
- Conditional Access evaluation
- Success: Return to macOS

**Desktop First View**
```
┌─────────────────────────────────────┐
│  ⬤ ⬤ ⬤          [Setting up your Mac...]          │
│                                     │
│  Installing essential software...   │
│  ████████████░░░░░░░░  60%         │
│                                     │
│  ✓ Microsoft Office                │
│  ✓ Web Browsers                    │
│  ⏳ Security Software               │
│  ⏳ Productivity Tools              │
│                                     │
│  Estimated time remaining: 5 min   │
└─────────────────────────────────────┘
```

#### Platform SSO Registration Prompt

**System Notification**
```
┌──────────────────────────────┐
│  🔐 Action Required          │
│                              │
│  Register your device for    │
│  seamless access to work apps│
│                              │
│  This one-time setup enables │
│  automatic sign-in to        │
│  Microsoft 365 and other     │
│  company resources.          │
│                              │
│   [Not Now]    [Register]    │
└──────────────────────────────┘
```

**Registration Flow**
1. User clicks "Register"
2. System Settings opens → Privacy & Security → Profiles
3. Platform SSO registration prompt:
   ```
   Microsoft wants to register this Mac to access
   work resources securely.
   
   This will enable:
   • Automatic sign-in to Microsoft 365
   • Hardware-backed security
   • Compliance with company policies
   
   [Cancel]  [Continue]
   ```
4. User clicks "Continue"
5. Brief MFA prompt (Touch ID or password)
6. Success message:
   ```
   ✓ Registration Complete
   
   Your Mac is now registered and you can access
   all company resources seamlessly.
   ```

#### Ongoing User Experience

**Daily Login**
- User enters local password at login screen
- Platform SSO automatically authenticates in background
- Microsoft 365 apps open without additional prompts
- Browser-based apps authenticate automatically

**Password Change**
- User changes password via Jamf Connect menu bar icon
- Or visits password change URL
- Local password automatically syncs (via Jamf Connect)
- Cloud password updates
- Platform SSO continues working seamlessly

**Offline Experience**
- User logs in with local password (works offline)
- Grace period: 14 days without network authentication
- Cloud apps require network connection
- Local work continues unaffected

---

## Troubleshooting

### Common Issues and Resolutions

#### Issue 1: Platform SSO Registration Fails

**Symptoms:**
- User clicks "Register" but receives error
- Registration completes but device not showing in Entra ID
- "Unable to communicate with identity provider" error

**Diagnosis:**

```bash
# Check Platform SSO status
/usr/bin/app-sso platform -s

# Check SSO extension logs
log show --predicate 'subsystem == "com.apple.extensibleSingleSignOn"' --last 1h

# Verify Company Portal is running
ps aux | grep "Company Portal"

# Check network connectivity to Microsoft endpoints
curl -v https://login.microsoftonline.com
curl -v https://enterpriseregistration.windows.net
```

**Resolution:**

1. **Verify Prerequisites**
   ```bash
   # Ensure Company Portal is installed
   ls -la "/Applications/Company Portal.app"
   
   # Check Platform SSO profile is installed
   profiles show | grep -A 5 "Platform SSO"
   
   # Verify SCEP certificate exists
   security find-certificate -a -c "PSSOe"
   ```

2. **Check Configuration Profile Settings**
   - Verify `enterpriseregistration.windows.net` is in the URL list
   - Confirm `authentication_method` is set to `SecureEnclave`
   - Validate Extension Data keys are correct

3. **Force Profile Reinstall**
   ```bash
   # From Jamf Pro, re-scope the Platform SSO profile
   # OR manually remove and reinstall:
   sudo profiles remove -identifier com.company.platformsso
   # Then push profile again from Jamf Pro
   ```

4. **Manual Registration Attempt**
   ```bash
   # Trigger registration via Company Portal
   open -a "Company Portal"
   # Navigate to Settings → Register Device
   ```

5. **Check Entra ID Device Registration**
   - Azure Portal → Entra ID → Devices
   - Search for device by serial number or name
   - Verify registration method shows "Microsoft Entra joined"

#### Issue 2: Duplicate Device Records in Entra ID

**Symptoms:**
- Two or more device records with same serial number
- One shows "Registered", others show "Pending" or "Stale"
- Authentication fails intermittently

**Cause:**
- Existing Device Compliance registration not cleaned before Platform SSO deployment

**Resolution:**

```bash
#!/bin/bash
###############################################################################
# Clean Existing Device Compliance Registration
# Run before deploying Platform SSO to devices with existing registrations
###############################################################################

echo "Cleaning existing Device Compliance registration..."

# Remove Workplace Join keys from keychain
security delete-generic-password -l "MS-Organization-Access" 2>/dev/null
security delete-generic-password -l "Jamf Conditional Access Account" 2>/dev/null
security delete-generic-password -a "com.microsoft.workplacejoin.*" 2>/dev/null

# Remove identity preferences
sudo rm -f /Library/Preferences/com.apple.security.identityservices.plist
sudo rm -f /Library/Preferences/com.apple.security.identityservices.plist.lockfile

# Remove JamfAAD data
sudo rm -rf /Library/Application\ Support/JamfAAD/
sudo defaults delete com.jamfsoftware.jamf azureDeviceId 2>/dev/null
sudo defaults delete com.jamfsoftware.jamf azureUserId 2>/dev/null

# Run inventory update
/usr/local/bin/jamf recon

echo "Cleanup complete. Deploy Platform SSO profile now."
```

**After Cleanup:**
1. Deploy Platform SSO profile
2. User registers device (new record created)
3. Manually delete old device records in Entra ID

#### Issue 3: Secure Enclave Authentication Fails

**Symptoms:**
- Platform SSO shows "Registered" but apps still prompt for passwords
- Error: "Authentication method not supported"
- Touch ID prompts appear but fail

**Diagnosis:**

```bash
# Verify Secure Enclave is available
/usr/sbin/system_profiler SPiBridgeDataType | grep "Model Name"

# Check authentication method in use
/usr/bin/app-sso platform -s | grep -i "authentication"

# Check Company Portal logs
log show --predicate 'process == "Company Portal"' --info --last 30m
```

**Resolution:**

1. **Verify Hardware Support**
   - T2 Chip: Intel Macs (2018+)
   - Apple Silicon: All M-series Macs
   
   If neither present, Secure Enclave key auth won't work.
   **Fallback:** Use Password authentication method instead.

2. **Update Extension Data in Profile**
   ```xml
   <!-- If Secure Enclave fails, temporarily use Password method -->
   <key>authentication_method</key>
   <string>Password</string>
   ```

3. **Verify Touch ID Configuration**
   ```bash
   # Check if Touch ID is configured for user
   bioutil -c
   ```

4. **Re-register Device**
   ```bash
   # Unregister Platform SSO
   /usr/bin/app-sso platform -u
   
   # Wait 30 seconds
   sleep 30
   
   # Trigger re-registration via Company Portal
   open -a "Company Portal"
   ```

#### Issue 4: Jamf Protect Blocks Platform SSO

**Symptoms:**
- Platform SSO registration fails with generic error
- Jamf Protect logs show blocked process
- Authentication requests time out

**Diagnosis:**

```bash
# Check Jamf Protect status
/usr/local/bin/protectctl status

# Review Jamf Protect logs for blocks
log show --predicate 'process == "JamfProtectDaemon"' --info --last 1h | grep -i "block"

# Check for Company Portal blocks
/usr/local/bin/protectctl logs | grep "Company Portal"
```

**Resolution:**

1. **Create Jamf Protect Allowlist**
   
   In Jamf Protect console:
   ```
   Plans → [Your Prevention Plan] → Edit
   
   Exclusions:
   Add Process Exclusions:
   • Process: Company Portal
   • Path: /Applications/Company Portal.app
   • Rule: Allow All
   
   Add Network Exclusions:
   • Domain: login.microsoftonline.com
   • Domain: enterpriseregistration.windows.net
   • Rule: Allow All
   ```

2. **Update Analytic Rules**
   ```
   Analytics → Edit Custom Analytic
   
   Exclude processes:
   • /usr/bin/app-sso
   • /Applications/Company Portal.app/Contents/MacOS/Company Portal
   ```

3. **Temporarily Disable Prevention (Testing Only)**
   ```bash
   # For testing purposes only - DO NOT use in production
   sudo /usr/local/bin/protectctl prevention disable
   
   # Attempt Platform SSO registration
   # ...
   
   # Re-enable prevention
   sudo /usr/local/bin/protectctl prevention enable
   ```

#### Issue 5: FileVault Recovery Key Not Escrowing

**Symptoms:**
- FileVault enabled but recovery key missing in Jamf Pro
- "No valid recovery key found" in inventory
- Users unable to reset forgotten passwords

**Diagnosis:**

```bash
# Check FileVault status
sudo fdesetup status

# Check if recovery key is escrowable
sudo fdesetup hasinstitutionalrecoverykey

# Verify MDM profile has FileVault payload
profiles show -type configuration | grep -A 10 "FileVault"
```

**Resolution:**

1. **Verify FileVault Configuration Profile**
   ```
   Jamf Pro → Configuration Profiles → FileVault
   
   Settings:
   ☑ Enable FileVault
   ☑ Defer enablement until logout
   ☑ Show recovery key to user: No
   ☑ Use institutional recovery key
   
   Escrow Location: Jamf Pro
   ```

2. **Force Key Escrow**
   
   Create and run a policy:
   ```bash
   #!/bin/bash
   # Force FileVault key escrow to Jamf Pro
   
   sudo /usr/local/bin/jamf policy -event escrowKey
   sudo fdesetup changerecovery -institutional
   sudo /usr/local/bin/jamf recon
   ```

3. **Verify Key Upload**
   - Jamf Pro → Computers → [Device] → Security
   - Check "FileVault 2 Individual Recovery Key" field
   - Should show encrypted key string

#### Issue 6: Users Locked Out After Password Change

**Symptoms:**
- User changes password via portal
- Local Mac password doesn't sync
- User locked out of Mac

**Cause:**
- Jamf Connect password sync not configured
- Platform SSO not syncing with Jamf Connect
- Network connectivity issue during sync

**Prevention:**

Ensure proper configuration:

```xml
<!-- Jamf Connect Menu Bar Configuration -->
<dict>
    <key>PasswordSync</key>
    <dict>
        <key>Enabled</key>
        <true/>
        
        <key>SyncOnPasswordChange</key>
        <true/>
        
        <key>RequireNetworkConnection</key>
        <true/>
    </dict>
</dict>
```

**Recovery:**

1. **Use Local Admin Account**
   ```bash
   # Log in as localadmin
   # Reset user password via System Settings
   sudo dscl . -passwd /Users/username newpassword
   ```

2. **Force Password Sync**
   ```bash
   # As logged in user, trigger Jamf Connect sync
   /Applications/Jamf\ Connect.app/Contents/MacOS/Jamf\ Connect password sync
   ```

3. **User Self-Service Recovery**
   - Direct user to Self Service app
   - Run "Reset Password" policy
   - Prompts for new password and syncs locally

### Logging and Diagnostics

#### Enable Verbose Logging

**Platform SSO Extension Logs**
```bash
# Stream live Platform SSO logs
log stream --predicate 'subsystem == "com.apple.extensibleSingleSignOn"' --level debug

# Show last hour of Platform SSO activity
log show --predicate 'subsystem == "com.apple.extensibleSingleSignOn"' --last 1h --style syslog
```

**Microsoft Company Portal Logs**
```bash
# Export Company Portal diagnostics
# Open Company Portal → Help → Save Diagnostics Report

# Logs saved to: ~/Downloads/CompanyPortal-Diagnostics-[timestamp].zip

# Key files:
# - SSOExtension.log
# - CompanyPortal.log
# - DeviceRegistration.log
```

**Jamf Connect Logs**
```bash
# Jamf Connect Login logs
tail -f /var/log/jamfconnect.log

# Menu bar app logs
log show --predicate 'process == "Jamf Connect"' --last 30m
```

**Jamf Protect Logs**
```bash
# Export Jamf Protect logs
/usr/local/bin/protectctl logs --export /tmp/protect_logs.zip

# View live threat events
/usr/local/bin/protectctl threat list

# Check prevention actions
/usr/local/bin/protectctl prevention list
```

#### Collect Diagnostic Bundle

Create a script to collect all relevant diagnostics:

```bash
#!/bin/bash
###############################################################################
# Platform SSO Diagnostic Collection Script
# Collects all relevant logs and configuration data for troubleshooting
###############################################################################

BUNDLE_DIR="/tmp/psso-diagnostics-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BUNDLE_DIR"

echo "Collecting Platform SSO diagnostics..."

# System information
echo "=== System Info ===" > "$BUNDLE_DIR/system_info.txt"
sw_vers >> "$BUNDLE_DIR/system_info.txt"
system_profiler SPHardwareDataType >> "$BUNDLE_DIR/system_info.txt"
system_profiler SPiBridgeDataType >> "$BUNDLE_DIR/system_info.txt"

# Platform SSO status
echo "=== Platform SSO Status ===" > "$BUNDLE_DIR/psso_status.txt"
/usr/bin/app-sso platform -s >> "$BUNDLE_DIR/psso_status.txt" 2>&1

# Configuration profiles
echo "=== Configuration Profiles ===" > "$BUNDLE_DIR/profiles.txt"
profiles show -type configuration >> "$BUNDLE_DIR/profiles.txt"

# SSO Extension logs (last 24h)
log show --predicate 'subsystem == "com.apple.extensibleSingleSignOn"' --last 24h --style syslog > "$BUNDLE_DIR/sso_extension_logs.txt"

# Jamf logs
tail -n 500 /var/log/jamf.log > "$BUNDLE_DIR/jamf.log"

# Network connectivity tests
echo "=== Network Tests ===" > "$BUNDLE_DIR/network_tests.txt"
curl -I https://login.microsoftonline.com >> "$BUNDLE_DIR/network_tests.txt" 2>&1
curl -I https://enterpriseregistration.windows.net >> "$BUNDLE_DIR/network_tests.txt" 2>&1

# Keychain certificates
echo "=== Certificates ===" > "$BUNDLE_DIR/certificates.txt"
security find-certificate -a -p /Library/Keychains/System.keychain >> "$BUNDLE_DIR/certificates.txt"

# Jamf Protect status
if [[ -f "/usr/local/bin/protectctl" ]]; then
    /usr/local/bin/protectctl status > "$BUNDLE_DIR/protect_status.txt"
fi

# Create archive
cd /tmp
zip -r "psso-diagnostics-$(date +%Y%m%d-%H%M%S).zip" "$(basename $BUNDLE_DIR)"
rm -rf "$BUNDLE_DIR"

echo "Diagnostic bundle created: /tmp/psso-diagnostics-*.zip"
echo "Please upload this file to your support ticket."
```

---

## Best Practices

### Security Best Practices

#### 1. Use Secure Enclave Authentication for 1:1 Devices

**Always** use Secure Enclave key authentication for devices assigned to individual users. This provides hardware-backed, phishing-resistant credentials.

**Configuration:**
```xml
<key>authentication_method</key>
<string>SecureEnclave</string>
```

**Benefits:**
- Non-exportable keys
- Phishing-resistant
- Hardware-bound identity
- Meets compliance requirements

#### 2. Implement Conditional Access Policies

Leverage identity provider Conditional Access to enforce security requirements.

**Recommended Policies:**
- Require compliant device for all cloud apps
- Block access from non-compliant devices
- Require MFA for administrative actions
- Limit access based on location and risk score

#### 3. Regular Compliance Monitoring

**Daily:**
- Review Jamf Pro compliance dashboard
- Monitor devices falling out of compliance
- Investigate unusual patterns

**Weekly:**
- Review Platform SSO registration rates
- Check for devices with old check-in dates
- Verify Jamf Protect is active on all devices

**Monthly:**
- Audit Conditional Access policy effectiveness
- Review security incidents and adjust analytics
- Update Smart Groups based on new requirements

#### 4. Enable FileVault with MDM Escrow

**Always** enable FileVault and escrow recovery keys to Jamf Pro.

```
FileVault Profile Settings:
☑ Use institutional recovery key
☑ Escrow key to Jamf Pro
☐ Show key to user (never)
☑ Rotate key on password change
```

#### 5. Implement Principle of Least Privilege

**User Accounts:**
- Create users as **standard accounts** by default
- Use privilege elevation tools (Jamf Connect Privileges, SAP Privileges)
- Grant temporary admin rights only when needed

**Service Accounts:**
- Dedicated service accounts for specific functions
- Unique credentials (never shared)
- Regular credential rotation

### Operational Best Practices

#### 1. Phased Rollout Strategy

**Phase 1: Pilot (10-20 devices)**
- IT team members
- Early adopters from various departments
- Test all workflows thoroughly
- Gather feedback

**Phase 2: Department Rollout (50-100 devices)**
- One or two departments
- Monitor closely for issues
- Refine based on feedback

**Phase 3: Organisationwide (All devices)**
- Gradual rollout over 2-4 weeks
- Communication and training plan
- Support resources ready

#### 2. User Communication Plan

**Before Deployment:**
- Email announcement about new authentication method
- Benefits: seamless access, better security
- What to expect during setup
- Support contact information

**During Onboarding:**
- Jamf Connect welcome screen with clear instructions
- In-app guidance for Platform SSO registration
- Link to video tutorial or knowledge base

**Post-Deployment:**
- Follow-up email confirming successful setup
- Tips for daily use
- How to change password
- Troubleshooting resources

#### 3. Documentation and Knowledge Base

**Admin Documentation:**
- Complete deployment procedures
- Troubleshooting guides
- Configuration profile templates
- Script repository

**User Documentation:**
- Quick start guide
- Platform SSO registration walkthrough
- Password change instructions
- FAQ

**Video Tutorials:**
- Unboxing and first setup
- Registering for Platform SSO
- Using Self Service
- Password management

#### 4. Backup and Recovery Planning

**Device Recovery:**
- Maintain local admin accounts on all devices
- Document recovery procedures
- Test recovery workflows regularly

**Configuration Backup:**
- Export all configuration profiles quarterly
- Document Smart Group criteria
- Maintain policy templates
- Version control for scripts

**Disaster Recovery:**
- Backup Jamf Pro database regularly
- Document Jamf Cloud hosting failover
- Test restoration procedures annually

#### 5. Change Management

**Configuration Changes:**
1. Test in development environment (separate Jamf instance if possible)
2. Deploy to pilot group (10 devices)
3. Monitor for 48 hours
4. Deploy to production

**Major Updates:**
- Announce changes to IT team 1 week before
- Deploy to pilot first
- Monitor for 1 week before wider rollout
- Have rollback plan ready

### Performance Optimisation

#### 1. Smart Group Efficiency

**Optimise Smart Group Criteria:**
- Use static groups where possible for large sets
- Limit nested group memberships
- Set appropriate update frequencies

**Example Optimised Group:**
```
Name: Platform SSO Eligible
Criteria:
  Computer Group: T2 or Apple Silicon Macs (static)
  AND
  Operating System Version: >= 14.0
  AND
  Enrolment Method: PreStage Enrolment

Update Frequency: 1 hour (not 15 minutes)
```

#### 2. Policy Execution Optimisation

**Chain Policies Instead of Large Single Policies:**
```
Policy: Essential Software (Event: provisioning)
  ├─ Trigger: 10-install-company-portal
  ├─ Trigger: 20-install-office
  ├─ Trigger: 30-install-browsers
  └─ Trigger: 40-final-config
```

**Benefits:**
- Faster individual policy execution
- Better fault isolation
- Easier troubleshooting
- More granular control

#### 3. Network Optimisation

**Configure Distribution Points Efficiently:**
- Use cloud distribution points for remote workers
- Regional distribution points for offices
- Limit package size where possible
- Cache frequently used packages

**Bandwidth Management:**
- Schedule large deployments during off-hours
- Use maintenance windows for major updates
- Implement bandwidth throttling if necessary

### Monitoring and Alerting

#### 1. Key Metrics to Monitor

**Device Health:**
- Enrolment success rate (target: >95%)
- Check-in frequency (devices checking in daily)
- OS version distribution
- Compliance rate (target: >98%)

**Platform SSO:**
- Registration rate (target: 100% within 24h of enrolment)
- Authentication success rate
- Secure Enclave usage percentage
- Failed registration attempts

**Security:**
- Jamf Protect active percentage (target: 100%)
- FileVault enabled percentage (target: 100%)
- Threats detected and blocked
- Non-compliant device count

#### 2. Alerting Configuration

**Critical Alerts (Immediate Action):**
- Device compliance drops below 95%
- Jamf Protect threats detected
- Platform SSO registration failures spike
- Widespread enrolment failures

**Warning Alerts (Review Within 24h):**
- Devices not checking in for 7+ days
- Platform SSO registration pending for 48h
- FileVault recovery key missing
- Configuration profile installation failures

**Informational (Weekly Review):**
- New device enrolments
- Software installation trends
- User password change patterns
- Help desk ticket trends

#### 3. Reporting

**Weekly Report:**
```
Subject: Jamf & Platform SSO Weekly Summary

Summary:
- Total Managed Devices: XXX
- New Enrolments This Week: XX
- Compliance Rate: XX%
- Platform SSO Registered: XX%

Highlights:
- [Key achievements]
- [Issues resolved]

Action Items:
- [Items requiring attention]
```

**Monthly Report:**
```
Subject: Platform SSO & Security Monthly Review

Enrolment:
- New devices: XX
- Success rate: XX%
- Average setup time: XX minutes

Platform SSO:
- Registration rate: XX%
- Authentication success: XX%
- Secure Enclave adoption: XX%

Security:
- Threats blocked: XX
- Compliant devices: XX%
- FileVault enabled: XX%

Recommendations:
- [Strategic recommendations]
```

---

## Appendix: Reference Information

### Useful Commands

```bash
# Platform SSO Commands
/usr/bin/app-sso platform -s                    # Show Platform SSO status
/usr/bin/app-sso platform -u                    # Unregister Platform SSO
/usr/bin/app-sso platform -L                    # List configurations

# Jamf Commands
sudo /usr/local/bin/jamf recon                  # Update inventory
sudo /usr/local/bin/jamf policy                 # Run all policies
sudo /usr/local/bin/jamf policy -event X        # Run specific event
sudo /usr/local/bin/jamf manage                 # Re-enrol device

# Profile Commands
profiles show                                    # List all profiles
profiles show -type configuration                # Show configuration profiles
profiles remove -identifier com.x.profile        # Remove specific profile
profiles renew -type enrollment                  # Renew MDM profile

# Certificate Commands
security find-certificate -a -p                  # List certificates
security find-certificate -c "PSSOe"             # Find Platform SSO cert

# FileVault Commands
sudo fdesetup status                             # Check FileVault status
sudo fdesetup list                               # List enabled users

# Jamf Protect Commands
/usr/local/bin/protectctl status                 # Check status
/usr/local/bin/protectctl threat list            # List threats
/usr/local/bin/protectctl logs --export          # Export logs
```

### Support Resources

**Apple Resources:**
- [Apple Platform Deployment](https://support.apple.com/guide/deployment)
- [Apple Business Manager User Guide](https://support.apple.com/guide/apple-business-manager)
- [macOS Security Compliance Project](https://github.com/usnistgov/macos_security)

**Jamf Resources:**
- [Jamf Pro Documentation](https://learn.jamf.com/)
- [Jamf Nation Community](https://community.jamf.com/)
- [Jamf Connect Documentation](https://learn.jamf.com/bundle/jamf-connect-documentation)
- [Jamf Protect Documentation](https://learn.jamf.com/bundle/jamf-protect-documentation)

**Microsoft Resources:**
- [Microsoft Entra ID Documentation](https://learn.microsoft.com/en-us/entra/)
- [Platform SSO for Microsoft Entra](https://learn.microsoft.com/en-us/entra/identity/devices/macos-psso)
- [Conditional Access Documentation](https://learn.microsoft.com/en-us/entra/identity/conditional-access/)

**Okta Resources:**
- [Okta Device Access](https://help.okta.com/en-us/content/topics/device-access/device-access.htm)
- [Okta macOS Platform SSO](https://help.okta.com/en-us/content/topics/device-access/macos-psso.htm)

### Glossary

| Term | Definition |
|------|------------|
| **ADE** | Automated Device Enrolment (formerly DEP) - Apple's zero-touch enrolment method |
| **ABM** | Apple Business Manager - Apple's device and content management portal |
| **MDM** | Mobile Device Management - System for remotely managing Apple devices |
| **Platform SSO** | Platform Single Sign-On - macOS framework for cloud identity integration |
| **SSO-E** | Single Sign-On Extension - macOS framework for web/app SSO |
| **Secure Enclave** | Dedicated security coprocessor in Apple devices |
| **T2** | Apple T2 Security Chip - Secure Enclave implementation in Intel Macs |
| **Conditional Access** | Identity provider feature enforcing security requirements |
| **Device Compliance** | Integration between MDM and identity provider for compliance enforcement |
| **PreStage** | Jamf Pro feature defining automated enrolment experience |
| **SCEP** | Simple Certificate Enrolment Protocol - Automated certificate issuance |
| **FileVault** | macOS full-disk encryption |
| **MFA** | Multifactor Authentication |
| **WPJ** | Workplace Join - Microsoft device registration method |
| **UPN** | User Principal Name - User identifier (email format) |

---

## Document Change Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 30 Oct 2025 | David Crosby | Initial document creation |
| 1.1 | 30 Oct 2025 | David Crosby | Updated for macOS 26 Tahoe release, added Simplified Setup section, updated version requirements, noted Intel Mac sunset |

