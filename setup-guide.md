# Setup guide: Active Directory lab on AWS (Free Tier)

This is the exact path I took to build the lab documented in the main [README](README.md). I've written it so someone who has never launched an EC2 instance or touched Active Directory can follow it end to end and come out with a working domain, a joined client, and every screenshot needed for the three help-desk scenarios.

I'll explain the *why* behind each decision as I go, because running commands without understanding them is how labs get rebuilt from scratch every time something breaks. By the end of this guide you'll have the same environment I used, and if something fails you'll know enough to fix it.

> **AMI substitution note.** AWS Free Tier doesn't include a Windows 10 or Windows 11 desktop image. I used Windows Server 2022 as both the domain controller and the "client workstation." Every Active Directory operation looks identical either way (domain join, GPO application, password reset, logon), so the learning and the screenshots translate one-to-one. I mention this again in the README so your portfolio readers understand the choice.

---

## Table of Contents

1. [What you'll build](#1-what-youll-build)
2. [Prerequisites](#2-prerequisites)
3. [Cost and Free Tier rules](#3-cost-and-free-tier-rules)
4. [Part 1. AWS networking](#part-1-aws-networking)
5. [Part 2. Launch the Domain Controller](#part-2-launch-the-domain-controller)
6. [Part 3. Install AD DS and promote the DC](#part-3-install-ad-ds-and-promote-the-dc)
7. [Part 4. Launch and join the client](#part-4-launch-and-join-the-client)
8. [Part 5. Build the OU, user, and group structure](#part-5-build-the-ou-user-and-group-structure)
9. [Scenario A. New hire onboarding](#scenario-a-new-hire-onboarding)
10. [Scenario B. Password reset and account unlock](#scenario-b-password-reset-and-account-unlock)
11. [Scenario C. Group Policy troubleshooting](#scenario-c-group-policy-troubleshooting)
12. [Cleanup and stopping instances](#12-cleanup-and-stopping-instances)
13. [Screenshot checklist and naming scheme](#13-screenshot-checklist-and-naming-scheme)
14. [Troubleshooting appendix](#14-troubleshooting-appendix)

---

## 1. What you'll build

**Topology**

```
         Internet
            │
            ▼
     ┌──────────────┐
     │   AWS VPC    │  10.0.0.0/16
     │              │
     │ ┌──────────┐ │
     │ │  Subnet  │ │  10.0.0.0/20  (public)
     │ │          │ │
     │ │  DC01    │ │  10.0.1.10    Windows Server 2022, AD DS, DNS
     │ │  CLIENT01│ │  10.0.1.20    Windows Server 2022 (acts as client)
     │ └──────────┘ │
     └──────────────┘
```

**Domain:** `corp.local`
**Forest functional level:** Windows Server 2016 (a safe default)
**DC hostname:** `DC01`
**Client hostname:** `CLIENT01`

A Virtual Private Cloud (VPC) is your private network inside AWS. Inside it, I created one public subnet and launched two Windows Server 2022 machines: one promoted to a Domain Controller running the AD DS and DNS roles, and one joined to the domain as a member server standing in for a user's workstation.

---

## 2. Prerequisites

Before starting you need:

- An AWS account. New accounts get 12 months of Free Tier on many services.
- An RDP client. Windows has Remote Desktop built in (`mstsc.exe`). On a Mac, install Microsoft Remote Desktop from the App Store.
- A text editor. Notepad++, VS Code, or plain Notepad all work.
- Your current public IP address. Grab it from <https://ifconfig.me> or <https://whatismyip.com>. You'll use it to lock down RDP access so only your home network can connect to the lab.

---

## 3. Cost and Free Tier rules

AWS Free Tier (first 12 months for new accounts) includes:

| Resource                              | Free allowance      |
|---------------------------------------|---------------------|
| EC2 Windows (free-tier-eligible type) | 750 hours / month   |
| EBS General Purpose SSD               | 30 GB total         |
| Data transfer out                     | 100 GB / month      |

> **Instance type I used:** `t3.small` (2 vCPU, 2 GB RAM). I confirmed it as Free-Tier eligible on my account. Free Tier instance-type eligibility varies by account age and by AWS's current Free Tier / Free Plan rules, so check the "Free tier eligible" label in the EC2 launch wizard before you pick one.

The important realities to understand up front:

- 750 hours is enough to run one instance 24/7 for a month. This lab has two instances, so if you leave both running continuously you'll exceed the allowance. The fix I use is to stop both instances between lab sessions. A stopped instance costs nothing for compute. You only pay for storage.
- Windows Server 2022 defaults to a 30 GB root volume. Two instances times 30 GB is 60 GB of EBS storage, which puts you roughly 30 GB over the Free Tier allowance. That works out to about $2 to $3 per month while the instances exist. For a portfolio lab that's fine.
- If you ever fall outside Free Tier coverage, `t3.small` Windows on-demand runs about $0.024 per hour while started and $0 while stopped. Four hours across a weekend is under a dollar a month.

**Golden rule:** stop instances when you're not using them. [Section 12](#12-cleanup-and-stopping-instances) covers the exact process.

---

## Part 1. AWS networking

A purpose-built VPC keeps the lab isolated from anything else in your AWS account and makes cleanup trivial at the end. I build the VPC first, then the security group that both instances will share.

### 1.1 Create the VPC

1. Sign in to the AWS Console and pick a region close to you. I used `us-east-1` (N. Virginia). Stay in the same region for every step in this lab, because VPCs, security groups, and EC2 instances don't cross regions.
2. In the top search bar, type `VPC` and open the VPC dashboard.
3. Click **Create VPC**.
4. Select **VPC and more**. This option creates the VPC, a subnet, an internet gateway, and the route tables in a single action instead of making you build each piece separately.
5. Fill in the form:
   - **Name tag auto-generation:** `ad-lab`
   - **IPv4 CIDR block:** `10.0.0.0/16`
   - **Number of Availability Zones:** `1`
   - **Number of public subnets:** `1`
   - **Number of private subnets:** `0`
   - **NAT gateways:** `None`
   - **VPC endpoints:** `None`
   - **DNS hostnames:** leave enabled (default)
   - **DNS resolution:** leave enabled (default)
6. Click **Create VPC**.

AWS will provision everything and show you a resource map of the new VPC, subnet, internet gateway, and route tables.

📸 **Screenshot:** `01-vpc-created.png`. The "VPC created successfully" confirmation with the resource map.

### 1.2 Create the security group

The security group controls what traffic can reach your instances. Both EC2 machines will share a single group with two inbound rules:

1. RDP on TCP port 3389, allowed only from your public IP.
2. All traffic from the security group itself, so the DC and client can talk to each other on the ports Active Directory needs (DNS, LDAP, Kerberos, and so on) without you having to list each one.

A security group can reference itself, but only after it exists. That means creating it in two passes: save it with the RDP rule, then edit it to add the self-reference.

**Pass 1. Create the group with the RDP rule.**

1. In the VPC dashboard, click **Security Groups**, then **Create security group**.
2. **Name:** `ad-lab-sg`
3. **Description:** `AD lab RDP from my IP + internal AD traffic`
   - AWS rejects em dashes and some other Unicode punctuation in the Description field. Plain ASCII only.
4. **VPC:** pick `ad-lab-vpc`.
5. **Inbound rules**, click **Add rule**:

   | Type | Protocol | Port | Source                                             |
   |------|----------|------|----------------------------------------------------|
   | RDP  | TCP      | 3389 | **My IP** (the console fills in your current IP/32) |

6. Leave the outbound rules at the default (all traffic allowed).
7. Click **Create security group**.

**Pass 2. Add the self-reference.**

1. In the Security Groups list, click `ad-lab-sg` to open its detail page.
2. On the **Inbound rules** tab, click **Edit inbound rules**, then **Add rule**:

   | Type        | Protocol | Port | Source                                                 |
   |-------------|----------|------|--------------------------------------------------------|
   | All traffic | All      | All  | **Custom**, start typing `ad-lab-sg`, pick its own ID  |

3. Click **Save rules**.

📸 **Screenshot:** `02-security-group-rules.png`. The security group detail page with both inbound rules showing.

---

## Part 2. Launch the Domain Controller

### 2.1 Launch the EC2 instance

1. Go to **EC2**, then **Instances**, then **Launch instances**.
2. **Name:** `DC01`
3. **Application and OS Images (AMI):**
   - Type `Windows Server 2022 Base` in the search box.
   - Pick **Microsoft Windows Server 2022 Base** with the **Free tier eligible** label.
4. **Instance type:** `t3.small` (2 vCPU, 2 GB RAM).
   - `t3.small` is the sweet spot for a Windows Server and AD DS lab. `t3.micro` technically runs on 1 GB of RAM but is painfully slow. `t3.medium` is overkill. 2 GB is enough for AD DS, DNS, Server Manager, and a couple of Microsoft Management Console (MMC) snap-ins open at once.
5. **Key pair (login):** click **Create new key pair**.
   - **Name:** `ad-lab-key`
   - **Type:** RSA
   - **Format:** `.pem`
   - Click **Create key pair**. The `.pem` file downloads automatically. Keep it safe. You need it to decrypt the Windows Administrator password, and you can't regenerate it later.
6. **Network settings**, click **Edit**:
   - **VPC:** `ad-lab-vpc`
   - **Subnet:** the public subnet, usually named something like `ad-lab-subnet-public1-us-east-1a`
   - **Auto-assign public IP:** **Enable**
   - **Firewall:** select **Select existing security group**, then pick `ad-lab-sg`
7. **Configure storage:** leave the default (30 GiB gp3).
8. Still inside Network settings (not the Advanced details panel further down the page), scroll to the bottom of the Network settings section and expand **Advanced network configuration**. Under **Network interface 1**, set **Primary IP** to `10.0.1.10`. The VPC and more wizard creates a `/20` public subnet spanning `10.0.0.0/20`, so `10.0.1.10` falls comfortably inside it. If your subnet's CIDR looks different, check the VPC resource map and adjust the Primary IP accordingly. Leave the other sub-fields at their defaults.
9. Click **Launch instance**.

📸 **Screenshot:** `03-dc-ec2-launched.png`. The success confirmation page with the new instance ID.

### 2.2 Retrieve the Administrator password

1. Wait about 4 minutes for the instance to finish booting and for Windows to initialize.
2. Select the `DC01` instance, click **Connect**, go to the **RDP client** tab, and click **Get password**.
3. Upload your `ad-lab-key.pem` file, then click **Decrypt password**.
4. Copy the Administrator password somewhere safe. You'll use it many times.
5. Click **Download remote desktop file**. That gives you a pre-configured `.rdp` file that already points at the right public DNS name.

📸 **Screenshot:** `04-dc-password-retrieved.png`. The RDP connect page showing the decrypted Administrator password. Blur or redact the password value before committing the screenshot to your portfolio.

### 2.3 RDP into DC01

1. Double-click the downloaded `.rdp` file.
2. At the credentials prompt:
   - **Username:** `Administrator`
   - **Password:** the one you just decrypted
3. Accept the certificate warning when it appears.

You should land on a Windows Server 2022 desktop with **Server Manager** opening automatically. If it doesn't, open it yourself from the Start menu. Type `Server Manager` and press Enter.

📸 **Screenshot:** `05-dc-rdp-connected.png`. The Server Manager dashboard on a freshly-connected DC01.

### 2.4 Rename the computer

A Domain Controller should have a meaningful hostname *before* you promote it to a DC. Promoting the server locks in the name.

1. Right-click **Start**, then click **System**.
2. Scroll down to **Rename this PC** and change the name to `DC01`, then click **Next**.
3. Choose **Restart later**. I have one more change to make before rebooting.

📸 **Screenshot:** `06-dc-renamed.png`. The System page showing the pending name `DC01`.

### 2.5 Point the DC's DNS at itself (leave the IP on DHCP)

> **Important AWS caveat.** Do not change the IP assignment from Automatic (DHCP) to Manual on an AWS Windows EC2 instance. Committing a manual IP often flips the Windows Firewall profile to Public, which blocks RDP. You'll lose the session and be unable to reconnect, forcing a relaunch. I learned this the hard way on my first attempt and had to relaunch DC01. AWS already reserves the ENI's primary private IP to your instance for the lifetime of that instance, so DHCP behaves exactly like a static reservation. The AD DS promotion wizard will warn about "dynamic IP," and on AWS that warning is safe to click past. Change only the DNS settings.

1. Right-click **Start**, then **Network Connections**, then click **Ethernet**.
2. Leave **IP assignment** at **Automatic (DHCP)**. Do not touch it.
3. Under **DNS server assignment**, click **Edit**:
   - **Manual**
   - **IPv4:** On
   - **Preferred DNS:** `127.0.0.1` (this machine itself, which will become a DNS server after DC promotion)
   - **Alternate DNS:** `10.0.0.2` (the AWS VPC resolver, which lives at VPC-base-plus-two)
4. Click **Save**.

📸 **Screenshot:** `07-dc-dns-configured.png`. The Ethernet settings page with IP assignment still Automatic (DHCP) and DNS assignment set to Manual with 127.0.0.1 and 10.0.0.2.

Now restart to apply the rename you made in Step 2.4: **Start**, **Power**, **Restart**. Your RDP session drops. Reconnect after about 60 seconds.

---

## Part 3. Install AD DS and promote the DC

Installing AD DS happens in two stages. First you install the role (the binaries land on disk and the services register). Then you run the promotion wizard, which creates the domain, the forest, and the DNS zone, and turns this regular server into a Domain Controller.

### 3.1 Install the AD DS role

1. Open **Server Manager**, then **Manage**, then **Add Roles and Features**.
2. Click **Next** through "Before You Begin" and "Installation Type" (leave it on Role-based or feature-based installation).
3. **Server Selection:** `DC01` is highlighted by default. Click **Next**.
4. **Server Roles:** tick **Active Directory Domain Services**.
   - A pop-up asks to add required features. Click **Add Features**.
5. **Features:** accept the defaults and click **Next**.
6. **AD DS:** read the description, then click **Next**.
7. **Confirmation:** tick **Restart the destination server automatically if required**, then click **Install**.
8. The install takes about 2 minutes.

📸 **Screenshot:** `08-ad-ds-role-installing.png`. The installation progress bar showing AD DS.

### 3.2 Promote the server to a Domain Controller

1. In Server Manager, a yellow warning flag appears in the top-right. Click it and choose **Promote this server to a domain controller**.
2. **Deployment Configuration:**
   - Select **Add a new forest**.
   - **Root domain name:** `corp.local`
3. **Domain Controller Options:**
   - Forest functional level: **Windows Server 2016**
   - Domain functional level: **Windows Server 2016**
   - Tick **Domain Name System (DNS) server**
   - Tick **Global Catalog (GC)**
   - **DSRM password:** pick a strong password and store it. You only need it for boot-into-recovery scenarios, but losing it is painful to recover from.
4. **DNS Options:** ignore the "delegation for this DNS server cannot be created" warning. That's expected for a lab with a non-routable domain name. Click **Next**.
5. **Additional Options:** the NetBIOS name defaults to `CORP`. Leave it and click **Next**.
6. **Paths:** leave the defaults (`C:\Windows\NTDS` and `C:\Windows\SYSVOL`). Click **Next**.
7. **Review Options:** click **Next**.
8. **Prerequisites Check:** expect a few yellow warnings about DNS and cryptography. Those are normal for a lab. Click **Install**.
9. The server reboots automatically once the install finishes. Your RDP session drops. Reconnect after about 3 minutes.
10. After the reboot, log in as `CORP\Administrator` using the same password as before. The promotion does not change the Administrator password, but the account is now a domain account rather than a local one.

📸 **Screenshot:** `09-dcpromo-forest-config.png`. The "Add a new forest" screen showing `corp.local`.
📸 **Screenshot:** `10-dcpromo-prereq-check.png`. The prerequisites check page. Warnings are fine.
📸 **Screenshot:** `11-dc-promotion-complete.png`. Server Manager after the reboot, showing AD DS and DNS with green check marks.

### 3.3 Sanity check

Open PowerShell as Administrator (Start, type `PowerShell`, right-click, Run as Administrator) and run:

```powershell
Get-ADDomain
Get-ADForest
nslookup corp.local
```

`Get-ADDomain` and `Get-ADForest` should both return details for `corp.local`. `nslookup corp.local` should resolve to `10.0.1.10`. If all three succeed, the domain is up.

📸 **Screenshot:** `12-dc-sanity-check-powershell.png`. The PowerShell output of those three commands.

---

## Part 4. Launch and join the client

### 4.1 Launch CLIENT01

Repeat the Part 2.1 launch process with these changes:

- **Name:** `CLIENT01`
- **Primary IP** (under Network settings, Advanced network configuration, Network interface 1): `10.0.1.20`
- **Key pair:** reuse `ad-lab-key`
- **Security group:** reuse `ad-lab-sg`
- **Instance type:** `t3.small` (same as DC01)
- Same AMI, same subnet

📸 **Screenshot:** `13-client-ec2-launched.png`. The EC2 instances list showing both `DC01` and `CLIENT01` in a running state.

### 4.2 RDP into CLIENT01

Same process as DC01:

1. Select `CLIENT01` in the EC2 console, click **Connect**, go to the **RDP client** tab.
2. Decrypt the password with your `ad-lab-key.pem`. Note that `CLIENT01` has its own local Administrator password, different from DC01's.
3. Download the `.rdp` file and double-click to connect.

### 4.3 Point DNS at the Domain Controller

For the client to find the domain, it has to ask the right DNS server. Right now it's asking the AWS resolver, which has no idea what `corp.local` is. I'll point it at DC01 instead.

1. Right-click **Start**, then **Network Connections**, then **Ethernet**.
2. Leave **IP assignment** at **Automatic (DHCP)**. Same reasoning as Part 2.5: don't flip this to Manual on an AWS instance.
3. Under **DNS server assignment**, click **Edit**, pick **Manual**:
   - **IPv4:** On
   - **Preferred DNS:** `10.0.1.10` (DC01)
   - **Alternate DNS:** leave blank
4. Click **Save**.

> **Why leave Alternate DNS blank?** A domain-joined client should resolve AD names through the DC, and fall back through the DC for non-AD names (the DC's DNS forwards public queries to AWS's resolver automatically). If I set `10.0.0.2` here as a fallback, the client will sometimes try to resolve `corp.local` against AWS's public-facing resolver, which fails in weird ways and causes intermittent domain issues. The trade-off is that if DC01 is stopped, this client loses all DNS. That's expected lab behavior, not a bug.

Test from PowerShell on CLIENT01:

```powershell
nslookup corp.local
Test-NetConnection -ComputerName dc01.corp.local -Port 389
```

Both should succeed. If `nslookup` fails, the DNS pointer isn't saved. If the `Test-NetConnection` fails, something's blocking LDAP on port 389 (usually the security group's self-reference rule is missing or misconfigured).

📸 **Screenshot:** `14-client-dns-configured.png`. The Ethernet settings showing `10.0.1.10` as the preferred DNS.

### 4.4 Rename and join the domain

1. Right-click **Start**, then **System**, then **Rename this PC (advanced)**.
2. On the **Computer Name** tab, click **Change**:
   - **Computer name:** `CLIENT01`
   - **Member of:** select **Domain**, type `corp.local`
3. Click **OK**. Windows prompts for domain credentials that are allowed to join a computer to the domain:
   - **Username:** `CORP\Administrator`
   - **Password:** the DC01 Administrator password
4. A "Welcome to the corp.local domain!" message appears. Click **OK**, then **Restart Now**.

📸 **Screenshot:** `15-client-domain-join-dialog.png`. The Computer Name Changes dialog with `corp.local` entered.
📸 **Screenshot:** `16-client-welcome-to-domain.png`. The welcome confirmation.

After the reboot, RDP back in to CLIENT01. The credentials change now that it's domain-joined.

> **Credentials gotcha after domain join.** The `.rdp` file stores `Username: Administrator` with no prefix. Windows on a domain-joined machine interprets that ambiguously and often routes it to `CORP\Administrator`, which will reject the *local* CLIENT01 password. Tell RDP which account you mean explicitly.
>
> At the RDP credentials prompt, click **More choices**, then **Use a different account**, and enter:
>
> - **Username:** `CORP\Administrator`
> - **Password:** the **DC01 Administrator password** (the domain admin password, which is the same one you decrypted for DC01 back in Part 2.2, not the separate CLIENT01 password)
>
> The CLIENT01 local Administrator account still exists after domain join and can also log in (use `CLIENT01\Administrator` with the CLIENT01-specific password AWS decrypted at launch). The point of this step is to prove that domain authentication works, so sign in as `CORP\Administrator`.

📸 **Screenshot:** `17-client-domain-login.png`. The client logon screen or RDP credentials dialog showing `CORP\Administrator` and the `Sign in to: CORP` indicator.

---

## Part 5. Build the OU, user, and group structure

A help-desk-ready environment needs three things: an Organizational Unit (OU) hierarchy that mirrors the company, security groups that grant access by role, and a Disabled Users OU for offboarded accounts. I'll build all of it.

> **Which machine?** Everything from 5.1 through 5.4 happens on DC01. The ADUC and Group Policy tools that manage AD aren't installed on the client, they live on the DC. Section 5.5 is the only part of Part 5 that uses CLIENT01.

### 5.1 Open Active Directory Users and Computers

On DC01: **Start**, type `dsa.msc`, press Enter. (The other path is Server Manager, **Tools**, **Active Directory Users and Computers**.)

ADUC is the GUI where most Tier 1 AD work happens. The left pane shows your domain's container and OU tree. The right pane shows the objects inside whichever container or OU is selected.

### 5.2 Create the OU structure

**What's an OU?** An Organizational Unit is a container inside Active Directory that holds other AD objects (users, computers, groups, or more OUs). It isn't a Windows folder, you can't browse to it in File Explorer. OUs exist only inside AD and serve two purposes: organizing objects to mirror the company's structure, and being a target for Group Policy.

**What I'm building.** The final tree:

```
corp.local
├── _HQ              (top-level OU for the company)
│   ├── IT
│   ├── HR
│   ├── Sales
│   └── Disabled Users
└── Groups           (holds the security groups from 5.4)
```

> Why the leading underscore on `_HQ`? ADUC sorts objects alphabetically and underscores come before letters, so `_HQ` lands at the top of the list above the default containers (`Computers`, `Users`, and so on). That's a small quality-of-life trick many admins use.

**Step A. Create `_HQ` at the top of the domain.**

1. In ADUC, click the domain name `corp.local` in the left pane to select it.
2. Right-click `corp.local`, hover **New**, click **Organizational Unit**.
3. In the dialog:
   - **Name:** `_HQ`
   - **Protect container from accidental deletion:** leave ticked (it's the default). This setting is a safety net that prevents a miss-click from deleting the whole OU. You can turn it off later if you need to delete the OU.
4. Click **OK**. The new `_HQ` OU appears in the left pane under `corp.local`.

**Step B. Create the four sub-OUs inside `_HQ`.**

Now create `IT`, `HR`, `Sales`, and `Disabled Users`, all nested inside `_HQ`.

For each of the four names:

1. Right-click `_HQ` (not the domain this time, you want the new OU to be a child of `_HQ`), hover **New**, click **Organizational Unit**.
2. **Name:** the OU name (`IT`, then `HR`, then `Sales`, then `Disabled Users`).
3. Leave "Protect from accidental deletion" ticked.
4. Click **OK**.

After all four, expand `_HQ` in the left pane by clicking the arrow next to it. You should see all four children.

**Step C. Create `Groups` at the top of the domain.**

This OU sits *beside* `_HQ`, not inside it, because groups serve the whole company rather than one department.

1. Right-click `corp.local` again (same target as Step A, the domain, not `_HQ`), hover **New**, click **Organizational Unit**.
2. **Name:** `Groups`
3. Click **OK**.

**Verify.** Your left-pane tree should look like this once everything's expanded:

```
corp.local
├── Builtin              (system-provided, ignore)
├── Computers            (system-provided, ignore)
├── Domain Controllers   (system-provided, contains DC01)
├── _HQ
│   ├── Disabled Users
│   ├── HR
│   ├── IT
│   └── Sales
├── ForeignSecurityPrincipals  (system-provided, ignore)
├── Groups
├── Keys                 (system-provided, ignore)
├── Managed Service Accounts   (system-provided, ignore)
└── Users                (system-provided, ignore)
```

The extra built-in containers (Builtin, Computers, Users, and so on) are created automatically when you promote a DC. They aren't OUs, they're a different object type called "containers" and they have a slightly different icon. Leave them alone. All the lab work goes inside `_HQ` and `Groups`.

📸 **Screenshot:** `18-ou-structure.png`. The ADUC left pane fully expanded, showing `_HQ` with its four sub-OUs and `Groups` as a sibling.

### 5.3 Create three users through the GUI

I'll create three test users, one per department, so the help-desk scenarios later have accounts to work with. A user in AD is an object that represents a person who can log in. Users live inside OUs and can be members of groups.

**Step A. Create John Smith in the IT OU.**

1. In the ADUC left pane, click `_HQ`, expand it, then click `IT` to select the IT OU.
2. Right-click `IT`, hover **New**, click **User**. A multi-screen wizard opens.
3. **First screen (user info):**
   - **First name:** `John`
   - **Initials:** leave blank
   - **Last name:** `Smith`
   - **Full name:** auto-populates as `John Smith`
   - **User logon name:** `jsmith` (the `@corp.local` part auto-fills)
   - **User logon name (pre-Windows 2000):** auto-populates as `CORP\jsmith`
   - Click **Next**.
4. **Second screen (password):**
   - **Password:** `Password123`
   - **Confirm password:** `Password123`
   - Checkbox options:
     - Untick **User must change password at next logon**. I want to log in as John right away without being forced to pick a new password.
     - Leave **User cannot change password** unticked.
     - Tick **Password never expires**. Lab convenience only. Never do this in production.
     - Leave **Account is disabled** unticked.
   - Click **Next**.
5. **Third screen (summary):** review the info and click **Finish**.

`John Smith` now appears inside the IT OU in the right pane.

**Step B. Create Mary Johnson in the HR OU.**

Repeat Step A with these changes:

- Start by right-clicking the HR OU (not IT).
- **First name:** `Mary`, **Last name:** `Johnson`, **User logon name:** `mjohnson`
- Password: `Password123` with the same three checkbox choices as before.

**Step C. Create Bob Williams in the Sales OU.**

Repeat Step A one more time, this time on the Sales OU.

- **First name:** `Bob`, **Last name:** `Williams`, **User logon name:** `bwilliams`
- Password: `Password123` with the same checkbox choices.

**Verify.** Click each of the three OUs (IT, HR, Sales) in the left pane. Each should have exactly one user in the right pane.

📸 **Screenshot:** `19-user-created-aduc.png`. The IT OU selected in the left pane with `John Smith` visible in the right pane.

### 5.4 Create security groups

**What's a security group?** A group in AD is an object that holds a list of users (or other groups). Instead of assigning permissions or GPOs to each user individually, you assign them to a group and then add users into that group. Add a new hire to the group, they inherit all of the group's access. Remove someone, they lose it. That's how IT scales access management.

- **Security groups** can be used for permissions (file access, logon rights, and so on) and for email distribution.
- **Distribution groups** can only be used for email. This lab doesn't need any.
- **Group scope** controls where the group can be used. **Global** is the right choice for groups that represent users in a single domain, which is all this lab has.

**Naming convention.** I prefix every group name with `GS-` (Global Security) so they sort together and you can tell at a glance what kind of group each one is. Large environments lean heavily on naming conventions like this.

**Step A. Create `GS-IT-Staff`.**

1. In the ADUC left pane, click the `Groups` OU to select it.
2. Right-click `Groups`, hover **New**, click **Group**.
3. In the dialog:
   - **Group name:** `GS-IT-Staff`
   - **Group name (pre-Windows 2000):** auto-fills as `GS-IT-Staff`
   - **Group scope:** **Global**
   - **Group type:** **Security**
4. Click **OK**. The group appears in the right pane inside `Groups`.

**Step B. Create the other three groups the same way.**

Repeat Step A three more times, changing only the group name:

- `GS-HR-Staff`
- `GS-Sales-Staff`
- `GS-All-Employees`

Keep **Global** and **Security** for all of them.

When you're done, clicking the `Groups` OU shows all four groups in the right pane.

📸 **Screenshot:** `20-security-groups-created.png`. The `Groups` OU selected with all four `GS-` groups visible in the right pane.

**Step C. Add each user to their department group and to `GS-All-Employees`.**

The goal is John in IT-Staff, Mary in HR-Staff, Bob in Sales-Staff, and all three in All-Employees.

For each user:

1. Navigate to the user's OU (for example, click `_HQ`, then `IT`, then select `John Smith` in the right pane).
2. Right-click the user, click **Add to a group**.
3. In the Select Groups dialog, type the group name and click **Check Names**. AD underlines the name if it recognizes it.
4. Click **OK**. A confirmation pops up, click **OK** to dismiss.

Do this for each user-to-group pairing:

| User                | Groups to add                                                   |
|---------------------|-----------------------------------------------------------------|
| John Smith (IT)     | `GS-IT-Staff`, then separately add to `GS-All-Employees`        |
| Mary Johnson (HR)   | `GS-HR-Staff`, then separately add to `GS-All-Employees`        |
| Bob Williams (Sales)| `GS-Sales-Staff`, then separately add to `GS-All-Employees`     |

> **Faster way.** The Select Groups dialog accepts multiple group names separated by semicolons. Type `GS-IT-Staff;GS-All-Employees`, click **Check Names**, then **OK**. That adds the user to both groups in one action.

**Step D. Verify the memberships.**

1. In the left pane, click `Groups`.
2. In the right pane, double-click `GS-All-Employees`, then switch to the **Members** tab.
3. You should see all three users (John Smith, Mary Johnson, Bob Williams).
4. Close the dialog.
5. Repeat the check for each department group. Each should show exactly one member: the matching user.

📸 **Screenshot:** `21-group-membership.png`. The Members tab of `GS-All-Employees` showing all three users.

### 5.5 Verify by logging into CLIENT01

> **Before trying to log in as a regular domain user, grant them RDP access to CLIENT01.** By default, Windows allows only local administrators to connect by Remote Desktop. Domain users get an "account is not authorized to remotely log in" error unless you add them to the local **Remote Desktop Users** group. This is a common help-desk ticket category (new hire can't RDP to their workstation), and the fix is the same in a production environment as it is here.

**Step A. Grant RDP access to `GS-All-Employees` on CLIENT01.**

RDP into CLIENT01 as `CORP\Administrator` (using the DC01 admin password, per the Part 4.4 gotcha). Open **PowerShell as Administrator** and run:

```powershell
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "CORP\GS-All-Employees"
Get-LocalGroupMember -Group "Remote Desktop Users"
```

The `Get-LocalGroupMember` output should now list `CORP\GS-All-Employees`. Every current and future member of that group can RDP into CLIENT01 without any per-user work.

> **What you just did is the AGDLP pattern.** Account in a Global group, Global group in a Domain Local (or local) group, Permission granted to the Domain Local group. Add a new employee to `GS-All-Employees` once, and they inherit RDP access to every machine where you've done this. It's the standard way Windows environments scale permissions.

**Step B. Log in as jsmith.**

1. Sign out of the `CORP\Administrator` RDP session (Start, Administrator icon, Sign out).
2. Reconnect with the same `.rdp` file.
3. At the credentials prompt, click **More choices**, then **Use a different account**:
   - **Username:** `CORP\jsmith`
   - **Password:** `Password123`
4. Accept the certificate warning.

You should land on a fresh desktop with `CORP\jsmith` shown in the Start menu. Domain authentication works end to end.

📸 **Screenshot:** `22-client-login-domain-user.png`. The CLIENT01 desktop after logging in as `jsmith`, with `CORP\jsmith` visible in the Start menu.

---

## Scenario A. New hire onboarding

**The ticket:** *"New hire Sarah Connor is starting Monday in the Sales department. Please create her account, give her access to the Sales shared resources, and make sure she has to change her password on first login."*

> **RDP session map for Scenario A:**
> - A.1 to A.3 (create user, set attributes, add to groups): DC01 (ADUC)
> - A.4 (test first login): CLIENT01
> - A.5 (bulk CSV): DC01 (PowerShell with the AD module)

### A.1 Create Sarah's user through ADUC

1. In your DC01 RDP session, open ADUC (`dsa.msc`) if it isn't already open. Right-click `_HQ > Sales` OU, click **New**, click **User**.
2. Fill in:
   - First name: `Sarah`
   - Last name: `Connor`
   - User logon name: `sconnor`
3. Click **Next**. Password: `Password123`.
4. Tick **User must change password at next logon**. This time we want it on, because the ticket explicitly asks for forced change.
5. Click **Finish**.

### A.2 Populate her profile attributes

An AD user object carries far more than a username and password. Filling out the Organization tab makes the account useful to the rest of the business. HR exports, the email Global Address List, and manager-based GPOs all pull from these fields.

Double-click `sconnor` to open Properties. On the **General** tab:

- **Description:** `Sales Representative, started 2026-04-21`
- **Telephone number:** `555-0100`
- **E-mail:** `sconnor@corp.local`

Switch to the **Organization** tab:

- **Title:** `Sales Representative`
- **Department:** `Sales`
- **Company:** `Corp Inc.`
- **Manager:** click **Change**, select `bwilliams`

📸 **Screenshot:** `23-newhire-user-properties.png`. The Organization tab filled in.

### A.3 Add Sarah to her groups

Still in Sarah's Properties, switch to the **Member Of** tab, click **Add**, then type `GS-Sales-Staff;GS-All-Employees`, click **Check Names**, click **OK**.

📸 **Screenshot:** `24-newhire-group-membership.png`. Sarah's Member Of tab showing both groups.

### A.4 Test her first login

On CLIENT01, log in as `CORP\sconnor` with `Password123`. Windows forces a password change. You can't reuse `Password123` because AD's password history blocks it, so set the new password to `Password123!`.

**Lab password convention.** Every time a password is changed during a scenario, I append one `!` to the previous one. `Password123` becomes `Password123!` becomes `Password123!!`. The count of exclamation marks tells you at a glance how many changes have happened.

📸 **Screenshot:** `25-newhire-force-password-change.png`. The "You must change your password before signing in" prompt on CLIENT01.

### A.5 Bonus: bulk-create five more new hires with PowerShell

Onboarding one user at a time through ADUC works, but it doesn't scale. When HR sends a batch, I'd rather run a single script than open the wizard five times. Here's the PowerShell approach.

On DC01, open **PowerShell as Administrator** and run this block. It creates the folder and writes the CSV in one step. I build the file with a string array rather than a here-string, because here-strings are fragile when pasted (the closing `"@` must be at column 0 with no leading whitespace, and any paste-induced whitespace silently breaks it).

```powershell
New-Item -Path C:\LabScripts -ItemType Directory -Force | Out-Null

$csvLines = @(
    "FirstName,LastName,SamAccountName,Department,Title"
    "Alice,Nguyen,anguyen,Sales,Sales Representative"
    "Ben,Ortiz,bortiz,Sales,Sales Representative"
    "Cara,Patel,cpatel,HR,HR Coordinator"
    "David,Kim,dkim,IT,Desktop Support Technician"
    "Eva,Lopez,elopez,IT,Desktop Support Technician"
)
$csvLines | Set-Content -Path C:\LabScripts\newhires.csv -Encoding UTF8

Get-Content C:\LabScripts\newhires.csv
```

`Get-Content` should echo all six lines back (header plus five data rows). If you see that, you're good.

Then run the import loop:

```powershell
Import-Csv C:\LabScripts\newhires.csv | ForEach-Object {
    $ou = "OU=$($_.Department),OU=_HQ,DC=corp,DC=local"
    $upn = "$($_.SamAccountName)@corp.local"
    $displayName = "$($_.FirstName) $($_.LastName)"

    New-ADUser `
        -Name $displayName `
        -GivenName $_.FirstName `
        -Surname $_.LastName `
        -SamAccountName $_.SamAccountName `
        -UserPrincipalName $upn `
        -Title $_.Title `
        -Department $_.Department `
        -Path $ou `
        -AccountPassword (ConvertTo-SecureString "Password123" -AsPlainText -Force) `
        -ChangePasswordAtLogon $true `
        -Enabled $true

    Add-ADGroupMember -Identity "GS-$($_.Department)-Staff" -Members $_.SamAccountName
    Add-ADGroupMember -Identity "GS-All-Employees" -Members $_.SamAccountName

    Write-Host "Created $displayName in $($_.Department)" -ForegroundColor Green
}
```

Expected output: five green "Created ..." lines and no red errors.

📸 **Screenshot:** `26-bulk-import-csv.png`. The PowerShell `Get-Content` output showing the CSV contents.
📸 **Screenshot:** `27-bulk-import-powershell.png`. The PowerShell window showing the five green "Created ..." lines.
📸 **Screenshot:** `28-bulk-import-aduc-verify.png`. ADUC with the Sales, HR, and IT OUs expanded showing the new accounts in each.

---

## Scenario B. Password reset and account unlock

**The ticket:** *"User jsmith called, he's locked out of his computer and can't log in. He also thinks he needs his password reset."*

> **RDP session map for Scenario B:**
> - B.1 (set lockout policy + run gpupdate): starts on DC01 (GPMC + PowerShell), then `gpupdate` on CLIENT01, then trigger the lockout by failing logins on CLIENT01
> - B.2 (unlock via ADUC): DC01
> - B.3 (reset via ADUC): DC01
> - B.4 (reset/unlock via PowerShell): DC01
> - B.5 (verify on the client): CLIENT01

### B.1 Reproduce the lockout

To show the scenario end to end I first need to configure an account lockout policy so the account locks when bad passwords are typed. Windows Server 2022 ships with Default Domain Policy blank on the lockout fields, so I'll enable a threshold.

> **Caution.** The policy I'm about to set applies to every domain account, including `CORP\Administrator`. If you typo your own admin password more than four times on either DC01 or CLIENT01 you'll lock yourself out for 30 minutes. If it happens, wait (the counter resets automatically) or unlock from another signed-in session.

On DC01, open **Group Policy Management** (`gpmc.msc`), navigate to **Forest: corp.local**, then **Domains**, then **corp.local**, right-click **Default Domain Policy**, click **Edit**.

Navigate to **Computer Configuration**, **Policies**, **Windows Settings**, **Security Settings**, **Account Policies**, **Account Lockout Policy**.

Set:

- **Account lockout threshold:** `5 invalid logon attempts`
- Windows auto-suggests the other two. Accept them: `Account lockout duration = 30 min` and `Reset account lockout counter after = 30 min`.

Close the editor.

📸 **Screenshot:** `29-lockout-policy-configured.png`. The Account Lockout Policy showing all three values set.

Now force a Group Policy refresh on both machines so the client picks up the new lockout threshold. Run this on DC01 first:

```powershell
gpupdate /force
```

Then RDP into CLIENT01 as `CORP\Administrator` and run the same command:

```powershell
gpupdate /force
```

Sign out of CLIENT01. On the logon screen, try to sign in as `jsmith` with the wrong password six times. On the sixth attempt, Windows blocks the logon with the lockout message: "The referenced account is currently locked out and may not be logged on to."

📸 **Screenshot:** `30-locked-out-message.png`. The CLIENT01 logon screen showing the lockout error.

### B.2 Unlock through ADUC

On DC01, open ADUC, navigate to `jsmith`, right-click, **Properties**, **Account** tab. Tick **Unlock account. This account is currently locked out on this Active Directory Domain Controller.**, click **OK**.

> The Account tab is visible by default. You don't need **View > Advanced Features** for unlocking (that option enables the Attribute Editor tab, useful for other tasks but not this one).

📸 **Screenshot:** `31-aduc-unlock-checkbox.png`. The Account tab with the Unlock checkbox highlighted.

### B.3 Reset the password through ADUC

Still on `jsmith` in ADUC, right-click, click **Reset Password**.

- **New password:** `Password123`
- **Confirm password:** `Password123`
- **User must change password at next logon:** this will likely be grayed out on `jsmith`, because he was created with Password Never Expires in Part 5.3 (the two settings are mutually exclusive). That's fine, leave it as-is. If you want to exercise the full force-change flow, open jsmith's Account tab first, untick Password never expires, click OK, then reopen Reset Password. The checkbox is now enabled.
- Tick **Unlock the user's account**. Convenient, does Step B.2 for you in one click.
- Click **OK**.

📸 **Screenshot:** `32-aduc-reset-password.png`. The Reset Password dialog with both checkboxes ticked.

### B.4 The same workflow through PowerShell

Clicking is fine for one user. PowerShell is how you scale. On DC01, open **PowerShell as Administrator**.

> **Prerequisite for `jsmith` specifically.** Because I ticked Password never expires on `jsmith` in Part 5.3, the `ChangePasswordAtLogon` step below errors with a constraint violation unless I clear that flag first. Run this one-liner:
>
> ```powershell
> Set-ADUser -Identity jsmith -PasswordNeverExpires $false
> ```
>
> You only need this line for users who have Password Never Expires set. The bulk-CSV users from A.5 don't, so you'd skip this for them.

Now the main reset, unlock, and verify block. Each line is a self-contained command, safe to paste individually or as a batch:

```powershell
Unlock-ADAccount -Identity jsmith

Set-ADAccountPassword -Identity jsmith -Reset -NewPassword (ConvertTo-SecureString "Password123" -AsPlainText -Force)

Set-ADUser -Identity jsmith -ChangePasswordAtLogon $true

Unlock-ADAccount -Identity jsmith

Get-ADUser jsmith -Properties LockedOut, PasswordLastSet, PasswordExpired | Format-List Name, LockedOut, PasswordLastSet, PasswordExpired
```

What each line does:

1. First `Unlock-ADAccount` unlocks in case the account is currently locked.
2. `Set-ADAccountPassword -Reset` does an administrative password reset (no old password needed).
3. `Set-ADUser -ChangePasswordAtLogon $true` forces the user to change the password on the next logon.
4. Second `Unlock-ADAccount` is a belt-and-suspenders unlock in case step 3 re-locked anything.
5. `Get-ADUser ... Format-List` verifies the result. The output should show `LockedOut: False` and a recent `PasswordLastSet`.

📸 **Screenshot:** `33-powershell-unlock-reset.png`. PowerShell with all commands executed cleanly and the `Get-ADUser` output showing `LockedOut: False`.

### B.5 Verify on the client, and meet an RDP limitation

Back on CLIENT01, try to log in as `jsmith` with `Password123`. You get this error:

> *"You must change your password before logging on the first time. Please update your password or contact your system administrator or technical support."*

And RDP refuses to let you complete the login. This isn't a lab bug, it's a documented Windows limitation that's good to understand for help-desk work.

**Why it happens.** RDP defaults to Network Level Authentication (NLA). NLA validates credentials *before* the session starts. A password change requires an interactive session (the user needs a Windows shell to type the new password into). Those two requirements contradict each other: NLA won't build a session without valid credentials, but the credentials are valid except for the "must change at next logon" flag, which can only be cleared inside a session.

**How it's resolved in production:**

- The user signs in at a physical console (or any interface without NLA) to change the password, then RDP works.
- The admin changes the password on the user's behalf with the old-and-new form of `Set-ADAccountPassword`, which clears the must-change flag, and communicates the new password out of band.
- Some legacy or non-NLA RDP setups allow the change prompt to proceed, but that's discouraged for security reasons.

**For this lab,** capturing the error dialog *is* the deliverable for B.5. It demonstrates that you recognize the NLA-versus-forced-change interaction. Save the dialog as your screenshot.

📸 **Screenshot:** `34-rdp-nla-forced-change-error.png`. The "You must change your password before logging on the first time" error dialog.

To finish the flow and continue the lab, use the admin-supplies-both-passwords workaround. On DC01 in PowerShell:

```powershell
Set-ADAccountPassword -Identity jsmith -OldPassword (ConvertTo-SecureString "Password123" -AsPlainText -Force) -NewPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force)
```

That simulates `jsmith` changing his own password from `Password123` to `Password123!`. It updates `pwdLastSet` and clears the must-change flag. Now RDP into CLIENT01 as `CORP\jsmith` with `Password123!` and you'll land on his desktop.

📸 **Screenshot (optional):** `34b-user-successful-login-after-workaround.png`. CLIENT01 desktop after `jsmith`'s successful logon post-workaround.

---

## Scenario C. Group Policy troubleshooting

**The ticket:** *"IT wants a corporate desktop wallpaper deployed to all Sales users. Marketing uploaded the image. Please deploy it, verify it works, and figure out why it's NOT applying to user cpatel, who also just joined Sales."*

This scenario has a built-in gotcha I planted on purpose: `cpatel` lives in the HR OU, not Sales (she joined Sales late and HR never moved her). Diagnosing why a GPO didn't apply to a specific user is the core help-desk-level Group Policy skill.

> **RDP session map for Scenario C:**
> - C.1 (create GPO, share, link): DC01 (File Explorer + GPMC)
> - C.2 (apply and verify): CLIENT01 (log in as `bwilliams`, see wallpaper, run `gpresult`)
> - C.3 (troubleshoot cpatel): mixed. Logon attempts and `gpresult` on CLIENT01. The `Set-ADUser -ChangePasswordAtLogon $false` prereq and `Get-ADUser` / `Move-ADObject` on DC01.
> - C.4 (Block Inheritance bonus): mixed. GPMC changes on DC01. Login tests and `gpresult /h` report on CLIENT01.

### C.1 Create the wallpaper GPO

On DC01:

1. Open **File Explorer**, create `C:\CorpAssets\`, and drop any `.bmp` or `.jpg` inside it named `wallpaper.jpg`. Any image works. I used a default Windows wallpaper copied from `C:\Windows\Web\Wallpaper`.
2. Share the folder. Right-click `C:\CorpAssets`, **Properties**, **Sharing** tab, **Advanced Sharing**, tick **Share this folder**, click **Permissions**, add **Authenticated Users** with **Read**, click **OK**. The UNC path is now `\\DC01\CorpAssets\wallpaper.jpg`. Authenticated Users covers both user and computer accounts, which matters because GPO retrieval runs under both contexts.
3. Open **Group Policy Management** (`gpmc.msc`).
4. Right-click `_HQ > Sales` OU, click **Create a GPO in this domain, and Link it here**.
   - **Name:** `U_Sales_Wallpaper`. The `U_` prefix is a convention that tells me this is a User-scope policy at a glance.
5. Right-click the new GPO, click **Edit**.
6. Navigate to **User Configuration**, **Policies**, **Administrative Templates**, **Desktop**, **Desktop**, double-click **Desktop Wallpaper**.
7. Set it to **Enabled**. **Wallpaper Name:** `\\DC01\CorpAssets\wallpaper.jpg`. **Wallpaper Style:** `Fill`. Click **OK**.
8. Close the editor.

📸 **Screenshot:** `35-gpo-created-linked.png`. GPMC showing `U_Sales_Wallpaper` linked under the Sales OU.
📸 **Screenshot:** `36-gpo-wallpaper-setting.png`. The Desktop Wallpaper policy editor showing Enabled and the UNC path.

### C.2 Apply and verify on the client

Log in to CLIENT01 as `CORP\bwilliams` with `Password123`. Bob Williams is one of the baseline Sales users. He doesn't have a force-change-at-logon flag, so RDP lets him in without any workaround.

The wallpaper should be on his desktop the moment he logs in. **No `gpupdate /force` needed.** User-scope GPOs apply automatically during logon, Windows reads and applies them while building the session. `gpupdate` is for refreshing policies *mid-session* (for example in Part C.3, where I make a change and want to see the effect without logging off).

Verify with `gpresult`. This is the proof that the wallpaper came from a GPO and not a local Windows setting:

```powershell
gpresult /r /scope:user
```

Under **Applied Group Policy Objects**, `U_Sales_Wallpaper` should be listed.

📸 **Screenshot:** `37-gpresult-applied.png`. The `gpresult /r /scope:user` output showing `U_Sales_Wallpaper` in Applied Group Policy Objects.
📸 **Screenshot:** `38-wallpaper-applied-desktop.png`. The CLIENT01 desktop with the corporate wallpaper.

### C.3 Why `cpatel` didn't get it

> **Prereq: clear `cpatel`'s force-change flag.** Every user created by the A.5 bulk CSV import has `ChangePasswordAtLogon = true`, which trips the same RDP/NLA wall you saw in Scenario B. For this troubleshoot I don't need to demonstrate a password change, I just need `cpatel` to be able to log in. On DC01, run:
>
> ```powershell
> Set-ADUser -Identity cpatel -ChangePasswordAtLogon $false
> ```
>
> Her password stays `Password123`. RDP will now let her in. Use the same one-liner on any other bulk-CSV user you want to sign in as during the lab.

**On CLIENT01,** log out of the current session (`bwilliams`) and log back in as `CORP\cpatel` with `Password123`. The wallpaper does not apply.

**Step 1. Confirm the symptom (on CLIENT01).** Still in cpatel's session on CLIENT01, open PowerShell and run:

```powershell
gpresult /r /scope:user
```

`U_Sales_Wallpaper` is not in Applied. It either appears under "The following GPOs were not applied because they were filtered out" or doesn't appear at all.

📸 **Screenshot:** `39-gpresult-not-applied.png`. The `gpresult` output for cpatel showing `U_Sales_Wallpaper` missing from Applied.

**Step 2. Find the root cause.** User-scope GPOs apply based on the OU the user *object* lives in, not the OU of the computer they're signed in on. So the question becomes: where does `cpatel` live in AD?

> **Run this on DC01, not CLIENT01.** `Get-ADUser` comes from the Active Directory PowerShell module, which is installed on Domain Controllers by default but not on domain-joined clients. On CLIENT01 you'd get "The term 'Get-ADUser' is not recognized." That also matches good help-desk hygiene: check AD objects from your admin station, not from the user's workstation.

Switch to your DC01 RDP session, open PowerShell, run:

```powershell
Get-ADUser cpatel -Properties DistinguishedName | Select-Object DistinguishedName
```

The output is `CN=Cara Patel,OU=HR,OU=_HQ,DC=corp,DC=local`. She lives in the HR OU, not Sales. That's why the Sales-linked GPO has no reason to target her.

📸 **Screenshot:** `40-get-aduser-dn.png`. The PowerShell output on DC01 showing cpatel's DN in the HR OU.

**Step 3. Fix it (on DC01).** In ADUC on DC01, navigate to `_HQ > HR`, right-click `cpatel`, click **Move**, select `_HQ > Sales`, click **OK**. Or run this PowerShell one-liner:

```powershell
Get-ADUser cpatel | Move-ADObject -TargetPath "OU=Sales,OU=_HQ,DC=corp,DC=local"
```

📸 **Screenshot:** `41-user-moved-to-sales.png`. ADUC showing `cpatel` now under the Sales OU.

**Step 4. Retest (on CLIENT01).** On CLIENT01, log out of cpatel's session and log back in as `CORP\cpatel` with `Password123`. The wallpaper appears immediately. User-scope GPOs re-resolve at logon based on the user's *current* OU, and cpatel's current OU is now Sales. No `gpupdate` needed. If you want to confirm, run `gpresult /r /scope:user` and you'll see `U_Sales_Wallpaper` under Applied Group Policy Objects instead of missing.

📸 **Screenshot:** `42-cpatel-wallpaper-applied.png`. cpatel's desktop now showing the corporate wallpaper.

### C.4 Bonus: the Block Inheritance trap

A second way the same kind of ticket gets opened: *"We had the wallpaper working last week, and now some users aren't getting it."* The usual cause when a previously-working GPO stops applying is Block Inheritance on the OU, especially when someone moved the GPO link from a child OU up to the parent.

I reproduced it in the lab to document the symptom and the fix. This section uses both RDP sessions. Keep them open side by side.

**Step 1. On DC01 (GPMC): turn on Block Inheritance on Sales.**

1. In your DC01 RDP session, open GPMC if it isn't already open.
2. Expand **Forest: corp.local**, **Domains**, **corp.local**, **_HQ**.
3. Right-click `Sales`, click **Block Inheritance**.
4. A small blue `!` icon appears on the Sales OU. That's the visual indicator that Block Inheritance is on.

📸 **Screenshot (on DC01):** `43-block-inheritance-indicator.png`. GPMC showing the blue `!` icon next to the Sales OU.

**Step 2. On CLIENT01: observe that this alone doesn't break the wallpaper.**

1. In your CLIENT01 RDP session, log out of whatever account is signed in.
2. Sign in as `CORP\bwilliams` with `Password123`.
3. The wallpaper still appears. Why? Because the GPO is linked directly to the Sales OU. Block Inheritance only blocks GPOs coming from parent OUs. A GPO linked right on the OU where Block Inheritance is enabled still runs.

**Step 3. On DC01 (GPMC): relink the GPO to `_HQ` instead of Sales (the mistake).**

This is the step a junior admin might take, thinking "I'll put it at the parent OU so every department gets it later":

1. Still in GPMC on DC01, expand **_HQ > Sales**. You'll see the `U_Sales_Wallpaper` link under Sales.
2. Right-click `U_Sales_Wallpaper` under Sales, click **Delete**. When it asks, pick **Remove the link**. This removes the link only. The GPO itself still exists under **Group Policy Objects** in the left pane.
3. Right-click `_HQ` (one level up from Sales), click **Link an Existing GPO**.
4. Select `U_Sales_Wallpaper`, click **OK**. The GPO is now linked at `_HQ` instead of at Sales.

**Step 4. On CLIENT01: observe the wallpaper is now broken by Block Inheritance.**

1. In your CLIENT01 RDP session, log out of `bwilliams` and log back in as `CORP\bwilliams` with `Password123`.
2. No wallpaper. Block Inheritance on Sales is stopping the `_HQ`-linked GPO from flowing down to Sales users. That's the bug that would show up in the ticket.

Generate a full HTML Group Policy report on CLIENT01 to confirm the denied state. In the `bwilliams` PowerShell session on CLIENT01, run:

```powershell
gpresult /h "$env:USERPROFILE\Desktop\rsop-report.html" /f
Start-Process "$env:USERPROFILE\Desktop\rsop-report.html"
```

The report opens in your default browser. Scroll down to **User Details** and look at the **Applied GPOs** and **Denied GPOs** sections. `U_Sales_Wallpaper` shows up under Denied with a reason. Capture that view.

📸 **Screenshot (on CLIENT01):** `44-gpresult-html-denied.png`. The `gpresult /h` HTML report showing `U_Sales_Wallpaper` under Denied GPOs.

> **Why `gpresult /h` instead of `rsop.msc`?** `rsop.msc` is legacy and only reports a narrow slice of policies: Security Settings and Software Restriction Policies. It doesn't show Administrative Templates, which is exactly where wallpaper and most user-facing settings live. `gpresult /h` is the complete replacement and is the current standard.

**Step 5. On DC01 (GPMC): fix the Block Inheritance.**

In GPMC on DC01, right-click `Sales`, click **Block Inheritance** again to untick it. The blue `!` icon disappears.

**Step 6. On CLIENT01: confirm the fix.**

Log out of `bwilliams` and log back in as `CORP\bwilliams` with `Password123`. The wallpaper returns. The `_HQ`-linked GPO now inherits down to Sales users because Block Inheritance is off.

**Step 7. On DC01 (GPMC): clean up to a known-good state.**

You can either leave the GPO linked at `_HQ` (it works fine there) or move the link back to Sales:

- Right-click `U_Sales_Wallpaper` under `_HQ`, click **Delete**, pick **Remove the link**.
- Right-click `Sales`, click **Link an Existing GPO**, select `U_Sales_Wallpaper`, click **OK**.

Confirm Block Inheritance is off on Sales (no blue `!`).

---

## 12. Cleanup and stopping instances

### Between lab sessions: stop, don't terminate

In the EC2 console, select both instances, click **Instance state**, **Stop instance**.

- You keep the EBS volumes, all config, AD state, users, and GPOs.
- You pay only for EBS storage, which is roughly $0.08 per GB per month, or a couple of dollars a month for both volumes.

When you want to keep working, start the instances again. Both the public IP and the auto-generated public DNS hostname change on every stop/start, so the `.rdp` file you downloaded previously is stale. Download a fresh one from **EC2**, select the instance, **Connect**, **RDP client**, **Download remote desktop file**. The private IPs (`10.0.1.10` and `10.0.1.20`) persist, and the Administrator password does not change.

**Start order matters.** Bring DC01 up first and let it fully boot (about 3 minutes) before starting CLIENT01. The client depends on the DC for DNS, and starting them in reverse order can leave the client unable to log in with domain accounts until it retries DNS.

### When you're fully done

1. In EC2, terminate both instances.
2. Wait for them to show **terminated**. Go to **Volumes** and delete any leftover EBS volumes. Terminated instances usually clean these up automatically, but verify.
3. In the VPC Dashboard, go to **Your VPCs**, select `ad-lab-vpc`, click **Actions**, click **Delete VPC**. That cleans up the subnet, internet gateway, route tables, and the security group in a single action.
4. In **Key Pairs**, delete `ad-lab-key`, and delete the local `.pem` file from your machine.

---

## 13. Screenshot checklist and naming scheme

**Naming convention:** `NN-section-description.png`.

- `NN` is a two-digit sequence number so the files sort correctly in a folder.
- `section` is a short keyword for what part of the lab the screenshot belongs to (`dc`, `client`, `gpo`, `newhire`, and so on).
- `description` is a hyphenated short phrase.

Save everything to `screenshots/` inside the project folder.

### Complete ordered list

| #   | Filename                                         | What it shows                                                     |
|-----|--------------------------------------------------|-------------------------------------------------------------------|
| 01  | `01-vpc-created.png`                             | "VPC created successfully" with resource map                      |
| 02  | `02-security-group-rules.png`                    | `ad-lab-sg` with RDP-from-my-IP and self-referencing all-traffic  |
| 03  | `03-dc-ec2-launched.png`                         | DC01 instance launch success page                                 |
| 04  | `04-dc-password-retrieved.png`                   | Decrypted Administrator password (REDACT BEFORE PUBLISHING)       |
| 05  | `05-dc-rdp-connected.png`                        | First RDP into DC01, Server Manager dashboard                     |
| 06  | `06-dc-renamed.png`                              | System page showing pending rename to `DC01`                      |
| 07  | `07-dc-dns-configured.png`                       | DC's Ethernet, IP still DHCP, DNS manual 127.0.0.1 / 10.0.0.2     |
| 08  | `08-ad-ds-role-installing.png`                   | AD DS role installation progress                                  |
| 09  | `09-dcpromo-forest-config.png`                   | "Add a new forest" with root domain name `corp.local`             |
| 10  | `10-dcpromo-prereq-check.png`                    | Prerequisites check page                                          |
| 11  | `11-dc-promotion-complete.png`                   | Server Manager showing AD DS and DNS with green check marks       |
| 12  | `12-dc-sanity-check-powershell.png`              | `Get-ADDomain`, `Get-ADForest`, `nslookup corp.local` output      |
| 13  | `13-client-ec2-launched.png`                     | EC2 list showing both DC01 and CLIENT01 running                   |
| 14  | `14-client-dns-configured.png`                   | Client Ethernet settings with DNS = 10.0.1.10                     |
| 15  | `15-client-domain-join-dialog.png`               | Computer Name Changes dialog joining `corp.local`                 |
| 16  | `16-client-welcome-to-domain.png`                | "Welcome to the corp.local domain" confirmation                   |
| 17  | `17-client-domain-login.png`                     | CLIENT01 logon screen showing `Sign in to: CORP`                  |
| 18  | `18-ou-structure.png`                            | ADUC with the full OU tree expanded                               |
| 19  | `19-user-created-aduc.png`                       | IT OU showing `John Smith`                                        |
| 20  | `20-security-groups-created.png`                 | Groups OU with all four `GS-` groups                              |
| 21  | `21-group-membership.png`                        | `GS-All-Employees` Members tab with 3 users                       |
| 22  | `22-client-login-domain-user.png`                | CLIENT01 desktop after `jsmith` logs in                           |
| 23  | `23-newhire-user-properties.png`                 | Sarah Connor's Organization tab                                   |
| 24  | `24-newhire-group-membership.png`                | Sarah's Member Of tab                                             |
| 25  | `25-newhire-force-password-change.png`           | "Must change password" prompt on CLIENT01                         |
| 26  | `26-bulk-import-csv.png`                         | PowerShell `Get-Content` output of `newhires.csv`                 |
| 27  | `27-bulk-import-powershell.png`                  | PowerShell bulk-create loop output                                |
| 28  | `28-bulk-import-aduc-verify.png`                 | ADUC showing 5 new users distributed across OUs                   |
| 29  | `29-lockout-policy-configured.png`               | Account Lockout Policy with threshold = 5                         |
| 30  | `30-locked-out-message.png`                      | CLIENT01 logon screen with lockout error                          |
| 31  | `31-aduc-unlock-checkbox.png`                    | `jsmith` Account tab with Unlock account ticked                   |
| 32  | `32-aduc-reset-password.png`                     | Reset Password dialog                                             |
| 33  | `33-powershell-unlock-reset.png`                 | PowerShell unlock/reset/verify commands with `Get-ADUser` output  |
| 34  | `34-rdp-nla-forced-change-error.png`             | RDP error: "must change password before logging on"               |
| 34b | `34b-user-successful-login-after-workaround.png` | (Optional) CLIENT01 desktop after admin-supplied password change  |
| 35  | `35-gpo-created-linked.png`                      | GPMC with `U_Sales_Wallpaper` under Sales OU                      |
| 36  | `36-gpo-wallpaper-setting.png`                   | Desktop Wallpaper policy editor, Enabled with UNC path            |
| 37  | `37-gpresult-applied.png`                        | `gpresult /r /scope:user` showing GPO applied                     |
| 38  | `38-wallpaper-applied-desktop.png`               | CLIENT01 desktop with corporate wallpaper                         |
| 39  | `39-gpresult-not-applied.png`                    | `gpresult` for cpatel, GPO NOT applied                            |
| 40  | `40-get-aduser-dn.png`                           | PowerShell `Get-ADUser cpatel` showing DN in HR OU                |
| 41  | `41-user-moved-to-sales.png`                     | ADUC with cpatel now under Sales                                  |
| 42  | `42-cpatel-wallpaper-applied.png`                | cpatel's desktop now with the wallpaper                           |
| 43  | `43-block-inheritance-indicator.png`             | GPMC showing blue `!` on Sales from Block Inheritance             |
| 44  | `44-gpresult-html-denied.png`                    | `gpresult /h` HTML report showing `U_Sales_Wallpaper` as denied   |

**Total: 44 screenshots** plus optional `34b`. Every step that produces a visually distinct artifact is captured. If you're short on time, the minimum-viable portfolio set is: `01, 02, 03, 11, 12, 15, 17, 18, 20, 22, 25, 28, 30, 32, 33, 34, 35, 38, 39, 40, 42` (21 shots).

### Redaction checklist before publishing

- Screenshot 04: blur the Administrator password.
- Any shot showing your public IP (security group rules): blur it or replace with `X.X.X.X/32`.
- Any AWS account ID visible in the console top-right or in an ARN: blur it.
- The `ad-lab-key.pem` content, if it ever ends up on screen.

---

## 14. Troubleshooting appendix

### "Can't RDP to the instance"

- Security group: is TCP 3389 open from *your current* IP? Public IPs change when your network changes. Edit the SG rule with the new address.
- Instance state: is it running? Stopped instances don't respond.
- Windows Firewall: the default AWS AMI allows RDP. After promoting to DC the firewall profile sometimes flips to Domain. Verify in `wf.msc`.

### "Domain join fails: cannot contact domain"

- Did you set the client's DNS to `10.0.1.10` (DC's private IP)? That's the most common cause.
- `ping dc01.corp.local` from the client. It should resolve. If it doesn't, DNS isn't reaching the DC.
- Security group: the self-reference rule must allow all traffic between instances. If you've locked it down, make sure TCP 53, 88, 135, 389, 445, 464, 636, 3268 to 3269, and dynamic RPC 49152 to 65535 are all open.

### "DC promotion fails at the prerequisites check"

- DNS not pointing at `127.0.0.1` (preferred) and `10.0.0.2` (alternate) in Windows? Go back to Part 2.5. Ignore the wizard's "dynamic IP" warning, that's expected and safe on AWS.
- Server still showing the old name? Reboot after rename, then retry.

### "GPO not applying on client"

- Run `gpupdate /force` on the client.
- For *user* policies, the user must log off and back on. For *computer* policies, the machine needs a reboot.
- Run `gpresult /h report.html` and open the HTML for a per-setting explanation of what applied and why.
- Check the *scope* of the GPO (is it linked to the right OU?) and where the user or computer object lives (it's often in a different OU than you think, see Scenario C Step 2).

### "Instance won't start back up"

- First, wait 3 to 5 minutes. Windows takes time to boot.
- Check the System Log in the EC2 console (**Actions**, **Monitor and troubleshoot**, **Get system log**). Look for the `Ec2Config` or `EC2Launch` lines.
- If you changed the machine's hostname or IP incorrectly, you may need to detach the EBS volume, mount it to a rescue instance, and fix the registry.

### "My bill went up"

- Check **Billing**, **Cost Explorer**, filter by service. The usual culprits are EBS volumes from instances that didn't terminate cleanly, Elastic IPs that aren't attached to a running instance (AWS charges for unattached EIPs), and NAT Gateways (this lab shouldn't have any, you picked "None" when creating the VPC).

---

## What's next (ideas for portfolio expansion)

Once the base lab is done and all screenshots are captured, easy extensions that make the portfolio richer:

- Add a second DC to demonstrate replication and FSMO roles.
- Deploy a file server and map a home drive through GPO.
- Add WSUS or Windows Admin Center for a systems-admin flavor.
- Set up a site-to-site VPN from your home to the lab VPC instead of RDP over public IP.
- Write a PowerShell help-desk module wrapping the reset, unlock, and new-hire flows into clean functions.
