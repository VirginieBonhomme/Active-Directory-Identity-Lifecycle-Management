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

In Lab 01, I built the `redblue.local` Active Directory domain from scratch to simulate a small company network. This allowed me to practice managing user access, authentication, and permissions in a controlled lab environment.

A major takeaway from this lab was understanding that user account activity is logged and can be reviewed for security, troubleshooting, and accountability. This reinforced the importance of audit logs for tracking administrative actions and investigating suspicious or unauthorized changes.

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
> In a business environment, user accounts are usually not deleted immediately when an employee leaves. Instead, the account is disabled and moved to a designated Disabled Users folder for a retention period, such as 30 days.
This helps preserve account history for auditing, troubleshooting, and security investigations. If the organization needs to review what the user accessed before leaving, the account record is still available.

<img width="1280" height="720" alt="aduc-ou-structure-complete" src="https://github.com/user-attachments/assets/cba0514d-af7c-4ed9-bb4c-feb790f862ff" />

---

### Step 2 — Creating Access Groups

Instead of assigning permissions to each user individually, I created access **groups** to manage permissions more efficiently. Users are added to groups based on their role, and each group is assigned the permissions needed for that role.

This follows the principle of **role-based access control** **(RBAC)**, where users receive only the access required to perform their job duties. RBAC helps improve security, reduce permission errors, and make access management easier to maintain.

| Group Name | Folder | What This Group Can Do |
|---|---|---|
| `IT-Admins` | IT | Full admin access — IT staff need to manage the systems |
| `Finance-ReadOnly` | Finance | Can read financial data but cannot change anything |
| `HR-ReadOnly` | HR | Can read employee records but cannot change anything |
| `Management-ReadOnly` | Management | Can view data across departments but cannot change anything |

> **Why name them `HR-ReadOnly` instead of just `HR`?**  
> Clear group names make permissions easier to understand and manage. A name like **HR-ReadOnly** tells administrators exactly which department the group belongs to and what level of access it provides.
> 
In a larger environment with many security groups, consistent naming helps reduce confusion, prevent permission mistakes, and make audits easier.

`Example of one of the groups I created/` — HR group inside its correct folder
<img width="1280" height="720" alt="aduc-create-hr-readOnly-group" src="https://github.com/user-attachments/assets/e15f1b8b-d540-42fa-99da-b62e6e7d38e9" />

---

### Step 3 — Adding Users One at a Time

I created the first four users by hand, clicking through menus in a tool called **ADUC (Active Directory Users and Computers)**. This is the visual way to add accounts.

| Name | Username | Department | Group |
|---|---|---|---|
| Steven Williams | `s.williams` | IT | `IT-Admins` |
| Sarah Chen | `s.chen` | Finance | `Finance-ReadOnly` |
| Tom Harris | `t.harris` | HR | `HR-ReadOnly` |
| James Wilson | `j.wilson` | Management | `Management-ReadOnly` |

For each account I:
- Placed them in the correct department folder
- Set a temporary starting password
- Turned on **"User must change password at next logon"**

> **Why force a password change on first login?**  
> When an administrator creates a new user account, they often assign a temporary password. If the user is not required to change it, the administrator would still know that password, which creates a security risk.
> 
Forcing a password change at first login ensures the user creates their own private password. This supports better account security, protects user privacy, and helps ensure that only the account owner knows their login credentials.

<img width="1280" height="720" alt="VirtualBox_ADDC01_19_05_2026_17_00_41" src="https://github.com/user-attachments/assets/4b0b6db1-43d1-48c6-979c-e9e9ccaf3028" />


---

### Step 4 — Adding Users Automatically with a Script

Creating user accounts manually works for a small number of users, but it does not scale well in a business environment. If a company hires many employees at once, creating each account individually can take a significant amount of time and increase the risk of errors.

For this step, I wanted to challenge myself by using **scripting** instead of only creating accounts manually. I researched proper PowerShell scripts for bulk user creation, created a simple user list using the Notes app, and used the script to create multiple accounts from that list automatically.

