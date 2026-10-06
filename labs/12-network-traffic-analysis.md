# Lab 12 — Network Traffic Analysis with Wireshark
## Objective

The objective of this lab was to analyze network traffic using Wireshark and understand how common network protocols behave at packet level.
The lab focused on capturing and analyzing ARP, DNS, TCP, HTTP, HTTPS/TLS, and SSH traffic, while comparing plaintext HTTP communication with encrypted HTTPS communication.

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

## 1. Network Connectivity Verification
The Ubuntu Server network configuration was verified:
```bash
ip addr show ens33
```

The server was assigned:
```
192.168.153.129/24
```

The Kali Linux machine used:
```
192.168.153.130/24
```

Connectivity was verified from Kali:
```
ping -c 4 192.168.153.129
```

The test successfully returned:
```
4 packets transmitted, 4 received, 0% packet loss
```
This confirmed that the Kali and Ubuntu machines could communicate successfully over the lab network.


## 2. ARP Traffic Analysis
Wireshark was used to capture ARP traffic:
arp

An ARP request was observed:
```
Who has 192.168.153.2?
Tell 192.168.153.130
```

The corresponding ARP response provided the MAC address associated with the requested IP address.

### Observation

ARP is used to resolve an IPv4 address to a MAC address within the local network.
The packet inspection showed fields including:
```
Opcode: request (1)
Sender IP address: 192.168.153.130
Target IP address: 192.168.153.2
```

A corresponding reply contained:
```
Opcode: reply (2)
Sender IP address: 192.168.153.2
Target IP address: 192.168.153.130
```

### Result
This demonstrated how hosts discover Layer 2 addresses before communicating with devices on the local network.


## 3. DNS Traffic Analysis
DNS traffic was captured using the following Wireshark filter:
dns

A DNS query for example.com was observed:
```
192.168.153.130 → 192.168.153.2
DNS Standard query A example.com
```

The DNS server responded with IPv4 addresses including:
```
104.20.23.154
172.66.147.243
```

An AAAA query was also observed for IPv6 resolution.
### DNS Request
The captured packet contained:
```
Queries: 1
Name: example.com
Type: A (Host Address)
Class: IN
```
### DNS Response
The response contained:
Answers: 2
```
example.com → 104.20.23.154
example.com → 172.66.147.243
```

### Result
The capture demonstrated the normal DNS resolution process:
```
Client
   |
   | DNS Query
   v
DNS Server
   |
   | DNS Response
   v
Client
```

This showed how domain names are translated into IP addresses before establishing application-layer connections.


## 4. HTTP Traffic Analysis
HTTP traffic was generated using:
```
curl -I http://example.com
```

Wireshark was filtered with:
```
tcp.port == 80
```

The TCP connection was established before the HTTP request was transmitted.
The captured HTTP request contained:
```
HEAD / HTTP/1.1
Host: example.com
User-Agent: curl/8.20.0
Accept: */*
```

The server responded with:
```
HTTP/1.1 200 OK
```

Additional HTTP response headers were also visible.

### Follow TCP Stream — HTTP
The HTTP connection was inspected using:
Follow → TCP Stream

The complete request and response were readable in plaintext.
Example:
```
HEAD / HTTP/1.1
Host: example.com
User-Agent: curl/8.20.0
Accept: */*

HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Connection: keep-alive
```

### Security Observation
Because HTTP does not encrypt the application data, an observer capable of capturing the traffic can inspect the HTTP request and response contents.
This demonstrates why HTTP is unsuitable for transmitting sensitive information over untrusted networks.


## 5. SSH Traffic Analysis
SSH traffic between Kali Linux and the Ubuntu Server was also captured.
Wireshark was filtered using:
```
tcp.port == 22
```

The connection showed the TCP three-way handshake followed by SSH protocol negotiation.
The captured traffic included:
```
SSH Version 2
```

and key exchange messages such as:
```
Client: Key Exchange Init
Server: Key Exchange Init
Client: PQ/T Hybrid Key Exchange Init
Server: PQ/T Hybrid Key Exchange Reply
```

The negotiated encryption included:
```
chacha20-poly1305@openssh.com
```

