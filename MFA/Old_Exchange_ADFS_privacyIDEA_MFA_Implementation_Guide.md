# Exchange Server + AD FS + privacyIDEA MFA
## Complete Architecture and Implementation Guide

**Document type:** Technical design and implementation runbook  
**Lab domain:** `lab.com`  
**Documentation baseline:** 8 November 2026  
**Intended audience:** Windows / Exchange / Active Directory / Linux administrators  
**Status:** Existing lab infrastructure documented; remaining integration steps are clearly marked as pending or requiring verification.

---

## 1. Executive summary

This project integrates an on-premises Microsoft Exchange Server 2019 environment with Microsoft Active Directory Federation Services (AD FS) and privacyIDEA to provide a second authentication factor for supported web sign-in flows.

The intended web sign-in sequence is:

1. A user opens Outlook on the web (OWA).
2. Exchange redirects the browser to AD FS.
3. The user completes primary authentication with their Active Directory username and password.
4. AD FS invokes the privacyIDEA MFA provider.
5. The user enters a time-based one-time password (TOTP) from an authenticator app.
6. AD FS issues a claims token after successful authentication.
7. Exchange validates the token and provides mailbox access.

**Important scope boundary:** Microsoft's documented Exchange AD FS claims-based integration in this design is for Outlook on the web (OWA) and the Exchange admin center (EAC/ECP). It does not, by itself, enable the same AD FS MFA flow for classic Outlook desktop using MAPI/HTTP, EWS, ActiveSync, or every Exchange client protocol. Do not describe the current design as desktop Outlook MFA until a separate, supported client-authentication design has been implemented and tested.

**Important scale boundary:** privacyIDEA's current documentation states that, without a subscription file, the AD FS provider is served while no more than 50 users have active tokens. Above this threshold, a growing share of provider requests can be rejected with `No subscription for your client`. The privacyIDEA server itself and the AD FS provider have different licensing/scale considerations. A 2,000-user rollout must not proceed on the assumption that the AD FS provider is unrestricted Community Edition software. Confirm subscription and licensing with NetKnights, or redesign and validate a different supported integration before production.

---

## 2. Project goals and non-goals

### 2.1 Goals

- Centralize web authentication through AD FS.
- Use Active Directory as the identity source.
- Use privacyIDEA as the OTP/token management and validation server.
- Enroll users with TOTP authenticator applications.
- Protect OWA and ECP with password plus OTP.
- Keep the MFA server on-premises.
- Use the existing internal certificate authority and internal DNS.
- Document both GUI and command-line administration.

### 2.2 Non-goals / limitations

- This runbook does not claim that classic Outlook desktop is protected by this OWA/ECP AD FS configuration.
- It does not claim that Exchange MAPI, EWS, ActiveSync, SMTP AUTH, POP, or IMAP automatically inherit OWA MFA.
- It does not define a complete Internet publishing design. A Web Application Proxy (WAP), reverse proxy, firewall, and public DNS design must be added if external access is required.
- It does not treat a successful AD FS metadata page as proof that Exchange trusts, claims, MFA, and end-to-end mailbox access are working.
- It does not treat a single test user as proof of production scale.
- It does not store passwords, OTP seeds, private keys, or service-account secrets in this document.

---

## 3. Current lab inventory (as reported in project conversations)

### 3.1 Servers

| Role | Hostname / FQDN | IP address | Operating system | Software / role | Current recorded state |
|---|---|---:|---|---|---|
| Exchange | `MAIL` / `mail.lab.com` | `192.168.1.121` | Windows Server 2022 | Exchange Server 2019 CU15, build `15.2.1748.10` | CU15 upgrade completed; mail reported working |
| AD CS + AD FS | `mfadc.lab.com` / `fs.lab.com` | `10.10.10.12` | Windows Server 2022 | Enterprise Root CA `LAB-MFADC-CA`; AD FS | AD FS service running; metadata returned HTTP 200 |
| privacyIDEA | `pi` / `pi.lab.com` | `10.10.10.13` | Ubuntu Server 24.04.5 LTS | privacyIDEA `3.14-1noble`, Apache2 | HTTPS and LDAP resolver configured; test user TOTP reported working |
| Domain controller | `dc01.lab.com` | **Not recorded** | Not recorded | AD DS / DNS assumed from project configuration | LDAPS endpoint used by privacyIDEA |

**Do not guess the domain controller IP.** Retrieve it from DNS or the DC itself before documenting firewall rules.

### 3.2 Domain and identity settings

| Setting | Recorded value |
|---|---|
| Active Directory domain | `lab.com` |
| AD FS federation service name | `fs.lab.com` |
| AD FS display name | `LAB Federation` |
| Exchange namespace | `mail.lab.com` |
| privacyIDEA namespace | `pi.lab.com` |
| privacyIDEA LDAP server | `ldaps://dc01.lab.com:636` |
| LDAP base DN | `DC=lab,DC=com` |
| LDAP bind identity | `svc_privacyidea@lab.com` |
| privacyIDEA realm | `lab.com` |
| privacyIDEA resolver name | `LAB-AD` |
| LDAP CA file on privacyIDEA | `/etc/privacyidea/ldap-ca.crt` |
| Test user | `u2` |
| Test token | TOTP; enrollment and test reported successful |

The bind account password is intentionally omitted.

### 3.3 Recorded Exchange endpoints

| Service | URL |
|---|---|
| OWA | `https://mail.lab.com/owa` |
| ECP | `https://mail.lab.com/ecp` |
| MAPI | `https://mail.lab.com/mapi` |
| EWS | `https://mail.lab.com/EWS/Exchange.asmx` |
| Autodiscover | `https://mail.lab.com/Autodiscover` |
| OAB | `https://mail.lab.com/OAB` |
| ActiveSync | `https://mail.lab.com/Microsoft-Server-ActiveSync` |

Recorded MAPI authentication methods were `Ntlm`, `OAuth`, and `Negotiate`.

### 3.4 Certificate facts already recorded

- Internal CA: `LAB-MFADC-CA`.
- AD FS service communication certificate subject: `CN=fs.lab.com`.
- Recorded service communication certificate thumbprint:  
  `3A6175F0F8FFDBDCEB3591E9AFF48013DDE0D161`
- It had a private key and was recorded as expiring on 7 November 2028.
- Issuer was the internal `LAB-MFADC-CA`.

**Critical distinction:** the thumbprint above belongs to the recorded `CN=fs.lab.com` service communication certificate. It must not be assumed to be the AD FS **token-signing** certificate. Exchange's `AdfsSignCertificateThumbprint` setting must use the actual AD FS token-signing certificate thumbprint.

---

## 4. Architecture

### 4.1 Logical architecture

```text
                          Active Directory Domain
                                  lab.com
                                     |
                         +-----------+-----------+
                         |                       |
                  Domain Controller         Enterprise CA
                    dc01.lab.com             LAB-MFADC-CA
                         |                       |
                    LDAPS :636              Issues TLS certs
                         |                       |
                         |                 mfadc.lab.com
                         |                 Windows Server 2022
                         |                 AD FS: fs.lab.com
                         |                       |
                         |                 AD FS primary auth
                         |                 Username + password
                         |                       |
                         |                 AD FS additional MFA
                         |                 privacyIDEA AD FS provider
                         |                       |
                         |                 HTTPS validation/API
                         |                       |
                         |                 pi.lab.com
                         |                 Ubuntu 24.04
                         |                 privacyIDEA 3.14
                         |                 Apache2 + database
                         |                       |
                         |                 TOTP token validation
                         |                       |
                         +-----------------------+
                                                 |
                                         AD FS issues token
                                                 |
                                      Exchange Server 2019
                                         mail.lab.com
                                                 |
                                       OWA / ECP virtual dirs
                                                 |
                                           Mailbox / EAC
```

### 4.2 Authentication sequence

```text
Browser             Exchange OWA             AD FS             privacyIDEA             Active Directory
   |                      |                     |                    |                         |
   |---- Open /owa ------>|                     |                    |                         |
   |<--- Redirect --------|                     |                    |                         |
   |------------------------------------------->|                    |                         |
   |                      |                     |---- Validate primary credentials ----------->|
   |                      |                     |<--- Primary auth success --------------------|
   |<--------------------- AD FS sign-in / MFA challenge -------------|                         |
   |---- Submit OTP --------------------------->|                    |                         |
   |                      |                     |---- Validate OTP -->|                         |
   |                      |                     |<--- Valid / invalid-|                         |
   |<--- AD FS token / error ------------------|                    |                         |
   |---- Return token --->|                     |                    |                         |
   |<--- OWA session -----|                     |                    |                         |
```

