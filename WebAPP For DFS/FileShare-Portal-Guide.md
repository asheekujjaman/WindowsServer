# File Share Portal - Features and Implementation Guide

A web portal on IIS (HTTPS) where domain users sign in with their Active Directory account and work with files on DFS / shared folders. What each person can see and do is decided by their real NTFS permissions. Several domains and DFS paths can be added by an admin.

This guide has three parts:

1. [What the portal can do](#1-what-the-portal-can-do) - the full feature list
2. [Step-by-step implementation](#2-step-by-step-implementation) - IIS, HTTPS, first start, permissions, **LibreOffice**
3. [If you change something later](#3-if-you-change-something-later) - recipes for turning features on and off

Nothing is installed by a script. All steps are done by hand.

---

## 1. What the portal can do

### 1.1 For normal users

| Feature | What the user sees |
|---|---|
| Sign in | Choose domain, type username and password. If the site is opened with a location's web address (for example `files.lab.com`), the domain is chosen and locked automatically. |
| Company branding | Company name at the top middle of the sign-in page and in the header. Dark sign-in page with a file-server pattern; light pattern on the inside pages. |
| Locations (Home page) | One card per domain / DFS location with a live status: **Up** (green), **Degraded** (amber), **Down** (red). Refreshes every 30 seconds. Users never see share paths or server names. |
| Folder browsing | Only folders the user is allowed to open are shown (like Windows access-based enumeration). Breadcrumb path, search box, file-type icons, size and date. |
| Permission badge | "Read only" or "Read & Write" for the current folder. Buttons that the user may not use are not shown. |
| Download | Single files (resume supported) and whole folders as a zip. |
| Preview (View button) | PDF and images (png, jpg, gif, bmp, webp) in a window. Word, Excel and PowerPoint (doc, docx, xls, xlsx, ppt, pptx, odt, ods, odp, rtf) are converted to PDF on the server and shown. |
| Upload | Files and whole folders (structure kept), progress bar per file, very large files supported. Blocked file types are refused. |
| New folder | With write permission. |
| Delete | Needs write permission and a delete mode other than "Disabled". A confirmation window names the item; for folders it shows how many files are inside and asks to type the folder name. |
| Permissions page | Shows the user's groups and the folder's NTFS rules (the rules that apply to the user are highlighted). |
| Announcement banner | A message from the admin on the Home page. |

### 1.2 For administrators

An administrator is any member of **Domain Admins**, **Enterprise Admins** or local **Administrators**, or an account listed under "Extra administrators".

| Admin page | What it does |
|---|---|
| Dashboard | Active users (last 15 minutes), locations up, failed sign-ins (24 h), uploads and downloads today, server status table with uptime %, recent activity. |
| Who is signed in | User, IP address, browser, sign-in time, last activity, current folder. Sign out one user or everyone else. Clear temporary sign-in blocks. |
| Locations | Add, edit, disable, remove. Per location: domain, DFS / share path, web address, **delete mode**, optional "only these AD groups". **Test** button checks every real DFS target. |
| Audit log | Sign-ins (ok, failed, blocked), logout, timeouts, kicks, uploads, downloads, previews, deletes, admin changes, errors. Filter by type, user, days. **Export to CSV**. |
| Settings | Company name, banner, sign-in groups, idle and absolute timeouts, failed-sign-in limit and block time, **maintenance mode**, upload size limit, blocked file types, zip on/off and maximum size, preview on/off, preview size, **LibreOffice path**, recycle retention days, audit retention days, extra administrators. |
| System health | Version, memory, free disk space, running account, HTTPS check, whether LibreOffice was found, configuration **backup and restore**. |
| Recycle bin | Per location: list and restore deleted items (only used when the delete mode is "Recycle"). |

### 1.3 How "Up / Degraded / Down" works

For every location the portal finds the **real DFS targets** behind the path (the actual file servers) and tests each one on port 445 and with a share probe.

| Result | Meaning | Colour |
|---|---|---|
| Up | All targets answer | Green |
| Degraded | Some targets answer, some do not (for example "1 of 2") | Amber |
| Down | No target answers | **Red** |

A domain name alone is not used for the check, because any domain controller answers for it.

### 1.4 Security features

- HTTPS only (redirect from HTTP, HSTS), cookies are `Secure`, `HttpOnly`, `SameSite=Strict`.
- Strict browser security policy: no inline scripts or styles, no outside web addresses.
- Anti-forgery protection on every form and upload.
- Sign-in throttling **before** Active Directory is contacted (protects accounts from lockout abuse). Generic error messages (nobody can find out which user names exist).
- Passwords: used once for the Windows check, never stored, never logged, never written to a file, cookie, session or page. The password box is always emptied.
- Idle timeout and absolute timeout for sessions. Admin can sign users out at any time.
- Every action runs **as the signed-in user** (impersonation), so NTFS decides. The portal never gives extra rights.
- Path-traversal protection, safe file names, blocked upload types, no server paths or error details shown to normal users.
- Cookie keys are stored in `App_Data\keys` and encrypted with Windows DPAPI.
- Audit log with retention, CSV export with protection against spreadsheet formula injection.
- Optional Windows single sign-on (Kerberos): the password never reaches the website.

### 1.5 Not included (ideas for later)

E-mail or Event Log alerts when a location goes down, company logo upload, reordering locations, manually blocking a single user or IP address from the admin page.

---

## 2. Step-by-step implementation

Names used below (change them if you like): site folder `C:\inetpub\FileShare`, application pool `FileSharePool`, site name `FileShare`, web address `files.hm.com`.

### Step 1 - Prepare the server

1. Windows Server 2016 or newer, joined to your domain (a member server is best; a domain controller works for testing).
2. Install IIS: **Server Manager > Add Roles and Features > Web Server (IIS)**. Include *Static Content*, *Default Document*, *HTTP Errors*, *Request Filtering*. Add *Windows Authentication* only if you want single sign-on.
3. Install the **.NET 8 Hosting Bundle** (not the SDK) from `https://dotnet.microsoft.com/download/dotnet/8.0`.
4. Open PowerShell as Administrator and run:

```powershell
iisreset
```

5. Import your HTTPS certificate into **Local Computer > Personal** (certificate for `files.hm.com`).
6. Create a DNS record for `files.hm.com` that points to this server. Allow TCP 443 (and 80 for the redirect) in the firewall.

### Step 2 - Copy the files

Copy everything **inside** the `site` folder of the package into `C:\inetpub\FileShare`. Check:

```powershell
Get-ChildItem C:\inetpub\FileShare
Test-Path C:\inetpub\FileShare\web.config
Test-Path C:\inetpub\FileShare\WebTest.dll
```

Both `Test-Path` lines must say `True`. If they say `False`, the files are one folder too deep. Move them up:

```powershell
Move-Item C:\inetpub\FileShare\site\* C:\inetpub\FileShare\ -Force
Remove-Item C:\inetpub\FileShare\site
```

Make sure the two working folders exist:

```powershell
New-Item -ItemType Directory C:\inetpub\FileShare\App_Data -Force
New-Item -ItemType Directory C:\inetpub\FileShare\logs -Force
```

### Step 3 - Application pool

**IIS Manager > Application Pools > Add Application Pool**

| Setting | Value |
|---|---|
| Name | `FileSharePool` |
| .NET CLR version | **No Managed Code** |
| Managed pipeline | Integrated |

Then select the pool > **Advanced Settings**:

| Setting | Value | Why |
|---|---|---|
| Start Mode | `AlwaysRunning` | Starts without waiting for the first visitor |
| Idle Time-out (minutes) | `0` | Do not stop the site when quiet |
| Regular Time Interval (minutes) | `0` | No automatic restart (a restart signs every user out) |
| **Load User Profile** | **`True`** | **Required for LibreOffice preview** |
| Identity | `ApplicationPoolIdentity` | Fine for most setups |

The same from PowerShell (alternative to clicking):

```powershell
$a = "$env:windir\system32\inetsrv\appcmd.exe"
& $a set apppool "FileSharePool" /startMode:AlwaysRunning /processModel.idleTimeout:00:00:00 /recycling.periodicRestart.time:00:00:00 /processModel.loadUserProfile:true
```

### Step 4 - Website and HTTPS

**IIS Manager > Sites > Add Website**

| Setting | Value |
|---|---|
| Site name | `FileShare` |
| Application pool | `FileSharePool` |
| Physical path | `C:\inetpub\FileShare` |
| Binding type | `https`, port `443`, host name `files.hm.com`, tick *Require Server Name Indication*, choose your certificate |

Add a second binding: `http`, port `80`, same host name. The portal redirects it to HTTPS.

Several web addresses (for example `files.lab.com` and `files.hm.com`): add one more `https` binding for each name on the same site, each with its own certificate (or one certificate that covers all names).

If your HTTPS port is not 443, open `C:\inetpub\FileShare\appsettings.json` and change `"HttpsPort"`, then recycle the pool.

### Step 5 - Permissions on the website folder

```powershell
icacls C:\inetpub\FileShare\App_Data /grant "IIS AppPool\FileSharePool:(OI)(CI)M"
icacls C:\inetpub\FileShare\logs     /grant "IIS AppPool\FileSharePool:(OI)(CI)M"
```

Each line must end with `Successfully processed 1 files; Failed processing 0 files`. If you see *No mapping between account names and security IDs*, the application pool from Step 3 does not exist yet - create it first.

Only administrators and the pool identity should have access to `App_Data` (it holds settings, the audit log, cookie keys and the preview cache; it never holds passwords).

### Step 6 - IIS authentication

**Site > Authentication**

- Normal password sign-in: *Anonymous Authentication* = **Enabled**, *Windows Authentication* = **Disabled**.
- Single sign-on (optional, see Step 12): enable both.

### Step 7 - First start and setup

Open `https://files.hm.com`. The **Setup** page opens:

| Field | Example |
|---|---|
| Company name | `Hamim Group` |
| Location display name | `Gazipur` |
| Domain | `hmlocal.com` |
| DFS / share path | `\\hmlocal.com\Data` |
| Web address (optional) | `files.hm.com` |
| Domain admin username and password | an account in Domain Admins |

Click **Save and sign in**. You land in the Admin area.

### Step 8 - Add more domains or DFS paths

**Admin > Locations > Add location**

| Field | Example |
|---|---|
| Display name | `LAB` |
| Domain | `lab.com` |
| DFS / share path | `\\lab.com\OfficeData` |
| Web address | `files.lab.com` |

Click **Test** next to the new location. It lists every real DFS target. Requirements for another domain: the web server must resolve that domain's DNS, reach its file servers on port 445, and be a member of that domain or in a domain that trusts it.

### Step 9 - Folder permissions on the file server (NTFS)

The portal acts as the signed-in user, so set permissions with normal NTFS tools on the **file server**:

| Wanted result | Permission |
|---|---|
| Read only (browse, preview, download) | `(OI)(CI)RX` |
| Read and write (also upload, new folder, delete) | `(OI)(CI)M` |

```powershell
# example: HR folder, two groups
icacls "D:\Data\HR" /inheritance:r
icacls "D:\Data\HR" /grant "BUILTIN\Administrators:(OI)(CI)F" "SYSTEM:(OI)(CI)F"
icacls "D:\Data\HR" /grant "HM\HR-Read:(OI)(CI)RX" "HM\HR-Write:(OI)(CI)M"
```

**Important:** folders created on a drive root inherit "Authenticated Users: Modify", so everyone can write. Always remove inheritance (`/inheritance:r`) on every top-level shared folder and grant only the groups you want. A sample is in `Setup-Permissions.ps1`.

Users see only folders they can open. If a user sees an empty list, the user has no NTFS permission.

### Step 10 - Delete behaviour (recycle bin is optional)

You do **not** need the recycle bin. Each location has a delete mode. Set it in **Admin > Locations > Edit > "When a user deletes something"**:

| Mode | What happens | Needs `.recycle` folder? |
|---|---|---|
| **Do not allow deleting** | No delete button is shown. Safest while you are starting. | No |
| **Delete permanently** | Deleted at once, no undo. The confirmation window still asks (folders: type the name). | No |
| Move to the recycle bin | Item goes to a hidden `.recycle` folder, **Undo** button, admin can restore. | **Yes** (see section 3.1) |

Recommendation while you do not use the recycle bin: **Do not allow deleting**, and let administrators delete directly on the file server. If you leave the mode on "Recycle" without creating the `.recycle` folder, a delete shows an error message and nothing is deleted.

### Step 11 - LibreOffice (Word, Excel, PowerPoint preview)

**Why only PDF works today:** PDF files and images are shown by the browser itself. Word, Excel and PowerPoint cannot be shown by a browser, so the server converts them to PDF with LibreOffice first. If LibreOffice is not installed or cannot run under IIS, those files show an error in the preview window.

**11.1 Install LibreOffice on the web server (the IIS server, not your own PC)**

1. Download the Windows x64 installer from `https://www.libreoffice.org/download/download/` and run it (choose *Typical*, install for all users).
2. Check the program exists:

```powershell
Test-Path "C:\Program Files\LibreOffice\program\soffice.exe"
```

It must say `True`.

**11.2 Prove that LibreOffice itself works (as Administrator)**

Copy any small Word file to `C:\lotest\test.docx`, then:

```powershell
New-Item -ItemType Directory C:\lotest -Force | Out-Null
& "C:\Program Files\LibreOffice\program\soffice.exe" --headless --norestore --convert-to pdf --outdir C:\lotest C:\lotest\test.docx
Get-ChildItem C:\lotest
```

A file `test.pdf` must appear (the first run can take 20 to 40 seconds). If it does not, fix LibreOffice first (reinstall, check antivirus) before touching the portal.

**11.3 Make sure the application pool can run it**

```powershell
# profile must be loaded (needed by LibreOffice)
& "$env:windir\system32\inetsrv\appcmd.exe" set apppool "FileSharePool" /processModel.loadUserProfile:true

# the pool identity must be able to start LibreOffice and write the preview cache
icacls "C:\Program Files\LibreOffice" /grant "IIS AppPool\FileSharePool:(OI)(CI)RX"
icacls C:\inetpub\FileShare\App_Data /grant "IIS AppPool\FileSharePool:(OI)(CI)M"

# restart the pool (this signs users out)
& "$env:windir\system32\inetsrv\appcmd.exe" recycle apppool "FileSharePool"
```

**11.4 Tell the portal and check**

1. **Admin > Settings**: tick *Allow file preview*. If LibreOffice is not in the default folder, type the full path to `soffice.exe` in *LibreOffice program*. Save.
2. **Admin > System health**: the line *LibreOffice (Office preview)* must say **found**. ("found" only means the file exists; the next step proves it runs.)
3. Open a `.docx` in the portal and press **View**. The first conversion can take 10 to 30 seconds; later views of the same unchanged file are instant (cached).

**11.5 If it still does not work**

The preview window shows the reason in plain text:

| Message in the preview window | Meaning and fix |
|---|---|
| "Preview for Office files needs LibreOffice installed on the server" | The portal cannot find `soffice.exe`. Install it on the **web server** or set the full path in Admin > Settings. |
| "This file could not be converted for preview" | LibreOffice started but produced no PDF. Usually the pool identity cannot run it or has no profile. Do 11.3 again. Check antivirus. The file may be password protected or damaged. |
| "Preview timed out" | Conversion took more than 60 seconds. Try again (the first run is slow), check CPU load, or the file is very large. |
| "This file is too large to preview" | Raise *Maximum size of a file to preview* in Admin > Settings, or download the file instead. |
| "Preview is turned off" | Tick *Allow file preview* in Admin > Settings. |

**Quick test to find a permission problem:** temporarily run the pool as the built-in system account. If preview then works, it is a permission or profile problem of `IIS AppPool\FileSharePool`.

```powershell
$a = "$env:windir\system32\inetsrv\appcmd.exe"
& $a set apppool "FileSharePool" /processModel.identityType:LocalSystem
# ... test the preview, then put it back:
& $a set apppool "FileSharePool" /processModel.identityType:ApplicationPoolIdentity
```

Do not leave the pool on `LocalSystem` in production.

Each conversion uses roughly 200 to 400 MB of memory and at most two run at the same time.

### Step 12 - Optional: Windows single sign-on (Kerberos)

Only if you want users on domain PCs to sign in without typing a password.

1. IIS: enable **Windows Authentication** and keep **Anonymous Authentication** enabled.
2. In `appsettings.json` set `"WindowsSso": true`, then recycle the pool. The sign-in page shows **Sign in with Windows**.
3. Add an SPN for the site name: `setspn -S HTTP/files.hm.com HM\svc-fileshare` (use the web server's computer account if the pool runs as `ApplicationPoolIdentity`).
4. Add the site name to the browsers' Local intranet zone (Group Policy).
5. To reach **other** file servers as the user, configure constrained delegation in Active Directory: web server (or service account) > *Delegation* > *Trust this computer for delegation to specified services only* > add the `cifs` service of each file server.

This was not tested by the author. Test it with one user first. If it fails, use the normal sign-in.

### Step 13 - Test before you give it to users

1. Sign in with one normal user and one admin.
2. **Admin > Locations > Test** shows your real DFS targets.
3. Use two users on the same folder, one read-only and one read-write. The badges and buttons must differ.
4. Upload a file, create a folder, download a file, download a folder (zip).
5. View a PDF, an image, a Word file and an Excel file.
6. **Admin > System health > Download backup**.

---

## 3. If you change something later

### 3.1 Turn the recycle bin on later

1. Create the hidden `.recycle` folder in the **root of the location** (on the file server) with these permissions:

```powershell
$domain = "HM"
$root   = "D:\Data"
New-Item -ItemType Directory "$root\.recycle" -Force | Out-Null
attrib +h "$root\.recycle"
icacls "$root\.recycle" /inheritance:r
icacls "$root\.recycle" /grant "SYSTEM:(OI)(CI)F" "BUILTIN\Administrators:(OI)(CI)F"
icacls "$root\.recycle" /grant "${domain}\Domain Users:(WD,AD)"
```

   Users may add items but cannot list or read them.
2. Give the account that runs the automatic clean-up Modify rights on `.recycle` (the pool service account, or the web server's computer account such as `HM\WEBSERVER$`).
3. **Admin > Locations > Edit** > delete mode **Move to the recycle bin**.
4. **Admin > Settings** > *Keep deleted items for (days)*.

If a DFS folder lives on a different disk than the `.recycle` folder, moving it fails with a message; deleting is then refused (nothing is lost).

### 3.2 Switch features off

| I want to... | Change | Where |
|---|---|---|
| Stop all deleting | Delete mode = *Do not allow deleting* | Admin > Locations > Edit |
| Stop folder zip downloads | Untick *Allow folder download as zip* | Admin > Settings |
| Limit zip size | *Maximum zip size (MB)* | Admin > Settings |
| Stop file preview completely | Untick *Allow file preview* | Admin > Settings |
| Keep PDF/images only (no LibreOffice) | Leave LibreOffice uninstalled; Office files simply show the "needs LibreOffice" message | - |
| Limit upload size | *Maximum upload size per file (MB)* (0 = no limit) | Admin > Settings |
| Block file types | *Blocked upload file types* (example `.exe,.bat,.ps1`) | Admin > Settings |
| Hide a location from users | Untick *Location is enabled* | Admin > Locations > Edit |
| Allow only some groups into a location | *Only these AD groups* (example `Sales, HM\Managers`) | Admin > Locations > Edit |
| Allow only some groups to sign in | *Only these AD groups may sign in* | Admin > Settings |
| Close the portal for work | *Maintenance mode* (tick "sign out everyone who is not an administrator") | Admin > Settings |
| Show a message to everyone | *Announcement* | Admin > Settings |

### 3.3 Settings in `appsettings.json` (edit the file, then recycle the pool)

| Key | Default | Meaning |
|---|---|---|
| `RequireHttps` | `true` | Redirect HTTP to HTTPS. Set `false` only for a quick local test. |
| `HttpsPort` | `443` | Port of your HTTPS binding. |
| `LogonMode` | `Credentials` | `Credentials`: Windows keeps the user's credentials for the session, so DFS and remote servers work with no extra setup. `Network`: Windows keeps nothing (most secure) but remote file servers need Kerberos delegation. |
| `WindowsSso` | `false` | `true` shows the "Sign in with Windows" button. |
| `StatusPort` | `445` | Port used for the Up / Down check. |

### 3.4 Common changes

| I want to... | Do this |
|---|---|
| Add another domain | Admin > Locations > Add location (Step 8) |
| Change the web address or certificate | IIS Manager > Site > Bindings. Update the location's *Web address* in the portal. |
| Change the company name | Admin > Settings |
| Add an administrator | Put the account in Domain Admins, or Admin > Settings > *Extra administrators* (`HM\username`) |
| Change session length | Admin > Settings > idle minutes / absolute hours |
| Move to a new server | Install Steps 1-6 on the new server, copy `App_Data` from the old one, start the site |
| Update to a new version | Stop the site, copy the new files over `C:\inetpub\FileShare` but **keep `App_Data`**, start the site |
| Restore settings | Admin > System health > *Restore from file* (use a backup you downloaded) |
| See startup errors | In `web.config` set `stdoutLogEnabled="true"`, look in `C:\inetpub\FileShare\logs`, then set it back to `false` |
| Switch off the "Credentials" mode | Set `LogonMode` to `Network` and configure delegation (Step 12), or use Windows single sign-on |

### 3.5 Daily operation

- **Backup:** Admin > System health > *Download backup* and copy the `App_Data` folder.
- **Check the audit log** regularly (Admin > Audit log); export it to your log archive.
- **Free disk space:** keep 20 to 50 GB free on the system drive (temporary uploads, zips, preview cache). The preview cache cleans itself after 3 days.
- **Restarting IIS or recycling the pool signs everyone out.** Do it outside working hours.

---

## 4. Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| `icacls` says *cannot find the file specified* | `App_Data` or `logs` does not exist, or the files are in a nested folder (Step 2). |
| `icacls` says *No mapping between account names* | The application pool `FileSharePool` does not exist yet (Step 3). |
| HTTP 500.19 or 500.0 | Hosting Bundle not installed (Step 1), then `iisreset`. |
| HTTP 500.30 | The app could not start. Turn on the stdout log (section 3.4). |
| HTTP 404.13 on big uploads | The upload limit in `web.config` was removed. Put back `maxAllowedContentLength="4294967295"`. |
| Users are asked to sign in again and again | The pool recycled. Check Idle Time-out = 0 and Regular Time Interval = 0. |
| "Cannot reach Active Directory" | Wrong domain name, DNS, or no route to a domain controller. |
| Location shows **Down** | DFS targets do not answer on port 445. Use Admin > Locations > Test. |
| Location shows **Degraded** | One DFS target is down; users are still served by the others. |
| User sees an empty folder list | The user has no NTFS permission (hidden on purpose). |
| Everyone can write | Folders inherit "Authenticated Users: Modify". Step 9. |
| Delete shows an error | Delete mode is "Recycle" but the `.recycle` folder does not exist (use "Do not allow deleting" or "Delete permanently", or section 3.1). |
| Word / Excel preview empty or an error | Step 11. |
| Too many redirects / wrong port | Set `HttpsPort` in `appsettings.json`. |
| Sign-in says "Too many failed attempts" | Wait for the block time, or Admin > Who is signed in > *Clear sign-in blocks*. |

---

## 5. Passwords and security - honest limits

- The password travels only inside the HTTPS request, is used once for the Windows check, and is not stored or logged by the portal. Nobody can promise that a password can never be stolen.
- A password can exist in the web server's memory for a short moment until .NET cleans it. Anyone who can read the memory of the web server (its administrators) could in theory see it. Keep the web server locked down, use a dedicated member server, use HTTPS only.
- In `Credentials` mode Windows keeps the credentials for the session in its own protected store. In `Network` mode nothing is kept, but you must set up Kerberos delegation for remote file servers.
- The strongest option is Windows single sign-on (Step 12): no password reaches the website at all.
- Checklist: HTTPS with a trusted certificate - dedicated member server - `App_Data` limited to administrators and the pool identity - no "Authenticated Users / Users / Everyone: Modify" on shared folders - short list of administrators - review the audit log - keep Windows, IIS and the Hosting Bundle updated.

## 6. Sizing for 200 users

4 CPU cores, 8 GB RAM, fast disk, 1 Gbps network. Large uploads and zips are limited by the network and disk, not by the website. At most 3 zip downloads are prepared at the same time and 2 document conversions run at the same time. Sessions are kept in memory, so a restart signs everyone out.

## 7. What was tested and what was not

Tested on Linux with a simulated Windows layer and a real browser: setup, sign-in rules, status colours, hidden folders, permission-based buttons, upload, download, zip limit, delete with undo, PDF / image / Word / Excel preview (with LibreOffice), all admin pages, security headers.

**Test these first on your real servers:** real Active Directory sign-in, real NTFS enforcement, the real DFS lookup, Kerberos single sign-on, HTTPS binding and key encryption in IIS, LibreOffice under the IIS pool, PDF display in real browsers, and a load test with 200 users.
