# Lab-02-Active-Directory-Identity-Lifecycle-Management

**Domain:** `redblue` / `redblue.local` &nbsp;|&nbsp; **Network:** `192.168.10.0/24` &nbsp;|&nbsp; **Date:** May 2026  
**Author:** Virginie Solomon-Bonhomme &nbsp;|&nbsp; **Status:** Complete  
**Document ID:** `LAB-02-AD-IDENTITY-2026`

---

## Table of Contents

- [What I Practiced](#what-i-practiced)
- [What This Lab Was About](#what-this-lab-was-about)
- [My Lab Setup](#my-lab-setup)
- [What I Built and Why](#what-i-built-and-why)
  - [Step 1 — Organizing Users into Folders](#step-1--organizing-users-into-folders)
  - [Step 2 — Creating Access Groups](#step-2--creating-access-groups)
  - [Step 3 — Adding Users One at a Time](#step-3--adding-users-one-at-a-time)
  - [Step 4 — Adding Users Automatically with a Script](#step-4--adding-users-automatically-with-a-script)
  - [Step 5 — Putting Users in the Right Groups](#step-5--putting-users-in-the-right-groups)
- [Common Helpdesk Tasks I Practiced](#common-helpdesk-tasks-i-practiced)
- [Checking My Work in Splunk](#checking-my-work-in-splunk)
- [How This Connects to Real Attacks](#how-this-connects-to-real-attacks)
- [What I Learned](#what-i-learned)
- [Problems I Hit and How I Fixed Them](#problems-i-hit-and-how-i-fixed-them)
- [Screenshots](#screenshots)
- [What Comes Next](#what-comes-next)

---

## What I Practiced

- Organizing user accounts into department folders
- Creating groups that control who can access what
- Adding user accounts manually through a visual interface
- Adding user accounts automatically using a script and a spreadsheet
- Disabling, deleting, and resetting passwords for accounts
- Using Splunk to confirm that every change was recorded automatically

---

## What This Lab Was About

In Lab 01, I built the domain `redblue.local` from scratch — think of a domain like a company's private network that controls who can log in and what they can do.

In this lab, I used that domain to practice the everyday tasks that IT helpdesk and security teams actually do: setting up new employees, removing old ones, resetting forgotten passwords, and making sure everything gets recorded automatically.

The big takeaway: **every single thing you do to a user account is automatically saved as a log.** Nothing is invisible. That matters both for security and for accountability.

---

## My Lab Setup

| Machine | Operating System | IP Address | What It Does |
|---|---|---|---|
| **Splunk** | Ubuntu Server 22.04 | `192.168.10.10` | Records and stores all logs |
| **ADDC01** | Windows Server 2022 | `192.168.10.7` | Manages all user accounts |
| **Target-PC** | Windows 10 | `192.168.10.100` | The workstation I worked from |

**Physical machine:** HP ENVY x360 — Intel i5, 16 GB RAM  
**All three machines are virtual** — they run as software inside my laptop using VirtualBox

---

## What I Built and Why

### Step 1 — Organizing Users into Folders

Active Directory lets you organize user accounts into folders called **OUs (Organizational Units)**. Without them, every user account sits in one big unsorted pile.

I created five folders — one per department, plus a special folder for deactivated accounts:

```
redblue.local
├── IT          — IT staff
├── Finance     — Finance staff
├── HR          — HR staff
├── Management  — Managers
└── Disabled    — Accounts that have been turned off
```

> **Why a Disabled folder?**  
> When someone leaves a company, you don't delete their account right away. You turn it off and move it to this folder. You keep it for 30 days before deleting it permanently. That way, if anyone ever asks "what did that person access before they left?" — the record still exists.

`screenshots/lab-02/ou-structure.png`

---

### Step 2 — Creating Access Groups

Instead of giving each person their own individual permissions, you put people into **groups** and give the group permissions. Everyone in the group automatically gets the same access.

This is called **role-based access control** — one of the most important ideas in IT security. The rule is simple: give people only the access they need to do their job, nothing more.

| Group Name | Folder | What This Group Can Do |
|---|---|---|
| `IT-Admins` | IT | Full admin access — IT staff need to manage the systems |
| `Finance-ReadOnly` | Finance | Can read financial data but cannot change anything |
| `HR-ReadOnly` | HR | Can read employee records but cannot change anything |
| `Management-ReadOnly` | Management | Can view data across departments but cannot change anything |

> **Why name them `Finance-ReadOnly` instead of just `Finance`?**  
> The name tells anyone looking at it exactly what access the group provides. In a large company with hundreds of groups, clear names make everything faster and safer.

`screenshots/lab-02/` — Each group inside its correct folder

---

### Step 3 — Adding Users One at a Time

I created the first four users by hand, clicking through menus in a tool called **ADUC (Active Directory Users and Computers)**. This is the visual way to add accounts.

| Name | Username | Department | Group |
|---|---|---|---|
| Steven Williams | `swilliams` | IT | `IT-Admins` |
| Sarah Chen | `s.chen` | Finance | `Finance-ReadOnly` |
| Tom Harris | `t.harris` | HR | `HR-ReadOnly` |
| James Wilson | `j.wilson` | Management | `Management-ReadOnly` |

For each account I:
- Placed them in the correct department folder
- Set a temporary starting password
- Turned on **"User must change password at next logon"**

> **Why force a password change on first login?**  
> If I set a password and the user never changes it, I still know their password — which is a security problem. Forcing them to create their own password at first login means only they know it from day one.

`screenshots/lab-02/gui-users-*.png`

---

### Step 4 — Adding Users Automatically with a Script

Creating accounts one at a time is fine for a few people. But imagine a company that hires 50 people at once — that would take hours and introduce mistakes.

The solution is a **script** — a set of instructions you write once and then run. I created a simple spreadsheet file listing four new users, then wrote a script that read the file and created all four accounts automatically.

**The spreadsheet (`users.csv`)**

```csv
FirstName,LastName,Username,Department,OU
Linda,Martinez,l.martinez,Finance,Finance
David,Thompson,d.thompson,HR,HR
Emily,Davis,e.davis,IT,IT
Robert,Brown,r.brown,Management,Management
```

**The script that reads it and creates the accounts**

```powershell
Import-Csv "C:\Users\Administrator\Desktop\users.csv" | ForEach-Object {
    New-ADUser `
        -GivenName $_.FirstName `
        -Surname $_.LastName `
        -Name "$($_.FirstName) $($_.LastName)" `
        -SamAccountName $_.Username `
        -UserPrincipalName "$($_.Username)@redblue.local" `
        -Path "OU=$($_.OU),DC=redblue,DC=local" `
        -AccountPassword (ConvertTo-SecureString "Passtheword2020" -AsPlainText -Force) `
        -ChangePasswordAtLogon $true `
        -Enabled $true
    Write-Host "Created user: $($_.FirstName) $($_.LastName)"
}
```

The script reads each row of the spreadsheet and creates one user account per row, placing each person in the correct folder automatically.

`screenshots/lab-02/powershell-bulk-import.png`

---

### Step 5 — Putting Users in the Right Groups

Creating an account and adding it to a group are two separate steps. After all accounts were created, I assigned everyone to their group using a script:

```powershell
Add-ADGroupMember -Identity "Finance-ReadOnly"    -Members "s.chen","l.martinez"
Add-ADGroupMember -Identity "HR-ReadOnly"         -Members "t.harris","d.thompson"
Add-ADGroupMember -Identity "IT-Admins"           -Members "swilliams","e.davis"
Add-ADGroupMember -Identity "Management-ReadOnly" -Members "j.wilson","r.brown"
```

Then I ran a check to confirm every group had the right people:

```powershell
$groups = @("Finance-ReadOnly","HR-ReadOnly","IT-Admins","Management-ReadOnly")
foreach ($group in $groups) {
    Write-Host "`n$group" -ForegroundColor Green
    Get-ADGroupMember -Identity $group | Select-Object Name, SamAccountName
}
```

`screenshots/lab-02/powershell-group-verification.png`

---

## Common Helpdesk Tasks I Practiced

These are three of the most common tickets any IT helpdesk handles every day.

### Task 1 — Turning Off an Account: Tom Harris

**Situation:** Tom Harris left the company.

**What I did:**
1. Disabled his account — he can no longer log in
2. Moved his account to the `Disabled` folder
3. It stays there for 30 days before being permanently deleted

> Deleting an account immediately destroys all records of what that person did. Keeping a disabled account for 30 days protects the company if questions come up later.

`screenshots/lab-02/tom-harris-disabled-ou.png`

---

### Task 2 — Resetting a Password: Sarah Chen

**Situation:** Sarah Chen forgot her password and is locked out.

**What I did:**
1. Set a new temporary password for her in ADUC
2. Turned on **"User must change password at next logon"**

> This way I give her a temporary way back in, but she immediately replaces it with something only she knows. The helpdesk never has long-term access to a user's password.

`screenshots/lab-02/sarah-chen-password-reset.png`

---

### Task 3 — Deleting an Account: Linda Martinez

**Situation:** Linda Martinez's account had been disabled for 30 days and was approved for permanent deletion.

**What I did:**
1. Deleted the account from ADUC
2. Confirmed the Finance folder no longer shows her account

> Deleting is always the last step, never the first. The correct order is: disable → move to Disabled folder → wait 30 days → delete with approval.

`screenshots/lab-02/linda-martinez-deleted.png`

---

## Checking My Work in Splunk

This was one of the most important things I learned. **Every action I took in Active Directory automatically created a log entry in Splunk.** I did not have to do anything special to make this happen — it just works.

This means in a real company, nothing you do to a user account is invisible. Every change leaves a record.

### What Each Log Code Means

| Code | What Happened | When I Triggered It |
|---|---|---|
| `4720` | A new account was created | When I created all 8 users |
| `4722` | An account was turned on | When the script-created accounts were enabled |
| `4725` | An account was turned off | When I disabled Tom Harris |
| `4726` | An account was deleted | When I deleted Linda Martinez |
| `4728` | A user was added to a group | Every group assignment |
| `4723` | A password was reset | When I reset Sarah Chen's password |

### How I Searched for These in Splunk

**See every account that was created:**
```spl
index=endpoint host=ADDC01 EventCode=4720
| table _time, EventCode, Account_Name, ComputerName
| sort -_time
```

**See every group change:**
```spl
index=endpoint host=ADDC01 EventCode=4728
| table _time, EventCode, Account_Name, ComputerName
| sort -_time
```

**See every account that was disabled:**
```spl
index=endpoint host=ADDC01 EventCode=4725
| table _time, EventCode, Account_Name, ComputerName
| sort -_time
```

**See every account that was deleted:**
```spl
index=endpoint host=ADDC01 EventCode=4726
| table _time, EventCode, Account_Name, ComputerName
| sort -_time
```

**See everything from this lab in one view:**
```spl
index=endpoint host=ADDC01
(EventCode=4720 OR EventCode=4722 OR EventCode=4725 OR EventCode=4726 OR EventCode=4728)
| table _time, EventCode, Account_Name, ComputerName
| sort -_time
```

`screenshots/lab-02/splunk-lab02-all-events.png`

---

## How This Connects to Real Attacks

This lab was about defense — building and managing accounts properly. But everything I built here connects to the attacks I studied in Lab 01. Understanding what I built helps explain why those attacks were dangerous.

| What I Built | Why Attackers Go After It | Lab 01 Attack |
|---|---|---|
| The domain controller running on ADDC01 | It stores every password on the network | `T1003` — attacker pulled password data from memory |
| `swilliams` account with a weak password | Easy passwords are easy to guess | `T1110` — a tool called Hydra guessed the password automatically |
| The Domain Admins group | Members of this group control everything | `T1078` — attackers look for high-privilege accounts to take over |
| Remote access (RDP) turned on | Lets people log in from anywhere — including attackers | `T1021.001` — tools called Crowbar and Hydra attacked the remote login |

---

## What I Learned

**Folders do more than organize — they control security rules.**  
When I created the department folders, I was not just tidying things up. Folders in Active Directory are where security policies get applied. If Finance users need a stricter password policy than IT staff, you set that on the folder and it applies automatically to everyone inside.

**Name your groups so anyone can understand them.**  
`Finance-ReadOnly` tells you exactly what it does. `Finance-Users` tells you nothing about access. Good names make security audits faster and mistakes less likely.

**Scripts save enormous amounts of time.**  
Four users by hand took several minutes. The script did four users in seconds. At the scale of a real company, this is not optional — it is the only practical way to work.

**The Disabled folder is a real security practice.**  
It is not just about being organized. It gives you a clear place to track who has been deactivated, confirm they truly cannot log in, and know exactly when their 30-day window expires.

**Everything is logged automatically.**  
Before this lab I knew logs existed. After this lab I understood what that actually means in practice. Every single thing I did showed up in Splunk on its own, within seconds. There is no way to hide changes to user accounts.

**Disable first, delete last.**  
This sequence exists for a reason: Disable → move to Disabled folder → wait 30 days → delete with approval. Skipping steps either creates a security hole or destroys evidence.

---

## Problems I Hit and How I Fixed Them

| Problem | What Caused It | How I Fixed It |
|---|---|---|
| Could not search for a user in the group dialog | That dialog only searches for groups, not users | Went through the user's own settings and added the group from there |
| Group not found when trying to assign a user | I tried to add someone to a group that did not exist yet | Created the group first, then went back to the assignment |
| Script only created one user instead of four | The other three already existed from my manual step and failed silently | Added a check to the script that prints `SKIPPED` if an account already exists |
| Script threw errors when I pasted it in | The script had decorative `===` lines that PowerShell tried to read as commands | Removed the decorative lines and ran each command cleanly |

---

## Screenshots

All screenshots are in `/screenshots/lab-02/`

| File | What It Shows |
|---|---|
| `ou-structure.png` | All five folders set up in Active Directory |
| `finance-readOnly-group.png` | Finance-ReadOnly group created |
| `hr-readOnly-group.png` | HR-ReadOnly group created |
| `it-admins-group.png` | IT-Admins group created |
| `management-readOnly-group.png` | Management-ReadOnly group created |
| `gui-users-finance-ou.png` | Sarah Chen added manually |
| `gui-users-hr-ou.png` | Tom Harris added manually |
| `gui-users-it-ou.png` | Steven Williams added manually |
| `gui-users-management-ou.png` | James Wilson added manually |
| `users-csv-notepad.png` | The spreadsheet file used for bulk import |
| `powershell-bulk-import.png` | Script output showing four users created at once |
| `aduc-all-users-verified.png` | All eight users confirmed in the right folders |
| `powershell-group-verification.png` | All groups confirmed with correct members |
| `tom-harris-disabled-ou.png` | Tom Harris account disabled and moved |
| `sarah-chen-password-reset.png` | Password reset with forced change at next login |
| `linda-martinez-deleted.png` | Linda Martinez account permanently removed |
| `splunk-4720-account-creation.png` | Splunk log — all eight accounts created |
| `splunk-4728-group-membership.png` | Splunk log — all group assignments recorded |
| `splunk-4725-account-disabled.png` | Splunk log — Tom Harris disabled |
| `splunk-4726-account-deleted.png` | Splunk log — Linda Martinez deleted |
| `splunk-lab02-all-events.png` | Splunk — every Lab 02 action in one view |

---

## What Comes Next

**Lab 03 — Helpdesk Ticketing (Spiceworks)**

Every task from this lab becomes a formal support ticket:

- New employee setup — Steven Williams, Sarah Chen, Tom Harris, James Wilson
- Employee departure — Tom Harris account disabled and moved
- Password reset — Sarah Chen
- Security escalation — the credential attack from Lab 01

Each ticket will link back to the Splunk log codes from this lab, creating a complete paper trail from the account change all the way to the support ticket.

---

*Lab series: `redblue.local` homelab &nbsp;|&nbsp; Lab 01: Domain Build &nbsp;→&nbsp; **Lab 02: User Account Management** &nbsp;→&nbsp; Lab 03: Helpdesk Ticketing*
