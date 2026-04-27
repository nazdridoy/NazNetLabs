# RHEL 8 Lab Assignment: User, Group & Network Administration

This lab covers basic server setup on RHEL 8: configuring a static IP, setting up department groups, creating user accounts, and locking down shared directories with proper permissions. Everything runs as `root`.

```
 ____  _   _ _____ _        ___    _          _       
|  _ \| | | | ____| |      ( _ )  | |    __ _| |__    
| |_) | |_| |  _| | |      / _ \  | |   / _` | '_ \   
|  _ <|  _  | |___| |___  | (_) | | |__| (_| | |_) |  
|_| \_\_| |_|_____|_____|  \___/  |_____\__,_|_.__/   
           System Administration Lab Assignment
```

---

## Scenario

**Dotcom System Ltd (DTCM)** has three departments: **Engineering**, **Marketing**, and **Finance**. A new server needs to be set up with:

```
                    ┌───────────────────────────────┐
                    │    Dotcom System Ltd (DTCM)    │
                    │     IP: 192.168.1.100/24      │
                    └───────────────┬───────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            │                       │                       │
  ┌─────────▼──────────┐  ┌────────▼─────────┐  ┌──────────▼────────┐
  │   /dtcm/           │  │   /dtcm/         │  │   /dtcm/          │
  │   engineering      │  │   marketing      │  │   finance         │
  │   drwxrws--T       │  │   drwxrws--T     │  │   drwxrws--T      │
  │                    │  │                  │  │                   │
  │  👤 nazmul         │  │  👤 banna        │  │  👤 farid         │
  │  👤 anna           │  │  👤 walid        │  │  👤 jahid         │
  │  👤 mahbub         │  │                  │  │                   │
  └────────────────────┘  └──────────────────┘  └───────────────────┘
```

1.  A static IP on the network interface.
2.  A group for each department.
3.  User accounts for the new hires.
4.  Each user placed in the right department group.
5.  Shared directories per department.
6.  Permissions locked down: only department members get access.

All work is done on a RHEL 8 VM as the `root` user.

---

## Task 1: Static IP Configuration with `nmcli`

`nmcli` talks to `NetworkManager`, which handles network config on RHEL 8 and saves it across reboots.

### 1.1 - Find the Network Interface

First, figure out what interface is available:

```bash
# List all network devices
[root@rhel8-A ~]# nmcli device status
DEVICE      TYPE      STATE      CONNECTION
enp9s0      ethernet  connected  Profile 1
enp1s0      ethernet  connected  enp1s0
virbr0      bridge    connected  virbr0
lo          loopback  unmanaged  --
virbr0-nic  tun       unmanaged  --
```

Two ethernet interfaces show up here. `enp1s0` is the one being configured, it already has a connection profile with the same name.

```bash
# Pull up all the details for this connection
[root@rhel8-A ~]# nmcli connection show enp1s0
```

Important fields in the output:

| Property | What it means |
|----------|---------------|
| `ipv4.method` | `auto` (DHCP) or `manual` (static) |
| `ipv4.addresses` | Static IP and prefix |
| `ipv4.gateway` | Default gateway |
| `ipv4.dns` | DNS server(s) |

### 1.2 - Set a Static IP

Switch from DHCP to static:

```bash
# Set method to manual and Set the IP and subnet
[root@rhel8-A ~]# nmcli connection modify enp1s0 ipv4.method manual ipv4.addresses 192.168.1.100/24

# Set the gateway
[root@rhel8-A ~]# nmcli connection modify enp1s0 ipv4.gateway 192.168.1.1

