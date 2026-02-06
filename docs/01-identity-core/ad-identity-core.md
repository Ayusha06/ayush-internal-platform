# Active Directory – Identity Core Design

## 1. Problem

Organizations require a centralized and authoritative identity system to control access
to infrastructure and services.

Local user accounts on servers do not scale and do not provide auditability, lifecycle
management or consistent access control.

---

## 2. Design decision

Windows Active Directory is used as the authoritative identity platform.

The domain controller is hosted in AWS to simplify availability and remote access while
serving both cloud and on-prem workloads.

---

## 3. Architecture

Domain:

corp.lab

Primary domain controller:

DC-CLOUD-01

---

## 4. Deployment

- Windows Server 2022
- Active Directory Domain Services
- Integrated DNS

The domain controller hosts:

- authentication services
- Kerberos
- LDAP
- domain DNS

---

## 5. Organizational design

The directory is structured using administrative tiering.

High-privilege objects are separated from standard workloads.

Example structure:

- Admins
  - Tier0
  - Tier1
- Groups
  - Access (ACC-*)
  - Role (ROLE-*)
- Users
- Servers

---

## 6. Access model

Two types of groups are used:

Access groups:

ACC-*

These groups control which systems or services a user may access.

Role groups:

ROLE-*

These groups control what level of privilege a user has after access is granted.

---

## 7. Example objects

Administrative user:

ayush.admin

Standard user:

ayush.agrawal

Example access group:

ACC-Linux-SSH

Example role group:

ROLE-Linux-Admins

---

## 8. Validation

- domain operational
- DNS resolving corp.lab
- users and groups created
- administrative separation enforced

---

## 9. Operational notes

The domain controller also provides DNS services for all integrated workloads.
