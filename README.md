# Windows 10 to Windows 11 Upgrade with (WUfB) & Windows Autopatch

A comprehensive guide to upgrade Windows 10 to Windows 11 devices using Microsoft Intune's Windows Update for Business (WUfB) policies.

---

## Requirements

### License Requirements

Windows Update for Business (WUfB) is available with the following licenses:

| License | Included |
|:--------|:---------|
| **Microsoft 365 Business Premium** |
| **Windows 10/11 Education A3 or A5** | (included in Microsoft 365 A3 or A5) |
| **Windows 10/11 Enterprise E3 or E5** | (included in Microsoft 365 F3, E3, or E5) |

### Infrastructure Requirements

- **Microsoft Entra ID and Intune**
  - Microsoft Entra ID P1 or P2
  - Microsoft Intune (required)
  - Microsoft Entra ID must be the source of authority for all user accounts, **OR**
  - User accounts must be synchronized from on-premises Active Directory using the latest supported version of Microsoft Entra Connect to enable Microsoft Entra hybrid join

- **Policy Delivery**
  - WUfB in Intune is implemented as MDM policy using the Policy CSP
  - Devices must be enrolled with Microsoft Intune **before** registering with Windows Autopatch
  - Intune must be set as the Mobile Device Management (MDM) authority

---

## Device Eligibility

### Supported Device Types

#### Entra-joined or Hybrid-joined and MDM-managed
- A device that is joined to Microsoft Entra ID or hybrid-joined **and** enrolled in Intune (MDM-managed) can receive MDM policies from Intune
- These devices can receive Windows Update for Business (WUfB) policies

### Unsupported Device Types

#### Microsoft Entra ID Registered, Non-MDM-managed
- A device that is only Microsoft Entra **registered** (e.g., a personal device with a work account added) **cannot** receive:
  - Intune device configuration
  - Update ring policies
  - Feature update policies

#### Hybrid-joined but Not Enrolled in Intune
- A device that is Microsoft Entra hybrid joined but **not enrolled in Intune** also **cannot** receive:
  - Intune device configuration
  - Update ring policies
  - Feature update policies

### Alternative for Non-MDM Devices

#### Local Active Directory Devices
- **Domain-joined devices:** Use Group Policy–based Windows Update for Business settings
- **Standalone devices:** Use local policies to configure update behavior

---

## Deployment Guide

### Step 1: Create Microsoft Entra Dynamic Device Groups

Create dynamic device groups to target the correct devices for WUfB policies.

#### Group 1: Windows 10 Entra or Hybrid-Join MDM Devices

```
Name: Windows-10-DEV-Entra-Or-Hybrid-Join-MDM-Devices
Description: Windows 10 Entra-Join Or Hybrid-Join Intune-Managed Company Devices (Excludes Entra-Registered Devices)
Dynamic Rule Syntax:
((device.deviceTrustType -eq "AzureAD") or (device.deviceTrustType -eq "ServerAD")) and (device.deviceOSType -eq "Windows") and (device.deviceOSVersion -startsWith "10.0.1") and (device.managementType -eq "MDM")
```

#### Group 2: Windows 10 Entra-Join MDM Devices

```
Name: Windows-10-DEV-Entra-Join-MDM-Devices
Description: Windows 10 Entra-Join Intune-Managed Company Devices (Excludes Entra-Registered Devices)
Dynamic Rule Syntax:
(device.deviceTrustType -eq "AzureAD") and (device.deviceOSType -eq "Windows") and (device.deviceOSVersion -startsWith "10.0.1") and (device.managementType -eq "MDM")
```

#### Group 3: Windows 11 Entra-Join MDM Devices

```
Name: Windows-11-DEV-Entra-Join-MDM-Devices
Description: Windows 11 Entra-Join Intune-Managed Company Devices (Excludes Entra-Registered Devices)
Dynamic Rule Syntax:
(device.deviceTrustType -eq "AzureAD") and (device.deviceOSType -eq "Windows") and (device.deviceOSVersion -startsWith "10.0.2") and (device.managementType -eq "MDM")
```

#### Group 4: Windows Update for Business - Pilot Group

```
Name: WIN - DEV - Windows-Update-For-Business - Pilot
Description: Windows 10 & Windows 11 Entra-Join Intune-Managed Company Devices (Excludes Entra-Registered Devices)
Dynamic Rule Syntax:
(device.deviceTrustType -eq "AzureAD") and (device.deviceOSType -eq "Windows") and ((device.deviceOSVersion -startsWith "10.0.1") or (device.deviceOSVersion -startsWith "10.0.2")) and (device.managementType -eq "MDM")
```

> **Note:** The OS version prefixes:
> - `10.0.1` = Windows 10 (builds 10.0.19041+)
> - `10.0.2` = Windows 11 (builds 10.0.22000+)

---

### Step 2: Enable Endpoint Analytics

Endpoint analytics provides the **Hardware Readiness Report** to identify which devices are ready for Windows 11.

#### Configuration Steps

1. In the **Intune admin center**, navigate to:
   ```
   Reports > Endpoint Analytics > Work from anywhere
   ```

2. Click the **Windows** tab

3. Review device readiness:
   - **Capable** - Ready for Windows 11
   - **Not Capable** - Has blockers (TPM, Secure Boot, CPU, storage)
   - **Upgraded** - Already on Windows 11
   - **Unknown** - Device inactive or not reporting

4. Check the **Readiness reason** column for specific blockers

> **Tip:** Devices marked **Unknown** often mean they've been inactive. Check the last check-in time and focus on active devices.

---

### Step 3: Configure Windows Health Monitoring Device Configuration Policy

This policy enables health monitoring for Endpoint Analytics.

