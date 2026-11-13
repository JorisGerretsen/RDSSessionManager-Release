# RDS Session Manager

A modern, secure Windows WPF desktop application (.NET 10 LTS) for monitoring and managing Remote Desktop user sessions across multiple Windows Server 2025 / 2022 / 2019 RDS Session Host servers.

Designed specifically for **servicedesk and support engineers** who are **not Domain Admins** and **not local Administrators** on the managed RDS Session Hosts.

---

## 1. Key Features

- **Multi-Server Monitoring**: Centralized real-time overview of all active, disconnected, and idle sessions across configured RDS servers (`RDS-2025-01`, `RDS-2025-02`, etc.).
- **Clientless Architecture (Variant A)**: Direct communication via native Windows Terminal Services RPC APIs (`wtsapi32.dll` over RPC/SMB) and Kerberos authentication. **Zero extra agents or services required** on session hosts.
- **Least-Privilege Session Management**:
  - **Send Message**: Instant pop-up message delivered to the user's remote session (`WTSSendMessageW`).
  - **Disconnect Session**: Gracefully disconnect user sessions without data loss (`WTSDisconnectSession`).
  - **Sign Out User (Logoff)**: Terminate sessions with explicit confirmation and safety warnings (`WTSLogoffSession`).
  - **Remote Shadowing (View / Control)**: Integrated with `mstsc.exe /shadow` respecting enterprise Group Policies and Windows user consent dialogs.
- **Multilingual Support (EN / DE / NL)**:
  - English is the default application language.
  - Runtime dynamic switching between **English**, **German** (Deutsch), and **Dutch** (Nederlands) via Settings without restarting.
- **Enterprise RBAC & Security**:
  - Evaluates Windows Kerberos token groups of the signed-in user against configured Active Directory groups (`Viewer`, `Operator`, `Administrator`).
  - Disables unauthorized buttons and enforces security verification inside `SecureRdsSessionService` decorator.
