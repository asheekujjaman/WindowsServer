# AD CS (Enterprise CA) Implementation Guide — hmlocal.com

> উদ্দেশ্য: Internal IIS web server ও domain-joined PC-তে HTTPS-এর "Not secure" warning দূর করা, নিজস্ব Enterprise CA দিয়ে।
> এই document-এ যা আছে: CA install, auto-enrollment, SAN certificate (একাধিক নাম), non-domain ডিভাইসে trust, renewal, backup, troubleshooting ও নিরাপত্তা পরামর্শ।

---

## ১. পরিবেশ ও লক্ষ্য

| বিষয় | তথ্য |
|---|---|
| Domain | `hmlocal.com` |
| Client | ~৩,০০০ domain-joined PC |
| Web server | ~২০টি IIS server |
| CA type | Enterprise Root CA (single tier, online) |
| CA নাম (বাস্তবে দেখা) | `hmlocal-DC-CA` |
| Web server উদাহরণ | `web.hmlocal.com` (computer account: `WEB$`) |
| Alias নাম | `files.hmlocal.com` (একই server-এ) |
| Template ১ (auto) | `HM-Computer-TLS` |
| Template ২ (manual SAN) | `HM-WebServer` |

**লক্ষ্য:**
1. সব domain-joined PC ও server স্বয়ংক্রিয়ভাবে certificate পাবে ও CA-কে trust করবে।
2. Alias নামে (`files.hmlocal.com`) খুললেও warning আসবে না।
3. Domain-এর বাইরের ডিভাইসেও (laptop, phone) trust দেওয়া যাবে।

---

## ২. Architecture সংক্ষেপে

```
                 ┌──────────────────────────┐
                 │  Enterprise Root CA       │
                 │  hmlocal-DC-CA            │
                 └────────────┬─────────────┘
          AD-তে Root CA publish│        │Template (HM-Computer-TLS, HM-WebServer)
                              ▼        ▼
   ┌───────────────────────────────────────────────┐
   │ Domain-joined PC / Server                     │
   │  • GPO দিয়ে Root CA trust (স্বয়ংক্রিয়)        │
   │  • Auto-enrollment → Computer certificate     │
   └───────────────────────────────────────────────┘
                              │
                              ▼
   ┌───────────────────────────────────────────────┐
   │ IIS Web Server (WEB)                          │
   │  • HM-WebServer cert: SAN = web + files       │
   │  • HTTPS binding-এ bind করতে হবে (manual)      │
   └───────────────────────────────────────────────┘

   Non-domain ডিভাইস → Root CA .cer হাতে install করতে হবে
```

**মূল কথা:** Certificate-এর নাম (SAN) ও browser-এ যে URL খোলা হয় তা মিলতে হবে। Alias হলে সেই নাম SAN-এ থাকতে হবে।

---

## ৩. CA server প্রস্তুতি

1. Windows Server (2022 হলে ভালো), static IP।
2. Server-এর নাম ঠিক করুন, CA install করার পর নাম বদলানো যায় না।
3. **Enterprise Admins** group-এর member হিসেবে login করুন।

> ⚠️ **পর্যবেক্ষণ:** CA-র নাম `hmlocal-DC-CA` ও requester তালিকা দেখে মনে হচ্ছে CA role সম্ভবত Domain Controller-এই বসানো হয়েছে। এটা কাজ করে, কিন্তু best practice হলো CA আলাদা member server-এ রাখা (কারণ DC compromise হলে CA-ও যায়, আর DC demote করলে CA সমস্যায় পড়ে)। পরে সুযোগ হলে migration বিবেচনা করুন (দেখুন §১৪)।

---

## ৪. AD CS role install ও configure

### ৪.১ Install
1. Server Manager → **Manage → Add Roles and Features**
2. Role: **Active Directory Certificate Services**
3. Role Services: শুধু **Certification Authority** (CES/CEP লাগবে না)
4. Install

### ৪.২ Configure
Server Manager-এর flag থেকে **Configure Active Directory Certificate Services**:

| ধাপ | সেটিং |
|---|---|
| Role | Certification Authority |
| Setup Type | **Enterprise CA** |
| CA Type | **Root CA** |
| Private Key | Create a new private key |
| Cryptography | **RSA 4096, SHA256** |
| CA Name | যেমন `HM-Root-CA` |
| Validity | 10 বছর |
| Database/log | Default (বা আলাদা drive) |

Enterprise Root CA নিজে থেকেই AD-তে publish হয়, তাই domain-joined PC গুলো কিছুক্ষণের মধ্যে একে trust করে। আলাদা GPO লাগে না।

---

## ৫. Template ১: `HM-Computer-TLS` (auto-enrollment)

**উদ্দেশ্য:** সব domain computer (PC ও server) নিজে নিজে certificate পাবে, server-এর নিজের FQDN-এর জন্য।

`certtmpl.msc` → **Computer** template → Duplicate:

| Tab | সেটিং |
|---|---|
| Compatibility | CA: Server 2012 R2, Recipient: Windows 8.1 / Server 2012 R2 |
| General | নাম `HM-Computer-TLS`, Validity 2 years, Renewal 6 weeks |
| Cryptography | Key Storage Provider, RSA, min 2048 |
| Subject Name | **Build from this Active Directory information**, Common name, **DNS name SAN-এ include** |
| Extensions | Server Authentication + Client Authentication |
| Security | **Domain Computers**: Read, Enroll, **Autoenroll** |

`certsrv.msc` → Certificate Templates → New → **Certificate Template to Issue** → `HM-Computer-TLS`।

> Group ব্যবহার না করে Domain Computers দেওয়ার কারণ: web server গুলোকে একটা AD group-এ রাখা সম্ভব নয়, আর নতুন PC/server join করলেই certificate পেয়ে যাবে।

---

## ৬. Auto-enrollment GPO

`gpmc.msc` → নতুন GPO বানিয়ে domain (বা computer-এর OU)-এ link করুন:

**Computer Configuration → Policies → Windows Settings → Security Settings → Public Key Policies → Certificate Services Client – Auto-Enrollment**

- Configuration Model: **Enabled**
- ✅ Renew expired certificates, update pending certificates, and remove revoked certificates
- ✅ Update certificates that use certificate templates

### যাচাই
```
gpupdate /force
```
তারপর `certlm.msc` → Personal → Certificates-এ নতুন certificate এবং Trusted Root-এ CA আছে কিনা দেখুন। CA-তে `certsrv.msc` → Issued Certificates-এও entry দেখা যাবে।

---

## ৭. IIS-এ certificate bind (auto-enrollment-এর পরের কাজ)

Auto-enrollment certificate শুধু server-এর **Personal store**-এ বসায়, IIS binding নিজে বদলায় না।

1. IIS Manager → Site → **Bindings → https → Edit**
2. SSL certificate: সঠিক certificate select করুন, **View** দিয়ে SAN ও Issuer যাচাই করুন।
3. একাধিক site একই IP:443-এ থাকলে প্রতিটিতে **Host name** দিন ও **Require Server Name Indication** টিক দিন।

---

## ৮. Template ২: `HM-WebServer` (alias নামের জন্য SAN certificate)

**কেন দরকার:** `HM-Computer-TLS` শুধু server-এর নিজের নাম (`web.hmlocal.com`) দেয়। `files.hmlocal.com` খুললে নাম মেলে না, তাই warning আসে।

### ৮.১ Template বানানো
`certtmpl.msc` → **Web Server** template → Duplicate:

| Tab | সেটিং |
|---|---|
| Compatibility | CA: Server 2012 R2, Recipient: Windows 8.1 / Server 2012 R2 |
| General | নাম `HM-WebServer`, Validity 2 years |
| Cryptography | Key Storage Provider, RSA, min 2048 |
| Subject Name | **Supply in the request** |
| Extensions | Server Authentication |
| Security | নির্দিষ্ট web server-এর **computer account**-কে Read + Enroll। **Autoenroll নয়** |
| Issuance Requirements (ঐচ্ছিক) | CA certificate manager approval |

