# ITIL Change Management Log

All modifications to Wazuh configuration, detection rules, or infrastructure require change control documentation per ITIL v4 Change Management process.

## Change Summary

| CHG-ID | Type | Date | Description | Status |
|--------|------|------|-------------|--------|
| CHG-001 | Standard | 2026-04-04 | Wazuh SIEM initial deployment | Approved |
| CHG-002 | Standard | 2026-04-05 | Syslog forwarding configuration from network devices | Approved |
| CHG-003 | Standard | 2026-04-06 | Custom detection rules deployment | Approved |
| CHG-004 | Standard | 2026-04-08 | Email alerting configuration for critical events | Approved |
| CHG-005 | Standard | 2026-04-10 | Prometheus metrics export integration | Approved |
| CHG-006 | Normal | 2026-04-15 | Detection rule threshold tuning (false positive reduction) | Pending CAB |

## Detailed Changes

### CHG-001: Wazuh SIEM Initial Deployment

Category: Standard Change (pre-approved infrastructure setup)
Date: April 4, 2026
Requester: Security Team
Environment: Production (VLAN 20, 10.10.20.10)

What's Changing:
- Deploy Wazuh SIEM server on dedicated Ubuntu 22.04 VM
- Configure ossec.conf for syslog ingestion on UDP 514
- Enable alerting to security team via email

Impact:
- Network traffic: Syslog from 4 devices + pfSense to central server (low volume ~1MB/day)
- System load: <5% CPU, 200MB RAM (well within 4GB allocation)
- No impact to production network traffic

Risk: Low
- Wazuh runs on isolated VLAN 20 management subnet
- Read-only syslog collection, no configuration changes to network devices

Rollback:
- VM shutdown, revert syslog configs on devices to send to /dev/null
- Takes <30 minutes

Verification:
- Alert pipeline end-to-end: ssh brute force test triggers Rule 100001
- All device syslog received: check alerts.json for source IPs 10.10.1.1, 10.10.1.2, etc.

### CHG-002: Syslog Forwarding Configuration

Category: Standard Change
Date: April 5, 2026
Requester: Network Engineering

What's Changing:
- Configure R1-CORE, SW-DIST, SW-ACC-1, SW-ACC-2 to forward syslog to 10.10.20.10:514
- Configure pfSense System > System Logs to remote syslog server 10.10.20.10
- All devices use facility local6, severity info (6)

Impact:
- Network traffic: ~5KB/sec per device during normal operation
- Config change on 5 devices, no downtime (syslog is best-effort UDP)

Risk: Low
- Syslog is diagnostic only, loss doesn't break traffic flow
- Firewall already allows UDP 514 from VLAN 10,20,30,99 to VLAN 20

Rollback:
- Remove logging commands from each device
- Takes <10 minutes per device

Verification:
- tcpdump on Wazuh server: tcpdump -i eth0 -n udp port 514
- Observe syslog packets from all 5 sources

### CHG-003: Custom Detection Rules Deployment

Category: Standard Change
Date: April 6, 2026
Requester: Security Team

What's Changing:
- Copy custom-rules.xml to /var/ossec/etc/rules/ on Wazuh manager
- Load rules: rules 100001-100010 for network security, authentication, topology

Rules Deployed:
- 100001: SSH brute force (level 7)
- 100002: Port scan detection (level 5)
- 100003: ACL violation (level 6)
- 100004: STP topology change (level 7)
- 100005: Guest VLAN unauthorized access (level 8)
- 100006: Privileged account auth failure (level 8)
- 100007-100010: Port security, MAC, OSPF, interface events

Impact:
- No impact during deployment (rules loaded on next service restart)
- Alert volume: ~10-50 per day during normal operations

Risk: Medium
- False positives possible if thresholds too sensitive
- Mitigation: Thresholds tuned conservatively, monitoring dashboard configured

Rollback:
- Remove custom-rules.xml, restart Wazuh
- Takes <5 minutes

Verification:
- Rule syntax: /var/ossec/bin/wazuh-control rule-test
- Alert triggers: SSH test generates Rule 100001 alert within 30 seconds

### CHG-004: Email Alerting for Critical Events

Category: Standard Change
Date: April 8, 2026
Requester: Security Team

What's Changing:
- Configure email notification for alerts level 7 and above (high/critical)
- Email recipients: security-team@company.com, ciso@company.com
- SMTP server: mail.company.com

Recipients:
- Level 8 (critical): CISO, VP of Security, Incident Commander
- Level 7 (high): Security Engineer on-call, SOC team lead

Impact:
- Email traffic: ~5-10 emails/day during normal ops
- SMTP dependency: If mail.company.com down, alerts queue locally

Risk: Low
- Email is supplementary to dashboard monitoring
- Alert delivery not guaranteed but fallback is dashboard

Rollback:
- Disable email in ossec.conf, restart Wazuh
- Takes <5 minutes

Verification:
- Trigger test alert (SSH brute force x5 attempts)
- Verify email received within 1 minute
- Check email headers for Wazuh SIEM origin

### CHG-005: Prometheus Metrics Export

Category: Standard Change
Date: April 10, 2026
Requester: DevOps / Monitoring Team

What's Changing:
- Install Wazuh API client on Wazuh server
- Export metrics to Prometheus on port 9090
- Metrics: Alert count, rules triggered, source IPs

Impact:
- Network traffic: ~100KB/min to Prometheus (negligible)
- Prometheus scrape job configured: 10s intervals

Metrics Available:
- wazuh_alerts_total (counter of all alerts)
- wazuh_alerts_by_level (100001=7, 100002=5, etc.)
- wazuh_syslog_sources (active source count)

Risk: Low
- Read-only API access, no configuration changes
- If metrics export fails, alerts still function

Rollback:
- Stop Wazuh API client, remove Prometheus scrape job
- Takes <5 minutes

Verification:
- curl http://10.10.20.10:1514/metrics (should show Wazuh metrics)
- Prometheus dashboard queries: wazuh_alerts_by_level

### CHG-006: Detection Rule Threshold Tuning

Category: Normal Change (requires CAB approval)
Date: April 15, 2026 (Pending)
Requester: Security Team

What's Changing:
- Reduce Rule 100001 (SSH brute force) threshold from 5 attempts to 3 attempts
- Reduce false alert rate on Rule 100002 (port scan) by filtering localhost scans
- Increase alert suppression window to 15 minutes to reduce email spam

Justification:
- Initial 2-week monitoring showed 40% false positives on Rule 100002
- SSH threshold too high, missing early attack attempts
- Email alerting generating 200+ daily messages (alert fatigue)

CAB Notes:
- Pending Change Advisory Board review
- Expected approval: April 20, 2026
- Implementation window: April 22, 2026, 10 PM (maintenance window)

Impact Assessment:
- Alert volume will increase initially (more true positives caught)
- Email volume will decrease due to suppression window
- Net effect: Better detection, less alert noise

Rollback Plan:
- Revert to original thresholds in custom-rules.xml
- Takes <10 minutes
- Decision to rollback if false positive rate >15%

Risk: Medium
- Could miss attacks if thresholds tuned too high
- Mitigation: Dashboard monitored 24/7 during test period

Verification:
- Monitor alert count for 24 hours after change
- Review false positive feedback from on-call engineer
- Adjust thresholds based on data