- **Audit Logging**: Every operator action (timestamp, user identity, action, server, session ID, success/failure) is written to a tamper-evident `.jsonl` audit log (`%ProgramData%\RdsSessionManager\AuditLogs\`).
- **Dynamic Server Management**: Add, modify, test, enable/disable RDS host connections on the fly. Settings persist to `%ProgramData%\RdsSessionManager\config.json`.

---

## 2. Architecture & Solution Structure

```
rdssessionManager/
├── src/
│   ├── RdsSessionManager.Core/          # Domain entities, models, enums, interfaces
│   ├── RdsSessionManager.Security/      # Active Directory token evaluation & RBAC authorization
│   ├── RdsSessionManager.Infrastructure/# Native wtsapi32 RPC, audit logger, JSON configuration
│   └── RdsSessionManager.App/           # WPF GUI (.NET 10), MVVM, Localization, Windows
├── tests/
│   └── RdsSessionManager.Tests/         # Unit and integration test suite (xUnit, Moq, FluentAssertions)
└── README.md
```

---

## 3. Requirements & Build Instructions

### Prerequisites
- Windows 10 (Version 1809 or newer) or Windows 11 / Windows Server 2022/2025.
- [.NET 10.0 SDK (LTS)](https://dotnet.microsoft.com/download/dotnet/10.0).

### Build Solution
```powershell
dotnet build -c Release
```

### Run Unit Tests
```powershell
dotnet test
```

### Publish Self-Contained Deployment
To build a standalone package for the helpdesk workstation:
```powershell
dotnet publish src/RdsSessionManager.App/RdsSessionManager.App.csproj -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -o ./publish
```

---

## 4. RDS Server Configuration & Least-Privilege Setup (Run once per RDS Host)

To allow helpdesk staff to manage sessions **without Domain Admin or local Administrator privileges**, permissions must be granted on the **`RDP-Tcp` listener** and within **WMI CIMV2**.

### A. PowerShell Script: Grant RDP-Tcp Listener Permissions
Execute this script as Administrator on each RDS Session Host:

```powershell
<#
    .SYNOPSIS
        Grants least-privilege permissions to helpdesk groups on the RDP-Tcp listener.
#>
$Domain = "YOURDOMAIN.local"                  # Your Active Directory domain FQDN or NetBIOS name
$OperatorGroup = "APP_RDS_Helpdesk_Operators" # Group with management rights (Message, Disconnect, Logoff, Shadow)
$ViewGroup = "APP_RDS_Helpdesk_View"          # Group with view-only rights

Write-Host "Configuring RDP-Tcp listener on $env:COMPUTERNAME..." -ForegroundColor Cyan

# 1. Retrieve the RDP-Tcp listener via Terminal Services WMI
$TSPermissions = Get-WmiObject -Namespace "root\cimv2\TerminalServices" -Class Win32_TSPermissionsSetting -Filter "TerminalName='RDP-Tcp'"

if (-not $TSPermissions) {
    Write-Error "Could not find Win32_TSPermissionsSetting for RDP-Tcp on this server."
    return
}

# 2. Add Operators (Level 2 = WTS Session Full Control: Message, Disconnect, Logoff, Shadow)
$resOperator = $TSPermissions.AddAccount("$Domain\$OperatorGroup", 2)
if ($resOperator.ReturnValue -eq 0) {
    Write-Host "  [OK] Successfully granted permissions to: $Domain\$OperatorGroup (Level 2 - Full Control)" -ForegroundColor Green
} else {
    Write-Warning "  [ERROR] Failed to grant permissions to $OperatorGroup. ReturnCode: $($resOperator.ReturnValue)"
}

# 3. Add Viewers (Level 1 = WTS_USER: View sessions and query information)
$resView = $TSPermissions.AddAccount("$Domain\$ViewGroup", 1)
if ($resView.ReturnValue -eq 0) {
    Write-Host "  [OK] Successfully granted permissions to: $Domain\$ViewGroup (Level 1 - View Only)" -ForegroundColor Green
} else {
    Write-Warning "  [ERROR] Failed to grant permissions to $ViewGroup. ReturnCode: $($resView.ReturnValue)"
}

Write-Host "`nVerify configured accounts on RDP-Tcp listener:" -ForegroundColor Cyan
Get-WmiObject -Namespace "root\cimv2\TerminalServices" -Class Win32_TSAccount |
    Where-Object { $_.TerminalName -eq "RDP-Tcp" -and ($_.AccountName -match $OperatorGroup -or $_.AccountName -match $ViewGroup) } |
    Select-Object AccountName, TerminalName, PermissionsAllowed
```

---

### B. WMI CIMV2 Permissions (via `wmimgmt.msc`)
To allow remote querying of server connection status and session information:

1. Open the **Run** dialog (`Win + R`), type `wmimgmt.msc`, and press **Enter**.
2. Right-click **WMI Control (Local)** and select **Properties**.
3. Go to the **Security** tab.
4. Expand `Root` and select **`CIMV2`**.
5. Click the **Security** button at the bottom.
6. Click **Add...** and select your groups (e.g. `$Domain\$OperatorGroup` and `$Domain\$ViewGroup`).
7. Check the following three permissions:
   - ✅ **Execute Methods**
   - ✅ **Enable Account**
   - ✅ **Remote Enable** *(Crucial for remote access!)*
8. Click **Advanced**, select the group, and click **Edit**:
   - Ensure **Applies to** is set to: `This namespace and subnamespaces`.
9. Click **OK** to apply.

---

### C. Firewall Rules on RDS Session Hosts
Ensure inbound Windows Defender Firewall rules are enabled:
```powershell
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
Enable-NetFirewallRule -DisplayGroup "Remote Desktop Management"
Enable-NetFirewallRule -DisplayGroup "Windows Management Instrumentation (WMI)"
```

---

## 5. Rollback Procedure
To revert the RDS Session Host listener back to factory defaults:

```powershell
$TSPermissions = Get-WmiObject -Namespace "root\cimv2\TerminalServices" -Class Win32_TSPermissionsSetting -Filter "TerminalName='RDP-Tcp'"
$TSPermissions.RestoreDefaults()
```

---

## 6. License & Credits

Developed by **Joris Gerretsen**.  
Copyright © 2026 Joris Gerretsen. All rights reserved.
