🔐 Lab 01 — Active Directory & Identity Management
Platform Cloud Cost Time Certs

🎬 Watch Me Build This Lab!
Watch the video

Full walkthrough — I demo every step from VM provisioning to GPO enforcement and help desk tasks.

Scenario: An organisation needs a centralised identity system. Users must log in with domain accounts, access is controlled by group membership, and security policies must be enforced automatically across every machine — without touching each one manually.

📋 Table of Contents
Watch Me Build This Lab
Business Context
What I Built
Skills Demonstrated
Environment Setup
Lab Walkthrough
Step 1 — Provision the Environment
Step 2 — Install AD DS
Step 3 — Promote to Domain Controller
Step 4 — Build the Organisational Structure
Step 5 — Configure Group Policy
Step 6 — Simulate Help Desk Tasks
Verification
Troubleshooting
Key Takeaways
Resources
💼 Business Context
Active Directory is the identity backbone of virtually every enterprise Windows environment. It answers the fundamental question: who is allowed to do what?

When an employee joins, IT creates one account in Active Directory — and access to email, file shares, printers, and applications is granted automatically through group membership. When they leave, one account disable closes every door simultaneously.

This lab builds that system from scratch. The same concepts apply directly to Microsoft Entra ID (Azure AD) in hybrid and cloud-only environments, making this foundational knowledge for IT support, sysadmin, cloud engineering, and security roles.

Role	How This Lab Applies
IT Support / Help Desk	Password resets, account unlocks, and group changes — the top three ticket types in any enterprise
Sysadmin	Designing OU structure, deploying GPOs, managing domain-joined machines at scale
Cloud Engineer	Entra ID uses the same model: users, groups, roles, conditional access. On-prem AD knowledge transfers directly
Security Analyst	AD is the most targeted system in ransomware attacks. Understanding how it works is the foundation of defending it
🏗 What I Built
lab.local (Active Directory Forest)
│
├── OU: IT
│   ├── User: alice.chen
│   └── Group: IT_Admins  ◄── GPO: IT Security Policy (password + lock + USB policy)
│
├── OU: Finance
│   ├── User: bob.patel
│   └── Group: Finance_Users
│
├── OU: HR
│   ├── User: carol.jones
│   └── Group: HR_Users
│
├── OU: Sales
│   ├── User: david.smith
│   └── Group: Sales_Users
│
└── OU: Computers
    └── (domain-joined workstations)
One Domain Controller running Windows Server 2025, deployed in Azure at zero cost within the free tier, with four departments, four security groups, four user accounts, and a Group Policy Object enforcing security settings across the IT OU.

✅ Skills Demonstrated
Skill	Tool Used
Provisioning a Windows Server VM in Azure	Azure Portal
Installing the AD DS role	Server Manager / PowerShell
Promoting a server to Domain Controller	Install-ADDSForest
Creating Organisational Units	ADUC / New-ADOrganizationalUnit
Creating security groups with RBAC	ADUC / New-ADGroup
Creating and enabling user accounts	ADUC / New-ADUser
Assigning users to groups	Add-ADGroupMember
Creating and linking a GPO	Group Policy Management Console
Configuring password policy, screen lock, USB restrictions	Group Policy Object Editor
Performing help desk tasks (reset, unlock, disable)	PowerShell AD cmdlets
Auditing accounts and group memberships	Get-ADUser, Get-ADPrincipalGroupMembership
🛠 Environment Setup
Option A — Azure (Recommended)
No local hardware required. VM runs in Azure, connected via RDP.

