# Incident Response Workflows

Mapped to ITIL Incident Management. Wazuh alerts trigger these workflows based on severity and rule type. Each playbook follows the standard detect → triage → contain → remediate → close cycle.

---

## IR-001: SSH Brute Force

**Triggered by:** Rule 5001 (Level 7)
**Alert fields:** authentication, ssh, brute_force

**Triage:**
Check if the source IP is authorized - VPN range, admin subnet, or a known backup host. More than 10 failures in under a minute from an unknown source is a real incident. Single failures or slow attempts from known IPs are usually just misconfigs.

**Containment:**
Block the source IP at the pfSense firewall using an alias-based block rule. Set a 1-hour expiry for unknown external IPs, indefinite for persistent internal threats. Notify the security channel with the source IP, targeted account, and attempt count.

**Investigation:**
```bash
# pull all failed auth events for the source IP
grep "$(SRCIP)" /var/ossec/logs/alerts/alerts.json | jq '.rule,.data' | head -50

# check what account was targeted
grep "$(SRCIP)" /var/log/auth.log | grep "Failed password" | awk '{print $9}' | sort | uniq -c

# see if other hosts are scanning at the same time (lateral movement indicator)
grep "brute_force" /var/ossec/logs/alerts/alerts.json | jq '.data.srcip' | sort | uniq -c | sort -rn | head -10
```

**Remediation:**
If source is an internal host, pull it off the network and image the disk before doing anything else. Run `rkhunter` and check for cron jobs, new SSH keys in ~/.ssh/authorized_keys, and any setuid binaries added recently. If external, the block rule is enough for now - document the IP in the threat log and check if it shows up in any threat intel feeds.

**ITIL Close:** Update incident ticket with timeline, containment action taken, and whether root cause was determined. If this is a repeat offense from the same IP range, raise a problem record.

---

## IR-002: Port Scan Detection

**Triggered by:** Rule 5002 (Level 6)
**Alert fields:** recon, port_scan, network

**Triage:**
Port scans from the guest VLAN (10.10.99.x) toward server or management VLANs are high priority - guests should not be scanning anything. Scans from within the same VLAN tier might be a misconfigured monitoring tool. Check if source IP is one of the Prometheus/blackbox exporters before escalating.

**Containment:**
If scan is from an unauthorized host, isolate the port via switch access port shutdown. Log the MAC address and port number.

**Investigation:**
```bash
# get scan profile from alert
grep "$(SRCIP)" /var/ossec/logs/alerts/alerts.json | jq '.data' | grep -E "dstport|dstip"

# check if source is a known scanner
cat /etc/wazuh-monitoring/known-scanners.txt | grep "$(SRCIP)"

# look at arp table to identify the host
arp -n | grep "$(SRCIP)"
```

**Remediation:**
Document the host and scan pattern. If the host is compromised, follow IR-001 steps for isolation. Update firewall rules to block the scan destination if it was targeting a sensitive port.

**ITIL Close:** Record source, targets, port range, and disposition. If scan was from a compromised guest device, raise a change request to tighten guest VLAN ACLs.

---

## IR-003: ACL Violation

**Triggered by:** Rule 5003 (Level 5)
**Alert fields:** firewall, acl, policy_violation

**Triage:**
Most ACL violations are misconfigs - a server trying to reach the wrong VLAN, or a DHCP scope that put a client in the wrong segment. Check whether the violation makes sense given the source host's role before treating it as an attack.

**Investigation:**
```bash
# find the full ACL hit context
grep "acl_violation" /var/ossec/logs/alerts/alerts.json | jq '.data' | grep -E "srcip|dstip|dstport" | tail -20

# verify current ACL on the relevant interface (from R1-CORE)
# show ip access-lists VLAN10-IN
# show ip access-lists VLAN20-IN
```

**Remediation:**
If misconfig: raise a Standard change to fix the ACL or VLAN assignment. If intentional violation from a compromised host: escalate to IR-001 flow for containment. Document ACL rule number and matched traffic pattern in the change ticket.

