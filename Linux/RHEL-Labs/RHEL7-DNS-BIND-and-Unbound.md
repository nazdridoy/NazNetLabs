# RHEL 7 DNS Server Lab: BIND (Authoritative) and Unbound (Caching)

This lab covers setting up two complementary DNS server roles on Red Hat Enterprise Linux 7: an **authoritative name server** using BIND, and a **caching recursive resolver** using Unbound. Both are deployed together to create a complete internal DNS infrastructure.

---

# 1. RHEL 7 Authoritative DNS Server (BIND)

The **Berkeley Internet Name Domain (BIND)** is the most widely deployed DNS software on the Internet. On RHEL 7, BIND is provided by the `bind` package and managed by the `named` service via `systemd`. An authoritative DNS server holds the definitive records for one or more DNS zones and answers queries directly from its own zone database. Unlike a recursive resolver, it does not forward or cache answers from upstream servers.

---

## 1.1 Conceptual Foundations

### DNS Hierarchy and Zones

The DNS namespace is a distributed, hierarchical tree. BIND manages **zones**, which are administrative units of the namespace. For full forward and reverse name resolution, two zone types are needed:

| Zone Type | Purpose | Example Zone Name |
| --- | --- | --- |
| **Forward Zone** | Resolves hostnames → IP addresses (`A` / `AAAA` records) | `nhr.local` |
| **Reverse Zone** | Resolves IP addresses → hostnames (`PTR` records) | `1.168.192.in-addr.arpa` |

### Key Record Types

| Record  | Full Name          | Function                                                   |
| ------- | ------------------ | ---------------------------------------------------------- |
| `SOA`   | Start of Authority | Declares the primary nameserver and zone timing parameters |
| `NS`    | Name Server        | Lists authoritative nameservers for the zone               |
| `A`     | Address            | Maps a hostname to an IPv4 address                         |
| `PTR`   | Pointer            | Maps an IPv4 address to a hostname (reverse lookup)        |
| `MX`    | Mail Exchanger     | Directs email delivery                                     |
| `CNAME` | Canonical Name     | Creates an alias for another hostname                      |

---

## 1.2 Lab Environment and Server Provisioning

### Lab Environment Topology

| Role              | Hostname           | IP Address     |
| ----------------- | ------------------ | -------------- |
| DNS Server (BIND) | `ns1.nhr.local`    | `192.168.1.10` |
| Client Machine    | `client.nhr.local` | `192.168.1.20` |

### Server Hardware Sizing Requirements

The following hardware covers a medium-sized company with up to 10,000 active devices and typical DNS query loads:

| Resource | Minimum Specification | Recommended Sizing | Purpose / Notes |
| --- | --- | --- | --- |
| **CPU** | 1 vCPU / Core | 2 vCPUs / Cores | BIND is multi-threaded; extra cores handle concurrent crypto validation and log compression. |
| **RAM** | 1 GB | 2 GB - 4 GB | Memory scales with the size of cached zone databases. 2GB easily handles tens of thousands of records. |
| **Disk** | 10 GB | 30 GB (SSD preferred) | Required for the RHEL OS, log file storage, query logs, and scheduled system backups. |
| **Network** | 100 Mbps | 1 Gbps | Low-latency network adapter for responsive DNS querying. |

> [!NOTE]
> All commands on the server are executed as `root` or with `sudo`. The domain used throughout this lab is `nhr.local`.

---

## 1.3 Installation

```bash
# Install the BIND packages
sudo yum install -y bind bind-utils

# Enable and start the named service
sudo systemctl enable named
sudo systemctl start named

# Verify the service is running
sudo systemctl status named
```

---

## 1.4 BIND Configuration: Split-File Structure (`/etc/named/`)

On RHEL, BIND's configuration is organized into multiple files to improve maintainability. The three key files are:

- `/etc/named.conf` - main entry point; uses `include` directives to load the others
- `/etc/named/named.conf.options` - global daemon behavior and ACLs
- `/etc/named/named.conf.local` - zone declarations and local overrides

### `/etc/named.conf`

```bash
sudo vi /etc/named.conf
```

```text
#
# RHEL BIND Authoritative DNS Server: Main Configuration
# Domain: nhr.local | Server: ns1.nhr.local (192.168.1.10)
# This file is the top-level entry point; it loads the three split
# configuration files and defines global logging / statistics channels.
#

# Global behaviour (options, access controls, forwarders, DNSSEC)
include "/etc/named/named.conf.options";

# Zone declarations and local overrides (nhr.local, reverse zone, DDNS keys)
include "/etc/named/named.conf.local";

# Optional: default localhost and broadcast zones shipped with BIND
#include "/etc/named.rfc1912.zones";

# Logging: write detailed query logs to a dedicated file instead of syslog
logging {
    channel query_log {
        # Log file location with log rotation (3 old copies, max 20 MB each)
        file "/var/log/named/named.log" versions 3 size 20m;
        severity info;              # Capture all queries and general events
        print-time yes;             # Timestamp every entry
        print-severity yes;         # Show log level (info, warning, error …)
        print-category yes;         # Show the BIND category (queries, config …)
    };
    # Route BIND categories to the query_log channel
    category queries { query_log; };   # Every incoming/outgoing DNS query
    category general { query_log; };   # Startup, shutdown, reload messages
    category config  { query_log; };   # Configuration parse/load events
};

# Statistics channel: exposes BIND metrics on 127.0.0.1:8053 for
# external monitoring tools (Nagios, Zabbix). Must be top-level, not
# inside "options".
statistics-channels {
    inet 127.0.0.1 port 8053 allow { 127.0.0.1; };
};
```

