# Storage Spaces Direct (S2D) + Hyper-V — Implementation Guide

> **Scope:** Build a hyper-converged Hyper-V failover cluster on Windows Server using Storage Spaces Direct.
> **Method:** Every step shows the **GUI** way and the **PowerShell** way.
> **Version note:** Written for Windows Server 2022 / 2025 Datacenter. Commands are the same on 2019 unless noted.

---

## 0. Important note about the source video

The YouTube link could not be opened from my side (the fetch was blocked), so this guide is **not a transcript-based analysis** of that video. Instead it follows the standard, Microsoft-documented S2D deployment flow, which is what virtually every S2D + Hyper-V tutorial covers. The step order in section 1 should line up with the video. If you paste the video's chapter titles or transcript, I can map each section to it and add anything that is missing.

---

## 1. Main Steps (Overview)

| # | Phase | Goal | GUI | PowerShell |
|---|-------|------|:---:|:----------:|
| 1 | **Plan & check requirements** | Hardware, licensing, network, IP plan | ✔ | – |
| 2 | **Prepare servers (OS + updates)** | Same OS, drivers, firmware, domain join | ✔ | ✔ |
| 3 | **Install roles & features** | Hyper-V, Failover Clustering, File Services | ✔ | ✔ |
| 4 | **Configure networking** | Management + storage (SMB/RDMA) networks, SET switch | ✔ | ✔ |
| 5 | **Validate the cluster** | `Test-Cluster` must pass | ✔ | ✔ |
| 6 | **Create the failover cluster + witness** | Cluster object, quorum | ✔ | ✔ |
| 7 | **Enable Storage Spaces Direct** | Build pool + cache automatically | ⚠ (WAC) | ✔ |
| 8 | **Create volumes (CSV)** | Resilient, shared storage for VMs | ✔ (WAC/FCM) | ✔ |
| 9 | **Configure Hyper-V & create HA VMs** | VMs stored on CSV, live migration | ✔ | ✔ |
| 10 | **Verify, monitor, maintain** | Health checks, patching, disk replacement | ✔ | ✔ |

**Flow diagram**

```
Plan → Prepare OS → Roles → Network → Validate → Create Cluster
                                                      │
                          Create HA VMs ← Create Volumes ← Enable S2D
                                │
                        Monitor / Patch / Maintain
```

---

## 2. Step 1 — Plan and Check Requirements

### 2.1 What S2D is (in simple words)

S2D takes the **local disks inside each server** and combines them into **one shared storage pool**. Data is copied across servers, so if a disk or even a whole server fails, VMs keep running. In a **hyper-converged** design, the same servers run both **Hyper-V (compute)** and **S2D (storage)**.

### 2.2 Requirements

| Item | Requirement |
|------|-------------|
| **OS edition** | Windows Server **Datacenter** (Standard does not support S2D in classic Windows Server editions; Hyper-V Server free edition does not either) |
| **Servers** | 2 to 16 servers, identical model/config recommended |
| **Install type** | Server Core (recommended) or Desktop Experience |
| **Drives** | Local SATA, SAS, NVMe, or persistent memory, each disk attached to **one** server only. **No RAID controllers** — use HBA / pass-through mode |
| **Minimum disks** | At least 2 cache + 4 capacity drives per server (for HDD+SSD hybrid); all-flash designs can skip the cache tier |
| **Memory** | Enough for the OS + VMs, plus about **4 GB RAM per 1 TB of cache drive capacity** for S2D metadata |
| **Network** | Minimum 10 GbE between servers; **RDMA (iWARP or RoCEv2)** strongly recommended; 2 or more ports |
| **Boot disk** | Any supported boot device (SATADOM, mirrored SSD, etc.) — not part of the pool |
| **Domain** | All nodes in the same Active Directory domain |
| **Witness** | File share witness or Cloud witness (needed for 2-node clusters especially) |
| **Firmware** | Same BIOS, HBA, NIC, and drive firmware on every node |

> ✅ Use hardware that is listed in the Windows Server Catalog (Software-Defined Data Center / "Azure Local" validated nodes) for production support.
> ✅ Use the Microsoft **S2D Calculator** to plan capacity and resiliency before buying hardware.

### 2.3 Resiliency choices (very important decision)

| Servers | Default resiliency | Survives | Storage efficiency |
|---------|--------------------|----------|--------------------|
| 2 | Two-way mirror | 1 failure | 50% |
| 3 | Three-way mirror | 2 simultaneous failures | 33% |
| 4+ | Three-way mirror, or Mirror-accelerated parity | 2 failures | 33% – ~50%+ |
| 2 (special) | Nested resiliency (2019+) | 2 failures (one server + one drive) | 25% – 40% |

