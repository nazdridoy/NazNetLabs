# RHEL 7 Mail Server Lab: Postfix and Dovecot

This lab covers setting up a complete internal mail server on Red Hat Enterprise Linux 7 using **Postfix** as the Mail Transfer Agent (MTA) and **Dovecot** as the IMAP/POP3 server. The setup includes TLS encryption, SASL authentication, SpamAssassin filtering, and a Thunderbird mail client walkthrough.

---

## 1. Server Setup

This lab provisions a dedicated RHEL 7 server to host internal mail services. Before starting the installation, verify the server meets the hardware requirements for handling concurrent SMTP, IMAP, and POP3 connections.

### Lab Environment Topology

| Role              | Hostname           | IP Address     |
| ----------------- | ------------------ | -------------- |
| Mail Server       | `mail.nhr.local`   | `192.168.1.30` |
| DNS Server (BIND) | `ns1.nhr.local`    | `192.168.1.10` |
| Client Machine    | `client.nhr.local` | `192.168.1.20` |

### Server Hardware Sizing Requirements

The following hardware specifications cover a medium-sized company with up to 500 concurrent mailboxes and typical SMTP/IMAP loads:

| Resource    | Minimum Specification | Recommended Sizing     | Purpose / Notes                                                                                    |
| ----------- | --------------------- | ---------------------- | -------------------------------------------------------------------------------------------------- |
| **CPU**     | 2 vCPUs / Cores       | 4 vCPUs / Cores        | Postfix and Dovecot are multi-threaded; extra cores handle concurrent TLS handshakes and delivery. |
| **RAM**     | 2 GB                  | 4 GB - 8 GB            | Dovecot keeps mailbox indexes in memory for fast IMAP responses; more RAM means faster access.     |
| **Disk**    | 50 GB                 | 200 GB (SSD preferred) | Mail spool (`/var/mail/`), Maildir storage, log files, and scheduled backups all consume disk.     |
| **Network** | 100 Mbps              | 1 Gbps                 | Low-latency network adapter for responsive email transmission and reception.                       |

> [!NOTE]
> All commands on the server are executed as `root` or with `sudo`. The domain used throughout this lab is `nhr.local`. The mail server hostname is `mail.nhr.local` with IP `192.168.1.30`.

---

## 2. Choose Mail Server Software

Two open-source packages build the complete mail service stack:

| Component    | Software    | Role                                                                                    |
| ------------ | ----------- | --------------------------------------------------------------------------------------- |
| **MTA**      | **Postfix** | Mail Transfer Agent — handles incoming (SMTP) and outgoing (SMTP relay) mail            |
| **MDA/IMAP** | **Dovecot** | Mail Delivery Agent + IMAP/POP3 server — delivers mail to mailboxes and serves clients  |

Postfix is the standard MTA on RHEL. It has a modular security design where each component runs in a chroot jail with minimal privileges, and it has native support for SASL authentication and TLS/SSL encryption. It also integrates cleanly with SpamAssassin and Dovecot via Unix sockets.

Dovecot handles IMAP and POP3 access and scales to tens of thousands of simultaneous connections. It handles Maildir and mbox formats natively and provides the `dovecot-lda` and `LMTP` interfaces so Postfix can deliver mail directly to it. It also ships with built-in SSL/TLS support using the system OpenSSL library.

---

## 3. Installation of Mail Server Components

### Step 1: Install Postfix and Dovecot

```bash
# Install Postfix (MTA) - replaces sendmail, which may be installed by default
sudo yum install -y postfix

# Install Dovecot (MDA/IMAP/POP3 server)
sudo yum install -y dovecot

# Install SSL/TLS tools and mailx for testing
sudo yum install -y openssl mailx
```

### Step 2: Remove sendmail (if installed)

RHEL 7 ships with `sendmail` by default. Remove it before starting Postfix to avoid port conflicts on port 25:

```bash
# Stop and disable sendmail before removing
sudo systemctl stop sendmail
sudo systemctl disable sendmail
sudo yum remove -y sendmail
```

### Step 3: Enable and Start Services

```bash
# Enable Postfix to start on boot and start the service
sudo systemctl enable postfix
sudo systemctl start postfix

# Enable Dovecot to start on boot and start the service
sudo systemctl enable dovecot
sudo systemctl start dovecot

# Verify both services are running
sudo systemctl status postfix
sudo systemctl status dovecot
```

---

## 4. Configuring Postfix

Postfix configuration lives in two main files under `/etc/postfix/`:

- `/etc/postfix/main.cf` — primary configuration parameters (global settings)
- `/etc/postfix/master.cf` — defines the daemons and services Postfix runs

### 4.1 Main Configuration File: `/etc/postfix/main.cf`

```bash
sudo vi /etc/postfix/main.cf
```

The settings below cover server identity, listening interfaces, relay policy, mailbox format, SASL authentication, and TLS. Each parameter is commented to explain its purpose:

