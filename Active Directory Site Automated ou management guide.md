# Active Directory Multi-Site & Automated OU Management Guide

## 1. Environment Architecture & Subnet Matrix

This infrastructure follows a **Hub-and-Spoke** network model under the `domain.local` Active Directory domain. `CHQ` serves as the central hub site hosting multiple subnets, while `PUB` and `DU` function as branch spoke sites.

| Site Name | Subnet Range | Domain Controller | IP Address | Primary Role & Topology |
| :--- | :--- | :--- | :--- | :--- |
| **CHQ-Site** | `192.168.1.0/24`<br>`192.168.3.0/24`<br>`192.168.100.0/24` | `chq1.domain.local` | `192.168.1.41` | Primary DC / Central Hub |
| **PUB-Site** | `10.10.10.0/24` | `pub1.domain.local` | `10.10.10.41` | Additional DC / Branch Spoke |
| **DU-Site** | `10.30.0.0/24` | `du1.domain.local` | `10.30.0.41` | Additional DC / Branch Spoke |

---

## 2. AD Sites & Services Subnet Configuration

# See at Active Directory Sites and Services Implementation Guide

---

## 3. Root Cause Analysis & DNS Troubleshooting

### Solution & Recommended DNS Hierarchy

Set the DNS configuration on clients (or via DHCP Scope Option 006) for each site as follows:

#### PUB Site Clients (`10.10.10.0/24`):
* **Preferred DNS:** `10.10.10.41` (`pub1` - Local DC)
* **Alternate DNS:** `192.168.1.41` (`chq1` - Backup Central DC)

#### DU Site Clients (`10.30.0.0/24`):
* **Preferred DNS:** `10.30.0.41` (`du1` - Local DC)
* **Alternate DNS:** `192.168.1.41` (`chq1` - Backup Central DC)

#### Client-Side Cache Reset Commands
Execute the following on the client PC in an elevated Command Prompt:

```cmd
:: Flush local DNS resolver cache
ipconfig /flushdns

:: Restart Netlogon to trigger AD Site re-evaluation
net stop netlogon
net start netlogon

:: Verify local DC attachment
nltest /dsgetdc:domain.local
```

---

## 4. Automated Subnet-Based OU Assignment

Active Directory Sites and Services manages network traffic and DC authentication automatically, but newly joined computer objects land in the default `CN=Computers` container.

The solution below automatically inspects newly joined computers and moves them into their site-specific Organizational Unit (`CHQ`, `PUB`, or `DU`) based on their network subnet.

### Step 1: Organizational Unit Hierarchy

Create the following structure in **Active Directory Users and Computers** (`dsa.msc`):

```text
domain.local
└── Workstation (OU)
    ├── CHQ (OU)
    ├── PUB (OU)
    └── DU  (OU)
```

Distinguished Names:
* `OU=Workstation,DC=domain,DC=local`
* `OU=CHQ,OU=Workstation,DC=domain,DC=local`
* `OU=PUB,OU=Workstation,DC=domain,DC=local`
* `OU=DU,OU=Workstation,DC=domain,DC=local`

---

### Step 2: Automated PowerShell Script (`AutoMoveOU.ps1`)

Save this script on `chq1` as `C:\Scripts\AutoMoveOU.ps1`:

```powershell
<#
===========================================================================
 Script Name: AutoMoveOU.ps1
 Objective:   Automatically relocate computers from CN=Computers to site-specific
              OUs based on their active IP Subnet.
 Domain:      domain.local
===========================================================================
#>

Import-Module ActiveDirectory -ErrorAction Stop

# Mapping table of IP Subnet prefixes to target OUs
$OUMapping = @{
    "192.168.1."   = "OU=CHQ,OU=Workstation,DC=domain,DC=local"
    "192.168.3."   = "OU=CHQ,OU=Workstation,DC=domain,DC=local"
    "192.168.100." = "OU=CHQ,OU=Workstation,DC=domain,DC=local"
    "10.10.10."    = "OU=PUB,OU=Workstation,DC=domain,DC=local"
    "10.30.0."     = "OU=DU,OU=Workstation,DC=domain,DC=local"
}

# Fetch all computer accounts currently residing in default CN=Computers
$Computers = Get-ADComputer -Filter * -SearchBase "CN=Computers,DC=domain,DC=local"

foreach ($comp in $Computers) {
    try {
        # Resolve client IPv4 address via DNS
        $ipList = [System.Net.Dns]::GetHostAddresses($comp.DNSHostName) | Where-Object { $_.AddressFamily -eq 'InterNetwork' }
        $ip = $ipList[0].IPAddressToString

        if ($ip) {
            foreach ($prefix in $OUMapping.Keys) {
                if ($ip.StartsWith($prefix)) {
                    $targetOU = $OUMapping[$prefix]
                    Move-ADObject -Identity $comp.DistinguishedName -TargetPath $targetOU
                    Write-Host "Successfully moved $($comp.Name) ($ip) to $targetOU" -ForegroundColor Green
                }
            }
        }
    } catch {
        # Skip if computer is unreachable or DNS resolution fails
        Write-Warning "Could not resolve or move computer: $($comp.Name)"
    }
}
```

---

### Step 3: Windows Task Scheduler Setup

1. Open **Task Scheduler** (`taskschd.msc`) on `chq1.domain.local`.
2. Click **Create Task...** and configure:
   * **General:**
     * Name: `Auto-Move-Computers-By-Subnet`
     * User account: `NT AUTHORITY\SYSTEM` (or a Domain Admin account)
     * Select **Run whether user is logged on or not**
     * Check **Run with highest privileges**
   * **Triggers:**
     * New Trigger $\rightarrow$ **At startup** or **On a schedule** (Repeat every 15 minutes).
   * **Actions:**
     * Action: **Start a program**
     * Program/script: `powershell.exe`
     * Add arguments: `-ExecutionPolicy Bypass -File "C:\Scripts\AutoMoveOU.ps1"`

---

## 5. Frequently Asked Questions (FAQ)

### Is it mandatory to run this script on the PDC Emulator?

**No, it is not mandatory.** 

Active Directory operates on a **multi-master replication** model for standard directory objects and organizational units. Moving a computer object (`Move-ADObject`) can be executed on **any writable Domain Controller** in the domain. 

Once executed on `chq1`, Active Directory will automatically replicate the object relocation changes to `pub1`, `du1`, and all other DCs across the site links during the next replication cycle. Running the script on `chq1` is recommended simply because it sits at the central hub of your network.