### 2.4 Example planning table (replace with your values)

| Item | Value |
|------|-------|
| Domain | `corp.local` |
| Node names | `HV-N1`, `HV-N2`, `HV-N3` |
| Cluster name | `HV-CL01` (IP `10.10.10.50`) |
| Management network | `10.10.10.0/24`, VLAN 10 |
| Storage network 1 (SMB1) | `10.10.71.0/24`, VLAN 711 |
| Storage network 2 (SMB2) | `10.10.72.0/24`, VLAN 712 |
| Live Migration | Uses the storage (SMB) networks |
| Witness | `\\FS01\HV-CL01-Witness` or Azure Cloud Witness |
| Volume names | `Volume1`, `Volume2`, … |

---

## 3. Step 2 — Prepare the Servers

Do this **on every node**.

### 3.1 Install the OS and update

**GUI**
1. Boot from the Windows Server Datacenter ISO and install (Server Core recommended).
2. Run `sconfig` (Core) or **Server Manager → Local Server** to set computer name, IP, time zone.
3. Install all Windows Updates and the latest vendor drivers/firmware.

**PowerShell**
```powershell
# Rename the server
Rename-Computer -NewName "HV-N1" -Restart

# After reboot: set time zone and check OS edition
Set-TimeZone -Id "Bangladesh Standard Time"
(Get-ComputerInfo).WindowsProductName      # Must say Datacenter
```

### 3.2 Join the domain

**GUI:** Server Manager → Local Server → Computer name → *Change…* → Domain → enter `corp.local` and credentials → restart.

**PowerShell**
```powershell
Add-Computer -DomainName "corp.local" -Credential (Get-Credential) -Restart
```

> Tip: Put all cluster nodes in the same OU and use a domain account that has permission to create computer objects (needed for the cluster name object).

### 3.3 Make sure drives are clean and eligible

S2D only claims **empty, unpartitioned, unformatted** drives.

```powershell
# List physical disks and their eligibility
Get-PhysicalDisk | Sort-Object FriendlyName |
  Format-Table FriendlyName, MediaType, BusType, Size, CanPool, CannotPoolReason, HealthStatus
```

If a disk has old data and you are **sure** it can be erased:

```powershell
# ⚠ DESTRUCTIVE — erases every non-boot, non-system disk. Run only on new cluster nodes.
Update-StorageProviderCache
Get-StoragePool | Where-Object IsPrimordial -eq $false | Set-StoragePool -IsReadOnly:$false -ErrorAction SilentlyContinue
Get-StoragePool | Where-Object IsPrimordial -eq $false | Get-VirtualDisk | Remove-VirtualDisk -Confirm:$false -ErrorAction SilentlyContinue
Get-StoragePool | Where-Object IsPrimordial -eq $false | Remove-StoragePool -Confirm:$false -ErrorAction SilentlyContinue
Get-PhysicalDisk | Reset-PhysicalDisk -ErrorAction SilentlyContinue
Get-Disk | Where-Object { $_.Number -ne $null -and $_.IsBoot -ne $true -and $_.IsSystem -ne $true -and $_.PartitionStyle -ne 'RAW' } | ForEach-Object {
    $_ | Set-Disk -IsOffline:$false
    $_ | Set-Disk -IsReadOnly:$false
    $_ | Clear-Disk -RemoveData -RemoveOEM -Confirm:$false
    $_ | Set-Disk -IsReadOnly:$true
    $_ | Set-Disk -IsOffline:$true
}
Get-Disk | Where-Object { $_.Number -ne $null -and $_.IsBoot -ne $true -and $_.IsSystem -ne $true -and $_.PartitionStyle -eq 'RAW' } | ForEach-Object {
    $_ | Set-Disk -IsOffline:$false
    $_ | Initialize-Disk -PartitionStyle GPT
    $_ | Clear-Disk -RemoveData -RemoveOEM -Confirm:$false
    $_ | Set-Disk -IsOffline:$true
}
```

---

## 4. Step 3 — Install Roles and Features

| Feature | Why |
|---------|-----|
| Hyper-V | Run virtual machines |
| Failover-Clustering | Cluster engine |
| File-Services | Required by S2D / SMB |
| RSAT-Clustering-PowerShell | Cluster cmdlets |
| Data-Center-Bridging | Needed for RoCE RDMA (lossless network) |
| Storage-Replica | Optional, for DR / stretch clusters |

**GUI**
1. **Server Manager → Manage → Add Roles and Features**.
2. Role-based installation → select the server → tick **Hyper-V** and **File and Storage Services**.
3. On the Features page tick **Failover Clustering**, **Data Center Bridging**, and **Storage Replica** (optional).
4. Finish and restart.

