# 14 - Intrusion Detection with Suricata

## Objective

Deploy and configure Suricata as an Intrusion Detection System (IDS) on Ubuntu Server and generate alerts using custom detection rules.

The main objectives were:

- Install and configure Suricata
- Deploy the Emerging Threats Open ruleset
- Configure packet capture on the correct network interface
- Analyze Suricata logs
- Create and load custom IDS rules
- Detect ICMP traffic between Kali Linux and Ubuntu Server
- Detect HTTP GET requests
- Investigate alerts through `fast.log`
- Understand the difference between traffic detection and alert generation

---

## Lab Environment

| Component | Details |
|---|---|
| IDS | Suricata 8.0.7 |
| Server | Ubuntu Server |
| Security Client | Kali Linux |
| Server IP | `192.168.153.129` |
| Kali IP | `192.168.153.130` |
| Network Interface | `ens33` |
| Ruleset | Emerging Threats Open |
| Rules Directory | `/var/lib/suricata/rules/` |
| Alert Log | `/var/log/suricata/fast.log` |
| Event Log | `/var/log/suricata/eve.json` |

---

## 1. Suricata Installation and Configuration

Suricata was installed on Ubuntu Server and its configuration was validated.

The installed version was:

```text
Suricata 8.0.7
```

The configuration was tested using:

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml
```

After resolving configuration and application-layer protocol compatibility issues, the configuration test completed successfully.

The service was enabled and started:

```bash
sudo systemctl enable --now suricata
```

The service status confirmed that Suricata was running.

---

## 2. Emerging Threats Open Ruleset

The Emerging Threats Open ruleset was downloaded and updated using:

```bash
sudo suricata-update
```

The update process reported:

- 69,082 rules in the downloaded ruleset
- 53,141 enabled rules
- 136 rules dropped due to flowbit dependencies

During initial configuration testing, some DNP3 and Modbus rules could not be loaded because the corresponding application-layer protocol detection was disabled.

The configuration was adjusted to enable the required protocols, and the configuration test subsequently succeeded.

---

## 3. Network Interface Configuration

The Ubuntu Server network interfaces were inspected using:

```bash
ip -br addr
```

The server's network interface was identified as:

```text
ens33
192.168.153.129/24
```

The initial Suricata configuration referenced `eth0`, which was not the active interface on the Ubuntu Server.

The AF_PACKET configuration was corrected to use `ens33`.

This resolved the interface mismatch and allowed Suricata to observe traffic on the lab network.

The interface was verified in EVE JSON events:

```json
{
  "in_iface": "ens33",
  "event_type": "flow"
}
```

This confirmed that Suricata was processing traffic through the intended interface.

---

## 4. Log Analysis

Suricata log files were inspected under:

```text
/var/log/suricata/
```

The primary log files included:

```text
fast.log
eve.json
stats.log
```

The alert log was monitored using:

```bash
sudo tail -f /var/log/suricata/fast.log
```

EVE JSON output was inspected using:

```bash
sudo tail -f /var/log/suricata/eve.json
```

Flow events were observed in EVE JSON, confirming that network traffic was being processed.

Existing Emerging Threats rules also generated alerts during testing. These events were treated as detections requiring context and investigation rather than automatically being classified as confirmed attacks.

---

## 5. Custom Rule Configuration

A local rules file was created at:

```text
/var/lib/suricata/rules/local.rules
```

The file was added to the `rule-files` section of:

```text
/etc/suricata/suricata.yaml
```

The relevant configuration became:

```yaml
rule-files:
  - suricata.rules
  - local.rules
```

This allowed Suricata to load both the Emerging Threats ruleset and the locally created rules.

---

## 6. Custom ICMP Detection Rule

The following rule was added to detect ICMP traffic destined for the configured home network:

```text
alert icmp any any -> $HOME_NET any (msg:"LOCAL ICMP TEST - Kali to Ubuntu"; sid:1000001; rev:1;)
```

The rule was assigned:

```text
SID: 1000001
Message: LOCAL ICMP TEST - Kali to Ubuntu
```

The configuration and rules were tested with:

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml
```

The test completed successfully.

