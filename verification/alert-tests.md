# Alert Injection and Verification Tests

Validate Wazuh detection rules end-to-end using the lab network topology from Project 1. Run these after any rule change or Wazuh upgrade. Expected results are listed for each test so you know what passing looks like.

---

## Test 1: SSH Brute Force (Rule 100001)

**Objective:** Verify rule 100001 fires on repeated SSH authentication failures.

**From a test VM on the guest VLAN (10.10.30.x):**

```bash
# generate 15 failed SSH attempts in quick succession
for i in $(seq 1 15); do
  ssh -o ConnectTimeout=3 -o StrictHostKeyChecking=no admin@10.10.20.10 2>/dev/null
done
```

**On Wazuh manager - watch for the alert:**

```bash
tail -f /var/ossec/logs/alerts/alerts.json | python3 -c "
import sys, json
for line in sys.stdin:
  try:
    a = json.loads(line)
    if a.get('rule', {}).get('id') == '100001':
      print(json.dumps(a, indent=2))
  except: pass
"
```

**Expected result:** Alert with rule.id 100001, level 7, group "authentication,ssh,brute_force". Should fire within 60 seconds of the 10th failure. Source IP should show 10.10.30.x.

---

## Test 2: Port Scan Detection (Rule 100002)

**Objective:** Verify port scan rule triggers on rapid connection attempts from unexpected sources.

```bash
# from a test host - simulate a port scan toward the server VLAN
nmap -sS -T4 --top-ports 100 10.10.20.0/24 2>&1 | head -20
```

**Check Wazuh:**

```bash
grep "rule_id.*100002" /var/ossec/logs/alerts/alerts.json | tail -5
```

**Expected result:** Alert with rule.id 100002, level 6, group "recon,port_scan". If scan came from guest VLAN, severity should be higher. Alert fires within 2 minutes of scan start.

---

## Test 3: ACL Violation Logging (Rule 100003)

**Objective:** Verify that Cisco ACL deny hits generate syslog and trigger Wazuh rule 100003.

On R1-CORE, ACL denies are logged via syslog. Try to reach a restricted destination from the guest VLAN:

```bash
# from a guest VLAN host - attempt connection to management VLAN
curl -m 3 http://10.10.99.1 2>&1
ping -c 3 10.10.20.10
```

**On Wazuh, verify syslog arrived and rule fired:**

```bash
grep "acl_violation|rule_id.*100003" /var/ossec/logs/alerts/alerts.json | tail -5

# also check the raw syslog archive
grep "ACLLOG|SEC-6-IPACCESSLOGP" /var/ossec/logs/archives/archives.log | tail -10
```

**Expected result:** Wazuh archives show the ACL hit syslog from R1-CORE (10.0.1.1). Rule 100003 alert appears with source 10.10.30.x and destination 10.10.20.x or 10.10.99.x.

---

## Test 4: Spanning Tree BPDU (Rule 100004)

**Objective:** Verify unexpected BPDU detection on access ports.

This test requires physically connecting a switch or sending a raw BPDU. In a GNS3/EVE-NG lab:

```
# on SW-ACC-1 - connect a second switch port to an access port
# STP will generate a topology change notification
# check switch log for: %SPANTREE-6-PORTDEL_ACCESS_PVST or %STP-3-BAD_BPDU
```

Alternatively, trigger a topology change by enabling then disabling a trunk:

```
SW-ACC-1# conf t
SW-ACC-1(config)# interface GigabitEthernet1/1
SW-ACC-1(config-if)# shutdown
SW-ACC-1(config-if)# no shutdown
```

**Expected result:** STP topology change syslog from SW-ACC-1 appears in Wazuh archives. Rule 100004 fires with level 8 if a new root bridge is elected, level 5 for a standard topology change.

---

## Test 5: Guest-to-Server VLAN Violation (Rule 100005)

**Objective:** Confirm cross-VLAN policy violation detection from the guest segment.

```bash
# from guest VLAN (10.10.30.x) - attempt to reach server VLAN directly
nmap -p 22,80,443,3306 10.10.20.0/24 --open 2>&1 | head -20
curl -m 3 http://10.10.20.10:55000  # wazuh API
```

**Expected result:** pfSense or R1-CORE drops the traffic and logs the ACL hit. Wazuh sees the syslog and fires rule 100005. Alert level 7, group "policy_violation,vlan,lateral_movement".

---

## Test 6: Privileged Account Failure (Rule 100006)

**Objective:** Verify privileged auth failure detection for sudo and enable-mode attempts.

```bash
# generate failed sudo attempts on Wazuh manager itself
sudo -u nobody ls / 2>/dev/null || true
su - root <<< "wrongpassword" 2>/dev/null || true
```

**Expected result:** Rule 100006 fires, level 8, group "authentication,privilege_escalation,admin". Should appear in alerts within 30 seconds.

---

## Syslog Connectivity Check

Before running alert tests, confirm all 4 network devices are sending syslog:

```bash
# check which sources are in the last 5 minutes of archives
grep "$(date +'%Y %b %d %H')" /var/ossec/logs/archives/archives.log | \
  grep -oP '\d+\.\d+\.\d+\.\d+' | sort | uniq -c | sort -rn | head -20
```

You should see traffic from 10.0.1.1 (R1-CORE), 10.10.99.1 (SW-DIST), 10.10.99.11 (SW-ACC-1), 10.10.99.12 (SW-ACC-2). If any are missing, check the syslog config on that device and verify UDP 514 is open in the inter-VLAN ACLs.
