# Lab: Active Directory & Group Policy for an MSP Client

## Overview

Built a complete Active Directory environment simulating a 25-person law firm
client engagement. Covers domain controller deployment, organizational unit
architecture, user and group provisioning, a domain-joined workstation, and five
Group Policy Objects enforcing security baselines aligned with ABA Model Rule
1.6(c) and NIST SP 800-171 access-control requirements.

This lab was built to mirror the on-premises identity work an MSP performs during
client onboarding and ongoing Tier 2 support.

## Environment

| Role | Host | OS | Address |
|------|------|-----|---------|
| Hypervisor | Debian workstation | Debian 13 (Trixie), VirtualBox 7.x | — |
| Domain Controller | DC01 | Windows Server 2022 Standard Eval (Desktop Experience) | 192.168.56.10 |
| Workstation | WS01 | Windows 11 Enterprise Eval | 192.168.56.20 |

- Forest / domain: `lawfirm.local` (NetBIOS: `LAWFIRM`)
- Functional level: Windows Server 2016 (forest and domain)
- Roles installed: AD DS, DNS (AD-integrated), Global Catalog
- Networking: NAT adapter for internet/updates, Host-Only adapter (vboxnet0) for the client LAN
- DC DNS configuration: preferred 127.0.0.1, alternate 8.8.8.8

## What I Built

### OU Structure

A single top-level OU (`_LawFirmOU`) containing nine child OUs. The underscore
prefix sorts the custom OU above the default containers — a small convention that
matters when an environment grows past a few hundred objects.

| OU | Purpose |
|----|---------|
| `Attorneys` | Attorney user accounts; target for drive-mapping policy |
| `Paralegals` | Paralegal accounts; on-site only, no VPN entitlement |
| `Administration` | Office management and support staff |
| `IT` | Administrative accounts used by the MSP |
| `Workstations` | Domain-joined endpoints; target for endpoint hardening GPOs |
| `Servers` | Member servers, separated so server policy never inherits workstation policy |
| `SecurityGroups` | All security groups, kept out of department OUs |
| `ServiceAccounts` | Non-human accounts, isolated for auditing |
| `DisabledUsers` | Offboarding destination; keeps disabled objects out of active OUs |

Objects were deliberately moved out of the default `Users` and `Computers`
containers, because Group Policy cannot be linked to a container — only to an OU.

### Security Groups

Six global security groups, all `GRP-` prefixed:

`GRP-Attorneys`, `GRP-Paralegals`, `GRP-Administration`, `GRP-AllStaff`,
`GRP-VPN-Users`, `GRP-MFA-Required`

The last two exist to drive access policy rather than file permissions —
VPN entitlement and MFA enforcement are assigned by group membership, which is
how both are managed at scale.

### User Accounts

| User | Logon | OU | Group Memberships |
|------|-------|-----|-------------------|
| Sarah Chen (Managing Partner) | `schen` | Attorneys | Attorneys, AllStaff, VPN-Users, MFA-Required |
| Marcus Rodriguez (Associate) | `mrodriguez` | Attorneys | Attorneys, AllStaff, VPN-Users, MFA-Required |
| Diana Thompson (Paralegal) | `dthompson` | Paralegals | Paralegals, AllStaff, MFA-Required |
| James Patterson (Office Manager) | `jpatterson` | Administration | Administration, AllStaff |
| Jiordani Estama (MSP Admin) | `jestama` | IT | (administrative) |

Paralegals were intentionally excluded from `GRP-VPN-Users` — the firm's model has
them on-site only. Entitlement follows role, not convenience.

### Service Account

`svc-backup` in the `ServiceAccounts` OU, configured with password-never-expires,
user-cannot-change-password, and marked sensitive so the account cannot be
delegated. Expiring a service account password on a schedule breaks whatever task
depends on it; rotation belongs in a planned maintenance window instead.

### Domain Join

The Windows 11 workstation was pointed at DC01 for DNS, joined to `lawfirm.local`,
and then relocated from the default `Computers` container into the `Workstations`
OU so endpoint GPOs would apply.

### Group Policy Objects

| GPO | Linked To | Enforces |
|-----|-----------|----------|
| `GPO-PasswordPolicy` | Domain root | 14-character minimum, complexity, account lockout |
| `GPO-AttorneyDriveMap` | Attorneys OU | Maps `S:` to the LegalFiles share via item-level targeting |
| `GPO-DisableUSBStorage` | Workstations OU | Blocks USB removable-storage read and write |
| `GPO-WindowsUpdate` | Workstations OU | Automatic install, 3:00 AM daily |
| `GPO-LogonBanner` | Domain root | Legal warning notice presented before authentication |

Password policy is linked at the domain root rather than an OU, because domain
account policy is only honored when applied at that level.

## Verification

- `gpresult /r` on the workstation, run as `LAWFIRM\schen`, confirming applied vs. denied GPOs
- `gpresult /h GPReport.html` — full HTML Resultant Set of Policy report
- `S:` drive present for attorney accounts and absent for non-attorney accounts
- USB mass-storage device rejected on a `Workstations` OU machine
- Legal warning banner displayed ahead of the credential prompt
- AD Users and Computers showing final OU placement and group membership

Screenshots for each of the above are in `screenshots/`.

## Known Limitations

- `.local` was used as the domain suffix to mirror legacy SMB environments commonly
  inherited at MSP clients. Current Microsoft guidance favors a delegated subdomain
  of a routable name (for example `ad.example.com`).
- Lab accounts use non-expiring passwords for convenience. This would be a finding
  in any real assessment.
- Single domain controller with no replication partner — acceptable for a lab,
  unacceptable in production, where a second DC is the baseline.

## Lessons Learned

- DNS is the first thing to check on a failed domain join. A client pointed at the
  wrong resolver cannot locate the domain at all, and the error doesn't
  say so.
- GPOs apply to OUs and not the default containers. Objects left in `Computers` or
  `Users` receive nothing.
- Domain account policy must be linked at the domain root to take effect.
- Item-level targeting makes a single drive-mapping GPO serve multiple groups,
  rather than maintaining one GPO per department.
- `gpresult /r` distinguishes "not linked" from "denied by filtering" — the two
  failure modes look identical to the user and require different fixes.

## Repository Contents

- `README.md` — this document
- `screenshots/` — verification evidence referenced above