```text
#
# /etc/postfix/main.cf - Postfix Primary Configuration
# Server: mail.nhr.local (192.168.1.30)
# Domain: nhr.local
#

# -- Server Identity ----------------------------------------------------------
# myhostname: the FQDN that Postfix uses to identify itself in SMTP banners.
myhostname = mail.nhr.local

# mydomain: the base domain for this mail server.
mydomain = nhr.local

# myorigin: the domain appended to locally posted email with no @domain part.
myorigin = $mydomain

# -- Listening Interfaces -----------------------------------------------------
# inet_interfaces: network interfaces Postfix listens on.
# "all" means Postfix binds to every available IP (loopback + LAN + WAN).
inet_interfaces = all

# inet_protocols: restrict to IPv4 only (change to "all" to enable IPv6).
inet_protocols = ipv4

# -- Local Delivery Domain ----------------------------------------------------
# mydestination: list of domains for which Postfix accepts final delivery.
# Mail for these domains is delivered to local mailboxes.
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain

# -- Relay and Network --------------------------------------------------------
# mynetworks: networks permitted to relay mail through this server without
# authentication. Include localhost and the internal company subnet only.
# Do NOT include 0.0.0.0/0 - that creates an open relay.
mynetworks = 127.0.0.0/8, 192.168.1.0/24

# relayhost: route outbound Internet mail through an upstream smart host
# (e.g., your ISP's relay). Leave empty to deliver directly (requires a
# public IP with valid rDNS / SPF records).
relayhost =

# -- Mailbox Format -----------------------------------------------------------
# home_mailbox: store each user's mail in ~/Maildir/ (Maildir format).
# Maildir is preferred over mbox because each message is a separate file,
# preventing index corruption and supporting multiple concurrent IMAP clients.
home_mailbox = Maildir/

# -- SASL Authentication (for outbound relay auth) ----------------------------
# smtpd_sasl_auth_enable: allow clients to authenticate before sending mail.
smtpd_sasl_auth_enable = yes

# smtpd_sasl_type: use the Dovecot SASL back-end (avoids installing cyrus-sasl
# imapd; Dovecot is already running and exposes its auth socket).
smtpd_sasl_type = dovecot

# smtpd_sasl_path: path to the Dovecot auth socket used for SASL.
smtpd_sasl_path = private/auth

# smtpd_sasl_security_options: disallow anonymous authentication and
# plain-text passwords over unencrypted connections.
smtpd_sasl_security_options = noanonymous

# broken_sasl_auth_clients: advertise the AUTH extension in a format that
# older Microsoft Outlook versions understand.
broken_sasl_auth_clients = yes

# -- TLS Encryption -----------------------------------------------------------
# smtpd_tls_cert_file / smtpd_tls_key_file: paths to the SSL certificate and
# private key used for STARTTLS and SMTPS. Generated in step 4.2 below.
smtpd_tls_cert_file  = /etc/pki/dovecot/certs/mail.nhr.local.crt
smtpd_tls_key_file   = /etc/pki/dovecot/private/mail.nhr.local.key

# smtpd_use_tls: advertise STARTTLS capability in the SMTP banner.
smtpd_use_tls = yes

# smtpd_tls_auth_only: require SASL authentication ONLY over TLS connections.
# This prevents plain-text credentials from being sent unencrypted.
smtpd_tls_auth_only = yes

# smtpd_tls_loglevel: log TLS handshake events (1 = brief summary per session).
smtpd_tls_loglevel = 1

# smtp_tls_security_level: use opportunistic TLS for outbound delivery
# (encrypt when the remote server supports it).
smtp_tls_security_level = may

# smtp_tls_loglevel: log outbound TLS negotiation events.
smtp_tls_loglevel = 1

# -- Access and Relay Restrictions --------------------------------------------
# smtpd_recipient_restrictions: ordered list of rules applied to each RCPT TO
# command. Reject mail from unknown/unverified senders and permit_mynetworks
# so internal hosts can send without authentication.
smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination

# -- Alias Database -----------------------------------------------------------
# alias_maps: resolves local aliases (e.g., root -> admin).
alias_maps = hash:/etc/aliases

# alias_database: the database that newaliases builds from /etc/aliases.
alias_database = hash:/etc/aliases
```

### 4.2 Generate a Self-Signed SSL/TLS Certificate

For a production environment, obtain a certificate from a trusted CA such as Let's Encrypt. For an internal lab setup, a self-signed certificate is sufficient:

```bash
# Generate a 2048-bit RSA key and self-signed cert (valid 10 years)
sudo openssl req -new -x509 \
    -nodes \
    -days 3650 \
    -newkey rsa:2048 \
    -keyout /etc/pki/dovecot/private/mail.nhr.local.key \
    -out    /etc/pki/dovecot/certs/mail.nhr.local.crt \
    -subj   "/C=BD/ST=Dhaka/L=Dhaka/O=NHR Company/OU=IT/CN=mail.nhr.local"

# Lock down the private key
sudo chown root:dovecot /etc/pki/dovecot/private/mail.nhr.local.key
sudo chmod 640 /etc/pki/dovecot/private/mail.nhr.local.key
sudo chmod 644 /etc/pki/dovecot/certs/mail.nhr.local.crt
```

### 4.3 Master Service File: `/etc/postfix/master.cf`

The `master.cf` file controls which Postfix services run and on which ports. Uncomment the `smtps` and `submission` lines to enable secure SMTP on ports 465 and 587, and override the `pickup` service to prevent a SpamAssassin re-injection loop (explained in §7.2):

```bash
sudo vi /etc/postfix/master.cf
```

The following lines should be present and uncommented:

```text
#
# /etc/postfix/master.cf - Postfix Service Daemon Table (selected entries)
#

# SMTP (port 25) - standard SMTP for server-to-server mail exchange (MTA relay).
smtp      inet  n       -       n       -       -       smtpd

# Submission (port 587) - preferred port for mail clients (MUAs) to submit
# outbound mail. Requires STARTTLS and SASL authentication.
submission inet n       -       n       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject

# SMTPS (port 465) - legacy SMTPS (implicit TLS). Some older clients require
# this port instead of port 587 with STARTTLS.
smtps     inet  n       -       n       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject

# pickup - override content_filter so SpamAssassin-reinjected messages are not
# scanned a second time, which would otherwise create an infinite mail loop.
# Only SMTP inbound mail goes through SpamAssassin; locally submitted mail
# (system cron jobs, mailx) is trusted and delivered directly.
pickup    fifo  n       -       n       60      1       pickup
  -o content_filter=
  -o receive_override_options=no_header_body_checks

# SpamAssassin content filter transport (defined here, activated in §7.2)
spamassassin unix  -       n       n       -       -       pipe
  user=spamd argv=/usr/bin/spamc -f -e /usr/sbin/sendmail -oi -f ${sender} ${recipient}
```