> 🔒 **নিরাপত্তা:** "Supply in the request" মানে requester যেকোনো নাম চাইতে পারে। তাই **Domain Computers-কে Enroll দেবেন না**। শুধু যে server-এ লাগবে তার computer account (যেমন `WEB$`) দিন। Manager approval চালু রাখলে আরও নিরাপদ।

`certsrv.msc` → Certificate Templates → New → Template to Issue → `HM-WebServer`।

### ৮.২ Web server থেকে request

1. `certlm.msc` → Personal → right-click → **All Tasks → Request New Certificate**
2. **Active Directory Enrollment Policy** → Next
3. `HM-WebServer` select → **Details → Properties**
4. **Subject** tab:
   - Subject name: Type **Common name**, Value `web.hmlocal.com` → **Add**
   - Alternative name: Type **DNS**, Value `web.hmlocal.com` → **Add**, তারপর `files.hmlocal.com` → **Add**
   - ⚠️ প্রতিটি নাম লিখে অবশ্যই **Add** চাপুন (না চাপলে যুক্ত হয় না, এটা সবচেয়ে সাধারণ ভুল)
5. **Private Key** tab → Key size **2048**
6. OK → **Enroll**

### ৮.৩ Manager approval থাকলে
CA-তে `certsrv.msc` → **Pending Requests** → request-এ right-click → **All Tasks → Issue**। তারপর web server-এ:
```
certreq -retrieve <RequestID> C:\web.cer
certreq -accept C:\web.cer
```

### ৮.৪ যাচাই
Issued certificate-এ double-click → **Details → Subject Alternative Name**-এ দেখা উচিত:
```
DNS Name=web.hmlocal.com
DNS Name=files.hmlocal.com
```
এরপর §৭ অনুযায়ী IIS binding-এ এই certificate দিন (একাধিক certificate থাকলে **thumbprint** মিলিয়ে নিন)।

### ৮.৫ পরীক্ষা
1. `iisreset` (Admin Command Prompt)
2. Client-এ browser পুরোপুরি বন্ধ করে নতুন InPrivate window-তে `https://files.hmlocal.com` খুলুন।

---

## ৯. Domain-এর বাইরের ডিভাইসে trust (non-domain PC, phone)

DNS resolve হলেই certificate trust হয় না। Non-domain ডিভাইস `hmlocal-DC-CA`-কে চেনে না, তাই "Not secure" আসবে। সমাধান: **Root CA-র public certificate** ডিভাইসে install করা।

### ৯.১ Root CA certificate export (GUI)
**পদ্ধতি ১:** `certsrv.msc` → CA-তে right-click → **Properties** → General → CA certificate select → **View Certificate** → Details → **Copy to File...**

**পদ্ধতি ২:** `certlm.msc` → Trusted Root Certification Authorities → CA certificate-এ right-click → All Tasks → **Export**

Wizard-এ:
- **No, do not export the private key** (ধূসর/disabled থাকাই স্বাভাবিক)
- Format: **Base-64 encoded X.509 (.CER)** (বা DER; দুটোই সব প্ল্যাটফর্মে চলে)
- File name: `HM-Root-CA.cer`

**Command line (বিকল্প):**
```
certutil -ca.cert C:\HM-Root-CA.cer
```

> ❗ কখনও private key বা `.pfx` ছড়াবেন না। শুধু `.cer` (public) যাবে।

### ৯.২ ডিভাইসে install

| ডিভাইস | পদ্ধতি |
|---|---|
| **Windows (non-domain)** | `.cer` double-click → Install Certificate → **Local Machine** → Place all certificates in the following store → **Trusted Root Certification Authorities** |
| **Android** | Settings → Security → Encryption & credentials → Install a certificate → **CA certificate** |
| **iPhone/iPad** | File খুলে profile install → তারপর Settings → General → About → **Certificate Trust Settings**-এ CA চালু করুন (এই শেষ ধাপ বাদ দিলে কাজ করবে না) |
| **Mac** | Keychain Access-এ import → certificate-এ double-click → Trust → **Always Trust** |
| **Firefox** | Settings → Privacy & Security → Certificates → View Certificates → Authorities → Import। অথবা `about:config`-এ `security.enterprise_roots.enabled = true` |