### `/etc/named/named.conf.options`

```bash
sudo vi /etc/named/named.conf.options
```

```text
#
# RHEL BIND: Global Options and Access Controls
# Requirements covered:
#   - Server setup: listen interfaces, directory paths
#   - Security: restrict zone transfers (allow-transfer), access controls
#   - DNSSEC: global validation and key management
#   - Forwarding: upstream public DNS resolvers for external queries
#

options {
    # Server Setup: Listening Interfaces
    # DNS listens on TCP and UDP port 53 by default.
    # Bind to the server's internal IP and localhost only.
    # Do NOT expose to 0.0.0.0 unless this server must face the public Internet.
    listen-on port 53 { 127.0.0.1; 192.168.1.10; };
    listen-on-v6 port 53 { ::1; };

    # Server Setup: File Paths
    # "directory" is the base path used by relative paths in zone files.
    directory       "/var/named";
    dump-file       "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";

    # Security: Access Controls
    # Only the local host and the internal corporate subnet (192.168.1.0/24)
    # are allowed to query this DNS server. All other sources are dropped.
    allow-query     { localhost; 192.168.1.0/24; };

    # Security: Restrict Zone Transfers
    # By default, NO external host is permitted to perform a full AXFR zone
    # transfer. This prevents reconnaissance attacks that enumerate the
    # entire corporate DNS structure. Override per-zone with an ACL if a
    # secondary/slave nameserver (e.g., 192.168.1.15) is added later.
    allow-transfer  { none; };

    # Forwarding: Upstream Public DNS Resolvers
    # Recursion is enabled so internal clients can resolve external names
    # (e.g., www.google.com). Queries for non-local zones are forwarded to
    # Google Public DNS instead of being resolved iteratively from the
    # root servers, which is faster for end users.
    recursion yes;
    forwarders {
        8.8.8.8;      # Google Public DNS: primary
        8.8.4.4;      # Google Public DNS: secondary
    };
    forward only;    # Do NOT attempt iterative recursion if forwarders fail

    # DNSSEC: keep BIND signing-aware for local zones.
    # Validation is disabled because forward only prevents BIND from
    # fetching the DNSSEC chain from root; 8.8.8.8 handles that upstream.
    dnssec-enable yes;
    dnssec-validation no;

    # Security: Root Key Management
    # Path to the built-in ISC root trust anchor (installed with bind package).
    bindkeys-file "/etc/named.root.key";

    # Directory where BIND stores managed keys for automated trust-anchor
    # updates (managed-keys / RFC 5011).
    managed-keys-directory "/var/named/dynamic";

    # Runtime lock and session files
    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
};
```

### `/etc/named/named.conf.local`

```bash
sudo vi /etc/named/named.conf.local
```

```text
#
# RHEL BIND: Local Zone Definitions and Overrides
# Requirements covered:
#   - Forward Lookup Zone: nhr.local (A, MX, CNAME, NS, SOA records)
#   - Reverse Lookup Zone: 1.168.192.in-addr.arpa (PTR records)
#   - DNSSEC: inline-signing, auto-dnssec maintain, key-directory
#   - DDNS / DHCP integration: allow-update with TSIG key (optional)
#   - Security: restrict zone transfers to authorised secondary servers
#

#============================================================================
# FORWARD ZONE: nhr.local
# Maps hostnames to IP addresses and provides mail routing.
# Records (defined in /var/named/dynamic/nhr.local.zone):
#   SOA, NS, A (ns1, client, www, mail), CNAME (ftp), MX
#============================================================================
zone "nhr.local" IN {
    type master;                    # This server is authoritative (primary)

    # Zone file stored under /var/named/dynamic/ so that BIND's in-line
    # signing can write the signed copy (.zone.signed) and journal files.
    # The directory is chown'd to root:named with 770 perms.
    file "dynamic/nhr.local.zone";

    # DNSSEC Zone Signing: Key Directory
    # Points to the directory holding the ZSK and KSK generated by
    # dnssec-keygen. Required for inline-signing to work.
    key-directory "/var/named/keys";

    # DNSSEC Zone Signing: Enable Automatic Signing
    # "inline-signing yes"   → BIND signs the zone in memory on reload
    # "auto-dnssec maintain" → BIND manages key rollover automatically
    inline-signing yes;
    auto-dnssec maintain;

    # Security: Dynamic Updates (DDNS / DHCP integration)
    # Set to "none" for a static server. To allow a DHCP server to
    # automatically update DNS records, replace with the TSIG key name
    # generated in section 1.16 (e.g., allow-update { key dhcp-key; };).
    allow-update { none; };
};

#
# OPTIONAL: Per-Zone Zone Transfer Override
# If a secondary nameserver (e.g., 192.168.1.15) is required, define the
# ACL in named.conf.options and uncomment the line below to permit AXFR.
# allow-transfer { secondary-dns; };

#============================================================================
# REVERSE ZONE: 1.168.192.in-addr.arpa
# Maps IP addresses to hostnames (PTR records) for reverse DNS lookups.
# This is used by services such as SMTP, SSH, and FTP for logging/verification.
# Records (defined in /var/named/dynamic/nhr.local.rev):
#   SOA, NS, PTR (192.168.1.10 → ns1, .20 → client, .30 → mail)
#============================================================================
zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "dynamic/nhr.local.rev";
    key-directory "/var/named/keys";
    inline-signing yes;
    auto-dnssec maintain;
    allow-update { none; };
};

#
# OPTIONAL: Additional ACL or TSIG key blocks can be defined here (top level,
# outside of any zone {}) for DDNS/DHCP integration, e.g.:
#
# key "dhcp-key" {
#     algorithm hmac-md5;
#     secret "base64-encoded-secret-here==";
# };
#
```

