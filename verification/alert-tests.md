# Alert Injection & Verification Tests

Validate Wazuh detection rules by triggering alerts from lab environment. All tests use Project 1 network topology.

## Test 1: SSH Brute Force Detection (Rule 5001)

Test Objective: Verify SSH brute force rule detects repeated login failures

Steps:
1. From external VM or attacker simulator:
2.    ssh admin@10.10.99.1 -p 22 (incorrect password, repeat 10 times)

3.2. Monitor Wazuh alerts in real-time:
   tail -f /var/ossec/logs/alerts/alerts.json | grep "5001"

   3. Expected Alert:
   4.    - source_ip: attacker IP
         -    - rule_id: 5001
              -    - level: 7 (high)
                   -    - group: [authentication, ssh, brute_force]
                    
                        - 4. Verify Alert Action:
                          5.    - Check if firewall has rule to drop source IP (dependent on IR playbook)
                                -    - Alert email sent to security team
                                 
                                     - Pass Criteria: Alert fires within 30 seconds of 5th failed attempt
                                 
                                     - ## Test 2: Port Scan Detection (Rule 5002)
                                 
                                     - Test Objective: Verify port scan detection works
                                 
                                     - Steps:
                                     - 1. Run nmap from external VM targeting management VLAN:
                                       2.    nmap -p 1-1000 10.10.99.0/24 -sV
                                      
                                       3.2. Monitor Wazuh:
                                          tail -f /var/ossec/logs/alerts/alerts.json | grep "5002"

                                       3. Expected Alert:
                                       4.    - source_ip: nmap VM
                                             -    - rule_id: 5002
                                                  -    - level: 5 (medium)
                                                       -    - group: [network, scan, reconnaissance]
                                                        
                                                            - 4. Verify Firewall Logs:
                                                              5.    Check pfSense logs for dropped packets to unreachable ports
                                                             
                                                              6.Pass Criteria: Alert fires during scan, firewall blocks attempts

                                                              ## Test 3: ACL Violation (Rule 5003)

                                                              Test Objective: Guest VLAN should not reach Server VLAN

                                                              Steps:
                                                              1. Connect test VM to Guest VLAN (10.10.30.0/24)
                                                              2. 2. From test VM: ping 10.10.20.10 (Wazuh server, restricted)
                                                                 3. 3. Packet should be dropped by ACL on SW-DIST (rule 5003)
                                                                   
                                                                    4. Monitor on R1-CORE:
                                                                    5.    sh ip access-lists GUEST-RESTRICT
                                                                    6.   (should show match count increasing)
                                                                   
                                                                    7.   Expected Wazuh Alert:
                                                                    8.      - rule_id: 5003
                                                                    9.     - level: 6
                                                                    10.    - group: [network, security, acl]
                                                                       
                                                                           - Pass Criteria: Ping fails, ACL match count increments, Wazuh alert fires
                                                                       
                                                                           - ## Test 4: Spanning Tree Topology Change (Rule 5004)
                                                                       
                                                                           - Test Objective: Detect unexpected STP topology changes
                                                                       
                                                                           - Steps:
                                                                           - 1. Shutdown root bridge (SW-DIST):
                                                                             2.    conf t
                                                                             3.   shutdown (on Gi0/0)
                                                                            
                                                                             4.   2. Monitor spanning tree:
                                                                                  3.    sh spanning-tree summary
                                                                               
                                                                                  4.3. Wait for new root election (should be R1-CORE or SW-ACC-1)

                                                                                  Expected Wazuh Alert:
                                                                                     - rule_id: 5004
                                                                                     -    - level: 7
                                                                                          -    - Keywords: "topology change", "new root", "bridge ID"
                                                                                           
                                                                                               - 4. Restore:
                                                                                                 5.    no shutdown
                                                                                                
                                                                                                 6.Pass Criteria: Alert fires when topology changes, resolves when network stabilized

                                                                                                 ## Test 5: Unauthorized VLAN Access (Rule 5005)

                                                                                                 Test Objective: Detect guest attempting to reach server resources

                                                                                                 Steps:
                                                                                                 1. Assign test VM to Guest VLAN
                                                                                                 2. 2. Attempt to SSH to server subnet device (10.10.20.10):
                                                                                                    3.    ssh admin@10.10.20.10
                                                                                                   
                                                                                                    4.3. SSH attempt blocked by GUEST-RESTRICT ACL on SW-DIST
                                                                                                    
                                                                                                    Expected Wazuh Alert:
                                                                                                       - rule_id: 5005
                                                                                                       -    - level: 8 (critical)
                                                                                                            -    - group: [network, vlan, policy_violation]
                                                                                                             
                                                                                                                 - 4. Verify escalation:
                                                                                                                   5.    - Check for email/Slack notification to security team
                                                                                                                         -    - Incident should be logged at level 8
                                                                                                                          
                                                                                                                              - Pass Criteria: Alert fires immediately, escalation triggers, incident logged
                                                                                                                          
                                                                                                                              - ## Test 6: Failed Privileged Account Auth (Rule 5006)
                                                                                                                          
                                                                                                                              - Test Objective: Detect attacks on admin/root accounts
                                                                                                                          
                                                                                                                              - Steps:
                                                                                                                              - 1. Attempt login with wrong password:
                                                                                                                                2.    ssh root@10.10.99.1 (wrong password, retry 5 times)
                                                                                                                               
                                                                                                                                3.2. Monitor auth logs:
                                                                                                                                   grep "Failed password for root" /var/log/auth.log
                                                                                                                                
                                                                                                                                Expected Wazuh Alert:
                                                                                                                                   - rule_id: 5006
                                                                                                                                   -    - level: 8 (critical)
                                                                                                                                        -    - group: [authentication, privilege_escalation, critical]
                                                                                                                                         
                                                                                                                                             - 4. Verify:
                                                                                                                                               5.    - Alert fires on 2nd failed attempt
                                                                                                                                                     -    - Email notification sent immediately
                                                                                                                                                      
                                                                                                                                                          - Pass Criteria: Alert within 10 seconds, source IP tracked, escalation priority set
                                                                                                                                                      
                                                                                                                                                          - ## Test 7: Port Security Violation (Rule 5007)
                                                                                                                                                      
                                                                                                                                                          - Test Objective: Detect port security violations on access switches
                                                                                                                                                      
                                                                                                                                                          - Steps:
                                                                                                                                                          - 1. Connect 3 devices to single access port on SW-ACC-1 (exceeds sticky MAC limit of 1):
                                                                                                                                                            2.    - Device A, Device B, Device C
                                                                                                                                                              
                                                                                                                                                                  - 2. Port should go to "secure-shutdown" state
                                                                                                                                                                   
                                                                                                                                                                    3. Monitor on SW-ACC-1:
                                                                                                                                                                    4.    sh port-security
                                                                                                                                                                   
                                                                                                                                                                    5.Expected Wazuh Alert:
                                                                                                                                                                       - rule_id: 5007
                                                                                                                                                                       -    - level: 6
                                                                                                                                                                            -    - Keywords: "port security", "violation", "secure MAC"
                                                                                                                                                                             
                                                                                                                                                                                 - Pass Criteria: Port security event logged, Wazuh alert fires
                                                                                                                                                                             
                                                                                                                                                                                 - ## Test 8: Unauthorized MAC Address (Rule 5008)
                                                                                                                                                                             
                                                                                                                                                                                 - Test Objective: Detect rogue MAC addresses
                                                                                                                                                                             
                                                                                                                                                                                 - Steps:
                                                                                                                                                                                 - 1. Connect device with random/spoofed MAC to network
                                                                                                                                                                                   2. 2. Monitor ARP table changes on SW-ACC-1
                                                                                                                                                                                     
                                                                                                                                                                                      3. Expected Wazuh Alert:
                                                                                                                                                                                      4.    - rule_id: 5008
                                                                                                                                                                                            -    - level: 7
                                                                                                                                                                                                 -    - Keywords: "unknown MAC", "unauthorized"
                                                                                                                                                                                                  
                                                                                                                                                                                                      - Pass Criteria: Alert fires when new MAC learned on switch
                                                                                                                                                                                                  
                                                                                                                                                                                                      - ## Test 9: OSPF Neighbor Down (Rule 5009)
                                                                                                                                                                                                  
                                                                                                                                                                                                      - Test Objective: Verify OSPF neighbor loss detection
                                                                                                                                                                                                  
                                                                                                                                                                                                      - Steps:
                                                                                                                                                                                                      - 1. Shutdown OSPF on R1-CORE:
                                                                                                                                                                                                        2.    conf t
                                                                                                                                                                                                        3.   router ospf 1
                                                                                                                                                                                                        4.      shutdown
                                                                                                                                                                                                       
                                                                                                                                                                                                        5.  2. Monitor on SW-DIST:
                                                                                                                                                                                                            3.    sh ip ospf neighbor (should show neighbor down)
                                                                                                                                                                                                          
                                                                                                                                                                                                            4.Expected Wazuh Alert:
                                                                                                                                                                                                               - rule_id: 5009
                                                                                                                                                                                                               -    - level: 5
                                                                                                                                                                                                                    -    - Keywords: "OSPF", "neighbor", "down"
                                                                                                                                                                                                                     
                                                                                                                                                                                                                         - 3. Restore:
                                                                                                                                                                                                                           4.    no shutdown
                                                                                                                                                                                                                          
                                                                                                                                                                                                                           5.Pass Criteria: Alert fires when neighbor goes down, clears when restored
                                                                                                                                                                                                                           
                                                                                                                                                                                                                           ## Test 10: Interface Flap (Rule 5010)
                                                                                                                                                                                                                           
                                                                                                                                                                                                                           Test Objective: Detect frequent link state changes
                                                                                                                                                                                                                           
                                                                                                                                                                                                                           Steps:
                                                                                                                                                                                                                           1. On SW-DIST, flap an interface 5 times quickly:
                                                                                                                                                                                                                           2.    conf t
                                                                                                                                                                                                                           3.   interface Gi0/1
                                                                                                                                                                                                                           4.      shutdown
                                                                                                                                                                                                                           5.     no shutdown
                                                                                                                                                                                                                           6.    (repeat 5 times rapidly)
                                                                                                                                                                                                                          
                                                                                                                                                                                                                           7.Expected Wazuh Alert:
                                                                                                                                                                                                                              - rule_id: 5010
                                                                                                                                                                                                                              -    - level: 4
                                                                                                                                                                                                                                   -    - Keywords: "line protocol", "up", "down"
                                                                                                                                                                                                                                    
                                                                                                                                                                                                                                        - Pass Criteria: Multiple alerts for up/down transitions, pattern recognized
                                                                                                                                                                                                                                    
                                                                                                                                                                                                                                        - ## Overall Verification Checklist
                                                                                                                                                                                                                                    
                                                                                                                                                                                                                                        - - All 10 rules generate alerts at expected severity levels
                                                                                                                                                                                                                                          - - Email notifications sent for Level 7+ alerts
                                                                                                                                                                                                                                            - - Alert JSON format valid and parseable
                                                                                                                                                                                                                                              - - Wazuh dashboard displays alerts in real-time
                                                                                                                                                                                                                                                - - Playbook runbooks referenced correctly in alerts
                                                                                                                                                                                                                                                  - - Change control documentation complete
                                                                                                                                                                                                                                                   
                                                                                                                                                                                                                                                    - ## Alert Aggregation Query
                                                                                                                                                                                                                                                   
                                                                                                                                                                                                                                                    - View all security events in 24-hour period:
                                                                                                                                                                                                                                                    - /var/ossec/logs/alerts/alerts.json | jq '.[] | select(.rule.level >= 5)' | jq -s 'group_by(.rule.id) | map({rule_id: .[0].rule.id, description: .[0].rule.description, count: length})' | jq sort_by(.count) | jq reverse
