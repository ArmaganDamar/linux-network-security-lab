# 11 - Wireshark Network Analysis

## Objective

Capture and analyze network traffic between a Kali Linux client and an Ubuntu Server using Wireshark.

The main objectives were:

- Analyze ICMP traffic
- Observe TCP three-way handshake
- Analyze SSH traffic at packet level
- Inspect SSH key exchange and encrypted traffic
- Analyze HTTP traffic
- Inspect TLS 1.3 handshake
- Identify the negotiated TLS cipher suite
- Observe encrypted HTTPS application data
- Understand the difference between plaintext and encrypted application traffic

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

## 1. ICMP Packet Capture

The first experiment captured ICMP traffic between Kali Linux and the Ubuntu Server.

Kali was used to generate ICMP traffic:

```bash
ping -c 4 192.168.153.129
```

The following display filter was used:
```bash
icmp
```

The captured packets included:
```bash
Echo (ping) request
Echo (ping) reply
```

The packet structure was inspected through:
```bash
Ethernet II
Internet Protocol Version 4
Internet Control Message Protocol
```

The source and destination addresses were observed as:
```bash
Source:      192.168.153.130
Destination: 192.168.153.129
```
The ICMP request and reply packets demonstrated the basic request/response behavior of ICMP.


## 2. TCP Three-Way Handshake

SSH traffic was captured using:
```bash
tcp.port == 22
```

A new SSH connection was established from Kali:
```bash
ssh armagan@192.168.153.129
```

The TCP connection was observed through the following packet sequence:
```bash
[SYN]
[SYN, ACK]
[ACK]
```

The captured connection used:
```bash
Client IP:       192.168.153.130
Server IP:       192.168.153.129
Client Port:     36680
Server Port:     22
```
The TCP handshake demonstrated the establishment of a TCP connection before SSH protocol communication began.


## 3. TCP Packet Analysis

The initial SYN packet was inspected in detail.

Observed fields included:
```bash
Source Port:      36680
Destination Port: 22
Flags:            SYN
Sequence Number:  0 (relative)
Acknowledgment:   0
```

The following packets completed the handshake:
```bash
Client → Server : SYN
Server → Client : SYN, ACK
Client → Server : ACK
```

This confirmed the TCP three-way handshake at packet level.


## 4. SSH Protocol Analysis

After the TCP handshake, the SSH protocol negotiation was observed.

The captured stream contained:
```bash
SSH Protocol
Client: Key Exchange Init
Server: Key Exchange Init
Key Exchange
New Keys
Encrypted packet
```

The SSH traffic was isolated with:
```bash
tcp.stream == 0
```
The SSH `KEXINIT` packets were inspected from both the client and server.


## 5. SSH Key Exchange Analysis

The SSH Key Exchange Init packet contained several algorithm negotiation fields, including:
```bash
kex_algorithms
server_host_key_algorithms
encryption_algorithms_client_to_server
encryption_algorithms_server_to_client
mac_algorithms
compression_algorithms
```

The packet displayed the following key exchange method:
```bash
mlkem768x25519-sha256
```

The supported encryption algorithms included:
```bash
chacha20-poly1305@openssh.com
```

The server also advertised key exchange options including:
```bash
mlkem768x25519-sha256
sntrup761x25519-sha512
```
The `KEXINIT` packets were used to demonstrate that the client and server exchange supported cryptographic algorithms before establishing the encrypted SSH session.


## 6. SSH Encrypted Traffic

After key exchange, the SSH connection transitioned to encrypted traffic.

Wireshark displayed packets such as:
```bash
Client: New Keys, Encrypted packet
Server: Encrypted packet
```

The selected SSH encryption was displayed as:
```bash
chacha20-poly1305@openssh.com
```

After the `New Keys` stage, application traffic was displayed as encrypted SSH packets.

A test string was sent through the SSH session:
```bash
echo "Wireshark test"
```

The application data was not visible as plaintext in the captured SSH packets.

This demonstrated the transition from SSH negotiation to encrypted application traffic.


## 7. HTTP Traffic Analysis

HTTP traffic was captured using:
```bash
tcp.port == 80
```

A request was generated from Kali:
```bash
curl -I http://192.168.153.129
```

Wireshark captured the following sequence:
```bash
TCP SYN
TCP SYN, ACK
TCP ACK
HTTP HEAD / HTTP/1.1
HTTP/1.1 301 Moved Permanently
```

The response contained:
```bash
Location: https://192.168.153.129/
```
The HTTP request and response were identifiable at packet level because the traffic was not protected by TLS.

This also demonstrated the HTTP to HTTPS redirection previously configured on the Nginx server.