This is a conceptual flow. Exact screens and request sequencing depend on the installed AD FS provider version and its configuration.

### 4.3 Trust boundaries

1. **Browser to Exchange / AD FS:** HTTPS; certificates must be trusted by client devices.
2. **AD FS to Active Directory:** domain authentication and directory lookup.
3. **privacyIDEA to AD:** LDAPS on TCP 636; validate the issuing CA.
4. **AD FS provider to privacyIDEA:** HTTPS; certificate validation must remain enabled.
5. **Exchange to AD FS:** token trust; Exchange must trust the AD FS token-signing certificate.
6. **Administrative plane:** restrict access to AD FS, Exchange Management Shell, privacyIDEA administration, and CA private keys.

### 4.4 DNS records

Create or verify these internal records:

| DNS name | Record | Address / target |
|---|---|---|
| `mail.lab.com` | A | `192.168.1.121` |
| `mfadc.lab.com` | A | `10.10.10.12` |
| `fs.lab.com` | A or CNAME | `10.10.10.12` |
| `pi.lab.com` | A | `10.10.10.13` |
| `dc01.lab.com` | Existing AD record | Verify actual DC address |

`fs.lab.com` is the AD FS service name. It is not interchangeable with `mfadc.lab.com` unless the federation service name, DNS, certificate SAN, SPNs, and AD FS farm configuration are deliberately changed together.

### 4.5 Required network flows

| Source | Destination | Port | Purpose |
|---|---|---:|---|
| Domain members / clients | AD DNS/DC | TCP/UDP 53 | DNS |
| Domain members / clients | DC | TCP/UDP 88 | Kerberos |
| Domain members / clients | DC | TCP/UDP 389 | LDAP / domain operations as required |
| privacyIDEA | `dc01.lab.com` | TCP 636 | LDAPS |
| Admin workstation | Exchange | TCP 443 | Exchange Admin / OWA test |
| Admin workstation | AD FS | TCP 443 | AD FS management endpoints / test |
| AD FS | privacyIDEA | TCP 443 | MFA provider API |
| Exchange | AD FS | TCP 443 | Federation / token-related communication |
| Admin workstation | servers | TCP 3389 | RDP, only where permitted |
| Servers | approved NTP source | UDP 123 | Time synchronization |

Add only flows required by the real topology. Do not expose LDAP 389/636, AD FS, privacyIDEA administration, or RDP directly to the Internet.

---

## 5. Phase workflow (implementation order)

| Phase | Workstream | Main outcome | Current project status |
|---:|---|---|---|
| 0 | Design and prerequisites | Confirm scope, naming, network, time, backup, and rollback | Partially documented |
| 1 | DNS and network foundation | All servers resolve names and reach required ports | Existing setup; verify |
| 2 | AD CS and certificates | Trusted TLS certificates for Exchange, AD FS, and privacyIDEA | CA and AD FS service cert reported complete; Exchange cert needs verification |
| 3 | Exchange baseline | Healthy Exchange CU15 and correct virtual directories | CU15 upgrade complete; verify all URLs and auth settings |
| 4 | privacyIDEA installation | Running MFA server on Ubuntu | Installed |
| 5 | privacyIDEA directory and tokens | LDAPS resolver, realm, TOTP enrollment and policy | LDAP and test TOTP reported working |
| 6 | AD FS deployment | Federation service and metadata endpoint | Installed and metadata HTTP 200 |
| 7 | privacyIDEA AD FS provider | AD FS can invoke privacyIDEA for MFA | Pending / not verified |
| 8 | Exchange AD FS claims integration | OWA/ECP trusts, claims, token-signing trust, Exchange config | Pending / must verify |
| 9 | End-to-end MFA validation | Password + OTP works on OWA/ECP | Not completed |
| 10 | Operational readiness | Monitoring, recovery, backup, scale and security | Pending |

**Recommended implementation rule:** complete and validate one phase before moving to the next. Do not enable AD FS authentication on the production-facing OWA/ECP virtual directories until the federation trust, test account, MFA provider, and rollback access are ready.

---

# 6. Detailed implementation

## Phase 0 — Design, prerequisites, and recovery planning

### Objective

Record the current state and make sure that the administrator can recover Exchange access if a federation or certificate change fails.

### GUI procedure

1. Confirm the server names and static IP addresses in Server Manager.
2. Open DNS Manager on the DNS server.
3. Verify `mail.lab.com`, `mfadc.lab.com`, `fs.lab.com`, `pi.lab.com`, and `dc01.lab.com`.
4. Confirm time synchronization on all Windows and Ubuntu servers.
5. Export current Exchange virtual-directory settings and organization settings.
6. Export AD FS relying party trusts, certificates, and global authentication policy details.
7. Take VM snapshots or application-consistent backups according to the hypervisor and application backup policy.
8. Confirm that an administrator can access Exchange Management Shell without relying on the OWA/ECP sign-in being changed.

### CLI — Windows PowerShell

Run on each Windows server:

```powershell
hostname
ipconfig /all
w32tm /query /status
Resolve-DnsName mail.lab.com
Resolve-DnsName fs.lab.com
Resolve-DnsName pi.lab.com
Resolve-DnsName dc01.lab.com
Test-NetConnection fs.lab.com -Port 443
Test-NetConnection pi.lab.com -Port 443
```

Run on the Ubuntu privacyIDEA server:

```bash
hostnamectl
ip -br address
timedatectl
getent hosts dc01.lab.com
getent hosts fs.lab.com
getent hosts mail.lab.com
nc -vz dc01.lab.com 636
nc -vz mfadc.lab.com 443
```

### Recovery checklist

- Keep a local/remote Exchange administrative route that does not depend on OWA.
- Keep a tested local administrator account on each Windows server.
- Keep the AD FS service communication certificate and private key securely backed up.
- Back up the privacyIDEA database, configuration, encryption key, and audit keys.
- Record current OWA/ECP authentication settings before changing them.
- Do not use a VM snapshot as the only backup for AD DS, CA, Exchange, or privacyIDEA.

### Acceptance criteria

- All hostnames resolve correctly from the systems that need them.
- Time is synchronized.
- Required ports are reachable.
- Current configuration and rollback access are recorded.

---

## Phase 1 — DNS and network foundation

### GUI procedure

1. Open **Server Manager → Tools → DNS** on the DNS server.
2. Expand **Forward Lookup Zones → lab.com**.
3. Verify the A record for `mail` points to `192.168.1.121`.
4. Verify the A record for `mfadc` points to `10.10.10.12`.
5. Create or verify `fs` points to `10.10.10.12`.
6. Create or verify `pi` points to `10.10.10.13`.
7. Verify the `dc01` host record.
8. On each Windows server, run `ipconfig /flushdns` after DNS changes.

### CLI — DNS records (run on a DNS management host)

First inspect the zone:

```powershell
Get-DnsServerResourceRecord -ZoneName "lab.com"
```

Create records only if they do not already exist:

```powershell
Add-DnsServerResourceRecordA -ZoneName "lab.com" -Name "mail" -IPv4Address "192.168.1.121"
Add-DnsServerResourceRecordA -ZoneName "lab.com" -Name "mfadc" -IPv4Address "10.10.10.12"
Add-DnsServerResourceRecordA -ZoneName "lab.com" -Name "fs" -IPv4Address "10.10.10.12"
Add-DnsServerResourceRecordA -ZoneName "lab.com" -Name "pi" -IPv4Address "10.10.10.13"
```

If records already exist, inspect and correct them rather than creating duplicates.

### Validation

```powershell
Resolve-DnsName mail.lab.com
Resolve-DnsName fs.lab.com
Resolve-DnsName pi.lab.com
Test-NetConnection pi.lab.com -Port 443
Test-NetConnection fs.lab.com -Port 443
```

---

## Phase 2 — AD CS and certificate deployment

### Objective

Use certificates with correct subject names/SANs and a trusted chain. The CA certificate, AD FS service communication certificate, AD FS token-signing certificate, and Exchange IIS certificate are different certificate roles.

### 2.1 CA baseline

Recorded CA:

- Server: `mfadc.lab.com`
- CA common name: `LAB-MFADC-CA`
- Type: Enterprise Root CA
- Template work: a Web Server template named `LAB-WebServer` was configured during the lab.

### GUI — verify CA and template

