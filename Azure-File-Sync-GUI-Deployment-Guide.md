# Azure File Sync Deployment Guide (Portal / GUI Method)
### Syncing on-prem share `C:\OnpremShare` with Azure Files

---

## 1. Overview

This guide deploys Azure File Sync (AFS) using the **Azure Portal (GUI)** end-to-end. Commands are only used where the Portal has no equivalent (agent install, registration launch).

### Architecture

```
On-Prem Server (Windows Server 2016+)
   C:\OnpremShare  <-- Server Endpoint
        |
        |  Azure File Sync Agent
        v
   Sync Group  <-- Storage Sync Service (Azure)
        |
        v
   Azure File Share  <-- Cloud Endpoint (Azure Storage Account)
```

---

## 2. Prerequisites

| Requirement | Detail |
|---|---|
| OS | Windows Server 2016, 2019, or 2022, domain-joined recommended |
| Azure subscription | Contributor access minimum |
| Outbound connectivity | HTTPS (443) to Azure endpoints (see §8) |
| .NET Framework | 4.7.2+ on the server |
| Server endpoint path | `C:\OnpremShare` (already exists) |
| Browser | Access to https://portal.azure.com from an admin workstation |

---

## 3. High-Level Steps

1. Create a Storage Account (Portal)
2. Create an Azure File Share inside it (Portal)
3. Create a Storage Sync Service (Portal)
4. Create a Sync Group (Portal)
5. Download & install the Azure File Sync agent on the server (installer GUI)
6. Register the server via the Server Registration wizard (GUI)
7. Add a Cloud Endpoint to the Sync Group (Portal)
8. Add a Server Endpoint for `C:\OnpremShare` (Portal)
9. Validate sync
10. Harden & monitor (Portal)

---

## 4. Step-by-Step (Azure Portal)

### Step 1 — Create a Storage Account

1. Go to **https://portal.azure.com** → search **"Storage accounts"** → **+ Create**
2. **Basics tab**:
   - Subscription: select yours
   - Resource group: **Create new** → e.g. `rg-afs-demo`
   - Storage account name: e.g. `stafsyncdemo001` (must be globally unique, lowercase)
   - Region: choose closest to your on-prem server, e.g. `East US`
   - Performance: **Standard** (or Premium FileStorage for high-IO workloads)
   - Redundancy: **LRS** (or **ZRS/GRS** for higher resiliency — recommended for production)
3. Click **Review** → **Create**
4. Wait for deployment → **Go to resource**

---

### Step 2 — Create the Azure File Share

1. Inside the Storage Account, left menu → **Data storage** → **File shares**
2. Click **+ File share**
3. Name: `onpremshare`
4. Tier: **Transaction Optimized** (default) or **Hot/Cool** depending on access pattern
5. Provisioned/Quota: set a size quota, e.g. `1024 GiB`
6. Click **Create**

---

### Step 3 — Create a Storage Sync Service

1. In the Portal search bar, type **"Azure File Sync"** → select **Azure File Sync** service
2. Click **+ Create**
3. Fill in:
   - Subscription / Resource group: same as above (`rg-afs-demo`)
   - Storage Sync Service name: e.g. `afs-sync-service`
   - Region: **same region as the Storage Account**
4. Click **Review + Create** → **Create**

---

### Step 4 — Create a Sync Group

1. Go to the newly created **Storage Sync Service** resource
2. Left menu → **Sync groups** → **+ Sync group**
3. Sync group name: `onprem-share-syncgroup`
4. Under **Azure file share settings**:
   - Subscription: select yours
   - Storage account: `stafsyncdemo001`
   - Azure file share: select `onpremshare` (or create new from here)
5. Click **Create**

> This step also creates the **Cloud Endpoint** automatically — Step 7 below is only needed if you add the cloud endpoint separately later.

---

### Step 5 — Install the Azure File Sync Agent on the Server

**On the Windows Server hosting `C:\OnpremShare`:**

1. Open a browser → go to **https://aka.ms/AFSAgent**
2. Download the MSI installer matching your OS version (2016/2019/2022)
3. Run the installer → **Next** through the GUI wizard → **Install**

> **CMD/PowerShell needed only if doing a silent/unattended install:**
> ```
> StorageSyncAgent.msi /quiet
> ```

4. At the end of installation, the **Server Registration** wizard launches automatically.

---

### Step 6 — Register the Server (GUI Wizard)

1. In the **Server Registration** wizard that pops up after install:
   - Click **Sign in** → authenticate with your Azure AD account
   - Select **Subscription**: your subscription
   - Select **Resource group**: `rg-afs-demo`
   - Select **Storage Sync Service**: `afs-sync-service`
2. Click **Register**
3. Confirm — the server is now listed under:
   `Storage Sync Service > Registered servers` in the Portal

> If the wizard doesn't auto-launch, open it manually from:
> **Start Menu → Azure File Sync → Server Registration**
> (This just opens the GUI tool — no command needed unless the shortcut is missing, in which case run:)
> ```
> "C:\Program Files\Azure\StorageSyncAgent\ServerRegistration.exe"
> ```

---

### Step 7 — Verify / Add the Cloud Endpoint (if not auto-created)

1. Go to **Storage Sync Service** → **Sync groups** → `onprem-share-syncgroup`
2. If no cloud endpoint is listed:
   - Click **+ Add cloud endpoint**
   - Select the Storage account (`stafsyncdemo001`) and Azure file share (`onpremshare`)
   - Click **Create**