> [!IMPORTANT]
> The zone files are configured to reside in `/var/named/dynamic/`. On RHEL, this directory has write permissions pre-allocated for the `named` user, which is required for BIND to dynamically generate signed zone copies and journals.

---

## 1.5 Forward Zone File: `/var/named/dynamic/nhr.local.zone`

```bash
sudo vi /var/named/dynamic/nhr.local.zone
```

```text
;
; RHEL BIND: Forward Lookup Zone File
; File: /var/named/dynamic/nhr.local.zone
; Zone: nhr.local
; Requirements covered:
;   - Forward Lookup Zone: A records, CNAME records, MX records
;   - Record types: SOA, NS, A, CNAME, MX
;

$TTL 86400
@   IN  SOA  ns1.nhr.local. admin.nhr.local. (
                2026062401  ; Serial   (YYYYMMDDnn)
                3600        ; Refresh
                1800        ; Retry
                604800      ; Expire
                86400 )     ; Minimum TTL

; NS: authoritative nameservers for this zone
@       IN  NS   ns1.nhr.local.

; A: hostname to IPv4 address mappings
ns1     IN  A    192.168.1.10
client  IN  A    192.168.1.20
www     IN  A    192.168.1.10
mail    IN  A    192.168.1.30

; CNAME: alias (ftp resolves to the same address as www)
ftp     IN  CNAME www.nhr.local.

; MX: mail exchangers for the domain (priority 10)
@       IN  MX   10  mail.nhr.local.
```

### Serial Number Convention

The serial field (`2026062401`) follows the `YYYYMMDDnn` format. Increment it whenever a zone file is modified; secondary (slave) nameservers use the new value to detect changes and trigger zone transfers.

---

## 1.6 Reverse Zone File: `/var/named/dynamic/nhr.local.rev`

```bash
sudo vi /var/named/dynamic/nhr.local.rev
```

```text
;
; RHEL BIND: Reverse Lookup Zone File
; File: /var/named/dynamic/nhr.local.rev
; Zone: 1.168.192.in-addr.arpa
; Requirements covered:
;   - Reverse Lookup Zone: PTR records for 192.168.1.0/24
;

$TTL 86400
@   IN  SOA  ns1.nhr.local. admin.nhr.local. (
                2026062401  ; Serial
                3600        ; Refresh
                1800        ; Retry
                604800      ; Expire
                86400 )     ; Minimum TTL

; NS: authoritative nameservers for this zone
@       IN  NS   ns1.nhr.local.

; PTR: last octet of IPv4 address mapped to FQDN
10      IN  PTR  ns1.nhr.local.
20      IN  PTR  client.nhr.local.
30      IN  PTR  mail.nhr.local.
```

---

## 1.7 File Permissions and SELinux Context

The `named` daemon runs as the restricted `named` user. Configuration files, zone files, and log directories must all carry the correct POSIX ownership and SELinux security context so that BIND can read them and, where required, write dynamic updates.

### Ownership and POSIX Permissions

> **Prerequisite before restarting named:** Create the log directory with correct ownership. The `logging` block in `/etc/named.conf` references `/var/log/named/named.log`, which must exist before starting the service.

```bash
# Create log directory BEFORE restarting named (required for logging)
sudo mkdir -p /var/log/named
sudo chown named:named /var/log/named
sudo chmod 750 /var/log/named

# Set ownership for BIND configuration and zone files
# The named group grants the daemon read access; world is excluded.
sudo chown root:named /etc/named.conf
sudo chown -R root:named /var/named/dynamic/
sudo chmod 770 /var/named/dynamic

# Restrict file permissions: group-readable, world-inaccessible
sudo chmod 640 /etc/named.conf
sudo chmod 640 /var/named/dynamic/nhr.local.zone
sudo chmod 640 /var/named/dynamic/nhr.local.rev
```

### SELinux Context Configuration

RHEL 7 requires specific SELinux labels for DNS assets to prevent unauthorized process access.

| Asset | Required Context | Reason |
| --- | --- | --- |
| `/etc/named.conf` | `named_conf_t` | Readable by the named daemon |
| `/var/named/dynamic/` | `named_cache_t` | Writable by BIND for inline-signed zone copies and journal files |
| `/var/log/named/` | `named_log_t` | Writable by BIND for query and event logs |

```bash
# Apply default SELinux context to the main config file
sudo restorecon -v /etc/named.conf

# Apply named_cache_t context to the dynamic zone directory
# This allows BIND to write signed zone copies and journal files.
sudo semanage fcontext -a -t named_cache_t "/var/named/dynamic(/.*)?"
sudo restorecon -Rv /var/named/dynamic/

# Apply SELinux context for the log directory (created above)
sudo semanage fcontext -a -t named_log_t "/var/log/named(/.*)?"
sudo restorecon -Rv /var/log/named
```

---

## 1.8 Syntax Validation and Service Restart

Always validate every BIND configuration and zone file before restarting the service. A syntax error in any file will cause the daemon to fail, creating a DNS outage.

```bash
# Check the global named.conf syntax (all included files are parsed)
sudo named-checkconf /etc/named.conf

# Validate the forward zone file
sudo named-checkzone nhr.local /var/named/dynamic/nhr.local.zone

# Validate the reverse zone file
sudo named-checkzone 1.168.192.in-addr.arpa /var/named/dynamic/nhr.local.rev

# Start / restart the named service to apply all changes
sudo systemctl restart named
```

A clean `named-checkzone` output reports `OK` and prints the zone serial number.