Create a free account at azure.microsoft.com/free
Sign in to portal.azure.com
Create a Virtual Machine with the settings below
Setting	Value	Why
Region	East US	Most available VM sizes under free tier
Image	Windows Server 2025 Datacenter — Gen2	Latest server OS, includes 180-day eval licence
Size	Standard_B2s (2 vCPU, 4 GB RAM)	Smallest size that runs AD comfortably
Authentication	Password	Used for RDP access
Public inbound ports	Allow RDP (3389)	Required to connect remotely
OS disk	Standard SSD	Included in free tier
💡 Cost tip: Stop the VM (don't delete it) at the end of every session. A B2s VM runs at ~$0.05/hour. Stopping it pauses compute billing. Your $200 free credit goes much further this way.

Enable clipboard sharing before connecting:

Open Remote Desktop on your local machine
Enter the VM's public IP → click Show Options
Click Local Resources tab → check Clipboard
Click Connect
Note: If using the Azure browser console, clipboard sharing is limited. Download the RDP file from the portal (Connect → Download RDP File) and open it with the native Remote Desktop app instead.

Option B — VirtualBox (Local)
Requirement	Minimum
RAM	8 GB (4 GB for VM, 4 GB for host OS)
Disk	60 GB free
CPU	Quad-core with virtualisation enabled in BIOS
Download VirtualBox — free, no account needed
Download the Windows Server 2025 Evaluation ISO
Create a VM: 4 GB RAM, 60 GB disk, type = Windows Server 2022
Mount the ISO, boot, and select Windows Server 2025 Datacenter with Desktop Experience
🔬 Lab Walkthrough
Step 1 — Provision the Environment
RDP into the VM. Server Manager opens automatically on login. All configuration from here runs inside the VM.

📸 Screenshot — Azure VM running + RDP clipboard setting enabled
<img width="1905" height="949" alt="Screenshot The Environment Step 1_Section 1_2" src="https://github.com/user-attachments/assets/bc0bd6ce-c299-4a0d-a2aa-a5c9fc828d1d" />

<img width="1382" height="770" alt="Screenshot The Environment Step 1_Section 1_4_network" src="https://github.com/user-attachments/assets/6c13387d-d570-47f5-b57c-51330d763a8e" />


Step 2 — Install Active Directory Domain Services
What is a Domain Controller? A Domain Controller (DC) is the brain of Active Directory. Every login attempt on the domain is authenticated against the DC. Every user account lives on the DC. There is typically more than one in production for redundancy — we are building one here as the foundation.

GUI: Server Manager → Manage → Add Roles and Features → Server Roles → check Active Directory Domain Services → Add Features → Install

PowerShell equivalent:

# Install AD DS role with management tools
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Also install Group Policy Management Console (required for Step 5)
Install-WindowsFeature -Name GPMC
📸 Screenshot — AD DS role selected in Add Roles wizard + install complete

<img width="1842" height="895" alt="Screenshot Install Active Directory Domain Services Step 2_Section 2" src="https://github.com/user-attachments/assets/d10936ee-a7f4-4f86-9611-b5ed62df4fb3" />
<img width="851" height="621" alt="Screenshot Install Active Directory Domain Services Step 2_Section 2_2" src="https://github.com/user-attachments/assets/c2ca29d0-b417-4e15-884c-570adb0cfb8e" />
<img width="928" height="622" alt="Screenshot Install Active Directory Domain Services Step 2_Section 2_5" src="https://github.com/user-attachments/assets/92cab35f-c81d-483c-8297-aeed4276c080" />

Why install GPMC now? Step 5 requires the Group Policy Management Console — a separate tool from Active Directory Users and Computers. Installing it here means it's ready when you need it. After installation, it appears under Tools in Server Manager.

Step 3 — Promote the Server to a Domain Controller
What is a Forest and Domain? A Forest is the top-level container of your entire Active Directory structure — think of it as the organisation. A Domain is a named boundary inside the forest (lab.local). Everything inside it is managed together. Most small-to-medium organisations have one domain in one forest.

GUI: Server Manager → yellow warning flag (top right) → Promote this server to a domain controller → Add a new forest → Root domain name: lab.local → Set a DSRM password → Install

The server restarts automatically when promotion completes.

PowerShell equivalent:

Import-Module ADDSDeployment

Install-ADDSForest `
    -DomainName 'lab.local' `
    -DomainNetBiosName 'LAB' `
    -InstallDns:$true `
    -SafeModeAdministratorPassword (ConvertTo-SecureString 'YourDSRMPassword!' -AsPlainText -Force) `
    -Force:$true
📸 Screenshot — Domain promotion wizard + Server Manager showing lab.local active

What just happened: lab.local is now an Active Directory forest. This server is the root Domain Controller — it runs DNS for the domain and is the authoritative source for all authentication decisions. Every machine that joins lab.local will trust this server.

Step 4 — Build the Organisational Structure
Open Active Directory Users and Computers (ADUC) from the Tools menu in Server Manager.

What is an Organisational Unit (OU)? An OU is a folder inside Active Directory. You use OUs to organise users, computers, and groups by department or function. The real power: you can link a Group Policy to an OU, and every user or computer inside automatically receives that policy. IT gets one set of rules. Finance gets another. All managed centrally.

Create Organisational Units
GUI: Right-click lab.local in ADUC → New → Organizational Unit

# Create all 5 OUs at once
New-ADOrganizationalUnit -Name "IT"        -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Finance"   -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "HR"        -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Sales"     -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Computers" -Path "DC=lab,DC=local"
Create Security Groups
What is a Security Group? A Security Group holds user accounts. Instead of granting access to resources user-by-user, you grant it once to a group, then add users to that group. This is role-based access control (RBAC). At scale: giving 50 Finance employees access to a new system means adding one group — not 50 accounts. Removing someone from the group instantly revokes all their group-based access.

GUI: Right-click each OU → New → Group → Group scope: Global → Group type: Security

New-ADGroup -Name "IT_Admins"     -GroupScope Global -GroupCategory Security -Path "OU=IT,DC=lab,DC=local"
New-ADGroup -Name "Finance_Users" -GroupScope Global -GroupCategory Security -Path "OU=Finance,DC=lab,DC=local"
New-ADGroup -Name "HR_Users"      -GroupScope Global -GroupCategory Security -Path "OU=HR,DC=lab,DC=local"
New-ADGroup -Name "Sales_Users"   -GroupScope Global -GroupCategory Security -Path "OU=Sales,DC=lab,DC=local"
Create User Accounts and Assign Group Membership
⚠️ Important: Run the entire block below as one unit — do not execute it line by line. The $password variable must be defined before the New-ADUser commands run. Select all, then paste into PowerShell and press Enter.

# ── Step 1: Define the password variable ──────────────────────────────────────
$password = ConvertTo-SecureString "Welcome@2026!" -AsPlainText -Force

# ── Step 2: Create all 4 users ────────────────────────────────────────────────
New-ADUser -Name "alice.chen" -GivenName "Alice" -Surname "Chen" `
    -SamAccountName "alice.chen" -UserPrincipalName "alice.chen@lab.local" `
    -Path "OU=IT,DC=lab,DC=local" -AccountPassword $password -Enabled $true

New-ADUser -Name "bob.patel" -GivenName "Bob" -Surname "Patel" `
    -SamAccountName "bob.patel" -UserPrincipalName "bob.patel@lab.local" `
    -Path "OU=Finance,DC=lab,DC=local" -AccountPassword $password -Enabled $true

New-ADUser -Name "carol.jones" -GivenName "Carol" -Surname "Jones" `
    -SamAccountName "carol.jones" -UserPrincipalName "carol.jones@lab.local" `
    -Path "OU=HR,DC=lab,DC=local" -AccountPassword $password -Enabled $true

New-ADUser -Name "david.smith" -GivenName "David" -Surname "Smith" `
    -SamAccountName "david.smith" -UserPrincipalName "david.smith@lab.local" `
    -Path "OU=Sales,DC=lab,DC=local" -AccountPassword $password -Enabled $true

# ── Step 3: Add each user to their department group ───────────────────────────
Add-ADGroupMember -Identity "IT_Admins"     -Members "alice.chen"
Add-ADGroupMember -Identity "Finance_Users" -Members "bob.patel"
Add-ADGroupMember -Identity "HR_Users"      -Members "carol.jones"
Add-ADGroupMember -Identity "Sales_Users"   -Members "david.smith"
📸 Screenshot — ADUC showing OU structure + PowerShell confirming users and group memberships

Step 5 — Configure Group Policy
Open Group Policy Management from the Tools menu in Server Manager.

What is a Group Policy Object (GPO)? A GPO is a collection of settings that applies automatically to every user or computer inside an OU. Create it once, link it to an OU, and every machine in that OU gets those rules on next login or after gpupdate /force. Password policies, screen lock timers, USB restrictions, software controls — all enforced centrally, at scale, without touching each machine individually.

Steps:

Expand Forest: lab.local → Domains → lab.local
Right-click the IT OU → Create a GPO in this domain and link it here
Name it: IT Security Policy
Right-click the new GPO → Edit
Configure the four settings below
Policy Path	Setting	Value	Purpose
Computer Config → Windows Settings → Security → Account Policies → Password Policy	Minimum password length	12	Enforces strong passwords
Computer Config → Windows Settings → Security → Account Policies → Password Policy	Password must meet complexity requirements	Enabled	Requires upper, lower, number, and symbol
Computer Config → Windows Settings → Security → Local Policies → Security Options	Interactive logon: Machine inactivity limit	900 seconds	Auto-locks screen after 15 minutes
Computer Config → Administrative Templates → System → Removable Storage Access	All removable storage classes: Deny all access	Enabled	Blocks USB data exfiltration
📸 Screenshot — GPMC showing IT Security Policy linked + GPO editor with settings configured

Test the GPO: Join a second VM to lab.local, move its computer account into the IT OU, run gpupdate /force, log in as alice.chen, and verify the screen lock policy takes effect.

Step 6 — Simulate Help Desk Tasks
These are the highest-frequency tasks in any IT support role. Each one is practiced on the test accounts created in Step 4.

Reset a Password
# Reset and force the user to set their own password on next login
Set-ADAccountPassword -Identity "bob.patel" `
    -Reset -NewPassword (ConvertTo-SecureString "NewPass@2026!" -AsPlainText -Force)

Set-ADUser -Identity "bob.patel" -ChangePasswordAtLogon $true
Unlock a Locked Account
# Accounts lock after repeated failed login attempts — this is the fix
Unlock-ADAccount -Identity "carol.jones"
Disable an Account (Offboarding)
# Disable preserves account history and group memberships for audit purposes
# Deletion is permanent — always disable first
Disable-ADAccount -Identity "david.smith"

# Find all currently disabled accounts
Search-ADAccount -AccountDisabled | Select-Object Name, SamAccountName
Audit and Reporting
# Find accounts inactive for 90+ days
$cutoff = (Get-Date).AddDays(-90)
Get-ADUser -Filter {LastLogonDate -lt $cutoff -and Enabled -eq $true} `
    -Properties LastLogonDate | Select-Object Name, LastLogonDate

# Check group memberships for a specific user
Get-ADPrincipalGroupMembership -Identity "alice.chen" | Select-Object Name
📸 Screenshot — PowerShell output: password reset, account unlock, and group membership audit

✔️ Verification
Run these checks to confirm everything is working correctly before calling the lab complete.

Check	Command	Expected Result
Domain Controller is running	Get-ADDomainController	Returns DC info including forest lab.local
All OUs exist	Get-ADOrganizationalUnit -Filter *	Lists all 5 OUs
Users exist and are enabled	Get-ADUser -Filter {Enabled -eq $true}	Returns your 4 test accounts
Group memberships are correct	Get-ADGroupMember -Identity IT_Admins	Returns alice.chen
GPO is linked to IT OU	Get-GPInheritance -Target 'OU=IT,DC=lab,DC=local'	Shows IT Security Policy as linked
🔧 Troubleshooting
Problem	Fix
PowerShell prompts for Name: when creating users	You ran New-ADUser before defining $password. Run the entire script block at once — $password must be declared first.
Cannot copy/paste into the VM	Open RDP client → Show Options → Local Resources → check Clipboard → reconnect. Or download the RDP file from the Azure portal and use the native Remote Desktop app.
Promotion fails — DNS conflict	Set the NIC's preferred DNS to 127.0.0.1 before promoting, or use the VM's static IP
Cannot RDP after domain join	Log in as LAB\Administrator (domain admin), not just Administrator
GPO not applying	Run gpupdate /force on the target machine, then gpresult /r to see applied policies
User cannot log in after creation	Confirm the account is Enabled and ChangePasswordAtLogon is not blocking access
AD Users and Computers not showing	Run dsa.msc from the Run dialog, or run Add-WindowsFeature RSAT-ADDS
🧠 Key Takeaways
Active Directory is the identity plane for the majority of enterprise Windows environments. Everything — email, file shares, printers, applications — flows through it.
Group membership is access control. Users don't get permissions directly; they inherit them through groups. This is RBAC in practice.
GPOs enforce policy at scale. One policy object linked to an OU governs every machine and user inside it. This is how enterprises manage thousands of endpoints without manual configuration.
Disabling accounts is not the same as deleting them. Disabled accounts preserve audit trails and group memberships. Always disable before you delete.
The same concepts run in the cloud. Microsoft Entra ID (Azure AD) uses users, groups, roles, and conditional access policies — direct extensions of everything built here.
📚 Resources
Microsoft: What is Active Directory?
Microsoft: Group Policy Overview
Microsoft: Azure Free Account
CompTIA Security+ Exam Objectives
AZ-104 Azure Administrator Study Guide
Part of an ongoing hands-on IT/cloud security lab series. Each lab documents a real-world infrastructure scenario built from scratch using free tools and cloud free tiers.