1. On `mfadc.lab.com`, open **Server Manager → Tools → Certification Authority**.
2. Expand the CA `LAB-MFADC-CA`.
3. Confirm the CA service is running.
4. Open **Certificate Templates**.
5. Confirm the Web Server template is published and the intended enrollment permissions are present.
6. Open **Certificate Templates Console** (`certtmpl.msc`) to inspect template subject-name and EKU settings.
7. Do not grant broad enrollment permissions without a reason.

### CLI — CA and certificate checks

```powershell
Get-Service CertSvc
certutil -config - -ping
certutil -CAInfo
```

List local certificates:

```powershell
Get-ChildItem Cert:\LocalMachine\My |
  Select-Object Subject, Issuer, Thumbprint, NotAfter, HasPrivateKey
```

### 2.2 AD FS service communication certificate

The existing lab certificate was recorded as:

```text
Subject:      CN=fs.lab.com
Thumbprint:   3A6175F0F8FFDBDCEB3591E9AFF48013DDE0D161
HasPrivateKey: True
Issuer:       LAB-MFADC-CA
```

Verify it on `mfadc.lab.com`:

```powershell
Get-ChildItem Cert:\LocalMachine\My |
  Where-Object Thumbprint -eq "3A6175F0F8FFDBDCEB3591E9AFF48013DDE0D161" |
  Format-List Subject,Issuer,Thumbprint,NotAfter,HasPrivateKey
```

Check AD FS SSL certificate binding:

```powershell
Import-Module ADFS
Get-AdfsSslCertificate
```

### 2.3 AD FS token-signing certificate — separate from SSL certificate

On `mfadc.lab.com`:

```powershell
Import-Module ADFS
Get-AdfsCertificate -CertificateType Token-Signing |
  Format-List CertificateType,IsPrimary,Thumbprint,NotAfter
```

Record both primary and secondary token-signing certificates if present. Exchange must trust the currently used signing certificate(s). The `CN=fs.lab.com` SSL certificate thumbprint above is not a substitute.

### GUI — export token-signing certificate

1. Open **AD FS Management**.
2. Expand **Service → Certificates**.
3. Under **Token-signing**, right-click the primary certificate.
4. Select **View Certificate → Details → Copy to File**.
5. Export as **Base-64 encoded X.509 (.CER)**.
6. Do not export the token-signing private key for Exchange.
7. Transfer the public certificate securely to the Exchange server.

### CLI — import public token-signing certificate on Exchange

On `MAIL`, in elevated PowerShell:

```powershell
Import-Certificate `
  -FilePath "C:\Temp\ADFS-TokenSigning.cer" `
  -CertStoreLocation "Cert:\LocalMachine\Root"
```

Verify:

```powershell
Get-ChildItem Cert:\LocalMachine\Root |
  Where-Object Subject -like "*ADFS Signing*" |
  Format-List Subject,Thumbprint,NotAfter
```

### 2.4 Exchange IIS certificate

The Exchange web certificate must include `mail.lab.com` in its Subject Alternative Name (SAN) or subject and must be trusted by client machines.

On Exchange:

```powershell
Get-ExchangeCertificate |
  Format-List Thumbprint,Subject,CertificateDomains,Services,NotAfter,Status
```

If the Exchange certificate is missing or does not include `mail.lab.com`, issue a new certificate using the correct template and SAN. Import the PFX with its private key, then assign it to IIS:

```powershell
Enable-ExchangeCertificate -Thumbprint "<EXCHANGE_IIS_CERT_THUMBPRINT>" -Services IIS
```

Use the real certificate thumbprint; do not copy the AD FS certificate thumbprint.

### GUI — Exchange certificate

1. Open Exchange Admin Center → **Servers → Certificates**.
2. Select the Exchange server.
3. Inspect the certificate subject, SANs, expiration, and assigned services.
4. Complete pending requests or import the PFX as required.
5. Assign the correct certificate to IIS.
6. Restart IIS only during an approved maintenance window if required.

### Acceptance criteria

- Clients trust the CA chain.
- `https://fs.lab.com` presents the correct AD FS service certificate.
- `https://mail.lab.com` presents a certificate valid for `mail.lab.com`.
- AD FS token-signing certificate is separately identified and its public certificate is imported into Exchange's Local Computer Trusted Root store.
- No one confuses the service communication certificate with the token-signing certificate.

---

## Phase 3 — Exchange Server baseline

### Current recorded Exchange state

- Server: `MAIL`
- FQDN: `mail.lab.com`
- IP: `192.168.1.121`
- OS: Windows Server 2022
- Exchange Server 2019 CU15
- Recorded build: `15.2.1748.10`
- Upgrade to CU15 was reported successful and mail was reported working.

The build is the CU15 base build recorded in the project. It does not, by itself, prove that the latest security update is installed.

### GUI checks

1. Open Exchange Admin Center: `https://mail.lab.com/ecp`.
2. Confirm mailbox access in OWA: `https://mail.lab.com/owa`.
3. Open **Servers → Servers** and verify the Exchange server version.
4. Open **Servers → Virtual directories** and review OWA/ECP URLs and authentication settings.
5. Verify the Exchange IIS certificate in **Servers → Certificates**.
6. Check **mail flow → send connectors**. The CU upgrade reported a warning that no Send connector had address space `*`; determine whether Internet outbound mail is expected in this lab and configure a connector only if required by the topology.

### CLI — Exchange Management Shell

Run from Exchange Management Shell:

```powershell
Get-ExchangeServer |
  Format-List Name,Edition,AdminDisplayVersion

Get-Service *Exchange* |
  Sort-Object Status,Name |
  Format-Table Status,Name -AutoSize

Get-OwaVirtualDirectory |
  Format-List Identity,InternalUrl,ExternalUrl,*Authentication*

Get-EcpVirtualDirectory |
  Format-List Identity,InternalUrl,ExternalUrl,*Authentication*

Get-WebServicesVirtualDirectory |
  Format-List Identity,InternalUrl,ExternalUrl

Get-MapiVirtualDirectory |
  Format-List Identity,InternalUrl,ExternalUrl,IISAuthenticationMethods

Get-AutodiscoverVirtualDirectory |
  Format-List Identity,InternalUrl,ExternalUrl

Get-ExchangeCertificate |
  Format-Table Thumbprint,Subject,Services,NotAfter,Status -AutoSize

Get-SendConnector |
  Format-List Name,AddressSpaces,Enabled,SmartHosts,DNSRoutingEnabled
```

### Record the current OWA/ECP authentication settings

```powershell
Get-OwaVirtualDirectory |
  Format-List Identity,AdfsAuthentication,FormsAuthentication,
              BasicAuthentication,WindowsAuthentication,OAuthAuthentication

Get-EcpVirtualDirectory |
  Format-List Identity,AdfsAuthentication,FormsAuthentication,
              BasicAuthentication,WindowsAuthentication,OAuthAuthentication
```

Save the output before changing anything.

### RBAC note

Earlier, remote Exchange PowerShell access returned an error that `LAB\Administrator` was not assigned to Exchange management roles. Later management commands succeeded. If the error returns, use an account that is legitimately assigned the required Exchange role groups; do not try to bypass RBAC.

### Acceptance criteria

- Exchange services and mailbox access work before AD FS changes.
- OWA/ECP URLs are correct.
- Exchange IIS certificate is valid.
- Current authentication configuration is saved.
- The Send Connector warning is understood and either resolved or explicitly accepted for the lab.

---

## Phase 4 — Install privacyIDEA on Ubuntu

### Current recorded state

- Host: `pi.lab.com`
- IP: `10.10.10.13`
- OS: Ubuntu Server 24.04.5 LTS
- Package: `privacyidea-apache2`
- Version reported: `3.14-1noble`
- Web server: Apache2
- HTTPS configured
- CA certificate path: `/etc/privacyidea/ldap-ca.crt` (used for LDAPS trust)

This phase is included as a reproducible installation procedure. Since privacyIDEA is already installed in the lab, do not reinstall it just to follow this document.

### 4.1 Prepare Ubuntu

```bash
sudo apt update
sudo apt full-upgrade -y
sudo hostnamectl set-hostname pi
sudo timedatectl set-ntp true
hostname -f
timedatectl
```

Ensure DNS resolves the intended FQDN:

```bash
getent hosts pi.lab.com
getent hosts dc01.lab.com
```

### 4.2 Add the privacyIDEA package repository

Download the signing key:

```bash
wget https://lancelot.netknights.it/NetKnights-Release.asc
gpg --import --import-options show-only --with-fingerprint NetKnights-Release.asc
```

Verify the published fingerprint before trusting the key:

```text
0940 4ABB EDB3 586D EDE4 AD22 00F7 0D62 AE25 0082
```

For Ubuntu 24.04, install the key and repository:

```bash
sudo mv NetKnights-Release.asc /etc/apt/trusted.gpg.d/
sudo add-apt-repository http://lancelot.netknights.it/community/noble/stable
sudo apt update
```

Install the Apache package:

```bash
sudo apt install privacyidea-apache2
```

### 4.3 Verify package and services

```bash
dpkg -l | grep -i privacyidea
privacyidea --version 2>/dev/null || true
sudo systemctl status apache2
sudo systemctl status cron
sudo ss -lntp | grep -E ':80|:443'
```

The package may configure Apache virtual hosts and disable default sites. Review the installed configuration rather than assuming a generic Apache layout.

### 4.4 Configure HTTPS

Use a certificate whose SAN contains `pi.lab.com`, issued by `LAB-MFADC-CA` or another CA trusted by AD FS and administrators.

Inspect certificate and key paths:

```bash
sudo find /etc/apache2 -type f -maxdepth 3 -print
sudo grep -R "SSLCertificateFile\|SSLCertificateKeyFile" /etc/apache2
```

After placing the certificate and private key with restrictive permissions, configure the appropriate Apache SSL virtual host. Example directives (replace paths with the actual paths used by the server):

```apache
SSLEngine on
SSLCertificateFile /etc/ssl/certs/pi.lab.com.crt
SSLCertificateKeyFile /etc/ssl/private/pi.lab.com.key
```

Enable SSL and reload Apache:

```bash
sudo a2enmod ssl
sudo apache2ctl configtest
sudo systemctl reload apache2
```

Do not use a self-signed certificate or disable certificate verification in the AD FS provider for a production environment.

### 4.5 Verify HTTPS and certificate chain

```bash
curl -I https://pi.lab.com
openssl s_client -connect pi.lab.com:443 -servername pi.lab.com -showcerts
```

If the CA is not trusted by Ubuntu, install the CA certificate into the operating system trust store:

```bash
sudo cp LAB-MFADC-CA.crt /usr/local/share/ca-certificates/LAB-MFADC-CA.crt
sudo update-ca-certificates
```

Use the actual CA certificate file, not a private key.

### GUI — privacyIDEA initial setup

1. From an administrator workstation, browse to `https://pi.lab.com`.
2. Complete the initial administrator setup if not already completed.
3. Sign in using the privacyIDEA administrator account.
4. Verify that the dashboard loads without a certificate warning.
5. Open the configuration and token areas and confirm they are accessible.
6. Do not use the administrator account as the AD FS provider's end-user authentication account.

### Acceptance criteria

- privacyIDEA service responds over HTTPS.
- Certificate is valid for `pi.lab.com`.
- Apache is healthy.
- The administrator can sign in.
- System time is synchronized.

---

## Phase 5 — privacyIDEA LDAP resolver, realm, TOTP and policies

### Objective

Use Active Directory as the user source and privacyIDEA to store token assignments and validate OTPs.

### 5.1 LDAPS prerequisites

The recorded LDAP endpoint is:

```text
ldaps://dc01.lab.com:636
Base DN: DC=lab,DC=com
Bind user: svc_privacyidea@lab.com
Resolver: LAB-AD
Realm: lab.com
```

The bind account password is not included in this document.

On Ubuntu:

```bash
nc -vz dc01.lab.com 636
openssl s_client -connect dc01.lab.com:636 -servername dc01.lab.com -showcerts
```

Verify the CA file:

```bash
sudo openssl x509 -in /etc/privacyidea/ldap-ca.crt -noout -subject -issuer -dates
```

If the LDAP certificate chain is invalid, fix the CA chain or LDAP certificate. Do not permanently disable certificate validation.

### 5.2 GUI — create the LDAP resolver

In the privacyIDEA Web UI (menu labels may vary slightly by version):

1. Open **Config → Users / UserIdResolvers**.
2. Choose **Create new resolver → LDAP Resolver**.
3. Enter a resolver name: `LAB-AD`.
4. Configure the server URI: `ldaps://dc01.lab.com:636`.
5. Set the Base DN: `DC=lab,DC=com`.
6. Set the bind identity to `svc_privacyidea@lab.com`.
7. Enter the bind password securely in the UI.
8. Set the user search scope and filter appropriate for the directory.
9. Set the login-name attribute to `sAMAccountName` or a deliberate list such as `sAMAccountName,userPrincipalName`.
10. Map user attributes such as UPN, email, given name, and surname as required.
11. Configure TLS certificate validation using the trusted CA certificate.
12. Test the resolver connection and save it.

Do not set the resolver to editable unless privacyIDEA is intentionally allowed to create or modify AD accounts.

### 5.3 CLI — LDAPS checks

From Ubuntu:

```bash
ldapsearch -H ldaps://dc01.lab.com:636 \
  -D "svc_privacyidea@lab.com" -W \
  -b "DC=lab,DC=com" \
  "(sAMAccountName=u2)" \
  sAMAccountName userPrincipalName objectSid
```

Install the client if required:

```bash
sudo apt install ldap-utils
```

If the command fails due to CA trust, fix the CA trust chain. Avoid putting the bind password directly on the command line.

### 5.4 Create the realm

In privacyIDEA:

1. Open **Config → Realms**.
2. Create realm `lab.com`.
3. Add resolver `LAB-AD`.
4. Save the realm.
5. Open the user view and confirm that AD users are visible.

### 5.5 Enroll a test TOTP token

1. Open **Tokens → Enroll Token**.
2. Select **TOTP**.
3. Select the user `u2` in realm `lab.com`.
4. Generate the QR code.
5. Scan it with a compatible authenticator application.
6. Enter the current OTP in privacyIDEA to verify enrollment.
7. Save the token assignment.
8. Confirm that the token is active and assigned to the intended user.

Use a separate test account for integration tests. Do not test by enrolling an administrator's only MFA token.

### 5.6 CLI — test OTP validation API

The privacyIDEA validation endpoint is `POST /validate/check`. Example (replace the OTP with a current value and use a trusted CA certificate; `-k` is for temporary lab diagnosis only):

```bash
curl --cacert /path/to/LAB-MFADC-CA.crt \
  -X POST "https://pi.lab.com/validate/check" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "user=u2" \
  --data-urlencode "realm=lab.com" \
  --data-urlencode "pass=<CURRENT_TOTP>"
```

A successful response should indicate a successful validation (`result.value` true). Never place a real OTP in scripts, shell history, tickets, or documentation.

### 5.7 Policies and enrollment

Create policies deliberately. At minimum, decide:

- Which realm/users can enroll TOTP.
- Which users can view or manage their own tokens.
- Which administrators can manage tokens.
- Whether token enrollment requires an administrator or self-service.
- Whether lost-token recovery requires identity verification.
- Whether authentication is denied when no token exists.
- Whether challenge-based token types are in scope.
- How failed attempts and lockouts are handled.

For the initial lab, enroll `u2` manually and test TOTP before adding more users.

### Acceptance criteria

- `LAB-AD` resolver connects over LDAPS.
- Realm `lab.com` resolves users.
- `u2` appears in privacyIDEA.
- `u2` has an active TOTP token.
- A valid OTP succeeds and an invalid/expired OTP fails.
- PrivacyIDEA logs the test authentication.

---

## Phase 6 — AD FS deployment and baseline verification

### Current recorded state

- AD FS role installed on `mfadc.lab.com`.
- Federation service name: `fs.lab.com`.
- Display name: `LAB Federation`.
- `adfssrv` service was reported Running.
- Federation metadata URL returned HTTP 200:

`https://fs.lab.com/FederationMetadata/2007-06/FederationMetadata.xml`

### 6.1 GUI — verify AD FS

1. On `mfadc.lab.com`, open **Server Manager → Tools → AD FS Management**.
2. Confirm the federation service is visible.
3. Open **Service → Certificates**.
4. Inspect Service Communications and Token-signing certificates separately.
5. Open **Service → Endpoints**.
6. Confirm the required browser sign-in endpoint is enabled.
7. Open **Authentication Policies** and inspect primary and additional authentication methods.
8. Confirm the federation service name is `fs.lab.com`.

### 6.2 CLI — verify service, properties, metadata