# Set DNS
[root@rhel8-A ~]# nmcli connection modify enp1s0 ipv4.dns 8.8.8.8
```

These changes get written to `/etc/sysconfig/network-scripts/` but won't take effect until the connection is reloaded.

### 1.3 - Apply the Changes

```bash
# Bounce the connection
[root@rhel8-A ~]# nmcli connection down enp1s0 && nmcli connection up enp1s0
Connection 'enp1s0' successfully deactivated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/3)
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/6)
```

Or just bring it up again:
```bash
[root@rhel8-A ~]# nmcli connection up enp1s0
```

### 1.4 - Check That It Worked

```bash
# See the IP on the interface
[root@rhel8-A ~]# ip addr show enp1s0
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:41:f6:0a brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.100/24 brd 192.168.1.255 scope global noprefixroute enp1s0
       valid_lft forever preferred_lft forever
    inet6 fe80::c010:a374:e1b7:c26d/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever

# Check the routing table
[root@rhel8-A ~]# ip route
default via 192.168.1.1 dev enp1s0 proto static metric 100
192.168.1.0/24 dev enp1s0 proto kernel scope link src 192.168.1.100

# Ping test
[root@rhel8-A ~]# ping -c 3 8.8.8.8
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=12.3 ms
...

# DNS test
[root@rhel8-A ~]# ping -c 3 google.com
```

> [!TIP]
> All the `nmcli modify` calls can go in one shot:
> ```bash
> nmcli con mod enp1s0 ipv4.method manual \
>   ipv4.addresses 192.168.1.100/24 \
>   ipv4.gateway 192.168.1.1 \
>   ipv4.dns "8.8.8.8 8.8.4.4"
> ```

### nmcli Quick Reference
| Task | Command |
|------|---------|
| List connections | `nmcli connection show` |
| Device status | `nmcli device status` |
| Change a property | `nmcli connection modify <name> <property> <value>` |
| Bring up | `nmcli connection up <name>` |
| Bring down | `nmcli connection down <name>` |
| Add a new connection | `nmcli connection add con-name <name> type ethernet ifname <dev>` |
| Delete a connection | `nmcli connection delete <name>` |

---

## Task 2: Create Department Groups

One group per department: that way file access for a whole team is controlled in one place instead of per-user rules.

### 2.1 - Create the Groups

```bash
[root@rhel8-A ~]# groupadd engineering
[root@rhel8-A ~]# groupadd marketing
[root@rhel8-A ~]# groupadd finance
```

Each command adds a line to `/etc/group` and grabs the next free GID (from the range in `/etc/login.defs`, usually 1000+).

### 2.2 - Verify

```bash
[root@rhel8-A ~]# tail -3 /etc/group
engineering:x:1001:
marketing:x:1002:
finance:x:1003:
```

Format: `groupname:password:GID:members`. Members field is empty; nobody's been added yet.

```bash
# Or look up one group directly
[root@rhel8-A ~]# getent group engineering
engineering:x:1001:
```

### Group Management Reference
| Task | Command |
|------|---------|
| Create group | `groupadd <groupname>` |
| Create with specific GID | `groupadd -g <GID> <groupname>` |
| Delete group | `groupdel <groupname>` |
| Rename group | `groupmod -n <new> <old>` |
| Look up group | `getent group <groupname>` |

---

## Task 3: Create New User Accounts

DTCM has seven new hires that need accounts.

### 3.1 - Create the Users

```bash
# Engineering
[root@rhel8-A ~]# useradd -c "Nazmul Hossain - Engineer" nazmul
[root@rhel8-A ~]# useradd -c "Atikur Rahman Anna - Engineer" anna
[root@rhel8-A ~]# useradd -c "Mahbubur Rahman - Engineer" mahbub

# Marketing
[root@rhel8-A ~]# useradd -c "Hasanul Banna - Marketer" banna
[root@rhel8-A ~]# useradd -c "Khalid Walid - Marketer" walid

