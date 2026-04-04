# Incident Response Workflows

Mapped to ITIL Incident Management process. Alerts from Wazuh trigger these workflows based on severity and alert type.

## IR-001: SSH Brute Force Response

Triggered by: Rule 5001 (Level 7)
Alert Type: authentication, ssh, brute_force

Immediate Actions:
1. Identify source IP from alert: $(SRCIP)
2. 2. Check if source is authorized (VPN, admin subnet, known backup host)
   3. 3. If not authorized: Block source IP at firewall for 1 hour
      4. 4. Notify security team via Slack/email
        
         5. Investigate:
         6. - Who owns the source system? (DNS reverse lookup, ARP table)
            - - How many failed attempts? (grep $(SRCIP) /var/ossec/logs/alerts/alerts.json)
              - - Which user account targeted? (extract from auth logs)
                - - Are other hosts scanning concurrently?
                 
                  - Remediate:
                  - - If internal system compromised: Isolate host, rebuild OS, audit for persistence
                    - - If external attacker: File incident record in ITSM system
                      - - Update firewall rules to prevent future access from that source range
                       
                        - Verify:
                        - - SSH logs show no more attempts from source
                          - - Firewall block rule active
                            - - Incident ticket documented with root cause
                             
                              - Change Control: If implementing firewall rule change, file CHG ticket for next maintenance window
                             
                              - ## IR-002: Port Scan Detection Response
                             
                              - Triggered by: Rule 5002 (Level 5)
                              - Alert Type: network, scan, reconnaissance
                             
                              - Immediate Actions:
                              - 1. Capture full packet capture of scanning traffic (tcpdump)
                                2. 2. Identify scanning source and target ports
                                   3. 3. Determine intent (service fingerprinting, vulnerability assessment, lateral movement prep)
                                     
                                      4. Investigate:
                                      5. - Is this a known scan (e.g., external vulnerability assessor, internal admin tool)?
                                         - - What ports are being targeted?
                                           - - Is there follow-up exploitation activity?
                                             - - Check if firewall rules block unexpected port attempts
                                              
                                               - Remediate:
                                               - - If internal authorized scan: White-list source, update documentation
                                                 - - If unknown: Block source, escalate to security team
                                                   - - Review port exposure: Do we really need those services exposed?
                                                    
                                                     - Notify:
                                                     - - Infrastructure team if legitimate service exposure detected
                                                       - - Security incident response team if suspicious
                                                        
                                                         - ## IR-003: ACL Violation Response
                                                        
                                                         - Triggered by: Rule 5003 (Level 6)
                                                         - Alert Type: network, security, acl
                                                        
                                                         - Immediate Actions:
                                                         - 1. Extract source/dest IP and violated ACL rule number from alert
                                                           2. 2. Check if violation is expected (e.g., guest user trying to reach server VLAN)
                                                              3. 3. If unexpected: Investigate compromised host immediately
                                                                
                                                                 4. Investigate:
                                                                 5. - Is source host compromised? (Check for other alerts from same host)
                                                                    - - Is this a config error (wrong VLAN assignment)?
                                                                      - - Is the ACL rule effective or does traffic still flow?
                                                                       
                                                                        - Remediate:
                                                                        - - If host is compromised: Isolate from network, preserve logs, initiate incident
                                                                          - - If ACL rule needs adjustment: File CHG ticket with engineering
                                                                            - - If host misconfigured: Correct port VLAN assignment, re-run network audit
                                                                             
                                                                              - Document:
                                                                              - - Record ACL rule effectiveness
                                                                                - - Update network diagram if VLAN topology changed
                                                                                 
                                                                                  - ## IR-004: Spanning Tree Topology Change
                                                                                 
                                                                                  - Triggered by: Rule 5004 (Level 7)
                                                                                  - Alert Type: network, spanning_tree, topology
                                                                                 
                                                                                  - Immediate Actions:
                                                                                  - 1. Check current topology: sh spanning-tree root on affected switch
                                                                                    2. 2. Confirm expected root bridge
                                                                                       3. 3. If unexpected: Check for new switch connected to network (rogue device?)
                                                                                         
                                                                                          4. Investigate:
                                                                                          5. - Is there a new switch on the network?
                                                                                             - - Did someone upgrade/reboot a core switch?
                                                                                               - - Are BPDU Guard settings working correctly?
                                                                                                
                                                                                                 - Remediate:
                                                                                                 - - If rogue device: Disconnect immediately, audit access controls
                                                                                                   - - If planned change: Update change log, verify ports configured correctly
                                                                                                     - - Check port-fast, BPDU guard configs on all access switches
                                                                                                      
                                                                                                       - Verify:
                                                                                                       - - Root election completed properly
                                                                                                         - - All ports in expected forwarding state
                                                                                                           - - No loops detected
                                                                                                            
                                                                                                             - ## IR-005: Guest VLAN Unauthorized Access
                                                                                                            
                                                                                                             - Triggered by: Rule 5005 (Level 8 - Critical)
                                                                                                             - Alert Type: network, vlan, policy_violation
                                                                                                            
                                                                                                             - Immediate Actions:
                                                                                                             - 1. Identify source MAC/IP: $(SRCIP)
                                                                                                               2. 2. Locate switch port where guest device connected
                                                                                                                  3. 3. Immediately disable port if access should not be allowed
                                                                                                                     4. 4. Escalate to security: Unauthorized access to restricted resources
                                                                                                                       
                                                                                                                        5. Investigate:
                                                                                                                        6. - What host is this? (Reverse DNS, network inventory lookup)
                                                                                                                           - - How did it get assigned to guest VLAN?
                                                                                                                             - - Did DHCP incorrectly assign it?
                                                                                                                               - - Is this a compromised device or intentional breach attempt?
                                                                                                                                
                                                                                                                                 - Remediate:
                                                                                                                                 - - Contain the compromised device immediately
                                                                                                                                   - - Force port shut down or VLAN isolation
                                                                                                                                     - - File critical incident ticket
                                                                                                                                       - - Audit DHCP snooping and port security configs
                                                                                                                                        
                                                                                                                                         - Escalate:
                                                                                                                                         - - This is a policy violation requiring immediate response
                                                                                                                                           - - Notify security leadership
                                                                                                                                             - - Preserve all logs for forensic analysis
                                                                                                                                              
                                                                                                                                               - ## IR-006: Failed Privileged Account Authentication
                                                                                                                                              
                                                                                                                                               - Triggered by: Rule 5006 (Level 8 - Critical)
                                                                                                                                               - Alert Type: authentication, privilege_escalation, critical
                                                                                                                                              
                                                                                                                                               - Immediate Actions:
                                                                                                                                               - 1. Lock the targeted privileged account immediately
                                                                                                                                                 2. 2. Alert security team: Attempt to access admin/root account
                                                                                                                                                    3. 3. Enable enhanced logging on all systems
                                                                                                                                                      
                                                                                                                                                       4. Investigate:
                                                                                                                                                       5. - From which source IP? Is it authorized?
                                                                                                                                                          - - How many failed attempts? Single mistake or systematic attack?
                                                                                                                                                            - - Are there successful logins on this account during time of attack?
                                                                                                                                                              - - Are other privileged accounts being targeted?
                                                                                                                                                               
                                                                                                                                                                - Remediate:
                                                                                                                                                                - - Change privileged account password
                                                                                                                                                                  - - Audit sudo logs for unusual activity
                                                                                                                                                                    - - Enable MFA on all privileged accounts if not already done
                                                                                                                                                                      - - Review SSH key-based access policies
                                                                                                                                                                       
                                                                                                                                                                        - Escalate:
                                                                                                                                                                        - - Potential privilege escalation attack
                                                                                                                                                                          - - Critical incident requiring immediate C-level notification
                                                                                                                                                                            - - Full forensic investigation required
                                                                                                                                                                             
                                                                                                                                                                              - ## Response Severity Matrix
                                                                                                                                                                             
                                                                                                                                                                              - Critical (Level 8): Immediate escalation, stop what you're doing
                                                                                                                                                                              - High (Level 7): Investigate within 1 hour, notify management
                                                                                                                                                                              - Medium (Level 6): Investigate within 4 hours, document findings
                                                                                                                                                                              - Low (Level 5): Review during next shift, include in weekly security report
                                                                                                                                                                              - Info (Level 3-4): Log and monitor for patterns
                                                                                                                                                                             
                                                                                                                                                                              - ## Change Management Integration
                                                                                                                                                                             
                                                                                                                                                                              - Every remediation that modifies network configs requires a change ticket:
                                                                                                                                                                              - - File CHG ticket in ITSM system
                                                                                                                                                                                - - Document: What's changing, why, how to roll back
                                                                                                                                                                                  - - Get approval from change advisory board
                                                                                                                                                                                    - - Schedule during maintenance window
                                                                                                                                                                                      - - Execute and verify
                                                                                                                                                                                        - - Document actual vs planned changes