```powershell
Get-Service adfssrv
Import-Module ADFS
Get-AdfsProperties |
  Format-List HostName,DisplayName

Get-AdfsEndpoint -AddressPath "/adfs/ls/" |
  Format-List AddressPath,FullUrl,Enabled,Protocol

Get-AdfsCertificate -CertificateType Token-Signing |
  Format-Table IsPrimary,Thumbprint,NotAfter -AutoSize

Invoke-WebRequest `
  "https://fs.lab.com/FederationMetadata/2007-06/FederationMetadata.xml"
```

Expected metadata response: HTTP 200.

### 6.3 gMSA / service-account note

Microsoft's deployment guidance uses a group Managed Service Account (gMSA) for an AD FS farm. The project history does not contain a confirmed gMSA name or successful gMSA verification output. Do not invent one. Inspect the existing farm configuration and service identity before changing it.

On a domain controller, inspect existing service accounts:

```powershell
Get-ADServiceAccount -Filter * |
  Select-Object Name,SamAccountName,DNSHostName
```

On the AD FS server:

```powershell
Get-CimInstance Win32_Service -Filter "Name='adfssrv'" |
  Select-Object Name,StartName,State
```

If a new AD FS farm must be built, follow Microsoft's current AD FS deployment guide and plan the gMSA/KDS root key before configuration. Do not recreate an already-working farm just to match a sample account name.

### Acceptance criteria

- `adfssrv` is Running.
- `fs.lab.com` resolves to the AD FS server.
- AD FS service communication certificate is valid.
- Federation metadata returns HTTP 200.
- Token-signing certificate thumbprints are recorded.
- AD FS service identity is understood.

---

## Phase 7 — Install and configure the privacyIDEA AD FS provider

### Objective

Install the privacyIDEA AD FS authentication provider on the AD FS server and configure it to communicate with `https://pi.lab.com`.

**This phase is not recorded as completed in the previous project conversation.**

### 7.1 Obtain the provider

Official project:

`https://github.com/privacyidea/adfs-provider`

1. Review the repository README and release notes.
2. Download a release installer from the official repository's Releases page.
3. Verify the MSI's Authenticode signature before installation.
4. Record the provider version and installer hash.
5. Test the version against the installed Windows Server and AD FS version.

The repository states that .NET Framework 4.8 is required; it is included in-box on Windows Server 2022 and newer. Verify the downloaded package is an official signed release.

### 7.2 CLI — verify the installer

On `mfadc.lab.com`, after copying the MSI:

```powershell
Get-AuthenticodeSignature "C:\Temp\ADFSProvider.msi" |
  Format-List Status,StatusMessage,SignerCertificate
```

The signature status must be `Valid`, and the signer must match the publisher expected from the official release.

Install with a verbose log:

```powershell
msiexec /i "C:\Temp\ADFSProvider.msi" /L*V "C:\Temp\privacyidea-adfs-install.log"
```

Review the end of the log if installation fails.

### GUI — install

1. Log on to `mfadc.lab.com` as an authorized administrator.
2. Right-click the signed MSI and select **Install**.
3. Follow the installer.
4. Reboot only if the installer or vendor instructions require it.
5. Open **AD FS Management → Authentication Policies**.
6. Inspect the available Additional Authentication Methods.
7. Confirm that the privacyIDEA provider is listed before enabling it for a relying party.

### 7.3 Provider registry configuration

The provider repository documents its configuration under:

`HKLM\SOFTWARE\NetKnights GmbH\PrivacyIDEA-ADFS`

The exact installed provider version's README is authoritative. Common settings documented by the provider include:

| Registry value | Purpose |
|---|---|
| `url` | privacyIDEA base URL; must include `https://` |
| `disable_ssl` | Disables TLS certificate verification when set; **do not enable in production** |
| `debug_log` | Enables detailed provider logging; off by default |
| `log_path` | Optional full path for the debug log |
| `realm` | Realm appended to provider requests |
| `service_user` | privacyIDEA service account for challenge triggering / legacy enrollment functions |
| `service_pass` | Password for that service account |
| `service_realm` | Realm for the service account |
| `trigger_challenges` | Trigger challenges before login using the configured service account |
| `send_empty_pass` | Sends an empty-password validation request to trigger a challenge where configured |
| `use_upn` | Uses the Windows UPN as the privacyIDEA username |
| `tls_version` | Optional TLS version pin; TLS 1.2 is minimum |
| `forward_client_ip` | Forwards client IP to privacyIDEA |
| `trusted_proxies` | Restricts which proxy addresses may set a forwarded client IP |
| `forward_client_user_agent` | Forwards browser user-agent metadata |

Do not set every option blindly. Start with the minimum required values and the provider's current release instructions.

### 7.4 Example — minimum registry values

Run elevated PowerShell on `mfadc.lab.com`. This example sets only the base URL, realm, and UPN mapping; verify that the installed provider version uses these exact value names.

```powershell
$Key = "HKLM:\SOFTWARE\NetKnights GmbH\PrivacyIDEA-ADFS"

New-Item -Path $Key -Force | Out-Null

New-ItemProperty -Path $Key -Name "url" `
  -PropertyType String -Value "https://pi.lab.com" -Force

New-ItemProperty -Path $Key -Name "realm" `
  -PropertyType String -Value "lab.com" -Force

New-ItemProperty -Path $Key -Name "use_upn" `
  -PropertyType String -Value "1" -Force
```

If the provider version expects DWORD values for Boolean-like settings, follow that release's README and use the documented registry type. Do not assume the example types are universal across all releases.

**TLS:** keep certificate verification enabled (`disable_ssl` unset or `0`). Ensure the AD FS server trusts the CA that issued the privacyIDEA HTTPS certificate.

### 7.5 Service account and challenge settings

A privacyIDEA service account is required only for provider functions that trigger challenges using `trigger_challenges` or legacy auto-enrollment behavior. Do not use the privacyIDEA administrator account for this purpose.

If the provider's selected authentication flow needs a service account:

1. Create a dedicated, least-privilege privacyIDEA service identity.
2. Give it only the required challenge-triggering permission/policy.
3. Store its secret in the provider configuration using the provider's supported mechanism.
4. The provider README states that `service_pass` is encrypted at rest using Windows DPAPI on first read.
5. Because DPAPI encryption is machine-bound, re-enter the password on each AD FS farm node after restore, clone, migration, or new-node installation.
6. Do not export registry files containing secrets to unsecured storage.

Do not enable `send_empty_pass` or `trigger_challenges` without matching privacyIDEA policy configuration and a test that demonstrates the intended TOTP interaction.

### 7.6 Restart AD FS after provider configuration

```powershell
Restart-Service adfssrv
Get-Service adfssrv
```

### 7.7 Enable the provider as an additional authentication method

GUI:

1. Open **AD FS Management**.
2. Select **Authentication Policies**.
3. Under **Multi-factor Authentication**, open **Edit** for Additional Authentication Methods.
4. Select the privacyIDEA provider.
5. Apply the change.
6. Keep the intended primary authentication method (for this project, username/password through AD FS Forms Authentication).
7. Use an AD FS test policy or a limited test relying party before enforcing MFA broadly.

The exact GUI label may vary with AD FS version and provider release. Confirm the provider is actually registered and visible; do not assume a successful MSI installation automatically enables it.

### 7.8 Event log and debug logging

Provider errors are written to the Windows Application event log under source `privacyIDEAProvider`.

```powershell
Get-WinEvent -LogName Application -MaxEvents 100 |
  Where-Object ProviderName -like "*privacyIDEA*" |
  Format-List TimeCreated,Id,LevelDisplayName,Message