# Finance
[root@rhel8-A ~]# useradd -c "Farid Ahmed - Accountant" farid
[root@rhel8-A ~]# useradd -c "Jahid Hasan - Auditor" jahid
```

`-c` sets the GECOS comment (full name + role), stored in `/etc/passwd`.

RHEL 8 `useradd` defaults:
- Home directory: `/home/<username>`
- Shell: `/bin/bash`
- Creates a private group with the same name as the user (UPG scheme)

### 3.2 - Set Passwords

```bash
[root@rhel8-A ~]# echo "P@ssw0rd" | passwd --stdin nazmul
[root@rhel8-A ~]# echo "P@ssw0rd" | passwd --stdin anna
[root@rhel8-A ~]# echo "P@ssw0rd" | passwd --stdin mahbub
[root@rhel8-A ~]# echo "P@ssw0rd" | passwd --stdin banna
[root@rhel8-A ~]# echo "P@ssw0rd" | passwd --stdin walid
[root@rhel8-A ~]# echo "P@ssw0rd" | passwd --stdin farid
[root@rhel8-A ~]# echo "P@ssw0rd" | passwd --stdin jahid
```

`--stdin` reads the password from stdin, which is handy for scripting. In a real setup, a password change on first login should be forced:

```bash
[root@rhel8-A ~]# chage -d 0 nazmul
```

### 3.3 - Verify

```bash
[root@rhel8-A ~]# grep -E "nazmul|anna|mahbub|banna|walid|farid|jahid" /etc/passwd
nazmul:x:1001:1004:Nazmul Hossain - Engineer:/home/nazmul:/bin/bash
anna:x:1002:1005:Atikur Rahman Anna - Engineer:/home/anna:/bin/bash
mahbub:x:1003:1006:Mahbubur Rahman - Engineer:/home/mahbub:/bin/bash
banna:x:1004:1007:Hasanul Banna - Marketer:/home/banna:/bin/bash
walid:x:1005:1008:Khalid Walid - Marketer:/home/walid:/bin/bash
farid:x:1006:1009:Farid Ahmed - Accountant:/home/farid:/bin/bash
jahid:x:1007:1010:Jahid Hasan - Auditor:/home/jahid:/bin/bash

# Check home dirs
[root@rhel8-A ~]# ls -ld /home/nazmul /home/anna /home/mahbub /home/banna /home/walid /home/farid /home/jahid
drwx------. 2 nazmul  nazmul  62 Apr 27 10:00 /home/nazmul
drwx------. 2 anna    anna    62 Apr 27 10:00 /home/anna
...
```

> [!NOTE]
> Right now each user's primary group is their own private group (`nazmul:nazmul`, etc.). Department groups get added as supplementary groups next.

---

## Task 4: Assign Users to Department Groups

Users have one primary group (their private group) and can belong to extra supplementary groups. The department group is added as a supplementary group with `usermod -aG`.

### 4.1 - Add Users to Groups

```bash
# Engineering
[root@rhel8-A ~]# usermod -aG engineering nazmul
[root@rhel8-A ~]# usermod -aG engineering anna
[root@rhel8-A ~]# usermod -aG engineering mahbub

# Marketing
[root@rhel8-A ~]# usermod -aG marketing banna
[root@rhel8-A ~]# usermod -aG marketing walid

# Finance
[root@rhel8-A ~]# usermod -aG finance farid
[root@rhel8-A ~]# usermod -aG finance jahid
```

`-a` means append (add to existing groups). `-G` means supplementary group.

> [!CAUTION]
> `-G` without `-a` **replaces** all supplementary groups. If someone's in `wheel` and `usermod -G finance farid` runs without `-a`, farid loses `wheel`. Always use `-aG`.

### 4.2 - Verify

```bash
[root@rhel8-A ~]# id nazmul
uid=1001(nazmul) gid=1004(nazmul) groups=1004(nazmul),1001(engineering)

[root@rhel8-A ~]# id banna
uid=1004(banna) gid=1007(banna) groups=1007(banna),1002(marketing)