#### Policy Configuration

```
Policy Name: WIN - DC - Health Monitoring - DEV - Endpoint Analytics
Profile type: Windows health monitoring

Configuration settings:
├── Health monitoring: Enable
└── Scope: Endpoint analytics
```

#### Deployment Steps

1. Navigate to **Devices > Configuration profiles > Create profile**
2. Platform: **Windows 10 and later**
3. Profile type: **Windows health monitoring**
4. Configure settings as shown above
5. Assign to your device groups

---

### Step 4: Enable Windows Diagnostic Data

Enable Windows diagnostic data collection for update compliance and analytics.

#### Configuration Steps

1. Navigate to **Tenant administration > Connectors and tokens > Windows data**

2. Configure:
   ```
   Windows data: Enable
   Windows license verification: Enable
   ```

---

### Step 5: Create Windows Update Policies

Create the following WUfB policies to manage updates:

#### Policy 1: Update Ring (Windows 10 to Windows 11)

```
Policy Name: WUfB-Windows-10-To-Windows-11-Update-Ring
Policy Type: Update rings for Windows 10 and later

Purpose: Controls quality update behavior and prepares devices for feature updates
```

#### Policy 2: Feature Update (Windows 11 25H2)

```
Policy Name: WUfB-Windows-11-Feature-Update-25H2
Policy Type: Feature updates for Windows 10 and later

Purpose: Deploys Windows 11, version 25H2 to eligible devices
```

#### Policy 3: Driver Updates

```
Policy Name: WUfB-Windows-11-Driver-Update
Policy Type: Update rings for Windows 10 and later (Driver settings)

Purpose: Controls driver update behavior
```

#### Policy 4: Quality Updates

```
Policy Name: WUfB-Windows-11-Quality-Update
Policy Type: Update rings for Windows 10 and later

Purpose: Controls monthly cumulative update behavior
```

---

### Step 6: Deploy Platform Script

Deploy the **Windows-Updates-Readiness** script to prepare devices for updates.

#### Script Details

```
Script Name: Windows-Updates-Readiness_v1.5.ps1
Script Type: Platform Script
Run as: SYSTEM
64-bit PowerShell: Yes
```

#### What the Script Does

- Removes WSUS/GPO blocks
- Clears update pause states
- Validates hardware eligibility (TPM, Secure Boot, RAM)
- Performs disk cleanup if needed
- Repairs Windows Update components
- Triggers Windows Update scan

#### Deployment Steps

1. Navigate to **Devices > Scripts and remediations > Platform scripts**
2. Click **+ Add > Windows 10 and later**
3. Upload `Windows-Updates-Readiness_v1.5.ps1`
4. Configure:
   - Run using logged on credentials: **No**
   - Enforce script signature check: **No**
   - Run in 64-bit PowerShell: **Yes**
5. Assign to device groups

> **See also:** [Windows-Updates-Readiness Script Documentation](README.md)

---

### Step 7: Verify Microsoft Account Sign-In Assistant Service

The **Microsoft Account Sign-In Assistant (wlidsvc)** service must be enabled for feature updates to install.

#### Verification Steps

**Option 1: Manual Check (on a test device)**

1. Open **Services** (`services.msc`)
2. Locate **Microsoft Account Sign-In Assistant**
3. Verify:
   - **Status:** Running
   - **Startup type:** Manual (or Automatic)

**Option 2: PowerShell Check**

```powershell
Get-Service -Name wlidsvc | Select-Object Name, Status, StartType
```

Expected output:
```
Name     Status  StartType
----     ------  ---------
wlidsvc  Running Manual
```

#### Important

If this service is **disabled** or **blocked**, the Windows 11 25H2 upgrade will **not proceed successfully**.

**Remediation:**
- Ensure no Group Policy or registry setting is disabling this service
- If disabled, set to **Manual** startup type

---

## Monitoring and Reporting

### Available Reports

| Report | Location | Purpose |
|:-------|:---------|:--------|
| **Windows 11 Hardware Readiness** | Endpoint Analytics > Work from anywhere | Identify eligible devices |
| **Windows Feature Update Report** | Reports > Windows updates | Track feature update deployment status |
| **Windows Expedited Update Report** | Reports > Windows updates | Monitor expedited quality updates |
| **Update Compliance** | Reports > Windows updates | Overall update compliance status |

---

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|:------|:------|:---------|
| Device not receiving updates | Not MDM-managed | Verify device is Intune-enrolled |
| Feature update blocked | Hardware not eligible | Check Endpoint Analytics readiness report |
| Update stuck at "Pending restart" | No deadline configured | Configure deadlines in Update Ring policy |
| wlidsvc service disabled | Group Policy or registry block | Enable service and set to Manual startup |

### Useful PowerShell Commands

```powershell
# Check Windows Update service status
Get-Service wuauserv, bits, dosvc | Select-Object Name, Status, StartType

# Check for pending updates
Get-WindowsUpdate

# View Windows Update history
Get-WindowsUpdateLog

# Check device Intune enrollment
dsregcmd /status
```

---

## Additional Resources

- [Windows Update for Business Documentation](https://learn.microsoft.com/windows/deployment/update/waas-manage-updates-wufb)
- [Windows Autopatch Documentation](https://learn.microsoft.com/windows/deployment/windows-autopatch/)
- [Endpoint Analytics Documentation](https://learn.microsoft.com/mem/analytics/)
- [Windows 11 Requirements](https://www.microsoft.com/windows/windows-11-specifications)

---

## License

This documentation is provided as-is for educational and deployment purposes.

---

**Last Updated:** January 2026  
**Applies to:** Windows 10, Windows 11, Microsoft Intune
