# osTicket Help Desk Lab — Ubuntu Server + Active Directory

This is a documentation of me setting up osTicket in my home Active Directory lab. I'm learning IT and working toward a help desk career, so I wanted to build something that actually mirrors what you'd do on the job — a real ticketing system connected to a real AD domain. This guide covers everything I did, all the errors I ran into, and how I fixed them.

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
7. [Connect to Active Directory](#phase-7--connect-osticket-to-active-directory)
8. [Add DNS Record on DC](#phase-8--add-dns-record-on-the-domain-controller)
9. [Errors I Hit and How I Fixed Them](#errors-i-hit-and-how-i-fixed-them)
10. [What I Learned](#what-i-learned)

---

## Phase 1 — Create the Ubuntu Server VM

1. Download **Ubuntu Server 22.04 LTS** from [ubuntu.com/download/server](https://ubuntu.com/download/server)
2. In VirtualBox click **New**:
   - Name: `SRV-osTicket`
   - Type: Linux / Ubuntu (64-bit)
   - RAM: 1024–2048 MB
   - Hard disk: 20 GB (dynamically allocated)
3. Go to **Settings → Network** and set up adapters to match your lab:
   - One adapter on **Internal Network** (same name as your DC and Windows 10 VMs)
   - One adapter on **NAT** (so Ubuntu can reach the internet to download packages)
4. Mount the Ubuntu ISO under **Settings → Storage** and boot
5. Walk through the installer — set a username, password, and **check the box for OpenSSH Server**
6. Let it install and reboot

> **Note:** Make sure all your VMs that need to talk to each other are on the same Internal Network name in VirtualBox. I had mine set to `mytest`. If the names don't match exactly, the VMs are on completely separate virtual switches and can't reach each other.

---

## Phase 2 — Initial Ubuntu Configuration

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

> YAML is very picky — use spaces only, never tabs. Every indent is 2 spaces. One wrong character will break the whole file.

Save with `Ctrl+O` → Enter → `Ctrl+X`, then apply:

```bash
sudo netplan try
```

Verify the IP:
```bash
ip addr show
```

### Set the Hostname

```bash
sudo hostnamectl set-hostname osticket-srv
```

### Update the Hosts File

```bash
sudo nano /etc/hosts
```

Add:
```
192.168.1.10    dc01.mydomain.com    dc01
```

### Update Ubuntu

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Phase 3 — Install the LAMP Stack

osTicket needs Apache, MySQL, and PHP to run. Install them all:

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

Test it by going to `http://192.168.1.20` in your browser on the Windows 10 VM — you should see the Apache default page.

---

## Phase 4 — Set Up the MySQL Database

```bash
sudo mysql
```

Run these inside MySQL (change the password to something strong):

```sql
CREATE DATABASE osticket;
CREATE USER 'osticket'@'localhost' IDENTIFIED BY 'YourStrongPassword123!';
GRANT ALL PRIVILEGES ON osticket.* TO 'osticket'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

## Phase 5 — Install osTicket

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

Paste this in (replace `mydomain.com` with your actual domain):

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

Enable it and reload:

```bash
sudo a2ensite osticket.conf
sudo a2dissite 000-default.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

> Make sure `configtest` says `Syntax OK` before reloading. If it shows an error it will tell you exactly which line is broken.

---

## Phase 6 — Run the osTicket Web Installer

1. On your Windows 10 VM go to `http://192.168.1.20/setup`
2. All PHP extensions should show green checkmarks
3. Fill in the form:

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

## Phase 7 — Connect osTicket to Active Directory

### Install the LDAP Plugin

First download the plugin files from the osTicket plugins repo:

```bash
cd /tmp
curl -L "https://github.com/osTicket/osTicket-plugins/zipball/develop" -o plugins.zip
unzip plugins.zip
ls
```

Move the auth-ldap folder into the plugins directory (replace the folder name with whatever was created):

```bash
sudo mv osTicket-osTicket-plugins-*/auth-ldap /var/www/html/osticket/include/plugins/
sudo chown -R www-data:www-data /var/www/html/osticket/include/plugins/auth-ldap
```

Install the Net_LDAP2 library that the plugin needs:

```bash
cd /tmp
wget https://github.com/pear/Net_LDAP2/archive/refs/heads/master.zip -O ldap2.zip
unzip ldap2.zip
sudo mkdir -p /var/www/html/osticket/include/Net
sudo cp -r Net_LDAP2-master/Net/LDAP2.php /var/www/html/osticket/include/Net/
sudo cp -r Net_LDAP2-master/Net/LDAP2 /var/www/html/osticket/include/Net/
sudo chown -R www-data:www-data /var/www/html/osticket/include/Net/
sudo systemctl restart apache2
```

### Install Plugin in osTicket

1. Go to `http://192.168.1.20/scp` and log in
2. Go to **Admin Panel → Manage → Plugins** (or go directly to `http://192.168.1.20/scp/plugins.php`)
3. Click **Install** next to the LDAP plugin then **Enable** it
4. Go to **Instances → Add New Instance** and fill in:

| Field | Value |
|---|---|
| Default Domain | `mydomain.com` |
| DNS Servers | *(leave blank)* |
| LDAP Servers | `192.168.1.10` |
| Use TLS | Unchecked |
| Search User | `svc-osticket@mydomain.com` |
| Password | your service account password |
| Search Base | `DC=mydomain,DC=com` |
| LDAP Schema | Active Directory |
| Staff Authentication | ✅ Enabled |
| Client Authentication | ✅ Enabled |

### Create the LDAP Service Account on the DC

On your Domain Controller open PowerShell:

```powershell
New-ADUser -Name "osTicket Service" -SamAccountName "svc-osticket" `
  -UserPrincipalName "svc-osticket@mydomain.com" `
  -AccountPassword (Read-Host -AsSecureString "Password") `
  -PasswordNeverExpires $true -Enabled $true
```

> Never use your main domain admin account as the LDAP bind account. A dedicated service account is better practice.

---

## Phase 8 — Add DNS Record on the Domain Controller

1. Open **DNS Manager** on the DC
2. Expand your domain → **Forward Lookup Zones**
3. Right-click your domain → **New Host (A or AAAA)**
4. Set Name: `osticket-srv` and IP: `192.168.1.20`
5. Click **Add Host**

Now you can reach osTicket at `http://osticket-srv.mydomain.com`

---

## Errors I Hit and How I Fixed Them

### Invalid YAML: mapping values are not allowed in this context

This happened because there was a stray word `naml` at the top of my netplan file and the indentation was wrong under `nameservers`. YAML is extremely strict — one wrong character breaks the whole file.

**Fix:** Run `cat -A` on the file to reveal hidden characters, wipe it with `sudo truncate -s 0`, and retype it carefully using only spaces.

---

### `systemd-networkd is not running` during netplan try

This warning showed up after applying the netplan config. I panicked but it turned out to be safe to ignore. Ubuntu Server sometimes uses a different network backend. As long as `ip addr show` shows your static IP, everything is fine.

---

### `command not found: hostnamect1`

I typed the number `1` at the end instead of the lowercase letter `l`. Easy mistake to make when you're staring at a terminal.

- ❌ `hostnamect1`
- ✅ `hostnamectl`

---

### Apache failed to reload — Syntax error on line 12

The error said it expected `</VirtualHost>` but saw `</VitualHost>`. I had missed the `r` in Virtual. Apache won't start with mismatched tags.

**Fix:** Always run `sudo apache2ctl configtest` before reloading Apache. It tells you exactly which line is broken.

---

### Windows 10 couldn't reach the Apache default page

This took a while to figure out. The problem was that my Ubuntu VM was on a completely different subnet (`192.168.1.x`) than my Windows 10 VM (`192.168.30.x`) because my DC wasn't on yet. Without the DC running there's no DHCP, no routing, and no gateway — so nothing works.

**Fix:** Always boot the DC first. Once the DC was on, Windows 10 got the right IP and everything was on the same subnet.

---

### Ubuntu couldn't download packages — no internet

Ubuntu was set to Internal Network only, which is completely isolated from the internet by design. I needed a NAT adapter for internet access.

**Fix:** Add a third adapter in VirtualBox set to NAT. Then update netplan so that adapter uses DHCP while the Internal Network adapter keeps the static IP.

---

### Failed opening required `include/Net/LDAP2.php`

The LDAP plugin couldn't find the Net_LDAP2 PHP library it needed. osTicket was looking in `include/Net/` but the files weren't there.

**Fix:** Manually download the Net_LDAP2 library from GitHub and place it in exactly the right folder:

```bash
sudo mkdir -p /var/www/html/osticket/include/Net
sudo cp -r Net_LDAP2-master/Net/LDAP2.php /var/www/html/osticket/include/Net/
sudo cp -r Net_LDAP2-master/Net/LDAP2 /var/www/html/osticket/include/Net/
```

---

### osTicket LDAP plugin download returning 404

The official osTicket plugins repo has no formal releases — all the release tag URLs return 404. Had to download directly from the branch instead using curl:

```bash
curl -L "https://github.com/osTicket/osTicket-plugins/zipball/develop" -o plugins.zip
```

---

## What I Learned

This project taught me way more than just how to install osTicket. Here's what actually stuck with me:

**Networking fundamentals matter.** I spent a lot of time troubleshooting why VMs couldn't talk to each other. I learned about subnets, how VirtualBox Internal Network works, why adapter names have to match exactly, and why NAT and Internal Network serve completely different purposes.

**Boot order matters in a domain environment.** The DC has to come up first — everything else depends on it for DNS, DHCP, and routing. This is true in real enterprise environments too.

**YAML is unforgiving.** One wrong character, one tab instead of spaces, one stray word and the whole config breaks. I learned to always use `cat -A` to check for hidden characters and `netplan try` to validate before applying.

**Read error messages carefully.** Almost every error told me exactly what was wrong if I slowed down and read it. The Apache syntax error told me the exact line number and what it expected. The LDAP error told me exactly which file path it was looking for.

**Nothing works perfectly the first time and that's okay.** Every error I hit was a learning opportunity. Troubleshooting these issues gave me way more hands-on experience than if everything had worked first try.

---

## Help Desk Scenarios to Practice

Now that osTicket is running I can simulate real tickets:

| Ticket | AD Task |
|---|---|
| User account locked out | `Unlock-ADAccount -Identity "jsmith"` |
| New hire needs account | `New-ADUser ...` |
| Password reset request | `Set-ADAccountPassword` + force change at logon |
| User needs access to shared folder | Modify NTFS/Share permissions |
| Push software to all computers | Create a GPO with software deployment |

---

## Resources

- [osTicket Documentation](https://docs.osticket.com)
- [Ubuntu Server 22.04 Docs](https://ubuntu.com/server/docs)
- [Microsoft AD PowerShell Reference](https://learn.microsoft.com/en-us/powershell/module/activedirectory/)
- [VirtualBox Networking Explained](https://www.virtualbox.org/manual/ch06.html)