[root@rhel8-A ~]# id farid
uid=1006(farid) gid=1009(farid) groups=1009(farid),1003(finance)
```

```bash
# From the group file side
[root@rhel8-A ~]# grep -E "engineering|marketing|finance" /etc/group
engineering:x:1001:nazmul,anna,mahbub
marketing:x:1002:banna,walid
finance:x:1003:farid,jahid
```

### Who's Where

| User | Role | Primary Group | Department Group |
|------|------|--------------|------------------|
| `nazmul` | Engineer | `nazmul` | `engineering` |
| `anna` | Engineer | `anna` | `engineering` |
| `mahbub` | Engineer | `mahbub` | `engineering` |
| `banna` | Marketer | `banna` | `marketing` |
| `walid` | Marketer | `walid` | `marketing` |
| `farid` | Accountant | `farid` | `finance` |
| `jahid` | Auditor | `jahid` | `finance` |

---

## Task 5: Create Department Directories

Each department gets a shared directory. Root owns them, but the group is set to the department.

### 5.1 - Create the Directories

```bash
[root@rhel8-A ~]# mkdir -p /dtcm/{engineering,marketing,finance}

[root@rhel8-A ~]# tree /dtcm
/dtcm
├── engineering
├── finance
└── marketing

3 directories, 0 files
```

`-p` creates parent dirs if needed. Brace expansion `{engineering,marketing,finance}` makes all three in one go.

### 5.2 - Set Group Ownership

```bash
[root@rhel8-A ~]# chgrp engineering /dtcm/engineering
[root@rhel8-A ~]# chgrp marketing /dtcm/marketing
[root@rhel8-A ~]# chgrp finance /dtcm/finance
```

```bash
[root@rhel8-A ~]# ls -l /dtcm/
total 0
drwxr-xr-x. 2 root engineering 6 Apr 27 10:05 engineering
drwxr-xr-x. 2 root finance     6 Apr 27 10:05 finance
drwxr-xr-x. 2 root marketing   6 Apr 27 10:05 marketing
```

Groups are right, but permissions are still wide open. That gets fixed next.

---

## Task 6: Set Permissions per Department

The access policy:

| Rule | How |
|------|-----|
| Department members can read, write, and enter their dir | Group = `rwx` |
| Everyone else is locked out | Others = `---` |
| New files inherit the department group automatically | SGID |
| Users can't delete each other's files | Sticky bit |

### 6.1 - Base Permissions

```bash
# rwx for owner and group, nothing for others
[root@rhel8-A ~]# chmod 770 /dtcm/engineering
[root@rhel8-A ~]# chmod 770 /dtcm/marketing
[root@rhel8-A ~]# chmod 770 /dtcm/finance
```

### 6.2 - SGID

SGID on a directory makes new files inside inherit the directory's group instead of the creator's primary group. So when nazmul makes a file in `/dtcm/engineering`, it belongs to `engineering`, not `nazmul`.

```bash
[root@rhel8-A ~]# chmod g+s /dtcm/engineering
[root@rhel8-A ~]# chmod g+s /dtcm/marketing
[root@rhel8-A ~]# chmod g+s /dtcm/finance
```

### 6.3 - Sticky Bit

The sticky bit stops people from deleting files they don't own, even if they have write access to the directory.

```bash
[root@rhel8-A ~]# chmod +t /dtcm/engineering
[root@rhel8-A ~]# chmod +t /dtcm/marketing
[root@rhel8-A ~]# chmod +t /dtcm/finance
```

> [!TIP]
> Or do SGID + sticky + permissions all at once:
> ```bash
> chmod 3770 /dtcm/engineering
> ```
> First digit `3` = SGID (2) + Sticky (1). Then `770` = owner(rwx) + group(rwx) + others(---).

### 6.4 - Verify

```bash
[root@rhel8-A ~]# ls -l /dtcm/
total 0
drwxrws--T. 2 root engineering 6 Apr 27 10:05 engineering
drwxrws--T. 2 root finance     6 Apr 27 10:05 finance
drwxrws--T. 2 root marketing   6 Apr 27 10:05 marketing
```

Breaking down `drwxrws--T`:

| Position | Characters | What it means |
|----------|-----------|---------------|
| 1 | `d` | Directory |
| 2–4 | `rwx` | Owner (root) - full access |
| 5–7 | `rws` | Group - read, write, execute + SGID (lowercase `s` = SGID + execute bit both on) |
| 8–10 | `--T` | Others - no access + sticky bit (uppercase `T` = sticky is on but execute is off for others) |

### 6.5 - Test It

```bash
# nazmul (engineering) - should work fine
[root@rhel8-A ~]# su - nazmul
[nazmul@rhel8-A ~]$ cd /dtcm/engineering
[nazmul@rhel8-A engineering]$ touch design_v1.txt
[nazmul@rhel8-A engineering]$ ls -l
-rw-rw-r--. 1 nazmul engineering 0 Apr 27 10:10 design_v1.txt

