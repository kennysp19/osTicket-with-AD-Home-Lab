# Help Desk Scenario 2 - Password Reset

This is the second scenario I practiced in my osTicket help desk lab. The goal was to simulate a user forgetting their password and needing a reset - one of the most common help desk tickets in any IT environment. This scenario covers submitting the ticket, resetting the password in Active Directory, forcing a password change at next login, and properly closing the ticket only after confirming the user can log in.

---

## The Situation

Ben Shuto forgot his password and can't log into his computer. Unlike Scenario 1 his account is active and enabled, he just doesn't know his password. He needs the help desk to reset it to a temporary password and he'll set his own new password when he logs in.

---

## What I Used

- **DC01** - Windows Server 2022 (Active Directory, PowerShell)
- **SRV-osTicket** - Ubuntu Server 22.04 (osTicket running on Apache/MySQL/PHP)
- **PC01** - Windows 10 (end user machine)
- **osTicket Staff Portal** - `http://192.168.1.20/scp`
- **osTicket End User Portal** - `http://192.168.1.20`

---

## Step 1 - Verified Ben's Account Was Active

Before starting I confirmed Ben's account from Scenario 1 was still enabled:

```powershell
Get-ADUser -Identity "bshuto" -Properties Enabled | Select Name, Enabled
```

Output showed `Enabled: True` - account was active and ready.

<p align="center">
<br/>
<img src="https://i.imgur.com/c0kozKa.png" height="50%" width="50%" alt="Disk Sanitization Steps"/>
  
---

## Step 2 - Ben Submitted a Ticket

On the Windows 10 VM I went to the end user portal:

```
http://192.168.1.20
```

Clicked **Open a New Ticket** and filled it out as Ben:

| Field | What Ben Entered |
|---|---|
| **Name** | Ben Shuto |
| **Email** | bshuto@mydomain.com |
| **Help Topic** | Password Reset |
| **Subject** | Forgot my password |
| **Message** | Hi, I forgot my password and I can't log into my computer. Can someone please reset it for me? Thank you. |

Clicked **Create Ticket.**

The confirmation screen showed a green banner stating the ticket request was created along with a thank you message from the support team.

<p align="center">
Ticket creation <br/>
<img src="https://i.imgur.com/gdMkABu.png" height="50%" width="50%" alt="Disk Sanitization Steps"/>

<p align="center">
<br/>
<img src="https://i.imgur.com/t0u83i8.png" height="50%" width="50%" alt="Disk Sanitization Steps"/>


---

## Step 3 - Agent Picked Up the Ticket

I went to the staff portal:

```
http://192.168.1.20/scp
```

Logged in as agent **Kenny Panyavong** and found Ben's ticket under the Tickets tab:

| Field | Details |
|---|---|
| **Ticket #** | 685633 |
| **Subject** | Forgot my password |
| **From** | Ben Shuto |
| **Priority** | Normal |

Clicked into the ticket and assigned it to myself.

---

## Step 4 - Posted an Internal Note

Before making any changes in AD I documented my plan in an Internal Note.

*"Received ticket from Ben Shuto reporting he has forgotten his password and cannot log into his computer. Account appears to be active and enabled. Will reset password in Active Directory and force password change at next login."*

Clicked **Post Note.**

> Always document what you're going to do before you do it. If another tech picks up this ticket later they can see exactly what was done and why.


<p align="center">
<br/>
<img src="https://i.imgur.com/4ENEcf5.png" height="50%" width="50%" alt="Disk Sanitization Steps"/>


---

## Step 5 - Reset Ben's Password in Active Directory

On my DC I opened PowerShell as Administrator.

First I stored the new temporary password as a secure string:

```powershell
$newpassword = ConvertTo-SecureString "Newtemp123!" -AsPlainText -Force
```

Then reset Ben's password using that variable:

```powershell
Set-ADAccountPassword -Identity "bshuto" -Reset -NewPassword $newpassword
```

Then forced Ben to change his password at next login:

```powershell
Set-ADUser -Identity "bshuto" -ChangePasswordAtLogon $true
```

Verified the changes:

```powershell
Get-ADUser -Identity "bshuto" -Properties PasswordExpired, PasswordLastSet | Select Name, PasswordExpired, PasswordLastSet
```

Output showed:
- **PasswordExpired: True**
- **PasswordLastSet: (empty)**

> This is expected and correct behavior. When you force a password change at next logon Active Directory sets PasswordLastSet to 0 internally which shows as empty when queried. It doesn't mean something went wrong, it means AD has flagged the account to require a password change at next login. PasswordLastSet will update to the current date and time once Ben sets his new password.


<p align="center">
<br/>
<img src="https://i.imgur.com/rwohgZA.png" height="50%" width="50%" alt="Disk Sanitization Steps"/>


---

## Step 6 - Posted a Reply to Ben and Set Status to Pending

Back in the staff portal I clicked **Post Reply** on ticket **#685633** and wrote:

*"Hi Ben, thank you for contacting the help desk. I have reset your password. Your temporary password is Newtemp123! Please log into your computer using this temporary password and you will be prompted to create a new one immediately. Please make sure your new password meets the company password requirements. Let us know if you have any issues logging in. Thank you."*

