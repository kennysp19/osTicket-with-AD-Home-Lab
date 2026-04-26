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

1. [Create the Ubuntu VM](#phase-1:--create-the-ubuntu-server-vm)
2. [Configure Ubuntu](#phase-2:-initial-ubuntu-configuration)
3. [Install LAMP Stack](#phase-3:-install-the-lamp-stack)
4. [Set Up MySQL Database](#phase-4:-set-up-the-mysql-database)
5. [Install osTicket](#phase-5:-install-osticket)
6. [Run the Web Installer](#phase-6:-run-the-osticket-web-installer)
7. [LDAP/AD Integration — Skipped](#phase-7:-ldapad-integration)
8. [Add DNS Record on DC](#phase-8:-add-dns-record-on-the-domain-controller)
9. [Manual User Setup in osTicket](#manual-user-setup-in-osticket)
10. [Errors I Hit and How I Fixed Them](#errors-i-hit-and-how-i-fixed-them)
11. [What I Learned](#what-i-learned)

---

## Phase 1: Create the Ubuntu Server VM

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

## Phase 2: Initial Ubuntu Configuration

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

## Phase 3: Install the LAMP Stack

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

## Phase 4: Set Up the MySQL Database

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

## Phase 5: Install osTicket

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

## Phase 6: Run the osTicket Web Installer

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

## Phase 7: LDAP/AD Integration

I originally planned to connect osTicket to Active Directory using LDAP so that domain users could log in with their AD credentials automatically. However I ran into issues getting the LDAP plugin and its required libraries to work correctly with osTicket v1.18.1. The plugin kept throwing a `Failed opening required include/Net/LDAP2.php` error that I couldn't resolve. The library wasn't being found even after manually placing it in multiple locations.

Rather than get stuck on this one piece I made the decision to skip it and move forward. OsTicket works completely fine without it. Users and agents are created manually inside osTicket instead.

This is something I plan to come back to and figure out as I continue building my skills.

**update:** I was actually able to get it working.

---

## Phase 8: Add DNS Record on the Domain Controller

1. On the DC open **DNS Manager**
2. Expand your domain → **Forward Lookup Zones**
3. Right-click your domain → **New Host (A or AAAA)**
4. Set Name: `osticket-srv` and IP: `192.168.1.20`
5. Click **Add Host**

Now you can reach osTicket at `http://osticket-srv.mydomain.com` from your Windows 10 VM.

---

## Manual User Setup in osTicket

Since I skipped LDAP integration, users and agents are created manually inside osTicket. This is straightforward and still lets me practice all the help desk workflows I want to.

### Create Staff/Agent Accounts
1. Log into `http://192.168.1.20/scp`
2. Go to **Admin Panel → Agents → Add New Agent**
3. Fill in name, email, username, and password
4. Assign a department and role

### Create End Users
1. Go to **Admin Panel → Users → Add New User**
2. Fill in name and email
3. End users can submit tickets at `http://192.168.1.20`

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

## Help Desk Scenarios to Practice

Now that osTicket is running I can simulate real help desk tickets:

| Ticket | What I Do |
|---|---|
| User account locked out | `Unlock-ADAccount -Identity "jsmith"` then close ticket |
| New hire needs account | Create in AD + create in osTicket, document steps |
| Password reset request | `Set-ADAccountPassword` + force change at logon |
| User needs folder access | Modify NTFS/Share permissions, update ticket |
| Software install request | Deploy via GPO, resolve ticket with notes |

---

## Images

Coming soon!


## Resources

- [osTicket Documentation](https://docs.osticket.com)
- [Ubuntu Server 22.04 Docs](https://ubuntu.com/server/docs)
- [Microsoft AD PowerShell Reference](https://learn.microsoft.com/en-us/powershell/module/activedirectory/)
- [VirtualBox Networking Explained](https://www.virtualbox.org/manual/ch06.html)
