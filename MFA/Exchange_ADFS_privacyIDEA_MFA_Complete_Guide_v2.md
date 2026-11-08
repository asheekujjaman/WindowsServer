# Exchange Server 2019 + AD FS + privacyIDEA — OTP MFA for Outlook on the web
## Verified Architecture and Step-by-Step Implementation Guide (v2)

| | |
|---|---|
| **Document version** | 2.0 (supersedes `Exchange_ADFS_privacyIDEA_MFA_Implementation_Guide.md`) |
| **Date** | 8 November 2026 |
| **Lab domain** | `lab.com` |
| **Audience** | Windows / Exchange / Active Directory / Linux administrators |
| **Basis** | The uploaded v1 guide, the full project chat (5–8 Nov 2026 command outputs), and official vendor documentation |
| **Lab result** | Username + password + TOTP works end-to-end on OWA for test user `u2` |

> **সংক্ষিপ্ত সারাংশ (বাংলা):** এই ডকুমেন্টে আগের গাইড যাচাই করা হয়েছে, ভুল/অসম্পূর্ণ অংশ ঠিক করা হয়েছে, আর আসলে যে ক্রমে আপনার lab সফল হয়েছে সেই ক্রমে ১১টি phase সাজানো হয়েছে। প্রতিটি phase-এ GUI ও CLI দুটোই আছে, আর প্রতিটি phase-এর শেষে একটি **Gate** আছে। Gate পাস না করে পরের phase-এ যাবেন না, তাহলে error ধরা সহজ হবে। সবচেয়ে গুরুত্বপূর্ণ তিনটি সতর্কতা: (১) privacyIDEA realm-এর নাম `lab` (`lab.com` নয়); (২) শুধু provider চালু করলে MFA বাধ্যতামূলক হয় না, Relying Party Trust-এ "require MFA" policy দিতে হয়; (৩) এই সেটআপ শুধু OWA ও ECP রক্ষা করে, Outlook desktop/ActiveSync/EWS নয়।

---

## Table of contents

1. Verification report (what was checked and what was corrected)
2. Executive summary, scope and limits
3. Architecture
4. Phase workflow (the wire-work overview)
5. Lab variables (as-built values)
6. Detailed phases 0–10 (each with GUI, CLI, verification gate)
7. Test matrix and evidence
8. Rollback and recovery
9. Troubleshooting (including every real error seen in this lab)
10. Production readiness, licensing and security
11. Appendices
12. References

---

## 1. Verification report

### 1.1 Method

Every statement in the v1 guide was compared against (a) the real command output pasted in the project chat, and (b) current official documentation. Items are marked **Verified**, **Corrected**, **Added**, or **Unverified**.

### 1.2 Corrections to the v1 guide

| # | v1 guide said | Reality | Impact | Action in v2 |
|---|---|---|---|---|
| 1 | privacyIDEA realm is `lab.com` (§3.2, §5.4, §5.5, §5.6, §7.4) | The realm that was actually created is **`lab`**. Calling `/validate/check` with `realm=lab.com` returned **HTTP 400**; with `realm=lab` it returned a normal `wrong otp value` response | **Critical.** The provider would fail for every user | Realm is `lab` everywhere |
| 2 | Phase 7 (provider), 8 (Exchange claims), 9 (test) all "Pending / not verified" | Most of this is **done and verified**: both relying party trusts, both claim rules, token-signing certificate trusted by Exchange, `Set-OrganizationConfig`, OWA/ECP set to AD FS, provider v1.4.0 installed, MFA working on OWA | Status table was misleading | New as-built status in §5 and §10 |
| 3 | Installing the provider and adding it under *Additional Authentication Methods* enables MFA | **Not enough.** After adding the provider the OWA sign-in still went straight to the inbox. MFA only started when each relying party trust was switched to the access-control policy **"Permit everyone and require MFA"** | **Critical.** MFA would silently not apply | New Phase 9 with the policy step |
| 4 | Registry example sets `use_upn = 1` and creates the key with `New-Item -Force` | The installer already creates the key and hardens its ACL. The lab runs with `use_upn` empty (login name `u2`). `use_upn=1` sends `u2@lab.com`, which only works if the privacyIDEA resolver login attribute also matches UPN | Could cause "user not found" | Use the installer dialog, then `Set-ItemProperty`; leave `use_upn` empty |
| 5 | Relying party trusts created with `-IssuanceAuthorizationRules` | In the lab, the trust made with the wizard had access-control policy "Permit everyone"; the one made with legacy rules (EAC) had an **empty** `AccessControlPolicyName`. Mixed styles make the later MFA switch inconsistent | Medium | Create both trusts with `-AccessControlPolicyName` |
| 6 | `Get-AdfsEndpoint -AddressPath "/adfs/ls/"` | A filter on `Address` returned nothing in the lab. The property that works is **`AddressPath`**, verified with `Where-Object` | Low | `Where-Object` form used |
| 7 | Token-signing certificate "must be identified" | Identified: `CN=ADFS Signing - fs.lab.com`, thumbprint `D0743A62F7CEB205867D9F8AD0B1F18FC0D9A2F8`, expires 7 Nov 2027, auto-rollover **on**. `Format-List Subject` on the object returns nothing; use `.Certificate` | Medium | Recorded; command fixed |
| 8 | Exchange 2019 "keep on a supported servicing level" | Microsoft's final Exchange 2016/2019 ESU period ended in **October 2026**. As of 8 Nov 2026 no further security updates are published for Exchange 2019; only Exchange Server SE receives them | **High (security)** | New warning in §10 |
| 9 | `iisreset` | In the lab, `iisreset /noforce` returned `0x8007041d` (service timed out). Settings had been saved, IIS needed `iisreset /start` and app pool checks | Medium | Recovery steps added in Phase 7 and §9 |
| 10 | Phase order: provider (7) before Exchange (8) | Both orders work. The lab proved **Exchange + AD FS with password only first, MFA second**. That isolates faults | Improvement | v2 uses the proven order |

### 1.3 Statements checked and confirmed

| Statement | Result |
|---|---|
| AD FS provider is limited to 50 users with active tokens without a subscription | **Verified** in privacyIDEA's plugin table (ADFS = 50). The limit counts users with at least one active token across all realms. The privacyIDEA server itself has no such limit |
| Microsoft AD FS claims-based auth covers Outlook on the web and EAC only; "Outlook on the web for devices" is not supported | **Verified** (Microsoft Learn) |
| Audience URIs must match relying party identifiers exactly, including the trailing slash | **Verified** (Microsoft procedure) |
| privacyIDEA Ubuntu 24.04 (noble) repository `http://lancelot.netknights.it/community/noble/stable`, key fingerprint `0940 4ABB EDB3 586D EDE4 AD22 00F7 0D62 AE25 0082`, package `privacyidea-apache2` | **Verified** (privacyIDEA docs) |
| Provider needs .NET Framework 4.8 (in-box on Windows Server 2022), config in `HKLM\SOFTWARE\NetKnights GmbH\PrivacyIDEA-ADFS`, restart `adfssrv` after changes | **Verified** (provider README read in the lab) |
| `Get-SendConnector` empty explains the CU15 warning "No Send connector with address space '*'" | **Verified** in the lab (no connectors exist). Harmless for MFA |
| `https://pi.lab.com/` returning HTTP 404 | **Verified normal.** TLS succeeded, the site has no root page, and mfadc trusts `LAB-MFADC-CA` |

### 1.4 Still unverified (must be tested by you)

- ECP (EAC) sign-in with MFA. The trust and policy are set, but no ECP login was recorded.
- Exchange IIS certificate SAN (`mail.lab.com`) was never printed in the chat.
- DC IP address, gMSA name, and send connector design.
- Behaviour of users without a token, invalid and expired OTP, disabled accounts (§7).
- 2,000-user licensing with NetKnights.

---

## 2. Executive summary, scope and limits

### 2.1 What this builds

Outlook on the web (OWA) and the Exchange admin center (ECP/EAC) redirect to AD FS (`fs.lab.com`). AD FS authenticates the user against Active Directory (password), then invokes the privacyIDEA AD FS provider, which validates a TOTP code against `pi.lab.com`. AD FS then issues a claims token (SID + UPN) that Exchange accepts.

### 2.2 Hard limits — read before promising anything to management

| Limit | Detail |
|---|---|
| **Protocol coverage** | Only **OWA and ECP** are protected. **Outlook desktop (MAPI), EWS, ActiveSync, POP/IMAP, SMTP AUTH still accept password only** until separately handled. Do not describe the project as "Exchange MFA" |
| **Licensing** | AD FS provider is free only up to **50 users with active tokens**. 2,000 users needs a NetKnights subscription (or another validated design) |
| **Exchange 2019 support** | Out of support; ESU ended October 2026. Plan Exchange Server SE |
| **Single points of failure** | One AD FS server and one privacyIDEA server. If either is down, OWA sign-in fails (fail closed) |
| **Internet access** | Not designed here. External use needs WAP/reverse proxy, firewall and public DNS design |

### 2.3 Non-goals

No secrets (passwords, OTP seeds, private keys) are stored in this document. It does not cover hybrid/Exchange Online, Outlook modern authentication, or Web Application Proxy.

---

## 3. Architecture

### 3.1 Component diagram

```text
                       lab.com  (Active Directory)
   +-----------------------------------------------------------------+
   |                                                                 |
   |   dc01.lab.com                  mfadc.lab.com  10.10.10.12      |
   |   AD DS + DNS + NTP             Windows Server 2022             |
   |   LDAPS :636                    - AD CS Enterprise Root CA      |
   |        ^                          "LAB-MFADC-CA"                |
   |        |                        - AD FS  fs.lab.com             |
   |        |                          + privacyIDEA AD FS provider  |
   |        |                            v1.4.0 (registry config)    |
   |        |                                  |  HTTPS :443         |
   |        | LDAPS                            v                     |
   |   +----+------------------+      pi.lab.com  10.10.10.13        |
   |   | privacyIDEA resolver  |<-----Ubuntu 24.04 + Apache2         |
   |   | LAB-AD, realm "lab"   |      privacyIDEA 3.14 + MariaDB     |
   |   +-----------------------+      TOTP validation (/validate/..) |
   |                                                                 |
   |   mail.lab.com  192.168.1.121   Windows Server 2022             |
   |   Exchange Server 2019 CU15 (15.2.1748.10)                      |
   |   OWA / ECP use AD FS claims auth  (trusts AD FS token-signing) |
   +-----------------------------------------------------------------+
        ^
        | HTTPS :443   browser (OWA / ECP)
     User
```