---

## 1.9 Firewall Configuration

DNS uses TCP and UDP port 53. This rule must be added permanently so it survives firewall reloads.

```bash
# Permanently allow DNS service (TCP/53 and UDP/53) through firewalld
sudo firewall-cmd --permanent --add-service=dns

# Reload firewall rules to apply the change immediately
sudo firewall-cmd --reload

# Confirm the dns service is listed in the active zone
sudo firewall-cmd --list-services
```

---

## 1.10 Client Configuration

Configure each client machine to send DNS queries to the RHEL BIND server instead of an external resolver.

```bash
# Point the client resolver to the internal BIND server
# Performed on the CLIENT machine (192.168.1.20)
sudo vi /etc/resolv.conf
```

```text
#
# Client resolver configuration: /etc/resolv.conf
# Points to ns1.nhr.local (192.168.1.10) for all queries
#
search nhr.local
nameserver 192.168.1.10
```

---

## 1.11 Verification

Run name resolution tests from the client machine using `dig` and `nslookup`. All queries should be answered by the local BIND server (`192.168.1.10`).

```bash
# Forward lookup (hostname → IP) using nslookup
nslookup ns1.nhr.local

# Forward lookup (hostname → IP) using dig
dig ns1.nhr.local

# Reverse lookup (IP → hostname) using dig
dig -x 192.168.1.10

# Query the MX (mail exchanger) record for the domain
dig MX nhr.local
```

```
❯ dig MX nhr.local @192.168.1.10

; <<>> DiG 9.20.24 <<>> MX nhr.local @192.168.1.10
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 57295
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;nhr.local.			IN	MX

;; ANSWER SECTION:
nhr.local.		86400	IN	MX	10 mail.nhr.local.

;; AUTHORITY SECTION:
nhr.local.		86400	IN	NS	ns1.nhr.local.

;; ADDITIONAL SECTION:
mail.nhr.local.		86400	IN	A	192.168.1.30
ns1.nhr.local.		86400	IN	A	192.168.1.10

;; Query time: 1 msec
;; SERVER: 192.168.1.10#53(192.168.1.10) (UDP)
;; WHEN: Sat Jul 04 00:05:35 +06 2026
;; MSG SIZE  rcvd: 109
```

---

## 1.12 Restricting Zone Transfers and Access Controls

By default, an authoritative BIND server should refuse all AXFR (full zone transfer) requests. Allowing unrestricted zone transfers would expose the entire corporate DNS namespace to external enumeration.

### Restricting Zone Transfers Globally

In `/etc/named/named.conf.options`, the global default is to allow no transfers:

```text
#
# Global restriction: no zone transfers allowed by default
# Override per-zone with an ACL if a slave/secondary server is required.
#
options {
    ...
    allow-transfer { none; };
};
```

### Allowing Authorized Transfers

If a secondary nameserver (e.g., `192.168.1.15`) is later added, define an ACL and override the global restriction for specific zones:

```text
# In /etc/named/named.conf.options (define the ACL once at top level)
acl secondary-dns {
    192.168.1.15;
};

# In /etc/named/named.conf.local (override per zone)
zone "nhr.local" IN {
    type master;
    file "dynamic/nhr.local.zone";
    allow-transfer { secondary-dns; };
};
```

---

## 1.13 Upstream Forwarders for External Resolution

Internal clients need to resolve public names (e.g., `www.google.com`). Instead of querying the Internet root servers directly, BIND forwards these requests to faster upstream resolvers; here, Google Public DNS.

This is configured in the global `options` block of `/etc/named/named.conf.options`:

```text
#
# Forwarding: upstream public resolvers for external queries
# Recursion is permitted so internal clients can resolve any public name.
# "forward only" prevents iterative fallback if the forwarders are unreachable.
#
options {
    ...
    recursion yes;

    forwarders {
        8.8.8.8;      # Google Public DNS: primary
        8.8.4.4;      # Google Public DNS: secondary
    };
    forward only;
};
```

> **Note: SERVFAIL on external queries:** Using `forward only` with `dnssec-validation yes` will break external resolution. BIND tries to validate the answer itself but can't fetch the DNSSEC chain from root servers because all queries are forwarded. Set `dnssec-validation no;` because Google's resolver already validates DNSSEC upstream. Local zone signing on `nhr.local` still works fine.

---

## 1.14 DNSSEC Zone Signing Setup

DNSSEC adds cryptographic signatures to DNS records to protect clients from DNS spoofing and cache poisoning. BIND signs zones automatically when `inline-signing` and `auto-dnssec` are enabled.

### Step 1: Create Key Directory and Set Permissions

```bash
# Create a dedicated directory for DNSSEC keys (separate from zones)
sudo mkdir -p /var/named/keys
sudo chown root:named /var/named/keys
sudo chmod 750 /var/named/keys

# Apply SELinux context so BIND can read the keys
sudo semanage fcontext -a -t named_zone_t "/var/named/keys(/.*)?"
sudo restorecon -Rv /var/named/keys
```

### Step 2: Generate Cryptographic Keys

Generate the **Zone Signing Key (ZSK)** and **Key Signing Key (KSK)**. Use `-r /dev/urandom` to avoid entropy-blocking on headless servers.

```bash
# Navigate to the keys directory
cd /var/named/keys

# Generate Zone Signing Key (ZSK): used for signing individual RRsets
# Use -r /dev/urandom if the command hangs waiting for entropy
sudo dnssec-keygen -r /dev/urandom -a RSASHA256 -b 2048 -n ZONE nhr.local

# Generate Key Signing Key (KSK): used for signing the DNSKEY RRset
sudo dnssec-keygen -r /dev/urandom -f KSK -a RSASHA256 -b 4096 -n ZONE nhr.local
```

