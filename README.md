# Wazuh SIEM Deployment

A production-ready Wazuh setup integrated with the enterprise network from Project 1. Covers deployment, custom detection rules, incident response workflows, and ITIL-based change management.

## What This Is

This project demonstrates deploying Wazuh as a centralized SIEM for the network segmentation lab. Wazuh ingests syslog from all four network devices (R1-CORE, SW-DIST, SW-ACC-1, SW-ACC-2) and pfSense, enabling real-time threat detection and compliance monitoring.

The setup includes:
- Wazuh server and agent architecture
- Custom detection rules for network anomalies
- Incident response playbooks mapped to ITIL processes
- Integration with existing monitoring stack (Prometheus, syslog)

## Deployment Architecture

Wazuh server runs on VLAN 20 (10.10.20.10) alongside Prometheus. It receives syslog on UDP 514 from all network devices. The manager processes events, triggers alerts, and logs everything for forensics.

For network context, see the companion project: https://github.com/calvinanglo/enterprise-network-segmentation

## Getting Started

1. **Provision VM**: Debian 11 or Ubuntu 22.04, 4GB RAM minimum, 50GB disk, static IP on VLAN 20
2. **Install Wazuh Manager**: Follow wazuh-install.sh for automated setup
3. **Configure Syslog Forwarding**: Apply configs/syslog-forwarding.txt to each network device
4. **Define Detection Rules**: Load rules/custom-rules.xml into Wazuh manager
5. **Test Alert Pipeline**: Run verification/alert-tests.md to validate end-to-end detection

## File Structure

```
wazuh-siem-deployment/
  configs/
    ossec.conf                       # Wazuh manager config - syslog inputs, email alerts
    syslog-forwarding.txt            # IOS syslog config to apply on each Cisco device
  rules/
    custom-rules.xml                 # rules 5001-5006 for network security events
  playbooks/
    incident-response-workflows.md   # ITIL IR workflows for each alert type
  docs/
    wazuh-deployment-guide.md        # step-by-step install and config guide
    itil-change-log.md               # change records for all Wazuh modifications
  verification/
    alert-tests.md                   # alert injection tests to validate detection rules
```

## Skills Demonstrated

**Network Security (CCNA):** Syslog integration with segmented VLANs, port security evasion detection, unexpected spanning tree topology changes.

**Threat Detection (Security+):** Brute force SSH attempts, port scans, ACL violations, unusual traffic patterns, and privilege escalation indicators.

**Incident Management (ITIL):** Alert classification, incident assignment, escalation procedures, root cause analysis workflow, and change documentation for remediation.

## Custom Detection Rules

The rules/ directory includes patterns for:
- SSH brute force attempts (rule 5001)
- Port scan detection from unused interfaces (rule 5002)
- ACL violation logging (rule 5003)
- Unexpected spanning tree BPDUs (rule 5004)
- Guest VLAN reaching restricted subnets (rule 5005)
- Failed authentication on privileged accounts (rule 5006)

Each rule includes severity levels (critical, high, medium) and grouping for dashboard aggregation.

## ITIL Change Management

All modifications to Wazuh config, rules, or playbooks follow change control in docs/itil-change-log.md. Changes are categorized as Standard, Normal, or Emergency, with documented rollback procedures.

## Certifications this covers

CCNA 200-301 — Network monitoring, syslog protocol, VLAN inter-connectivity, routing protocols

CompTIA Security+ — Threat detection, vulnerability scanning, incident response, forensics

ISC2 CC — Security monitoring as a control for availability and integrity, incident response procedures

ITIL 4 — Incident management, change control, problem management, service continuity planning
