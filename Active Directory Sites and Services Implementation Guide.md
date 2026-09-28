# Active Directory Sites and Services Implementation Guide

## 1. Initial Scenario Details Provided

* **Site 1 (CHQ - Central HQ):**
  * Subnet: `192.168.1.0/24`
  * Domain Controller: `chq1.domain.local`
  * IP Address: `192.168.1.41`

* **Site 2 (PUB):**
  * Subnet: `10.10.10.0/24`
  * Additional DC (ADC): `pub1.domain.local`
  * IP Address: `10.10.10.41`

* **Site 3 (DU):**
  * Subnet: `10.30.0.0/24`
  * Additional DC (ADC): `du1.domain.local`
  * IP Address: `10.30.0.41`

---

### **Assistant Response:**

#### Topology & Mapping Summary

| Site Name | Subnet | Domain Controller | IP Address | Topology Role |
| :--- | :--- | :--- | :--- | :--- |
| **CHQ-Site** | `192.168.1.0/24` | `chq1.domain.local` | `192.168.1.41` | Primary DC / Central Hub |
| **PUB-Site** | `10.10.10.0/24` | `pub1.domain.local` | `10.10.10.41` | Additional DC / Spoke |
| **DU-Site** | `10.30.0.0/24` | `du1.domain.local` | `10.30.0.41` | Additional DC / Spoke |

---

#### Method 1: GUI Implementation Steps (`dssite.msc`)

1. **Create Sites:**
   * Open `dssite.msc`.
   * Create `CHQ-Site`, `PUB-Site`, and `DU-Site`.

2. **Create & Link Subnets:**
   * `192.168.1.0/24` $\rightarrow$ assign to `CHQ-Site`
   * `10.10.10.0/24` $\rightarrow$ assign to `PUB-Site`
   * `10.30.0.0/24` $\rightarrow$ assign to `DU-Site`

3. **Move Servers:**
   * Move `CHQ1` into `CHQ-Site`
   * Move `PUB1` into `PUB-Site`
   * Move `DU1` into `DU-Site`

4. **Configure Site Links:**
   * Create IP Site Links: `LINK-CHQ-PUB` (`CHQ-Site` + `PUB-Site`) and `LINK-CHQ-DU` (`CHQ-Site` + `DU-Site`).
   * Delete or disable `DEFAULTIPSITELINK`.

---

#### Method 2: PowerShell Automated Commands

```powershell
Import-Module ActiveDirectory

# 1. Create AD Sites
New-ADReplicationSite -Name "CHQ-Site"
New-ADReplicationSite -Name "PUB-Site"
New-ADReplicationSite -Name "DU-Site"

# 2. Create Subnets and bind to Sites
New-ADReplicationSubnet -Name "192.168.1.0/24" -Site "CHQ-Site"
New-ADReplicationSubnet -Name "10.10.10.0/24" -Site "PUB-Site"
New-ADReplicationSubnet -Name "10.30.0.0/24" -Site "DU-Site"

# 3. Move DCs to their respective Sites
Move-ADDirectoryServer -Identity "CHQ1" -TargetServerSite "CHQ-Site"
Move-ADDirectoryServer -Identity "PUB1" -TargetServerSite "PUB-Site"
Move-ADDirectoryServer -Identity "DU1" -TargetServerSite "DU-Site"

# 4. Create Hub-and-Spoke Site Links
New-ADReplicationSiteLink -Name "LINK-CHQ-PUB" -SitesIncluded "CHQ-Site","PUB-Site" -Cost 100 -ReplicationFrequencyInMinutes 15
New-ADReplicationSiteLink -Name "LINK-CHQ-DU" -SitesIncluded "CHQ-Site","DU-Site" -Cost 100 -ReplicationFrequencyInMinutes 15

# 5. Remove default Site Link
Remove-ADReplicationSiteLink -Identity "DEFAULTIPSITELINK" -Confirm:$false
```

---

#### Verification Commands

```cmd
repadmin /showrepl
repadmin /summary
dcdiag /test:dns
```

---