**PowerShell** (run on all nodes; `-Restart` reboots automatically)
```powershell
Install-WindowsFeature -Name Hyper-V, Failover-Clustering, File-Services, `
    Data-Center-Bridging, RSAT-Clustering-PowerShell, Hyper-V-PowerShell `
    -IncludeManagementTools -Restart
```

**Run on all nodes at once from one admin PC**
```powershell
$Nodes = "HV-N1","HV-N2","HV-N3"
Invoke-Command -ComputerName $Nodes -ScriptBlock {
    Install-WindowsFeature -Name Hyper-V, Failover-Clustering, File-Services, Data-Center-Bridging, RSAT-Clustering-PowerShell, Hyper-V-PowerShell -IncludeManagementTools
}
Restart-Computer -ComputerName $Nodes -Force
```

---

## 5. Step 4 — Configure Networking

### 5.1 Design (recommended: converged with SET)

```
        ┌──────────── Physical Server ────────────┐
        │  NIC1 (RDMA) ──┐                        │
        │                ├── SET vSwitch ──┬── vNIC "Mgmt"  (VLAN 10)
        │  NIC2 (RDMA) ──┘  (Switch Embedded Team) ├── vNIC "SMB1"  (VLAN 711)
        │                                          └── vNIC "SMB2"  (VLAN 712)
        └──────────────────────────────────────────┘
```

- **SET (Switch Embedded Teaming)** is the supported team type for S2D. Do **not** use LBFO teaming with RDMA.
- **vNICs "SMB1/SMB2"** carry storage traffic and live migration with RDMA.
- Switch ports must be **trunk** ports allowing the VLANs (10, 711, 712).
- For **RoCE** you must also configure **DCB/PFC/ETS** on both servers and switches. For **iWARP** it is not required.

> Windows Server 2025 also has **Network ATC** to automate this. Check Microsoft Learn for current availability and use it if you want a one-command intent. The manual method below always works and is good for learning.

### 5.2 Rename physical adapters (GUI)

**Control Panel → Network and Sharing Center → Change adapter settings** → right-click each NIC → Rename (e.g. `NIC1`, `NIC2`).

### 5.3 Create the SET switch and virtual NICs (PowerShell)

```powershell
# 1. Create the SET switch with two physical NICs
New-VMSwitch -Name "ConvergedSwitch" -NetAdapterName "NIC1","NIC2" `
             -EnableEmbeddedTeaming $true -AllowManagementOS $false

# 2. Add host virtual NICs
Add-VMNetworkAdapter -ManagementOS -Name "Mgmt" -SwitchName "ConvergedSwitch"
Add-VMNetworkAdapter -ManagementOS -Name "SMB1" -SwitchName "ConvergedSwitch"
Add-VMNetworkAdapter -ManagementOS -Name "SMB2" -SwitchName "ConvergedSwitch"

# 3. VLANs
Set-VMNetworkAdapterVlan -ManagementOS -VMNetworkAdapterName "Mgmt" -Access -VlanId 10
Set-VMNetworkAdapterVlan -ManagementOS -VMNetworkAdapterName "SMB1" -Access -VlanId 711
Set-VMNetworkAdapterVlan -ManagementOS -VMNetworkAdapterName "SMB2" -Access -VlanId 712

# 4. IP addresses (change per node: .11, .12, .13)
New-NetIPAddress -InterfaceAlias "vEthernet (Mgmt)" -IPAddress 10.10.10.11 -PrefixLength 24 -DefaultGateway 10.10.10.1
Set-DnsClientServerAddress -InterfaceAlias "vEthernet (Mgmt)" -ServerAddresses 10.10.10.5
New-NetIPAddress -InterfaceAlias "vEthernet (SMB1)" -IPAddress 10.10.71.11 -PrefixLength 24
New-NetIPAddress -InterfaceAlias "vEthernet (SMB2)" -IPAddress 10.10.72.11 -PrefixLength 24

# 5. Do NOT register storage NICs in DNS
Set-DnsClient -InterfaceAlias "vEthernet (SMB1)","vEthernet (SMB2)" -RegisterThisConnectionsAddress $false
```

**GUI equivalent:** Hyper-V Manager → **Virtual Switch Manager** → New virtual switch (External). SET team creation and management-OS vNICs have no full GUI; PowerShell is required.

### 5.4 Enable RDMA and map each vNIC to one physical NIC

```powershell
# Enable RDMA on the host vNICs and on physical NICs
Enable-NetAdapterRdma -Name "vEthernet (SMB1)","vEthernet (SMB2)"
Get-NetAdapterRdma