## 8. HTTPS and TLS 1.3 Analysis

HTTPS traffic was captured using:
```bash
tcp.port == 443
```

A new HTTPS request was generated:
```bash
curl -k https://192.168.153.129
```

The captured traffic included:
```bash
TCP SYN
TCP SYN, ACK
TCP ACK
TLSv1.3 Client Hello
TLSv1.3 Server Hello
TLSv1.3 Application Data
```
The TLS handshake was inspected in detail.


## 9. TLS Client Hello

The `Client Hello` packet was analyzed.

The packet contained:
```bash
Supported Versions:
TLS 1.3
TLS 1.2
```

The packet also contained a key share extension including:
```bash
X25519MLKEM768
X25519
```

The client advertised multiple supported cipher suites and TLS extensions before the server selected the parameters for the connection.


## 10. TLS Server Hello

The server response was inspected through the `Server Hello` packet.

The negotiated TLS version was identified through:
```bash
supported_versions:
TLS 1.3
```

The selected cipher suite was:
```bash
TLS_AES_256_GCM_SHA384
```

The server also provided a key share using:
```bash
X25519MLKEM768
```

This demonstrated the difference between the client-offered parameters and the parameters selected by the server.


## 11. HTTPS Encrypted Application Data

After the TLS handshake, Wireshark displayed:
```bash
TLSv1.3
Application Data
Encrypted Application Data
```

The captured packet contained encrypted application data instead of readable HTML content.

The Ubuntu Server response itself contained the custom web page:
```html
<h1>Security Homelab</h1>
```

However, the HTTPS packet payload did not expose this HTML as plaintext.

The packet was identified as:
```bash
Application Data Protocol: Hypertext Transfer Protocol
```

while the application payload remained encrypted.

This demonstrated the protection provided by TLS for HTTP application data.


## 12. HTTP vs HTTPS Comparison

The packet captures were compared to demonstrate the difference between HTTP and HTTPS.

### HTTP
```bash
TCP :80
   ↓
HTTP
   ↓
HEAD /
   ↓
301 Moved Permanently
```
HTTP application information was directly visible in the packet capture.

### HTTPS
```bash
TCP :443
   ↓
TLS 1.3
   ↓
Client Hello
   ↓
Server Hello
   ↓
Application Data
```
The application payload was encrypted.

This demonstrated that while network metadata such as source/destination IP addresses and ports remains observable, the protected application data is not directly readable from the packet payload.


## Results

Observed results:

| Analysis Area                   | Result                |
| ------------------------------- | --------------------- |
| ICMP request/response analysis  | Completed             |
| TCP three-way handshake         | Captured and verified |
| SSH traffic capture             | Completed             |
| SSH KEXINIT analysis            | Completed             |
| SSH encryption analysis         | Completed             |
| SSH encrypted packet analysis   | Completed             |
| HTTP traffic capture            | Completed             |
| HTTP redirect analysis          | Completed             |
| TLS 1.3 Client Hello analysis   | Completed             |
| TLS 1.3 Server Hello analysis   | Completed             |
| TLS cipher suite identification | Completed             |
| HTTPS encrypted data analysis   | Completed             |
| HTTP vs HTTPS comparison        | Completed             |

The packet captures demonstrated the progression from lower-level network communication to encrypted application protocols.

The lab also confirmed that HTTP application traffic can be inspected directly at packet level, while HTTPS application data is protected by TLS encryption.

## Skills Demonstrated
### Network Analysis
- Wireshark
- Packet capture
- Packet filtering
- ICMP analysis
- IPv4 analysis
- TCP analysis
- TCP three-way handshake
- Port analysis
### SSH Analysis
- SSH protocol inspection
- KEXINIT analysis
- Key exchange analysis
- Encryption analysis
- Encrypted traffic identification
### Web and TLS Analysis
- HTTP packet inspection
- HTTP redirection analysis
- TLS 1.3 handshake analysis
- Client Hello analysis
- Server Hello analysis
- Cipher suite identification
- Encrypted application data analysis


## Conclusion

The Wireshark network analysis lab provided packet-level visibility into the communication between the Kali Linux client and Ubuntu Server.

ICMP, TCP, SSH, HTTP, and HTTPS traffic were captured and analyzed. The TCP three-way handshake was verified, SSH key exchange and encrypted traffic were inspected, and the TLS 1.3 handshake was analyzed in detail.

The HTTP and HTTPS captures demonstrated the difference between unencrypted application traffic and TLS-protected application data.

This lab provides a practical foundation for future network traffic inspection, vulnerability analysis, IDS/IPS, and SIEM monitoring exercises.




















































































































































































































