```

For temporary troubleshooting only, enable the provider's detailed debug logging according to its README. The log can contain usernames, UPNs, domains, token serials, and returned user attributes. Restrict access and retention, and disable debug logging after troubleshooting.

### Acceptance criteria

- Provider installer signature is valid.
- Provider appears in AD FS additional authentication methods.
- AD FS trusts the privacyIDEA TLS certificate.
- Provider registry configuration is minimal and correct.
- AD FS service restarts successfully.
- Provider event log has no connection or configuration errors.
- A test sign-in reaches the intended privacyIDEA OTP step.

---

## Phase 8 — Configure Exchange OWA/ECP claims-based authentication with AD FS

### Important

This is the Microsoft-documented Exchange AD FS integration path for OWA and ECP. Do this only after AD FS itself and the privacyIDEA provider have been tested. The Microsoft procedure requires the ECP virtual directory to be configured before OWA.

### 8.1 Confirm exact URLs and trailing slashes

For this lab, use these exact audience URLs consistently:

```text
https://mail.lab.com/owa/
https://mail.lab.com/ecp/
```

The relying-party identifiers and Exchange audience URIs must match exactly, including the trailing slash. Do not mix `.../owa` and `.../owa/`.

AD FS issuer:

```text
https://fs.lab.com/adfs/ls/
```

### 8.2 Create OWA and ECP relying party trusts

GUI:

1. On `mfadc.lab.com`, open **AD FS Management**.
2. Expand **Trust Relationships → Relying Party Trusts**.
3. Select **Add Relying Party Trust**.
4. Choose **Enter data about the relying party manually**.
5. Create the first trust:
   - Display name: `Outlook on the web`
   - Enable WS-Federation Passive protocol.
   - URL: `https://mail.lab.com/owa/`
   - Do not configure an optional token encryption certificate for this Exchange scenario.
   - Permit all users initially for the lab test.
6. Repeat for ECP:
   - Display name: `EAC`
   - URL: `https://mail.lab.com/ecp/`
7. Confirm both identifiers include the trailing slash.

### CLI — create the trusts

Run on `mfadc.lab.com` in elevated PowerShell:

```powershell
Import-Module ADFS

$Owa = "https://mail.lab.com/owa/"
$Ecp = "https://mail.lab.com/ecp/"

$AllowAll = '@RuleTemplate = "AllowAllAuthzRule" => issue(Type = "http://schemas.microsoft.com/authorization/claims/permit", Value = "true");'

Add-AdfsRelyingPartyTrust `
  -Name "Outlook on the web" `
  -Notes "Exchange OWA trust for https://mail.lab.com/owa/" `
  -Identifier $Owa `
  -WSFedEndpoint $Owa `
  -IssuanceAuthorizationRules $AllowAll `
  -IssueOAuthRefreshTokensTo NoDevice

Add-AdfsRelyingPartyTrust `
  -Name "EAC" `
  -Notes "Exchange ECP trust for https://mail.lab.com/ecp/" `
  -Identifier $Ecp `
  -WSFedEndpoint $Ecp `
  -IssuanceAuthorizationRules $AllowAll `
  -IssueOAuthRefreshTokensTo NoDevice
```

If the trusts already exist, do not create duplicates. Inspect and update the existing trusts.

Verify:

```powershell
Get-AdfsRelyingPartyTrust |
  Format-Table Name,Identifier,Enabled -AutoSize
```

### 8.3 Add the required claims rules

Both OWA and EAC need the AD user SID (`primarysid`) and UPN claims.

Run on `mfadc.lab.com`:

```powershell
$Rules = @'
@RuleName = "ActiveDirectoryUserSID"
c:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer == "AD AUTHORITY"]
=> issue(store = "Active Directory", types = ("http://schemas.microsoft.com/ws/2008/06/identity/claims/primarysid"), query = ";objectSID;{0}", param = c.Value);

@RuleName = "ActiveDirectoryUPN"
c:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer == "AD AUTHORITY"]
=> issue(store = "Active Directory", types = ("http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn"), query = ";userPrincipalName;{0}", param = c.Value);
'@

Set-AdfsRelyingPartyTrust `
  -TargetName "Outlook on the web" `
  -IssuanceTransformRules $Rules

Set-AdfsRelyingPartyTrust `
  -TargetName "EAC" `
  -IssuanceTransformRules $Rules
```

GUI:

1. In **AD FS Management → Relying Party Trusts**, right-click `Outlook on the web`.
2. Select **Edit Claim Rules**.
3. Under **Issuance Transform Rules**, add a rule.
4. Select **Send Claims Using a Custom Rule**.
5. Add the SID rule shown above.
6. Add another custom rule for UPN.
7. Repeat for `EAC`.
8. Save both trusts.

Verify:

```powershell
Get-AdfsRelyingPartyTrust -Name "Outlook on the web" |
  Format-List Name,Identifier,IssuanceTransformRules

Get-AdfsRelyingPartyTrust -Name "EAC" |
  Format-List Name,Identifier,IssuanceTransformRules
```

### 8.4 Export and import the AD FS token-signing certificate

On AD FS, identify the primary token-signing certificate:

```powershell
Get-AdfsCertificate -CertificateType Token-Signing |
  Format-Table IsPrimary,Thumbprint,NotAfter -AutoSize
```

Export the public `.cer` file through the AD FS Management certificate UI, then import it into `MAIL`:

```powershell
Import-Certificate `
  -FilePath "C:\Temp\ADFS-TokenSigning.cer" `
  -CertStoreLocation "Cert:\LocalMachine\Root"
```

Find the imported thumbprint on Exchange:

```powershell
Get-ChildItem Cert:\LocalMachine\Root |
  Where-Object Subject -like "*ADFS Signing*" |
  Format-List Subject,Thumbprint,NotAfter
```

If AD FS token-signing certificate rollover is enabled, document the secondary certificate and the process for updating Exchange trust before rollover.

### 8.5 Configure Exchange organization AD FS settings

Run in Exchange Management Shell on `MAIL`.

First inspect the parameters supported by the installed Exchange build:

```powershell
Get-Command Set-OrganizationConfig -Syntax
(Get-Command Set-OrganizationConfig).Parameters.Keys |
  Where-Object { $_ -like "Adfs*" }
```

Microsoft's current Exchange documentation describes the issuer and audience settings and the token-signing certificate thumbprint. Some documentation examples use the singular parameter spelling `AdfsSignCertificateThumbprint`, while the cmdlet reference also documents `AdfsSignCertificateThumbprints`. Use the parameter name exposed by the installed Exchange cmdlet.

Example using the parameter spelling in Microsoft's OWA/ECP procedure:

```powershell
Set-OrganizationConfig `
  -AdfsIssuer "https://fs.lab.com/adfs/ls/" `
  -AdfsAudienceUris "https://mail.lab.com/owa/","https://mail.lab.com/ecp/" `
  -AdfsSignCertificateThumbprint "<ACTUAL_ADFS_TOKEN_SIGNING_THUMBPRINT>"
```

If the installed cmdlet exposes only the plural form, use:

```powershell
Set-OrganizationConfig `
  -AdfsIssuer "https://fs.lab.com/adfs/ls/" `
  -AdfsAudienceUris "https://mail.lab.com/owa/","https://mail.lab.com/ecp/" `
  -AdfsSignCertificateThumbprints "<ACTUAL_ADFS_TOKEN_SIGNING_THUMBPRINT>"
```

Do not use the `3A6175F0...` service communication certificate thumbprint unless independent inspection proves that it is also the token-signing certificate (normally it is not).

Verify:

```powershell
Get-OrganizationConfig |
  Format-List AdfsIssuer,AdfsAudienceUris,AdfsSignCertificateThumbprints
```

### 8.6 Configure ECP first, then OWA

**Before executing:** save the current settings and confirm that Exchange Management Shell is available independently of OWA/ECP.

First configure ECP:

```powershell
Set-EcpVirtualDirectory `
  -Identity "MAIL\ecp (Default Web Site)" `
  -AdfsAuthentication $true `
  -BasicAuthentication $false `
  -DigestAuthentication $false `
  -FormsAuthentication $false `
  -OAuthAuthentication $false `
  -WindowsAuthentication $false
```

Then configure OWA:

```powershell
Set-OwaVirtualDirectory `
  -Identity "MAIL\owa (Default Web Site)" `
  -AdfsAuthentication $true `
  -BasicAuthentication $false `
  -DigestAuthentication $false `
  -FormsAuthentication $false `
  -OAuthAuthentication $false `
  -WindowsAuthentication $false
```

Inspect the actual identity strings first:

```powershell
Get-EcpVirtualDirectory | Format-Table Identity
Get-OwaVirtualDirectory | Format-Table Identity
```

If the identity is different, use the exact value returned by the server.

### GUI note

Exchange Admin Center exposes virtual-directory settings, but the documented AD FS configuration is performed with Exchange Management Shell. Use the shell commands for the authentication flags and retain the GUI for inspection.

### 8.7 Restart IIS on Exchange

This interrupts web services. Perform in a maintenance window.

```powershell
iisreset
```

Alternative:

```cmd
net stop w3svc /y
net start w3svc
```

### 8.8 Validate the Exchange configuration

