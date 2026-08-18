# Kerberos Deployment Guide for CDP (Cloudera Data Platform)

This guide walks through installing and configuring a Kerberos Key Distribution
Center (KDC) and Admin Server for securing a CDP cluster, creating principals
for cluster services and users, and generating keytab files for
non-interactive authentication.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Step 1: Install the KDC and Admin Server](#step-1-install-the-kdc-and-admin-server)
4. [Step 2: Initialize the Kerberos Realm](#step-2-initialize-the-kerberos-realm)
5. [Step 3: Configure `krb5.conf` and `kdc.conf`](#step-3-configure-krb5conf-and-kdcconf)
6. [Step 4: Create the Kerberos Admin Principal](#step-4-create-the-kerberos-admin-principal)
7. [Step 5: Configure ACLs for the Admin Server](#step-5-configure-acls-for-the-admin-server)
8. [Step 6: Install Kerberos Client Tools](#step-6-install-kerberos-client-tools)
9. [Step 7: Create Service and User Principals](#step-7-create-service-and-user-principals)
10. [Kerberos Administration Commands](#kerberos-administration-commands)
11. [Working with Keytab Files](#working-with-keytab-files)
12. [Quick Reference Summary](#quick-reference-summary)

---

## Overview

Kerberos is a network authentication protocol that uses secret-key
cryptography to provide strong authentication for client/server applications.
In a CDP deployment, Kerberos is used to secure communication between Hadoop
ecosystem services (HDFS, YARN, Hive, etc.) and to authenticate users and
service accounts.

This deployment consists of:

- **KDC (Key Distribution Center)** — issues Kerberos tickets.
- **Admin Server (`kadmind`)** — manages principals and policies.
- **Realm** — the administrative domain, in this case `HADOOP.COM`.
- **Principals** — identities (users or services) that Kerberos authenticates.
- **Keytabs** — files storing encrypted keys, used for password-less
  (non-interactive) authentication by services and scripts.

---

## Prerequisites

- A dedicated host to run the KDC and Admin Server (commonly the Cloudera
  Manager host, referred to below as `<private-dns-of-cm>`).
- Root or `sudo` access on all hosts involved.
- A decided-upon **realm name** (convention: uppercase domain), e.g.
  `HADOOP.COM`.
- Network connectivity between the KDC host and all client hosts.

---

## Step 1: Install the KDC and Admin Server

Run the following **on the Cloudera Manager (CM) host**:

```bash
sudo apt-get install -y rng-tools
sudo apt install krb5-kdc krb5-admin-server
```

- `rng-tools` improves entropy generation, which speeds up key generation.
- `krb5-kdc` installs the Key Distribution Center daemon.
- `krb5-admin-server` installs the administrative server (`kadmind`) used to
  manage principals remotely.

During installation you will be prompted for:

| Prompt | Value |
|---|---|
| Default Kerberos version 5 realm | `HADOOP.COM` |
| Kerberos servers for your realm | `<private-dns-of-cm>` |
| Administrative server for your Kerberos realm | `<private-dns-of-cm>` |

---

## Step 2: Initialize the Kerberos Realm

Create the realm's master database and stash key:

```bash
sudo krb5_newrealm
```

This command:
- Creates the KDC database for the realm.
- Prompts for a **master password** (used to encrypt the database).
- Generates a `/etc/krb5kdc/stash` file so the KDC can start without manual
  password entry.

---

## Step 3: Configure `krb5.conf` and `kdc.conf`

### `krb5.conf` — client-facing realm configuration

```bash
nano /etc/krb5.conf
```

Confirm the `[libdefaults]`, `[realms]`, and `[domain_realm]` sections
correctly reference your realm (`HADOOP.COM`) and KDC host.

### `kdc.conf` — KDC server configuration

```bash
sudo nano /usr/share/doc/krb5-kdc/examples/kdc.conf
```

- Uncomment the **supported encryption types** line so the KDC advertises the
  ciphers you intend to use (e.g. `aes256-cts:normal aes128-cts:normal`).

Restart the admin server to apply changes:

```bash
sudo /etc/init.d/krb5-admin-server restart
```

---

## Step 4: Create the Kerberos Admin Principal

Enter the local Kerberos admin shell (uses the local database directly, no
network auth required):

```bash
sudo kadmin.local
```

Inside the `kadmin.local` prompt:

```
addprinc cm/admin
quit
```

This creates the `cm/admin@HADOOP.COM` principal, which will be used to
administer the realm remotely (e.g. from Cloudera Manager).

---

## Step 5: Configure ACLs for the Admin Server

Grant the admin principal full administrative rights:

```bash
sudo nano /etc/krb5kdc/kadm5.acl
```

Add the line:

```
cm/admin@HADOOP.COM *
```

This grants `cm/admin@HADOOP.COM` **all** (`*`) privileges over the realm's
principal database.

Restart the admin server to load the new ACL:

```bash
sudo /etc/init.d/krb5-admin-server restart
```

---

## Step 6: Install Kerberos Client Tools

On any host that needs to authenticate against the KDC (including the KDC
host itself, for testing):

```bash
sudo apt-get install krb5-user -y
```

During setup, when prompted for the default encryption type, select:

```
aes256-cts
```

---

## Step 7: Create Service and User Principals

Re-enter the admin shell:

```bash
sudo kadmin.local
```

### HDFS superuser principal

```
addprinc hdfs@HADOOP.COM
```

> **Note:** Realm names are case-sensitive and conventionally uppercase.
> Use `HADOOP.COM`, not `HADOOP.com`.

### Additional user principals

```
addprinc usera
addprinc userb
addprinc user2
addprinc userc
```

Each `addprinc` prompts for (or auto-generates, with `-randkey`) a password
for that principal.

---

## Kerberos Administration Commands

These commands are run inside `kadmin` or `kadmin.local`:

| Task | Command |
|---|---|
| Add a principal | `addprinc princname` |
| Delete a principal | `delprinc princname` |
| List all principals | `listprincs` |
| Get info on a principal | `getprinc princname` |
| Change a principal's password | `cpw princname` |

---

## Working with Keytab Files

A **keytab** is a file containing one or more principal/key pairs, allowing a
service or script to authenticate to Kerberos **without** interactively
typing a password — essential for automated Hadoop services.

### 1. Create the principal and export its keytab

```bash
sudo kadmin.local
addprinc user2
xst -kt /tmp/user2.keytab user2@HADOOP.COM
```

`xst` ("extract service key table") writes the current key(s) for
`user2@HADOOP.COM` into `/tmp/user2.keytab`.

### 2. Transfer the keytab to the target user/host

```bash
sudo chmod a+r /tmp/user2.keytab
scp -i key.pem /tmp/user2.keytab ubuntu@ip-172-31-87-98.ec2.internal:~
```

> **Security note:** Keytabs are equivalent to a plaintext password for that
> principal. Restrict read access as tightly as possible (ideally to the
> owning service account only) and avoid leaving world-readable copies
> lying around after transfer.

### 3. Authenticate using the keytab

On the destination host:

```bash
chmod 600 user2.keytab
kinit -kt user1.keytab user1
klist
```

- `chmod 600` restricts the keytab to the owning user only.
- `kinit -kt <keytab> <principal>` obtains a Kerberos ticket-granting ticket
  (TGT) using the keytab instead of a password.
- `klist` displays the currently cached tickets, confirming authentication
  succeeded.

> **Tip:** Make sure the keytab filename and the principal passed to `kinit`
> correspond to the same identity — a common source of confusion is
> generating a keytab for one principal (`user2`) and then testing `kinit`
> with another (`user1`).

---

## Quick Reference Summary

| Component | Package | Runs On |
|---|---|---|
| KDC | `krb5-kdc` | CM host |
| Admin server | `krb5-admin-server` | CM host |
| Client tools | `krb5-user` | All cluster/client hosts |

| File | Purpose |
|---|---|
| `/etc/krb5.conf` | Client-side realm/KDC configuration |
| `/usr/share/doc/krb5-kdc/examples/kdc.conf` (→ `/etc/krb5kdc/kdc.conf`) | KDC server configuration, encryption types |
| `/etc/krb5kdc/kadm5.acl` | Admin server access control list |
| `/etc/krb5kdc/stash` | Stashed master key (auto-created by `krb5_newrealm`) |

**End-to-end flow:** install KDC/admin server → initialize realm →
configure `krb5.conf`/`kdc.conf` → create admin principal → set ACLs →
install client tools on all nodes → create service/user principals →
generate and distribute keytabs → services/users authenticate via `kinit`.