After the key exchange, the packets were displayed as:
```
Encrypted packet
```

### Result
The capture demonstrated that SSH protects the contents of the communication after the cryptographic negotiation phase.
Although the packets themselves can be captured, the actual SSH session contents were not available as plaintext through the packet capture.


## 6. HTTPS / TLS 1.3 Traffic Analysis
The Ubuntu Nginx server was accessed through HTTPS:
```
curl -k https://192.168.153.129
```

Wireshark was filtered using:
```
tcp.port == 443
```

The capture showed:
```
TCP
TLSv1.3
```

The TLS handshake included:
```
Client Hello
Server Hello
Change Cipher Spec
Application Data
```

The Client Hello contained TLS-related negotiation information, including:
```
supported_versions
TLS 1.3
key_share
X25519MLKEM768
```

The server selected a cipher suite including:
```
TLS_AES_256_GCM_SHA384
```


## 7. HTTPS Application Data Analysis
After the TLS handshake, Wireshark displayed packets as:
```
TLSv1.3 Application Data
```

The application data was not readable as HTTP plaintext.
Instead, Wireshark displayed encrypted application data.
This was further demonstrated using:
```
Follow → TCP Stream
```

Unlike the HTTP stream, the HTTPS stream consisted of unreadable encrypted data.

### HTTP vs HTTPS Comparison
| Feature | HTTP | HTTPS |
|---|---|---|
| Port | 80 | 443 |
| Encryption | No | Yes |
| Application data | Readable | Encrypted |
| HTTP headers | Visible | Protected |
| HTML content | Visible | Protected |
| Wireshark Follow TCP Stream | Plaintext | Encrypted data |
| Security | Low | Significantly stronger |


## 8. TCP Connection Analysis
TCP handshakes were observed during both HTTP and HTTPS connections.
A typical connection followed:
```
SYN
   ↓
SYN/ACK
   ↓
ACK
```

For example, the HTTPS connection showed:
```
192.168.153.130 → 192.168.153.129
SYN
```

followed by:
```
192.168.153.129 → 192.168.153.130
SYN, ACK
```

and finally:
```
192.168.153.130 → 192.168.153.129
ACK
```
After the TCP connection was established, TLS or HTTP communication followed depending on the destination port.


### Key Findings

#### ARP
ARP was observed resolving IPv4 addresses to MAC addresses within the local network.

#### DNS
DNS queries and responses were captured, demonstrating domain-name resolution.

#### HTTP
HTTP application data was transmitted in plaintext and could be directly inspected using Wireshark.

#### HTTPS
HTTPS traffic used TLS 1.3, and application data was encrypted.

#### SSH
SSH established a secure session through cryptographic key exchange and encrypted subsequent traffic.

#### TCP
TCP handshakes and connection termination packets were observed during application communication.


## Security Analysis
The most important observation of this lab was the difference between plaintext and encrypted application protocols.
With HTTP:
```
Client → HTTP Request → Server
```

the actual application content was visible in the packet capture.
With HTTPS:
```
Client → TLS Handshake → Encrypted Application Data → Server
```

the application content was protected from direct inspection.
This demonstrates the importance of using encrypted protocols such as HTTPS and SSH when transmitting sensitive information across networks.

## Skills Demonstrated
- Wireshark packet capture and analysis
- Network traffic inspection
- ARP analysis
- DNS packet analysis
- TCP handshake analysis
- HTTP request/response analysis
- HTTPS/TLS 1.3 analysis
- SSH traffic analysis
- TCP Stream analysis
- Protocol filtering in Wireshark
- Identification of plaintext vs encrypted traffic
- Basic network security analysis

## Conclusion
Lab 12 provided practical experience with network traffic analysis using Wireshark. By capturing ARP, DNS, TCP, HTTP, HTTPS, and SSH traffic, the lab demonstrated how different protocols operate at packet level.
The HTTP and HTTPS comparison was particularly important: HTTP traffic could be reconstructed and read directly, while HTTPS traffic was protected by TLS encryption.
This lab strengthened practical understanding of network protocols, packet analysis, encryption, and the security implications of transmitting data over plaintext versus encrypted channels.






















































































































