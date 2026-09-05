# AD Lab — Troubleshooting Log

Issues encountered building the Windows Server Active Directory lab
(DC01 + CLIENT01 on an isolated KVM network, `lab.internal`).

Format for each entry: symptom, hypotheses considered, evidence
gathered, root cause, fix, prevention.

---

## Issue 1: virt-manager would not expose the DHCP setting when creating a virtual network

**Symptom**

Creating the `adlab` virtual network through the virt-manager GUI
wizard, the dialog offered a name and a mode but no IPv4 subnet
field and no way to disable DHCP. The build requires DHCP to be
off on this network.

**Hypotheses**

- The options were hidden behind an advanced or expandable section.
- The wizard required the network to be created first and edited after.
- The installed virt-manager version had simplified the wizard.

**Evidence**

No advanced section existed in the dialog, and the created network
could not be edited to add the missing settings. `virsh net-dumpxml`
on the auto-created `default` network showed the full schema,
including the `<dhcp>` element the wizard was not surfacing —
confirming the capability existed at the libvirt layer and was
simply absent from the GUI.

**Root cause**

Newer virt-manager builds simplified the add-network wizard and
removed the IPv4 and DHCP configuration steps. The underlying
libvirt network definition still supports them; only the GUI path
was gone.

**Fix**

Defined the network directly from XML instead of the GUI:

```xml
<network>
  <name>adlab</name>
  <bridge name='virbr-adlab' stp='on' delay='0'/>
  <ip address='192.168.10.1' netmask='255.255.255.0'>
  </ip>
</network>
```

```
virsh net-define ~/adlab-net.xml
virsh net-start adlab
virsh net-autostart adlab
```

Two absences make this definition correct: no `<forward>` element,
which is what makes the network isolated, and no `<dhcp>` block
inside `<ip>`, so libvirt hands out no addresses.

**Why this mattered**

This was not cosmetic. Had libvirt been serving DHCP on this
network, CLIENT01 would have received libvirt's DNS server rather
than the domain controller's. The client would then have been
unable to resolve the `_ldap._tcp.dc._msdcs.lab.internal` SRV
record, and the domain join would have failed with a vague
"the domain could not be contacted" error several steps later —
far from the actual cause.

**Prevention**

Verify a virtual network from `virsh net-dumpxml` rather than
trusting the GUI wizard produced what was intended. Confirm the
absence of `<forward>` and `<dhcp>` before building anything on
top of it.

---

## Issue 2: dcdiag reported failed tests on a domain controller that was actually healthy

**Symptom**

Immediately after promotion, `dcdiag /q` reported:

```
DC01 failed test DFSREvent
DC01 failed test SystemLog
```

**Hypotheses**

- SYSVOL replication was genuinely broken.
- The promotion had partially failed.
- DNS was misconfigured on the domain controller.
- The tests were reporting on something other than current state.

**Evidence**

Read the actual event entries dcdiag surfaced rather than trusting
the pass/fail summary. They were:

- `netprofm` (Network List Service) terminated — the service
  restarted itself.
- Printer Extensions and Notifications marked as an interactive
  service — cosmetic, appears on stock Server installs.
- Group Policy processing failed "because of lack of network
  connectivity to a domain controller", timestamped *during* the
  promotion reboot.
- A later run showed a failed name resolution for
  `watson.events.data.microsoft.com`, a Microsoft telemetry host.

Meanwhile the checks that test actual function all passed:

- `nslookup lab.internal` → 192.168.10.10
- `nslookup -type=SRV _ldap._tcp.dc._msdcs.lab.internal` →
  dc01.lab.internal, port 389
- dcdiag ran partition tests against ForestDnsZones,
  DomainDnsZones, Schema, Configuration, and lab — all five AD
  naming contexts present, meaning the directory built completely.

**Root cause**

DFSREvent and SystemLog do not test whether anything works. They
scan the last 24 hours of event logs and fail if they find any
error entry. Two distinct classes of harmless entry triggered them:

1. Errors generated *during* promotion, before AD existed — the
   Group Policy entry was the server reporting that it could not
   reach a domain controller at a moment when it was not yet one.
2. Failed external DNS lookups from Windows telemetry. The lab
   network is isolated with no forwarders configured, so every
   telemetry lookup fails and logs an error. This recurs
   indefinitely and never indicates an AD fault.

**Fix**

No remediation required. Verified health through function rather
than through log scraping. For a clean demonstration run,
`Clear-EventLog -LogName System` followed by `gpupdate /force`
removes the stale pre-promotion entries.

**Prevention**

Distinguish tests that measure function from tests that scrape
logs. When a log-scraping test fails, read the underlying events
before treating it as an incident. Note also that clearing a log
to produce a clean test result is acceptable in a lab and is not
appropriate in production, where the entries would be investigated.

---

## Issue 3: nslookup returned "Server: UnKnown" and a DNS timeout on every query

**Symptom**

Every `nslookup` run on DC01 printed:

```
DNS request timed out.
    timeout was 2 seconds.
Server:  UnKnown
Address:  ::1
```

The queries themselves still returned correct answers, but the
header showed a timeout and an unresolved server name.

**Hypotheses**

- No reverse lookup zone existed for 192.168.10.0/24, so the DNS
  server's own address could not be resolved back to a name.
- The DNS service was failing.
- The adapter's preferred DNS server was set incorrectly.

**Evidence**

`ipconfig /all` confirmed the adapter's DNS server was
192.168.10.10 and nothing else — correct.

Created the reverse zone and the PTR record. `Get-DnsServerZone`
confirmed `10.168.192.in-addr.arpa` existed, was
`IsReverseLookupZone: True` and `IsDsIntegrated: True`. The
symptom persisted unchanged.

The decisive detail was in the nslookup output itself:
`Address: ::1`. The query was going to the IPv6 loopback, not to
192.168.10.10.

**Root cause**

nslookup was resolving through IPv6 loopback. No IPv4 reverse
lookup zone can ever resolve `::1` to a hostname, so the reverse
zone — although correctly created and genuinely useful — could not
fix this particular symptom. The initial hypothesis was reasonable
but incomplete: the missing reverse zone was a real gap, just not
the cause of what was on screen.

**Fix**

Two options, both verified:

1. Query the IPv4 address explicitly —
   `nslookup lab.internal 192.168.10.10` — which returned
   `Server: dc01.lab.internal`, confirming the reverse zone was
   working correctly.
2. Disable IPv6 on the adapter so plain nslookup uses IPv4:

```
Disable-NetAdapterBinding -Name "Ethernet" -ComponentID ms_tcpip6
```

After this, `nslookup lab.internal` resolved instantly with no
timeout.

**Prevention and caveat**

Read the `Address:` line in nslookup output before diagnosing a
name resolution problem — it states which server was actually
queried, which is frequently not the one assumed.

Disabling IPv6 on a domain controller is a lab convenience taken
here to produce clean output. It is not a production practice:
Microsoft supports IPv6 on domain controllers and disabling it can
cause problems in real environments. The explicit-server form of
the command achieves the same clarity without changing the
configuration, and would be the correct choice on a live system.

---

## Pattern across all three

Each of these presented as one thing and was caused by another.
The GUI appeared to be missing a feature that actually existed
underneath it. A health test reported failure on a healthy system.
A name resolution symptom pointed at a missing zone that was
genuinely missing but not responsible.

In all three, the resolution came from reading the specific
evidence — the XML, the underlying event entries, the address
nslookup actually queried — rather than acting on the summary.