### 4.4 Rebuild the Alias Database

`/etc/aliases` maps local system addresses to real users. Add a `root` forward so system-generated mail (cron jobs, logwatch, etc.) goes to a real mailbox:

```bash
sudo vi /etc/aliases
```

```text
# /etc/aliases — forward root's mail to the admin user
root:   nazmul
```

After saving, rebuild the binary database that Postfix reads:

```bash
# Rebuild the alias database
sudo newaliases

# Verify the entry was picked up
grep root /etc/aliases
```

### 4.5 Restart Postfix

```bash
# Check configuration for syntax errors before restarting
sudo postfix check

# reload is not enough when inet_protocols or inet_interfaces changed
sudo systemctl stop postfix
sudo systemctl start postfix

# Verify it came up cleanly
sudo systemctl status postfix
```

---

## 5. Configuring Dovecot

Dovecot handles IMAP (port 143 / 993) and POP3 (port 110 / 995) access for mail clients, and also provides the SASL authentication socket that Postfix delegates authentication to. Its configuration files are under `/etc/dovecot/`.

### 5.1 Main Configuration File: `/etc/dovecot/dovecot.conf`

```bash
sudo vi /etc/dovecot/dovecot.conf
```

Enable IMAP and POP3 protocols:

```text
#
# /etc/dovecot/dovecot.conf - Main Dovecot configuration
#

# protocols: enable IMAP and POP3 services.
# IMAP allows clients to manage mail on the server (folders, flags).
# POP3 downloads mail locally and (optionally) deletes it from the server.
protocols = imap pop3
```

### 5.2 Mail Location: `/etc/dovecot/conf.d/10-mail.conf`

```bash
sudo vi /etc/dovecot/conf.d/10-mail.conf
```

```text
#
# /etc/dovecot/conf.d/10-mail.conf - Mailbox location and format
#

# mail_location: tell Dovecot where each user's mailbox is stored.
# "maildir:~/Maildir" matches the home_mailbox = Maildir/ setting in Postfix.
mail_location = maildir:~/Maildir
```

### 5.3 Authentication: `/etc/dovecot/conf.d/10-auth.conf`

```bash
sudo vi /etc/dovecot/conf.d/10-auth.conf
```

```text
#
# /etc/dovecot/conf.d/10-auth.conf - Authentication mechanisms
#

# disable_plaintext_auth: disallow plain-text authentication unless the
# connection is encrypted with TLS. Set to "no" ONLY for testing.
disable_plaintext_auth = yes

# auth_mechanisms: authentication methods to advertise to clients.
# "plain" and "login" are widely supported. Both are safe over TLS.
auth_mechanisms = plain login
```

### 5.4 SSL/TLS: `/etc/dovecot/conf.d/10-ssl.conf`

```bash
sudo vi /etc/dovecot/conf.d/10-ssl.conf
```

```text
ssl = required
ssl_cert = </etc/pki/dovecot/certs/mail.nhr.local.crt
ssl_key  = </etc/pki/dovecot/private/mail.nhr.local.key
ssl_protocols = !SSLv2 !SSLv3 !TLSv1 !TLSv1.1
```

> [!NOTE]
> On RHEL 7, `/etc/pki/dovecot/` already carries the correct `dovecot_cert_t` SELinux type. Storing certs anywhere else (e.g. `/etc/postfix/ssl/`) will cause a **Permission denied** error at startup even if Unix permissions look correct.

### 5.5 SASL Socket for Postfix: `/etc/dovecot/conf.d/10-master.conf`

Dovecot exposes a Unix socket so Postfix can hand off SASL authentication to it. This way there is no need to run a separate Cyrus-SASL daemon.

```bash
sudo vi /etc/dovecot/conf.d/10-master.conf
```

The `service auth {}` block **already exists** in this file. Inside it, find the commented-out `# Postfix smtp-auth` section and **uncomment** those lines, then set the correct `mode`, `user`, and `group`:

Find this (lines ~91–94 in the default file):

```text
  # Postfix smtp-auth
  #unix_listener /var/spool/postfix/private/auth {
  #  mode = 0666
  #}
```

Change it to:

```text
  # Postfix smtp-auth
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }
```

> [!NOTE]
> Do **not** add a second `service auth { }` block — one already exists. Only edit inside the existing block. `mode = 0660` (not `0666`) restricts access to the `postfix` group only, which is more secure.

### 5.6 Restart Dovecot

```bash
sudo doveconf -n
sudo systemctl restart dovecot
sudo systemctl status dovecot
```

---

## 6. User Authentication

### 6.1 Local System Account Authentication

By default, Postfix and Dovecot authenticate against local system accounts via PAM (`/etc/passwd`). Create a dedicated mail user for testing without giving it shell access:

```bash
# Create a local mail user account (no shell login required)
sudo useradd -m -s /sbin/nologin mailuser1

# Set the user's password (users authenticate with this over IMAP/SMTP)
sudo passwd mailuser1

# Create the Maildir structure for the new user
sudo mkdir -p /home/mailuser1/Maildir/{new,cur,tmp}
sudo chown -R mailuser1:mailuser1 /home/mailuser1/Maildir
```

### 6.2 LDAP Integration (Optional)

If the company already has an LDAP directory such as OpenLDAP or Active Directory, replace the PAM back-end in Dovecot with LDAP for centralized authentication:

