# Wazuh SIEM Deployment -- Setup Guide

Step-by-step instructions for deploying Wazuh 4.7 (manager + indexer + dashboard) on the segmented enterprise network from Project 1.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Base VM Setup on VLAN 20](#2-base-vm-setup-on-vlan-20)
3. [Wazuh 4.7 Installation](#3-wazuh-47-installation)
4. [Host Firewall Configuration](#4-host-firewall-configuration)
5. [Custom Rules Deployment](#5-custom-rules-deployment)
6. [Syslog Forwarding on Network Devices](#6-syslog-forwarding-on-network-devices)
7. [Agent Enrollment for Linux Hosts](#7-agent-enrollment-for-linux-hosts)
8. [Verification](#8-verification)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Prerequisites

Before starting this deployment, confirm the following are in place.

### Project 1 Must Be Complete

- [Enterprise Network Segmentation](https://github.com/calvinanglo/enterprise-network-segmentation) is fully deployed
- All four VLANs are operational: VLAN 10 (servers), VLAN 20 (management), VLAN 30 (guest), VLAN 99 (native)
- OSPF adjacencies are established between R1-CORE and all switches
- Inter-VLAN ACLs are applied and tested
- pfSense firewall is routing and filtering traffic between VLANs

### Network Devices Online

All five devices must be reachable and configured:

| Device    | Role              | Management IP |
|-----------|-------------------|---------------|
| R1-CORE   | Core router       | Loopback0     |
| SW-DIST   | Distribution      | VLAN 99       |
| SW-ACC-1  | Access switch 1   | VLAN 99       |
| SW-ACC-2  | Access switch 2   | VLAN 20       |
| pfSense   | Firewall/gateway  | VLAN 20       |

### VM Requirements

| Resource   | Minimum       |
|------------|---------------|
| OS         | Ubuntu 22.04 LTS or Debian 11 |
| RAM        | 4 GB          |
| Disk       | 50 GB         |
| CPU        | 2 vCPUs       |
| Network    | Bridged to VLAN 20 |
| Static IP  | 10.10.20.10   |

### Software

- SSH access to the VM
- `curl` and `apt` available on the VM
- Console or SSH access to each Cisco device and pfSense web UI

---

## 2. Base VM Setup on VLAN 20

### 2.1 Create the VM

Provision a new VM in your hypervisor (Proxmox, VMware, VirtualBox). Attach its network adapter to the VLAN 20 trunk or access port.

### 2.2 Set Hostname

```bash
sudo hostnamectl set-hostname wazuh-manager
```

Edit `/etc/hosts` to map the hostname:

```bash
sudo nano /etc/hosts
```

Add the line:

```
10.10.20.10   wazuh-manager
```

### 2.3 Configure Static IP

Edit the Netplan config (Ubuntu) or `/etc/network/interfaces` (Debian).

**Ubuntu 22.04 (Netplan):**

```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens18:
      addresses:
        - 10.10.20.10/24
      routes:
        - to: default
          via: 10.10.20.1
      nameservers:
        addresses:
          - 10.10.20.1
          - 8.8.8.8
```

Apply:

```bash
sudo netplan apply
```

**Debian 11:**

```bash
sudo nano /etc/network/interfaces
```

```
auto ens18
iface ens18 inet static
    address 10.10.20.10
    netmask 255.255.255.0
    gateway 10.10.20.1
    dns-nameservers 10.10.20.1 8.8.8.8
```

Restart networking:

```bash
sudo systemctl restart networking
```

### 2.4 Verify Connectivity

```bash
# Confirm IP assignment
ip addr show ens18

# Test gateway
ping -c 3 10.10.20.1

# Test internet (needed for Wazuh package download)
ping -c 3 packages.wazuh.com
```

### 2.5 Update the System

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 3. Wazuh 4.7 Installation

This section installs the all-in-one Wazuh stack: Wazuh Indexer, Wazuh Manager, and Wazuh Dashboard on a single node.

### 3.1 Download the Wazuh Install Assistant

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.7/config.yml
```

### 3.2 Edit config.yml

Replace the default values with the lab environment settings:

```bash
nano config.yml
```

Set the following (single-node deployment):

```yaml
nodes:
  indexer:
    - name: wazuh-indexer
      ip: 10.10.20.10

  server:
    - name: wazuh-manager
      ip: 10.10.20.10

  dashboard:
    - name: wazuh-dashboard
      ip: 10.10.20.10
```

### 3.3 Run the Installer

Generate certificates first, then install each component:

```bash
# Generate certificates
sudo bash wazuh-install.sh --generate-config-files

# Install the indexer
sudo bash wazuh-install.sh --wazuh-indexer wazuh-indexer

# Start the indexer cluster
sudo bash wazuh-install.sh --start-cluster

# Install the manager
sudo bash wazuh-install.sh --wazuh-server wazuh-manager

# Install the dashboard
sudo bash wazuh-install.sh --wazuh-dashboard wazuh-dashboard
```

### 3.4 Extract Default Credentials

The installer generates admin credentials in a tar file:

```bash
sudo tar -xvf wazuh-install-files.tar
cat wazuh-install-files/wazuh-passwords.txt
```

Record the `admin` user password. You will use this to log in to the dashboard.

### 3.5 Apply the Custom ossec.conf

Copy the project ossec.conf over the default manager config:

```bash
sudo cp /path/to/wazuh-siem-deployment/configs/ossec.conf /var/ossec/etc/ossec.conf
sudo chown root:wazuh /var/ossec/etc/ossec.conf
sudo chmod 640 /var/ossec/etc/ossec.conf
```

Restart the manager to load the new config:

```bash
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager
```

Confirm the manager is running and listening on UDP 514:

```bash
sudo ss -ulnp | grep 514
```

### 3.6 Verify All Services

```bash
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-dashboard
```

All three services must show `active (running)`.

> Screenshot: Wazuh dashboard login page showing the HTTPS login form at https://10.10.20.10
> Save as: docs/screenshots/01-wazuh-dashboard-login.png

---

## 4. Host Firewall Configuration

Configure UFW on the Wazuh server to allow only the traffic it needs.

### 4.1 Enable UFW

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

### 4.2 Allow Required Ports

```bash
# SSH management access
sudo ufw allow from 10.10.20.0/24 to any port 22 proto tcp comment "SSH from VLAN 20"

# Wazuh dashboard (HTTPS)
sudo ufw allow 443/tcp comment "Wazuh Dashboard"

# Wazuh agent registration
sudo ufw allow 1514/tcp comment "Wazuh agent registration"
sudo ufw allow 1515/tcp comment "Wazuh agent enrollment"

# Syslog from all network devices
sudo ufw allow from 10.10.0.0/16 to any port 514 proto udp comment "Syslog from network devices"

# Wazuh indexer (cluster communication, localhost only in single-node)
sudo ufw allow from 10.10.20.10 to any port 9200 proto tcp comment "Wazuh Indexer API"
```

### 4.3 Enable and Verify

```bash
sudo ufw enable
sudo ufw status verbose
```

Expected output should show all five rules active.

### 4.4 UFW Rules Summary

| Port     | Proto | Source            | Purpose                     |
|----------|-------|-------------------|-----------------------------|
| 22/tcp   | TCP   | 10.10.20.0/24     | SSH management              |
| 443/tcp  | TCP   | any               | Wazuh Dashboard             |
| 514/udp  | UDP   | 10.10.0.0/16      | Syslog from network devices |
| 1514/tcp | TCP   | any               | Wazuh agent communication   |
| 1515/tcp | TCP   | any               | Wazuh agent enrollment      |
| 9200/tcp | TCP   | 10.10.20.10       | Indexer API (local)         |

---

## 5. Custom Rules Deployment

### 5.1 Copy Custom Rules

Copy `rules/custom-rules.xml` from this repository to the Wazuh manager:

```bash
sudo cp /path/to/wazuh-siem-deployment/rules/custom-rules.xml /var/ossec/etc/rules/custom-rules.xml
sudo chown root:wazuh /var/ossec/etc/rules/custom-rules.xml
sudo chmod 640 /var/ossec/etc/rules/custom-rules.xml
```

### 5.2 Validate the Rules

Run the Wazuh rule test utility to check for syntax errors:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

At the prompt, paste a sample log line (such as a syslog message from R1-CORE) and confirm the custom rule ID appears in the output. Press Ctrl+C to exit.

You can also validate the full configuration:

```bash
sudo /var/ossec/bin/wazuh-control -t
```

This should output `Configuration OK` if the rules XML is well-formed.

### 5.3 Restart the Manager

```bash
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager
```

### 5.4 Confirm Rules Are Loaded

Check that all 10 custom rules (100001--100010) are loaded:

```bash
sudo /var/ossec/bin/wazuh-control info -r | grep "100"
```

Or verify through the dashboard under **Management > Rules > Custom Rules**.

> Screenshot: Wazuh dashboard showing custom rules loaded under Management > Rules, filtered to show rules 100001-100010
> Save as: docs/screenshots/02-custom-rules-loaded.png

### Custom Rules Reference

| Rule ID | Description                          | Severity |
|---------|--------------------------------------|----------|
| 100001  | SSH brute force attempts             | High     |
| 100002  | Port scan from unused interfaces     | High     |
| 100003  | ACL violation logging                | Medium   |
| 100004  | Unexpected spanning tree BPDUs       | Critical |
| 100005  | Guest VLAN reaching restricted subs  | Critical |
| 100006  | Failed auth on privileged accounts   | High     |
| 100007  | Port security violation              | Medium   |
| 100008  | Unauthorized device connection       | High     |
| 100009  | OSPF neighbor down                   | Critical |
| 100010  | Interface flap detection             | Medium   |

---

## 6. Syslog Forwarding on Network Devices

Apply the configurations from `configs/syslog-forwarding.txt` to each network device. All devices forward syslog to 10.10.20.10 on UDP 514.

### 6.1 R1-CORE (Cisco IOS Router)

Connect via console or SSH and enter the following:

```
configure terminal

logging enable
logging host 10.10.20.10
logging source-interface Loopback0
logging facility local6
logging buffered 8000
logging trap informational

end
write memory
```

### 6.2 SW-DIST (Cisco IOS Distribution Switch)

```
configure terminal

logging enable
logging host 10.10.20.10
logging source-interface Vlan99
logging facility local6
logging buffered 8000

end
write memory
```

### 6.3 SW-ACC-1 (Cisco IOS Access Switch)

```
configure terminal

logging enable
logging host 10.10.20.10
logging source-interface Vlan99
logging facility local6
logging buffered 4096

end
write memory
```

### 6.4 SW-ACC-2 (Cisco IOS Access Switch)

```
configure terminal

logging enable
logging host 10.10.20.10
logging source-interface Vlan20
logging facility local6
logging buffered 4096

end
write memory
```

### 6.5 pfSense (Web UI)

1. Navigate to **System > System Logs > Settings**
2. Set **Syslog Server** to `10.10.20.10`
3. Set **Syslog Protocol** to UDP (RFC 3164)
4. Set **Syslog Facility** to Local6
5. Check **Send output to remote syslog server**
6. Under **Services > DHCP Server**, enable syslog forwarding for all pools
7. Under **System > Firewall & NAT > Advanced**, check **Log firewall default blocks**

### 6.6 Verify Syslog Is Arriving

On the Wazuh manager, watch for incoming syslog:

```bash
tail -f /var/ossec/logs/alerts/alerts.json | grep source_ip
```

You should see entries from R1-CORE, SW-DIST, SW-ACC-1, SW-ACC-2, and pfSense within a few minutes. Generate traffic if needed (e.g., bounce an interface, trigger an ACL deny).

> Screenshot: Wazuh dashboard showing syslog sources from all five network devices under the Events or Agents view
> Save as: docs/screenshots/03-syslog-sources-visible.png

---

## 7. Agent Enrollment for Linux Hosts

Enroll any Linux hosts on the network as Wazuh agents. This covers the Wazuh manager itself (local agent) and any additional Linux VMs.

### 7.1 Install the Wazuh Agent

On each Linux host:

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt update
sudo WAZUH_MANAGER="10.10.20.10" apt install wazuh-agent -y
```

### 7.2 Start the Agent

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo systemctl status wazuh-agent
```

### 7.3 Verify Enrollment on the Manager

Check the agent list on the manager:

```bash
sudo /var/ossec/bin/agent_control -l
```

The newly enrolled agent should appear with status `Active`.

You can also verify through the dashboard under **Agents**.

> Screenshot: Wazuh dashboard Agents page showing the list of enrolled agents with their status, OS, and IP
> Save as: docs/screenshots/04-agent-list.png

---

## 8. Verification

Run through each of these checks to confirm the deployment is fully operational.

### 8.1 Dashboard Access

Open a browser and navigate to:

```
https://10.10.20.10
```

Log in with the admin credentials extracted in step 3.4. You should see the Wazuh overview dashboard with agent counts and recent alerts.

> Screenshot: Wazuh dashboard overview page after login, showing agent summary and alert counts
> Save as: docs/screenshots/01-wazuh-dashboard-login.png

### 8.2 Syslog Arriving from All Devices

In the dashboard, navigate to **Security Events** and filter by source. Confirm you see events from all five syslog sources:

- R1-CORE
- SW-DIST
- SW-ACC-1
- SW-ACC-2
- pfSense

If events are missing, check the device syslog config and the UFW rule for port 514.

### 8.3 Alert Injection Tests

Run these tests from a separate machine on the network to generate detectable events.

**SSH Brute Force (triggers rule 100001):**

```bash
# From another VLAN 20 host, attempt multiple failed SSH logins
for i in $(seq 1 10); do
    sshpass -p 'wrongpassword' ssh -o StrictHostKeyChecking=no baduser@10.10.20.10 2>/dev/null
done
```

**Port Scan (triggers rule 100002):**

```bash
# From any host, run a basic nmap scan
nmap -sS -p 1-1000 10.10.20.10
```

After running these tests, wait 1-2 minutes and check the Wazuh dashboard under **Security Events**. Filter by rule ID 100001 and 100002.

> Screenshot: Wazuh dashboard Security Events page showing active alerts triggered by the SSH brute force and port scan tests
> Save as: docs/screenshots/05-active-alerts.png

### 8.4 Custom Rules Firing

Verify custom rules are generating alerts by checking the alerts log:

```bash
sudo cat /var/ossec/logs/alerts/alerts.json | python3 -m json.tool | grep -A5 "rule_id.*1000"
```

In the dashboard, navigate to **Management > Rules** and confirm all custom rules (100001--100010) appear in the loaded rules list.

Generate additional test events as needed:

- **ACL violation (100003):** Send traffic that hits a deny ACL on R1-CORE
- **Port security violation (100007):** Connect an unauthorized MAC to SW-ACC-1
- **OSPF neighbor down (100009):** Shut down an OSPF interface temporarily

### 8.5 Verification Checklist

| Check                              | Expected Result                              | Pass |
|------------------------------------|----------------------------------------------|------|
| Dashboard loads at https://10.10.20.10 | Login page appears, admin credentials work | [ ]  |
| All three services running         | wazuh-indexer, wazuh-manager, wazuh-dashboard active | [ ]  |
| Syslog from R1-CORE                | Events visible in Security Events            | [ ]  |
| Syslog from SW-DIST                | Events visible in Security Events            | [ ]  |
| Syslog from SW-ACC-1               | Events visible in Security Events            | [ ]  |
| Syslog from SW-ACC-2               | Events visible in Security Events            | [ ]  |
| Syslog from pfSense                | Events visible in Security Events            | [ ]  |
| Agent enrolled and active          | Agent appears in Agents list                 | [ ]  |
| SSH brute force alert fires        | Rule 100001 triggers in alerts               | [ ]  |
| Port scan alert fires              | Rule 100002 triggers in alerts               | [ ]  |
| Custom rules 100001-100010 loaded  | All visible under Management > Rules         | [ ]  |
| UFW rules active                   | `ufw status` shows all expected rules        | [ ]  |

---

## 9. Troubleshooting

### Wazuh Manager Will Not Start

```bash
# Check the logs for errors
sudo cat /var/ossec/logs/ossec.log | tail -50

# Validate the configuration
sudo /var/ossec/bin/wazuh-control -t

# Common fix: permissions on ossec.conf or custom-rules.xml
sudo chown root:wazuh /var/ossec/etc/ossec.conf
sudo chown root:wazuh /var/ossec/etc/rules/custom-rules.xml
sudo chmod 640 /var/ossec/etc/ossec.conf
sudo chmod 640 /var/ossec/etc/rules/custom-rules.xml
```

### No Syslog Events Arriving

```bash
# Confirm UDP 514 is open and listening
sudo ss -ulnp | grep 514

# Check UFW is not blocking
sudo ufw status verbose | grep 514

# Test from a device manually
# On R1-CORE:
#   ping 10.10.20.10
# If ping fails, check VLAN routing and ACLs in Project 1

# Watch raw syslog on the wire
sudo tcpdump -i ens18 udp port 514 -c 20
```

### Dashboard Not Loading (HTTPS Timeout)

```bash
# Verify the dashboard service
sudo systemctl status wazuh-dashboard

# Check if port 443 is bound
sudo ss -tlnp | grep 443

# Restart if needed
sudo systemctl restart wazuh-dashboard

# Check dashboard logs
sudo cat /var/log/wazuh-dashboard/opensearch_dashboards.log | tail -30
```

### Agent Not Connecting

```bash
# On the agent host, check agent status
sudo systemctl status wazuh-agent

# Verify the manager IP is correct in the agent config
sudo cat /var/ossec/etc/ossec.conf | grep "<address>"

# Test connectivity to the manager
nc -zv 10.10.20.10 1514
nc -zv 10.10.20.10 1515

# On the manager, check for enrollment requests
sudo cat /var/ossec/logs/ossec.log | grep "agent"
```

### Custom Rules Not Triggering

```bash
# Confirm the rules file is in the correct location
ls -la /var/ossec/etc/rules/custom-rules.xml

# Test a rule manually with wazuh-logtest
sudo /var/ossec/bin/wazuh-logtest
# Paste a sample log line and check if the custom rule ID appears

# Check the rules are loaded in memory
sudo /var/ossec/bin/wazuh-control info -r | grep "1000"

# If rules are not loaded, restart the manager
sudo systemctl restart wazuh-manager
```

### Indexer Health Issues

```bash
# Check indexer status
sudo systemctl status wazuh-indexer

# Query the indexer health API
curl -k -u admin:PASSWORD https://10.10.20.10:9200/_cluster/health?pretty

# Check disk space (indexer needs room for logs)
df -h /var/lib/wazuh-indexer
```

### Useful Log Locations

| Log File                                          | Purpose                          |
|---------------------------------------------------|----------------------------------|
| `/var/ossec/logs/ossec.log`                       | Manager operational log          |
| `/var/ossec/logs/alerts/alerts.json`              | Alert output (JSON)              |
| `/var/ossec/logs/alerts/alerts.log`               | Alert output (plain text)        |
| `/var/log/wazuh-indexer/wazuh-indexer.log`        | Indexer log                      |
| `/var/log/wazuh-dashboard/opensearch_dashboards.log` | Dashboard log                 |

---

## Screenshot Reference

| # | Description | File |
|---|-------------|------|
| 1 | Wazuh dashboard login page | `docs/screenshots/01-wazuh-dashboard-login.png` |
| 2 | Custom rules loaded (100001-100010) | `docs/screenshots/02-custom-rules-loaded.png` |
| 3 | Syslog sources from all network devices | `docs/screenshots/03-syslog-sources-visible.png` |
| 4 | Agent list showing enrolled hosts | `docs/screenshots/04-agent-list.png` |
| 5 | Active alerts from injection tests | `docs/screenshots/05-active-alerts.png` |
