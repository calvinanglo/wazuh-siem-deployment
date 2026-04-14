# Wazuh Deployment Guide

Step-by-step setup for production Wazuh SIEM on VLAN 20 (10.10.20.10). This covers the full deployment from base OS through agent enrollment and rule loading.

## Prerequisites

- Debian 11 or Ubuntu 22.04 LTS VM
- 4GB RAM minimum, 50GB disk
- Static IP: 10.10.20.10 on VLAN 20
- Network connectivity to R1-CORE, SW-DIST, SW-ACC-1, SW-ACC-2, and pfSense
- Root or sudo access

## Phase 1: Base VM Setup

Set the static IP and hostname before anything else. Wazuh agents use the manager's hostname for enrollment so changing it later breaks things.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget gnupg2 software-properties-common

# set hostname
sudo hostnamectl set-hostname wazuh-manager
echo "10.10.20.10 wazuh-manager" | sudo tee -a /etc/hosts

# disable unused services
sudo systemctl disable avahi-daemon cups bluetooth 2>/dev/null || true
```

Configure static IP in /etc/network/interfaces:

```
auto eth0
iface eth0 inet static
  address 10.10.20.10
  netmask 255.255.255.0
  gateway 10.10.20.1
  dns-nameservers 8.8.8.8 1.1.1.1
```

## Phase 2: Wazuh Manager Installation

Wazuh 4.7 is the target version. The install script handles the full stack including Elasticsearch-compatible indexer.

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.7/config.yml

# edit config.yml to set your node IPs before running
# nodes.indexer[0].ip = 10.10.20.10
# nodes.server[0].ip = 10.10.20.10
# nodes.dashboard[0].ip = 10.10.20.10

sudo bash wazuh-install.sh -a
```

After install, save the admin credentials printed at the end. Change them on first login.

Verify services are running:

```bash
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
```

## Phase 3: Firewall / Port Rules

Open necessary ports on the host firewall. pfSense handles inter-VLAN ACLs but the host firewall still needs these:

```bash
sudo ufw allow 1514/udp   # agent communication
sudo ufw allow 1515/tcp   # agent enrollment
sudo ufw allow 514/udp    # syslog from network devices
sudo ufw allow 55000/tcp  # wazuh API
sudo ufw allow 9200/tcp   # indexer (local only)
sudo ufw allow 443/tcp    # dashboard
sudo ufw enable
```

## Phase 4: Custom Rules

Load the custom rules from rules/custom-rules.xml into Wazuh. These cover rules 100001-100010 for network-specific detection.

```bash
sudo cp rules/custom-rules.xml /var/ossec/etc/rules/
sudo chown root:wazuh /var/ossec/etc/rules/custom-rules.xml
sudo chmod 660 /var/ossec/etc/rules/custom-rules.xml

# validate config before restarting
sudo /var/ossec/bin/wazuh-logtest

# apply
sudo systemctl restart wazuh-manager
```

Check rules loaded correctly:

```bash
sudo /var/ossec/bin/wazuh-control status
grep -r "rule id=\"100" /var/ossec/logs/ossec.log | tail -5
```

## Phase 5: Syslog Forwarding from Network Devices

Apply configs/syslog-forwarding.txt to each device. Devices send UDP 514 syslog to 10.10.20.10. Wazuh is already listening by default.

Verify syslog is arriving:

```bash
sudo tcpdump -i eth0 udp port 514 -nn -c 20
tail -f /var/ossec/logs/archives/archives.log | grep -E "R1-CORE|SW-DIST|SW-ACC"
```

## Phase 6: Agent Enrollment (Linux Hosts)

For any Linux hosts on the network that need agent-based monitoring:

```bash
# on the agent host
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo apt-key add -
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update && sudo apt install wazuh-agent

# set manager IP
sudo sed -i 's/MANAGER_IP/10.10.20.10/' /var/ossec/etc/ossec.conf
sudo systemctl enable --now wazuh-agent
```

Approve enrollment in the Wazuh dashboard under Agents > Pending.

## Phase 7: Validation

Run the alert injection tests in verification/alert-tests.md to confirm the full detection pipeline is working. Key checks:

- SSH brute force triggers rule 100001 within 60 seconds
- Port scan attempt triggers rule 100002
- Syslog from all 4 network devices is appearing in archives
- Dashboard is accessible at https://10.10.20.10

## Rollback

If the deployment needs to be undone:

```bash
sudo bash wazuh-install.sh -u  # uninstall all components
sudo ufw delete allow 1514/udp
sudo ufw delete allow 514/udp
```

Document the rollback in docs/itil-change-log.md under the original change record with reason and timestamp.