# Pin each SMB vNIC to a different physical NIC (gives path redundancy)
Set-VMNetworkAdapterTeamMapping -ManagementOSVmNetworkAdapterName "SMB1" -PhysicalNetAdapterName "NIC1"
Set-VMNetworkAdapterTeamMapping -ManagementOSVmNetworkAdapterName "SMB2" -PhysicalNetAdapterName "NIC2"
```

### 5.5 DCB / QoS (RoCE only — skip for iWARP)

```powershell
# Tag SMB Direct traffic (port 445) with priority 3
New-NetQosPolicy "SMB" -NetDirectPortMatchCondition 445 -PriorityValue8021Action 3
# Cluster heartbeat traffic with priority 7
New-NetQosPolicy "Cluster" -Cluster -PriorityValue8021Action 7
# Default for everything else
New-NetQosPolicy "Default" -Default -PriorityValue8021Action 0

# Lossless only for priority 3
Enable-NetQosFlowControl -Priority 3
Disable-NetQosFlowControl -Priority 0,1,2,4,5,6,7

# Reserve bandwidth
New-NetQosTrafficClass "SMB"     -Priority 3 -BandwidthPercentage 50 -Algorithm ETS
New-NetQosTrafficClass "Cluster" -Priority 7 -BandwidthPercentage 1  -Algorithm ETS

# Apply to physical NICs
Disable-NetAdapterQos -Name "NIC1","NIC2"      # reset
Enable-NetAdapterQos  -Name "NIC1","NIC2"
```
> The **switch** must have matching PFC/ETS settings. Mismatch is the #1 cause of slow S2D with RoCE.

### 5.6 Verify network

```powershell
# Test connectivity between nodes on storage networks
Test-NetConnection 10.10.71.12
Test-NetConnection 10.10.72.12

# Check SMB multichannel and RDMA after cluster is running
Get-SmbClientNetworkInterface
Get-SmbMultichannelConnection
```

---

## 6. Step 5 — Validate the Cluster

Validation checks hardware, drivers, network, and **S2D disk eligibility**. Microsoft support requires a clean validation report.

**GUI**
1. **Server Manager → Tools → Failover Cluster Manager**.
2. Right-click *Failover Cluster Manager* → **Validate Configuration…**
3. Add all nodes (`HV-N1`, `HV-N2`, `HV-N3`).
4. Select **Run only tests I select** and tick: **Storage Spaces Direct**, **Inventory**, **Network**, **System Configuration**.
5. Run and review the report. Fix every **Error**; review every **Warning**.

**PowerShell**
```powershell
Test-Cluster -Node "HV-N1","HV-N2","HV-N3" `
    -Include "Storage Spaces Direct","Inventory","Network","System Configuration"
```
The HTML report is saved in `C:\Windows\Cluster\Reports\`.

> A warning that says *"disks are not eligible for clustering"* on local drives is normal for S2D. Failing network or firmware-mismatch tests are not.

---

## 7. Step 6 — Create the Failover Cluster and Witness

### 7.1 Create the cluster

**GUI**
1. Failover Cluster Manager → **Create Cluster…**
2. Add the servers, run (or skip) validation.
3. Name: `HV-CL01`. Choose the management network and IP `10.10.10.50`.
4. **Untick** "Add all eligible storage to the cluster" (important for S2D).
5. Finish.

**PowerShell**
```powershell
New-Cluster -Name "HV-CL01" -Node "HV-N1","HV-N2","HV-N3" `
            -StaticAddress 10.10.10.50 -NoStorage
```
`-NoStorage` prevents the cluster from grabbing disks before S2D does.

### 7.2 Configure the witness (quorum)

| Cluster size | Recommendation |
|--------------|----------------|
| 2 nodes | **Witness is required** (file share or cloud) |
| 3+ nodes | Witness strongly recommended (survives more failures) |

**Cloud witness (Azure Storage account)**
```powershell
Set-ClusterQuorum -CloudWitness -AccountName "<storageaccount>" -AccessKey "<key>"
```

**File share witness**
```powershell
Set-ClusterQuorum -FileShareWitness "\\FS01\HV-CL01-Witness"
```

**GUI:** Failover Cluster Manager → right-click cluster → **More Actions → Configure Cluster Quorum Settings…** → *Select the quorum witness* → choose type.

### 7.3 Cluster networks (check roles)

```powershell
Get-ClusterNetwork | Format-Table Name, Role, Address
# Role 3 = cluster + client, Role 1 = cluster only
(Get-ClusterNetwork "Cluster Network 2").Role = 1
```
Management network = Role 3. Storage networks = Role 1.