---

### Step 8 — Add the Server Endpoint (`C:\OnpremShare`)

1. Still inside `onprem-share-syncgroup` → click **+ Add server endpoint**
2. Fill in:
   - **Registered server**: select your on-prem server from the dropdown
   - **Path**: `C:\OnpremShare`
   - **Cloud Tiering**: toggle **On** (recommended, saves local disk) or **Off** (full local copy, better for first-time validation)
   - **Volume free space (%)**: e.g. `20` (keeps 20% of the volume free)
   - **Tier files older than (days)**: default `7`, adjust as needed (only shown when tiering is On)
3. Click **Create**

> 💡 Recommendation: For the **first deployment**, set Cloud Tiering **Off**, confirm data integrity after initial sync, then edit the server endpoint later and turn tiering **On**.

---

## 5. Validation (GUI)

1. On the server, add a test file into `C:\OnpremShare` via File Explorer.
2. In the Portal, go to **Storage Account → File shares → onpremshare → Browse** — confirm the file appears (allow a few minutes).
3. Reverse test: upload a file directly into the Azure File Share via **Storage Browser** (left menu of the Storage Account) → confirm it appears in `C:\OnpremShare` on the server.
4. Check sync status: **Storage Sync Service → Sync groups → onprem-share-syncgroup → server endpoint** — status should show **Healthy**.
5. If tiering is On, tiered files show a small cloud icon overlay in File Explorer; double-clicking recalls them transparently from Azure.

---

## 6. Monitoring (GUI)

1. Go to **Storage Sync Service** → left menu → **Diagnose and solve problems** for built-in health checks
2. Left menu → **Monitoring → Diagnostic settings** → **+ Add diagnostic setting** → send logs to a **Log Analytics workspace** for alerting/dashboards
3. Left menu → **Sync groups → server endpoint** shows:
   - Sync status (Healthy / Not healthy)
   - Files synced
   - Last sync time
   - Errors count (click to drill into per-file error report)

---

## 7. Production Hardening (GUI)

| Area | Where in Portal |
|---|---|
| Restrict network access | Storage Account → **Networking** → set to **Selected networks**, add server's public IP or VNet |
| Private Endpoint | Storage Account → **Networking → Private endpoint connections → + Private endpoint** (also create one for the Storage Sync Service resource) |
| Backup / Snapshots | Storage Account → File share `onpremshare` → **Backup** → **Configure backup** (creates/uses a Recovery Services Vault) |
| Redundancy | Only settable at creation; to change, Storage Account → **Redundancy** blade (some tiers require migration) |
| AD DS auth for SMB ACLs | Storage Account → **File shares → Active Directory: Configure** (or use `AzFilesHybrid` PowerShell module — GUI doesn't fully cover this yet, script required) |
| Alerts | Storage Sync Service → **Monitoring → Alerts → + Create alert rule** (e.g. alert on sync errors, low free space) |

> ⚠️ AD DS join for the storage account (for NTFS-permission-aware SMB) does not have a full GUI path — this is one of the few places a script is required:
> ```powershell
> Install-Module -Name AzFilesHybrid -Force
> Join-AzStorageAccountForAuth -ResourceGroupName "rg-afs-demo" `
>   -StorageAccountName "stafsyncdemo001" `
>   -DomainAccountType "ComputerAccount"
> ```

---

## 8. Networking Requirements

Allow outbound **HTTPS (443)** from the server to:

- `*.one.microsoft.com`
- `*.afs.azure.net`
- `*.core.windows.net`
- `login.microsoftonline.com`
- `login.windows.net`
- `management.azure.com`

If a proxy is required, this has no GUI equivalent — set via command:
```
netsh winhttp set proxy proxy-server="http://proxyserver:port"
```

---

## 9. Troubleshooting (GUI-first)

| Symptom | Where to check / fix |
|---|---|
| Server registration fails | Server Registration wizard error message; verify time sync and outbound HTTPS |
| Sync not starting | Portal → Sync group → server endpoint → check **Health** status and error details |
| Files not tiering | Portal → server endpoint → confirm **Cloud Tiering = On** and check **Volume free space %** setting |
| Slow initial sync | Expected for large datasets; check progress under server endpoint → **Cloud tiering status** |
| Access denied on tiered files | Confirm the **Azure File Sync** Windows service is running (Services.msc → "StorageSync Agent") |

If GUI diagnostics aren't enough, one command-line check is useful:
```
Get-WinEvent -LogName "Microsoft-FileSync-Agent/Operational" -MaxEvents 50
```

---

## 10. Summary Checklist

- [ ] Storage Account created (Portal)
- [ ] Azure File Share `onpremshare` created (Portal)
- [ ] Storage Sync Service created (Portal)
- [ ] Sync Group created (Portal)
- [ ] Agent installed on server via GUI installer
- [ ] Server registered via Server Registration wizard
- [ ] Cloud Endpoint present in Sync Group
- [ ] Server Endpoint added for `C:\OnpremShare`
- [ ] Initial sync validated (upload + download test)
- [ ] Cloud tiering reviewed and set appropriately
- [ ] Network security configured (firewall / Private Endpoint)
- [ ] Backup/snapshots configured
- [ ] Diagnostic logging + alerts configured

---

*Replace placeholder names (`rg-afs-demo`, `stafsyncdemo001`, `afs-sync-service`, etc.) with your actual environment values as you go through the Portal.*