### 3.2 Authentication sequence

```text
Browser          Exchange (OWA)        AD FS (fs.lab.com)        privacyIDEA (pi)       AD
  |-- GET /owa ------>|                      |                        |                  |
  |<-- 302 to fs.lab.com/adfs/ls (wtrealm=https://mail.lab.com/owa/) -|                  |
  |-- user + password ------------------->   |                        |                  |
  |                                          |-- verify password ------------------------>|
  |                                          |<-- OK ---------------------------------------|
  |<-- OTP page (provider "privacyIDEAADFSProvider") |                |                  |
  |-- 6-digit TOTP ------------------------->|                        |                  |
  |                                          |-- POST /validate/check (user, realm=lab, pass)
  |                                          |<-- ACCEPT / REJECT ----|                  |
  |                                          |  claims rules: primarysid + upn            |
  |<-- signed token (Token-signing cert D074...) --------------------|                  |
  |-- POST token to /owa/ -->|               |                        |                  |
  |                          | verify signature, audience, issuer     |                  |
  |<-- mailbox session ------|               |                        |                  |
```

### 3.3 The four certificate roles (the most common source of errors)

| Role | Used by | Lab value | Where it must be trusted |
|---|---|---|---|
| **Root CA certificate** | Everyone | `LAB-MFADC-CA` | Domain members: automatic (Enterprise CA). Ubuntu: manual (`update-ca-certificates`) and privacyIDEA LDAP (`/etc/privacyidea/ldap-ca.crt`) |
| **AD FS service communications (SSL)** | Browsers, Exchange, provider | `CN=fs.lab.com`, thumbprint `3A6175F0F8FFDBDCEB3591E9AFF48013DDE0D161`, exp. 7 Nov 2028 | Clients trust the CA |
| **AD FS token-signing** | Exchange validates AD FS tokens | `CN=ADFS Signing - fs.lab.com`, thumbprint `D0743A62F7CEB205867D9F8AD0B1F18FC0D9A2F8`, exp. 7 Nov 2027 | Public `.cer` in Exchange **Local Computer → Trusted Root**, and thumbprint in `Set-OrganizationConfig` |
| **Exchange IIS** | Browsers | CN/SAN `mail.lab.com` | Clients trust the CA |
| **privacyIDEA HTTPS** | AD FS provider, admins | CN/SAN `pi.lab.com` | mfadc trusts `LAB-MFADC-CA` (verified: HTTP 404 on `/` means TLS worked) |

**Never** put the `3A61…` SSL thumbprint into Exchange. Exchange needs the **token-signing** thumbprint `D074…`.

### 3.4 Network flows

| Source | Destination | Port | Purpose |
|---|---|---:|---|
| Clients / servers | DC | 53, 88, 389 (TCP/UDP) | DNS, Kerberos, LDAP |
| privacyIDEA | `dc01.lab.com` | TCP 636 | LDAPS user lookup |
| Browser | `mail.lab.com`, `fs.lab.com` | TCP 443 | OWA/ECP, AD FS sign-in |
| AD FS | `pi.lab.com` | TCP 443 | OTP validation |
| Exchange | `fs.lab.com` | TCP 443 | Metadata / federation |
| Admin PC | `pi.lab.com` | TCP 443 | privacyIDEA web UI |
| All servers | `dc01.lab.com` | UDP 123 | Time |
| Admin PC | servers | TCP 3389, 5985 | RDP, WinRM (restricted) |

Verified in the lab: `192.168.1.121 → fs.lab.com:443` OK, `mfadc → pi.lab.com:443` OK, `mfadc → github.com:443` OK.

### 3.5 DNS records

| Name | Type | Value |
|---|---|---|
| `mail.lab.com` | A | `192.168.1.121` |
| `mfadc.lab.com` | A | `10.10.10.12` |
| `fs.lab.com` | A (not CNAME for simplicity) | `10.10.10.12` |
| `pi.lab.com` | A | `10.10.10.13` |
| `dc01.lab.com` | A | *(record your DC address; not captured in the project)* |

---

## 4. Phase workflow (wire-work overview)

### 4.1 Flow of phases

```text
 P0 Prereqs, baseline, backup
  |
 P1 DNS + time sync ------------------------------> GATE 1: names resolve, clocks within seconds
  |
 P2 AD CS + certificates -------------------------> GATE 2: CA, templates, certs for fs / mail / pi
  |
 P3 Exchange baseline ----------------------------> GATE 3: OWA works with Forms auth; settings saved
  |
 P4 privacyIDEA install (Ubuntu) -----------------> GATE 4: https://pi.lab.com loads, valid cert
  |
 P5 LDAP resolver + realm "lab" + TOTP + API test -> GATE 5: /validate/check returns ACCEPT
  |
 P6 AD FS verify (reference build) ---------------> GATE 6: metadata 200, token-signing cert recorded
  |
 P7 Exchange <-> AD FS claims (password only) ----> GATE 7: OWA via AD FS reaches the inbox
  |
 P8 privacyIDEA AD FS provider -------------------> GATE 8: provider listed, API reachable from mfadc
  |
 P9 Enforce MFA on the relying parties -----------> GATE 9: OTP page appears, valid OTP -> inbox
  |
 P10 Operations: ECP/admin token, backups, rollover, monitoring, licensing, client gaps
```

### 4.2 Phase table

| Phase | Where you work | Output | Status in lab |
|---:|---|---|---|
| 0 | All servers | Inventory, baseline files, backups | Do before any change |
| 1 | DC / all servers | DNS, time sync | Done (reverify) |
| 2 | mfadc, MAIL, pi | CA, templates, certificates | CA + fs cert done; verify mail cert SAN |
| 3 | MAIL | Healthy Exchange, saved settings | Done |
| 4 | pi | privacyIDEA 3.14 running with HTTPS | Done |
| 5 | pi web UI | LDAP resolver, realm `lab`, TOTP, API test | Done |
| 6 | mfadc | AD FS verified | Done |
| 7 | mfadc + MAIL | RP trusts, claims, trust, org config, vdirs | **Done, OWA verified** |
| 8 | mfadc | Provider v1.4.0 + registry | **Done** |
| 9 | mfadc | MFA enforced via access-control policy | **Done, OWA verified** |
| 10 | All | Operations, ECP test, scale, gaps | **Open** |

### 4.3 Rules that keep the build error-free

1. Never skip a Gate.
2. Change one system at a time, and save the old setting first.
3. Test OWA with **password only** (Phase 7) before adding MFA (Phase 9).
4. Always test in a **new InPrivate window** (old AD FS cookies skip MFA and hide errors).
5. Keep one Exchange Management Shell session open while changing OWA/ECP.
6. OTP codes live 30 seconds. Type the code, press Enter immediately, or wait for the next code.

---

## 5. Lab variables (as-built values)

Replace values only if your environment differs.

| Item | Value |
|---|---|
| AD domain / NetBIOS | `lab.com` / `LAB` |
| Domain controller | `dc01.lab.com` (IP: record it) |
| CA server / CA name | `mfadc.lab.com` / `LAB-MFADC-CA` (Enterprise Root) |
| AD FS | `fs.lab.com` → `10.10.10.12`, display name `LAB Federation`, farm node `mfadc.lab.com` |
| AD FS endpoint | `https://fs.lab.com/adfs/ls/` (Enabled, Proxy True) |
| Exchange | `MAIL`, `mail.lab.com`, `192.168.1.121`, 2019 CU15, `15.2.1748.10` |
| Exchange URLs | `https://mail.lab.com/owa`, `/ecp`, `/mapi`, `/EWS/Exchange.asmx`, `/Autodiscover`, `/OAB`, `/Microsoft-Server-ActiveSync` |
| privacyIDEA | `pi.lab.com`, `10.10.10.13`, Ubuntu 24.04.5, `3.14-1noble`, Apache2 |
| LDAP | `ldaps://dc01.lab.com:636`, Base DN `DC=lab,DC=com`, bind `svc_privacyidea@lab.com` |
| privacyIDEA resolver / **realm** | `LAB-AD` / **`lab`** |
| LDAP CA file on pi | `/etc/privacyidea/ldap-ca.crt` |
| Test user / token | `u2` (mailbox on `Mailbox Database 0682720048`) / `TOTP00007431` |
| Relying party trusts | `Outlook on the web` → `https://mail.lab.com/owa/`; `EAC` → `https://mail.lab.com/ecp/` |
| Provider | privacyIDEA AD FS Provider **v1.4.0**, MSI `PrivacyIDEA-ADFS-Provider-v1.4.0.0.msi`, signed by NetKnights GmbH, SHA256 `FEC88B67D243271BB0A54066D09D94BC2292552B754C558C50F61DF3E06FC9B1` |
| AD FS provider name | `privacyIDEAADFSProvider` |
| Access-control policy used | `Permit everyone and require MFA` |

---

## 6. Detailed implementation

> **Conventions.** `[MAIL]` = Exchange Management Shell on MAIL. `[MFADC]` = elevated PowerShell on mfadc.lab.com. `[DC]` = PowerShell on a domain controller. `[PI]` = bash on pi.lab.com. Run every PowerShell block as Administrator. Replace `<...>` placeholders only.

---

### Phase 0 — Prerequisites, baseline and backup

**Objective:** know exactly what exists, save it, and make sure you can undo anything.

**Prerequisites**

- Domain admin and Exchange organization-management rights.
- Local administrator accounts on MAIL and mfadc that do not depend on OWA.
- Static IP addresses on all servers.

**GUI**

1. Server Manager on each server → confirm computer name, domain and static IP.
2. Hypervisor: take a snapshot of all four VMs (DC, MAIL, mfadc, pi). A snapshot is a convenience, not your only backup.
3. Confirm you can open **Exchange Management Shell** locally on MAIL (not through OWA/ECP).

