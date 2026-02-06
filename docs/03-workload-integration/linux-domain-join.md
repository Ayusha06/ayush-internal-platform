## 1. Problem

Linux workload servers must authenticate users against the centralized Windows
Active Directory platform instead of using local accounts.

---

## 2. Design decision

Linux servers are integrated with the identity platform using:

- Kerberos
- SSSD
- realmd

The domain controller is reached through the hybrid tunnel.

---

## 3. Components

- linux-app-01 (Ubuntu Server 22.04)
- DC-CLOUD-01 (Windows AD, reachable at 10.50.0.1)

## 4. Pre-join validation

The domain must be discoverable from the Linux server using:

realm discover corp.lab

## 5. Domain join

The Linux server is joined to the domain using:

sudo realm join corp.lab -U ayush.admin

## 6. Identity validation

Domain users must be resolvable through NSS:

id ayush.agrawal@corp.lab
