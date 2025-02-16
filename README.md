
# Provision & Setup Ubuntu Server 22.04

## Table of Contents
- [Prerequisites](#prerequisites)
- [Network Topology](#network-topology)
- [Email Security Server Overview](#email-security-server-overview)
  - [Overview](#overview)
  - [Security Implications](#security-implications)
- [Setup Security Server](#setup-security-server)
  - [Step 1](#step-1)
- [Connect Ubuntu Desktop to Active Directory](#connect-ubuntu-desktop-to-active-directory)
  - [Realmd + SSSD](#realmd-sssd)
    - [Step 1](#step-1-1)
    - [Step 2](#step-2)
    - [Step 3](#step-3)
    - [Step 4](#step-4)
    - [Step 5](#step-5)
  - [Samba Winbind](#samba-winbind)
    - [Step 1](#step-1-2)
    - [Step 2](#step-2-1)
    - [Step 3](#step-3-1)
    - [Step 4](#step-4-1)
    - [Step 5](#step-5-1)
    - [Step 6](#step-6)
    - [Step 7](#step-7)
    - [Step 8](#step-8)
    - [Step 9](#step-9)
    - [Step 10](#step-10)
    - [Step 11](#step-11)
    - [Step 12](#step-12)
    - [Step 13](#step-13)

## Prerequisites
1. Virtualbox installed.
2. Virtual Machine with Ubuntu 22.04 ISO Server has been configured and provisioned (the ISO should be attached to the new VM).
3. Windows Server 2025 with Active Directory Domain Services (ADDS) configured.

## Network Topology

## Email Security Server Overview

### Overview
An email server is a system designed to send, receive, store, and manage email communication for users. It uses protocols such as SMTP (Simple Mail Transfer Protocol) for sending emails, and IMAP (Internet Message Access Protocol) or POP3 (Post Office Protocol) for receiving and managing email messages.

We will be configuring Postfix as a Mail Transfer Agent (MTA), which is used for sending and routing emails on Linux servers.

Email servers are much less common today than they were 20+ years ago. Running an email server requires expertise with configuring DNS records, securing against spam, developing a good reputation via IP address for email delivery, and more. With the emergence of third-party applications such as Gmail, Microsoft 365, and ProtonMail (for personal use), these managed services provide scalable, secure email without having to manage the infrastructure of an email server.

### Why are we configuring an email server in this project then?
It provides good insight into how email works and why it’s important to secure your email gateways.

This guide will be used to set up the underlying Operating System, Ubuntu 22.04 Server LTS, and connection through Active Directory. Additional guides will be provided for configuring Postfix.

### Security Implications
While running an email server like Postfix on Ubuntu Server 22.04 gives you control, it also introduces several security considerations:

#### Common Threats
1. **Open Relay Exploitation**: If improperly configured, your server can be used by spammers to send large volumes of email, damaging your IP reputation.
2. **Brute Force Attacks**: Attackers often attempt to compromise accounts via brute force or credential stuffing.
3. **Spam and Phishing**: Attackers may spoof your domain or use your server for phishing campaigns.
4. **Data Breaches**: Poorly secured servers can expose sensitive emails and user credentials.
5. **Malware Delivery**: Your server could inadvertently become a vehicle for spreading malware if attachments are not scanned.

## Setup Security Server

### Step 1
Press “Enter”.

Select language.

Continue without updating.

Select “Ubuntu Server”.

Leave default Network configuration → Leave Proxy Page empty.

Leave “Mirror” configuration empty → “Done”

Select “Use an entire disk”. Press the “Tab” key until selecting “Done.”

Leave Storage Configuration as default.

Arrow to “Continue”.

Enter in the server’s hostname, username, and password.

Refer to the “Project Overview” guide for more information on default usernames and passwords.

Select “Skip for now” for the Ubuntu Pro → Select “Install OpenSSH server”. Use the Tab key until selecting “Done”.

Wait for the OS to install, press “enter for a reboot”.

Success!

## Connect Ubuntu Desktop to Active Directory

Switch the Network from “NAT Network” → Bridged.

Refer to this guide if you would like to insert VirtualBox Guest Additions (for copy/paste controls).

Connecting Ubuntu (and Debian-based systems) to Active Directory can be accomplished in a couple ways. The easiest way is to connect Ubuntu to Active Directory with realmd and SSSD.

[https://gist.github.com/magnetikonline/1e7e2dbd1b288fecf090f1ef12f0c80b](https://gist.github.com/magnetikonline/1e7e2dbd1b288fecf090f1ef12f0c80b)

SSSD (System Security Services Daemon) and Samba Winbind can also be used to join Linux systems if realmd / SSSD is not working.

### About SSSD / Realmd

- **System Security Services Daemon (SSSD)**: A service on Linux systems that provides a central access point for identity management and authentication. When connecting a Linux system to Active Directory (AD), SSSD allows for the integration by acting as an intermediary between the Linux system and AD.
- **realmd**: A tool that simplifies the process of joining Linux machines to AD domains.
- **Samba Winbind**: A component of the Samba suite that allows Linux systems to authenticate users against Windows Active Directory (AD).

### Realmd + SSSD

#### Step 1
Open a new terminal session.

Update the system with:

```
sudo apt update
```

#### Step 2
Adding the following under the [Time] block.

```
sudo nano /etc/systemd/timesyncd.conf
```

Install the necessary packages:

```
sudo apt install realmd sssd sssd-tools samba-common krb5-user packagekit libnss-sss libpam-sss adcli samba-common-bin
```

#### Step 3
Use the realm command to discover the domain.

#### Step 4
Enter the following command, enter the Administrator password:

```
sudo realm join --verbose --user=Administrator corp.project-x-dc.com
```

#### Step 5
If no output is shown in the console, then the VM has been connected.

Enter the following command to confirm:

```
realm list
```

### Samba Winbind

#### Step 1
Open a new terminal session.

Update the system with:

```
sudo apt update
```

Install the necessary packages (note that krb5-user is installed this time):

```
sudo apt -y install winbind libpam-winbind libnss-winbind krb5-config krb5-user samba-dsdb-modules samba-vfs-modules
```

Add CORP.PROJECT-X-DC.COM for the two Kerberos Authentication pages.

#### Step 2
Move the smb.conf.org file:

```
mv /etc/samba/smb.conf /etc/samba/smb.conf.org
```

#### Step 3
Edit the smb.conf file:

```
sudo nano /etc/samba/smb.conf
```

Replace realm and workgroup with the following:

```
[global]
   kerberos method = secrets and keytab
   realm = CORP.PROJECT-X-DC.COM
   workgroup = CORP
   security = ads
   template shell = /bin/bash
   winbind enum groups = Yes
   winbind enum users = Yes
   winbind separator = +
   idmap config * : rangesize = 1000000
   idmap config * : range = 1000000-19999999
   idmap config * : backend = autorid
```

#### Step 4
Confirm passwd and group have winbind set as a value:

```
sudo nano /etc/nsswitch.conf
```

#### Step 5
On Ubuntu, every user that has an interactive logon to the system needs a home directory. For domain users, we need to set this before a user can successfully logon.

Issue the following command:

```
sudo pam-auth-update
```

#### Step 6
Change DNS settings to refer to AD:

```
sudo nano /etc/resolv.conf
```

Add `corp.project-x-dc.com` to /etc/hosts:

```
sudo nano /etc/hosts
```

#### Step 7
Join the domain with Administrator:

```
sudo net ads join -U Administrator
```

#### Step 8
Restart winbind:

```
systemctl restart winbind
```

#### Step 9
Get Active Directory services information listing:

```
net ads info
```

#### Step 10
List all available users:

```
wbinfo -u
```

#### Step 11
Create email-svr’s AD account in your Domain Controller.

Go to Server Manager, then on the top right “Tools” → “Active Directory Users and Computers”.

Navigate to the “Users” folder. Right-click, then go to “New” → “User”.

Add the following information:

```
emailsvr@corp.project-x-dc.com
```

Set email-svr’s password (@password123!).

#### Step 12
Clear the winbind cache by restarting the service:

```
sudo systemctl restart winbind
```

#### Step 13
Login as email-svr (CORP+email-svr):

```
sudo login
```

Issue an id command to view status:

```
id
```

Success!

Go back to Server Manager, and you should see “LINUX-CLIENT” under the “Computers” folder.

Take Snapshot!
