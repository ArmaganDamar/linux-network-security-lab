# 10 - Log Analysis

## Objective

Analyze Linux system and Nginx logs to identify authentication events, HTTP activity, suspicious requests, source IPs, HTTP status codes, and client User-Agents.

The main objectives were:

- Analyze SSH authentication logs
- Analyze Nginx access logs
- Identify successful and failed authentication events
- Investigate HTTP status codes
- Identify suspicious HTTP requests
- Analyze source IP addresses and User-Agents
- Understand Nginx log rotation
- Analyze compressed `.gz` log files
- Build a basic security event timeline

---

## Lab Environment

| Component | Details |
|---|---|
| Server | Ubuntu Server |
| Client | Kali Linux |
| Web Server | Nginx 1.28.3 |
| Protocol | HTTP / HTTPS |
| Network | VMware Host-Only / NAT Lab Network |
| Server IP | 192.168.153.129 |

> The IP address belongs to an isolated home lab environment.

---

## 1. System Log Analysis

System messages were reviewed using:

```bash
sudo journalctl -p warning..alert --since "today" --no-pager
```

Several warning-level messages related to the virtualized environment were observed.

These messages were reviewed in context rather than automatically classified as security incidents.

This demonstrated the importance of correlating log messages with system state and environment

## 2. SSH Log Analysis

SSH events were analyzed using:
```bash
sudo journalctl -u ssh --since "3 days ago" --no-pager
```

Successful authentication events were filtered using:
```bash
sudo journalctl -u ssh --since "3 days ago" --no-pager | grep "Accepted"
```

A more structured output was generated using:
```bash
sudo journalctl -u ssh --since "3 days ago" --no-pager | grep "Accepted" | awk '{print $1, $2, $3, "|", $7, "|", $9, "|", $11}'
```
Observed successful authentication methods included:

- Password authentication
- Public key authentication

The logs showed a transition from password authentication to public key authentication during the SSH hardening process.

## 3. SSH Authentication Analysis

Authentication methods were counted using:
```bash
sudo journalctl -u ssh --since "3 days ago" --no-pager | grep "Accepted" | awk '{print $7}' | sort | uniq -c
```
Observed results:

| Authentication Method | Count |
| --------------------- | ----: |
| Password              |     2 |
| Public key            |     4 |

Source IP addresses were also analyzed:
```bash
sudo journalctl -u ssh --since "3 days ago" --no-pager | grep "Accepted" | awk '{print $11}' | sort | uniq -c
```
Observed sources:

| Source IP       | Successful Logins |
| --------------- | ----------------: |
| 192.168.153.130 |                 5 |
| 192.168.153.129 |                 1 |

The `192.168.153.13` address belongs to the Kali Linux system used for the lab.

The `192.168.153.129` event was investigated separately and was identified as a successful SSH session originating from the Ubuntu server's own address.

## 4. Failed SSH Authentication Analysis

Potential failed authentication events were searched using:
```bash
sudo journalctl -u ssh --since "3 days ago" --no-pager | grep -E "Failed password|Invalid user|authentication failure"
```
No matching failed authentication events were found in the analyzed three-day period.

This demonstrates that the absence of matching events should be treated as an observation for the selected time window rather than proof that no security activity occurred.

## 5. Fail2Ban Log Analysis

Fail2Ban activity was reviewed using:
```bash
sudo journalctl -u fail2ban --since "3 days ago" --no-pager
```
The service logs showed successful startup and shutdown events associated with system operation.


The SSH jail was also checked:
```bash
sudo fail2ban-client status sshd
```

Observed state:

- Currently failed: 0
- Total failed: 0
- Currently banned: 0
- Total banned: 0

The SSH jail was confirmed to be active and monitoring the systemd SSH journal.

## 6. Nginx Log Analysis

Nginx service logs were reviewed using:
```bash
sudo journalctl -u nginx --since "3 days ago" --no-pager
```

Nginx access logs were analyzed from:
```bash
/var/log/nginx/access.log
```

The active log and rotated logs were inspected using:
```bash
ls -lah /var/log/nginx/
```

The log directory contained active, rotated, and compressed log files, including:
```bash
access.log
access.log.1
access.log.2.gz
access.log.3.gz
access.log.4.gz
```

## 7. Log Rotation and Compressed Logs

Previous security testing activity was found in compressed Nginx log files.

Compressed logs were searched using:
```bash
sudo zgrep '/admin' /var/log/nginx/access.log*.gz
```

This demonstrated that previous events were stored in rotated and compressed log files rather than only in the active `access.log`.

The use of `zgrep` and `zcat` was required to analyze gzip-compressed log files.

## 8. HTTP Status Code Analysis

HTTP status codes across active and rotated logs were analyzed using:
```bash
sudo zcat -f /var/log/nginx/access.log* | awk 'NF >= 9 {print $9}' | sort | uniq -c | sort -nr
```

Observed results:

| Status Code | Count |
| ----------- | ----: |
| 301         |    71 |
| 404         |    45 |
| 200         |    45 |
| 403         |     5 |
| 304         |     1 |

The observed status codes represented:

- `200` - successful responses
- `301` - HTTP to HTTPS redirects
- `404` - requested resource not found
- `403` - access forbidden
- `304` - resource not modified