```powershell
Get-OrganizationConfig |
  Format-List AdfsIssuer,AdfsAudienceUris,AdfsSignCertificateThumbprints

Get-EcpVirtualDirectory |
  Format-List Identity,AdfsAuthentication,FormsAuthentication,BasicAuthentication

Get-OwaVirtualDirectory |
  Format-List Identity,AdfsAuthentication,FormsAuthentication,BasicAuthentication
```

### Acceptance criteria

- AD FS trusts exist for OWA and ECP.
- Audience identifiers exactly match Exchange audience URIs.
- Both trusts issue SID and UPN claims.
- Exchange trusts the correct AD FS token-signing certificate.
- Exchange organization settings contain the correct issuer and audiences.
- ECP is configured before OWA.
- OWA/ECP redirect to AD FS and do not produce a token/claim error.

---

## Phase 9 — End-to-end MFA testing

### 9.1 Test order

Use a dedicated test account with a known-good mailbox and enrolled TOTP token.

1. Confirm the user can authenticate to AD FS with username and password.
2. Confirm AD FS invokes the privacyIDEA provider.
3. Enter a valid TOTP.
4. Confirm AD FS completes authentication.
5. Confirm OWA loads the mailbox.
6. Repeat with an invalid OTP; access must fail.
7. Repeat with an expired OTP; access must fail.
8. Confirm a user without an enrolled token is denied or follows the explicitly approved enrollment policy.
9. Check privacyIDEA audit events and AD FS Application logs.
10. Check Exchange and IIS logs if the browser returns an error.

### 9.2 Test matrix

| Test | Expected result |
|---|---|
| Correct AD password + valid OTP | Login succeeds |
| Correct AD password + invalid OTP | Login denied |
| Correct AD password + expired OTP | Login denied |
| Wrong AD password | Primary authentication denied |
| Disabled AD account | Login denied |
| User without a token | Denied or enrollment workflow, according to policy |
| privacyIDEA unavailable | Fail closed for protected sign-in; document recovery procedure |
| AD FS unavailable | OWA/ECP sign-in unavailable; recovery through approved admin path |
| Token-signing certificate changed | Exchange trust must be updated before relying on the new signing certificate |

### 9.3 Logs

AD FS service:

```powershell
Get-WinEvent -LogName Application -MaxEvents 200 |
  Where-Object ProviderName -match "AD FS|privacyIDEA" |
  Select-Object TimeCreated,ProviderName,Id,LevelDisplayName,Message
```

privacyIDEA service / Apache:

```bash
sudo systemctl status apache2
sudo journalctl -u apache2 --since "30 minutes ago"
sudo tail -n 100 /var/log/apache2/error.log
```

Use the actual privacyIDEA log location configured on the server; do not assume a path if the package has customized it.

### 9.4 Evidence to capture

- Screenshot of successful OWA sign-in after OTP.
- Screenshot of invalid OTP rejection.
- AD FS event log entry for success/failure.
- privacyIDEA audit entry for success/failure.
- Exchange virtual-directory authentication output.
- Certificate subject, issuer, and thumbprint evidence (never private keys).
- Test date/time and test account identifier.

Redact usernames and personal data before sharing logs outside the administration team.

---

## Phase 10 — Rollback and recovery

### 10.1 Before cutover

Save current values:

```powershell
Get-OrganizationConfig |
  Format-List AdfsIssuer,AdfsAudienceUris,AdfsSignCertificateThumbprints |
  Out-File C:\Temp\Before-Adfs-Exchange.txt

Get-OwaVirtualDirectory |
  Format-List *Authentication* |
  Out-File C:\Temp\Before-OWA-Authentication.txt

Get-EcpVirtualDirectory |
  Format-List *Authentication* |
  Out-File C:\Temp\Before-ECP-Authentication.txt
```

Export the relying party trust configuration and record the certificate thumbprints.

### 10.2 Restore OWA/ECP Forms Authentication

Only use this if the federation configuration causes sign-in failure and an authorized administrator has confirmed the rollback plan.

```powershell
Set-EcpVirtualDirectory `
  -Identity "MAIL\ecp (Default Web Site)" `
  -AdfsAuthentication $false `
  -FormsAuthentication $true

Set-OwaVirtualDirectory `
  -Identity "MAIL\owa (Default Web Site)" `
  -AdfsAuthentication $false `
  -FormsAuthentication $true

iisreset
```

This is a recovery example, not a universal configuration. Restore the exact pre-change settings captured in Phase 3 if they differ from the example.

### 10.3 Disable the privacyIDEA provider

1. In AD FS Management, remove the provider from the applicable additional authentication method selection or policy.
2. Restore the previous AD FS authentication policy.
3. Restart AD FS if required.
4. Do not uninstall the provider until event logs and the AD FS policy confirm it is no longer in use.

### 10.4 Rollback acceptance

- Admin access is restored.
- OWA/ECP work using the approved fallback method.
- The cause of failure is recorded.
- No certificate private key or service password is exposed during recovery.

---

# 7. Operational and security hardening

## 7.1 TOTP security

- Enforce a reasonable OTP validity window and clock synchronization.
- Do not allow a user to enroll an arbitrary token without identity verification.
- Define lost-phone and replacement-token procedures.
- Require administrator approval or verified self-service recovery.
- Monitor repeated failed OTP attempts.
- Prevent users from sharing QR codes or TOTP seeds.
- Do not store QR codes or seed values in tickets or shared folders.

## 7.2 AD FS security

- Keep AD FS patched.
- Restrict AD FS administrative access.
- Monitor token-signing certificate expiry and rollover.
- Protect the AD FS service communication private key.
- Keep MFA enabled for the intended relying parties.
- Do not disable TLS certificate verification in the privacyIDEA provider.
- Keep debug logging disabled during normal operations.
- If using a reverse proxy, configure trusted proxy IPs before trusting forwarded client IP headers.

## 7.3 privacyIDEA security

- Use HTTPS with certificate verification.
- Use LDAPS with CA verification.
- Use a least-privilege LDAP bind account.
- Separate privacyIDEA administrator and provider service accounts.
- Back up the database, configuration, encryption key, and audit keys.
- Restrict the Web UI and API to approved networks.
- Monitor audit logs and disk usage.
- Patch Ubuntu and privacyIDEA using a controlled maintenance process.

## 7.4 Exchange security

- Keep Exchange on a supported and security-updated servicing level.
- Monitor certificate expiry.
- Restrict ECP to administrators and approved management networks.
- Do not expose ECP publicly unless there is a documented business requirement and suitable access controls.
- Review all enabled authentication methods and virtual directories.
- Test every client protocol separately.

## 7.5 Time synchronization

TOTP depends on accurate time. Configure a reliable time source for domain controllers and Linux servers. Verify:

Windows:

```powershell
w32tm /query /status
w32tm /query /source
```

Ubuntu:

```bash
timedatectl
timedatectl timesync-status
```

Do not independently configure domain-joined Windows members to arbitrary NTP sources if the domain time hierarchy is intended to be authoritative.

## 7.6 Backup plan

Back up at least:

- AD CS CA database and CA private key using a documented CA backup process.
- AD FS configuration and certificates/private keys through a supported backup procedure.
- Exchange configuration and mailbox data using the organization's Exchange-aware backup solution.
- privacyIDEA database.
- `/etc/privacyidea/` configuration and relevant Apache configuration.
- privacyIDEA encryption key and audit signing keys.
- LDAP CA certificate and server certificate renewal information.
- Provider installer version, configuration values, and recovery notes (secrets protected separately).

Test restore procedures. A backup that has never been restored is not a verified recovery plan.

---

# 8. Scale and production-readiness assessment

The stated future environment is approximately 2,000 users. The lab currently demonstrates only a small test case (including `u2`).

### 8.1 Known provider limit

privacyIDEA documentation states:

- Without a subscription file, the AD FS provider is served while no more than 50 users have active tokens.
- Above the limit, an increasing share of provider requests can be rejected with `No subscription for your client`.
- Other privacyIDEA integrations have different limits; do not transfer another plugin's limit to the AD FS provider.

**Decision required before a 2,000-user pilot:** obtain written confirmation from NetKnights about the required subscription for the AD FS provider and the intended active-token count, or choose and validate a different supported integration architecture.

### 8.2 Capacity and resilience

For a production design, measure:

- Peak authentication requests per minute.
- Concurrent AD FS requests.
- privacyIDEA API response time.
- LDAP response time and DC availability.
- Database growth and backup duration.
- OTP failure rates and help-desk recovery volume.
- AD FS farm availability and privacyIDEA service availability.

