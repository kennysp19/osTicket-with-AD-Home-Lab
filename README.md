# osTicket Help Desk Lab Ubuntu Server + Active Directory

This is a documentation of me setting up osTicket in my home Active Directory lab. I'm learning IT and wanted to build something that actually mirrors what you'd do on the job. A real ticketing system running in the same environment as my Active Directory domain. This guide covers everything I did, all the errors I ran into, and how I fixed them.

---

## My Lab Environment

| VM | OS | Role | RAM |
|---|---|---|---|
| DC01 | Windows Server 2022 | Active Directory, DNS, DHCP, RAS | 2GB |
| SRV-osTicket | Ubuntu Server 22.04 LTS | osTicket Web Server | 1–2GB |
| PC01 | Windows 10 | Client / End User | 2GB |

**Virtualization Platform:** Oracle VirtualBox

> **Important:** Always boot the DC first before anything else. Without it running, nothing gets an IP, DNS doesn't work, and nothing can communicate. I learned this the hard way.

---

## Table of Contents

1. [Create the Ubuntu VM](#phase-1--create-the-ubuntu-server-vm)
2. [Configure Ubuntu](#phase-2--initial-ubuntu-configuration)
3. [Install LAMP Stack](#phase-3--install-the-lamp-stack)
4. [Set Up MySQL Database](#phase-4--set-up-the-mysql-database)
5. [Install osTicket](#phase-5--install-osticket)
6. [Run the Web Installer](#phase-6--run-the-osticket-web-installer)
7. [LDAP/AD Integration — Skipped](#phase-7--ldapad-integration)
8. [Add DNS Record on DC](#phase-8--add-dns-record-on-the-domain-controller)
9. [Manual User Setup in osTicket](#manual-user-setup-in-osticket)
10. [Errors I Hit and How I Fixed Them](#errors-i-hit-and-how-i-fixed-them)
11. [What I Learned](#what-i-learned)

---

## Phase 1  Create the Ubuntu Server VM

1. Download **Ubuntu Server 22.04 LTS** from [ubuntu.com/download/server](https://ubuntu.com/download/server)
2. In VirtualBox click **New** and configure:
   - Name: `SRV-osTicket`
   - Type: Linux / Ubuntu (64-bit)
   - RAM: 1024–2048 MB
   - Hard disk: 20 GB (dynamically allocated)
3. Go to **Settings → Network** and set up adapters:
   - Adapter 1: **Internal Network** (same name as your DC and Windows 10 VMs)
   - Adapter 2: **Internal Network** (same name)
   - Adapter 3: **NAT** (for internet access to download packages)
4. Mount the Ubuntu ISO under **Settings → Storage** and boot
5. Walk through the installer — set a username, password, and **check the box for OpenSSH Server**
6. Let it install and reboot

> **Note:** Make sure all your VMs that need to talk to each other are on the same Internal Network name in VirtualBox. Mine was set to `mytest`. If the names don't match exactly the VMs are on completely separate virtual switches and can't reach each other at all.

---

## Phase 2  Initial Ubuntu Configuration

### Set a Static IP

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Replace the contents with this (adjust IPs to match your lab):

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: yes
    enp0s8:
      dhcp4: no
      addresses:
        - 192.168.1.20/24
      nameservers:
        addresses:
          - 192.168.1.10
```

> YAML is very strict — use spaces only, never tabs. Every indent is 2 spaces. One wrong character breaks the whole file. Run `cat -A` on the file to check for hidden characters if you get errors.

Save with `Ctrl+O` → Enter → `Ctrl+X`, then validate and apply:

```bash
sudo netplan try
```

Verify the IP was assigned:

```bash
ip addr show
```

### Set the Hostname

```bash
sudo hostnamectl set-hostname osticket-srv
```

Verify:

```bash
hostname
```

### Update the Hosts File

```bash
sudo nano /etc/hosts
```

Add this line using your actual DC IP and domain:

```
192.168.1.10    dc01.mydomain.com    dc01
```

### Update Ubuntu

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Phase 3  Install the LAMP Stack

osTicket needs Apache, MySQL, and PHP to run. Install them all on Ubuntu:

```bash
sudo apt install apache2 -y
sudo apt install mysql-server -y
sudo apt install php php-mysql php-xml php-intl php-mbstring php-curl php-zip php-gd php-imap php-apcu php-ldap libapache2-mod-php -y
```

Enable the rewrite module and restart Apache:

```bash
sudo a2enmod rewrite
sudo systemctl restart apache2
```

Test it: on your Windows 10 VM open a browser and go to `http://192.168.1.20`. You should see the Apache default page.

---

## Phase 4  Set Up the MySQL Database

Run this on Ubuntu:

```bash
sudo mysql
```

Then run these commands inside MySQL (change the password to something strong):

```sql
CREATE DATABASE osticket;
CREATE USER 'osticket'@'localhost' IDENTIFIED BY 'YourStrongPassword123!';
GRANT ALL PRIVILEGES ON osticket.* TO 'osticket'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

## Phase 5  Install osTicket

### Download and Extract

```bash
cd /tmp
wget https://github.com/osTicket/osTicket/releases/download/v1.18.1/osTicket-v1.18.1.zip
sudo apt install unzip -y
unzip osTicket-v1.18.1.zip
```

### Move Files to Web Root

```bash
sudo mv upload /var/www/html/osticket
```

### Set Up Config File

```bash
sudo cp /var/www/html/osticket/include/ost-sampleconfig.php /var/www/html/osticket/include/ost-config.php
sudo chmod 0666 /var/www/html/osticket/include/ost-config.php
sudo chown -R www-data:www-data /var/www/html/osticket
```

### Create Apache Virtual Host

```bash
sudo nano /etc/apache2/sites-available/osticket.conf
```

Paste this in (replace `mydomain.com` with your actual domain name):

```apache
<VirtualHost *:80>
    ServerName osticket-srv.mydomain.com
    DocumentRoot /var/www/html/osticket

    <Directory /var/www/html/osticket>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/osticket_error.log
    CustomLog ${APACHE_LOG_DIR}/osticket_access.log combined
</VirtualHost>
```

Enable the site and reload:

```bash
sudo a2ensite osticket.conf
sudo a2dissite 000-default.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

> Always run `apache2ctl configtest` before reloading. It will tell you exactly which line has a problem if something is wrong.

---

## Phase 6  Run the osTicket Web Installer

1. On your Windows 10 VM go to `http://192.168.1.20/setup`
2. All PHP extensions should show green checkmarks
3. Fill in the setup form:

| Field | Value |
|---|---|
| Helpdesk Name | My Lab Help Desk |
| Default Email | helpdesk@mydomain.com |
| Admin First Name | Lab |
| Admin Last Name | Admin |
| Admin Email | admin@mydomain.com |
| Admin Username | labadmin |
| Admin Password | something strong |
| Database Host | localhost |
| Database Name | osticket |
| Database User | osticket |
| Database Password | whatever you set in Phase 4 |

4. Click **Install Now**
5. After it finishes lock down the config and remove the setup folder:

```bash
sudo chmod 0644 /var/www/html/osticket/include/ost-config.php
sudo rm -rf /var/www/html/osticket/setup
```

---

## Phase 7  LDAP/AD Integration

This is the part I almost gave up on. I originally skipped LDAP integration because I kept running into errors I couldn't figure out. But I came back to it and eventually got it working. This documents everything I did, every error I hit, and exactly what fixed it.

The idea behind LDAP is that instead of creating separate accounts inside osTicket for every user, osTicket talks directly to Active Directory. When someone logs in with their domain username and password osTicket checks with the DC to verify it. Way cleaner than managing two separate sets of accounts.

---

## Step 1 — Install the LDAP Plugin

The LDAP plugin isn't included with osTicket by default. I had to download it separately. The official download links on the osTicket plugins repo kept returning 404 errors because the repo has no formal releases at all. I eventually got it working by downloading directly from the branch using curl on Ubuntu:

```bash
cd /tmp
curl -L "https://github.com/osTicket/osTicket-plugins/zipball/develop" -o plugins.zip
unzip plugins.zip
ls
```

That created a folder with a long random name like `osTicket-osTicket-plugins-a1b2c3d`. I moved the auth-ldap folder from it into the osTicket plugins directory:

```bash
sudo mv osTicket-osTicket-plugins-*/auth-ldap /var/www/html/osticket/include/plugins/
sudo chown -R www-data:www-data /var/www/html/osticket/include/plugins/auth-ldap
sudo systemctl restart apache2
```

---

## Step 2 — Install the Net_LDAP2 Library

When I tried to configure the plugin I kept getting this error:

```
Failed opening required 'include/Net/LDAP2.php'
```

The plugin needs a PHP library called Net_LDAP2 that wasn't bundled with it. I downloaded it manually on Ubuntu:

```bash
cd /tmp
wget https://github.com/pear/Net_LDAP2/archive/refs/heads/master.zip -O ldap2.zip
unzip ldap2.zip
sudo mkdir -p /var/www/html/osticket/include/plugins/auth-ldap/include/Net
sudo cp -r Net_LDAP2-master/Net/LDAP2.php /var/www/html/osticket/include/plugins/auth-ldap/include/Net/
sudo cp -r Net_LDAP2-master/Net/LDAP2 /var/www/html/osticket/include/plugins/auth-ldap/include/Net/
sudo chown -R www-data:www-data /var/www/html/osticket/include/plugins/
sudo systemctl restart apache2
```

---

## Error — PHP Fatal Error: Cannot Redeclare Class Net_LDAP2

While trying to fix the LDAP2.php error I made it worse by copying the library into multiple locations. PHP started loading it twice and the whole site crashed with a 500 error:

```
PHP Fatal error: Cannot redeclare class Net_LDAP2 (previously declared in
/var/www/html/osticket/include/plugins/auth-ldap/include/Net/LDAP2.php:57)
in /var/www/html/osticket/include/Net/LDAP2.php on line 57
```

The fix was to remove all the duplicate copies and leave it only in the plugin's own folder:

```bash
sudo rm -rf /var/www/html/osticket/include/Net
sudo rm -rf /var/www/html/osticket/include/pear/Net
sudo systemctl restart apache2
```

Verified only one copy existed:

```bash
find /var/www/html/osticket -name "LDAP2.php"
```

Output showed only one result:

```
/var/www/html/osticket/include/plugins/auth-ldap/include/Net/LDAP2.php
```

That's exactly where it needed to be.

---

## Step 3 — Install and Enable the Plugin in osTicket

On my Windows 10 VM I went to:

```
http://192.168.1.20/scp/plugins.php
```

The LDAP Authentication and Lookup plugin showed up in the list:

```
LDAP Authentication and Lookup — Version 0.6.2 — Installed: 4/21/26
```

Clicked **Install** then **Enable** to activate it.

---

## Step 4 — Create a Dedicated Service Account on the DC

Before configuring the LDAP instance I created a dedicated AD account for osTicket to use when connecting to the DC. You never want to use your main admin account for this — a dedicated service account with minimal permissions is the right way to do it.

On my DC in PowerShell as Administrator:

```powershell
New-ADUser -Name "osTicket Service" -SamAccountName "svc-osticket" -UserPrincipalName "svc-osticket@mydomain.com" -PasswordNeverExpires $true -Enabled $false
```

```powershell
Set-ADAccountPassword -Identity "svc-osticket" -Reset -NewPassword (Read-Host -AsSecureString "New Password")
```

```powershell
Enable-ADAccount -Identity "svc-osticket"
```

Verified the account was created and enabled:

```powershell
Get-ADUser -Identity "svc-osticket" -Properties Enabled | Select Name, Enabled
```

Output showed `Enabled: True`

> My DC kept freezing while running these commands. If that happens it's a resource issue — shut down the DC VM and increase the RAM to at least 3GB and CPU to 2 cores in VirtualBox settings. Also make sure PowerShell is always opened as Administrator on the DC or you'll get Access Denied errors.

---

## Step 5 — Configure the LDAP Instance in osTicket

In osTicket I went to:

**Admin Panel → Manage → Plugins → LDAP Authentication and Lookup → Instances → Add New Instance**

Filled in the following settings:

| Field | Value |
|---|---|
| **Default Domain** | `mydomain.com` |
| **DNS Servers** | *(left blank)* |
| **LDAP Servers** | `192.168.1.10` |
| **Use TLS** | Unchecked |
| **Search User** | `svc-osticket@mydomain.com` |
| **Password** | svc-osticket account password |
| **Search Base** | `DC=mydomain,DC=com` |
| **LDAP Schema** | Active Directory |
| **Staff Authentication** | ✅ Enabled |
| **Client Authentication** | ✅ Enabled |

> For the Search Base just split your domain name at every dot and put `DC=` in front of each part separated by commas. So `mydomain.com` becomes `DC=mydomain,DC=com`. If your domain was `lab.local` it would be `DC=lab,DC=local`.

> Leave DNS Servers blank. Since Windows 10 is already connected to the DC and resolving DNS correctly the plugin will find the DC automatically using the Default Domain field.

---

## Error — Unable to Connect to LDAP Server

When I first tried saving the instance I got this error:

```
Unable to connect any listed LDAP servers
Bind failed: Can't contact LDAP server: Unable to bind to server 192.168.1.10
```

In my case this was caused by my DC being frozen — not a configuration problem at all. After rebooting the DC the connection issue went away.

> Always check that your DC is fully booted and responding before troubleshooting LDAP connection errors. A frozen or unresponsive DC will cause this error even if your config is perfectly correct.

---

## Step 6 — Test the LDAP Connection From Ubuntu

Before saving the instance I tested the connection directly from Ubuntu using ldap-utils to confirm everything was working at the network level:

```bash
sudo apt install ldap-utils -y
```

```bash
ldapsearch -x -H ldap://192.168.1.10 -D "svc-osticket@mydomain.com" -W -b "DC=mydomain,DC=com"
```

It prompted me for the svc-osticket password.

First attempt returned **Invalid credentials** — I had forgotten the password I set for the service account. I reset it on the DC:

```powershell
Set-ADAccountPassword -Identity "svc-osticket" -Reset -NewPassword (Read-Host -AsSecureString "New Password")
Enable-ADAccount -Identity "svc-osticket"
```

Ran ldapsearch again with the new password. This time it returned a full list of AD objects confirming LDAP was working correctly.

---

## Step 7 — Update Password in osTicket and Save

Went back to the LDAP instance in osTicket, updated the password field with the new svc-osticket password and clicked save.

**Result: Instance updated successfully.**

LDAP was finally connected.

---

## Step 8 — Add AD Users as Staff Agents

LDAP handles authentication — checking if the username and password are correct. But osTicket still needs to know who each agent is and what role and department they have. Just because LDAP is connected doesn't mean every AD user can automatically log in.

To add an AD user as a staff agent:

1. Log into `http://192.168.1.20/scp`
2. Go to **Admin Panel → Agents → Add New Agent**
3. Fill in:

| Field | Value |
|---|---|
| **First Name** | user's first name |
| **Last Name** | user's last name |
| **Email** | their AD email |
| **Username** | their AD username exactly as it appears in AD |

4. Uncheck **Send a password reset email** and **Require password change at next login**
5. Click the **Access** tab and assign a **Department** and **Role**
6. Click **Create Agent**

That agent can now log into `http://192.168.1.20/scp` using their AD username and password.

---

## End User Login

End users log in through a completely different portal than staff agents:

| Portal | URL | Who Uses It |
|---|---|---|
| **Staff/Agent** | `http://192.168.1.20/scp` | Help desk agents and admins |
| **End User** | `http://192.168.1.20` | Regular users submitting tickets |

For end users to log in with their AD credentials make sure **Client Authentication** is enabled in the LDAP instance settings. If they still can't get in add them manually:

1. Go to **Admin Panel → Users → Add New User**
2. Enter their AD email and username
3. They can now log in at `http://192.168.1.20`

---

## Errors I Hit and How I Fixed Them

### Failed opening required include/Net/LDAP2.php

The LDAP plugin needed the Net_LDAP2 PHP library which wasn't bundled with it. Downloaded it manually from GitHub and placed it inside the plugin's include folder.

---

### PHP Fatal Error: Cannot Redeclare Class Net_LDAP2 — HTTP 500 Error

Copied the library into too many locations trying to fix the above error. PHP was loading it twice and crashing the whole site. Fixed by removing all duplicate copies and leaving the library only in the plugin's own folder.

---

### Unable to Connect to LDAP Server — Bind Failed

My DC was frozen. Rebooted it and the error went away. Always check your DC is actually up and running before troubleshooting LDAP connection errors.

---

### Invalid Credentials on ldapsearch

Forgot the password set for the svc-osticket service account. Reset it on the DC using Set-ADAccountPassword and tried again.

---

### Access Denied After LDAP Connected

LDAP authenticates users but doesn't automatically give them access inside osTicket. Had to create agent accounts inside osTicket and assign them departments and roles before they could log in.

---

### osTicket Plugin Download Returning 404

The osTicket plugins repo has no formal releases so all release tag download URLs return 404. Fixed by downloading directly from the branch using curl instead of wget.

---

## What I Learned

**LDAP authenticates but doesn't authorize.** This was the biggest thing I learned. Authentication means proving who you are — LDAP handles that by checking AD. Authorization means what you're allowed to do — osTicket handles that through roles and departments. They're two separate things and you need both set up for users to actually get in.

**Test connectivity in layers.** When LDAP wasn't connecting I tested ping first, then used ldapsearch to test the actual LDAP bind directly on Ubuntu. Breaking it down step by step made it way easier to find exactly where the problem was instead of guessing.

**Duplicate files cause conflicts.** Copying the LDAP2 library into multiple locations seemed like a good idea at the time but caused PHP to load it twice and crash the site. The lesson is to put files exactly where they need to go and nowhere else.

**Always check the obvious stuff first.** The bind error turned out to be caused by a frozen DC — not a config issue at all. Before going deep into troubleshooting always make sure all your VMs are actually up and responding.

**Read error messages carefully.** Every error in this phase told me exactly what was wrong if I slowed down and read it. The PHP error told me the exact file causing the conflict. The ldapsearch error said invalid credentials which told me immediately it was a password issue not a network issue. The errors are helpful if you actually read them.

**update:** I was actually able to get it working.

---

## Phase 8  Add DNS Record on the Domain Controller

1. On the DC open **DNS Manager**
2. Expand your domain → **Forward Lookup Zones**
3. Right-click your domain → **New Host (A or AAAA)**
4. Set Name: `osticket-srv` and IP: `192.168.1.20`
5. Click **Add Host**

Now you can reach osTicket at `http://osticket-srv.mydomain.com` from your Windows 10 VM.

---

## Errors I Hit and How I Fixed Them

### Invalid YAML: mapping values are not allowed in this context

This happened because there was a stray word at the top of my netplan file and the indentation was off. YAML is extremely strict — one wrong character breaks the whole file.

**Fix:** Run `cat -A` on the file to reveal hidden characters like tabs (`^I`) and Windows line endings (`^M$`). Wipe the file with `sudo truncate -s 0` and retype it carefully using only spaces.

---

### `systemd-networkd is not running` during netplan try

This warning showed up after applying the netplan config. It's safe to ignore. Ubuntu Server sometimes uses a different network backend. As long as `ip addr show` shows your static IP everything is working fine.

---

### `command not found: hostnamect1`

I typed the number `1` at the end instead of the lowercase letter `l`. Easy mistake when staring at a terminal.

- ❌ `hostnamect1` — number one
- ✅ `hostnamectl` — lowercase letter L

---

### Apache failed to reload, Syntax error on line 12

The error said it expected `</VirtualHost>` but saw `</VitualHost>`. I had missed the `r` in Virtual. Apache won't start with mismatched tags.

**Fix:** Always run `sudo apache2ctl configtest` before reloading Apache. It tells you the exact line number and what it expected to find.

---

### Windows 10 couldn't reach the Apache default page

This took a while to figure out. My Ubuntu VM was on a different subnet than my Windows 10 VM because the DC wasn't on yet. Without the DC there's no DHCP, no routing, and no default gateway so nothing can communicate.

**Fix:** Always boot the DC first. Once it was running Windows 10 grabbed the right IP and everything was on the same subnet.

---

### Ubuntu couldn't download packages — no internet

Ubuntu was only on Internal Network which is completely isolated from the internet by design.

**Fix:** Add a third adapter in VirtualBox set to NAT. Update netplan so the NAT adapter uses DHCP for internet while the Internal Network adapter keeps the static IP.

---

### Failed opening required `include/Net/LDAP2.php`

This was the error that caused me to skip Phase 7. The LDAP plugin couldn't find the Net_LDAP2 PHP library it needed. Even after manually downloading the library and placing it in multiple locations osTicket still couldn't find it. This is something I plan to revisit.

---

### osTicket LDAP plugin returning 404 on download

The osTicket plugins repo has no formal releases so all the release tag download URLs return 404.

---

## What I Learned

This project taught me a lot more than just how to install a ticketing system. Here's what actually stuck with me:

**Networking fundamentals matter.** I spent a good amount of time troubleshooting why VMs couldn't talk to each other. I learned about subnets, how VirtualBox Internal Network works, why adapter names have to match exactly, and why NAT and Internal Network serve completely different purposes.

**Boot order matters in a domain environment.** The DC has to come up first — everything else depends on it for DNS, DHCP, and routing. This is true in real enterprise environments too, not just home labs.

**YAML is unforgiving.** One wrong character, one tab instead of a space, one stray word and the whole config breaks. I learned to always use `cat -A` to check for hidden characters and `netplan try` to validate before applying.

**Read error messages carefully.** Almost every error told me exactly what was wrong if I slowed down and read it. The Apache syntax error told me the exact line number. The LDAP error told me the exact file path it was looking for. The errors are actually helpful if you take the time to read them.

**It's okay to skip something and come back to it.** I couldn't get the LDAP integration working and instead of getting stuck I made the decision to move forward and document why I skipped it. Not everything works perfectly the first time and knowing when to move on is a skill too. I plan to come back to Phase 7 as I keep learning.

**Troubleshooting is the real skill.** Everything going wrong and having to figure out why taught me more than if everything had worked first try.

---


## Resources

- [osTicket Documentation](https://docs.osticket.com)
- [Ubuntu Server 22.04 Docs](https://ubuntu.com/server/docs)
- [Microsoft AD PowerShell Reference](https://learn.microsoft.com/en-us/powershell/module/activedirectory/)
- [VirtualBox Networking Explained](https://www.virtualbox.org/manual/ch06.html)