**ITIL Close:** Close with corrective action taken or escalation reference if escalated.

---

## IR-004: Unexpected STP BPDU

**Triggered by:** Rule 5004 (Level 8)
**Alert fields:** network, spanning_tree, topology_change

**Triage:**
Any unexpected BPDU on an edge access port is serious - it means either someone plugged in a rogue switch or there's a misconfigured trunk. Check which port and VLAN the BPDU came in on. A BPDU storm or new root bridge election is a P1 incident.

**Containment:**
Enable BPDU guard on the affected port immediately if it isn't already set. Shut the port if a rogue device is confirmed.

**Investigation:**
```bash
# correlate with switch syslog in wazuh
grep "SPANTREE" /var/ossec/logs/archives/archives.log | tail -30

# check current STP root for affected VLANs (on SW-DIST)
# show spanning-tree vlan 10
# show spanning-tree vlan 20
```

**Remediation:**
Re-enable portfast on edge ports. Audit all trunk ports to confirm they're terminating on expected switches. If a new switch was connected intentionally, raise a Normal change to add it to the documented topology.

**ITIL Close:** Confirm root bridge is correct, STP topology is stable, and affected ports are back in forwarding state.

---

## IR-005: Guest VLAN Reaching Restricted Subnet

**Triggered by:** Rule 5005 (Level 7)
**Alert fields:** policy_violation, vlan, lateral_movement

**Triage:**
Guest VLAN (10.10.99.x) should never reach VLAN 20 (servers) or VLAN 30 (management). Any alert here is either a routing misconfiguration or an active intrusion attempt. Check if the source IP is a guest device or something that got mis-assigned to the guest VLAN.

**Containment:**
Apply a deny ACE at the top of the inter-VLAN routing ACL immediately. Isolate the source host's switch port.

**Investigation:**
```bash
grep "guest_vlan" /var/ossec/logs/alerts/alerts.json | jq '.data.srcip, .data.dstip, .data.dstport' | tail -20
```

**Remediation:**
Verify that inter-VLAN ACLs on R1-CORE explicitly deny 10.10.99.0/24 to 10.10.20.0/24 and 10.10.30.0/24. If the ACL was missing or had a misconfigured permit, raise a Normal change to correct it and add a test case to the verification runbook.

**ITIL Close:** Confirm ACL is in place, test traffic from guest range is dropped, and close ticket with config change reference.

---

## IR-006: Privileged Account Failure

**Triggered by:** Rule 5006 (Level 8)
**Alert fields:** authentication, privilege_escalation, admin

**Triage:**
Failed sudo or enable-mode authentication on managed hosts or network devices. Single failures are probably typos. Three or more in quick succession from a non-admin source IP is an escalation.

**Containment:**
If the account is a service account, check if credentials were rotated recently and the service wasn't updated. If it's an admin account from an unusual source, lock the account temporarily and investigate.

**Investigation:**
```bash
# failed sudo attempts
grep "sudo" /var/log/auth.log | grep "FAILED" | tail -30

# failed enable on network devices (comes via syslog)
grep "AAA" /var/ossec/logs/archives/archives.log | grep "FAIL" | tail -20
```

**Remediation:**
Reset credentials through the proper change management process. If account compromise is confirmed, revoke all active sessions, rotate SSH keys, and audit recent command history on all systems the account had access to.

**ITIL Close:** Document the account, source IPs, and whether root cause was user error, misconfiguration, or confirmed compromise. If compromise, raise a problem record to track recurrence.

---

## Severity Reference

Level 3-4: Informational - log and review weekly
Level 5-6: Low/Medium - investigate within 4 hours, document findings
Level 7: High - investigate within 1 hour, notify security team
Level 8+: Critical - immediate response, escalate to senior admin

## Change Management

All containment actions that modify firewall rules, ACLs, or switch configs require a change record in docs/itil-change-log.md. Emergency changes (immediate containment) are documented within 2 hours of the action being taken. Normal changes (preventive hardening post-incident) follow the standard CAB review cycle.