Both commands generate `.key` and `.private` files. Secure them:

```bash
# Restrict ownership and permissions on the key files
sudo chown root:named /var/named/keys/K*
sudo chmod 640 /var/named/keys/K*
```

### Step 3: Configure Zone for Auto-Signing

The zone blocks in `/etc/named/named.conf.local` already reference the key directory and enable automatic signing:

```text
#
# Zone auto-signing: BIND manages key rollover and signs
# the zone on reload. No manual "rndc sign" is needed.
#
zone "nhr.local" IN {
    type master;
    file "dynamic/nhr.local.zone";
    key-directory "/var/named/keys";
    inline-signing yes;
    auto-dnssec maintain;
    allow-update { none; };
};

# (Apply the same `key-directory`, `inline-signing`, and `auto-dnssec` settings to the reverse zone block.)
```

### Step 4: Reload BIND and Verify DNSSEC

```bash
# Reload BIND to initiate zone signing
sudo rndc reload
```

BIND creates signed zone files (e.g., `nhr.local.zone.signed`) and journal files under `/var/named/dynamic/`.

Verify DNSSEC is active:

```bash
# Query for the DNSKEY record (should return RRSIG)
dig @192.168.1.10 nhr.local DNSKEY +dnssec

# Verify a signed A record (look for RRSIG)
dig @192.168.1.10 ns1.nhr.local A +dnssec
```

```
❯ dig @192.168.1.10 ns1.nhr.local A +dnssec

; <<>> DiG 9.20.24 <<>> @192.168.1.10 ns1.nhr.local A +dnssec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 20584
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 2, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags: do; udp: 4096
;; QUESTION SECTION:
;ns1.nhr.local.			IN	A

;; ANSWER SECTION:
ns1.nhr.local.		86400	IN	A	192.168.1.10
ns1.nhr.local.		86400	IN	RRSIG	A 8 3 86400 20260802172742 20260703164449 57246 nhr.local. pNSEg4J5...

;; AUTHORITY SECTION:
nhr.local.		86400	IN	NS	ns1.nhr.local.
nhr.local.		86400	IN	RRSIG	NS 8 2 86400 20260802165459 20260703164449 57246 nhr.local. f2X6sHz5...

;; Query time: 1 msec
;; SERVER: 192.168.1.10#53(192.168.1.10) (UDP)
;; WHEN: Sat Jul 04 00:04:58 +06 2026
;; MSG SIZE  rcvd: 666
```

Look for `RRSIG` records in the output. The `ad` (Authentic Data) flag appears only when querying **through a validating resolver** (e.g., Unbound), not the authoritative server directly. For testing validation, configure Unbound as a forwarder and query it instead.

> **Note:** The authoritative BIND server returns signed records (`RRSIG` present) but does not set `ad` since it is the source of truth, not a validator.

---

## 1.15 Custom Logging Configuration

The logging block is defined in `/etc/named.conf` (section 1.4). After BIND is running, verify logs are being written correctly:

```bash
# Confirm logs are being written
sudo systemctl status named
tail -f /var/log/named/named.log
```

---

## 1.16 DHCP Integration (Dynamic DNS - DDNS)

Dynamic DNS lets a DHCP server automatically update BIND's forward and reverse records whenever a device obtains a lease. A TSIG (Transaction Signature) key authenticates these updates so only the authorized DHCP daemon can modify DNS data.

### Step 1: Generate TSIG Key for Authentication

```bash
# Generate a shared HMAC-MD5 key for DHCP-to-BIND updates
# The key name "dhcp-key" must match on both sides.
dnssec-keygen -a HMAC-MD5 -b 128 -n USER dhcp-key
```

View the generated `.key` file to extract the base-64 secret.

### Step 2: Configure BIND with the TSIG Key

In `/etc/named/named.conf.local`, define the key once at the top level, then allow the relevant zones to accept updates using that key:

```text
#
# TSIG key block: shared secret between DHCP and BIND
# Place this at the top level (outside any zone { } block)
# The secret value comes from the .key file generated by dnssec-keygen
#
key "dhcp-key" {
    algorithm hmac-md5;
    secret "<extract-from-dhcp-key.+157.+...";
};

# Forward zone: permit updates authenticated by dhcp-key
zone "nhr.local" IN {
    type master;
    file "dynamic/nhr.local.zone";
    allow-update { key dhcp-key; };
};

# Reverse zone: permit updates authenticated by dhcp-key
zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "dynamic/nhr.local.rev";
    allow-update { key dhcp-key; };
};
```

### Step 3: Configure DHCP Server (`/etc/dhcp/dhcpd.conf`)

Configure the DHCP service to push updates to the BIND server whenever leases change:

```text
#
# /etc/dhcp/dhcpd.conf: DDNS client configuration
# References the same TSIG key defined in named.conf.local
#

ddns-update-style standard;
update-static-leases on;
authoritative;

# Shared TSIG key: must match the BIND key block exactly
# Use the secret value from the generated .key file (found in dhcp-key.+157.+...)
key "dhcp-key" {
    algorithm hmac-md5;
    secret "<extract-from-dhcp-key.+157.+...";
}

# Forward zone update forwarding
zone nhr.local. {
    primary 192.168.1.10;
    key dhcp-key;
}

# Reverse zone update forwarding
zone 1.168.192.in-addr.arpa. {
    primary 192.168.1.10;
    key dhcp-key;
}

# Subnet: DHCP clients receive the BIND server as their DNS resolver
subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.50 192.168.1.100;
    option routers 192.168.1.1;
    option domain-name-servers 192.168.1.10;
    option domain-name "nhr.local";
}
```