### Generating Test Traffic

ICMP traffic was generated from Kali Linux:

```bash
ping -c 4 192.168.153.129
```

Suricata generated alerts associated with the custom rule.

An observed alert included:

```text
[1:1000001:1] LOCAL ICMP TEST - Kali to Ubuntu
```

The log showed ICMP traffic between:

```text
192.168.153.130 → 192.168.153.129
192.168.153.129 → 192.168.153.130
```

### Result

The custom ICMP rule successfully generated IDS alerts, demonstrating that Suricata could detect matching traffic and write alerts to `fast.log`.

---

## 7. Custom HTTP Detection Rule

A second custom rule was added to detect HTTP GET requests from Kali Linux to the Ubuntu Server:

```text
alert http 192.168.153.130 any -> 192.168.153.129 any (msg:"LOCAL HTTP GET TEST - Kali to Ubuntu"; flow:to_server,established; http.method; content:"GET"; sid:1000002; rev:1;)
```

The rule was assigned:

```text
SID: 1000002
Message: LOCAL HTTP GET TEST - Kali to Ubuntu
```

The rule checks for HTTP traffic from the specified source IP to the specified destination IP and matches GET requests in an established connection.

### Generating Test Traffic

An HTTP request was generated from Kali Linux:

```bash
curl http://192.168.153.129/
```

Suricata successfully detected the request.

The following alert was observed in `fast.log`:

```text
[1:1000002:1] LOCAL HTTP GET TEST - Kali to Ubuntu
```

The captured connection was:

```text
Protocol: TCP
Source: 192.168.153.130:38236
Destination: 192.168.153.129:80
```

### Result

The custom HTTP rule successfully detected the GET request and generated an alert.

The test demonstrated how protocol-aware detection rules can identify application-layer activity rather than only matching generic network packets.

---

## 8. Alert Investigation

The custom alerts were verified directly in:

```text
/var/log/suricata/fast.log
```

Each custom rule used a unique signature ID:

| SID | Detection |
|---|---|
| `1000001` | ICMP traffic |
| `1000002` | HTTP GET request |

The signature IDs allowed the alerts to be associated with their corresponding custom rules.

The tests demonstrated the following detection workflow:

```text
Traffic Generation
        |
        v
Network Interface (ens33)
        |
        v
Suricata Packet Inspection
        |
        v
Custom Rule Matching
        |
        v
IDS Alert
        |
        v
fast.log
```

This confirmed the complete detection process from traffic generation to alert logging.

---

## Results

| Assessment Area | Result |
|---|---|
| Suricata installation | Completed |
| Configuration validation | Successful |
| Emerging Threats Open update | Completed |
| Network interface configuration | Corrected |
| EVE JSON flow monitoring | Verified |
| Local rules configuration | Completed |
| Custom ICMP rule | Triggered successfully |
| Custom HTTP GET rule | Triggered successfully |
| Alert generation | Verified |
| Alert log investigation | Completed |

Both custom detection rules were successfully triggered by controlled traffic generated from Kali Linux.

---

## Skills Demonstrated

### IDS Configuration

- Suricata installation and configuration
- AF_PACKET interface configuration
- Ruleset management
- Configuration validation
- Custom signature creation

### Network Monitoring

- ICMP traffic detection
- HTTP request detection
- Interface-level traffic inspection
- Flow event analysis

### Security Monitoring

- IDS alert investigation
- Signature ID identification
- `fast.log` analysis
- EVE JSON inspection
- Rule validation and troubleshooting

---

## Conclusion

This lab provided practical experience deploying Suricata as an IDS and validating its detection capabilities.

The initial configuration issues demonstrated the importance of matching the capture interface to the actual network interface and ensuring that custom rules are included in the active rule configuration.

Two custom rules were then created and tested successfully. The ICMP rule detected traffic between Kali Linux and Ubuntu Server, while the HTTP rule detected an HTTP GET request from the Kali client.

The resulting alerts were verified in `fast.log`, demonstrating that Suricata was able to inspect network traffic, match custom signatures, and generate security alerts.

This lab established a foundation for future IDS/IPS, security monitoring, and incident response exercises.
