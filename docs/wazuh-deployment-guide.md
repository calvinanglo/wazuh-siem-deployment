# Wazuh Deployment Guide

Step-by-step setup for production Wazuh SIEM on VLAN 20 (10.10.20.10).

## Prerequisites

- Debian 11 or Ubuntu 22.04 LTS VM
- - 4GB RAM minimum, 50GB disk
  - - Static IP: 10.10.20.10 on VLAN 20
    - - Network connectivity to all devices (R1-CORE, SW-DIST, SW-ACC-1, SW-ACC-2, pfSense)
      - - Root or sudo access
       
        - ## Phase 1: Base VM Setup
       
        - 1. Install OS and basic tools:
          2.    sudo apt update && sudo apt upgrade -y
          3.   sudo apt install -y curl wget tar gzip net-tools
         
          4.   2. Set hostname:
               3.    sudo hostnamectl set-hostname wazuh-siem
               4.   echo "10.10.20.10 wazuh-siem" | sudo tee -a /etc/hosts
            
               5.   3. Disable firewall (or configure to allow ports 514/UDP and 1514/TCP):
                    4.    sudo systemctl stop ufw || sudo systemctl stop firewalld
                 
                    5.4. Verify connectivity to network devices:
                       ping 10.10.1.1 (R1-CORE)
                       ping 10.10.1.2 (SW-DIST)

                    ## Phase 2: Wazuh Manager Installation

                    1. Download latest Wazuh 4.7:
                    2.    cd /tmp
                    3.   wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-manager/wazuh-manager_4.7.0-1_amd64.deb
                 
                    4.   2. Install:
                         3.    sudo dpkg -i wazuh-manager_4.7.0-1_amd64.deb
                         4.   sudo systemctl daemon-reload
                      
                         5.   3. Enable and start service:
                              4.    sudo systemctl enable wazuh-manager
                              5.   sudo systemctl start wazuh-manager
                           
                              6.   4. Verify installation:
                                   5.    sudo systemctl status wazuh-manager
                                   6.   sudo /var/ossec/bin/wazuh-control info
                                
                                   7.   ## Phase 3: Configure Ossec Manager
                                
                                   8.   1. Copy ossec.conf from configs/:
                                        2.    sudo cp configs/ossec.conf /var/ossec/etc/
                                     
                                        3.2. Create syslog input listener (port 514/UDP):
                                           Verify configs/ossec.conf has:
                                           <remote>
                                                <connection>syslog</connection>
                                                     <port>514</port>
                                                          <protocol>udp</protocol>
                                                               <allowed-ips>10.10.0.0/16</allowed-ips>
                                                                  </remote>

                                                                  3. Load custom detection rules:
                                           sudo cp rules/custom-rules.xml /var/ossec/etc/rules/
                                           sudo chown root:ossec /var/ossec/etc/rules/custom-rules.xml

                                        4. Verify rules syntax:
                                        5.    sudo /var/ossec/bin/wazuh-control rule-test
                                     
                                        6.5. Restart manager:
                                           sudo systemctl restart wazuh-manager

                                        ## Phase 4: Configure Network Device Syslog Forwarding

                                        Apply configurations from configs/syslog-forwarding.txt to:
                                        - R1-CORE: logging 10.10.20.10
                                        - - SW-DIST: logging 10.10.20.10
                                          - - SW-ACC-1: logging 10.10.20.10
                                            - - SW-ACC-2: logging 10.10.20.10
                                              - - pfSense: System > System Logs > Remote Syslog Server 10.10.20.10
                                               
                                                - Test from each device:
                                                -   Device# debug condition syslog buffer 1 no-list
                                                -     Device# no debug all
                                                -   (generates test syslog entry)
                                               
                                                -   ## Phase 5: Verify Alert Pipeline
                                               
                                                -   1. Monitor incoming syslog:
                                                    2.    sudo tail -f /var/ossec/logs/alerts/alerts.json | jq '.[]'
                                                 
                                                    3.2. Trigger test alert (SSH brute force simulation):
                                                       ssh admin@10.10.99.1 (wrong password x10)

                                                    3. Check if alert appears:
                                                    4.    sudo grep "5001" /var/ossec/logs/alerts/alerts.json
                                                 
                                                    5.4. Validate alert structure:
                                                       sudo /var/ossec/bin/wazuh-control info

                                                    ## Phase 6: Wazuh Dashboard Installation

                                                    1. Install Wazuh Dashboard (UI for browsing alerts):
                                                    2.    wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-dashboard/wazuh-dashboard_4.7.0-1_amd64.deb
                                                    3.   sudo dpkg -i wazuh-dashboard_4.7.0-1_amd64.deb
                                                 
                                                    4.   2. Start dashboard:
                                                         3.    sudo systemctl enable wazuh-dashboard
                                                         4.   sudo systemctl start wazuh-dashboard
                                                      
                                                         5.   3. Access UI:
                                                              4.    https://10.10.20.10:443
                                                              5.   Username: admin
                                                              6.      Password: (default generated during install)
                                                           
                                                              7.  ## Phase 7: Alert Email Configuration
                                                           
                                                              8.  Edit /var/ossec/etc/ossec.conf email settings:
                                                              9.     <email_notification>yes</email_notification>
                                                              10.    <email_from>wazuh@10.10.20.10</email_from>
                                                                 <smtp_server>mail.company.com</smtp_server>

                                                                 Configure alert recipients:
                                                                 <email_alert_level>7</email_alert_level>

                                                                 Verify SMTP connectivity:
                                                                 telnet mail.company.com 25

                                                              Restart manager:
                                                                 sudo systemctl restart wazuh-manager

                                                              ## Phase 8: Metrics Export for Prometheus

                                                              Install Wazuh API exporter:
                                                                 sudo apt install -y python3-pip
                                                                 pip install wazuh-api-client

                                                              Configure Prometheus scrape (in Prometheus config):
                                                                 - job_name: 'wazuh'
                                                                 -      static_configs:
                                                                 -         - targets: ['10.10.20.10:1514']
                                                           
                                                                 -     ## Phase 9: Backup & Disaster Recovery
                                                           
                                                                 - 1. Backup Wazuh configuration and rules:
                                                                   2.    sudo tar -czf /backup/wazuh-backup-$(date +%Y%m%d).tar.gz /var/ossec/etc/
                                                                  
                                                                   3.2. Backup alerts database:
                                                                      sudo tar -czf /backup/wazuh-alerts-$(date +%Y%m%d).tar.gz /var/ossec/logs/

                                                                   3. Schedule daily backups:
                                                                   4.    0 2 * * * root tar -czf /backup/wazuh-backup-$(date +\%Y\%m\%d).tar.gz /var/ossec/etc/ >> /var/log/wazuh-backup.log 2>&1
                                                                  
                                                                   5.## Phase 10: Hardening & Maintenance

                                                                   1. Restrict file permissions:
                                                                   2.    sudo chown -R root:ossec /var/ossec/etc/
                                                                   3.   sudo chmod -R 550 /var/ossec/etc/
                                                                  
                                                                   4.   2. Enable log rotation:
                                                                        3.    Configure logrotate for /var/ossec/logs/
                                                                     
                                                                        4.3. Monthly security patching:
                                                                           sudo apt upgrade wazuh-manager

                                                                        4. Quarterly rule review:
                                                                        5.    Review custom-rules.xml for obsolete rules
                                                                        6.   Update rule thresholds based on false positive analysis
                                                                     
                                                                        7.   ## Troubleshooting
                                                                     
                                                                        8.   No alerts from devices:
                                                                        9.   1. Verify syslog forwarding configured on each device
                                                                             2. 2. Check firewall allows UDP 514 inbound
                                                                                3. 3. Monitor: sudo tcpdump -i eth0 -n udp port 514
                                                                                   4. 4. Restart manager: sudo systemctl restart wazuh-manager
                                                                                     
                                                                                      5. Alert not triggering:
                                                                                      6. 1. Verify rule syntax: sudo /var/ossec/bin/wazuh-control rule-test
                                                                                         2. 2. Check rule is loaded: sudo grep "custom-rules" /var/ossec/etc/ossec.conf
                                                                                            3. 3. Simulate alert: ssh admin@10.10.99.1 with wrong password
                                                                                               4. 4. Check logs: sudo tail -f /var/ossec/logs/ossec.log
                                                                                                 
                                                                                                  5. Connection refused on 10.10.20.10:
                                                                                                  6. 1. Verify Wazuh manager running: sudo systemctl status wazuh-manager
                                                                                                     2. 2. Check port listening: sudo netstat -tlnp | grep wazuh
                                                                                                        3. 3. Restart: sudo systemctl restart wazuh-manager