This helped me understand how scripting and automation can make IT administration faster, more consistent, and less error-prone.


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

The script reads each row of the csv and creates one user account per row, placing each person in the correct folder automatically.

<img width="1280" height="720" alt="VirtualBox_ADDC01_19_05_2026_17_22_17" src="https://github.com/user-attachments/assets/9fa33d42-5eb9-4e74-be86-063d124b6f8b" />


---

### Step 5 — Putting Users in the Right Groups

Creating an account and adding it to a group are two separate steps. After all accounts were created, I assigned everyone to their group using a script:

```powershell
Add-ADGroupMember -Identity "Finance-ReadOnly"    -Members "s.chen","l.martinez"
Add-ADGroupMember -Identity "HR-ReadOnly"         -Members "t.harris","d.thompson"
Add-ADGroupMember -Identity "IT-Admins"           -Members "s.williams","e.davis"
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

<img width="1280" height="720" alt="VirtualBox_ADDC01_19_05_2026_17_59_01" src="https://github.com/user-attachments/assets/df5ae88a-4bb9-4e1c-b92b-5cecda135a13" />
<img width="1280" height="720" alt="VirtualBox_ADDC01_19_05_2026_18_00_51" src="https://github.com/user-attachments/assets/9aba6b2c-a208-4cc9-a9cf-62e5e0d72540" />


---

## Common Helpdesk Tasks I Practiced


### Task 1 — Disabled an Account: Tom Harris

**Situation:** Tom Harris left the company.

**What I did:**
1. Disabled his account — he can no longer log in
2. Moved his account to the `Disabled` folder
3. It stays there for 30 days before being permanently deleted

> Deleting an account immediately destroys all records of what that person did. Keeping a disabled account for 30 days protects the company if questions come up later.

<img width="1280" height="720" alt="VirtualBox_ADDC01_19_05_2026_19_18_11" src="https://github.com/user-attachments/assets/6d225254-f111-4044-a954-dc45c178c90d" />

---

### Task 2 — Resetting a Password: Sarah Chen

**Situation:** Sarah Chen forgot her password and is locked out.

**What I did:**
1. Set a new temporary password for her in ADUC
2. Turned on **"User must change password at next logon"**

> This way I give her a temporary way back in, but she immediately replaces it with something only she knows. The helpdesk never has long-term access to a user's password.

<img width="1280" height="720" alt="VirtualBox_ADDC01_19_05_2026_19_24_09" src="https://github.com/user-attachments/assets/70cfc91d-e636-4177-8f6f-943484302d04" />


---

### Task 3 — Deleting an Account: Linda Martinez

**Situation:** Linda Martinez's account had been disabled for 30 days and was approved for permanent deletion.

**What I did:**
1. Deleted the account from ADUC
2. Confirmed the Finance folder no longer shows her account

> Deleting is always the last step, never the first. The correct order is: disable → move to Disabled folder → wait 30 days → delete with approval.

<img width="1280" height="720" alt="VirtualBox_ADDC01_19_05_2026_19_25_01" src="https://github.com/user-attachments/assets/a4580afc-e395-4051-8d0d-7855e44795c4" />
<img width="1280" height="720" alt="VirtualBox_ADDC01_19_05_2026_19_25_28" src="https://github.com/user-attachments/assets/3b23262a-6f90-42ce-bd12-9ef806e74359" />



---

## Checking My Work in Splunk

One of the most important parts of this lab was verifying that my **Active Directory** actions were being captured in **Splunk**. As I created, modified, and managed user accounts, those actions generated log entries that could be searched and reviewed.
This helped me understand how logging supports security monitoring, troubleshooting, and accountability.

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
<img width="1024" height="768" alt="VirtualBox_Target-PC_19_05_2026_19_57_09" src="https://github.com/user-attachments/assets/75833db5-86b9-4a75-878d-032f7fba7fef" />

**See every group change:**
```spl
index=endpoint host=ADDC01 EventCode=4728
| table _time, EventCode, Account_Name, ComputerName
| sort -_time
```
<img width="1024" height="768" alt="VirtualBox_Target-PC_19_05_2026_19_58_28" src="https://github.com/user-attachments/assets/9125f8f4-f38d-4f85-828b-a1e1d52b8428" />

**See every account that was disabled:**
```spl
index=endpoint host=ADDC01 EventCode=4725
| table _time, EventCode, Account_Name, ComputerName
| sort -_time
```
<img width="1024" height="768" alt="VirtualBox_Target-PC_19_05_2026_20_00_40" src="https://github.com/user-attachments/assets/f319ddfd-e4b3-4e26-aa13-c4452ab5c309" />


**See every account that was deleted:**
```spl
index=endpoint host=ADDC01 EventCode=4726
| table _time, EventCode, Account_Name, ComputerName
| sort -_time
```
<img width="1024" height="768" alt="VirtualBox_Target-PC_19_05_2026_20_01_50" src="https://github.com/user-attachments/assets/f71a5349-1529-4519-976d-41f94abccef7" />

**See everything from this lab in one view:**
```spl
index=endpoint host=ADDC01
(EventCode=4720 OR EventCode=4722 OR EventCode=4725 OR EventCode=4726 OR EventCode=4728)
| table _time, EventCode, Account_Name, ComputerName
| sort -_time
```

<img width="1024" height="768" alt="VirtualBox_Target-PC_19_05_2026_20_27_45" src="https://github.com/user-attachments/assets/d5ff80d6-dfca-4fb4-bb48-d97abf0beca7" />

---

## How This Connects to Real Attacks

This lab was about defense — building and managing accounts properly. But everything I built here connects to the attacks I studied in Lab 01. Understanding what I built helps explain why those attacks were dangerous.

| What I Built | Why Attackers Go After It | Lab 01 Attack |
|---|---|---|
| The domain controller running on ADDC01 | It stores every password on the network | `T1003` — attacker pulled password data from memory |
| `s.williams` account with a weak password | Easy passwords are easy to guess | `T1110` — a tool called Hydra guessed the password automatically |
| The Domain Admins group | Members of this group control everything | `T1078` — attackers look for high-privilege accounts to take over |
| Remote access (RDP) turned on | Lets people log in from anywhere — including attackers | `T1021.001` — tools called Crowbar and Hydra attacked the remote login |

---

## What I Learned

**Folders do more than organize, they control security rules.**  
When I created the department folders, I was not just tidying things up. Folders in Active Directory are where security policies get applied. If Finance users need a stricter password policy than IT staff, you set that on the folder and it applies automatically to everyone inside.

**Name your groups so anyone can understand them.**  
`HR-ReadOnly` tells you exactly what it does. `HR-Users` tells you nothing about access. Good names make security audits faster and mistakes less likely.

**Scripts save enormous amounts of time.**  
Four users by hand took several minutes. The script did four users in seconds. At the scale of a real company.

**The Disabled folder is a real security practice.**  
It is not just about being organized. It gives you a clear place to track who has been deactivated, confirm they truly cannot log in, and know exactly when their 30-day window expires.

**Everything is logged automatically.**  
Before this lab I knew logs existed. After this lab I understood what that actually means in practice. Every single thing I did showed up in Splunk on its own, within seconds. There is no way to hide changes to user accounts.

**Disable first, delete last.**  
This sequence exists for a reason: Disable → move to Disabled folder → wait 30 days → delete with approval. Skipping steps either creates a security hole or destroys evidence.

---

*Lab series: `redblue.local` homelab &nbsp;|&nbsp; Lab 01: Domain Build &nbsp;→&nbsp; **Lab 02: User Account Management** &nbsp;→&nbsp; Lab 03: Helpdesk Ticketing*
