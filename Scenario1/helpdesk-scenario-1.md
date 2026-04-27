# Help Desk Scenario 1 — Account Lockout

This is the first scenario I practiced in my osTicket help desk lab. The goal was to simulate a real account lockout situation from start to finish. From the user submitting the ticket all the way to closing it out after fixing the problem in Active Directory.

---

## The Situation

Ben Shuto is an employee who can't log into his computer. His account was disabled and he had no way to access anything on the domain. He borrowed a coworker's computer to submit a help desk ticket through the osTicket end user portal.

---

## What I Used

- **DC01** — Windows Server 2022 (Active Directory, PowerShell)
- **SRV-osTicket** — Ubuntu Server 22.04 (osTicket running on Apache/MySQL/PHP)
- **PC01** — Windows 10 (end user machine)
- **osTicket Staff Portal** — `http://192.168.1.20/scp`
- **osTicket End User Portal** — `http://192.168.1.20`

---

## Step 1 — Created the Test User in Active Directory

On my DC I opened PowerShell as Administrator and created Ben Shuto's account:

```powershell
New-ADUser -Name "Ben Shuto" -SamAccountName "bshuto" -UserPrincipalName "bshuto@mydomain.com" -AccountPassword (Read-Host -AsSecureString "Password22") -Enabled $true
```

Verified the account was created:
```powershell
Get-ADUser -Identity "bshuto" -Properties Enabled | Select Name, Enabled
```

Output showed `Enabled: True` confirming the account existed and was active.

---

## Step 2 — Simulated the Account Lockout

I disabled Ben's account to simulate him being locked out:

```powershell
Disable-ADAccount -Identity "bshuto"
```

Verified it was disabled:
```powershell
Get-ADUser -Identity "bshuto" -Properties Enabled | Select Name, Enabled
```

Output showed `Enabled: False`

> Note: I originally tried `Lock-ADAccount` to simulate a real lockout but that command wasn't recognized in my environment. `Disable-ADAccount` achieves the same end result for practice purposes. The user cannot log in either way.

> When I disabled Ben's account his session on the Windows 10 VM froze immediately. This is realistic — in a real environment a disabled account gets cut off quickly. Ben had to use a coworker's computer to submit his ticket.

---

## Step 3 — Ben Submitted a Ticket

On the Windows 10 VM I logged in as a different user to simulate Ben borrowing a coworker's computer. I opened a browser and went to:

```
http://192.168.1.20
```

I clicked **Open a New Ticket** and filled it out as Ben:

| Field | What Ben Entered |
|---|---|
| **Name** | Ben Shuto |
| **Email** | bshuto@mydomain.com |
| **Help Topic** | Account Locked Out |
| **Subject** | Can't log into my computer |
| **Message** | I tried logging into my computer this morning and it says my account is locked. I have not changed my password and I did not do anything different. Can someone please help me get back in? |

Clicked **Create Ticket.**

The confirmation screen showed:

*"Thank you for contacting us. A support ticket has been created."*

> osTicket didn't display a ticket number on the confirmation screen. The ticket was still created and waiting in the queue.

---

## Step 4 — Agent Picked Up the Ticket

I opened a new browser tab and went to the staff portal:

```
http://192.168.1.20/scp
```

I logged in as my help desk agent **Joe Client.**

In the staff portal I clicked **Tickets** and found Ben's ticket:

| Field | Details |
|---|---|
| **Ticket #** | 580006 |
| **Subject** | Can't log into my computer |
| **From** | Ben Shuto |
| **Priority** | High |

I clicked on the ticket to open it and assigned it to myself by selecting my agent name in the Assignee field.

---

## Step 5 — Posted an Internal Note

Before touching anything in AD I documented my plan inside the ticket using an Internal Note.

I wrote:

*"Received ticket from Ben Shuto reporting he cannot log into his computer. Verified ticket details. Checking AD account status before proceeding with resolution."*

Clicked **Post Note.**

> This is something I learned is really important in real help desk work. You always document what you're doing and why before you do it. If another tech picks up the ticket later they can see exactly what happened and what was already tried.

---

## Step 6 — Verified and Fixed the Account in Active Directory

On my DC I opened PowerShell as Administrator and checked Ben's account status:

```powershell
Get-ADUser -Identity "bshuto" -Properties Enabled | Select Name, Enabled
```

Output confirmed `Enabled: False` — account was disabled.

I re-enabled it:

```powershell
Enable-ADAccount -Identity "bshuto"
```

Verified it worked:

```powershell
Get-ADUser -Identity "bshuto" -Properties Enabled | Select Name, Enabled
```

Output now showed `Enabled: True` — account was active again.

---

## Step 7 — Posted a Reply to Ben

Back in the staff portal I clicked the **Post Reply** tab and wrote a response that Ben would see:

*"Hi Ben, thank you for contacting the help desk. I was able to look into your account and found that it had been disabled. I have gone ahead and re-enabled your account and you should now be able to log into your computer. Please attempt to log in and let us know if you experience any further issues. If your password has expired you will be prompted to set a new one at login. Thank you."*

---

## Step 8 — Closed the Ticket

To close the ticket I posted one final reply with the status change:

*"Closing ticket. Issue resolved, account has been re-enabled. Ben has been notified."*

Changed the ticket status to **Resolved** at the same time as posting the reply and clicked **Post Reply.**

> osTicket requires something to be written in the reply box before it will let you change the status. It won't close a ticket with an empty response which makes sense. You should always document why you're closing it.

The ticket moved out of the open queue and showed as **Closed** under the Tickets tab.

---

## Step 9 — Verified the Fix

On the Windows 10 VM I logged in as Ben Shuto using his credentials. He was able to log in successfully confirming the issue was fully resolved.

---

## Full Ticket Timeline

| Time | Action |
|---|---|
| Ticket opened | Ben submitted ticket from coworker's computer |
| Agent assigned | Joe Client picked up and assigned ticket #580006 |
| Internal note posted | Documented plan before making changes |
| AD verified | Confirmed account was disabled in PowerShell |
| Account fixed | Re-enabled account with Enable-ADAccount |
| Reply posted | Notified Ben that issue was resolved |
| Ticket closed | Status changed to Resolved |
| Fix verified | Ben successfully logged into his account |

---

## PowerShell Commands Used

```powershell
# Check if account exists and is enabled
Get-ADUser -Identity "bshuto" -Properties Enabled | Select Name, Enabled

# Disable account (simulating lockout)
Disable-ADAccount -Identity "bshuto"

# Re-enable account (the fix)
Enable-ADAccount -Identity "bshuto"
```

---

## What I Learned From This Scenario

**Documentation matters as much as the fix.** Writing internal notes before touching anything and posting a clear reply to the user is just as important as actually fixing the problem. A ticket with no notes is useless to the next tech who has to look at it.

**Always verify before and after.** I ran `Get-ADUser` before making any changes to confirm what the problem was and again after to confirm it was fixed. Never assume a command worked, verify it.

**Realistic things happen in a lab too.** Ben's session freezing when I disabled his account wasn't something I planned. It just happened and it ended up being a realistic detail that actually made the scenario more authentic.

**Users can't always submit their own tickets.** A locked out user can't log into a system that requires authentication. In a real environment users submit tickets from their phone, a coworker's computer, or by calling the help desk directly. The agent can also create the ticket on their behalf.
