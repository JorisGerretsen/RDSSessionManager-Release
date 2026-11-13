# RDS Server Configuratie Handleiding (Eenmalig per RDS Host)

Volg onderstaande stappen op elke te beheren Windows Server 2025 RDS Session Host (bijv. `RDS-2025-01`, `RDS-2025-02`, etc.) om beheer mogelijk te maken voor helpdeskmedewerkers zonder administratorrechten.

---

## Stap 1: RDP-Tcp Listener Permissies toekennen (PowerShell)

Open **PowerShell als Administrator** op de RDS-server en voer onderstaand script uit.

> **Tip:** Pas de drie variabelen bovenaan aan naar de namen van uw eigen domein en groepen.

```powershell
# --- CONFIGURATIE (Aanpassen aan uw omgeving) ---
$Domain = "JOUWDOMEIN.local"                  # Uw AD domeinnaam (bijv. CORP of CONTOSO.local)
$OperatorGroup = "APP_RDS_Helpdesk_Operators"   # Groep met beheerbevoegdheden (Message, Disconnect, Logoff, Shadow)
$ViewGroup = "APP_RDS_Helpdesk_View"            # Groep met uitsluitend lees-/kijkrechten

# 1. Haal de RDP-Tcp listener op via Terminal Services WMI
$TSPermissions = Get-WmiObject -Namespace "root\cimv2\TerminalServices" -Class Win32_TSPermissionsSetting -Filter "TerminalName='RDP-Tcp'"

# 2. Operators toevoegen (Level 2 = Volledig sessiebeheer: Bericht, Disconnect, Afmelden, Shadow)
$resOp = $TSPermissions.AddAccount("$Domain\$OperatorGroup", 2)
if ($resOp.ReturnValue -eq 0) {
    Write-Host "[OK] Operators toegevoegd: $Domain\$OperatorGroup" -ForegroundColor Green
} else {
    Write-Warning "[FOUT] Toevoegen Operators mislukt. Code: $($resOp.ReturnValue)"
}

# 3. Viewers toevoegen (Level 1 = Alleen sessies en status inzien)
$resView = $TSPermissions.AddAccount("$Domain\$ViewGroup", 1)
if ($resView.ReturnValue -eq 0) {
    Write-Host "[OK] Viewers toegevoegd: $Domain\$ViewGroup" -ForegroundColor Green
} else {
    Write-Warning "[FOUT] Toevoegen Viewers mislukt. Code: $($resView.ReturnValue)"
}
```

---

## Stap 2: WMI CIMV2 Rechten toekennen (via `wmimgmt.msc`)

1. Druk op de RDS-server op **Win + R**, typ `wmimgmt.msc` en druk op **Enter**.
2. Klik met de rechtermuisknop op **WMI Control (Local)** en kies **Properties** (Eigenschappen).
3. Ga naar het tabblad **Security**.
4. Vouw de map `Root` open en selecteer de submap **`CIMV2`**.
5. Klik onderin op de knop **Security** (Beveiliging).
6. Klik op **Add...** en selecteer uw helpdesk-groepen (bijv. `$OperatorGroup` en `$ViewGroup`).
7. Vink bij de machtigingen voor de groep de volgende drie opties aan:
   - [x] **Execute Methods** (Methoden uitvoeren)
   - [x] **Enable Account** (Account inschakelen)
   - [x] **Remote Enable** (Op afstand inschakelen)
8. Klik op de knop **Advanced** (Geavanceerd), selecteer de groep en klik op **Edit** (Bewerken):
   - Stel **Applies to** (Van toepassing op) in op: **This namespace and subnamespaces**.
9. Bevestig met **OK** om op te slaan.

---

## Stap 3: Firewall Openzetten (PowerShell)

Voer dit uit in PowerShell op de RDS-server zodat de beheerpoorten bereikbaar zijn:

```powershell
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
Enable-NetFirewallRule -DisplayGroup "Remote Desktop Management"
Enable-NetFirewallRule -DisplayGroup "Windows Management Instrumentation (WMI)"
```

---

## Rollback (Optioneel)

Om de RDP-Tcp permissies op de RDS-server weer te herstellen naar standaard:

```powershell
$TS = Get-WmiObject -Namespace "root\cimv2\TerminalServices" -Class Win32_TSPermissionsSetting -Filter "TerminalName='RDP-Tcp'"
$TS.RestoreDefaults()
```
