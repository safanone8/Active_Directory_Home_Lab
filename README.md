# Active Directory Home Lab

**Windows Server 2022 Domain Controller + Windows 11 client, built in Oracle VirtualBox**

![Windows Server](https://img.shields.io/badge/Windows%20Server-2022-blue)
![Windows](https://img.shields.io/badge/Windows-11-blue)
![VirtualBox](https://img.shields.io/badge/Oracle-VirtualBox-orange)
![Domain](https://img.shields.io/badge/Domain-LAB.local-green)

## Overview

In this project I built a small company network on my own computer using two virtual machines (VMs):

- **DC01** runs Windows Server 2022. It is the **Domain Controller**: the main server that stores every user account and checks every login.
- **CLIENT01** runs Windows 11. It acts like an employee's work computer and is joined to the domain.

Once the network was working, I used it for everyday IT help desk tasks: creating users, forcing new users to change their password, unlocking a locked account, and resetting a forgotten password.

Every step is written in simple language, so anyone can build the same lab.

## Skills Demonstrated

- Creating and configuring virtual machines in Oracle VirtualBox
- Virtual networking with NAT and Host-only adapters
- Setting static IP addresses and DNS settings
- Installing Active Directory Domain Services (AD DS) and promoting a server to a Domain Controller
- DNS for Active Directory
- Joining a Windows 11 computer to a domain
- Organizing Active Directory with Organizational Units (OUs)
- Creating user accounts that must change their password at first login
- Setting an Account Lockout Policy with Group Policy
- Help desk tasks: unlocking accounts and resetting passwords
- Basic troubleshooting with `ipconfig`, `ping`, `whoami` and `gpupdate`

## Table of Contents

1. [Lab Setup at a Glance](#lab-setup-at-a-glance)
2. [What You Need](#what-you-need)
3. [Key Terms in Plain English](#key-terms-in-plain-english)
4. [Part 1: Install VirtualBox](#part-1-install-virtualbox)
5. [Part 2: Create the Virtual Machines](#part-2-create-the-virtual-machines)
6. [Part 3: Set Up the Network Adapters](#part-3-set-up-the-network-adapters)
7. [Part 4: Install Windows Server 2022](#part-4-install-windows-server-2022)
8. [Part 5: Prepare the Server (Name and Static IP)](#part-5-prepare-the-server-name-and-static-ip)
9. [Part 6: Install Active Directory and Create LAB.local](#part-6-install-active-directory-and-create-lablocal)
10. [Part 7: Set Up the Windows 11 Client](#part-7-set-up-the-windows-11-client)
11. [Part 8: Join Windows 11 to the Domain](#part-8-join-windows-11-to-the-domain)
12. [Part 9: Create an OU and Users](#part-9-create-an-ou-and-users)
13. [Part 10: Help Desk Tasks (Lockout, Unlock, Password Reset)](#part-10-help-desk-tasks-lockout-unlock-password-reset)
14. [How I Verified Everything Works](#how-i-verified-everything-works)
15. [Troubleshooting](#troubleshooting)

## Lab Setup at a Glance

```text
            ┌──────────────── Internet ────────────────┐
            │                                          │
            │ Adapter 1: NAT                           │ Adapter 1: NAT
            │                                          │
  ┌─────────┴─────────┐                      ┌─────────┴─────────┐
  │       DC01        │                      │     CLIENT01      │
  │  Windows Server   │                      │    Windows 11     │
  │ 2022 (AD DS, DNS) │                      │  (domain member)  │
  └─────────┬─────────┘                      └─────────┬─────────┘
            │ Adapter 2: Host-only                     │ Adapter 2: Host-only
            │ 192.168.56.10                            │ 192.168.56.20
            │                                          │
            └────────── Private lab network ───────────┘
                          192.168.56.0/24
```

| Setting | DC01 (server) | CLIENT01 (workstation) |
|---|---|---|
| Operating system | Windows Server 2022 Standard (Desktop Experience) | Windows 11 Pro / Enterprise |
| Job in the lab | Domain Controller + DNS server | Employee PC joined to the domain |
| Adapter 1: NAT | Automatic (gives internet) | Automatic (gives internet) |
| Adapter 2: Host-only IP | `192.168.56.10` (static) | `192.168.56.20` (static) |
| Subnet mask | `255.255.255.0` | `255.255.255.0` |
| Default gateway (Host-only) | Leave empty | Leave empty |
| Preferred DNS (Host-only) | `192.168.56.10` (itself) | `192.168.56.10` (the DC) |
| Suggested RAM / CPUs / disk | 4 GB / 2 / 50 GB | 4 GB / 2 / 50 GB |

**Domain:** `LAB.local` (short NetBIOS name: `LAB`)

> [!NOTE]
> `192.168.56.x` is VirtualBox's default host-only network. I used `.10` and `.20` because VirtualBox's built-in DHCP hands out addresses from `.101` upward, so these fixed addresses never clash with it.

## What You Need

- A computer with virtualization (Intel VT-x or AMD-V) turned on in the BIOS/UEFI, 16 GB of RAM recommended, and at least 80 GB of free disk space
- **Oracle VirtualBox**: https://www.virtualbox.org/wiki/Downloads
- **Windows Server 2022 ISO**: free 180-day evaluation from the Microsoft Evaluation Center (https://www.microsoft.com/en-us/evalcenter)
- **Windows 11 ISO**: the Pro edition (https://www.microsoft.com/software-download/windows11) or the free 90-day Enterprise evaluation from the Evaluation Center

> [!WARNING]
> Windows 11 **Home** cannot join a domain. Use Pro, Enterprise or Education.

## Key Terms in Plain English

- **Virtual machine (VM):** a computer running inside a window on your real computer. You can break it and rebuild it without harming your real PC.
- **Active Directory (AD):** the company's central list of every user and computer, plus the rules for who can log in where. Think of it as the employee directory and the security desk in one.
- **Domain Controller (DC):** the server that runs Active Directory. When someone logs in, the DC checks their username and password, like a security guard checking ID badges.
- **Domain (`LAB.local`):** the name of the company network. A computer "joins" the domain the way a new employee joins a company; after that, company accounts can log in on it.
- **DNS:** the network's phone book. It turns names like `LAB.local` into addresses like `192.168.56.10`. Computers find the Domain Controller through DNS, which is why the Windows 11 PC must use the server as its DNS.
- **Static IP:** an address that never changes, like a house address. Servers need one so other computers can always find them.
- **NAT adapter:** gives a VM internet by sharing your real computer's connection.
- **Host-only adapter:** a private "cable" between the VMs and your computer, with no internet. All the lab traffic (logins, DNS, domain join) travels here.
- **Organizational Unit (OU):** a folder inside Active Directory for grouping users and computers, usually by department (like Accounting). Rules can be applied to a whole OU.
- **Group Policy (GPO):** rules the server pushes out to users and computers, for example "lock the account after 5 wrong passwords".
- **Account lockout:** a safety feature that blocks an account after too many wrong passwords, so nobody can keep guessing.

## Part 1: Install VirtualBox

1. Go to https://www.virtualbox.org/wiki/Downloads and download the installer for your computer (for example, **Windows hosts**).
2. Run the installer and keep the default options.
3. If it warns that your network will disconnect for a moment, click **Yes**. VirtualBox is just adding its virtual network adapters.
4. Click **Finish** and open VirtualBox.

> [!TIP]
> If a VM refuses to start with a message about VT-x or AMD-V, turn on virtualization in your computer's BIOS/UEFI settings.

## Part 2: Create the Virtual Machines

### 2.1 Windows Server 2022 VM (DC01)

1. In VirtualBox, click **New**.
2. **Name:** `DC01`. **ISO Image:** select your Windows Server 2022 ISO. VirtualBox fills in the type and version for you.
3. Tick **Skip Unattended Installation**, so you can choose the correct edition yourself in Part 4.
4. **Hardware:** Base Memory `4096 MB`, Processors `2`.
5. **Hard Disk:** create a new virtual hard disk of `50 GB`.
6. Click **Finish**. Don't start the VM yet; set up its network first (Part 3).

### 2.2 Windows 11 VM (CLIENT01)

1. Click **New** again.
2. **Name:** `CLIENT01`. **ISO Image:** select your Windows 11 ISO.
3. Tick **Skip Unattended Installation**, so you can choose the **Pro** edition yourself.
4. **Hardware:** Base Memory `4096 MB`, Processors `2`. Recent versions of VirtualBox turn on EFI, Secure Boot and TPM 2.0 automatically for Windows 11, which Windows 11 needs. If Windows setup later says the PC can't run Windows 11, check these under **Settings → System**.
5. **Hard Disk:** `50 GB` (the minimum for Windows 11).
6. Click **Finish**.


## Part 3: Set Up the Network Adapters

Set these up **before** installing Windows, so each VM has both networks from its very first boot. Each VM gets two adapters:

| Adapter | Type | Why |
|---|---|---|
| Adapter 1 | NAT | Internet access (Windows updates, downloads) |
| Adapter 2 | Host-only | Private lab network where the server and PC talk to each other (DNS, logins, domain join) |

**First, check that the host-only network exists:**

1. In VirtualBox, open the Network Manager (**File → Tools → Network Manager**) and click the **Host-only Networks** tab.
2. You should see one network (on a Windows computer it's called *VirtualBox Host-Only Ethernet Adapter*) with the address `192.168.56.1` and mask `255.255.255.0`.
3. If the list is empty, click **Create**.

**Then, for BOTH VMs (make these changes while the VM is turned off):**

1. Select the VM → **Settings → Network**.
2. **Adapter 1:** tick **Enable Network Adapter** → **Attached to:** `NAT`.
3. **Adapter 2:** tick **Enable Network Adapter** → **Attached to:** `Host-only Adapter` → **Name:** the host-only network from above.
4. Click **OK**.

> [!IMPORTANT]
> Both VMs must be connected to the **same** host-only network, or they won't be able to see each other.



## Part 4: Install Windows Server 2022

1. Select **DC01** → **Start**.
2. If you see *Press any key to boot from CD or DVD*, click inside the VM window and press any key.
3. Choose your language, time and keyboard → **Next** → **Install now**.
4. Choose **Windows Server 2022 Standard Evaluation (Desktop Experience)** which usually is 2nd option. The options without "(Desktop Experience)" are Server Core, which has no desktop, only a command line.
5. Accept the license terms → choose **Custom: Install Microsoft Server Operating System only (advanced)**.
6. Select **Drive 0 Unallocated Space** → **Next**. Windows installs and restarts a few times.
7. Create a strong password for the built-in **Administrator** account → **Finish**.
8. To reach the sign-in screen, press **Right Ctrl + Delete** (or use the menu **Input → Keyboard → Insert Ctrl-Alt-Del**). The normal Ctrl + Alt + Delete goes to your real computer instead of the VM.
9. Sign in. **Server Manager** opens by itself.

**Optional: install Guest Additions** (better screen size, smoother mouse, copy and paste between your PC and the VM):

1. In the VM window menu, click **Devices → Insert Guest Additions CD image**.
2. Open **File Explorer** → the CD drive → run **VBoxWindowsAdditions** → install with the defaults → restart.
3. Turn on copy and paste with **Devices → Shared Clipboard → Bidirectional**.

## Part 5: Prepare the Server (Name and Static IP)

### 5.1 Rename the server

Do this **before** installing Active Directory. Renaming a server after it becomes a Domain Controller is much harder.

1. In **Server Manager**, click **Local Server** on the left.
2. Click the name next to **Computer name** (it looks like `WIN-8A3B...`).
3. In **System Properties**, click **Change…** → type `DC01` → **OK**.
4. Restart when asked.

### 5.2 Find out which adapter is which

1. Press **Win + R**, type `cmd` and press **Enter**. Then run:
   ```
   ipconfig
   ```
2. The adapter with an address like `10.0.2.15` is the **NAT** adapter. The other one (`192.168.56.x` or `169.254.x.x`) is the **Host-only** adapter. Note their names, for example *Ethernet* and *Ethernet 2*.
3. Press **Win + R**, type `ncpa.cpl` and press **Enter** to open **Network Connections**.
4. Right-click each adapter → **Rename** → name them `NAT` and `Lab-HostOnly`. Clear names make every next step easier.

### 5.3 Give the server a static IP

1. In **Network Connections**, right-click **Lab-HostOnly** → **Properties**.
2. Double-click **Internet Protocol Version 4 (TCP/IPv4)**.
3. Select **Use the following IP address** and enter:
   - **IP address:** `192.168.56.10`
   - **Subnet mask:** `255.255.255.0`
   - **Default gateway:** leave it **empty**
4. Select **Use the following DNS server addresses** and enter:
   - **Preferred DNS server:** `192.168.56.10`
5. Click **OK** → **Close**.
6. Leave the **NAT** adapter as it is (automatic).
7. Run `ipconfig` again and check that **Lab-HostOnly** now shows `192.168.56.10`.

**Why these settings?**

- **Static IP:** the Domain Controller must always be at the same address, or other computers lose track of it.
- **DNS points to itself:** this server is about to become the DNS server for `LAB.local`.
- **Empty gateway:** the NAT adapter already provides internet. A gateway on the host-only adapter can make Windows send internet traffic down the private network, where there is no internet.


## Part 6: Install Active Directory and Create LAB.local

> [!TIP]
> Take a VirtualBox snapshot first (in the VM window: **Machine → Take Snapshot**). If anything goes wrong in this part, you can roll back in seconds.

### 6.1 Install the AD DS role

1. In **Server Manager**, click **Manage → Add Roles and Features**.
2. **Before You Begin:** click **Next**.
3. **Installation Type:** choose **Role-based or feature-based installation** → **Next**.
4. **Server Selection:** make sure **DC01** is selected → **Next**.
5. **Server Roles:** tick **Active Directory Domain Services** → click **Add Features** in the pop-up → **Next**.
6. **Features:** click **Next**. **AD DS:** click **Next**.
7. **Confirmation:** click **Install** and wait for it to finish.

### 6.2 Promote the server to a Domain Controller

Installing the role only adds the software. **Promoting** the server is the step that actually creates the domain.

1. In **Server Manager**, click the **yellow warning flag** at the top → **Promote this server to a domain controller**.
2. **Deployment Configuration:** choose **Add a new forest** → **Root domain name:** `LAB.local` → **Next**.
3. **Domain Controller Options:**
   - Leave the forest and domain functional levels as they are (**Windows Server 2016** is the highest level available on Server 2022).
   - Keep **Domain Name System (DNS) server** and **Global Catalog (GC)** ticked.
   - Type a **DSRM password** twice. This is a separate emergency password for repairing Active Directory, so write it down.
   - Click **Next**.
4. **DNS Options:** a warning about DNS delegation appears. This is normal in a lab → **Next**.
5. **Additional Options:** wait a few seconds for the **NetBIOS domain name** to fill in as `LAB` → **Next**.
6. **Paths:** keep the defaults → **Next**.
7. **Review Options:** click **Next**. (Optional: **View script** shows the PowerShell command that does the same thing.)
8. **Prerequisites Check:** a few yellow warnings are normal in a lab (for example about DNS delegation, or about an adapter without a static IP, which is the NAT adapter). Once it says all prerequisite checks passed, click **Install**.
9. The server restarts by itself when it's done.

> [!NOTE]
> `.local` is fine for a practice lab. In a real company, Microsoft recommends using a subdomain of a domain you own (for example `ad.company.com`).

### 6.3 Sign in and check the domain

1. After the restart, the sign-in screen shows `LAB\Administrator`. Sign in with the same Administrator password as before.
2. Open **Server Manager → Tools → Active Directory Users and Computers**. You should see `LAB.local`.
3. Open **Server Manager → Tools → DNS** → **DC01 → Forward Lookup Zones**. You should see `LAB.local` and `_msdcs.LAB.local`.

> [!NOTE]
> After promotion, the server's DNS setting may change to `127.0.0.1`. That's normal: `127.0.0.1` always means "this computer".



## Part 7: Set Up the Windows 11 Client

### 7.1 Install Windows 11

1. Select **CLIENT01** → **Start**. If you see *Press any key to boot from CD or DVD*, press a key.
2. Choose your language and keyboard, then continue to install Windows 11. If asked for a product key, click **I don't have a product key**.
3. When asked for the edition, choose **Windows 11 Pro**, because Home can't join a domain. (The Enterprise evaluation ISO skips this step.)
4. Accept the license and install on **Drive 0 Unallocated Space**.
5. Finish the first-time setup and create a **local** account. If setup pushes you to use a Microsoft account, choose **Set up for work or school → Sign-in options → Domain join instead**.
6. (Optional) Install Guest Additions the same way as on the server.

### 7.2 Rename the PC

1. Open **Settings → System → About**.
2. Click **Rename this PC** → type `CLIENT01` → **Next** → **Restart now**.

### 7.3 Point the PC to the Domain Controller for DNS

1. Open **Command Prompt** and run `ipconfig`. The adapter with `10.0.2.15` is **NAT**; the other one is **Host-only**.
2. Press **Win + R** → `ncpa.cpl` → rename the adapters `NAT` and `Lab-HostOnly`.
3. Right-click **Lab-HostOnly** → **Properties** → double-click **Internet Protocol Version 4 (TCP/IPv4)** and enter:
   - **IP address:** `192.168.56.20`
   - **Subnet mask:** `255.255.255.0`
   - **Default gateway:** leave it **empty**
   - **Preferred DNS server:** `192.168.56.10` (the Domain Controller)
4. Click **OK** → **Close**.

**Why this matters:** to join the domain, the PC has to find the Domain Controller, and it finds it by asking DNS. Only the DC's DNS knows where `LAB.local` is. If the PC asks a normal internet DNS server instead, the domain join fails. This is one of the most common domain problems at a help desk.

> [!WARNING]
> Don't add a public DNS server (like `8.8.8.8`) as the alternate DNS on this adapter. It doesn't know about `LAB.local` and can cause random login problems.

### 7.4 Test the connection

Open **Command Prompt** and run:

```
ping 192.168.56.10
ping LAB.local
```

Both should get replies from `192.168.56.10`. If `ping LAB.local` fails, fix DNS before moving on (see [Troubleshooting](#troubleshooting)).

## Part 8: Join Windows 11 to the Domain

1. Press **Win + R** → type `sysdm.cpl` → **Enter** to open **System Properties**.
2. On the **Computer Name** tab, click **Change…**.
3. Under **Member of**, choose **Domain** → type `LAB.local` → **OK**.
4. When asked for an account with permission to join the domain, enter `LAB\Administrator` and its password.
5. A message welcomes you to the LAB.local domain → **OK**.
6. Click **OK** on the restart message → **Close** → **Restart Now**.
7. At the sign-in screen, click **Other user** and sign in as `LAB\Administrator` to confirm domain logins work.

**Check on the server:** in **Active Directory Users and Computers**, open **LAB.local → Computers**. **CLIENT01** should now be listed.

> [!TIP]
> Another way to join: **Settings → Accounts → Access work or school → Connect → Join this device to a local Active Directory domain**.


## Part 9: Create an OU and Users

### 9.1 Create the Accounting OU

1. On **DC01**, open **Server Manager → Tools → Active Directory Users and Computers** (or **Win + R** → `dsa.msc`).
2. Right-click **LAB.local** → **New → Organizational Unit**.
3. **Name:** `Accounting`. Leave **Protect container from accidental deletion** ticked → **OK**.

**Why not just use the built-in Users folder?** The built-in **Users** and **Computers** folders are *containers*, not OUs, so Group Policy can't be applied to them. Your own OUs let you group people by department and apply rules to just that department.

### 9.2 Create users who must change their password at first login

1. Right-click the **Accounting** OU → **New → User**.
2. Fill in **First name**, **Last name** and **User logon name** (for example *John Smith* → `jsmith`) → **Next**.
3. Type a temporary password twice.
4. Tick **User must change password at next logon**. Leave the other boxes unticked → **Next** → **Finish**.
5. Repeat for more users.

Example users:

| Full name | Logon name | OU |
|---|---|---|
| John Smith | `jsmith` | Accounting |
| Emma Brown | `ebrown` | Accounting |

**Why force a password change?** The IT person who created the account knows the temporary password. Making the user change it means only the employee knows their real password, which is standard practice in real companies.

> [!NOTE]
> By default, domain passwords must be at least 7 characters long, use 3 of these 4: uppercase letters, lowercase letters, numbers, symbols, and can't contain the user's name.


### 9.3 Test the first login

1. On **CLIENT01**, sign out → click **Other user**.
2. Sign in as `LAB\jsmith` with the temporary password.
3. Windows says the password must be changed → **OK** → type a new password twice → press **Enter**.
4. Once signed in, open **Command Prompt** and run `whoami`. It should show `lab\jsmith`.

## Part 10: Help Desk Tasks (Lockout, Unlock, Password Reset)

### 10.1 Turn on account lockout

A brand-new domain **never locks accounts** (the lockout threshold is 0), so the **Unlock account** option stays greyed out. To practise unlocking, first create a lockout rule:

1. On **DC01**, open **Server Manager → Tools → Group Policy Management**.
2. Expand **Forest: LAB.local → Domains → LAB.local**.
3. Right-click **Default Domain Policy** → **Edit…**
4. Go to **Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Account Lockout Policy**.
5. Double-click **Account lockout threshold** → set it to `5` invalid logon attempts → **OK**.
6. Windows suggests values for **Account lockout duration** and **Reset account lockout counter after** → click **OK** to accept them.
7. Close the editor, open **Command Prompt** on DC01 and run:
   ```
   gpupdate /force
   ```

> [!NOTE]
> Lockout and password rules for domain accounts only work when they are set in a policy linked to the whole domain (like the Default Domain Policy), not on an OU.


### 10.2 Ticket #1: "I'm locked out"

**Simulate the problem:**

1. On **CLIENT01**, try to sign in as `LAB\jsmith` with the **wrong** password 5 times.
2. The account is now locked. Any further sign-in attempt, even with the right password, is refused with a "locked out" message.

**Fix it:**

1. On **DC01**, open **Active Directory Users and Computers** → **Accounting**.
2. Right-click **John Smith** → **Properties** → **Account** tab.
3. Tick **Unlock account** (Windows shows that the account is currently locked out) → **Apply** → **OK**.
4. The user can now sign in with the correct password.

### 10.3 Ticket #2: "I forgot my password"

1. In **Active Directory Users and Computers**, right-click **John Smith** → **Reset Password…**
2. Type a new temporary password twice.
3. Tick **User must change password at next logon**.
4. If the account is also locked, tick **Unlock the user's account** in the same window.
5. Click **OK**. Windows confirms the password was changed.
6. On **CLIENT01**, sign in as `LAB\jsmith` with the temporary password. Windows makes the user choose a new one.

> [!TIP]
> In a real job, always verify the caller's identity (for example, employee ID or a manager's confirmation) before unlocking an account or resetting a password.


## How I Verified Everything Works

| Check | How | Expected result |
|---|---|---|
| Server has a static IP | `ipconfig` on DC01 | Lab-HostOnly shows `192.168.56.10` |
| Domain and DNS exist | Active Directory Users and Computers, DNS Manager | `LAB.local` domain and DNS zone are there |
| Client can find the domain | `ping LAB.local` on CLIENT01 | Replies from `192.168.56.10` |
| Client joined the domain | ADUC → **Computers** | **CLIENT01** is listed |
| Domain user can sign in | `whoami` on CLIENT01 | `lab\jsmith` |
| Login was checked by the DC | `echo %logonserver%` on CLIENT01 | `\\DC01` |
| Forced password change works | First sign-in as a new user | Asked to change the password |
| Lockout works | 5 wrong passwords | Account is locked out |
| Unlock works | ADUC → **Account** tab | User can sign in again |
| Password reset works | ADUC → **Reset Password** | Temporary password works, then must be changed |

## Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| *"An Active Directory Domain Controller (AD DC) for the domain LAB.local could not be contacted"* when joining | Windows 11 isn't using the DC for DNS | Set the Preferred DNS on **Lab-HostOnly** to `192.168.56.10`, make sure both VMs use the same host-only network, then test with `ping LAB.local`. If it still fails, set the **NAT** adapter's DNS to `192.168.56.10` too, so every lookup goes to the DC |
| No option to join a domain | Windows 11 Home edition | Use Windows 11 Pro, Enterprise or Education |
| VM lost internet after setting the static IP | A default gateway was added on the host-only adapter | Clear the gateway on **Lab-HostOnly**. The NAT adapter handles internet |
| `ping LAB.local` replies from `10.0.2.15` | The DC also registered its NAT address in DNS | On DC01: **NAT** adapter → IPv4 → **Advanced → DNS** tab → untick **Register this connection's addresses in DNS**. In DNS Manager, delete Host (A) records that point to `10.0.2.15`. Then run `ipconfig /flushdns` on CLIENT01 |
| **Unlock account** box is greyed out | The account isn't locked (lockout is off by default) | Set the lockout policy from Part 10.1 and run `gpupdate /force` |
| New password is rejected | It breaks the complexity rules or was used before | Use 7+ characters with 3 of: uppercase, lowercase, number, symbol, and don't include the user's name |
| Pinging CLIENT01 from DC01 fails | Windows 11's firewall blocks incoming ping by default | Normal. Test from the client to the server instead |
| *"The trust relationship between this workstation and the primary domain failed"* | Often happens after restoring an old snapshot of the client | Sign in with the local account and rejoin the domain, or run `Test-ComputerSecureChannel -Repair -Credential LAB\Administrator` in PowerShell as admin |
| Ctrl + Alt + Delete doesn't work in the VM | The keys go to your real computer | Press **Right Ctrl + Delete**, or use **Input → Keyboard → Insert Ctrl-Alt-Del** |

## Author

**Safan** · [LinkedIn](https://www.linkedin.com/in/safan-vhora/)