### 7.4 Optional: Live migration settings

```powershell
# Use SMB transport for live migration (fast, uses RDMA)
Set-VMHost -VirtualMachineMigrationAuthenticationType Kerberos -VirtualMachineMigrationPerformanceOption SMB
Enable-VMMigration
Set-SmbBandwidthLimit -Category LiveMigration -BytesPerSecond 750MB   # requires SMB bandwidth limit feature
```
```powershell
Install-WindowsFeature FS-SMBBW    # if the cmdlet above is not found
```

---

## 8. Step 7 — Enable Storage Spaces Direct

This is the key step. It automatically:
- Creates the **storage pool** from all eligible disks on all nodes.
- Chooses **cache** (fastest drives) and **capacity** drives.
- Creates **storage tiers** (Performance / Capacity).
- Creates a small **ClusterPerformanceHistory** volume.

**PowerShell** (run on any node, or remotely)
```powershell
Enable-ClusterStorageSpacesDirect -CimSession "HV-CL01" -Verbose
```

If the confirmation prompt appears, answer `Y`. This can take several minutes.

**GUI — Windows Admin Center (WAC)**
1. Open WAC → **Add** → *Server clusters* → `HV-CL01`.
2. **Cluster Manager → Storage Spaces Direct** (or the *Drives/Volumes* tool). When S2D is not yet on, WAC shows an **Enable** wizard.

> Failover Cluster Manager shows the pool and disks after enabling but does **not** have a button to enable S2D. Use PowerShell or WAC.

### 8.1 Verify the pool

```powershell
Get-StoragePool S2D*
Get-StorageTier | Format-Table FriendlyName, MediaType, ResiliencySettingName
Get-PhysicalDisk | Sort-Object Usage | Format-Table FriendlyName, MediaType, Usage, Size, HealthStatus
```
Expected `Usage`: `Journal` for cache drives, `Auto-Select` for capacity drives.

### 8.2 Optional: all-flash or lab cache settings

```powershell
# Lab/nested only: disable the cache when all disks are the same type
Enable-ClusterStorageSpacesDirect -CacheState Disabled

# If disks report wrong media type (common in VMs)
Get-PhysicalDisk | Where-Object Size -eq 100GB | Set-PhysicalDisk -MediaType SSD
```

---

## 9. Step 8 — Create Volumes (CSV)

### 9.1 Plan volume size

- Leave **reserve capacity** equal to at least one capacity drive per server (up to 4 drives) so the pool can self-repair after a drive failure.
- Typical rule: create **one volume per node** so ownership is balanced.
- File system: **ReFS** (recommended) with **CSVFS_ReFS**. Use NTFS only for special cases (dedup on older versions).

### 9.2 Create volumes

**PowerShell**
```powershell
# Three-way mirror (3+ nodes), 2 TB
New-Volume -FriendlyName "Volume1" -FileSystem CSVFS_ReFS `
           -StoragePoolFriendlyName "S2D*" -Size 2TB -ResiliencySettingName Mirror

# Two-way mirror for 2-node clusters is picked automatically.

# Mirror-accelerated parity (4+ nodes, hybrid or all-flash)
New-Volume -FriendlyName "Archive1" -FileSystem CSVFS_ReFS -StoragePoolFriendlyName "S2D*" `
           -StorageTierFriendlyNames Performance, Capacity -StorageTierSizes 200GB, 1.8TB
```

**GUI — Windows Admin Center**
1. WAC → cluster → **Volumes → Inventory → Create**.
2. Enter name, resiliency (Two-way / Three-way mirror, Parity), size → **Create**.

**GUI — Failover Cluster Manager** (works too but fewer options)
1. **Storage → Pools** → right-click pool → *New Virtual Disk…* → choose layout/size.
2. Complete the *New Volume wizard* (ReFS) and add as **Cluster Shared Volume**.

### 9.3 Verify

```powershell
Get-VirtualDisk | Format-Table FriendlyName, ResiliencySettingName, Size, HealthStatus, OperationalStatus
Get-ClusterSharedVolume | Format-Table Name, State, @{n="Owner";e={$_.OwnerNode}}
```
Each volume appears on every node as `C:\ClusterStorage\VolumeN`.

---

## 10. Step 9 — Configure Hyper-V and Create Highly Available VMs

### 10.1 Set default paths (every node)