```bash
# Install the Dovecot LDAP authentication driver
sudo yum install -y dovecot-ldap
```

In `/etc/dovecot/conf.d/10-auth.conf`, comment out the PAM passdb/userdb blocks and add:

```text
# Enable LDAP authentication driver
!include auth-ldap.conf.ext
```

Configure `/etc/dovecot/dovecot-ldap.conf.ext`:

```text
#
# /etc/dovecot/dovecot-ldap.conf.ext - LDAP authentication (optional)
#

# LDAP server URI
hosts           = ldap://192.168.1.5

# Bind DN and password used by Dovecot to query LDAP
dn              = cn=dovecot,dc=nhr,dc=local
dnpass          = ldap_bind_password

# Base DN for user lookups
base            = ou=Users,dc=nhr,dc=local

# Attribute mappings
user_attrs      = uid=user
pass_attrs      = uid=user,userPassword=password
```

### 6.3 SASL Authentication Flow

The full authentication path from a mail client through to the back-end:

```
Mail Client (Thunderbird / Outlook)
        |  SMTP AUTH (PLAIN / LOGIN) over TLS
        v
Postfix (smtpd)
        |  delegates auth via Unix socket (private/auth)
        v
Dovecot (auth service)  ->  PAM / LDAP
        |  returns OK / FAIL
        v
Postfix permits or rejects the MAIL FROM
```

---

## 7. SPAM Filtering and Security

### 7.1 Install and Configure SpamAssassin

SpamAssassin scores each incoming message using heuristic tests and Bayesian analysis. Messages that exceed the score threshold are tagged or rejected.

```bash
# Install SpamAssassin
sudo yum install -y spamassassin

# Enable and start the spamd daemon
sudo systemctl enable spamassassin
sudo systemctl start spamassassin
```

Edit `/etc/mail/spamassassin/local.cf` to configure thresholds and behavior:

```bash
sudo vi /etc/mail/spamassassin/local.cf
```

```text
#
# /etc/mail/spamassassin/local.cf - SpamAssassin local policy
#

# required_score: messages scoring above this value are treated as spam.
# 5.0 is the default; lower = more aggressive filtering.
required_score  5.0

# report_safe: 0 = modify headers only (do not attach original as .eml).
report_safe     0

# rewrite_header: prepend "[SPAM]" to the subject of identified spam.
rewrite_header  Subject [SPAM]

# use_bayes: enable Bayesian classifier
use_bayes       1

# bayes_auto_learn: automatically learn from ham and spam that Postfix delivers
bayes_auto_learn 1
```

### 7.2 Integrate SpamAssassin with Postfix

Connecting SpamAssassin to Postfix requires five steps: enabling the content filter in `main.cf`, creating the dedicated `spamd` OS user, fixing TLS certificate permissions, adding the transport and pickup override in `master.cf`, and restarting Postfix.

#### Step 1 — Enable the Content Filter in `/etc/postfix/main.cf`

Add the following line to route all inbound SMTP mail through SpamAssassin before final delivery:

```bash
sudo vi /etc/postfix/main.cf
```

```text
# content_filter: pipe all incoming SMTP mail through SpamAssassin
content_filter = spamassassin
```

#### Step 2 — Create the `spamd` OS User

The `spamassassin` transport in `master.cf` runs as OS user `spamd`, but this user is **not created automatically** by the RHEL 7 package. Without it, every delivery attempt fails with:

```
fatal: get_service_attr: unknown username: spamd
```

Create it as a locked system account with no shell access:

```bash
# Create the spamd system user required by the Postfix pipe transport
sudo useradd -r -s /sbin/nologin -d /var/lib/spamassassin spamd
sudo mkdir -p /var/lib/spamassassin
sudo chown spamd:spamd /var/lib/spamassassin
```

#### Step 3 — Fix TLS Certificate Permissions

Postfix reads the TLS certificate and private key at startup. If the key is not readable by the `postfix` group, STARTTLS is silently disabled and the following errors appear in `/var/log/maillog`:

```
warning: cannot get RSA certificate from file ...: disabling TLS support
warning: TLS library problem: fopen: Permission denied
```

Fix the permissions so Postfix can read both files:

```bash
# Certificate: world-readable is fine (contains no secret)
sudo chmod 644 /etc/pki/dovecot/certs/mail.nhr.local.crt

# Private key: readable by root and postfix group only
sudo chown root:postfix /etc/pki/dovecot/private/mail.nhr.local.key
sudo chmod 640 /etc/pki/dovecot/private/mail.nhr.local.key
```

#### Step 4 — Verify `master.cf` Contains the Transport and Pickup Override

The `spamassassin` transport and the `pickup` override were already added in §4.3. Confirm they are present:

```bash
sudo grep -A2 'spamassassin\|^pickup' /etc/postfix/master.cf
```

Expected output:

```text
pickup    fifo  n       -       n       60      1       pickup
  -o content_filter=
  -o receive_override_options=no_header_body_checks

spamassassin unix  -       n       n       -       -       pipe
  user=spamd argv=/usr/bin/spamc -f -e /usr/sbin/sendmail -oi -f ${sender} ${recipient}
```

The `pickup` override prevents the infinite re-injection loop: when SpamAssassin finishes scanning and re-submits the message via `/usr/sbin/sendmail`, the pickup daemon delivers it directly without triggering `content_filter` a second time.

#### Step 5 — Restart Postfix

```bash
# Check configuration for syntax errors first
sudo postfix check

# Restart to load all changes
sudo systemctl restart postfix

# Confirm the service came up cleanly
sudo systemctl status postfix
```

### 7.3 Firewall Rules with firewalld

Open only the ports that the mail server actually needs:

```bash
# Allow standard SMTP (port 25) for server-to-server relay
sudo firewall-cmd --permanent --add-service=smtp

# Allow SMTPS (port 465) for legacy secure submission
sudo firewall-cmd --permanent --add-port=465/tcp

# Allow SMTP Submission (port 587) for mail client authentication
sudo firewall-cmd --permanent --add-port=587/tcp

# Allow IMAP (port 143) for IMAP without SSL (STARTTLS required)
sudo firewall-cmd --permanent --add-service=imap

# Allow IMAPS (port 993) for IMAP over SSL
sudo firewall-cmd --permanent --add-service=imaps

# Allow POP3 (port 110)
sudo firewall-cmd --permanent --add-service=pop3

# Allow POP3S (port 995) for POP3 over SSL
sudo firewall-cmd --permanent --add-service=pop3s

# Reload the firewall to apply all permanent rules
sudo firewall-cmd --reload

# Verify all services and ports in the active zone
sudo firewall-cmd --list-all
```

---

## 8. DNS Configuration

Mail delivery depends on correct DNS records. Other mail servers look up MX records to find where to send mail for `nhr.local`, and PTR records are checked by receiving servers to validate the sender's identity. Configure both on the existing BIND server (`ns1.nhr.local`).

### 8.1 MX Record in the Forward Zone

On the BIND DNS server (`ns1.nhr.local`), edit the forward zone file `/var/named/dynamic/nhr.local.zone`:

```bash
sudo vi /var/named/dynamic/nhr.local.zone
```

Add the mail server A record and MX record (and increment the serial number at the top of the file):

```text
; A record for the mail server
mail    IN  A    192.168.1.30

; MX record - directs inbound mail for nhr.local to mail.nhr.local
; Priority 10 means this is the preferred (and only) mail exchanger.
@       IN  MX   10  mail.nhr.local.
```

Check zone syntax and reload BIND:

```bash
# Check zone file syntax after editing
sudo named-checkzone nhr.local /var/named/dynamic/nhr.local.zone

# Reload the zone without restarting the entire named daemon
sudo rndc reload nhr.local
```

### 8.2 PTR Record in the Reverse Zone

On the BIND DNS server, edit the reverse zone file `/var/named/dynamic/nhr.local.rev`:

```bash
sudo vi /var/named/dynamic/nhr.local.rev
```

Add:

```text
; PTR - last octet of the mail server's IP mapped to its FQDN
; 192.168.1.30 -> mail.nhr.local.
30      IN  PTR  mail.nhr.local.
```

Validate and reload:

```bash
sudo named-checkzone 1.168.192.in-addr.arpa /var/named/dynamic/nhr.local.rev
sudo rndc reload 1.168.192.in-addr.arpa
```

### 8.3 Verify DNS Records from the Mail Server

```bash
# Forward lookup - confirm MX record resolution
dig MX nhr.local

# Verify the mail server A record
dig A mail.nhr.local

# Reverse lookup - confirm PTR record
dig -x 192.168.1.30
```

---

## 9. Testing and Validation

### 9.1 Test SMTP with telnet

Use `telnet` to connect directly to port 25 and issue SMTP commands manually to confirm the server is running and advertising STARTTLS and AUTH:

```bash
# Connect to port 25 on the mail server
telnet mail.nhr.local 25
```

Expected SMTP conversation:

```text
Trying 192.168.1.30...
Connected to mail.nhr.local.
Escape character is '^]'.
220 mail.nhr.local ESMTP Postfix
EHLO client.nhr.local
250-mail.nhr.local
250-PIPELINING
250-SIZE 10240000
250-STARTTLS
250-AUTH PLAIN LOGIN
250 8BITMIME
QUIT
221 2.0.0 Bye
```

### 9.2 Send a Test Email Using mailx

```bash
# Send a test email from the command line to mailuser1
echo "Test mail body from NHR mail server." | mailx \
    -s "Mail Server Test" \
    -r "admin@nhr.local" \
    mailuser1@nhr.local
```

Actual terminal output confirming the command completed without error:

```text
[root@mail ~]# echo "Test mail body from NHR mail server." | mailx \
>     -s "Mail Server Test" \
>     -r "admin@nhr.local" \
>     mailuser1@nhr.local
[root@mail ~]#
```

The mail log confirms acceptance and delivery through the SpamAssassin content filter:

```text
Jul  4 07:13:48 rhel postfix/pickup[3643]: AC3E2109C125: uid=0 from=<admin@nhr.local>
Jul  4 07:13:48 rhel postfix/cleanup[3649]: AC3E2109C125: message-id=<6a485e4c.Fwc7C/VLlxlwmyGv%admin@nhr.local>
Jul  4 07:13:48 rhel postfix/qmgr[3644]: AC3E2109C125: from=<admin@nhr.local>, size=458, nrcpt=1 (queue active)
Jul  4 07:13:48 rhel postfix/local[3651]: AC3E2109C125: to=<mailuser1@nhr.local>, relay=local, delay=0.06, delays=0.04/0.01/0/0.01, dsn=2.0.0, status=sent (delivered to maildir)
Jul  4 07:13:48 rhel postfix/qmgr[3644]: AC3E2109C125: removed
```

`status=sent (delivered to maildir)` confirms Postfix wrote the message directly into `mailuser1`'s Maildir.

### 9.3 Verify Mail Delivery in the Maildir

```bash
# List received messages in mailuser1's Maildir new folder
ls -la /home/mailuser1/Maildir/new/

# Read the content of the first received message
cat /home/mailuser1/Maildir/new/*
```

Actual terminal output confirming the message file is present:

```text
[root@mail ~]# ls -la /home/mailuser1/Maildir/new/
total 4
drwxr-xr-x. 2 mailuser1 mailuser1  58 Jul  4 07:13 .
drwxr-xr-x. 5 mailuser1 mailuser1  39 Jul  4 06:38 ..
-rw-------. 1 mailuser1 mailuser1 545 Jul  4 07:13 1783127628.Vfd02I10a33M733280.mail.nhr.local
```

Reading the full message with all SMTP headers:

```text
[root@mail ~]# cat /home/mailuser1/Maildir/new/*
Return-Path: <admin@nhr.local>
X-Original-To: mailuser1@nhr.local
Delivered-To: mailuser1@nhr.local
Received: by mail.nhr.local (Postfix, from userid 0)
	id AC3E2109C125; Sat,  4 Jul 2026 07:13:48 +0600 (+06)
Date: Sat, 04 Jul 2026 07:13:48 +0600
From: admin@nhr.local
To: mailuser1@nhr.local
Subject: Mail Server Test
Message-ID: <6a485e4c.Fwc7C/VLlxlwmyGv%admin@nhr.local>
User-Agent: Heirloom mailx 12.5 7/5/10
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit

Test mail body from NHR mail server.
```

The message was received with correct `Return-Path`, `Delivered-To`, and `Received` headers, confirming end-to-end delivery from Postfix to Maildir format.

### 9.4 Test IMAPS with openssl

Test the IMAP service over TLS to confirm Dovecot is operating correctly. Since `openssl s_client` may send stray keystrokes before a manual command can be typed, pipe all IMAP commands in a single subshell with short pauses between each command:

```bash
# Connect to IMAPS (port 993) and run a full IMAP session non-interactively
(sleep 1; echo "a LOGIN mailuser1 <password>"; sleep 1; echo 'a LIST "" "*"'; sleep 1; echo "a LOGOUT"; sleep 1) | openssl s_client -connect mail.nhr.local:993 -quiet 2>/dev/null
```

The `sleep 1` pauses give the TLS handshake time to complete before each IMAP command is sent, ensuring the server only ever receives valid input.

#### Actual Test Output

```text
[root@mail ~]# (sleep 1; echo "a LOGIN mailuser1 redhat"; sleep 1; echo 'a LIST "" "*"'; sleep 1; echo "a LOGOUT"; sleep 1) | openssl s_client -connect mail.nhr.local:993 -quiet 2>/dev/null
* OK [CAPABILITY IMAP4rev1 LITERAL+ SASL-IR LOGIN-REFERRALS ID ENABLE IDLE AUTH=PLAIN AUTH=LOGIN] Dovecot ready.
a OK [CAPABILITY IMAP4rev1 LITERAL+ SASL-IR LOGIN-REFERRALS ID ENABLE IDLE SORT SORT=DISPLAY THREAD=REFERENCES THREAD=REFS THREAD=ORDEREDSUBJECT MULTIAPPEND URL-PARTIAL CATENATE UNSELECT CHILDREN NAMESPACE UIDPLUS LIST-EXTENDED I18NLEVEL=1 CONDSTORE QRESYNC ESEARCH ESORT SEARCHRES WITHIN CONTEXT=SEARCH LIST-STATUS BINARY MOVE SNIPPET=FUZZY SPECIAL-USE] Logged in
* LIST (\HasNoChildren) "." INBOX
a OK List completed (0.001 + 0.000 secs).
* BYE Logging out
a OK Logout completed (0.001 + 0.000 secs).
```

Key observations from the output:

| Line                                | Meaning                                  | Status  |
| ----------------------------------- | ---------------------------------------- | ------- |
| `* OK [...] Dovecot ready.`         | TLS handshake succeeded, IMAP service up | Success |
| `a OK [...] Logged in`              | SASL authentication with PAM successful  | Success |
| `* LIST (\HasNoChildren) "." INBOX` | mailuser1's INBOX folder exists          | Success |
| `a OK List completed`               | Folder listing completed cleanly         | Success |
| `* BYE Logging out`                 | Server acknowledged logout request       | Success |
| `a OK Logout completed`             | IMAP session closed cleanly              | Success |

> [!NOTE]
> The `verify error:num=18` (self-signed certificate) warning is suppressed here by `2>/dev/null`. It's expected in a lab environment — in production, a certificate from a trusted CA such as Let's Encrypt would eliminate it entirely.

### 9.5 Configure a Mail Client (Thunderbird)

Mozilla Thunderbird is a free, open-source desktop email client available for Windows, macOS, and Linux. Use it to connect to the mail server over IMAP and SMTP with TLS encryption.

#### Prerequisites on the Client Machine

On the client (`client.nhr.local`, `192.168.1.20`), ensure the mail server hostname resolves correctly:

```bash
# Verify DNS resolution of the mail server from the client
nslookup mail.nhr.local
```

Since the server uses a **self-signed certificate**, accept the security exception when Thunderbird prompts during the first connection.

#### Create a Second Test User on the Mail Server

To test sending and receiving between two mailboxes, create a second user:

```bash
# On the mail server - create a second mail user
sudo useradd -m -s /sbin/nologin mailuser2
sudo passwd mailuser2

# Create Maildir structure for mailuser2
sudo mkdir -p /home/mailuser2/Maildir/{new,cur,tmp}
sudo chown -R mailuser2:mailuser2 /home/mailuser2/Maildir
```

#### Step-by-Step: Add an Account in Thunderbird

**Step 1 — Launch Thunderbird**

Open Thunderbird. On first launch it shows the account setup wizard. If already open, go to:
`Menu → Account Settings → Account Actions → Add Mail Account…`

**Step 2 — Enter Account Details**

In the *Your Name*, *Email Address*, and *Password* fields, enter:

| Field         | Value                        |
| ------------- | ---------------------------- |
| Your Name     | `Mail User 1`                |
| Email Address | `mailuser1@nhr.local`        |
| Password      | *(password set with passwd)* |

