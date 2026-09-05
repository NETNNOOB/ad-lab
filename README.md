# Windows Server Active Directory Home Lab

A two-machine Active Directory environment built from scratch on
a Linux workstation: one Windows Server domain controller and one
domain-joined Windows client, running as virtual machines on an
isolated network.

The point of the lab is not that a domain exists. It is the
documentation — the design decisions, the verification steps, and
the problems encountered and resolved along the way.

---

## What is running

| Machine | Role | Address |
|---|---|---|
| DC01 | Domain controller: AD DS, DNS, DHCP | 192.168.10.10 (static) |
| CLIENT01 | Domain-joined Windows client | 192.168.10.100–150 (DHCP) |

- **Domain:** `lab.internal`
- **Network:** isolated KVM/libvirt virtual network, `192.168.10.0/24`
- **Host:** Zorin OS workstation, KVM/QEMU via virt-manager

---

## Design decisions

**Isolated virtual network with no hypervisor DHCP.** The domain
controller is the only source of DNS and DHCP on this segment. If
libvirt also served DHCP, the client would receive the wrong DNS
server and could not locate the domain. The network is defined in
XML rather than through the GUI specifically to guarantee this.

**`lab.internal` rather than `.local` or a publicly owned domain.**
`.local` collides with multicast DNS on Apple and Linux systems.
Using a domain you own publicly creates split-brain DNS. A private
non-routable suffix avoids both.

**No default gateway anywhere.** The segment is isolated, so there
is nowhere for a gateway to lead. Configuring one produces routes
to nothing and misleading connectivity errors.

**The DC's preferred DNS points at itself.** A domain controller
resolving through an external DNS server breaks Active Directory
in ways that surface as unrelated symptoms much later.

**Server Standard with Desktop Experience, not Datacenter or Core.**
Standard covers everything this lab needs. Desktop Experience keeps
the focus on directory concepts rather than PowerShell syntax
during initial learning.

---

## Verification

The build was verified by function rather than by absence of
warnings:

- `dcdiag` partition tests pass against all five AD naming
  contexts (ForestDnsZones, DomainDnsZones, Schema, Configuration,
  and the domain partition)
- `_ldap._tcp.dc._msdcs.lab.internal` resolves to
  `dc01.lab.internal` on port 389 — this is the record a client
  uses to locate the domain
- CLIENT01 receives a DHCP lease in scope with the domain
  controller as its only DNS server
- CLIENT01 appears as a computer object in Active Directory Users
  and Computers, and reports its full name as
  `CLIENT01.lab.internal`

---

## Troubleshooting log

[`troubleshooting-log.md`](troubleshooting-log.md) documents the
problems hit during the build, each in the form: symptom,
hypotheses, evidence, root cause, fix, prevention.

Three issues are recorded so far. Each presented as one thing and
turned out to be caused by another — a GUI missing a feature that
existed underneath it, a health check failing on a healthy system,
and a name resolution symptom pointing at the wrong culprit.

---

## Status

The base build is complete. Planned next: organisational unit
structure, Group Policy, file shares and permissions, PowerShell
automation for user provisioning, and a set of deliberately
introduced faults with documented diagnosis.

---

## Note on licensing

Windows Server and Windows client licences used here come from the
Azure Education Hub student benefit, which covers education,
non-commercial research, and software development and testing.
Nothing in this environment is used for any other purpose.