```powershell
$Nodes = "HV-N1","HV-N2","HV-N3"
Invoke-Command -ComputerName $Nodes -ScriptBlock {
    Set-VMHost -VirtualHardDiskPath "C:\ClusterStorage\Volume1\VHDs" `
               -VirtualMachinePath  "C:\ClusterStorage\Volume1\VMs"
}
```

### 10.2 Create a VM on the cluster

**GUI**
1. Failover Cluster Manager → **Roles → Virtual Machines… → New Virtual Machine…**
2. Pick a node, run the *New Virtual Machine Wizard*.
3. In *Specify Name and Location*, tick "Store the virtual machine in a different location" and select `C:\ClusterStorage\Volume1`.
4. Choose Generation 2, memory, a virtual switch (create a VM-facing switch if needed), and the VHDX.
5. The VM is automatically added as a clustered role (HA).

**PowerShell**
```powershell
# Create a VM-facing external switch on all nodes (same name on every node!)
Invoke-Command -ComputerName $Nodes -ScriptBlock {
    New-VMSwitch -Name "VM-Switch" -NetAdapterName "NIC1","NIC2" -EnableEmbeddedTeaming $true -AllowManagementOS $false
}
# Or reuse "ConvergedSwitch" if you want VMs on the same SET team.

New-VM -Name "VM01" -Generation 2 -MemoryStartupBytes 4GB `
       -Path "C:\ClusterStorage\Volume1\VMs" `
       -NewVHDPath "C:\ClusterStorage\Volume1\VHDs\VM01.vhdx" -NewVHDSizeBytes 60GB `
       -SwitchName "ConvergedSwitch"

Set-VMProcessor -VMName "VM01" -Count 2
Add-ClusterVirtualMachineRole -VMName "VM01" -Cluster "HV-CL01"
Start-VM -Name "VM01"
```

### 10.3 Test high availability

```powershell
# Live migration to another node
Move-ClusterVirtualMachineRole -Name "VM01" -Node "HV-N2" -MigrationType Live

# Simulate node failure: power off a node and watch the VM restart on another node
Get-ClusterGroup "VM01" | Format-Table Name, OwnerNode, State
```

### 10.4 VM settings worth setting

```powershell
# Preferred owners / anti-affinity so critical VMs do not share a host
(Get-ClusterGroup "VM01").AntiAffinityClassNames = "SQL-Cluster"

# Auto-start delay and priority
(Get-ClusterGroup "VM01").Priority = 3000       # High
```

---

## 11. Step 10 — Verify, Monitor, and Maintain

### 11.1 Quick health checks (daily/weekly)

```powershell
# One-line summary of storage health
Get-StorageSubSystem Cluster* | Get-StorageHealthReport

# Any current problems?
Get-StorageSubSystem Cluster* | Debug-StorageSubSystem

# Repair / resync jobs in progress
Get-StorageJob

# Physical disks
Get-PhysicalDisk | Where-Object HealthStatus -ne Healthy

# Virtual disks
Get-VirtualDisk | Where-Object { $_.HealthStatus -ne "Healthy" -or $_.OperationalStatus -ne "OK" }

# Nodes
Get-ClusterNode | Format-Table Name, State, StatusInformation
```

**GUI:** WAC **Dashboard** shows alerts, performance (IOPS / latency / throughput), drive and volume health. FCM shows node, role, and CSV status.

### 11.2 Performance history

```powershell
Get-ClusterPerformanceHistory
Get-ClusterNode HV-N1 | Get-ClusterPerf -ClusterNodeSeriesName ClusterNode.Cpu.Usage -TimeFrame LastDay
```

### 11.3 Patching a node safely (rolling update)

Never reboot two nodes at the same time while storage is repairing.

```powershell
# 1. Put the node in maintenance: moves VMs away and pauses the node
Suspend-ClusterNode -Name "HV-N1" -Drain -Wait

# 2. Put storage in maintenance mode for that node
Get-StorageFaultDomain -Type StorageScaleUnit | Where-Object FriendlyName -eq "HV-N1" | Enable-StorageMaintenanceMode

# 3. Patch and reboot the node (Windows Update / CAU / sconfig)

# 4. Bring back
Get-StorageFaultDomain -Type StorageScaleUnit | Where-Object FriendlyName -eq "HV-N1" | Disable-StorageMaintenanceMode
Resume-ClusterNode -Name "HV-N1" -Failback Immediate

# 5. WAIT until repair jobs finish before touching the next node
Get-StorageJob            # must be empty
Get-VirtualDisk           # all Healthy / OK
```

**Easier:** use **Cluster-Aware Updating (CAU)** — Failover Cluster Manager → right-click cluster → **More Actions → Cluster-Aware Updating**, or:
```powershell
Add-CauClusterRole -ClusterName "HV-CL01" -DaysOfWeek Sunday -WeeksOfMonth 2 -EnableFirewallRules -Force
```

### 11.4 Replace a failed drive