## 9. Suspicious HTTP Request Analysis

A controlled set of requests was generated from Kali Linux against the lab server.

The following paths were tested:
```bash
/admin
/login
/wp-admin
/.env
/etc/passwd
```

These requests were identified in rotated Nginx logs using:
```bash
sudo zgrep -hE '"GET /(admin|login|wp-admin|\.env|etc/passwd) HTTP/1.1" 404' /var/log/nginx/access.log*.gz
```

Five targeted requests returned `404` responses.

The source IP was:
```bash
192.168.153.130
```

which corresponds to the Kali Linux client.

Observed requests included:
```bash
/admin
/login
/wp-admin
/.env
/etc/passwd
```
These requests were intentionally generated as part of the security lab.

## 10. Source IP Analysis

HTTP activity was analyzed by source IP:
```bash
sudo zcat -f /var/log/nginx/access.log* | awk 'NF >= 9 {print $1, $9}' | sort | uniq -c | sort -nr
```

The analysis showed activity from multiple local lab addresses, including:

- `192.168.153.129`
- `192.168.153.130`
- `192.168.153.1`

The majority of the security testing activity originated from `192.168.153.130`, the Kali Linux system.

## 11. User-Agent Analysis

User-Agent values were extracted using:
```bash
sudo zcat -f /var/log/nginx/access.log* | awk -F'"' 'NF >= 6 {print $6}' | sort | uniq -c | sort -nr
```

Observed clients included:
```bash
Mozilla/5.0 (compatible; Nmap Scripting Engine...)
curl/8.20.0
Mozilla/5.0 ...
-
```

The Nmap Scripting Engine User-Agent was associated with automated Nmap HTTP script activity.

The `curl/8.20.0` User-Agent was associated with manually generated HTTP requests during the lab.

User-Agent values were treated as indicators rather than definitive proof of client identity because they can be modified by clients.

## 12. 403 Analysis

HTTP 403 responses were investigated using:
```bash
sudo zcat -f /var/log/nginx/access.log* | awk '$9 == 403 {print}'
```

Observed sources included:

- `192.168.153.129`
- `192.168.153.130`
- `192.168.153.1`

The 403 events were reviewed in their historical context.

Several of these events were associated with the initial Nginx HTTPS configuration stage, when the server returned `403 Forbidden` before the custom `index.html` page was created.

This demonstrated the importance of correlating log events with configuration changes and previous troubleshooting activity.

## 13. Incident Timeline

A basic SSH authentication timeline was constructed from the logs:
| Time            | Source          | Authentication |
| --------------- | --------------- | -------------- |
| Sep 28 10:45:54 | 192.168.153.130 | Password       |
| Sep 28 10:46:11 | 192.168.153.130 | Public key     |
| Sep 28 11:04:47 | 192.168.153.129 | Password       |
| Sep 28 11:05:50 | 192.168.153.130 | Public key     |
| Sep 28 11:11:08 | 192.168.153.130 | Public key     |
| Sep 28 11:41:50 | 192.168.153.130 | Public key     |

The timeline showed the transition from password-based SSH authentication to public key authentication during the SSH hardening process.

## Results

Observed results:
| Analysis Area                      | Result                   |
| ---------------------------------- | ------------------------ |
| System journal analysis            | Completed                |
| SSH authentication analysis        | Completed                |
| Successful authentication analysis | Completed                |
| Failed authentication search       | No matching events found |
| Fail2Ban log analysis              | Completed                |
| Nginx service log analysis         | Completed                |
| Nginx access log analysis          | Completed                |
| Log rotation analysis              | Completed                |
| Compressed log analysis            | Completed                |
| HTTP status code analysis          | Completed                |
| Source IP analysis                 | Completed                |
| User-Agent analysis                | Completed                |
| Suspicious HTTP request analysis   | Completed                |
| SSH event timeline                 | Created                  |

The analysis demonstrated how system and application logs can be used to reconstruct activity, identify authentication methods, investigate HTTP behavior, and correlate events with configuration changes.

The lab also demonstrated that security events should be analyzed in context rather than classified solely based on a single log entry or status code.

## Skills Demonstrated
### Linux Log Analysis
- `journalctl`
- `grep`
- `awk`
- `sort`
- `uniq`
- `zgrep`
- `zcat`
- Log filtering
- Time-based analysis
- Log rotation
### Security Monitoring
- SSH authentication monitoring
- HTTP activity analysis
- Source IP identification
- User-Agent analysis
- Suspicious request identification
- Security event correlation
- Basic incident timeline creation
### Web Server Monitoring
- Nginx access logs
- Nginx service logs
- HTTP status codes
- HTTP redirects
- Error analysis
- Historical log investigation

## Conclusion

The Linux Log Analysis lab demonstrated how system and application logs can be collected, filtered, correlated, and analyzed to investigate system activity.

SSH authentication logs were used to identify authentication methods and source addresses, while Nginx logs were analyzed to identify HTTP behavior, suspicious requests, status codes, and client information.

The lab also demonstrated log rotation and the analysis of compressed historical logs using `zgrep` and `zcat`.

These techniques provide a foundation for future SIEM, security monitoring, and incident response labs.











































































