I left the ticket Open instead of Resolved right away because I wanted to wait for Ben to confirm the fix worked before closing it out.

> Best practice is to set the ticket to Pending after sending the fix to the user. This means you're waiting on them to confirm it worked. Only resolve and close the ticket after the issue is fully confirmed. Not just after you made the change in AD.




---

## Step 7 - Verified the Fix on Windows 10

On the Windows 10 VM I logged in as Ben Shuto using the temporary password:

- **Username:** bshuto
- **Password:** Newtemp123!

Windows immediately prompted Ben to set a new password. I set a new password and Ben was able to log in successfully.

<p align="center">
<br/>
<img src="https://i.imgur.com/NFr7UGM.png" height="50%" width="50%" alt="Disk Sanitization Steps"/>


<p align="center">
<br/>
<img src="https://i.imgur.com/hUAwJYW.png" height="50%" width="50%" alt="Disk Sanitization Steps"/>


<p align="center">
<br/>
<img src="https://i.imgur.com/UtNkIzj.png" height="50%" width="50%" alt="Disk Sanitization Steps"/>


<p align="center">
<br/>
<img src="https://i.imgur.com/lzjBOJU.png" height="50%" width="50%" alt="Disk Sanitization Steps"/>


<p align="center">
<br/>
<img src="https://i.imgur.com/VgZIHKl.png" height="50%" width="50%" alt="Disk Sanitization Steps"/>


---

## Step 8 - Closed the Ticket

Back in the staff portal I posted a final closing note on ticket **#685633:**

*"Verified Ben was able to log in successfully with the temporary password and set a new password. Issue fully resolved."*

Changed the ticket status to **Resolved** and posted the reply. Ticket closed.


<p align="center">
<br/>
<img src="https://i.imgur.com/Twt4kLH.png" height="50%" width="50%" alt="Disk Sanitization Steps"/>


<p align="center">
<br/>
<img src="https://i.imgur.com/aW4EsMT.png" height="50%" width="50%" alt="Disk Sanitization Steps"/>


---

## Full Ticket Timeline

| Action | Details |
|---|---|
| Ticket opened | Ben submitted ticket from end user portal |
| Ticket assigned | Kenny Panyavong picked up ticket #685633 |
| Internal note posted | Documented plan before making AD changes |
| Password reset | Reset in AD using Set-ADAccountPassword |
| Force password change | Set-ADUser -ChangePasswordAtLogon $true |
| Reply posted | Sent Ben his temporary password, status set to Pending |
| Fix verified | Ben logged in with temp password and set new password |
| Ticket closed | Status changed to Resolved after confirming fix |

---

## PowerShell Commands Used

```powershell
# Verify account is enabled
Get-ADUser -Identity "bshuto" -Properties Enabled | Select Name, Enabled

# Store temporary password as secure string
$newpassword = ConvertTo-SecureString "Newtemp123!" -AsPlainText -Force

# Reset the password
Set-ADAccountPassword -Identity "bshuto" -Reset -NewPassword $newpassword

# Force password change at next login
Set-ADUser -Identity "bshuto" -ChangePasswordAtLogon $true

# Verify the changes
Get-ADUser -Identity "bshuto" -Properties PasswordExpired, PasswordLastSet | Select Name, PasswordExpired, PasswordLastSet
```

---

## Error I Hit

### Cannot bind parameter NewPassword - cannot convert System.String to System.Security.SecureString

When I first tried running `Set-ADAccountPassword` with the password typed directly into the command PowerShell threw this error:

```
Cannot bind parameter 'NewPassword'. Cannot convert the "Newtemp123!" value
of type "System.String" to type "System.Security.SecureString"
```

**What caused it:** `Set-ADAccountPassword` requires the password to be passed as a SecureString object - not plain text. Typing it directly into the command passes it as a regular string which PowerShell can't convert automatically.

**How I fixed it:** Stored the password in a variable first using `ConvertTo-SecureString` which converts it to the right format, then passed that variable to `Set-ADAccountPassword`:

```powershell
$newpassword = ConvertTo-SecureString "Newtemp123!" -AsPlainText -Force
Set-ADAccountPassword -Identity "bshuto" -Reset -NewPassword $newpassword
```

---

## What I Learned From This Scenario

**Don't close the ticket until the fix is confirmed.** After resetting the password I set the ticket to Pending instead of immediately resolving it. I only closed it after logging in as Ben and confirming he could set a new password and get in. That's the professional way to handle it - the fix isn't done until the user confirms it works.


**SecureString is a security feature not a bug.** PowerShell requires passwords to be passed as SecureString objects to prevent plain text passwords from being exposed in logs or command history. Understanding why that requirement exists is just as important as knowing how to work around it.

**PasswordLastSet being empty is expected.** I initially thought something went wrong when the verification showed PasswordLastSet as empty after forcing a password change. It's actually correct behavior, AD sets it to 0 internally to flag the account for a required password change. It updates automatically when the user sets their new password.

**Password resets are the most common help desk ticket.** Almost every help desk job will have password resets as a daily task. Knowing how to do it properly in AD, force a change at next login, and communicate the temporary password to the user clearly and professionally is a core skill.