**CLI — create a working folder and save baselines**

`[MAIL]`
```powershell
New-Item -ItemType Directory -Path C:\Temp -Force | Out-Null
Get-OrganizationConfig | Format-List *Adfs* | Out-File C:\Temp\Before-OrgConfig.txt
Get-OwaVirtualDirectory | Format-List Identity,InternalUrl,ExternalUrl,*Authentication* | Out-File C:\Temp\Before-OWA.txt
Get-EcpVirtualDirectory | Format-List Identity,InternalUrl,ExternalUrl,*Authentication* | Out-File C:\Temp\Before-ECP.txt
Get-ExchangeServer | Format-List Name,Edition,AdminDisplayVersion | Out-File C:\Temp\Before-Version.txt
Get-ExchangeCertificate | Format-List Thumbprint,Subject,CertificateDomains,Services,NotAfter,Status | Out-File C:\Temp\Before-Certs.txt
Get-SendConnector | Format-List Name,AddressSpaces,Enabled | Out-File C:\Temp\Before-SendConn.txt
```

`[MFADC]` (only if AD FS already exists)
```powershell
New-Item -ItemType Directory -Path C:\Temp -Force | Out-Null
Import-Module ADFS
Get-AdfsProperties | Out-File C:\Temp\Before-AdfsProps.txt
Get-AdfsRelyingPartyTrust | Export-Clixml C:\Temp\Before-RPTs.xml
Get-AdfsGlobalAuthenticationPolicy | Out-File C:\Temp\Before-AdfsAuthPolicy.txt
Get-AdfsCertificate | Format-List CertificateType,IsPrimary,Thumbprint | Out-File C:\Temp\Before-AdfsCerts.txt
```

**Gate 0:** the `C:\Temp\Before-*.txt` files exist and show the original OWA/ECP settings (typically `FormsAuthentication True`, `BasicAuthentication True`, `AdfsAuthentication False`).

---

### Phase 1 — DNS and time synchronization

**Objective:** every server resolves every name and agrees on the time (TOTP depends on it).

**GUI (DNS)**

1. On the DNS server: **Server Manager → Tools → DNS**.
2. Expand **Forward Lookup Zones → lab.com**.
3. Confirm or create the A records in §3.5. Right-click the zone → **New Host (A or AAAA)**.
4. On each Windows server run `ipconfig /flushdns`.

**CLI (DNS)** `[DC]`
```powershell
Get-DnsServerResourceRecord -ZoneName "lab.com" -RRType A | Format-Table HostName,@{n='IP';e={$_.RecordData.IPv4Address}}

# Create only the ones missing:
Add-DnsServerResourceRecordA -ZoneName "lab.com" -Name "mail"  -IPv4Address "192.168.1.121"
Add-DnsServerResourceRecordA -ZoneName "lab.com" -Name "mfadc" -IPv4Address "10.10.10.12"
Add-DnsServerResourceRecordA -ZoneName "lab.com" -Name "fs"    -IPv4Address "10.10.10.12"
Add-DnsServerResourceRecordA -ZoneName "lab.com" -Name "pi"    -IPv4Address "10.10.10.13"
```

**CLI (verify from every Windows server)**
```powershell
foreach ($n in "mail.lab.com","fs.lab.com","pi.lab.com","dc01.lab.com") { Resolve-DnsName $n | Select-Object Name,IPAddress }
Test-NetConnection fs.lab.com -Port 443
Test-NetConnection pi.lab.com -Port 443
```

**Time — Windows:** domain members follow the domain hierarchy; do not point them at random NTP servers.
```powershell
w32tm /query /status
w32tm /query /source
w32tm /resync
```

**Time — Ubuntu** `[PI]` (the lab syncs from `dc01.lab.com`)
```bash
sudo mkdir -p /etc/systemd/timesyncd.conf.d
printf "[Time]\nNTP=dc01.lab.com\n" | sudo tee /etc/systemd/timesyncd.conf.d/lab.conf
sudo timedatectl set-ntp true
sudo systemctl restart systemd-timesyncd
timedatectl
timedatectl timesync-status
```
Expected: `System clock synchronized: yes`.

**Gate 1:** all names resolve from all servers; `pi.lab.com:443` and `fs.lab.com:443` reachable from mfadc/MAIL (both were verified in the lab); clocks agree within a few seconds. Set the phone clock to automatic as well.

---

### Phase 2 — AD CS and certificates

**Objective:** one trusted CA and correct certificates for `fs.lab.com`, `mail.lab.com` and `pi.lab.com`.

> **Design note.** CA and AD FS share one server in this lab. That is acceptable for a lab. For production use a separate, offline root CA and issuing CA.

#### 2.1 Install the Enterprise Root CA (reference — already done in the lab)

**GUI:** Server Manager → **Add roles and features** → *Active Directory Certificate Services* → role service *Certification Authority* → after install click **Configure AD CS** → *Enterprise CA* → *Root CA* → new private key (RSA 4096, SHA256) → common name `LAB-MFADC-CA` → validity 10 years.

**CLI** `[MFADC]`
```powershell
Install-WindowsFeature ADCS-Cert-Authority -IncludeManagementTools
Install-AdcsCertificationAuthority -CAType EnterpriseRootCA -CACommonName "LAB-MFADC-CA" `
  -CryptoProviderName "RSA#Microsoft Software Key Storage Provider" -KeyLength 4096 `
  -HashAlgorithmName SHA256 -ValidityPeriod Years -ValidityPeriodUnits 10 -Force
```

**Verify**
```powershell
Get-Service CertSvc
certutil -CAInfo
certutil -ping
```

#### 2.2 Create and publish the web server template

**GUI**

1. `certtmpl.msc` → right-click **Web Server** → **Duplicate Template**.
2. **General:** name `LAB-WebServer`, validity 2 years.
3. **Request Handling:** if you will move keys between servers, allow private key export; otherwise leave it off.
4. **Subject Name:** *Supply in the request*.
5. **Security:** give **Enroll** to *Domain Computers* (and to the admins who will run `certreq`). Do not grant it to *Authenticated Users* unless you accept that risk.
6. `certsrv.msc` → your CA → **Certificate Templates** → right-click → **New → Certificate Template to Issue** → choose `LAB-WebServer`.

**CLI**
```powershell
Add-CATemplate -Name "LAB-WebServer" -Force
Get-CATemplate | Format-Table Name
```

#### 2.3 AD FS service communications certificate (`fs.lab.com`)

`[MFADC]`
```powershell
$c = Get-Certificate -Template "LAB-WebServer" -SubjectName "CN=fs.lab.com" -DnsName "fs.lab.com" -CertStoreLocation Cert:\LocalMachine\My
$c.Certificate | Format-List Subject,Thumbprint,NotAfter,HasPrivateKey
```
Record the thumbprint (lab value: `3A6175F0F8FFDBDCEB3591E9AFF48013DDE0D161`).

Verify what AD FS actually uses:
```powershell
Import-Module ADFS
Get-AdfsSslCertificate
```

#### 2.4 Exchange IIS certificate (`mail.lab.com`)

This was **never printed** in the project, so check it first. `[MAIL]`
```powershell
Get-ExchangeCertificate | Format-List Thumbprint,Subject,CertificateDomains,Services,NotAfter,Status
```
Continue only if no valid certificate has `mail.lab.com` in `CertificateDomains` and `IIS` in `Services`.

```powershell
# 1. Generate the request
$req = New-ExchangeCertificate -GenerateRequest -FriendlyName "mail.lab.com" `
        -SubjectName "CN=mail.lab.com" -DomainName mail.lab.com -PrivateKeyExportable $true
Set-Content -Path C:\Temp\mail.req -Value $req

# 2. Submit to the CA
certreq -submit -config "mfadc.lab.com\LAB-MFADC-CA" -attrib "CertificateTemplate:LAB-WebServer" C:\Temp\mail.req C:\Temp\mail.cer

# 3. Import and bind
Import-ExchangeCertificate -FileData ([Byte[]]$(Get-Content -Path C:\Temp\mail.cer -Encoding Byte -ReadCount 0))
Get-ExchangeCertificate | Format-Table Thumbprint,Subject,Services,NotAfter
Enable-ExchangeCertificate -Thumbprint "<MAIL_CERT_THUMBPRINT>" -Services IIS
```
If you also use SMTP with this certificate add `,SMTP`. If Exchange prompts to overwrite the existing default certificate, answer `N` unless you intend to replace it.

**GUI alternative:** Exchange Admin Center → **Servers → Certificates** → select server → **+** (new) → create request → submit to CA → **Complete** → **Edit → Services → IIS**.

#### 2.5 privacyIDEA HTTPS certificate (`pi.lab.com`)

Already done in the lab. To repeat: `[PI]`
```bash
sudo openssl req -new -newkey rsa:2048 -nodes \
  -keyout /etc/ssl/private/pi.lab.com.key -out ~/pi.lab.com.csr \
  -subj "/CN=pi.lab.com" -addext "subjectAltName=DNS:pi.lab.com"