---

## 1.17 Automated Backup and Disaster Recovery

A backup strategy ensures fast recovery after hardware failure or human error. Back up the three BIND config files, DNSSEC keys, and zone databases under `/var/named/`.

### Step 1: Create Backup Script

```bash
# Create the automated backup script
sudo tee /usr/local/sbin/dns-backup.sh > /dev/null << 'EOF'
#!/bin/bash
# RHEL BIND: Automated Backup Script
# Archives config, keys, and zone files to /backup/dns/

BACKUP_DIR="/backup/dns"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/bind_backup_${DATE}.tar.gz"

mkdir -p "${BACKUP_DIR}"
chmod 700 "${BACKUP_DIR}"

# Archive all BIND split-config files, DNSSEC keys, and zone databases
tar -czf "${BACKUP_FILE}" \
    /etc/named.conf \
    /etc/named/named.conf.options \
    /etc/named/named.conf.local \
    /var/named/keys \
    /var/named/dynamic

# Prune archives older than 30 days to conserve disk space
find "${BACKUP_DIR}" -type f -name "bind_backup_*.tar.gz" -mtime +30 -delete
EOF

# Make the script executable
sudo chmod 700 /usr/local/sbin/dns-backup.sh
```

### Step 2: Configure Cron Automation

```text
# Run backup every Sunday at 02:00 AM (add to root's crontab)
0 2 * * 7 /usr/local/sbin/dns-backup.sh >/dev/null 2>&1
```

---

## 1.18 Server Monitoring and Alerts (Nagios and Zabbix)

Monitoring ensures the DNS server is reachable and resolving queries correctly. The BIND `statistics-channels` directive (configured in `/etc/named.conf`, section 1.4) exposes query counters on `127.0.0.1:8053`. External monitoring tools consume this data.

### Monitoring BIND with Nagios

Nagios is a centralized monitoring server that runs plugins against remote hosts. It uses the `check_dns` plugin to validate latency and resolution correctness.

**Prerequisites:**
- A Nagios server installed and running (separate from the DNS server)
- Network access from the Nagios server to `192.168.1.10` (UDP/TCP 53)

#### Step 1: Install the DNS Check Plugin

On the **Nagios server**:

```bash
# Install the Nagios plugins package (includes check_dns)
sudo yum install -y nagios-plugins-all
```

#### Step 2: Define the DNS Server Host

Create `/etc/nagios/objects/hosts/ns1.nhr.local.cfg` on the Nagios server:

```text
define host {
    use                     linux-server
    host_name               ns1.nhr.local
    alias                   BIND DNS Server
    address                 192.168.1.10
    max_check_attempts      5
    check_period            24x7
    notification_interval   30
    notification_period     24x7
}
```

#### Step 3: Define the DNS Service Check

Create `/etc/nagios/objects/services/dns-resolution.cfg` on the Nagios server:

```text
define service {
    use                     generic-service
    host_name               ns1.nhr.local
    service_description     DNS Resolution Check
    check_command           check_dns!ns1.nhr.local!-s 192.168.1.10!-a 192.168.1.10
    check_interval          5
    retry_interval          1
    max_check_attempts      3
    notification_period     24x7
}
```

#### Step 4: Verify the Check Manually

From the Nagios server, test the plugin directly:

```bash
# Test DNS resolution against the BIND server
/usr/lib64/nagios/plugins/check_dns -H ns1.nhr.local -s 192.168.1.10 -a 192.168.1.10
```

Expected output: `DNS OK: 0.001 seconds response time. ns1.nhr.local returns 192.168.1.10`

#### Step 5: Restart Nagios

```bash
# Verify configuration and restart Nagios
sudo nagios -v /etc/nagios/nagios.cfg
sudo systemctl restart nagios
```

---

### Monitoring BIND with Zabbix

Zabbix is an enterprise-grade monitoring platform that uses agents installed on target hosts. It uses the BIND statistics channel to produce real-time graphs of query rates, cache hits, and socket errors.

**Prerequisites:**
- A Zabbix server running (separate monitoring server)
- The Zabbix agent will be installed on the DNS server (`192.168.1.10`)

#### Step 1: Install and Configure the Zabbix Agent

On the **DNS server** (`192.168.1.10`):

```bash
# Install the Zabbix agent
sudo yum install -y zabbix-agent

# Enable and start the agent
sudo systemctl enable zabbix-agent
sudo systemctl start zabbix-agent
```

#### Step 2: Create a Custom BIND Monitoring Parameter

On the **DNS server**, create `/etc/zabbix/zabbix_agentd.d/userparameter_bind.conf`:

```text
# BIND query counter via statistics channel
UserParameter=dns.queries,curl -s http://127.0.0.1:8053/xml/v3/status | xmlstarlet sel -t -v "//counters[@type='opcode']/counter[@name='QUERY']"
```

Restart the agent to load the new parameter:

```bash
sudo systemctl restart zabbix-agent
```

#### Step 3: Test the Custom Parameter

On the **DNS server**, test that Zabbix can read the metric:

```bash
# Query the local Zabbix agent
zabbix_get -s 127.0.0.1 -k dns.queries
```

Expected output: A numeric value representing the total query count.

#### Step 4: Add the Host in the Zabbix Web Portal

1. Log in to the Zabbix web interface.
2. Go to **Configuration → Hosts → Create host**.
3. Set **Host name** to `ns1.nhr.local` and **Agent interfaces** to `192.168.1.10:10050`.
4. Link the host to the **Domain Name Service (BIND)** template (or a built-in DNS template).
5. Save the host.