```powershell
# 1. Identify the failed disk
Get-PhysicalDisk | Where-Object HealthStatus -ne Healthy

# 2. Retire it and (optionally) blink the LED
$d = Get-PhysicalDisk -SerialNumber "<serial>"
$d | Set-PhysicalDisk -Usage Retired
$d | Enable-PhysicalDiskIndicator

# 3. Remove from pool once repair completes
$pool = Get-StoragePool S2D*
Remove-PhysicalDisk -StoragePool $pool -PhysicalDisks $d

# 4. Insert the new drive; S2D adds it to the pool automatically
Get-PhysicalDisk | Where-Object CanPool -eq $true
Get-StorageJob           # watch the rebalance/repair
```

### 11.5 Add a new node (scale out)

1. Prepare it exactly like the others (sections 3–5).
2. `Test-Cluster` including the new node.
3. `Add-ClusterNode -Name "HV-N4"`.
4. S2D claims its disks automatically; the pool rebalances.
5. Optionally run `Optimize-StoragePool -FriendlyName "S2D*"`.

---

## 12. Troubleshooting Cheat Sheet

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `Enable-ClusterS2D` fails: "no disks eligible" | Disks have partitions / old pool / RAID mode | Clean disks (section 3.3), use HBA mode |
| Validation: "Storage Spaces Direct" test missing | Test not included, or feature missing | Re-run with `-Include "Storage Spaces Direct"` |
| Disk MediaType = `Unspecified` | Virtual / RAID-presented disks | `Set-PhysicalDisk -MediaType SSD/HDD` |
| Slow storage with RoCE | PFC / ETS mismatch on switch | Align DCB config on host and switch |
| RDMA shows `Disabled` | Driver / BIOS / wrong adapter mode | `Enable-NetAdapterRdma`; update NIC firmware |
| Volume shows "Degraded" after reboot | Repair/resync running | `Get-StorageJob`; wait. Do not reboot other nodes |
| Cluster cannot form | DNS / AD permissions / time sync | Check DNS records, create CNO rights, NTP |
| CSV shows "Redirected access" | Network or SMB path problem | Check storage NICs, `Get-SmbMultichannelConnection` |
| Live migration slow | Using TCP not SMB, or no RDMA | Set migration option to SMB, check RDMA |
| 2-node cluster loses quorum | No witness | Add file share or cloud witness |

**Useful logs:** Event Viewer → *Applications and Services Logs → Microsoft → Windows → StorageSpaces-Driver*, *FailoverClustering*, *SMBClient/Connectivity*, *SMBServer*.
```powershell
Get-ClusterLog -Destination C:\Temp -TimeSpan 30
```

---

## 13. Best Practices Checklist

- [ ] Datacenter edition, same OS build and patches on all nodes
- [ ] Identical hardware and firmware on every node
- [ ] HBA pass-through mode (no RAID) for data drives
- [ ] 10 GbE minimum, RDMA preferred, 2 NICs per node
- [ ] SET switch (not LBFO) with dedicated SMB vNICs on separate VLANs
- [ ] Switch DCB/PFC configured (RoCE) and jumbo frames consistent (if used)
- [ ] `Test-Cluster` passed before enabling S2D
- [ ] Witness configured (mandatory for 2 nodes)
- [ ] Reserve capacity left free in the pool (≥ 1 drive per node, up to 4)
- [ ] One CSV volume per node, ReFS, three-way mirror for important VMs
- [ ] Never reboot more than one node while `Get-StorageJob` shows activity
- [ ] Use CAU or the rolling update procedure above
- [ ] Backups (e.g., Veeam, Windows Server Backup, Azure Backup) — S2D resiliency is **not** a backup
- [ ] Monitoring: WAC dashboard, alerts to email/SCOM/Azure Monitor
- [ ] Document IP plan, firmware versions, and recovery steps

---

## Appendix A — Full Deployment Script (Condensed)

> Run from an admin PC or one node. Review variables first.

```powershell
#region Variables
$Nodes       = "HV-N1","HV-N2","HV-N3"
$ClusterName = "HV-CL01"
$ClusterIP   = "10.10.10.50"
$Witness     = "\\FS01\HV-CL01-Witness"
#endregion

# 1. Install features on all nodes
Invoke-Command -ComputerName $Nodes -ScriptBlock {
    Install-WindowsFeature Hyper-V, Failover-Clustering, File-Services, Data-Center-Bridging, `
        RSAT-Clustering-PowerShell, Hyper-V-PowerShell -IncludeManagementTools
}
Restart-Computer -ComputerName $Nodes -Wait -For PowerShell -Force

# 2. (Networking: run section 5 commands on each node)