Install করার আগে file-এর **Thumbprint** CA server-এর সাথে মিলিয়ে নিন।

> ⚠️ ডিভাইসে Root CA install করা মানে ওই ডিভাইস এই CA-র সব certificate-কে trust করবে। শুধু বিশ্বস্ত ডিভাইসে করুন এবং file শুধু অফিসিয়াল চ্যানেলে বিতরণ করুন।

### ৯.৩ বড় পরিসরে বিতরণ
- Windows non-domain: Intune বা অন্য MDM দিয়ে trusted root profile push করা যায়।
- Mobile: MDM (Intune, Workspace ONE ইত্যাদি) ব্যবহার করলে প্রতিটি ফোনে হাতে install করতে হয় না।

### ৯.৪ বাইরের (অচেনা) ব্যবহারকারী হলে
গ্রাহক বা অতিথি যদি site ব্যবহার করে, সবাইকে Root CA install করানো বাস্তবসম্মত নয়। সেক্ষেত্রে **public domain + public CA certificate** (Let's Encrypt বা কেনা certificate) ব্যবহার করুন। `hmlocal.com` internal নাম হলে তার জন্য public certificate পাওয়া যায় না।

---

## ১০. Revocation (CRL/AIA) — non-domain ডিভাইসের জন্য HTTP path

ডিফল্টে CRL ও AIA-র path-এ **LDAP** থাকে, যা domain-এর বাইরের ডিভাইস পায় না। বেশিরভাগ browser site খুলতে দেয়, কিন্তু কিছু client-এ revocation error আসতে পারে। তাই একটা **HTTP** path যোগ করা ভালো।

### ধাপ (সংক্ষেপে)
1. CA server বা অন্য web server-এ IIS-এ একটা site/virtual directory বানান, যেমন `http://pki.hmlocal.com/CertEnroll`, যার physical path `C:\Windows\System32\CertSrv\CertEnroll`।
2. Internal DNS-এ `pki.hmlocal.com` record দিন।
3. `certsrv.msc` → CA → Properties → **Extensions** tab:
   - **CRL Distribution Point (CDP)**: Add → `http://pki.hmlocal.com/CertEnroll/<CaName><CRLNameSuffix><DeltaCRLAllowed>.crl` → ✅ *Include in CDP extension of issued certificates*
   - **Authority Information Access (AIA)**: Add → `http://pki.hmlocal.com/CertEnroll/<ServerDNSName>_<CaName><CertificateName>.crt` → ✅ *Include in the AIA extension of issued certificates*
4. CA service restart করুন এবং CRL publish করুন:
   ```
   net stop certsvc && net start certsvc
   certutil -crl
   ```
5. **বিদ্যমান certificate-এ নতুন path ঢোকে না**; নতুন issue বা renew-এর পর ঢুকবে। তাই `HM-WebServer` certificate আবার request করতে হবে।

> HTTP (HTTPS নয়) ব্যবহার করা হয় কারণ revocation check-এর জন্য HTTPS দিলে circular dependency তৈরি হয়। CRL file signed থাকে, তাই HTTP-তে নিরাপত্তা সমস্যা নেই।

---

## ১১. Renewal ও রক্ষণাবেক্ষণ

| Certificate | Renewal |
|---|---|
| `HM-Computer-TLS` | **স্বয়ংক্রিয়** (auto-enrollment), তবে IIS binding আবার দিতে হতে পারে |
| `HM-WebServer` (SAN) | **ম্যানুয়াল**। Supply-in-request template-এ autoenroll চলে না। মেয়াদ শেষের ১-২ মাস আগে reminder রাখুন |
| CA নিজের certificate | ১০ বছর। শেষ হওয়ার আগে renew পরিকল্পনা করুন |

### IIS binding automate করার PowerShell (ঐচ্ছিক, আগে test করুন)
Renew হওয়া নতুন certificate-টা site-এর https binding-এ বসানোর স্ক্রিপ্ট। Site-এর নাম নিজের মতো বদলে নিন:

```powershell
Import-Module WebAdministration
$siteName = "files.hmlocal.com"      # IIS site-এর নাম
$dnsName  = "files.hmlocal.com"      # SAN-এ থাকা নাম

$cert = Get-ChildItem Cert:\LocalMachine\My |
  Where-Object { $_.DnsNameList.Unicode -contains $dnsName -and $_.NotAfter -gt (Get-Date) } |
  Sort-Object NotAfter -Descending | Select-Object -First 1

if ($cert) {
  $binding = Get-WebBinding -Name $siteName -Protocol https
  $binding.AddSslCertificate($cert.Thumbprint, "My")
  Write-Output "Bound: $($cert.Thumbprint) expires $($cert.NotAfter)"
}
```

### নিয়মিত যাচাই
- `pkiview.msc` (Enterprise PKI health)
- Expiry monitoring: certificate মেয়াদ শেষের ৩০ দিন আগে alert (monitoring tool বা scheduled script দিয়ে)

---

## ১২. Backup ও Disaster Recovery

CA হারালে সব certificate trust নষ্ট হয়। তাই:

1. `certsrv.msc` → CA-তে right-click → All Tasks → **Back up CA**
   - ✅ Private key and CA certificate
   - ✅ Certificate database and certificate database log
   - Strong password দিন
2. Backup file নিরাপদ, আলাদা ও offline জায়গায় রাখুন (কমপক্ষে দুই কপি)।
3. Registry settings ও template-ও রাখুন:
   ```
   reg export HKLM\SYSTEM\CurrentControlSet\Services\CertSvc\Configuration C:\CA-config.reg
   certutil -catemplates > C:\CA-templates.txt
   ```
4. নিয়মিত restore test করুন (test VM-এ)।
5. CA-র System State/VM snapshot-ও নিন।

---

## ১৩. Troubleshooting

| সমস্যা | সম্ভাব্য কারণ | সমাধান |
|---|---|---|
| Request CA-তে আসে না | Template-এ computer account-এর Enroll নেই, বা template issue করা হয়নি | Template Security tab ও `certsrv.msc` → Certificate Templates দেখুন। `gpupdate /force`, `certutil -pulse` |
| Request Pending-এ আটকে | Manager approval চালু | CA-তে Pending Requests থেকে **Issue** দিন, তারপর `certreq -retrieve` ও `-accept` |
| `files.hmlocal.com`-এ `ERR_CERT_COMMON_NAME_INVALID` | Certificate-এ SAN-এ নামটা নেই, বা IIS-এ পুরনো certificate bind | Certificate-এর SAN দেখুন; নাম না থাকলে আবার request (প্রতিটি নাম **Add** করে)। IIS binding-এ thumbprint মিলিয়ে নতুনটা দিন |
| SAN দিলেও কাজ হয় না | Template-এ **Supply in the request** নেই | Template-এর Subject Name tab ঠিক করে আবার request |
| `ERR_CERT_AUTHORITY_INVALID` | Client Root CA trust করে না | Domain PC: `gpupdate /force`, Trusted Root-এ CA দেখুন। Non-domain: §৯ অনুযায়ী install। iPhone-এ Certificate Trust Settings চালু করুন |
| Revocation সংক্রান্ত error | CRL পাওয়া যায় না (LDAP path বাইরের ডিভাইস পায় না) | §১০ অনুযায়ী HTTP CDP যোগ করে `certutil -crl` ও certificate re-issue |
| একই নামে দুটি certificate, গুলিয়ে যাচ্ছে | Auto-enroll + manual দুটোই Personal store-এ | IIS-এ **View** করে thumbprint, SAN ও expiry যাচাই করে নিন |
| Certificate বদলালেও পুরনোটা দেখাচ্ছে | Browser/IIS cache | `iisreset`, browser পুরোপুরি বন্ধ করে InPrivate-এ test |
| Template list-এ দেখায় না | AD replication দেরি | ১৫–৩০ মিনিট অপেক্ষা, `gpupdate /force` |

### কাজের কমান্ড
```
gpupdate /force                       # policy refresh
certutil -pulse                       # auto-enrollment trigger
certutil -viewstore -enterprise NTAuth   # NTAuth store দেখা
certutil -verify -urlfetch file.cer   # chain ও CRL/AIA fetch যাচাই
certutil -CRL                         # CRL নতুন করে publish (CA server-এ)
```

---

## ১৪. Security Best Practices ও উন্নতির পরামর্শ

1. **CA আলাদা server-এ রাখুন** (DC-তে নয়)। Access শুধু PKI admin-এর।
2. **Two-tier PKI বিবেচনা করুন:** Offline Root CA + Online Issuing CA। এখন single-tier চলছে, কিন্তু Root key online থাকায় ঝুঁকি বেশি। বড় পরিসরে ভবিষ্যতে উন্নত করার কথা ভাবুন।
3. **Template permission ন্যূনতম রাখুন**, বিশেষ করে *Supply in the request* template-এ।
4. **CA Auditing চালু করুন** (CA Properties → Auditing tab ও Group Policy Audit settings)।
5. **Wildcard certificate (`*.hmlocal.com`) এড়িয়ে চলুন।** Key চুরি গেলে যেকোনো নামে impersonate সম্ভব। SAN certificate বেশি নিরাপদ।
6. **Key export না করার** অপশন রাখুন (template-এ)।
7. **Certificate Managers ও CA Administrators role** আলাদা করুন (CA Properties → Security)।
8. Private key (`.pfx`) কখনও ইমেইল বা শেয়ারে রাখবেন না।

---

## ১৫. Implementation Checklist

**CA ও মূল পরিকাঠামো**
- [ ] CA server প্রস্তুত (static IP, domain-joined, সঠিক নাম)
- [ ] AD CS role install ও Enterprise Root CA configure
- [ ] `HM-Computer-TLS` template তৈরি ও issue
- [ ] Auto-enrollment GPO তৈরি ও link
- [ ] PC ও server-এ certificate ও Root trust যাচাই

**Web server**
- [ ] IIS-এ https binding-এ certificate bind
- [ ] `HM-WebServer` template তৈরি (Supply in request) ও issue
- [ ] Web server computer account-কে Enroll permission
- [ ] SAN certificate request (সব নাম Add করে)
- [ ] Certificate SAN যাচাই ও IIS-এ bind (thumbprint মিলিয়ে)
- [ ] `iisreset` ও client-এ test

**Non-domain ডিভাইস**
- [ ] Root CA `.cer` export (private key ছাড়া)
- [ ] ডিভাইসে install ও thumbprint মেলানো
- [ ] (প্রয়োজনে) HTTP CDP/AIA যোগ

**রক্ষণাবেক্ষণ**
- [ ] CA backup (key + database + registry + template list)
- [ ] `HM-WebServer` renewal reminder (মেয়াদ শেষের আগে)
- [ ] `pkiview.msc` দিয়ে নিয়মিত health check
- [ ] Binding automation script test (ঐচ্ছিক)

---

## পরিশিষ্ট: এই implementation-এ যা শেখা গেল

1. Auto-enrollment certificate শুধু server-এর নিজের FQDN-এর জন্য, alias নাম আলাদা ব্যবস্থা লাগে।
2. Template-এ **Supply in the request** না থাকলে দেওয়া SAN উপেক্ষিত হয়।
3. SAN-এ নাম লিখে **Add** না চাপলে নামটা certificate-এ ঢোকে না।
4. একই নামে একাধিক certificate থাকলে IIS-এ ভুলটা bind হওয়া সহজ, তাই thumbprint দিয়ে নিশ্চিত হন।
5. Certificate বদলানোর পর IIS ও browser restart/clear না করলে পুরনো certificate দেখাতে পারে।
6. DNS resolve হওয়া আর certificate trust হওয়া আলাদা বিষয়, non-domain ডিভাইসে Root CA হাতে install করতে হয়।