# Notice the file got 'engineering' as its group - SGID is working

[nazmul@rhel8-A engineering]$ exit
```

```bash
# banna (marketing) - should get blocked from engineering
[root@rhel8-A ~]# su - banna
[banna@rhel8-A ~]$ cd /dtcm/engineering
-bash: cd: /dtcm/engineering: Permission denied

# But marketing works fine
[banna@rhel8-A ~]$ cd /dtcm/marketing
[banna@rhel8-A marketing]$ touch campaign_q2.doc
[banna@rhel8-A marketing]$ exit
```

```bash
# Sticky bit test - anna can't delete nazmul's file
[root@rhel8-A ~]# su - anna
[anna@rhel8-A ~]$ cd /dtcm/engineering
[anna@rhel8-A engineering]$ rm design_v1.txt
rm: cannot remove 'design_v1.txt': Operation not permitted

# But she can manage her own stuff
[anna@rhel8-A engineering]$ touch anna_notes.txt
[anna@rhel8-A engineering]$ rm anna_notes.txt
[anna@rhel8-A engineering]$ exit
```

---

## Summary

```
                    ┌───────────────────────────────┐
                    │    Dotcom System Ltd (DTCM)    │
                    │     IP: 192.168.1.100/24      │
                    └───────────────┬───────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            │                       │                       │
  ┌─────────▼──────────┐  ┌────────▼─────────┐  ┌──────────▼────────┐
  │   /dtcm/           │  │   /dtcm/         │  │   /dtcm/          │
  │   engineering      │  │   marketing      │  │   finance         │
  │   drwxrws--T       │  │   drwxrws--T     │  │   drwxrws--T      │
  │                    │  │                  │  │                   │
  │  👤 nazmul         │  │  👤 banna        │  │  👤 farid         │
  │  👤 anna           │  │  👤 walid        │  │  👤 jahid         │
  │  👤 mahbub         │  │                  │  │                   │
  └────────────────────┘  └──────────────────┘  └───────────────────┘
```

| #   | Task                        | Commands Used                                    |
| --- | --------------------------- | ------------------------------------------------ |
| 1   | Static IP setup             | `nmcli connection modify`, `nmcli connection up` |
| 2   | Department groups           | `groupadd`                                       |
| 3   | User accounts               | `useradd -c`, `passwd --stdin`                   |
| 4   | Group assignment            | `usermod -aG`                                    |
| 5   | Shared directories          | `mkdir -p`, `chgrp`                              |
| 6   | Permissions + SGID + sticky | `chmod 3770`, `chmod g+s`, `chmod +t`            |

> [!IMPORTANT]
> After adding someone to a group with `usermod -aG`, run `id <username>` to confirm the new membership.

> [!WARNING]
> Don't use `usermod -G` without `-a`. That wipes out all existing supplementary groups: it could accidentally kick someone out of `wheel`.

> [!NOTE]
> Group membership changes only show up in new sessions. A logged-in user needs to log out and back in (or run `newgrp <groupname>`) for the change to take effect.
