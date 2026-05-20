# Help Desk Scenario 3 — New Hire Onboarding

This is the third scenario I practiced in my osTicket help desk lab. New hire onboarding is one of the most involved help desk tasks because it has multiple steps — creating an AD account, adding the user to the right security group, granting folder access, and verifying everything works before the employee walks in on their first day. This scenario covers the full onboarding workflow from ticket submission to closing.

---

## The Situation

Joe Client, a manager, submitted a ticket saying a new employee named Sarah Jones is starting Monday. She needs a domain account created, access to the Finance shared folder, and everything set up before she arrives. The help desk needs to complete all of this and confirm it's working.

---

## What I Used

- **DC01** — Windows Server 2022 (Active Directory Users and Computers, Shared Folder)
- **SRV-osTicket** — Ubuntu Server 22.04 (osTicket)
- **PC01** — Windows 10 (used to verify Sarah could log in and access the folder)
- **osTicket Staff Portal** — `http://192.168.1.20/scp`
- **osTicket End User Portal** — `http://192.168.1.20`

> For this scenario I did everything through the GUI instead of PowerShell to practice both methods. In a real help desk environment you'll use whichever is faster for the situation — knowing both is what matters.

---

## Pre-Work — Setting Up the Environment

Before starting the scenario I set up the shared folder and confirmed the security group that Sarah would need access to.

### Created the Finance Shared Folder

I created a folder called `Finance` on the DC desktop. In a real environment this would be on a dedicated file server but for the lab the desktop works fine.

<p align="center">
<br/>
<img src="https://i.imgur.com/mP3neTp.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>

### Verified the Finance-Users Security Group Existed

I already had a `Finance-Users` security group in AD from previous lab work. I confirmed it was there by opening **Active Directory Users and Computers** and finding it in the list.

<p align="center">
<img src="https://i.imgur.com/FD4fbfx.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>

---

## Step 1 — Manager Submitted a Ticket

On the Windows 10 VM I went to the end user portal:

```
http://192.168.1.20
```

Clicked **Open a New Ticket** and filled it out as Joe Client the manager:

| Field | What Joe Entered |
|---|---|
| **Name** | Joe Client |
| **Email** | jclient@mydomain.com |
| **Help Topic** | General Inquiry |
| **Subject** | New hire starting Monday |
| **Message** | Hi IT, we have a new employee starting Monday. Her name is Sarah Jones. She will need a domain account, access to the Finance shared folder, and her computer set up before she arrives. Please let me know when everything is ready. Thank you. |

Clicked **Create Ticket.**

Confirmation screen showed a green banner stating the support ticket request was created.

---

## Step 2 — Agent Picked Up the Ticket

Logged into the staff portal as **Kenny Panyavong** and found the ticket:

| Field | Details |
|---|---|
| **Ticket #** | 644877 |
| **Subject** | New hire starting Monday |
| **From** | Joe Client |
| **Priority** | Normal |

Clicked into the ticket and assigned it to myself.


<p align="center">
<img src="https://i.imgur.com/0pwnba7.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>

<p align="center">
<img src="https://i.imgur.com/EXl8ui8.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>


<p align="center">
<img src="https://i.imgur.com/e8yUOVA.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>


<p align="center">
<img src="https://i.imgur.com/eRHBYnz.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>



---

## Step 3 — Posted an Internal Note

Before touching anything I posted an Internal Note titled **"Onboarding Document"** outlining the plan:

*"Received new hire request from Joe Client (manager) for Sarah Jones starting Monday. Action items: 1) Create AD user account for Sarah Jones. 2) Add her to the Finance-Users security group. 3) Grant access to the Finance shared folder. 4) Verify she can log in. Will work through each item and document steps."*

> Posting an internal note before starting is important. It documents the plan, shows what actions need to be taken, and gives other techs full context if the ticket gets reassigned.


<p align="center">
<img src="https://i.imgur.com/prJUtgi.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>


---

## Step 4 — Created Sarah's AD Account

I created Sarah's account through Active Directory Users and Computers on the DC:

1. Opened **Active Directory Users and Computers**
2. Right-clicked the **Users** OU → **New → User**
3. Filled in:
   - First Name: Sarah
   - Last Name: Jones
   - User logon name: sjones
4. Set a temporary password
5. Checked **User must change password at next logon**
6. Clicked **Finish**

After creating the account I right-clicked Sarah's account and confirmed it was enabled through the GUI. The account showed as active in Active Directory Users and Computers with no disabled indicator.

> I did everything through the GUI for this scenario instead of PowerShell just to mix things up and practice both methods. In a real help desk environment either is perfectly acceptable. The GUI is straightforward for single account creation and gives you a clear visual confirmation of everything you set.

<p align="center">
<img src="https://i.imgur.com/8RXIm5X.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>

<p align="center">
<img src="https://i.imgur.com/VguBwcW.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>

<p align="center">
<img src="https://i.imgur.com/fVx2iTg.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>


---

## Step 5 — Added Sarah to the Finance-Users Security Group

I added Sarah to the Finance-Users group through the GUI:

1. In **Active Directory Users and Computers** found the **Finance-Users** group
2. Right-clicked → **Properties → Members tab → Add**
3. Typed `sjones` → **Check Names** → clicked **OK**
4. Sarah appeared in the members list
5. Clicked **Apply → OK**

Confirmed Sarah was listed as a member of Finance-Users by checking the Members tab in the group properties.

<p align="center">
<img src="https://i.imgur.com/sF1oRlC.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>

<p align="center">
<img src="https://i.imgur.com/pMadSLn.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>