# 3. Validate
Test-Cluster -Node $Nodes -Include "Storage Spaces Direct","Inventory","Network","System Configuration"

# 4. Create cluster + witness
New-Cluster -Name $ClusterName -Node $Nodes -StaticAddress $ClusterIP -NoStorage
Set-ClusterQuorum -Cluster $ClusterName -FileShareWitness $Witness

# 5. Enable S2D
Enable-ClusterStorageSpacesDirect -CimSession $ClusterName -Confirm:$false -Verbose

# 6. Create one volume per node
$i = 1
foreach ($n in $Nodes) {
    New-Volume -CimSession $ClusterName -FriendlyName "Volume$i" -FileSystem CSVFS_ReFS `
               -StoragePoolFriendlyName "S2D*" -Size 1TB
    $i++
}

# 7. Check
Get-StorageSubSystem -CimSession $ClusterName Cluster* | Get-StorageHealthReport
Get-ClusterSharedVolume -Cluster $ClusterName
```

---

## Appendix B — Testing S2D in a Lab (Nested Virtualization)

Microsoft supports Hyper-V VMs for **evaluation** of S2D (not production). Useful to practice what the video shows.

**Lab layout:** 1 domain controller VM + 2–4 S2D node VMs, each node with 1 vCPU×4 or more, 8+ GB RAM, and at least 4 extra data VHDX disks.

```powershell
# On the physical Hyper-V host
$vm = "S2D-N1"

# Allow nested Hyper-V inside the VM
Set-VMProcessor -VMName $vm -ExposeVirtualizationExtensions $true -Count 4

# Allow VM-in-VM networking
Set-VMNetworkAdapter -VMName $vm -MacAddressSpoofing On

# Disable dynamic memory
Set-VMMemory -VMName $vm -DynamicMemoryEnabled $false -StartupBytes 8GB

# Add 4 data disks (fixed or dynamic) on the SCSI controller
1..4 | ForEach-Object {
    $p = "D:\VMs\$vm-Data$_.vhdx"
    New-VHD -Path $p -SizeBytes 100GB -Dynamic | Out-Null
    Add-VMHardDiskDrive -VMName $vm -Path $p -ControllerType SCSI
}
```

Inside the lab node VMs:
- Skip RDMA, DCB, and QoS (not available in VMs); use a normal virtual NIC with a switch.
- Use `Enable-ClusterStorageSpacesDirect -CacheState Disabled` when all VHDX are the same type.
- Set `MediaType` manually if shown as *Unspecified*.
- Use Windows Server evaluation ISO (180-day) if you have no license.

> If you run the lab on a **laptop with VMware Workstation or Hyper-V**, make sure virtualization (Intel VT-x / AMD-V) is enabled in BIOS. Only one hypervisor should own the virtualization extensions at a time; mixing them may need extra configuration.

---

## Appendix C — Key PowerShell Commands Reference

| Task | Command |
|------|---------|
| List disks | `Get-PhysicalDisk` |
| Validate cluster | `Test-Cluster` |
| Create cluster | `New-Cluster -NoStorage` |
| Set witness | `Set-ClusterQuorum` |
| Enable S2D | `Enable-ClusterStorageSpacesDirect` |
| Pool info | `Get-StoragePool` |
| Create volume | `New-Volume` |
| Volume status | `Get-VirtualDisk` |
| CSV status | `Get-ClusterSharedVolume` |
| Health | `Get-StorageSubSystem Cluster* \| Get-StorageHealthReport` |
| Repair jobs | `Get-StorageJob` |
| Drain node | `Suspend-ClusterNode -Drain` |
| Storage maintenance | `Enable-StorageMaintenanceMode` |
| Resume node | `Resume-ClusterNode -Failback Immediate` |
| Make VM HA | `Add-ClusterVirtualMachineRole` |
| Live migrate | `Move-ClusterVirtualMachineRole` |

---

## Appendix D — References

- Microsoft Learn: *Storage Spaces Direct overview*
- Microsoft Learn: *Storage Spaces Direct hardware requirements*
- Microsoft Learn: *Deploy Storage Spaces Direct on Windows Server*
- Microsoft Learn: *Using Storage Spaces Direct in guest virtual machine clusters* (lab/nested)
- Microsoft Learn: *Plan volumes in Storage Spaces Direct* and *Fault tolerance and storage efficiency*
- Windows OS Hub: *Deploying Storage Spaces Direct (S2D) Cluster on Windows Server* (3-node Windows Server 2025 walk-through)
- Microsoft Ignite sessions on S2D design considerations (Claus Joergensen, Elden Christensen)

*Always check the current Microsoft documentation for the latest supported hardware and OS-specific behaviour before a production rollout.*