---

### Validation Summary

| Tool | Quick Validation Command |
| --- | --- |
| **Nagios** | `/usr/lib64/nagios/plugins/check_dns -H ns1.nhr.local -s 192.168.1.10 -a 192.168.1.10` |
| **Zabbix** | `zabbix_get -s 127.0.0.1 -k dns.queries` (run on DNS server) |

---

## 1.19 Regular Maintenance and Security Patching

DNS is critical infrastructure. Schedule regular OS updates and DNSSEC key rollovers to protect BIND against emerging vulnerabilities.

### System and BIND Patching

Regularly pull security updates for BIND from RHN or a local Satellite server:

```bash
# Check for BIND package updates
sudo yum check-update bind bind-utils

# Apply security patches to BIND
sudo yum update -y bind bind-utils

# Restart named safely after a package upgrade
sudo systemctl restart named
```

### DNSSEC Key Rollover Procedures

Maintain cryptographic strength by rotating keys on a schedule:

| Key Type | Rollover Interval |
| --- | --- |
| Zone Signing Key (ZSK) | Every 30–90 days |
| Key Signing Key (KSK) | Every 1–2 years |

When `auto-dnssec maintain` and `inline-signing` are enabled, BIND automatically rotates keys when new ones are added.

```bash
# Manual key rollover: generate a new ZSK
sudo dnssec-keygen -a RSASHA256 -b 2048 -n ZONE nhr.local

# Load the new keys and re-sign the zone dynamically
sudo rndc loadkeys nhr.local

# Confirm signatures are current
sudo rndc signing -list nhr.local
```

---

# 2. RHEL 7 Caching DNS Server (Unbound)

**Unbound** is a validating, recursive, and caching DNS resolver developed by NLnet Labs. Unlike BIND in authoritative mode, Unbound does **not** host zone files. Instead, it accepts client queries, recursively resolves them against upstream root/authoritative servers, caches the results for the duration of each record's TTL, and returns the final answer to the client. Because resolved answers are cached, repeated queries are served locally without hitting upstream servers again, which reduces latency noticeably.

---

## 2.1 Conceptual Foundations

### How a Caching Resolver Works

```
Client  ──►  Unbound (Cache)  ──►  Root Servers
                │                      │
                │◄── Cached Answer      ▼
                │               TLD Nameservers
                │                      │
                └──────────────────────►
                                Authoritative NS
```

1. The client sends a query (e.g., `www.google.com`) to Unbound.
2. Unbound checks its **cache**; if a valid cached answer exists (TTL > 0), it responds immediately.
3. On a **cache miss**, Unbound performs full recursive resolution: Root → TLD → Authoritative.
4. The resolved answer is **stored in cache** and returned to the client.
5. Subsequent identical queries are served from cache until the TTL expires.

### DNSSEC Validation

Unbound performs **DNSSEC** (DNS Security Extensions) validation by default, cryptographically verifying the authenticity and integrity of DNS responses. Without this, a resolver is vulnerable to DNS spoofing and cache poisoning attacks.

---

## 2.2 Lab Environment

| Role | Hostname | IP Address |
| --- | --- | --- |
| Caching DNS Server (Unbound) | `cache1.nhr.local` | `192.168.1.11` |
| Client Machine | `client.nhr.local` | `192.168.1.20` |

---

## 2.3 Installation

```bash
# Install the Unbound validating recursive resolver package
sudo yum install -y unbound

# Enable and start the Unbound service
sudo systemctl enable unbound
sudo systemctl start unbound

# Verify the service is active and listening
sudo systemctl status unbound
```

---

## 2.4 Configuration: `/etc/unbound/unbound.conf`

The main Unbound configuration file is `/etc/unbound/unbound.conf`. Edit it to define the listening interface, access control, and forwarding behavior.

```bash
sudo vi /etc/unbound/unbound.conf
```

Modify or add the following directives inside the `server:` clause:

```text
#
# RHEL Unbound: Caching DNS Resolver Configuration
# File: /etc/unbound/unbound.conf
# Server: cache1.nhr.local (192.168.1.11)
# Requirements covered:
#   - Caching resolver: recursive lookups with TTL-based caching
#   - DNSSEC: default validation with auto-managed trust anchors
#   - Access controls: restrict queries to localhost and internal subnet
#   - Root hints and prefetch for low-latency resolution
#

server:
    # Server Setup: Listen on all interfaces (0.0.0.0)
    interface: 0.0.0.0

    # Security: Access Controls
    # Allow queries only from localhost (127.0.0.0/8) and the internal
    # subnet (192.168.1.0/24). Refuse all other sources.
    access-control: 127.0.0.0/8 allow
    access-control: 192.168.1.0/24 allow
    access-control: 0.0.0.0/0 refuse

    # Security: Hide server identity and version from clients
    hide-identity: yes
    hide-version: yes

    # Server Setup: Protocol and daemon behaviour
    do-daemonize: yes
    do-ip4: yes
    do-ip6: no

    # Performance: Prefetch popular records before TTL expiry
    prefetch: yes

    # Logging: Verbosity levels (0=minimal, 1=operational, 2=detail)
    verbosity: 1

    # Root Hints: file containing the 13 DNS root server IPs
    # On RHEL 7 this is installed by the bind-libs package.
    root-hints: "/var/named/named.ca"

    # DNSSEC: auto-managed root trust anchor (RFC 5011)
    auto-trust-anchor-file: "/var/lib/unbound/root.key"
```