sudo chmod 600 /etc/ssl/private/pi.lab.com.key
```
Copy `pi.lab.com.csr` to mfadc, then `[MFADC]`:
```powershell
certreq -submit -config "mfadc.lab.com\LAB-MFADC-CA" -attrib "CertificateTemplate:LAB-WebServer" C:\Temp\pi.lab.com.csr C:\Temp\pi.lab.com.cer
certutil -ca.cert C:\Temp\LAB-MFADC-CA.cer
```
Copy `pi.lab.com.cer` and `LAB-MFADC-CA.cer` back to pi. `[PI]`
```bash
# Convert if the files are DER (binary). If the head of the file shows "BEGIN CERTIFICATE" they are already PEM.
head -1 pi.lab.com.cer
sudo openssl x509 -inform der -in pi.lab.com.cer -out /etc/ssl/certs/pi.lab.com.crt   # DER case
# PEM case:  sudo cp pi.lab.com.cer /etc/ssl/certs/pi.lab.com.crt
sudo openssl x509 -inform der -in LAB-MFADC-CA.cer -out /usr/local/share/ca-certificates/LAB-MFADC-CA.crt   # DER case
sudo update-ca-certificates
openssl verify -CAfile /usr/local/share/ca-certificates/LAB-MFADC-CA.crt /etc/ssl/certs/pi.lab.com.crt
```
Expected: `OK`. The Apache binding is done in Phase 4.

#### 2.6 AD FS token-signing certificate (identify it now, import later)

`[MFADC]`
```powershell
Get-AdfsCertificate -CertificateType Token-Signing | Format-Table IsPrimary,Thumbprint
(Get-AdfsCertificate -CertificateType Token-Signing).Certificate | Format-List Subject,NotAfter,Thumbprint
```
Lab result: `CN=ADFS Signing - fs.lab.com`, `D0743A62F7CEB205867D9F8AD0B1F18FC0D9A2F8`, expires `11/7/2027`.

**Gate 2:** CA service running; template published; fs and pi certificates issued by `LAB-MFADC-CA`; Exchange has a valid IIS certificate for `mail.lab.com` (check `CertificateDomains`); `openssl verify` says `OK`; both thumbprints (SSL and token-signing) are written down separately.

---

### Phase 3 — Exchange baseline

**Objective:** prove Exchange works with its original authentication before touching it.

**GUI**

1. Open `https://mail.lab.com/owa` and sign in as `u2`. The inbox must open.
2. Open `https://mail.lab.com/ecp` and sign in as an administrator.
3. EAC → **Servers → Servers**: verify version `15.2.1748.10`.
4. EAC → **Servers → Virtual directories**: note OWA/ECP URLs.
5. EAC → **Mail flow → Send connectors**. If empty, the CU15 warning *"No Send connector with address space '*'"* is expected. It does not affect MFA. Create a connector only if you need outbound Internet mail.

**CLI** `[MAIL]`
```powershell
Get-ExchangeServer | Format-List Name,Edition,AdminDisplayVersion
Get-Service MSExchange*,W3SVC,WAS | Where-Object {$_.Status -ne 'Running' -and $_.StartType -eq 'Automatic'} | Format-Table Name,Status
Get-MailboxDatabase -Status | Format-Table Name,Mounted
Get-Mailbox u2 | Format-List Name,UserPrincipalName,Database
Get-OwaVirtualDirectory | Format-List Server,InternalUrl,ExternalUrl,AdfsAuthentication,FormsAuthentication,BasicAuthentication,WindowsAuthentication
Get-EcpVirtualDirectory | Format-List Server,InternalUrl,ExternalUrl,AdfsAuthentication,FormsAuthentication,BasicAuthentication,WindowsAuthentication
Get-OrganizationConfig | Format-List AdfsIssuer,AdfsAudienceUris,AdfsSignCertificateThumbprints,AdfsEncryptCertificateThumbprint
```
Lab baseline: `InternalUrl = https://mail.lab.com/owa` and `/ecp`, `ExternalUrl` empty, Forms and Basic `True`, `AdfsAuthentication False`.

> **RBAC.** If you get *"not assigned to any management roles"*, use an account that really belongs to *Organization Management*. Do not bypass RBAC.

**Gate 3:** no stopped automatic Exchange/IIS service; `u2` has a mailbox; OWA login works the old way; baseline files from Phase 0 exist.

---

### Phase 4 — Install privacyIDEA on Ubuntu 24.04

**Objective:** a running privacyIDEA server reachable at `https://pi.lab.com` with a CA-signed certificate. (Already done in the lab; follow this only to rebuild.)

**CLI** `[PI]`
```bash
sudo apt update && sudo apt full-upgrade -y
sudo hostnamectl set-hostname pi
getent hosts pi.lab.com          # must return 10.10.10.13

# Signing key
wget https://lancelot.netknights.it/NetKnights-Release.asc
gpg --import --import-options show-only --with-fingerprint NetKnights-Release.asc
#   compare with: 0940 4ABB EDB3 586D EDE4 AD22 00F7 0D62 AE25 0082   (stop if different)
sudo mv NetKnights-Release.asc /etc/apt/trusted.gpg.d/

# Repository and package
sudo add-apt-repository http://lancelot.netknights.it/community/noble/stable
sudo apt update
sudo apt install privacyidea-apache2
```
The installer asks setup questions (local database, certificates). Accept the defaults for a local MariaDB/MySQL database.

**Create the first administrator**
```bash
sudo pi-manage admin add administrator
```
Enter a strong password. Keep this account for administration only.

**Bind the CA-signed certificate**

The package enables its own Apache site and a default self-signed certificate. Find the lines:
```bash
sudo grep -n "SSLCertificateFile\|SSLCertificateKeyFile" /etc/apache2/sites-available/privacyidea.conf
```
Edit that file and point the two lines at your files:
```apache
SSLCertificateFile    /etc/ssl/certs/pi.lab.com.crt
SSLCertificateKeyFile /etc/ssl/private/pi.lab.com.key
```
```bash
sudo apache2ctl configtest          # expect: Syntax OK
sudo systemctl reload apache2
```

**Verify**
```bash
systemctl is-enabled apache2
systemctl is-enabled mariadb 2>/dev/null || systemctl is-enabled mysql
sudo ss -lntp | grep ':443'
curl -I https://pi.lab.com
openssl s_client -connect pi.lab.com:443 -servername pi.lab.com </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

**GUI**

1. Browse to `https://pi.lab.com` from an admin PC (no certificate warning).
2. Sign in as `administrator` (type of login: local admin).
3. The dashboard loads.

**Gate 4:** valid HTTPS certificate for `pi.lab.com`, admin login works, Apache and the database are enabled for boot. A plain `GET https://pi.lab.com/` returning **404** from other clients is normal.

---

### Phase 5 — LDAP resolver, realm `lab`, TOTP and API test

**Objective:** privacyIDEA sees AD users and validates a TOTP for `u2`.

#### 5.1 Check LDAPS first `[PI]`
```bash
nc -vz dc01.lab.com 636
sudo apt install -y ldap-utils
sudo openssl x509 -in /etc/privacyidea/ldap-ca.crt -noout -subject -issuer -dates
LDAPTLS_CACERT=/etc/privacyidea/ldap-ca.crt ldapsearch -H ldaps://dc01.lab.com:636 \
  -D "svc_privacyidea@lab.com" -W -b "DC=lab,DC=com" "(sAMAccountName=u2)" sAMAccountName userPrincipalName
```
`-W` prompts for the password so it never lands in shell history. If the CA is not trusted, place the CA certificate at `/etc/privacyidea/ldap-ca.crt` (PEM).

#### 5.2 Create the resolver (GUI)

1. **Config → Users → New ldapresolver** (menu labels vary slightly by version).
2. Name: `LAB-AD`. Choose the **Active Directory** preset if offered.
3. Server URI: `ldaps://dc01.lab.com:636`. Tick **Verify TLS**, TLS CA file `/etc/privacyidea/ldap-ca.crt`.
4. Base DN: `DC=lab,DC=com` (better: the OU that holds your users).
5. Bind DN: `svc_privacyidea@lab.com`; enter the password in the UI only.
6. Login name attribute: `sAMAccountName`. Search filter: `(sAMAccountName=*)(objectCategory=person)`. UID attribute: `objectGUID`.
7. Attribute mapping (preset): `{"username":"sAMAccountName","email":"mail","givenname":"givenName","surname":"sn","phone":"telephoneNumber","mobile":"mobile"}`.
8. Leave **Edit user store** off. Click **Test LDAP Resolver**; expect *"Your LDAP config seems to be OK, found N users"*. Save.

> `svc_privacyidea` is **only the AD bind account**. It is not a privacyIDEA admin and must not be used as the AD FS provider service account.

#### 5.3 Create the realm `lab` (GUI)

1. **Config → Realms → Create realm.**
2. Realm name: **`lab`** (no `.com`). Tick resolver `LAB-AD`. Save.
3. Make `lab` the **default realm**.
4. **Users** tab → realm `lab` → `u2` appears.

> The name you choose here must be typed identically in the AD FS provider registry (`realm`) and in every API test. Using `lab.com` returned HTTP 400 in the lab.

#### 5.4 Enroll a TOTP token (GUI)

1. **Tokens → Enroll Token → TOTP.**
2. User: `u2`, realm `lab`. Defaults (SHA1, 6 digits, 30 s) match Microsoft/Google Authenticator and most apps.
3. Click **Enroll**, scan the QR code with the authenticator app, then verify with the current code in the UI.
4. In **Token details** confirm: Status *active*, user `u2`, **Fail Counter 0**. The lab token serial is `TOTP00007431`.
5. Never paste the QR code or seed in a ticket.

#### 5.5 Test the validation API from the AD FS server `[MFADC]`

Bad-OTP test (proves user, realm, TLS and API):
```powershell
$body = @{ user="u2"; realm="lab"; pass="000000" }
Invoke-RestMethod -Uri https://pi.lab.com/validate/check -Method Post -Body $body
```
Expected `message=wrong otp value`, `serial=TOTP00007431`, `authentication=REJECT`, `status=True`, `value=False`.

Good-OTP test (replace `123456` with the code currently shown in the app, then run **immediately**):
```powershell
$body = @{ user="u2"; realm="lab"; pass="123456" }
(Invoke-RestMethod -Uri https://pi.lab.com/validate/check -Method Post -Body $body).result
```
Expected `authentication=ACCEPT`, `value=True`.

To see the real error text on an HTTP 400/500:
```powershell
try { Invoke-RestMethod -Uri https://pi.lab.com/validate/check -Method Post -Body $body }
catch { (New-Object IO.StreamReader($_.Exception.Response.GetResponseStream())).ReadToEnd() }
```

> In the lab two valid-looking attempts returned `REJECT` most likely because the code had expired before it was sent (30 s window), and one run sent the literal placeholder text instead of a code. A third, fast attempt returned `ACCEPT`. Repeated failures raise the token **Fail Counter**; reset it in the UI if it reaches the limit.

**Policies — decide before production**

| Decision | Recommended default |
|---|---|
| User with no token | Leave the default (authentication fails). **Do not** enable `passthru` for the AD FS realm |
| Who enrolls | Admin-enrolled for the pilot; self-service later via a *user* scope policy that allows TOTP enrollment (action names vary per version; the policy UI lists them) |
| Lockout | Keep the default failcounter; document the reset procedure |
| Lost phone | Identity-verified admin re-enrollment; never email seeds |

