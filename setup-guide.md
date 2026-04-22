# Active Directory Lab on AWS (Free Tier)

A step-by-step, help-desk-focused Active Directory lab built entirely on AWS EC2 free-tier resources. This guide walks through provisioning the network, standing up a Domain Controller, joining a client, and then running three realistic help-desk scenarios: **New Hire Onboarding**, **Password Reset & Account Unlock**, and **Group Policy troubleshooting**.

> **Substitution note:** AWS Free Tier does not include Windows 10 / 11 desktop AMIs. This lab uses **Windows Server 2022** as the "client workstation" instead. All Active Directory operations (domain join, GPO application, password reset, login, etc.) look and behave identically, so the learning value is the same. Call out this substitution in your portfolio so readers understand the choice.

---

## Table of Contents

1. [What You'll Build](#1-what-youll-build)
2. [Prerequisites](#2-prerequisites)
3. [Cost & Free Tier Rules](#3-cost--free-tier-rules)
4. [Part 1 — AWS Networking](#part-1--aws-networking)
5. [Part 2 — Launch the Domain Controller](#part-2--launch-the-domain-controller)
6. [Part 3 — Install AD DS & Promote the DC](#part-3--install-ad-ds--promote-the-dc)
7. [Part 4 — Launch & Join the Client](#part-4--launch--join-the-client)
8. [Part 5 — Build the OU / User / Group Structure](#part-5--build-the-ou--user--group-structure)
9. [Scenario A — New Hire Onboarding](#scenario-a--new-hire-onboarding)
10. [Scenario B — Password Reset & Account Unlock](#scenario-b--password-reset--account-unlock)
11. [Scenario C — Group Policy Troubleshooting](#scenario-c--group-policy-troubleshooting)
12. [Cleanup & Stopping Instances](#12-cleanup--stopping-instances)
13. [Screenshot Checklist & Naming Scheme](#13-screenshot-checklist--naming-scheme)
14. [Troubleshooting Appendix](#14-troubleshooting-appendix)

---

## 1. What You'll Build

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
     │ │  DC01    │ │  10.0.1.10  — Windows Server 2022, AD DS, DNS
     │ │  CLIENT01│ │  10.0.1.20  — Windows Server 2022 (acts as client)
     │ └──────────┘ │
     └──────────────┘
```

**Domain:** `corp.local`
**Forest functional level:** Windows Server 2016 (safe default)
**DC hostname:** `DC01`
**Client hostname:** `CLIENT01`

---

## 2. Prerequisites

- An AWS account (new accounts get 12 months of free tier on many services)
- An RDP client — Windows has built-in Remote Desktop (`mstsc.exe`); on Mac use Microsoft Remote Desktop from the App Store
- A text editor (Notepad++, VS Code, or just Notepad)
- Your current public IP — grab it from <https://ifconfig.me> or <https://whatismyip.com>. You'll need it to lock down RDP access.

---

## 3. Cost & Free Tier Rules

AWS Free Tier (first 12 months for new accounts) includes:

| Resource                              | Free allowance                       |
|---------------------------------------|--------------------------------------|
| EC2 Windows (free-tier-eligible type) | 750 hours / month                    |
| EBS General Purpose SSD               | 30 GB total                          |
| Data transfer out                     | 100 GB / month                       |

> **Instance type for this build:** `t3.small` (confirmed free-tier eligible on your account). Free-tier instance-type eligibility varies by account age and AWS's current Free Tier / Free Plan rules — always check the "Free tier eligible" label in the EC2 launch wizard before selecting.

**Important realities:**
- 750 hours = one instance running 24/7. We have **two** instances, so if both run 24/7 you'll go over. Solution: **stop both instances** between lab sessions. A stopped instance costs $0 for compute; you only pay for EBS storage.
- Default Windows Server 2022 root volume is 30 GB. Two instances × 30 GB = 60 GB. You'll be ~30 GB over the EBS free tier → roughly **$2–$3 per month** until you terminate. That's a fair price for a portfolio lab.
- If you ever fall outside free-tier coverage, `t3.small` Windows on-demand is roughly **$0.024/hour running, $0 stopped**. Running both instances 4 hours a weekend = under $1/month. Still trivial.

**Golden rule: stop instances when not using them.** We'll cover how in [Section 12](#12-cleanup--stopping-instances).

---

## Part 1 — AWS Networking

We'll use a purpose-built VPC so the lab is isolated and easy to delete later.

### 1.1 Create a VPC

1. Sign in to the **AWS Console** and pick a region close to you (e.g., `us-east-1` N. Virginia or `us-east-2` Ohio). **Stick with the same region for every step** in this lab.
2. Search for **VPC** in the top search bar and open the VPC dashboard.
3. Click **Create VPC**.
4. Select **VPC and more** (this creates VPC + subnets + internet gateway + route tables in one shot).
5. Fill in:
   - **Name tag auto-generation:** `ad-lab`
   - **IPv4 CIDR:** `10.0.0.0/16`
   - **Number of Availability Zones:** `1`
   - **Number of public subnets:** `1`
   - **Number of private subnets:** `0`
   - **NAT gateways:** `None`
   - **VPC endpoints:** `None`
   - **DNS hostnames:** ✅ enabled
   - **DNS resolution:** ✅ enabled
6. Click **Create VPC**.

📸 **Screenshot:** `01-vpc-created.png` — the "VPC created successfully" confirmation showing the resource map.

### 1.2 Create a Security Group

This security group will be shared by both EC2 instances and allows:
- **Inbound RDP (3389)** from *your public IP only*
- **Inbound everything** from *itself* (so DC and client can talk to each other on AD ports)
- **Outbound all** (default)

Steps:

**Step A — create the SG with only the RDP rule first.** A self-referencing rule needs the SG's own ID, which doesn't exist until after creation, so we add that rule in a second step.

1. In the VPC dashboard, click **Security Groups** → **Create security group**.
2. **Name:** `ad-lab-sg`
3. **Description:** `AD lab RDP from my IP + internal AD traffic`
   - ⚠️ AWS rejects em dashes (`—`) and some other Unicode punctuation in the Description field. Stick to plain ASCII: letters, numbers, spaces, and `._-:/()#,@[]+=&;{}!$*`.
4. **VPC:** select `ad-lab-vpc`
5. **Inbound rules** → **Add rule**:

   | Type | Protocol | Port  | Source                                        |
   |------|----------|-------|-----------------------------------------------|
   | RDP  | TCP      | 3389  | My IP (console auto-fills your current IP/32) |

6. Leave **Outbound** at the default (all traffic allowed).
7. Click **Create security group**.

**Step B — now add the self-reference rule.** The SG now has an ID, so it can reference itself.

1. In the Security Groups list, click `ad-lab-sg`.
2. **Inbound rules** tab → **Edit inbound rules** → **Add rule**:

   | Type        | Protocol | Port | Source                                            |
   |-------------|----------|------|---------------------------------------------------|
   | All traffic | All      | All  | Custom → start typing `ad-lab-sg` and pick its ID |

3. **Save rules**.

📸 **Screenshot:** `02-security-group-rules.png` — the security group detail page with both inbound rules visible.

---

## Part 2 — Launch the Domain Controller

### 2.1 Launch the EC2 Instance

1. Go to **EC2** → **Instances** → **Launch instances**.
2. **Name:** `DC01`
3. **Application and OS Images (AMI):**
   - Type `Windows Server 2022 Base` in the search box
   - Pick **Microsoft Windows Server 2022 Base** that is marked **Free tier eligible**
4. **Instance type:** `t3.small` (2 vCPU / 2 GB RAM — confirmed free-tier eligible on your account)
   - ℹ️ Note: `t3.small` is the sweet spot for a Windows Server + AD DS lab. The cheaper `t3.micro` (1 GB RAM) technically runs but is painfully slow; the larger `t3.medium` is overkill. 2 GB RAM is enough to run AD DS, DNS, Server Manager, and a couple of MMC consoles comfortably.
5. **Key pair (login):** **Create new key pair**
   - Name: `ad-lab-key`
   - Type: RSA, Format: `.pem`
   - Click **Create key pair** — the `.pem` file will download automatically. **Keep it safe** — you need it to decrypt the Windows Administrator password.
6. **Network settings** → **Edit**:
   - **VPC:** `ad-lab-vpc`
   - **Subnet:** the public subnet (e.g., `ad-lab-subnet-public1-us-east-1a`)
   - **Auto-assign public IP:** **Enable**
   - **Firewall:** **Select existing security group** → choose `ad-lab-sg`
7. **Configure storage:** leave at default (30 GiB gp3)
8. Still inside **Network settings** (not the "Advanced details" panel further down the page), scroll to the bottom of that section and expand **▼ Advanced network configuration** → under **Network interface 1** set **Primary IP:** `10.0.1.10`. (The AWS "VPC and more" wizard creates a `/20` public subnet spanning `10.0.0.0/20`, so `10.0.1.10` sits comfortably inside it. Confirm your subnet's CIDR in the VPC resource map if it looks different.) Leave the other sub-fields at their defaults.
9. Click **Launch instance**.

📸 **Screenshot:** `03-dc-ec2-launched.png` — the "Success" confirmation page with the new instance ID.

### 2.2 Retrieve the Administrator Password

1. Wait ~4 minutes for the instance to finish boot and Windows to initialize.
2. Select the `DC01` instance → click **Connect** → **RDP client** tab → **Get password**.
3. Upload your `ad-lab-key.pem` file → **Decrypt password**.
4. Copy the **Administrator** password somewhere safe (you'll need it repeatedly).
5. Click **Download remote desktop file** — this gives you a pre-configured `.rdp` file.

📸 **Screenshot:** `04-dc-password-retrieved.png` — the RDP connect page showing the decrypted Administrator password (blur/redact the actual password value before committing to your portfolio).

### 2.3 RDP Into DC01

1. Double-click the downloaded `.rdp` file.
2. When prompted for credentials, use:
   - **Username:** `Administrator`
   - **Password:** the one you just decrypted
3. Accept the certificate warning.

You should land on a Windows Server 2022 desktop with **Server Manager** opening automatically.

📸 **Screenshot:** `05-dc-rdp-connected.png` — the fresh Server Manager dashboard on DC01.

### 2.4 Rename the Computer

A DC should have a meaningful hostname *before* you promote it.

1. Right-click **Start** → **System**.
2. Scroll to **Rename this PC** → change name to `DC01` → **Next**.
3. Choose **Restart later** (we'll do more first).

📸 **Screenshot:** `06-dc-renamed.png` — System page showing the pending name `DC01`.

### 2.5 Point the DC's DNS at Itself (keep IP on DHCP)

> **AWS-specific warning:** Do **not** change IP assignment from Automatic (DHCP) to Manual on an AWS Windows EC2 instance. Committing a manual IP often flips the Windows Firewall profile to "Public", which blocks RDP — you'll lose your RDP session and be unable to reconnect, forcing a relaunch. AWS already reserves the ENI's primary private IP to this instance for its lifetime, so DHCP behaves like a static reservation for our purposes. The AD DS promotion wizard will show a *warning* about dynamic IP, which we'll safely click through. **Only change the DNS settings.**

Steps:

1. Right-click **Start** → **Network Connections** → **Ethernet**.
2. Leave **IP assignment** at **Automatic (DHCP)** — do not touch it.
3. Under **DNS server assignment**, click **Edit**:
   - **Manual**
   - **IPv4:** On
   - **Preferred DNS:** `127.0.0.1` (itself — will be a DNS server after DC promotion)
   - **Alternate DNS:** `10.0.0.2` (AWS VPC DNS resolver — always at VPC-base + 2)
4. Save.

📸 **Screenshot:** `07-dc-dns-configured.png` — the Ethernet settings page showing IP assignment still Automatic (DHCP) and DNS assignment set to Manual with 127.0.0.1 / 10.0.0.2.

**Now reboot** to apply the rename from step 2.4: **Start** → **Power** → **Restart**. Your RDP will drop; reconnect after ~60 seconds.

---

## Part 3 — Install AD DS & Promote the DC

### 3.1 Install the AD DS Role

1. Open **Server Manager** → **Manage** → **Add Roles and Features**.
2. Click **Next** through "Before You Begin" and "Installation Type" (Role-based).
3. **Server Selection:** DC01 is already highlighted — **Next**.
4. **Server Roles:** ✅ **Active Directory Domain Services**.
   - A pop-up appears asking to add required features — click **Add Features**.
5. **Features:** accept defaults → **Next**.
6. **AD DS:** read the info → **Next**.
7. **Confirmation:** ✅ **Restart the destination server automatically if required** → **Install**.
8. Wait ~2 minutes for the install to finish.

📸 **Screenshot:** `08-ad-ds-role-installing.png` — the installation progress bar showing AD DS.

### 3.2 Promote the Server to a Domain Controller

1. In Server Manager, a **yellow ⚠️ flag** appears top-right — click it → **Promote this server to a domain controller**.
2. **Deployment Configuration:**
   - ✅ **Add a new forest**
   - **Root domain name:** `corp.local`
3. **Domain Controller Options:**
   - Forest functional level: **Windows Server 2016**
   - Domain functional level: **Windows Server 2016**
   - ✅ **Domain Name System (DNS) server**
   - ✅ **Global Catalog (GC)**
   - **DSRM password:** choose a strong password and store it (used only for boot-into-recovery scenarios)
4. **DNS Options:** ignore the "delegation could not be created" warning → **Next**.
5. **Additional Options:** NetBIOS name defaults to `CORP` → **Next**.
6. **Paths:** leave defaults (`C:\Windows\NTDS`, `C:\Windows\SYSVOL`) → **Next**.
7. **Review Options** → **Next**.
8. **Prerequisites Check:** expect a few yellow warnings about DNS / cryptography — these are normal for a lab → **Install**.
9. The server reboots automatically. RDP will drop; reconnect in ~3 minutes.
10. After reboot, log in as `CORP\Administrator` (same password as before).

📸 **Screenshot:** `09-dcpromo-forest-config.png` — the "Add a new forest" screen showing `corp.local`.
📸 **Screenshot:** `10-dcpromo-prereq-check.png` — the prerequisites check page (warnings are fine).
📸 **Screenshot:** `11-dc-promotion-complete.png` — Server Manager after reboot showing the **AD DS** and **DNS** roles both installed with green checkmarks.

### 3.3 Quick Sanity Check

Open **Windows Terminal** (or PowerShell as Admin) and run:

```powershell
Get-ADDomain
Get-ADForest
nslookup corp.local
```

You should see `corp.local` listed as your domain and forest, and `nslookup` should resolve to `10.0.1.10`.

📸 **Screenshot:** `12-dc-sanity-check-powershell.png` — the PowerShell output of those three commands.

---

## Part 4 — Launch & Join the Client

### 4.1 Launch CLIENT01

Repeat Part 2.1 with these differences:
- **Name:** `CLIENT01`
- **Primary IP** (under Network settings → Advanced network configuration → Network interface 1): `10.0.1.20`
- **Key pair:** reuse `ad-lab-key`
- **Security group:** reuse `ad-lab-sg`
- **Instance type:** `t3.small` (same as DC01)
- Same AMI, same subnet

📸 **Screenshot:** `13-client-ec2-launched.png` — EC2 instances list showing both `DC01` and `CLIENT01` in `running` state.

### 4.2 RDP Into CLIENT01

Decrypt the password with `ad-lab-key.pem`, download the RDP file, connect.

### 4.3 Point DNS to the Domain Controller

For the client to find the domain, its DNS must resolve `corp.local` via DC01.

1. Right-click **Start** → **Network Connections** → **Ethernet**.
2. Leave **IP assignment** at **Automatic (DHCP)** — same reasoning as Part 2.5, don't flip this to Manual on an AWS instance.
3. Under **DNS server assignment**, click **Edit** → **Manual**:
   - **IPv4:** On
   - **Preferred DNS:** `10.0.1.10` (DC01)
   - **Alternate DNS:** leave blank
4. Save.

> ℹ️ We deliberately leave Alternate DNS blank. A domain-joined client should resolve AD names via the DC first and fall back through the DC (which forwards to AWS's resolver for non-AD names). Adding `10.0.0.2` here as a fallback would let the client try to resolve `corp.local` against AWS's public-facing resolver, which fails weirdly and can cause intermittent domain issues. Trade-off: if DC01 is stopped, this client loses all DNS — that's expected lab behavior, not a bug.

Test from PowerShell:

```powershell
nslookup corp.local
Test-NetConnection -ComputerName dc01.corp.local -Port 389
```

Both should succeed.

📸 **Screenshot:** `14-client-dns-configured.png` — the Ethernet settings showing `10.0.1.10` as the preferred DNS.

### 4.4 Rename and Join the Domain

1. Right-click **Start** → **System** → **Rename this PC (advanced)**.
2. Under **Computer Name** tab → **Change**:
   - **Computer name:** `CLIENT01`
   - **Member of:** ● **Domain** → `corp.local`
3. Click **OK**. You'll be prompted for domain credentials:
   - **Username:** `CORP\Administrator`
   - **Password:** the DC's administrator password
4. "Welcome to the corp.local domain!" → **OK** → **Restart Now**.

📸 **Screenshot:** `15-client-domain-join-dialog.png` — the "Computer Name Changes" dialog with `corp.local` entered.
📸 **Screenshot:** `16-client-welcome-to-domain.png` — the "Welcome to the corp.local domain" confirmation.

After reboot, RDP back in to CLIENT01 — but the credentials change now that it's domain-joined:

> ⚠️ **Credentials gotcha after domain join.** The `.rdp` file has `Username: Administrator` with no prefix. On a domain-joined client, Windows interprets that ambiguously and often routes it to `CORP\Administrator`, which will reject the *local* CLIENT01 password. You must explicitly tell RDP which account you mean:
>
> - **In the RDP credentials prompt, click "More choices" → "Use a different account"**, then use:
>   - **Username:** `CORP\Administrator`
>   - **Password:** the **DC01 administrator password** (the domain admin password — the same one you decrypted for DC01 way back in Part 2.2, not the separate CLIENT01 password)
>
> The CLIENT01 local Administrator account still exists after domain join and can also log in (use `CLIENT01\Administrator` with the CLIENT01-specific password AWS decrypted at launch), but the point of this step is to prove that **domain auth** works — so log in as `CORP\Administrator`.

📸 **Screenshot:** `17-client-domain-login.png` — the client logon screen / RDP credentials dialog showing `CORP\Administrator` and the `Sign in to: CORP` indicator.

---

## Part 5 — Build the OU / User / Group Structure

A realistic help-desk environment has an **OU hierarchy** that mirrors the org, dedicated **security groups**, and a **Disabled Users** OU for offboarding. We'll build all of it.

> 🖥️ **Which machine?** Everything from 5.1 through 5.4 happens on **DC01** (the Domain Controller RDP session). The ADUC and Group Policy tools that manage AD aren't installed on the client — they live on the DC. Section 5.5 is the only part of Part 5 that uses **CLIENT01**.

### 5.1 Open Active Directory Users and Computers (ADUC)

On DC01: **Start** → type `dsa.msc` → Enter. (Or Server Manager → Tools → Active Directory Users and Computers.)

### 5.2 Create the OU Structure

**What's an OU?** An Organizational Unit is a container inside Active Directory that holds other AD objects (users, computers, groups, or more OUs). It's *not* a Windows folder — you can't browse to it in File Explorer. OUs exist only inside AD and are used for two things: organizing objects the way your company is structured, and being a target for Group Policy.

**What we're building.** The final tree:

```
corp.local
├── _HQ              (top-level OU for the company)
│   ├── IT
│   ├── HR
│   ├── Sales
│   └── Disabled Users
└── Groups           (holds the security groups we create in 5.4)
```

> Why the leading underscore on `_HQ`? ADUC sorts objects alphabetically, and an underscore comes before letters. Putting `_HQ` first keeps the "real" structure above the default built-in containers (`Computers`, `Users`, etc.) — a small quality-of-life trick admins commonly use.

**Step A — Create `_HQ` at the top of the domain.**

1. In ADUC (left pane), click the domain name `corp.local` once to select it.
2. **Right-click `corp.local`** → hover **New** → click **Organizational Unit**.
3. In the dialog that opens:
   - **Name:** `_HQ`
   - **Protect container from accidental deletion:** leave ✅ ticked (default). This is just a safety net; it prevents you from accidentally deleting the whole OU with a miss-click. You can always turn it off later if you need to delete the OU.
4. Click **OK**. The new `_HQ` OU appears in the left pane under `corp.local`.

**Step B — Create the four sub-OUs inside `_HQ`.**

Now we create `IT`, `HR`, `Sales`, and `Disabled Users` — all nested **inside** `_HQ`.

For each of the four names:
1. **Right-click `_HQ`** (not the domain this time — you want the new OU to be a child of `_HQ`) → **New** → **Organizational Unit**.
2. **Name:** the OU name (`IT`, then `HR`, then `Sales`, then `Disabled Users`)
3. Leave "Protect from accidental deletion" ticked.
4. **OK**.

After all four, expand `_HQ` in the left pane (click the `>` arrow next to it) — you should see all four children.

**Step C — Create `Groups` at the top of the domain.**

This one sits *beside* `_HQ`, not inside it, because groups serve the whole company rather than one department.

1. **Right-click `corp.local`** again (same as Step A — the domain, not `_HQ`) → **New** → **Organizational Unit**.
2. **Name:** `Groups`
3. **OK**.

**Verify.** Your left-pane tree should look like this (click the `>` arrows to expand everything):

```
corp.local
├── Builtin              (system-provided — ignore)
├── Computers            (system-provided — ignore)
├── Domain Controllers   (system-provided — contains DC01)
├── _HQ
│   ├── Disabled Users
│   ├── HR
│   ├── IT
│   └── Sales
├── ForeignSecurityPrincipals  (system-provided — ignore)
├── Groups
├── Keys                 (system-provided — ignore)
├── Managed Service Accounts  (system-provided — ignore)
└── Users                (system-provided — ignore)
```

The extra built-in containers (Builtin, Computers, Users, etc.) are created automatically when you promote a DC. They're not OUs — they're a different object type called "containers" and they have a slightly different icon. Leave them alone; we put all our lab work inside `_HQ` and `Groups`.

📸 **Screenshot:** `18-ou-structure.png` — the ADUC left pane fully expanded, showing `_HQ` with its four sub-OUs and `Groups` as a sibling.

### 5.3 Create a Few Users Manually (GUI Method)

We'll create three test users — one per department — so we have real accounts to work with in later scenarios. A user in AD is an object that represents a person who can log in. They live inside OUs and can be members of groups.

**Step A — Create John Smith in the `IT` OU.**

1. In the ADUC left pane, click `_HQ` → expand it → click `IT` to select the IT OU.
2. **Right-click `IT`** → hover **New** → click **User**. A wizard opens with several screens.
3. **First screen (user info):**
   - **First name:** `John`
   - **Initials:** leave blank
   - **Last name:** `Smith`
   - **Full name:** auto-populates as `John Smith`
   - **User logon name:** `jsmith` (the part before `@corp.local` auto-fills)
   - **User logon name (pre-Windows 2000):** auto-populates as `CORP\jsmith`
   - Click **Next**.
4. **Second screen (password):**
   - **Password:** `Password123`
   - **Confirm password:** `Password123`
   - **Checkbox options:**
     - ❌ Untick **User must change password at next logon** (we want to log in as John right away without being forced to pick a new password)
     - ❌ Leave **User cannot change password** unticked
     - ✅ Tick **Password never expires** (lab convenience only — never do this in production)
     - ❌ Leave **Account is disabled** unticked
   - Click **Next**.
5. **Third screen (summary):** review the info, then click **Finish**.

The user `John Smith` now appears inside the `IT` OU in the right pane.

**Step B — Create Mary Johnson in the `HR` OU.**

Repeat Step A, but this time:
- Start by right-clicking the `HR` OU (not IT).
- **First name:** `Mary`, **Last name:** `Johnson`, **User logon name:** `mjohnson`
- Password: `Password123` with the same three checkbox choices as before.

**Step C — Create Bob Williams in the `Sales` OU.**

Repeat Step A again, this time on the `Sales` OU.
- **First name:** `Bob`, **Last name:** `Williams`, **User logon name:** `bwilliams`
- Password: `Password123` with the same checkbox choices.

**Verify.** Click each of the three OUs (`IT`, `HR`, `Sales`) in the left pane — you should see exactly one user in each, on the right pane.

📸 **Screenshot:** `19-user-created-aduc.png` — the `IT` OU selected in the left pane, with `John Smith` visible in the right pane.

### 5.4 Create Security Groups

**What's a security group?** A group in AD is an object that holds a list of users (or other groups). Instead of assigning permissions or GPOs to each user individually, you assign them to a group and then add users into that group. Add a new hire → they inherit all the group's access. Remove someone → they lose it. It's how real IT scales.

- **Security groups** can be used for permissions (file access, login rights, etc.) and for email distribution.
- **Distribution groups** can only be used for email. We don't need those here.
- **Group scope** controls where the group can be used. **Global** is the right choice for groups that represent users in one domain — which is all we have.

**Naming convention.** We prefix all group names with `GS-` (Global Security) so they sort together and you can tell at a glance what kind of group it is. Large environments rely heavily on naming conventions like this.

**Step A — Create `GS-IT-Staff`.**

1. In the ADUC left pane, click the `Groups` OU to select it.
2. **Right-click `Groups`** → hover **New** → click **Group**.
3. In the dialog that opens:
   - **Group name:** `GS-IT-Staff`
   - **Group name (pre-Windows 2000):** auto-fills as `GS-IT-Staff`
   - **Group scope:** ● **Global**
   - **Group type:** ● **Security**
4. Click **OK**. The group appears in the right pane inside `Groups`.

**Step B — Create the other three groups the same way.**

Repeat Step A three more times, changing only the **Group name**:
- `GS-HR-Staff`
- `GS-Sales-Staff`
- `GS-All-Employees`

Keep `Global` / `Security` for all of them.

When you're done, clicking the `Groups` OU should show all four groups in the right pane.

📸 **Screenshot:** `20-security-groups-created.png` — the `Groups` OU selected with all four `GS-` groups visible in the right pane.

**Step C — Add each user to their department's group.**

We need John in IT-Staff, Mary in HR-Staff, Bob in Sales-Staff, and all three in All-Employees.

For each user:
1. Navigate to the user's OU (e.g., click `_HQ` → `IT` → select `John Smith` in the right pane).
2. **Right-click the user** → click **Add to a group…**
3. In the "Select Groups" box that opens, type the group name and click **Check Names**. The name will become underlined if AD recognizes it.
4. Click **OK**. A confirmation appears — click **OK** to dismiss.

Do this for each user-to-group pairing:

| User              | Groups to add them to                                         |
|-------------------|---------------------------------------------------------------|
| John Smith (IT)   | `GS-IT-Staff`, then separately add to `GS-All-Employees`      |
| Mary Johnson (HR) | `GS-HR-Staff`, then separately add to `GS-All-Employees`      |
| Bob Williams (Sales)| `GS-Sales-Staff`, then separately add to `GS-All-Employees` |

> ⚡ **Faster way (add multiple at once):** In the Select Groups dialog, you can type multiple group names separated by semicolons — `GS-IT-Staff;GS-All-Employees` — then Check Names → OK. That adds the user to both groups in one action.

**Step D — Verify the memberships.**

1. In the left pane, click `Groups`.
2. In the right pane, **double-click `GS-All-Employees`** → go to the **Members** tab.
3. You should see all three users (John Smith, Mary Johnson, Bob Williams).
4. Close the dialog.
5. Repeat the check for each department group — it should have exactly one member (the matching user).

📸 **Screenshot:** `21-group-membership.png` — the **Members** tab of `GS-All-Employees` showing all three users.

### 5.5 Verify by Logging Into CLIENT01

> ⚠️ **Before you try to log in as a regular domain user, you have to grant them RDP access to CLIENT01.** By default, Windows lets only local administrators connect via Remote Desktop. Domain users get an "account is not authorized to remotely log in" error unless you add them to the local **Remote Desktop Users** group first. This is a real help-desk ticket category — "new hire can't RDP to their workstation" — and the fix is the same in a real environment as it is here.

**Step A — Grant RDP access to the `GS-All-Employees` group on CLIENT01.**

RDP into CLIENT01 as `CORP\Administrator` (using the DC01 admin password — see the 4.4 gotcha note). Open **PowerShell as Administrator** and run:

```powershell
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "CORP\GS-All-Employees"
Get-LocalGroupMember -Group "Remote Desktop Users"
```

The `Get-LocalGroupMember` output should now list `CORP\GS-All-Employees`. Every current and future member of that group can now RDP into CLIENT01 — no per-user work needed.

> This is the classic **AGDLP** pattern (Account → Global group → Domain Local/local group → Permission). You nest a domain global group inside a local group, and grant permissions on the local group. Add a new employee to `GS-All-Employees` once, and they inherit RDP access to every machine where you've done this.

**Step B — Log in as jsmith.**

1. Log off the `CORP\Administrator` RDP session (Start → Administrator icon → Sign out).
2. Reconnect with the same `.rdp` file.
3. At the credentials prompt, click **More choices** → **Use a different account**:
   - **Username:** `CORP\jsmith`
   - **Password:** `Password123`
4. Accept the certificate warning.

You should land on a fresh desktop with `CORP\jsmith` shown in the Start menu. Domain auth works end-to-end.

📸 **Screenshot:** `22-client-login-domain-user.png` — the CLIENT01 desktop after logging in as `jsmith`, showing the `CORP\jsmith` username in Start.

---

## Scenario A — New Hire Onboarding

**The ticket:** *"New hire Sarah Connor is starting Monday in the Sales department. Please create her account, give her access to the Sales shared resources, and make sure she has to change her password on first login."*

> 🖥️ **RDP session map for Scenario A:**
> - **A.1–A.3 (create user, set attributes, add to groups):** on **DC01** (ADUC)
> - **A.4 (test first login):** on **CLIENT01**
> - **A.5 (bulk CSV):** on **DC01** (PowerShell with the AD module)

### A.1 Create Sarah's User via GUI (the Tier-1 way)

1. In your **DC01 RDP session**, open **ADUC** (`dsa.msc`) if it isn't already open. Right-click `_HQ > Sales` OU → **New** → **User**.
2. Fill in:
   - First name: `Sarah`
   - Last name: `Connor`
   - User logon name: `sconnor`
3. **Next** → Password: `Password123`
4. ✅ **User must change password at next logon** (this time we **do** want it ticked)
5. Finish.

### A.2 Populate Her Profile Attributes

Double-click `sconnor` → **General** tab:
- **Description:** `Sales Representative — started 2026-04-21`
- **Telephone number:** `555-0100`
- **E-mail:** `sconnor@corp.local`

**Organization** tab:
- **Title:** `Sales Representative`
- **Department:** `Sales`
- **Company:** `Corp Inc.`
- **Manager** → Change → select `bwilliams`

📸 **Screenshot:** `23-newhire-user-properties.png` — the Organization tab filled in.

### A.3 Add to Groups

**Member Of** tab → **Add** → type `GS-Sales-Staff;GS-All-Employees` → **Check Names** → **OK**.

📸 **Screenshot:** `24-newhire-group-membership.png` — Sarah's Member Of tab showing both groups.

### A.4 Test Her First Login

On CLIENT01, log in as `CORP\sconnor` / `Password123`. Windows will force a password change — you can't reuse `Password123` (AD's password history blocks it), so set the new password to `Password123!` (our lab convention: any time a password gets changed during a scenario, append an `!` — `Password123` → `Password123!` → `Password123!!` — so the "current" password is always obvious from how many exclamation marks it has).

📸 **Screenshot:** `25-newhire-force-password-change.png` — the "You must change your password before signing in" prompt on CLIENT01.

### A.5 Bonus — Bulk Create 5 More New Hires via PowerShell

Real onboarding rarely happens one user at a time. On DC01, open **PowerShell as Administrator** and run this block to create the folder and the CSV in one step (this avoids Notepad's habit of saving files as `.csv.txt` and is safer to paste than a here-string):

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

> ℹ️ Why not a here-string (`@"..."@`)? Here-strings require the closing `"@` to be at column 0 with nothing before it. If any leading whitespace sneaks in when you paste, PowerShell never closes the string and silently writes nothing. Using a string array (`@( "line1", "line2" )`) is immune to that problem.

`Get-Content` should echo all 6 lines back. If you see the header plus five data rows, you're good.

Then run the import:

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

📸 **Screenshot:** `26-bulk-import-csv.png` — the PowerShell `Get-Content` output showing the CSV contents.
📸 **Screenshot:** `27-bulk-import-powershell.png` — the PowerShell window showing the 5 green "Created..." lines.
📸 **Screenshot:** `28-bulk-import-aduc-verify.png` — ADUC with the `Sales`, `HR`, and `IT` OUs expanded showing all the new accounts.

---

## Scenario B — Password Reset & Account Unlock

**The ticket:** *"User `jsmith` called — he's locked out of his computer and can't log in. He also thinks he needs his password reset."*

> 🖥️ **RDP session map for Scenario B:**
> - **B.1 (set lockout policy + run gpupdate):** starts on **DC01** (GPMC + PowerShell), then `gpupdate` on **CLIENT01**, then trigger the lockout by failing logins on **CLIENT01**
> - **B.2 (unlock via ADUC):** on **DC01**
> - **B.3 (reset via ADUC):** on **DC01**
> - **B.4 (reset/unlock via PowerShell):** on **DC01**
> - **B.5 (verify on client):** on **CLIENT01**

### B.1 Reproduce the Lockout (so the lab is realistic)

First we need to actually lock the account. By default, Windows Server 2022 doesn't have a lockout policy configured, so let's enable one quickly.

> ⚠️ **Caution — this policy applies to every domain account, including `CORP\Administrator`.** After setting it, don't typo your own admin password more than 4 times or you'll lock yourself out for 30 minutes. If that happens, just wait (the counter resets automatically) or unlock from another signed-in session.

On DC01, open **Group Policy Management** (`gpmc.msc`) → **Forest: corp.local** → **Domains** → **corp.local** → right-click **Default Domain Policy** → **Edit**.

Navigate: **Computer Configuration** → **Policies** → **Windows Settings** → **Security Settings** → **Account Policies** → **Account Lockout Policy**.

Set:
- **Account lockout threshold:** `5 invalid logon attempts`
- (Windows will auto-suggest the other two — accept them: `Account lockout duration = 30 min`, `Reset account lockout counter after = 30 min`)

Close the editor.

📸 **Screenshot:** `29-lockout-policy-configured.png` — the Account Lockout Policy showing all three values set.

Force the policy update on **both** machines so the client actually picks up the new lockout threshold:

```powershell
# On DC01
gpupdate /force
```

Then RDP into CLIENT01 as `CORP\Administrator` and run:

```powershell
# On CLIENT01
gpupdate /force
```

Now log off CLIENT01 and try to log in as `jsmith` with the **wrong** password **six** times. On attempt 6 you'll get **"The referenced account is currently locked out and may not be logged on to."**

📸 **Screenshot:** `30-locked-out-message.png` — the CLIENT01 logon screen showing the lockout error.

### B.2 Unlock via ADUC (the clickiest method — Tier 1 classic)

On DC01 → ADUC → navigate to `jsmith` → right-click → **Properties** → **Account** tab → ✅ **Unlock account. This account is currently locked out on this Active Directory Domain Controller.** → **OK**.

> ℹ️ The Account tab is visible by default. You don't need **View → Advanced Features** for unlocking (that option enables the Attribute Editor tab, which is useful for other tasks but not this one).

📸 **Screenshot:** `31-aduc-unlock-checkbox.png` — the Account tab with the **Unlock account** checkbox highlighted.

### B.3 Reset the Password via ADUC

Still on `jsmith` in ADUC → right-click → **Reset Password…**
- **New password:** `Password123`
- **Confirm password:** `Password123`
- **User must change password at next logon** — this will likely be **grayed out** on `jsmith` because he was created with "Password never expires" back in 5.3 (the two settings are mutually exclusive). That's fine; leave it as-is. If you want to exercise the full force-change flow, open `jsmith`'s **Account** tab first, untick **Password never expires**, OK, then re-open Reset Password — the checkbox will now be enabled.
- ✅ **Unlock the user's account** (convenient — does step B.2 for you in one click)
- **OK**

📸 **Screenshot:** `32-aduc-reset-password.png` — the Reset Password dialog with both checkboxes ticked.

### B.4 Reset/Unlock via PowerShell (the faster help-desk method)

Once you're comfortable with the GUI, the *fast* way — and the one you'll mention in interviews — is PowerShell. On DC01, open **PowerShell as Administrator** and run the commands below.

> ⚠️ **Prerequisite for `jsmith` specifically.** Because we ticked **Password never expires** on `jsmith` back in section 5.3, the `ChangePasswordAtLogon` step below will error with a constraint-violation unless we clear that flag first. Run this one-liner up front:
>
> ```powershell
> Set-ADUser -Identity jsmith -PasswordNeverExpires $false
> ```
>
> (You only need this line for users who have Password Never Expires set. The bulk-CSV users in A.5 don't have it, so you'd skip this for them.)

Now the main reset/unlock/verify block — each line is one self-contained command, safe to paste individually or as a batch:

```powershell
Unlock-ADAccount -Identity jsmith

Set-ADAccountPassword -Identity jsmith -Reset -NewPassword (ConvertTo-SecureString "Password123" -AsPlainText -Force)

Set-ADUser -Identity jsmith -ChangePasswordAtLogon $true

Unlock-ADAccount -Identity jsmith

Get-ADUser jsmith -Properties LockedOut, PasswordLastSet, PasswordExpired | Format-List Name, LockedOut, PasswordLastSet, PasswordExpired
```

What each line does:

1. First `Unlock-ADAccount` — unlocks in case the account is currently locked.
2. `Set-ADAccountPassword -Reset` — administrative password reset (no old password needed).
3. `Set-ADUser -ChangePasswordAtLogon $true` — forces the user to change it on next logon.
4. Second `Unlock-ADAccount` — a belt-and-suspenders unlock in case step 3 re-locked anything.
5. `Get-ADUser ... Format-List` — verifies the result. You want `LockedOut: False` and a recent `PasswordLastSet`.

📸 **Screenshot:** `33-powershell-unlock-reset.png` — the PowerShell window showing all commands executed cleanly and the final `Get-ADUser` output showing `LockedOut: False`.

### B.5 Verify on the Client (and meet a real RDP limitation)

Back to CLIENT01 → try to log in as `jsmith` with `Password123`. You'll get this error:

> *"You must change your password before logging on the first time. Please update your password or contact your system administrator or technical support."*

…and **RDP will refuse to let you complete the login.** This isn't a bug in the lab — it's a real, well-documented Windows limitation worth understanding for help-desk work:

**Why this happens.** RDP by default uses **Network Level Authentication (NLA)**. NLA validates the user's credentials *before* the session actually starts. A password change requires an interactive session to prompt for the new password. Those two requirements contradict each other — NLA won't create a session without valid credentials, but the credentials are valid *except* for the "must change at next logon" flag, which can only be resolved inside a session. So you're stuck.

**How this is resolved in the real world:**
- The user physically sits at a console (or a VM console without NLA) and changes the password there, then RDP works.
- The admin changes the password *for* the user (via `Set-ADAccountPassword` with both old and new, which clears the must-change flag) and communicates the new password out-of-band.
- Some legacy / non-NLA RDP setups allow the prompt to proceed, but this is generally discouraged for security reasons.

**For this lab.** Capturing this error IS the B.5 deliverable — it demonstrates that you recognize the NLA-vs-forced-change interaction. Save the error dialog as your screenshot.

📸 **Screenshot:** `34-rdp-nla-forced-change-error.png` — the "You must change your password before logging on the first time" error dialog from the RDP client.

**To actually finish the flow and continue the lab**, use the admin-supplies-both-passwords workaround. On DC01 in PowerShell:

```powershell
Set-ADAccountPassword -Identity jsmith -OldPassword (ConvertTo-SecureString "Password123" -AsPlainText -Force) -NewPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force)
```

This simulates the user changing their own password from `Password123` to `Password123!` — it updates `pwdLastSet` and clears the "must change" flag. Now RDP into CLIENT01 as `CORP\jsmith` / `Password123!` and you're in.

📸 **Screenshot (optional):** `34b-user-successful-login-after-workaround.png` — CLIENT01 desktop after `jsmith` successfully logs in post-workaround.

---

## Scenario C — Group Policy Troubleshooting

**The ticket:** *"IT wants a corporate desktop wallpaper deployed to all Sales users. Marketing created it and uploaded it. Please deploy it, verify it works, and also figure out why it's NOT applying to user `cpatel` who also just joined Sales."*

This scenario intentionally has a built-in gotcha — `cpatel` is actually in the **HR** OU (he joined Sales late and HR never moved him). Troubleshooting *why* a GPO didn't apply is the core help-desk-level GPO skill.

> 🖥️ **RDP session map for Scenario C:**
> - **C.1 (create GPO + share + link):** on **DC01** (File Explorer + GPMC)
> - **C.2 (apply + verify):** on **CLIENT01** (log in as `bwilliams`, see wallpaper, run `gpresult`)
> - **C.3 (troubleshoot cpatel):** mixed — logon attempts and `gpresult` on **CLIENT01**, the `Set-ADUser -ChangePasswordAtLogon $false` prereq and `Get-ADUser` / `Move-ADObject` on **DC01**
> - **C.4 (Block Inheritance bonus):** mixed — GPMC changes on **DC01**, login tests and `rsop.msc` on **CLIENT01**

### C.1 Create a Simple GPO — Desktop Wallpaper

In your **DC01 RDP session**:

1. Open **File Explorer** → create `C:\CorpAssets\` and drop any `.bmp` or `.jpg` in there named `wallpaper.jpg`. (Any image works — grab one from Pexels or use the default Windows wallpaper copied from `C:\Windows\Web\Wallpaper`.)
2. Share the folder: right-click `C:\CorpAssets` → **Properties** → **Sharing** → **Advanced Sharing** → ✅ **Share this folder** → **Permissions** → add **Authenticated Users** with **Read** → **OK**. UNC path is now `\\DC01\CorpAssets\wallpaper.jpg`.
3. Open **Group Policy Management** (`gpmc.msc`).
4. Right-click `_HQ > Sales` OU → **Create a GPO in this domain, and Link it here…**
   - Name: `U_Sales_Wallpaper`
5. Right-click the new GPO → **Edit**.
6. Navigate: **User Configuration** → **Policies** → **Administrative Templates** → **Desktop** → **Desktop** → double-click **Desktop Wallpaper**.
7. **Enabled** → Wallpaper Name: `\\DC01\CorpAssets\wallpaper.jpg` → Wallpaper Style: `Fill` → **OK**.
8. Close the editor.

📸 **Screenshot:** `35-gpo-created-linked.png` — GPMC showing `U_Sales_Wallpaper` linked under the Sales OU.
📸 **Screenshot:** `36-gpo-wallpaper-setting.png` — the Desktop Wallpaper policy editor showing Enabled + the UNC path.

### C.2 Apply and Verify on the Client

Log into CLIENT01 as `CORP\bwilliams` / `Password123` (Bob Williams — also in the Sales OU, and unlike `sconnor` he has no force-change-at-logon flag, so RDP works cleanly without workarounds).

The wallpaper should already be applied on your desktop as soon as you log in. **Why no `gpupdate /force`?** User-scope GPOs apply automatically during logon — Windows pulls and applies them as part of building the user session. `gpupdate /force` is only needed to refresh policies *mid-session* (for example, in C.3 where we make a change and want to see its effect without logging off). Since we're logging in fresh here, it's redundant.

Verify with `gpresult` — this is the proof that the wallpaper came from a real GPO, not a local Windows setting:

```powershell
gpresult /r /scope:user
```

Look under **Applied Group Policy Objects** — `U_Sales_Wallpaper` should be listed.

📸 **Screenshot:** `37-gpresult-applied.png` — the `gpresult /r /scope:user` output with `U_Sales_Wallpaper` in **Applied Group Policy Objects**.
📸 **Screenshot:** `38-wallpaper-applied-desktop.png` — the CLIENT01 desktop with the new corporate wallpaper visible.

### C.3 The Troubleshooting Bit — Why `cpatel` Didn't Get It

> ⚙️ **Prereq — clear `cpatel`'s force-change flag.** All users created by the A.5 bulk CSV import have `ChangePasswordAtLogon = true`, which trips the same RDP/NLA wall you saw in Scenario B. For this troubleshooting scenario we don't need to demonstrate a password change — we just need `cpatel` to log in — so the cleanest fix is to clear the flag on DC01:
>
> ```powershell
> Set-ADUser -Identity cpatel -ChangePasswordAtLogon $false
> ```
>
> Her password stays `Password123`. RDP will now let you in. (Apply the same one-liner to any other bulk-CSV user you want to log in as during the lab.)

**On CLIENT01:** log out of the current session (`bwilliams`) and log back in as `CORP\cpatel` / `Password123`. The wallpaper does **not** apply.

**Step 1 — confirm the symptom (on CLIENT01).** Still in the `cpatel` session on CLIENT01, open PowerShell and run:

```powershell
gpresult /r /scope:user
```

`U_Sales_Wallpaper` is **not** in Applied — it's under **The following GPOs were not applied because they were filtered out**, or not listed at all.

📸 **Screenshot:** `39-gpresult-not-applied.png` — gpresult output for cpatel, showing `U_Sales_Wallpaper` missing from Applied.

**Step 2 — find the root cause.** User-scope GPOs apply based on the **OU the user object lives in** (not the computer's OU). Check where `cpatel` actually lives.

> ⚠️ Run this on **DC01**, not CLIENT01. The `Get-ADUser` cmdlet comes from the Active Directory PowerShell module, which is installed on Domain Controllers by default but not on domain-joined clients. (Running it on CLIENT01 gives "The term 'Get-ADUser' is not recognized..." error.) This actually matches how real help-desk techs work — they check AD objects from their admin station, not from the user's workstation.

Switch to your DC01 RDP session → PowerShell → run:

```powershell
Get-ADUser cpatel -Properties DistinguishedName | Select-Object DistinguishedName
```

You'll see `CN=Cara Patel,OU=HR,OU=_HQ,DC=corp,DC=local` — she's in **HR**, not Sales. That's why the Sales-linked GPO doesn't target her.

📸 **Screenshot:** `40-get-aduser-dn.png` — the PowerShell output on DC01 showing cpatel's DN in the HR OU.

**Step 3 — fix it (on DC01).** In **ADUC on DC01**, navigate to `_HQ > HR`, right-click `cpatel` → **Move** → select `_HQ > Sales` → **OK**. Or run this in PowerShell on DC01:

```powershell
Get-ADUser cpatel | Move-ADObject -TargetPath "OU=Sales,OU=_HQ,DC=corp,DC=local"
```

📸 **Screenshot:** `41-user-moved-to-sales.png` — ADUC showing `cpatel` now under the Sales OU.

**Step 4 — retest (on CLIENT01).** On CLIENT01, log out of cpatel's current session and log back in as `CORP\cpatel` / `Password123`. That's it — the wallpaper appears immediately. Because user-scope GPOs apply automatically at logon, moving cpatel into the Sales OU means her next fresh login picks up the Sales-linked `U_Sales_Wallpaper` GPO without needing a separate `gpupdate /force`. (If you're curious, `gpresult /r /scope:user` will now show `U_Sales_Wallpaper` under **Applied Group Policy Objects** instead of N/A — but that's optional.)

📸 **Screenshot:** `42-cpatel-wallpaper-applied.png` — cpatel's desktop now with the corporate wallpaper.

### C.4 Bonus — Block Inheritance Gotcha

A second, slightly trickier scenario that teaches you how "Block Inheritance" breaks GPOs. **This section uses both RDP sessions — the DC01 RDP for GPMC work, and the CLIENT01 RDP to see the effect.** Keep both windows open side-by-side.

**Step 1 — On DC01 (GPMC)** — toggle Block Inheritance on the Sales OU:

1. In your **DC01 RDP session**, open **Group Policy Management** (`gpmc.msc`) if it isn't already open.
2. Expand **Forest: corp.local** → **Domains** → **corp.local** → **_HQ**.
3. **Right-click `Sales`** → click **Block Inheritance**.
4. A small **blue `!` icon** appears on the Sales OU — this is the visual indicator that Block Inheritance is on.

📸 **Screenshot (on DC01):** `43-block-inheritance-indicator.png` — GPMC showing the blue `!` icon next to the Sales OU.

**Step 2 — On CLIENT01** — observe that this *alone* didn't break the wallpaper yet.

1. In your **CLIENT01 RDP session**, log out of whatever account is currently logged in.
2. Log back in as `CORP\bwilliams` / `Password123`.
3. The wallpaper **still appears**. Why? Because the GPO is linked *directly* to the Sales OU — Block Inheritance only blocks GPOs coming *from parent OUs*, not ones linked to Sales itself.

**Step 3 — On DC01 (GPMC)** — relink the GPO to `_HQ` instead of Sales (this is the "mistake" a junior admin might make):

1. Still in **GPMC on DC01**: expand **_HQ → Sales**. You'll see the `U_Sales_Wallpaper` link under Sales.
2. **Right-click `U_Sales_Wallpaper`** (under Sales) → click **Delete**. When it asks, choose **Remove the link**. (This removes the link only — the GPO itself still exists under **Group Policy Objects** in the left pane.)
3. Now **right-click `_HQ`** (the parent OU, one level up from Sales) → click **Link an Existing GPO…**
4. In the dialog, select **`U_Sales_Wallpaper`** → **OK**. The GPO is now linked at `_HQ` instead of at Sales.

**Step 4 — On CLIENT01** — observe that the wallpaper is now broken by Block Inheritance:

1. In your **CLIENT01 RDP session**, log out of `bwilliams` and log back in as `CORP\bwilliams` / `Password123`.
2. **No wallpaper.** Block Inheritance on Sales is now preventing the `_HQ`-linked GPO from flowing down to Sales users. This is the bug that would show up in a real ticket.

📸 **Screenshot (on CLIENT01):** `44-gpresult-html-denied.png` — generate and open a full-fidelity Group Policy HTML report. In the `bwilliams` PowerShell session on CLIENT01, run:
>
> ```powershell
> gpresult /h "$env:USERPROFILE\Desktop\rsop-report.html" /f
> Start-Process "$env:USERPROFILE\Desktop\rsop-report.html"
> ```
>
> The report opens in your default browser. Scroll down to **User Details**, then look at the **Applied GPOs** and **Denied GPOs** sections — `U_Sales_Wallpaper` will show up under Denied with a reason. Capture that view.
>
> > ℹ️ Why not `rsop.msc`? It's a legacy tool that only reports a narrow slice of policies — Security Settings and Software Restriction Policies — and it does **not** show Administrative Templates (where the wallpaper setting lives). `gpresult /h` is the modern complete replacement and is what's actually used in real help-desk work today.

**Step 5 — On DC01 (GPMC)** — fix the Block Inheritance (the real-world resolution):

1. In **GPMC on DC01**, **right-click `Sales`** → click **Block Inheritance** again to untick it. The blue `!` icon disappears.

**Step 6 — On CLIENT01** — confirm the fix:

1. Log out of `bwilliams` and log back in as `CORP\bwilliams` / `Password123`.
2. Wallpaper returns. The `_HQ`-linked GPO now inherits down to Sales users because Block Inheritance is gone.

**Step 7 — On DC01 (GPMC)** — clean up so the lab is in a known good state:

1. Either leave the GPO linked at `_HQ` (it works fine there), **or** move the link back to Sales:
   - Right-click `U_Sales_Wallpaper` under `_HQ` → **Delete** → **Remove the link**.
   - Right-click `Sales` → **Link an Existing GPO…** → select `U_Sales_Wallpaper` → **OK**.
2. Confirm Block Inheritance is off on Sales (no blue `!`).

---

## 12. Cleanup & Stopping Instances

### Between Lab Sessions — Stop (don't terminate)

In EC2 console → select both instances → **Instance state** → **Stop instance**.
- You keep the EBS volumes, all config, AD state, users, GPOs.
- You pay only for EBS storage (~$0.08/GB/month = a couple dollars a month for both).

To resume: start the instances again. **Both the public IP and the auto-generated public DNS hostname change on every stop/start**, so the `.rdp` file you downloaded previously is stale. Re-download a fresh one from **EC2 → select instance → Connect → RDP client → Download remote desktop file** each time. The private IPs (`10.0.1.10` / `10.0.1.20`) persist, and the Administrator password does not change.

**Start order matters:** bring **DC01 up first** and let it fully boot (~3 minutes) before starting CLIENT01. The client depends on the DC for DNS; starting them in reverse order can leave the client unable to log in with domain accounts until it retries DNS.

### When You're Fully Done

1. **EC2** → terminate both instances.
2. Wait for them to show **terminated**, then go to **Volumes** and delete any leftover EBS volumes (terminated instances usually clean these automatically, but verify).
3. **VPC Dashboard** → **Your VPCs** → select `ad-lab-vpc` → **Actions** → **Delete VPC**. This cleans up the subnet, internet gateway, route tables, and the security group in one shot.
4. **Key Pairs** → delete `ad-lab-key` (and delete the local `.pem` file).

---

## 13. Screenshot Checklist & Naming Scheme

**Naming convention:** `NN-section-description.png`
- `NN` = two-digit sequence number so they sort correctly in a folder
- `section` = short keyword for what part of the lab (e.g., `dc`, `client`, `gpo`, `newhire`)
- `description` = hyphenated short phrase

Save everything into `screenshots/` inside this project folder.

### Complete ordered list

| #  | Filename                                   | What it shows                                                   |
|----|--------------------------------------------|-----------------------------------------------------------------|
| 01 | `01-vpc-created.png`                       | "VPC created successfully" + resource map                       |
| 02 | `02-security-group-rules.png`              | `ad-lab-sg` with RDP-from-my-IP + self-referencing all-traffic  |
| 03 | `03-dc-ec2-launched.png`                   | DC01 instance launch success page                               |
| 04 | `04-dc-password-retrieved.png`             | Decrypted Administrator password (REDACT BEFORE PUBLISHING)     |
| 05 | `05-dc-rdp-connected.png`                  | First RDP into DC01 — Server Manager dashboard                  |
| 06 | `06-dc-renamed.png`                        | System page showing pending rename to `DC01`                    |
| 07 | `07-dc-dns-configured.png`                 | DC's Ethernet: IP still DHCP, DNS manual 127.0.0.1 / 10.0.0.2   |
| 08 | `08-ad-ds-role-installing.png`             | AD DS role installation progress                                |
| 09 | `09-dcpromo-forest-config.png`             | "Add a new forest" — root domain name `corp.local`              |
| 10 | `10-dcpromo-prereq-check.png`              | Prerequisites check page                                        |
| 11 | `11-dc-promotion-complete.png`             | Server Manager showing AD DS + DNS with green checkmarks        |
| 12 | `12-dc-sanity-check-powershell.png`        | `Get-ADDomain` / `Get-ADForest` / `nslookup corp.local` output  |
| 13 | `13-client-ec2-launched.png`               | EC2 list showing both DC01 and CLIENT01 running                 |
| 14 | `14-client-dns-configured.png`             | Client Ethernet settings with DNS = 10.0.1.10                   |
| 15 | `15-client-domain-join-dialog.png`         | "Computer Name Changes" dialog joining `corp.local`             |
| 16 | `16-client-welcome-to-domain.png`          | "Welcome to the corp.local domain" confirmation                 |
| 17 | `17-client-domain-login.png`               | CLIENT01 logon screen showing `Sign in to: CORP`                |
| 18 | `18-ou-structure.png`                      | ADUC with the full OU tree expanded                             |
| 19 | `19-user-created-aduc.png`                 | IT OU showing `John Smith`                                      |
| 20 | `20-security-groups-created.png`           | Groups OU with all four `GS-` groups                            |
| 21 | `21-group-membership.png`                  | `GS-All-Employees` → Members tab with 3 users                   |
| 22 | `22-client-login-domain-user.png`          | CLIENT01 desktop after `jsmith` logs in                         |
| 23 | `23-newhire-user-properties.png`           | Sarah Connor's Organization tab                                 |
| 24 | `24-newhire-group-membership.png`          | Sarah's Member Of tab                                           |
| 25 | `25-newhire-force-password-change.png`     | "Must change password" prompt on CLIENT01                       |
| 26 | `26-bulk-import-csv.png`                   | PowerShell `Get-Content` output of `newhires.csv`               |
| 27 | `27-bulk-import-powershell.png`            | PowerShell bulk-create loop output                              |
| 28 | `28-bulk-import-aduc-verify.png`           | ADUC showing 5 new users distributed across OUs                 |
| 29 | `29-lockout-policy-configured.png`         | Account Lockout Policy with threshold=5                         |
| 30 | `30-locked-out-message.png`                | CLIENT01 logon screen with lockout error                        |
| 31 | `31-aduc-unlock-checkbox.png`              | `jsmith` Account tab with "Unlock account" ticked               |
| 32 | `32-aduc-reset-password.png`               | Reset Password dialog                                           |
| 33 | `33-powershell-unlock-reset.png`           | PowerShell unlock/reset/verify commands + `Get-ADUser` output   |
| 34 | `34-rdp-nla-forced-change-error.png`       | RDP error dialog: "must change password before logging on"      |
| 34b| `34b-user-successful-login-after-workaround.png` | (Optional) CLIENT01 desktop after admin-supplied password change |
| 35 | `35-gpo-created-linked.png`                | GPMC with `U_Sales_Wallpaper` under Sales OU                    |
| 36 | `36-gpo-wallpaper-setting.png`             | Desktop Wallpaper policy editor — Enabled + UNC path            |
| 37 | `37-gpresult-applied.png`                  | `gpresult /r /scope:user` showing GPO applied                   |
| 38 | `38-wallpaper-applied-desktop.png`         | CLIENT01 desktop with corporate wallpaper                       |
| 39 | `39-gpresult-not-applied.png`              | `gpresult` for cpatel — GPO NOT applied                         |
| 40 | `40-get-aduser-dn.png`                     | PowerShell `Get-ADUser cpatel` showing DN in HR OU              |
| 41 | `41-user-moved-to-sales.png`               | ADUC — cpatel now under Sales                                   |
| 42 | `42-cpatel-wallpaper-applied.png`          | cpatel's desktop now with the wallpaper                         |
| 43 | `43-block-inheritance-indicator.png`       | GPMC showing blue `!` on Sales from Block Inheritance           |
| 44 | `44-gpresult-html-denied.png`              | `gpresult /h` HTML report showing `U_Sales_Wallpaper` as denied |

**Total: 44 screenshots** (plus optional `34b`). Every step that produces a visually distinct artifact is captured. If you're short on time, the **minimum-viable portfolio set** is: `01, 02, 03, 11, 12, 15, 17, 18, 20, 22, 25, 28, 30, 32, 33, 34, 35, 38, 39, 40, 42` (21 shots).

### Redaction checklist before publishing

- **#04** — blur the Administrator password
- Any shot showing your **public IP** (security group rules) — blur or replace with `X.X.X.X/32`
- Any **AWS account ID** shown in the console top-right or in an ARN — blur it
- The `ad-lab-key.pem` content, if it ever ends up on screen

---

## 14. Troubleshooting Appendix

### "Can't RDP to the instance"
- Security group: is port 3389 open from *your current* IP? Your public IP changes when your network changes. Edit the SG rule.
- Instance state: is it actually `running`? Stopped instances don't respond.
- Windows Firewall: the default AWS AMI allows RDP. But after promoting to DC, sometimes the firewall profile flips to "Domain" — verify in `wf.msc`.

### "Domain join fails: cannot contact domain"
- Did you set the client's DNS to `10.0.1.10` (DC's private IP)? This is the #1 cause.
- `ping dc01.corp.local` from the client — it should resolve. If it doesn't, DNS isn't reaching the DC.
- Security group: the SG's self-reference rule must allow all traffic between instances. If you locked it down, make sure TCP 53, 88, 135, 389, 445, 464, 636, 3268–3269, and dynamic RPC (49152–65535) are open.

### "DC promotion fails at prerequisites check"
- DNS not pointing at `127.0.0.1` (preferred) and `10.0.0.2` (alternate) in Windows? Go back to Part 2.5. Ignore the promotion wizard's "dynamic IP" warning — that's expected and safe on AWS.
- Server still showing old name? Reboot after rename, then retry.

### "GPO not applying on client"
- Run `gpupdate /force` on the client.
- For **user** policies, the user must log off and log back on. For **computer** policies, a reboot.
- Run `gpresult /h report.html` and open the HTML for a detailed reason per-setting.
- Check the **scope** of the GPO (is it linked to the right OU?) and where the **user/computer object actually lives** (they're often in a different OU than you think — see Scenario C step 2).

### "Instance won't start back up"
- First, wait 3–5 minutes — Windows takes time to boot.
- Check the **System Log** in the EC2 console (Actions → Monitor and troubleshoot → Get system log). Look for the `Ec2Config` / `EC2Launch` lines.
- If you changed the machine's hostname or IP incorrectly, you may need to detach the EBS volume, mount it to a rescue instance, and fix the registry.

### "My bill went up"
- Check **Billing** → **Cost Explorer** → filter by service. Most likely culprits: EBS volumes (instances didn't terminate cleanly), Elastic IPs (you pay for EIPs NOT attached to a running instance), NAT Gateway (shouldn't exist in this lab — you picked "None").

---

## What's Next (Ideas for Portfolio Expansion)

Once the base lab is done and all screenshots are captured, easy extensions that make the portfolio richer:

- **Add a second DC** to demonstrate replication and FSMO roles
- **Deploy a file server** and map a home drive via GPO
- **Add WSUS** or **Windows Admin Center** for a light systems-admin flavor
- **Set up a site-to-site VPN** from your home to the lab VPC instead of RDP-over-public-IP
- **Write a PowerShell help-desk module** wrapping the reset/unlock/new-hire flows into clean functions

When you're ready, come back to me with the screenshots captured into `./screenshots/` and I'll build the portfolio page that tells this story.
