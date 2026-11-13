# RDS Session Manager - Downloads & Releases

Welcome to the official download repository for **RDS Session Manager**, a Windows desktop application (.NET 10 LTS) for servicedesk and support engineers to monitor and manage Remote Desktop user sessions across multiple Windows Server RDS Session Hosts.

---

## 🚀 Quick Download

| Version | Release Date | Download Link | Notes |
|:---|:---|:---|:---|
| **v1.1.0 (Latest)** | November 2026 | [⬇️ Download v1.1.0 (ZIP)](https://github.com/JorisGerretsen/RDSSessionManager-Release/releases/download/v1.1.0/RDSSessionManager-v1.1.0.zip) | Multi-language (EN/DE/NL), Clientless WTS RPC, Least-Privilege RBAC |

*(Previous versions will remain available in the [Releases Archive](https://github.com/JorisGerretsen/RDSSessionManager-Release/releases) whenever an update is released).*

---

## 💻 How to Install & Run

1. **Download**: Click the download link above to get `RDSSessionManager-v1.1.0.zip`.
2. **Extract**: Right-click the `.zip` file and select **Extract All...** to a folder on your computer (e.g. `C:\Tools\RdsSessionManager` or your Desktop).
3. **Run**: Double-click **`RdsSessionManager.App.exe`**.
   - *No installation wizard or administrative rights needed on your local workstation.*
   - If Windows Defender SmartScreen shows a prompt, click **More info** -> **Run anyway**.

---

## ⚙️ Prerequisites

- **Workstation OS**: Windows 10 (Version 1809 or newer) or Windows 11.
- **Runtime**: [.NET Desktop Runtime 10.0 (x64)](https://dotnet.microsoft.com/download/dotnet/10.0) installed on the helpdesk PC.
- **Permissions**: Your Windows user account should be a member of the configured Active Directory helpdesk group (`APP_RDS_Helpdesk_Operators` or `APP_RDS_Helpdesk_View`).

---

## 📖 RDS Host Server Setup

For servers to accept connections from helpdesk staff without requiring Domain Admin privileges, perform the one-time listener configuration described in:
👉 [SERVER_SETUP_INSTRUCTIES.md](SERVER_SETUP_INSTRUCTIES.md)

---

## 🔒 Security & Features

- **Clientless**: Directly uses native Windows Terminal Services RPC (`wtsapi32.dll`). No agents required on RDS hosts.
- **Languages**: English (default), German, Dutch (selectable in Settings).
- **Operations**:
  - ✉️ Send pop-up messages to users
  - ⚡ Disconnect idle/disconnected sessions
  - 🛑 Sign out users with safety prompt
  - 👁️ Shadow sessions (view / interactive control with user consent)
- **Audit Logging**: All actions are logged locally in `%ProgramData%\RdsSessionManager\Logs`.

---

© 2026 Joris Gerretsen. All rights reserved.
