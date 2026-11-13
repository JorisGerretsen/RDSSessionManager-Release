# RDS Session Host Configuration Guide (One-Time Setup per Host)

Follow these steps on each Windows Server 2025 / 2022 / 2019 RDS Session Host (e.g. `RDS-2025-01`, `RDS-2025-02`) to enable remote management for helpdesk staff without local administrator privileges.

---

## Step 1: Assign RDP-Tcp Listener Permissions (PowerShell)

Open **PowerShell as Administrator** on the RDS server and run the script below.

> **Tip:** Adjust the three configuration variables at the top to match your Active Directory domain and group names.

```powershell
# --- CONFIGURATION (Adjust to your environment) ---
$Domain = "YOURDOMAIN.local"                  # Your AD domain name (e.g. CORP or CONTOSO.local)
$OperatorGroup = "APP_RDS_Helpdesk_Operators" # Group with management rights (Message, Disconnect, Logoff, Shadow)
$ViewGroup = "APP_RDS_Helpdesk_View"          # Group with view-only rights

# 1. Retrieve the RDP-Tcp listener via Terminal Services WMI
$TSPermissions = Get-WmiObject -Namespace "root\cimv2\TerminalServices" -Class Win32_TSPermissionsSetting -Filter "TerminalName='RDP-Tcp'"

# 2. Add Operators (Level 2 = Full session control: Message, Disconnect, Logoff, Shadow)
$resOp = $TSPermissions.AddAccount("$Domain\$OperatorGroup", 2)
if ($resOp.ReturnValue -eq 0) {
    Write-Host "[OK] Operators added: $Domain\$OperatorGroup" -ForegroundColor Green
} else {
    Write-Warning "[ERROR] Failed to add Operators. Code: $($resOp.ReturnValue)"
}

# 3. Add Viewers (Level 1 = View sessions and status only)
$resView = $TSPermissions.AddAccount("$Domain\$ViewGroup", 1)
if ($resView.ReturnValue -eq 0) {
    Write-Host "[OK] Viewers added: $Domain\$ViewGroup" -ForegroundColor Green
} else {
    Write-Warning "[ERROR] Failed to add Viewers. Code: $($resView.ReturnValue)"
}
```

---

## Step 2: Grant WMI CIMV2 Permissions (via `wmimgmt.msc`)

1. Press **Win + R** on the RDS server, type `wmimgmt.msc`, and press **Enter**.
2. Right-click **WMI Control (Local)** and select **Properties**.
3. Go to the **Security** tab.
4. Expand `Root` and select the **`CIMV2`** subfolder.
5. Click the **Security** button at the bottom.
6. Click **Add...** and select your helpdesk groups (e.g. `$Domain\$OperatorGroup` and `$Domain\$ViewGroup`).
7. Check the following three permissions:
   - [x] **Execute Methods**
   - [x] **Enable Account**
   - [x] **Remote Enable** *(Crucial for remote queries)*
8. Click **Advanced**, select the group, and click **Edit**:
   - Set **Applies to** to: **This namespace and subnamespaces**.
9. Click **OK** to save.

---

## Step 3: Open Firewall Rules (PowerShell)

Run the following command as Administrator to open the necessary Windows Defender Firewall rules:

```powershell
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
Enable-NetFirewallRule -DisplayGroup "Remote Desktop Management"
Enable-NetFirewallRule -DisplayGroup "Windows Management Instrumentation (WMI)"
```

---

## Step 4: Verification

Test the connection from the RDS Session Manager application:
1. Open **Settings** -> **Managed Servers**.
2. Click **Test Connection & Permissions**.
3. Status should show: **Connected successfully (Online)**.