Click **Continue**. Thunderbird will attempt auto-discovery — it will fail for a private domain, which is expected.

**Step 3 — Enter Server Settings Manually**

Click **Configure Manually** and fill in the following:

| Direction    | Protocol | Server           | Port  | SSL      | Authentication  |
| ------------ | -------- | ---------------- | ----- | -------- | --------------- |
| **Incoming** | IMAP     | `mail.nhr.local` | `993` | SSL/TLS  | Normal password |
| **Outgoing** | SMTP     | `mail.nhr.local` | `587` | STARTTLS | Normal password |

- **Incoming Username**: `mailuser1`
- **Outgoing Username**: `mailuser1`

**Step 4 — Accept the Self-Signed Certificate**

Since this lab uses a self-signed certificate, Thunderbird will show a security warning. Click **Confirm Security Exception** to proceed.

**Step 5 — Done**

Click **Done**. Thunderbird will connect and display `mailuser1`'s inbox.

Repeat the process to add a second account for `mailuser2@nhr.local` to test mail exchange between users.

#### Quick Reference: Thunderbird Account Settings

| Setting            | Value                                  |
| ------------------ | -------------------------------------- |
| Incoming Server    | `mail.nhr.local`, port `993`, SSL/TLS  |
| Outgoing Server    | `mail.nhr.local`, port `587`, STARTTLS |
| Username           | `mailuser1` (or `mailuser2`)           |
| Authentication     | Normal password                        |
| Accept certificate | Yes (self-signed exception required)   |

#### Verify Mail Exchange

1. In Thunderbird, compose a new message **from** `mailuser1@nhr.local` **to** `mailuser2@nhr.local` and send it.
2. Switch to the `mailuser2` account — the message should appear in the Inbox.
3. Reply from `mailuser2` back to `mailuser1` and confirm delivery in the other inbox.

On the server side, confirm delivery in the mail log:

```bash
sudo tail -f /var/log/maillog
```

---

## 10. Monitoring and Logging

### 10.1 Log Files

On RHEL 7, rsyslog writes all mail activity to `/var/log/maillog`. This covers delivery attempts, authentication events, TLS handshakes, bounce reasons, and daemon errors.

```bash
# Tail the mail log in real-time to watch delivery events
sudo tail -f /var/log/maillog

# Search for delivery failures
sudo grep -i "status=bounced\|reject\|error" /var/log/maillog

# View Postfix-specific log entries
sudo grep "postfix" /var/log/maillog | tail -50
```

### 10.2 Postfix Queue Management

```bash
# List all messages currently in the Postfix mail queue
sudo mailq

# Force Postfix to flush the queue (retry deferred messages immediately)
sudo postfix flush

# Delete all messages from the deferred queue (use with caution)
sudo postsuper -d ALL deferred
```

### 10.3 Monitoring with Nagios

Nagios monitors the mail server by checking that SMTP, IMAP, and POP3 ports are responding. The following configuration goes on the Nagios monitoring server.

```bash
# On the Nagios server - install monitoring plugins
sudo yum install -y nagios-plugins-all
```

Define the mail server host in `/etc/nagios/objects/hosts/mail.nhr.local.cfg`:

```text
define host {
    use                     linux-server
    host_name               mail.nhr.local
    alias                   Postfix/Dovecot Mail Server
    address                 192.168.1.30
    max_check_attempts      5
    check_period            24x7
    notification_interval   30
    notification_period     24x7
}
```

Define SMTP, IMAP, and POP3 service checks in `/etc/nagios/objects/services/mail-services.cfg`:

```text
# SMTP check on port 25
define service {
    use                     generic-service
    host_name               mail.nhr.local
    service_description     SMTP Check
    check_command           check_smtp!-p 25
    check_interval          5
    max_check_attempts      3
    notification_period     24x7
}

# IMAP check on port 143
define service {
    use                     generic-service
    host_name               mail.nhr.local
    service_description     IMAP Check
    check_command           check_imap!-p 143
    check_interval          5
    max_check_attempts      3
    notification_period     24x7
}

# POP3 check on port 110
define service {
    use                     generic-service
    host_name               mail.nhr.local
    service_description     POP3 Check
    check_command           check_pop!-p 110
    check_interval          5
    max_check_attempts      3
    notification_period     24x7
}
```

Validate and restart Nagios:

```bash
sudo nagios -v /etc/nagios/nagios.cfg
sudo systemctl restart nagios
```

### 10.4 Monitoring with Zabbix

The Zabbix agent runs on the mail server and reports metrics back to the Zabbix monitoring server.

```bash
# On the mail server - install and configure Zabbix agent
sudo yum install -y zabbix-agent

# Enable and start the Zabbix agent
sudo systemctl enable zabbix-agent
sudo systemctl start zabbix-agent
```

Create a custom Postfix queue monitoring parameter in `/etc/zabbix/zabbix_agentd.d/userparameter_postfix.conf`:

```text
# Count messages currently in the Postfix mail queue
UserParameter=postfix.queue.size,mailq 2>/dev/null | grep -c "^[0-9A-F]"
```

Test the custom parameter:

```bash
sudo systemctl restart zabbix-agent
zabbix_get -s 127.0.0.1 -k postfix.queue.size
```

In the Zabbix web interface:

1. Go to **Configuration → Hosts → Create host**.
2. Set **Host name** to `mail.nhr.local` and **Agent interfaces** to `192.168.1.30:10050`.
3. Link the host to the **Postfix by Zabbix agent** template.
4. Save the host.

---

## 11. Backup and Disaster Recovery

The backup covers Postfix configuration files, Dovecot configuration files, SSL certificates, and all user mailboxes.