<p align="center">
<img src="https://i.imgur.com/7hueAJR.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>



---

## Step 6 — Set Up the Finance Shared Folder Permissions

I set up both NTFS security permissions and share permissions on the Finance folder.

### NTFS Security Permissions

1. Right-clicked the Finance folder → **Properties → Security tab → Edit → Add**
2. Typed `Finance-Users` → **Check Names** → clicked **OK**
3. Granted the following permissions:
   - Read & Execute
   - List folder contents
   - Read
4. Clicked **Apply → OK**

### Share Permissions

1. Right-clicked the Finance folder → **Properties → Sharing tab → Advanced Sharing**
2. Checked **Share this folder**
3. Named it `Finance`
4. Clicked **Permissions → Add**
5. Added `Finance-Users` and granted **Read/Write** permissions
6. Removed **Everyone** from the permissions list
7. Clicked **Apply → OK**

The network path for the shared folder was:

```
\\DC\Users\a-kpanyavong\Desktop\Finance\Finance
```

<p align="center">
<img src="https://i.imgur.com/DlyWPez.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>

<p align="center">
<img src="https://i.imgur.com/OmB6Ohk.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>


---

## Step 7 — Verified Sarah Could Access Everything

On the Windows 10 VM I logged in as Sarah Jones using the temporary password set during account creation.

Windows immediately prompted Sarah to change her password since **User must change password at next logon** was checked. She set a new password and logged in successfully.

I opened **File Explorer** and navigated to the network path:

```
\\DC01\Finance
```

Sarah was able to access the Finance folder and see the contents. I had placed a text file in the folder when I created it to confirm the contents were visible — it showed up successfully confirming access was working correctly.


<p align="center">
<img src="https://i.imgur.com/GmjQTQ6.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>

<p align="center">
<img src="https://i.imgur.com/fya3VOO.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>

<p align="center">
<img src="https://i.imgur.com/NWxGXN7.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>

<p align="center">
<img src="https://i.imgur.com/wu6RvXC.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>

<p align="center">
<img src="https://i.imgur.com/rP7ISRg.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>

<p align="center">
<img src="https://i.imgur.com/zL32kj9.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>



---

## Step 8 — Posted Reply to Joe and Closed the Ticket

Back in the staff portal I first posted a **Reply to Joe Client:** 

*"Hi Joe, I have completed the onboarding setup for Sarah Jones. Here is a summary of everything that was done: A domain account has been created for Sarah Jones (username: sjones). Her account has been added to the Finance-Users security group. She has been granted Read and Write access to the Finance shared folder at \\DC01\Finance. Her account is set to require a password change at first login for security purposes. Sarah should be all set for Monday. Please have her log into her computer and change her password when she arrives. Let us know if anything else is needed. Thank you."*

Then posted a final **Internal Note** summarizing everything that was done:

*"Onboarding complete for Sarah Jones. AD account created (sjones), added to Finance-Users security group, granted Read and Write access to Finance shared folder at \\DC01\Finance. Account verified working on Windows 10 VM — Sarah was able to log in, change password, and access the Finance folder successfully. Ticket resolved."*


Changed status to **Resolved** and posted the reply. Ticket closed.

<p align="center">
<img src="https://i.imgur.com/KEBwQb9.png" height="60%" width="60%" alt="Disk Sanitization Steps"/>

---

## Full Ticket Timeline

| Action | Details |
|---|---|
| Ticket opened | Joe Client submitted new hire request for Sarah Jones |
| Ticket assigned | Kenny Panyavong picked up ticket #644877 |
| Internal note posted | Documented onboarding action items |
| AD account created | Sarah Jones account created via GUI (sjones) |
| Account confirmed enabled | Verified through Active Directory Users and Computers |
| Added to group | Sarah added to Finance-Users security group via GUI |
| Folder permissions set | Finance-Users granted Read/Write on shared folder |
| Fix verified | Sarah logged in on Windows 10, changed password, accessed Finance folder |
| Reply posted | Sent Joe a full summary of everything completed |
| Ticket closed | Status changed to Resolved |

---

## What I Learned From This Scenario

**Onboarding has multiple moving parts.** Unlike a password reset or account unlock this scenario required several separate tasks — creating the account, enabling it, adding it to a group, setting folder permissions, and verifying access. Missing any one of those steps means the new hire shows up on Monday and can't do their job. The internal note checklist at the start helped make sure nothing got missed.

**Always verify each step visually.** After every action I confirmed it worked by checking the GUI. Creating an account doesn't automatically mean it's enabled. Adding someone to a group doesn't mean the folder permissions are right. Verifying each step individually before moving to the next one is the right habit to build.

**GUI and PowerShell both have their place.** I used the GUI for everything in this scenario instead of PowerShell. Both methods work and in a real help desk environment you'll use whatever is faster for the situation. Knowing both is what matters.

**Documentation needs to be accurate.** When I wrote the reply to Joe I made sure to include the exact permissions Sarah was granted — Read and Write — not just "access to the folder." In a real company that ticket is the official record of what was done. If there's ever a security audit or a question about who has access to what the ticket needs to reflect exactly what happened.

**Ticket accuracy protects you.** If you write that you granted Read only but actually granted Read and Write that's a problem. Always make sure your ticket notes match exactly what you did in AD and on the file system.

**Require password change at first login is standard practice.** You never want to give a new hire a permanent password that IT knows. Setting a temporary password and forcing a change at first login means only Sarah knows her password from the moment she logs in. That's basic security hygiene and something every help desk tech should do automatically.