**Gate 5:** resolver test OK, realm `lab` shows AD users, `u2` token active, **the good-OTP call returns ACCEPT**.

---

### Phase 6 — AD FS (verify, or build)

**Objective:** a working federation service `fs.lab.com` with a recorded token-signing certificate.

> **The lab already has AD FS.** Run only the **Verify** block unless you are rebuilding.

**Verify (GUI)**

1. `[MFADC]` Server Manager → Tools → **AD FS Management**.
2. **Service → Certificates:** note Service communications and Token-signing separately.
3. **Service → Endpoints:** `/adfs/ls/` is enabled.
4. **Authentication Policies:** primary = Forms (and Windows) authentication; note that no additional provider is selected yet.

**Verify (CLI)** `[MFADC]`
```powershell
Import-Module ADFS
Get-Service adfssrv | Format-Table Name,Status
Get-AdfsProperties | Format-List HostName,DisplayName,AutoCertificateRollover,CertificateGenerationThreshold,CertificatePromotionThreshold
Get-AdfsFarmInformation | Format-List FarmNodes
Get-AdfsEndpoint | Where-Object { $_.AddressPath -like "/adfs/ls*" } | Format-Table AddressPath,FullUrl,Enabled,Proxy -AutoSize
Get-AdfsGlobalAuthenticationPolicy | Format-List AdditionalAuthenticationProvider
Invoke-WebRequest "https://fs.lab.com/FederationMetadata/2007-06/FederationMetadata.xml" -UseBasicParsing | Select-Object StatusCode
Get-CimInstance Win32_Service -Filter "Name='adfssrv'" | Select-Object Name,StartName,State
```
Lab results: service Running, `HostName fs.lab.com`, `DisplayName LAB Federation`, one farm node `mfadc.lab.com`, endpoint `https://fs.lab.com/adfs/ls/` Enabled, metadata 200, `AutoCertificateRollover True` (generation threshold 20 days, promotion 5 days), additional provider list empty.

**Build (reference only)** — if you must create the farm

```powershell
Install-WindowsFeature ADFS-Federation -IncludeManagementTools
# Lab only: for production wait for KDS replication (10 hours) instead of back-dating
Add-KdsRootKey -EffectiveTime ((Get-Date).AddHours(-10))
New-ADServiceAccount -Name adfs-gmsa -DNSHostName fs.lab.com -ServicePrincipalNames "http/fs.lab.com"
Install-AdfsFarm -CertificateThumbprint "<FS_CERT_THUMBPRINT>" -FederationServiceName "fs.lab.com" `
  -FederationServiceDisplayName "LAB Federation" -GroupServiceAccountIdentifier "LAB\adfs-gmsa$"
```
Do not rebuild a working farm to match a sample gMSA name; the project did not record the original account.

> **Optional test page.** `Set-AdfsProperties -EnableIdPInitiatedSignonPage $true` exposes `https://fs.lab.com/adfs/ls/idpinitiatedsignon`, which lets you test sign-in and MFA without Exchange. Turn it off afterward (`$false`).

**Gate 6:** metadata HTTP 200 from MAIL and mfadc; token-signing thumbprint and expiry recorded; service account known.

---

### Phase 7 — Exchange ↔ AD FS claims trust (password only, no MFA yet)

**Objective:** OWA and ECP redirect to AD FS and accept its token. MFA is added later (Phase 9) so any fault here is clearly a claims/trust fault.

**Prerequisites:** Gates 2, 3 and 6 passed. Maintenance window. An EMS window on MAIL stays open throughout.

#### 7.1 Relying party trusts on AD FS

The identifiers must match the Exchange audience URIs **character for character, including the trailing slash**.

| Trust name | Identifier / WS-Fed endpoint |
|---|---|
| `Outlook on the web` | `https://mail.lab.com/owa/` |
| `EAC` | `https://mail.lab.com/ecp/` |

**GUI** `[MFADC]`

1. AD FS Management → **Relying Party Trusts → Add Relying Party Trust**.
2. *Claims aware* → **Enter data about the relying party manually**.
3. Display name `Outlook on the web`. Skip the token-encryption certificate.
4. Tick **Enable support for the WS-Federation Passive protocol**, URL `https://mail.lab.com/owa/`.
5. Identifier `https://mail.lab.com/owa/`.
6. Access control policy: **Permit everyone** (it is changed in Phase 9).
7. Finish. Repeat for `EAC` with `https://mail.lab.com/ecp/`.

**CLI** `[MFADC]` (skip if `Get-AdfsRelyingPartyTrust` already lists both; never create duplicates)
```powershell
Import-Module ADFS
Get-AdfsRelyingPartyTrust | Format-Table Name,Identifier,Enabled,AccessControlPolicyName -AutoSize

$Owa = "https://mail.lab.com/owa/"
$Ecp = "https://mail.lab.com/ecp/"

Add-AdfsRelyingPartyTrust -Name "Outlook on the web" -Identifier $Owa -WSFedEndpoint $Owa `
  -AccessControlPolicyName "Permit everyone" -IssueOAuthRefreshTokensTo NoDevice
Add-AdfsRelyingPartyTrust -Name "EAC" -Identifier $Ecp -WSFedEndpoint $Ecp `
  -AccessControlPolicyName "Permit everyone" -IssueOAuthRefreshTokensTo NoDevice
```
Use `-AccessControlPolicyName` **or** legacy `-IssuanceAuthorizationRules`, never both. v2 uses the policy form so Phase 9 works identically on both trusts.

#### 7.2 Claim rules (SID + UPN) `[MFADC]`

Both trusts need `primarysid` and `upn` claims.

```powershell
$Rules = @'
@RuleName = "ActiveDirectoryUserSID"
c:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer == "AD AUTHORITY"]
 => issue(store = "Active Directory", types = ("http://schemas.microsoft.com/ws/2008/06/identity/claims/primarysid"), query = ";objectSID;{0}", param = c.Value);

@RuleName = "ActiveDirectoryUPN"
c:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer == "AD AUTHORITY"]
 => issue(store = "Active Directory", types = ("http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn"), query = ";userPrincipalName;{0}", param = c.Value);
'@

Set-AdfsRelyingPartyTrust -TargetName "Outlook on the web" -IssuanceTransformRules $Rules
Set-AdfsRelyingPartyTrust -TargetName "EAC"                -IssuanceTransformRules $Rules
```

**GUI:** relying party trust → **Edit Claim Issuance Policy → Add Rule → Send Claims Using a Custom Rule** → paste each rule above (two rules per trust).

**Verify** (the lab output matched exactly)
```powershell
Get-AdfsRelyingPartyTrust | ForEach-Object { $_.Name; $_.IssuanceTransformRules }
```

#### 7.3 Make Exchange trust the AD FS token-signing certificate

`[MFADC]` export the **public** certificate only (never export the private key):
```powershell
$tsc = (Get-AdfsCertificate -CertificateType Token-Signing | Where-Object IsPrimary).Certificate
Export-Certificate -Cert $tsc -FilePath C:\Temp\ADFS-TokenSigning.cer -Type CERT
$tsc.Thumbprint
```
**GUI:** AD FS Management → Service → Certificates → right-click the primary Token-signing certificate → *View Certificate* → *Details* → *Copy to File* → DER or Base-64 `.cer`, **no private key**.