### 11.1 Create the Automated Backup Script

```bash
# Create the mail server backup script
sudo tee /usr/local/sbin/mail-backup.sh > /dev/null << 'EOF'
#!/bin/bash
# RHEL Mail Server - Automated Backup Script
# Archives Postfix config, Dovecot config, SSL certificates,
# and all user mailboxes to /backup/mail/

BACKUP_DIR="/backup/mail"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/mail_backup_${DATE}.tar.gz"

mkdir -p "${BACKUP_DIR}"
chmod 700 "${BACKUP_DIR}"

# Archive all critical mail server files
tar -czf "${BACKUP_FILE}" \
    /etc/postfix/ \
    /etc/dovecot/ \
    /etc/mail/spamassassin/ \
    /var/mail/ \
    /home/*/Maildir/ \
    2>/dev/null

echo "[$(date)] Backup completed: ${BACKUP_FILE}" >> /var/log/mail-backup.log

# Prune backups older than 30 days
find "${BACKUP_DIR}" -type f -name "mail_backup_*.tar.gz" -mtime +30 -delete
EOF

# Make the script executable
sudo chmod 700 /usr/local/sbin/mail-backup.sh
```

### 11.2 Configure Cron Automation

Schedule the backup to run nightly at 03:00 AM:

```bash
# Add to root's crontab
sudo crontab -e
```

Add the following cron entry:

```text
# Run mail server backup every day at 03:00 AM
0 3 * * * /usr/local/sbin/mail-backup.sh > /dev/null 2>&1
```

### 11.3 Disaster Recovery Procedure

To restore the mail server from a backup archive:

```bash
# List available backup archives
ls -lh /backup/mail/

# Restore configuration and mailboxes from the most recent archive
sudo tar -xzf /backup/mail/mail_backup_<DATE>.tar.gz -C /

# Restore correct SELinux contexts after extraction
sudo restorecon -Rv /etc/postfix/ /etc/dovecot/ /var/mail/ /home/

# Restart services after restoration
sudo systemctl restart postfix dovecot
```

---

## 12. Documentation and Maintenance

### 12.1 Regular Security Patching

Security updates for Postfix, Dovecot, and SpamAssassin should be applied regularly. On RHEL these come through the Red Hat Network or a local Satellite server:

```bash
# Check for available updates for mail server components
sudo yum check-update postfix dovecot spamassassin

# Apply all pending security updates
sudo yum update -y postfix dovecot spamassassin

# Restart services after updates to load new binaries
sudo systemctl restart postfix dovecot spamassassin
```

### 12.2 SSL Certificate Renewal

Self-signed certificates should be checked and renewed before expiry:

```bash
# Check the expiry date of the current certificate
sudo openssl x509 -noout -dates \
    -in /etc/pki/dovecot/certs/mail.nhr.local.crt

# Regenerate the certificate when approaching expiry
sudo openssl req -new -x509 \
    -nodes -days 3650 \
    -newkey rsa:2048 \
    -keyout /etc/pki/dovecot/private/mail.nhr.local.key \
    -out    /etc/pki/dovecot/certs/mail.nhr.local.crt \
    -subj   "/C=BD/ST=Dhaka/L=Dhaka/O=NHR Company/OU=IT/CN=mail.nhr.local"

# Reload services to apply the new certificate
sudo systemctl reload postfix dovecot
```

### 12.3 SpamAssassin Rule Updates

SpamAssassin ships with a rule update tool (`sa-update`) that pulls new detection rules from the project's repositories. Run this weekly via cron:

```bash
# Update SpamAssassin rules
sudo sa-update

# Reload SpamAssassin after rule updates
sudo systemctl reload spamassassin
```

Add a weekly cron entry:

```text
# Update SpamAssassin rules every Sunday at 01:00 AM
0 1 * * 0 /usr/bin/sa-update && /bin/systemctl reload spamassassin
```

### 12.4 Quick Reference: Service and Port Summary

| Service        | Protocol | Port | Purpose                                   |
| -------------- | -------- | ---- | ----------------------------------------- |
| Postfix SMTP   | TCP      | 25   | Server-to-server mail relay (MX delivery) |
| Postfix Sub.   | TCP      | 587  | Mail client submission (STARTTLS + SASL)  |
| Postfix SMTPS  | TCP      | 465  | Legacy secure submission (implicit TLS)   |
| Dovecot IMAP   | TCP      | 143  | IMAP with STARTTLS upgrade                |
| Dovecot IMAPS  | TCP      | 993  | IMAP over implicit TLS                    |
| Dovecot POP3   | TCP      | 110  | POP3 with STARTTLS upgrade                |
| Dovecot POP3S  | TCP      | 995  | POP3 over implicit TLS                    |

### 12.5 Validation Summary

| Component        | Quick Validation Command                                         |
| ---------------- | ---------------------------------------------------------------- |
| **Postfix**      | `sudo postfix check && sudo postfix status`                      |
| **Dovecot**      | `sudo doveconf -n && sudo systemctl status dovecot`              |
| **SMTP**         | `telnet mail.nhr.local 25`                                       |
| **IMAPS**        | `openssl s_client -connect mail.nhr.local:993`                   |
| **Mail Queue**   | `sudo mailq`                                                     |
| **Mail Log**     | `sudo tail -f /var/log/maillog`                                  |
| **MX Record**    | `dig MX nhr.local`                                               |
| **PTR Record**   | `dig -x 192.168.1.30`                                            |
| **SpamAssassin** | `sudo spamassassin --lint`                                       |
| **Nagios**       | `/usr/lib64/nagios/plugins/check_smtp -H mail.nhr.local -p 25`  |
| **Zabbix**       | `zabbix_get -s 127.0.0.1 -k postfix.queue.size`                 |