> [!NOTE]
> The `root-hints` file (`named.ca`) contains the IP addresses of the 13 DNS root server clusters. On RHEL 7, it is installed by the `bind-libs` package at `/var/named/named.ca`. Verify its presence with `ls -l /var/named/named.ca`.

---

## 2.5 Optional: Forward Specific Zones to BIND

If you have an internal authoritative BIND server (e.g., for `nhr.local`), configure Unbound to forward those queries directly instead of recursing to the Internet:

```bash
sudo vi /etc/unbound/unbound.conf
```

Append the following **below** the `server:` block:

```text
# Forward internal zone queries directly to the local BIND server
# instead of recursing to the public Internet (split-horizon setup).
# Place these blocks BELOW the "server:" section in unbound.conf.

forward-zone:
        name: "nhr.local"
        forward-addr: 192.168.1.10

forward-zone:
        name: "1.168.192.in-addr.arpa"
        forward-addr: 192.168.1.10
```

This creates a split-horizon architecture: internal names are resolved by BIND, while all other queries are handled recursively by Unbound.

---

## 2.6 File Permissions and SELinux Context

Unbound configuration files must be owned by the `unbound` group and possess the correct SELinux security context so that the `unbound` daemon can read them.

### Ownership and Standard POSIX Permissions

```bash
# Set group ownership so the unbound daemon can read configs
sudo chown -R root:unbound /etc/unbound/

# Restrict permissions: only root can write, unbound group can read
sudo chmod 640 /etc/unbound/unbound.conf
```

### SELinux Context Configuration

On RHEL 7, `/etc/unbound/` files must be labeled `named_conf_t` (shared across DNS daemons by policy).

```bash
# Check current SELinux context
ls -Z /etc/unbound/unbound.conf

# Restore the default context recursively
sudo restorecon -Rv /etc/unbound/

# (Optional) If using a custom config path, apply the policy label
sudo semanage fcontext -a -t named_conf_t "/custom/unbound(/.*)?"
sudo restorecon -Rv /custom/unbound/
```

---

## 2.7 Syntax Validation and Service Restart

```bash
# Validate the configuration file syntax before restart
sudo unbound-checkconf /etc/unbound/unbound.conf

# Restart Unbound to apply changes
sudo systemctl restart unbound

# Confirm the service is running without errors
sudo systemctl status unbound
```

A successful `unbound-checkconf` run outputs:

```text
unbound-checkconf: no errors in /etc/unbound/unbound.conf
```

---

## 2.8 Firewall Configuration

```bash
# Allow DNS service (TCP/53 and UDP/53) through firewalld
sudo firewall-cmd --permanent --add-service=dns

# Reload firewall rules to apply immediately
sudo firewall-cmd --reload

# Verify the service is listed in the active zone
sudo firewall-cmd --list-services
```

---

## 2.9 Client Configuration

Point the client's resolver to the Unbound caching server:

```bash
# Point the client resolver to the Unbound caching server
# Performed on the CLIENT machine (192.168.1.20)
sudo vi /etc/resolv.conf
```

```text
#
# Client resolver configuration: /etc/resolv.conf
# Points to cache1.nhr.local (192.168.1.11) for all queries
#
search nhr.local
nameserver 192.168.1.11
```

---

## 2.10 Verification

```bash
# Basic forward resolution through Unbound (client query)
nslookup www.google.com

# Explicit dig query via the Unbound server (192.168.1.11)
dig @192.168.1.11 www.google.com

# Verify DNSSEC validation (look for the 'ad' flag in the response)
dig @192.168.1.11 google.com +dnssec

# Query an internal name via the forward-zone rule to BIND
dig @192.168.1.11 ns1.nhr.local
```

### Interpreting `dig` DNSSEC Output

```text
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
```

| Flag | Meaning |
| --- | --- |
| `qr` | Query Response |
| `rd` | Recursion Desired (set by client) |
| `ra` | Recursion Available (set by server) |
| `ad` | **Authentic Data** (DNSSEC validation passed) |

> [!IMPORTANT]
> The presence of the `ad` (Authentic Data) flag in the `dig` response confirms that Unbound has successfully validated the DNSSEC chain of trust for the answer.

---

## 2.11 Cache Inspection and Management

```bash
# Dump all cached entries to stdout (from the Unbound server)
sudo unbound-control dump_cache

# Flush a specific record from the cache
sudo unbound-control flush www.google.com

# Flush the entire cache
sudo unbound-control flush_zone .

# View live statistics without resetting counters
sudo unbound-control stats_noreset | head -20
```

> [!TIP]
> `unbound-control stats_noreset` reports live statistics without resetting counters. Key metrics to observe are `total.num.cachehits` and `total.num.cachemiss`; a high cache-hit ratio indicates the server is working effectively.

---

## 2.12 Comparison: BIND (Authoritative) vs. Unbound (Caching)

| Feature              | BIND (`named`)                | Unbound                               |
| -------------------- | ----------------------------- | ------------------------------------- |
| **Primary Role**     | Authoritative name server     | Recursive caching resolver            |
| **Holds Zone Files** | Yes (`/var/named/*.zone`)     | No                                    |
| **Recursion**        | Disabled (`recursion no`)     | Enabled (core function)               |
| **DNSSEC**           | Optional (complex setup)      | Built-in, enabled by default          |
| **Caching**          | Limited internal cache        | Full-featured, tunable cache          |
| **Configuration**    | `/etc/named.conf`             | `/etc/unbound/unbound.conf`           |
| **Service Name**     | `named`                       | `unbound`                             |
| **Typical Use Case** | Hosting `example.com` records | Internal network resolver / forwarder |
| **Package**          | `bind`                        | `unbound`                             |