Copy `C:\Temp\ADFS-TokenSigning.cer` to MAIL (RDP clipboard or `Copy-Item C:\Temp\ADFS-TokenSigning.cer \\mail.lab.com\c$\Temp\`).

`[MAIL]` (elevated PowerShell)
```powershell
Import-Certificate -FilePath C:\Temp\ADFS-TokenSigning.cer -CertStoreLocation Cert:\LocalMachine\Root
Get-ChildItem Cert:\LocalMachine\Root | Where-Object Thumbprint -eq "D0743A62F7CEB205867D9F8AD0B1F18FC0D9A2F8" | Format-List Subject,Thumbprint,NotAfter
```
Expected: `CN=ADFS Signing - fs.lab.com`, expiry `11/7/2027`. (Verified in the lab.)

**GUI on MAIL:** `certlm.msc` → Trusted Root Certification Authorities → Certificates → right-click → All Tasks → Import → pick the `.cer`.

#### 7.4 Exchange organization settings `[MAIL]`

First see which spelling your Exchange build exposes (Microsoft's text uses `AdfsSignCertificateThumbprint`, the object property is `AdfsSignCertificateThumbprints`):
```powershell
$p = (Get-Command Set-OrganizationConfig).Parameters.Keys | Where-Object { $_ -like "AdfsSignCertificateThumbprint*" }
$p
```
Then apply, which works with whichever name was found:
```powershell
$params = @{
  AdfsIssuer       = "https://fs.lab.com/adfs/ls/"
  AdfsAudienceUris = "https://mail.lab.com/owa/","https://mail.lab.com/ecp/"
}
$params[$p] = "D0743A62F7CEB205867D9F8AD0B1F18FC0D9A2F8"
Set-OrganizationConfig @params
Get-OrganizationConfig | Format-List AdfsIssuer,AdfsAudienceUris,AdfsSignCertificateThumbprints
```
The lab already shows this applied, with audience URIs listed both with and without trailing slash. That works; for a new build use the slash form above. If `$p` returns two names, use the first.

> Not the `3A61…` thumbprint. That is the SSL certificate.

#### 7.5 Switch ECP first, then OWA `[MAIL]`

```powershell
Get-EcpVirtualDirectory | Format-Table Identity
Get-OwaVirtualDirectory | Format-Table Identity
```
Use the exact identities returned (lab: `MAIL\ecp (Default Web Site)`, `MAIL\owa (Default Web Site)`).

```powershell
Set-EcpVirtualDirectory -Identity "MAIL\ecp (Default Web Site)" -AdfsAuthentication $true `
  -BasicAuthentication $false -DigestAuthentication $false -FormsAuthentication $false `
  -OAuthAuthentication $false -WindowsAuthentication $false

Set-OwaVirtualDirectory -Identity "MAIL\owa (Default Web Site)" -AdfsAuthentication $true `
  -BasicAuthentication $false -DigestAuthentication $false -FormsAuthentication $false `
  -OAuthAuthentication $false -WindowsAuthentication $false
```
`-OAuthAuthentication $false` is part of Microsoft's procedure. The lab originally ran without it and OWA worked; include it on new builds.

**Restart IIS safely**
```powershell
iisreset /noforce
```
If you see **`The service did not respond to the start or control request in a timely fashion (0x8007041d)`** (it happened in the lab):

1. The settings were already saved; do not rerun them.
2. `iisreset /start`
3. Check the app pools **one at a time** (the cmdlet does not accept a list):
```powershell
Import-Module WebAdministration
foreach ($p in "MSExchangeOWAAppPool","MSExchangeECPAppPool","MSExchangeAutodiscoverAppPool") {
  "{0} : {1}" -f $p, (Get-WebAppPoolState -Name $p).Value
}
# if one is Stopped:  Start-WebAppPool -Name <pool>
Get-Service MSExchange*,W3SVC,WAS | Where-Object {$_.Status -ne 'Running' -and $_.StartType -eq 'Automatic'} | Format-Table Name,Status
```
4. Wait 2–3 minutes; Exchange services start slowly in a lab VM.

**Verify**
```powershell
Get-OwaVirtualDirectory | Format-List Server,AdfsAuthentication,FormsAuthentication,BasicAuthentication
Get-EcpVirtualDirectory | Format-List Server,AdfsAuthentication,FormsAuthentication,BasicAuthentication
```
Expected `AdfsAuthentication True`, Forms/Basic `False`.

#### 7.6 Test (password only)

1. Close all InPrivate windows; open a **new** one.
2. Browse to `https://mail.lab.com/owa`. The URL changes to `https://fs.lab.com/adfs/ls/?wa=wsignin1.0&wtrealm=https%3a%2f%2fmail.lab.com%2fowa%2f…`.
3. Sign in as `u2@lab.com` + password. The inbox opens. **(Verified in the lab.)**

**Gate 7:** OWA reaches the inbox through AD FS with password only. If not, stop and use §9 before Phase 8. ECP is tested after the admin token exists (Phase 10).

---

### Phase 8 — Install the privacyIDEA AD FS provider

**Objective:** AD FS can ask privacyIDEA to validate an OTP. Do not enable it yet.

**Prerequisites:** Gate 5 (API test returns ACCEPT with realm `lab`), Gate 6, .NET Framework 4.8 (in-box on Windows Server 2022), `github.com:443` and `pi.lab.com:443` reachable from mfadc.

#### 8.1 Download and verify `[MFADC]`

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
$r = Invoke-RestMethod "https://api.github.com/repos/privacyidea/adfs-provider/releases/latest"
$r.tag_name
$r.assets | Format-Table name,size,browser_download_url -AutoSize -Wrap

New-Item -ItemType Directory -Path C:\Temp -Force | Out-Null
$msi = "C:\Temp\" + $r.assets[0].name
Invoke-WebRequest -Uri $r.assets[0].browser_download_url -OutFile $msi -UseBasicParsing
Get-FileHash $msi -Algorithm SHA256
Get-AuthenticodeSignature $msi | Format-List Status,SignerCertificate
```
Continue **only** if `Status : Valid` and the subject is `CN=NetKnights GmbH`. Lab result: v1.4.0, `PrivacyIDEA-ADFS-Provider-v1.4.0.0.msi`, 1,388,544 bytes, signature Valid, SHA256 `FEC88B67…C9B1`. A later release will have a different hash; that is expected.

Read the README of the version you downloaded; it is authoritative for registry value names:
```powershell
Invoke-RestMethod "https://raw.githubusercontent.com/privacyidea/adfs-provider/master/README.md"
```

#### 8.2 Install

**GUI:** right-click the MSI → **Install**. In the dialog enter:

| Field | Value |
|---|---|
| URL | `https://pi.lab.com` (must start with `https://`) |
| Realm | **`lab`** |
| Service user / password / realm | **empty** |
| Trigger challenges, send empty pass, enable enrollment | **off** |

**CLI**
```powershell
msiexec /i "C:\Temp\PrivacyIDEA-ADFS-Provider-v1.4.0.0.msi" /L*V C:\Temp\install.log
```
Adjust the file name to the version you downloaded. If the installer fails, read the last third of `C:\Temp\install.log`.

> **Why no service account?** `service_user`/`service_pass` are only needed for `trigger_challenges` or the deprecated `enable_enrollment`. TOTP is typed by the user, so no challenge needs triggering. Do not reuse `svc_privacyidea` (an AD account) here. If you add challenge-based tokens later, create a dedicated least-privilege privacyIDEA service identity.

#### 8.3 Check and set the registry `[MFADC]`

```powershell
$k = "HKLM:\SOFTWARE\NetKnights GmbH\PrivacyIDEA-ADFS"
Get-ItemProperty $k | Format-List url,realm,use_upn,disable_ssl,debug_log,service_user,send_empty_pass,trigger_challenges,enable_enrollment
```
Required result: `url = https://pi.lab.com`, `realm = lab`, everything else empty. If the realm shows `lab.com` (or anything else), correct it and restart AD FS:
```powershell
Set-ItemProperty -Path $k -Name realm -Value "lab"
Restart-Service adfssrv
Get-Service adfssrv | Format-Table Name,Status
```
Notes:

- Do **not** use `New-Item -Force` on the key; the installer already created it and hardened its ACL (only SYSTEM, Administrators and the AD FS service account).
- Leave `disable_ssl` empty. The CA is trusted.
- Leave `use_upn` empty: privacyIDEA receives `u2`, which matches the resolver login attribute.
- `debug_log = 1` only while troubleshooting; the log (`C:\PrivacyIDEA-ADFS log.txt`) contains usernames and server responses. Set it back and delete the file afterward.

#### 8.4 Confirm AD FS sees the provider

```powershell
Get-AdfsAuthenticationProvider | Format-Table Name -AutoSize
```
Expected to include **`privacyIDEAADFSProvider`**. Provider errors appear in the Application log, source `privacyIDEAProvider`:
```powershell
Get-WinEvent -LogName Application -MaxEvents 50 | Where-Object ProviderName -like "*privacyIDEA*" | Format-List TimeCreated,Message
```

**Gate 8:** provider listed; registry `url`/`realm` correct; `adfssrv` Running; mfadc can call `/validate/check` with realm `lab` (Phase 5.5). MFA is still **off**.

---

### Phase 9 — Enforce MFA on OWA and ECP

**Objective:** make AD FS ask for the OTP.

> **Critical.** Registering the provider is not the same as requiring it. In the lab, after the provider was added, the sign-in still went straight to the inbox because the relying party trusts used "Permit everyone". MFA started only after the access-control policy was changed.

#### 9.1 Enable the provider and switch the policies `[MFADC]`

```powershell
# What exists
Get-AdfsGlobalAuthenticationPolicy | Format-List AdditionalAuthenticationProvider
Get-AdfsAccessControlPolicy | Format-Table Name -AutoSize     # confirm "Permit everyone and require MFA" is listed

# Enable the provider
Set-AdfsGlobalAuthenticationPolicy -AdditionalAuthenticationProvider "privacyIDEAADFSProvider"
Restart-Service adfssrv

# Require MFA on both trusts
Set-AdfsRelyingPartyTrust -TargetName "Outlook on the web" -AccessControlPolicyName "Permit everyone and require MFA"
Set-AdfsRelyingPartyTrust -TargetName "EAC"                -AccessControlPolicyName "Permit everyone and require MFA"

# Confirm
Get-AdfsGlobalAuthenticationPolicy | Format-List AdditionalAuthenticationProvider
Get-AdfsRelyingPartyTrust | Format-Table Name,AccessControlPolicyName -AutoSize
```
Expected: `{privacyIDEAADFSProvider}`; both trusts `Permit everyone and require MFA`. Because EAC had an empty policy before, its authorization behaviour changes slightly when this is applied; this is acceptable here and is why v2 creates both trusts identically.

**GUI equivalent**

1. AD FS Management → **Service → Authentication Methods → Multi-factor Authentication → Edit** → tick *privacyIDEAADFSProvider* → Apply.
2. **Relying Party Trusts** → right-click the trust → **Edit Access Control Policy** → *Permit everyone and require MFA* → Apply. Repeat for the second trust.

#### 9.2 Pilot-group option (recommended before 2,000 users)

Do **not** use the all-users policy when most users have no token yet; every user without a token is locked out of OWA. Limit MFA to a pilot group instead. Verify the exact parameter behavior in the lab first.

`[DC]`
```powershell
New-ADGroup -Name "MFA-Pilot" -GroupScope Global -Path "OU=Groups,DC=lab,DC=com"    # adjust the OU
Add-ADGroupMember -Identity "MFA-Pilot" -Members u2
(Get-ADGroup "MFA-Pilot").SID.Value
```
`[MFADC]`
```powershell
$gsid = "<SID from above>"
Set-AdfsRelyingPartyTrust -TargetName "Outlook on the web" `
  -AccessControlPolicyName "Permit everyone and require MFA for specific group" `
  -AccessControlPolicyParameters @{ GroupParameter = $gsid }
Get-AdfsRelyingPartyTrust -Name "Outlook on the web" | Format-List AccessControlPolicyName,AccessControlPolicyParameters
```
Members of the group get the OTP step; **everyone else is not asked for MFA**, so this is a rollout stage, not a security end state.

#### 9.3 Test

1. Close all InPrivate windows, open a new one.
2. `https://mail.lab.com/owa` → `u2@lab.com` + password.
3. **An OTP page must appear.** Enter the current 6-digit code within its 30 seconds.
4. The inbox opens. **(Verified in the lab: "wow it works".)**

If no OTP page appears, first check whether you reused an old window (SSO cookie), then run the checks in §9 of this guide (Troubleshooting).

**Do not test ECP with an administrator yet.** The administrator has no token and would be locked out of ECP until Phase 10 step 1 is done (Exchange Management Shell remains unaffected).

**Gate 9:** valid OTP → inbox; invalid OTP → denied; password only → OTP page, not inbox.

---

### Phase 10 — Operations and closing the gaps

Work through these in order. Items 1–3 are needed before you call the lab finished.

**1. ECP / administrator token**

1. privacyIDEA UI → **Tokens → Enroll Token → TOTP** for the administrator account that will use ECP (a separate admin account, not `administrator` of privacyIDEA).
2. Verify with the API call from Phase 5.5 using that user name.
3. Open a new InPrivate window → `https://mail.lab.com/ecp` → password → OTP. Record the result; the lab has not tested ECP yet.
4. Keep Exchange Management Shell access independent of ECP at all times.

**2. Switch off debugging**
```powershell
$k = "HKLM:\SOFTWARE\NetKnights GmbH\PrivacyIDEA-ADFS"
(Get-ItemProperty $k).debug_log          # must be empty/0
```
If it was enabled: `Set-ItemProperty $k -Name debug_log -Value ""`, `Restart-Service adfssrv`, delete `C:\PrivacyIDEA-ADFS log.txt`.

**3. Close the protocol gap** — see §10.2. Until that is decided, state clearly that only OWA/ECP require MFA.

**4. Token-signing certificate rollover (hard deadline)**

`AutoCertificateRollover = True`, threshold 20 days: AD FS will create a new token-signing certificate about **mid-October 2027** (current one expires 7 Nov 2027). Exchange pins the old thumbprint, so OWA/ECP sign-in will break when the new certificate becomes primary unless you act. Put a reminder in the calendar for **1 October 2027**.

Procedure when the secondary certificate appears:
```powershell
# [MFADC]
Get-AdfsCertificate -CertificateType Token-Signing | Format-Table IsPrimary,Thumbprint,NotAfter
# export the NEW public certificate, import it on MAIL into Cert:\LocalMachine\Root (as in Phase 7.3)
```
```powershell
# [MAIL]  - after it is primary, replace the thumbprint (use the parameter name found in Phase 7.4)
$params = @{ AdfsIssuer="https://fs.lab.com/adfs/ls/"; AdfsAudienceUris="https://mail.lab.com/owa/","https://mail.lab.com/ecp/" }
$params[$p] = "<NEW_THUMBPRINT>"
Set-OrganizationConfig @params
iisreset /noforce
```
Alternative: disable rollover and use a CA-issued certificate you renew yourself (`Set-AdfsProperties -AutoCertificateRollover $false`, `Set-AdfsCertificate -CertificateType Token-Signing ...`). Choose one approach and document it.

**5. Backups** (test a restore)

| What | How |
|---|---|
| CA database and key | `Backup-CARoleService -Path D:\Backup\CA -Password (Read-Host -AsSecureString)` |
| AD FS | Microsoft AD FS Rapid Restore Tool or documented export; save certificates with private keys securely |
| Exchange | Exchange-aware backup of databases; save `C:\Temp\Before-*.txt` |
| privacyIDEA | DB dump, `/etc/privacyidea/` (including `enckey`, `pi.cfg`, audit keys), Apache site files |
| Provider | Version, registry values (without secrets) |

```bash
# [PI] example - database dump (credentials are in /etc/privacyidea/pi.cfg)
sudo mysqldump --single-transaction pi | gzip > /root/pi-$(date +%F).sql.gz
sudo tar czf /root/pi-etc-$(date +%F).tgz /etc/privacyidea
```
Store the encryption key separately from the database dump; one without the other cannot decrypt tokens.

**6. Monitoring** — AD FS and `privacyIDEAProvider` Application events; privacyIDEA **Audit** page; Apache `error.log`; certificate expiry (all five); OTP failure spikes.

**7. Safe shutdown and startup of the lab**

| Order | Shutdown | Startup |
|---:|---|---|
| 1 | MAIL (Exchange) | dc01 (AD, DNS, time) |
| 2 | mfadc (AD FS) | pi (check `systemctl status apache2`) |
| 3 | pi (`sudo shutdown -h now`) | mfadc (AD FS) |
| 4 | dc01 | MAIL (Exchange; allow 5–10 minutes) |

privacyIDEA starts by itself at boot because Apache and its database are enabled (`systemctl is-enabled apache2 mariadb`). After startup, check:

```bash
systemctl status apache2 --no-pager
timedatectl
```
```powershell
Test-NetConnection pi.lab.com -Port 443
```
If privacyIDEA is down, nobody can sign in to OWA/ECP (fail closed), so it must be up before users test.

---

## 7. Test matrix and evidence

Use a dedicated test user with a mailbox and an active TOTP token. Open a **new InPrivate window per test**.

| # | Test | Expected | Lab status |
|---:|---|---|---|
| 1 | Password only through AD FS, MFA off (Phase 7) | Inbox | **Pass** |
| 2 | Password + valid OTP | Inbox | **Pass** |
| 3 | Password + invalid OTP | Denied | Not recorded |
| 4 | Password + expired OTP | Denied | Not recorded |
| 5 | Wrong password | Denied at password step | Not recorded |
| 6 | Disabled AD account | Denied | Not recorded |
| 7 | User with no token | Denied (fail closed) | Not recorded |
| 8 | privacyIDEA stopped | Sign-in fails; recovery per §8 | Not recorded |
| 9 | ECP as admin with token | EAC opens | Not recorded |
| 10 | Outlook desktop / ActiveSync / EWS | **Not protected by this design** | Known gap |

**Evidence to keep** (redact user data when sharing): screenshot of OTP page and of success; screenshot of invalid OTP rejection; `Get-AdfsRelyingPartyTrust | Format-Table Name,AccessControlPolicyName`; `Get-OrganizationConfig` ADFS fields; `Get-*VirtualDirectory` authentication output; privacyIDEA audit entries; certificate subject/issuer/thumbprint (never private keys); date and test account.

---

## 8. Rollback and recovery

Run rollbacks from a console, not from OWA/ECP. Keep the Phase 0 baseline files.

### 8.1 Turn MFA off, keep AD FS password sign-in (fastest)

`[MFADC]`
```powershell
Set-AdfsRelyingPartyTrust -TargetName "Outlook on the web" -AccessControlPolicyName "Permit everyone"
Set-AdfsRelyingPartyTrust -TargetName "EAC"                -AccessControlPolicyName "Permit everyone"
Set-AdfsGlobalAuthenticationPolicy -AdditionalAuthenticationProvider @()
Restart-Service adfssrv
```
Open a new InPrivate window to test; an existing AD FS session cookie hides the change.

### 8.2 Leave AD FS entirely and return Exchange to Forms authentication

`[MAIL]`
```powershell
Set-OwaVirtualDirectory -Identity "MAIL\owa (Default Web Site)" -AdfsAuthentication $false -FormsAuthentication $true -BasicAuthentication $true
Set-EcpVirtualDirectory -Identity "MAIL\ecp (Default Web Site)" -AdfsAuthentication $false -FormsAuthentication $true -BasicAuthentication $true
iisreset /noforce
```
Compare with `C:\Temp\Before-OWA.txt` and `Before-ECP.txt`; restore any other setting that differed (for example OAuth). If `iisreset` times out use the recovery steps in Phase 7.5.

### 8.3 Remove the provider

Only after §8.1 is in effect and no policy uses it: Apps → uninstall *PrivacyIDEA AD FS* (or `msiexec /x <MSI> /L*V C:\Temp\uninstall.log`), then `Restart-Service adfssrv`.

### 8.4 Locked-out user

Fail counter exceeded or lost phone: privacyIDEA UI → **Tokens** → token → **Reset Fail Counter**, or enroll a replacement token after identity verification. A user can also be removed from the pilot group (Phase 9.2) to bypass MFA temporarily; record who approved it.

---

## 9. Troubleshooting (including every real problem from this lab)

| Symptom | Cause found / likely | Fix |
|---|---|---|
| `/validate/check` returns **HTTP 400** | Realm name wrong (`lab.com` used; real realm is `lab`) | Use `realm=lab` in tests and in the provider registry; read the response body with the `catch` block in Phase 5.5 |
| `/validate/check` → `wrong otp value`, `REJECT` with a correct-looking code | Code expired before it was sent (30 s), or a placeholder was sent literally. Time skew and wrong token are the other causes | Type the current code and run at once; compare `timedatectl` / `w32tm /query /status`; check the token in the UI; reset Fail Counter |
| `https://pi.lab.com/` returns **404** | Normal. There is no root page; TLS succeeded | None |
| TLS / certificate error calling `pi.lab.com` | mfadc does not trust the CA, or wrong SAN | Confirm `LAB-MFADC-CA` is in Trusted Root; certificate SAN `pi.lab.com`. Never use `disable_ssl` in production |
| OWA goes straight to the inbox, **no OTP page** | Provider registered but trust policy still "Permit everyone"; or an old AD FS cookie; or provider list empty after a rollback | `Get-AdfsGlobalAuthenticationPolicy`, `Get-AdfsRelyingPartyTrust \| ft Name,AccessControlPolicyName`; re-run Phase 9; new InPrivate window |
| OTP page appears but every code fails | Registry realm wrong; resolver/user mismatch; `use_upn` set | Registry `realm=lab`; leave `use_upn` empty; test API with the same user name; `Restart-Service adfssrv` after any registry change |
| OTP page shows an error about the provider | Provider cannot reach privacyIDEA or `No subscription for your client` | Application log source `privacyIDEAProvider`; `Test-NetConnection pi.lab.com -Port 443`; count users with active tokens (limit 50 without subscription) |
| `iisreset /noforce` → **0x8007041d** | IIS/WAS/Exchange service start timeout (lab VM load) | `iisreset /start`; check app pools one by one; wait 2–3 minutes; do not repeat the settings |
| `Get-WebAppPoolState -Name a,b,c` error "Cannot convert System.Object[]" | The cmdlet accepts one name | Use the `foreach` loop in Phase 7.5 |
| `Get-AdfsEndpoint` filter returns nothing | Wrong property name | Filter on `AddressPath`, not `Address` |
| `Get-AdfsCertificate … \| Format-List Subject,NotAfter` shows nothing | Those properties are on the certificate object | `(Get-AdfsCertificate -CertificateType Token-Signing).Certificate \| fl Subject,NotAfter,Thumbprint` |
| Redirect works but Exchange shows an **audience** error | Trailing slash mismatch between trust identifier and `AdfsAudienceUris` | Both `https://mail.lab.com/owa/` and `/ecp/` with the slash |
| Exchange says **invalid signature / untrusted token** | Wrong thumbprint (SSL instead of token-signing), certificate not in Exchange Trusted Root, or AD FS rolled the certificate | Compare `D074…` on both servers; re-import; update `Set-OrganizationConfig` |
| Claims / "access denied" after AD FS login | Missing SID/UPN rules; user has no mailbox | Re-check the Phase 7.2 output; `Get-Mailbox <user>` |
| `Get-SendConnector` returns nothing and CU setup warned about address space `*` | No send connector exists | Harmless for MFA; create one only if Internet mail is required |
| Remote PowerShell: "not assigned to any management roles" | RBAC | Use an account in *Organization Management* |
| Outlook desktop still asks for password only | By design: not covered | See §10.2 |
| ECP locks out the admin | Admin has no token | Enroll a token (Phase 10 step 1) or apply §8.1 temporarily; EMS is unaffected |

**Logs to collect first**

```powershell
# [MFADC]
Get-WinEvent -LogName "AD FS/Admin" -MaxEvents 15 | Format-Table TimeCreated,Id,LevelDisplayName,Message -Wrap
Get-WinEvent -LogName Application -MaxEvents 50 | Where-Object ProviderName -like "*privacyIDEA*" | Format-List TimeCreated,Message
# [MAIL]
Get-EventLog -LogName Application -EntryType Error -Newest 15 | Format-Table TimeGenerated,Source,Message -Wrap
```
```bash
# [PI]
sudo tail -n 100 /var/log/apache2/error.log
sudo journalctl -u apache2 --since "30 minutes ago"
```
privacyIDEA UI → **Audit** shows each `validate/check` with the reason.

---

## 10. Production readiness, licensing and security

### 10.1 Licensing and scale — decision required

- The AD FS provider works without a subscription only while **50 or fewer users have an active token**; beyond that, a growing share of requests fail with `No subscription for your client`. The privacyIDEA **server** itself is unrestricted open source. The count includes users with at least one active token in any realm.
- A subscription file is imported in the privacyIDEA web UI (menu *Components → Subscription*, name varies by version). A demo file for 50 users exists on the provider's GitHub release page, but it does not raise the base limit.
- **2,000 users therefore needs a NetKnights subscription (or a different validated integration).** Obtain written confirmation of price and scope before a pilot above 50 users.
- Measure peak sign-ins per minute, privacyIDEA API latency and LDAP latency before go-live.

### 10.2 Client coverage

| Client / protocol | Covered by this build? |
|---|---|
| OWA in a browser | **Yes** |
| ECP / EAC in a browser | **Yes**, after Phase 10 step 1 test |
| "Outlook on the web for devices" (mobile OWA app) | **No**, unsupported with AD FS claims auth (Microsoft) |
| Outlook for Windows (MAPI) | **No** |
| EWS, ActiveSync, POP/IMAP, SMTP AUTH | **No** |

Until each is separately handled, a stolen password still reads mail through those protocols. Choose and test one approach:

1. **Reduce exposure:** disable protocols per mailbox where not needed (`Set-CASMailbox <user> -ActiveSyncEnabled $false -EwsEnabled $false -PopEnabled $false -ImapEnabled $false`; test the impact on MAPI/Autodiscover before changing `MapiEnabled`).
2. **Network control:** publish only OWA externally; restrict MAPI/EWS/ActiveSync by firewall or reverse proxy.
3. **Modern authentication for Outlook:** Microsoft documents AD FS–based modern authentication options for on-premises Exchange in current CUs. Check Microsoft Learn for your exact CU and AD FS level and design a separate project; it is not part of this guide.

### 10.3 Exchange 2019 support

Exchange 2016/2019 left extended support on 14 Oct 2025. The final ESU period ended **October 2026**; as of 8 Nov 2026 no further security updates are published for Exchange 2019 (only Exchange Server Subscription Edition is updated). The recorded build `15.2.1748.10` is the CU15 base and carries no Security Update information. Plan an in-place upgrade to Exchange SE (the upgrade is installed like a CU; read Microsoft's prerequisites first) and re-run Gates 7 and 9 afterwards. Do not expose this server to the Internet.

### 10.4 Hardening checklist

- [ ] TLS verification stays on in the provider (`disable_ssl` empty).
- [ ] `debug_log` empty.
- [ ] Separate accounts: privacyIDEA admin, AD bind (`svc_privacyidea`, read-only), provider service account (only if challenge tokens are used).
- [ ] Restrict the privacyIDEA UI/API to admin networks and AD FS.
- [ ] ECP reachable only from admin networks.
- [ ] LDAPS with CA validation; least-privilege bind account.
- [ ] All five certificate expiry dates in the calendar (CA, fs SSL, token-signing 7 Nov 2027, mail, pi).
- [ ] Lost-phone procedure and break-glass admin documented and tested.
- [ ] Redundancy plan: second AD FS node, privacyIDEA with replicated database, tested restores.
- [ ] If a reverse proxy is added: set `trusted_proxies` before enabling `forward_client_ip`.

### 10.5 Production rollout sequence (suggested)

1. Resolve licensing (§10.1) and Exchange support (§10.3).
2. Pilot group of 10–50 users (Phase 9.2); enroll tokens; collect help-desk issues.
3. Self-service enrollment policy and a user guide.
4. Widen the group in stages; confirm token enrollment before adding a user to the group.
5. Close the protocol gap (§10.2).
6. Switch to the all-users policy only when every user has a token.

---

## 11. Appendices

### A. Quick verification commands

```powershell
# [MAIL]
Get-OrganizationConfig | Format-List AdfsIssuer,AdfsAudienceUris,AdfsSignCertificateThumbprints
Get-OwaVirtualDirectory | Format-List Server,AdfsAuthentication,FormsAuthentication,BasicAuthentication
Get-EcpVirtualDirectory | Format-List Server,AdfsAuthentication,FormsAuthentication,BasicAuthentication
Get-ChildItem Cert:\LocalMachine\Root | Where-Object Thumbprint -eq "D0743A62F7CEB205867D9F8AD0B1F18FC0D9A2F8" | Format-List Subject,NotAfter
Test-NetConnection fs.lab.com -Port 443

# [MFADC]
Get-Service adfssrv
Get-AdfsRelyingPartyTrust | Format-Table Name,Identifier,Enabled,AccessControlPolicyName -AutoSize
Get-AdfsGlobalAuthenticationPolicy | Format-List AdditionalAuthenticationProvider
Get-AdfsAuthenticationProvider | Format-Table Name
Get-ItemProperty "HKLM:\SOFTWARE\NetKnights GmbH\PrivacyIDEA-ADFS" | Format-List url,realm,use_upn,disable_ssl,debug_log
Test-NetConnection pi.lab.com -Port 443
```
```bash
# [PI]
systemctl status apache2 --no-pager
timedatectl
curl -I https://pi.lab.com
nc -vz dc01.lab.com 636
```

### B. As-built status (8 Nov 2026)

| Item | Status |
|---|---|
| Exchange 2019 CU15 `15.2.1748.10` | Done |
| AD CS Enterprise Root CA `LAB-MFADC-CA` | Done |
| AD FS `fs.lab.com`, metadata 200 | Done, verified |
| privacyIDEA 3.14 + HTTPS + LDAP resolver + realm `lab` | Done, verified |
| TOTP for `u2` (`TOTP00007431`) | Done, `ACCEPT` verified |
| RP trusts `Outlook on the web`, `EAC` + SID/UPN rules | Done, verified |
| Token-signing certificate trusted by Exchange; org config applied | Done, verified |
| OWA and ECP set to `AdfsAuthentication True` | Done, verified |
| AD FS provider v1.4.0, registry `url=https://pi.lab.com`, `realm=lab` | Done, verified |
| MFA policy `Permit everyone and require MFA` on both trusts | Done |
| **OWA password + OTP for `u2`** | **Works** |
| ECP with MFA | Not tested |
| Invalid/expired OTP, no-token user, disabled user | Not recorded |
| Exchange IIS certificate SAN `mail.lab.com` | Not recorded |
| Outlook desktop / ActiveSync / EWS | Not protected |
| Licensing for >50 users | Open |
| Exchange SE migration | Open |

### C. Change log

| Date | Change |
|---|---|
| 2026-11-08 | v1 guide created from project history |
| 2026-11-08 | v2: verified against lab output and vendor documentation; realm corrected to `lab`; MFA enforcement phase added; phase order changed to the proven sequence; Exchange 2019 support status updated; troubleshooting expanded with lab errors |

---

## 12. References

1. Microsoft Learn — *Use AD FS claims-based authentication with Outlook on the web* — https://learn.microsoft.com/en-us/exchange/clients/outlook-on-the-web/ad-fs-claims-based-auth?view=exchserver-2019
2. Microsoft Learn — *Set-OrganizationConfig* — https://learn.microsoft.com/en-us/powershell/module/exchange/set-organizationconfig
3. Microsoft Exchange Team Blog — Exchange Server security update announcements and the end of the Exchange 2016/2019 ESU program (October 2026) — https://techcommunity.microsoft.com/blog/Exchange/released-october-2026-exchange-server-security-updates/4563014
4. privacyIDEA — Ubuntu packages — https://privacyidea.readthedocs.io/en/latest/installation/ubuntu.html
5. privacyIDEA — User ID resolvers (LDAP) — https://privacyidea.readthedocs.io/en/latest/configuration/useridresolvers.html
6. privacyIDEA — Validate API — https://privacyidea.readthedocs.io/en/latest/modules/api/validate.html
7. privacyIDEA — Plugin limits (ADFS = 50 users) — https://github.com/privacyidea/privacyidea
8. privacyIDEA AD FS provider (README, registry values, releases) — https://github.com/privacyidea/adfs-provider

*Maintenance:* update thumbprints, provider release, Exchange update level and the status table whenever the lab changes. Never store passwords, OTP seeds, QR codes, recovery codes or private keys in this document.
