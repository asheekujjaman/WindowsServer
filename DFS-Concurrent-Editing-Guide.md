# Getting Simultaneous, Multi-User Editing on a DFS File Share

*Internal IT reference — DFS file shares*

A working plan for Excel today, and an honest look at what's possible — and not possible — for Word, PowerPoint, and plain-text files without moving to SharePoint or a cloud drive.

> **GOAL**
> Everyone with write permission on a shared file — Excel, Word, PowerPoint, or text — should be able to edit it at the same time. No single user should lock the rest into read-only.

---

## 01. What's Actually Possible, by File Type

| File type | Simultaneous editing? | How |
|---|---|---|
| Excel (.xlsx) | **Yes — on-prem** | Built-in legacy Shared Workbook feature. No extra software. |
| Word (.docx) | **No equivalent** | Exclusive lock by default. Needs a WOPI server (e.g. self-hosted OnlyOffice/Collabora). |
| PowerPoint (.pptx) | **No equivalent** | Same as Word — needs a document server for co-authoring. |
| Plain text (.txt) | **Technically, but unsafe** | No lock at all — last save silently overwrites everyone else's changes. |

---

## 02. Excel Setup: Enable Shared Workbook (Legacy)

This is a per-file setting inside Excel itself — nothing to configure on the DFS server or in NTFS permissions beyond making sure the right people already have Modify access.

**1. Add the command to Excel**
File → Options → Quick Access Toolbar. Change "Choose commands from" to **All Commands**. Scroll to **Share Workbook (Legacy)** — note the word *(Legacy)*, distinct from other Share/Print entries nearby. Select it, click **Add >>**, then OK.

**2. Turn sharing on for the file**
Open the file from its DFS/UNC path. Click the new icon in the Quick Access Toolbar (top-left, near Save/Undo). In the **Share Workbook** dialog, Editing tab, check **"Use the old shared workbooks feature instead of the new co-authoring experience."** Click OK, then save.

**3. Confirm it took effect**
The title bar should now read the file name followed by `[Shared]`. Reopening the Share Workbook dialog lists everyone who currently has the file open — a live status view, not a settings list, so it shows just one name until someone else opens the file.

**4. Test with a second user**
Have another person open the same file from the same DFS path while you still have it open. Both of you should be able to type and save without either being pushed into read-only.

> **⚠ Common mistake**
> If clicking the toolbar icon opens "Page Setup" instead of "Share Workbook," the wrong command was added in step 1 — go back and re-select "Share Workbook (Legacy)" specifically, not a similarly-named print or sharing command.

> **⚠ If the checkbox is missing or greyed out**
> Some recent Microsoft 365 builds have removed the legacy sharing engine entirely. If so, on-prem simultaneous editing for Excel isn't available through this route — the self-hosted document server option becomes the only path.

---

## 03. Word, PowerPoint, and Text: the Real Trade-offs

There's no hidden setting for these — Microsoft never built a Word/PowerPoint equivalent of Shared Workbook. The options below are genuinely different in kind, not just in steps.

| Option | Fit | Notes |
|---|---|---|
| Self-hosted document server (OnlyOffice/Collabora) | Best for true concurrency | Files stay on DFS; server adds real-time multi-user editing for Word, Excel, PowerPoint via browser. No cloud, no SharePoint. |
| Keep single-writer locking | Safe default | No risk of overwritten work. Cost: users wait their turn. |
| Git-based version control | Text only | Real, safe merging for plain text. Not live-simultaneous — commit-then-merge. |
| Unmanaged .txt sharing | Not recommended | Zero setup, but last save silently wins. Fine for scratch notes only. |

---

## 04. Recommendation

Use Shared Workbook (Legacy) for Excel now — it costs nothing and is already configured per this guide. For Word, PowerPoint, and any file type going forward, the only way to get genuine simultaneous multi-user editing without SharePoint or a cloud drive is a self-hosted document server (OnlyOffice Docs or Collabora Online) sitting in front of the same DFS share. Everything else in this guide is a workaround, not a fix.

---

*Prepared as an internal reference. Legacy Shared Workbook is deprecated technology — test with real files (Tables, conditional formatting, merged cells) before rolling out to all users.*