A single AD FS server and a single privacyIDEA server are single points of failure. Production deployment should consider AD FS farm resilience, privacyIDEA/database backup and recovery, network redundancy, monitoring, and tested failover.

### 8.3 Client coverage decision

| Client / protocol | Covered by this documented OWA/ECP AD FS setup? |
|---|---|
| Browser OWA | Yes, when fully configured and tested |
| Browser ECP/EAC | Yes, when fully configured and tested |
| Classic Outlook desktop / MAPI | **No — separate supported design required** |
| EWS clients | Not established by this design |
| ActiveSync | Not established by this design |
| POP / IMAP | Not established by this design |
| SMTP AUTH | Not established by this design |

Do not report MFA as protecting all Exchange access until each enabled protocol has a separately verified enforcement mechanism.

---

# 9. Current project status and next actions

## 9.1 Confirmed / reported complete

- Exchange Server 2019 CU15 upgrade completed; mail reported working.
- AD CS Enterprise Root CA `LAB-MFADC-CA` installed on `mfadc.lab.com`.
- AD FS role installed on `mfadc.lab.com`.
- AD FS service `adfssrv` reported Running.
- AD FS host name `fs.lab.com` and display name `LAB Federation` recorded.
- AD FS federation metadata returned HTTP 200.
- privacyIDEA 3.14 installed on Ubuntu 24.04.5 at `pi.lab.com`.
- privacyIDEA HTTPS configured.
- privacyIDEA LDAP resolver to `ldaps://dc01.lab.com:636` configured.
- Realm `lab.com` and resolver `LAB-AD` recorded.
- AD users visible in privacyIDEA.
- Test user `u2` TOTP enrollment and validation reported successful.

## 9.2 Not yet proven complete

- Exchange IIS certificate is valid and assigned to IIS for `mail.lab.com`.
- AD FS token-signing certificate is identified and imported into Exchange.
- OWA and ECP relying party trusts exist with exact identifiers.
- OWA and ECP SID/UPN issuance transform rules are present.
- Exchange `Set-OrganizationConfig` AD FS settings are applied and verified.
- ECP and OWA virtual directories are configured for AD FS in the correct order.
- privacyIDEA AD FS provider is installed and registered.
- AD FS Additional Authentication Methods includes the provider.
- AD FS provider successfully validates a TOTP against privacyIDEA.
- End-to-end OWA MFA works.
- End-to-end ECP MFA works.
- Production subscription/scale requirement for 2,000 active-token users is resolved.
- Classic Outlook desktop MFA design is defined and tested.

## 9.3 Recommended next execution sequence

1. Run the Phase 3 Exchange baseline commands and save the output.
2. On AD FS, run `Get-AdfsCertificate -CertificateType Token-Signing` and record the actual signing thumbprint.
3. Verify the Exchange IIS certificate includes `mail.lab.com`.
4. Install and verify the signed privacyIDEA AD FS provider package.
5. Configure the provider and test its connection to privacyIDEA before changing Exchange.
6. Verify or create the OWA/ECP relying party trusts and claims.
7. Import the AD FS token-signing certificate into Exchange.
8. Configure Exchange organization AD FS settings.
9. Configure ECP first and OWA second, then restart IIS.
10. Test OWA with valid and invalid TOTP.
11. Test ECP separately.
12. Resolve the 50-active-token provider limit before any 2,000-user production rollout.

---

# 10. Troubleshooting reference

| Symptom | Likely area | First checks |
|---|---|---|
| `fs.lab.com` does not resolve | DNS | `Resolve-DnsName fs.lab.com` |
| AD FS metadata does not return 200 | AD FS / certificate / DNS | `Get-Service adfssrv`, `Get-AdfsSslCertificate`, metadata URL |
| Browser says AD FS certificate is untrusted | CA trust / certificate SAN | Inspect certificate chain and SAN; deploy CA trust |
| `mail.lab.com` is not secure | Exchange IIS certificate | `Get-ExchangeCertificate`; verify SAN and IIS assignment |
| Exchange redirects but returns audience error | URL mismatch | Compare trailing slash in trust identifier and `AdfsAudienceUris` |
| Exchange reports invalid signature | Wrong signing certificate | Compare AD FS token-signing thumbprint with Exchange trusted certificate |
| OWA/ECP claims error | Claims rules | Verify `primarysid` and `upn` rules |
| privacyIDEA user not found | LDAP resolver / realm | Check resolver test, login attribute, realm membership |
| privacyIDEA rejects all OTPs | Time / token / username mapping | Check TOTP time, realm, `use_upn`, token assignment |
| AD FS does not show privacyIDEA provider | Installer / provider registration | Verify signed MSI installation and AD FS provider registration |
| AD FS provider cannot reach privacyIDEA | DNS / TLS / firewall | Test `pi.lab.com:443`, certificate chain, provider event log |
| `No subscription for your client` | Provider subscription threshold | Check active-token count and AD FS provider subscription |
| Exchange admin access lost after cutover | AD FS / Exchange virtual directory | Use Exchange Management Shell and documented rollback |
| Outlook desktop still prompts for credentials | Client protocol scope | OWA/ECP claims integration does not automatically configure MAPI desktop MFA |

---

# 11. Reference commands — quick verification

## Exchange Management Shell

```powershell
Get-ExchangeServer | Format-List Name,AdminDisplayVersion

Get-OrganizationConfig |
  Format-List AdfsIssuer,AdfsAudienceUris,AdfsSignCertificateThumbprints

Get-OwaVirtualDirectory |
  Format-List Identity,*Authentication*

Get-EcpVirtualDirectory |
  Format-List Identity,*Authentication*

Get-ExchangeCertificate |
  Format-Table Thumbprint,Subject,CertificateDomains,Services,NotAfter -AutoSize
```

## AD FS PowerShell

```powershell
Get-Service adfssrv

Get-AdfsProperties |
  Format-List HostName,DisplayName

Get-AdfsCertificate -CertificateType Token-Signing |
  Format-Table IsPrimary,Thumbprint,NotAfter -AutoSize

Get-AdfsRelyingPartyTrust |
  Format-Table Name,Identifier,Enabled -AutoSize

Get-AdfsEndpoint -AddressPath "/adfs/ls/" |
  Format-List AddressPath,FullUrl,Enabled,Protocol
```

## Ubuntu / privacyIDEA

```bash
hostnamectl
timedatectl
dpkg -l | grep -i privacyidea
sudo systemctl status apache2
sudo ss -lntp | grep -E ':80|:443'
nc -vz dc01.lab.com 636
curl -I https://pi.lab.com
```

---

# 12. Official references

1. Microsoft Learn — **Use AD FS claims-based authentication with Outlook on the web**  
   https://learn.microsoft.com/en-us/exchange/clients/outlook-on-the-web/ad-fs-claims-based-auth?view=exchserver-2019

2. Microsoft Learn — **Set-OrganizationConfig**  
   https://learn.microsoft.com/en-us/powershell/module/exchange/set-organizationconfig?view=exchange-ps

3. privacyIDEA documentation — **Application Plugins / AD FS provider and subscription limits**  
   https://privacyidea.readthedocs.io/en/latest/application_plugins/

4. privacyIDEA documentation — **Ubuntu package installation**  
   https://privacyidea.readthedocs.io/en/latest/installation/ubuntu.html

5. privacyIDEA documentation — **LDAP user resolver**  
   https://privacyidea.readthedocs.io/en/latest/configuration/useridresolvers.html

6. privacyIDEA documentation — **Validate API**  
   https://privacyidea.readthedocs.io/en/latest/modules/api/validate.html

7. privacyIDEA GitHub — **AD FS provider README, configuration, registry values, installation and logging**  
   https://github.com/privacyidea/adfs-provider

8. privacyIDEA GitHub — **Main project**  
   https://github.com/privacyidea/privacyidea

---

## 13. Change log

| Date | Change |
|---|---|
| 2026-11-08 | Initial consolidated architecture and implementation runbook created from the available project history and current official product documentation. |
| 2026-11-08 | Added explicit 50-active-token AD FS provider limit and 2,000-user production decision. |
| 2026-11-08 | Clarified that OWA/ECP claims integration does not automatically secure classic Outlook desktop or other Exchange protocols. |
| 2026-11-08 | Distinguished AD FS service communication certificate from token-signing certificate. |

**Document maintenance:** update the inventory, certificate thumbprints, provider release, Exchange security-update level, and phase status whenever the lab changes. Never add passwords, OTP seeds, private keys, or recovery codes to this document.
