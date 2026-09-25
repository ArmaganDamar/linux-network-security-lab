# Linux, Network & Security Homelab

A hands-on cybersecurity and infrastructure homelab built with VMware Workstation, Ubuntu Server and Kali Linux.

The goal of this project is to develop practical skills in Linux system administration, networking, network security, firewall configuration, service discovery and security monitoring.

---

## Lab Environment

| Component | Technology |
|---|---|
| Virtualization | VMware Workstation |
| Server | Ubuntu Server |
| Security Testing | Kali Linux |
| Web Server | Nginx |
| Firewall | UFW |
| Network Scanning | Nmap |
| Network | VMware NAT |

---

## Lab Architecture

```text
                    VMware NAT Network
                           |
              192.168.153.0/24
                           |
              +------------+------------+
              |                         |
        Ubuntu Server               Kali Linux
        Linux Server                Security Lab
              |                         |
        +-----+-----+                   |
        |           |                   |
      SSH         HTTP             Nmap / Testing
     TCP/22      TCP/80
        |           |
        +-----+-----+
              |
             UFW
           Firewall
```

## Completed Labs

### 01 - Linux Server Setup

- Installed and configured Ubuntu Server
- Inspected network interfaces using `ip addr`
- Inspected routing using `ip route`
- Tested external connectivity using `ping`
- Configured and tested SSH
- Connected to Ubuntu Server remotely using SSH

### 02 - Nginx Web Server

- Installed Nginx
- Enabled and started the Nginx service
- Verified the service using `systemctl`
- Tested HTTP locally using `curl`
- Tested HTTP remotely using the server IP
- Identified TCP/80 as the HTTP service

### 03 - UFW Firewall

Configured Ubuntu's UFW firewall.

**Allowed services:**

| Port | Protocol | Service |
|---|---|---|
| 22 | TCP | SSH |
| 80 | TCP | HTTP |

Firewall configuration was verified using:

```bash
sudo ufw status numbered
sudo ufw status verbose
```

### 04 - Network Service Enumeration

Used ss to inspect listening network services:

```bash
sudo ss -tulpn
```
Identified the following services:

- SSH
- HTTP / Nginx
- DNS / systemd-resolved
- Chrony

### 05 - Nmap Network Scanning

Performed network discovery and service enumeration from Kali Linux.

Basic host scan:
```bash
nmap <TARGET_IP>
```

Service and version detection:
```bash
nmap -sV <TARGET_IP>
```

Specific port scan:
```bash
nmap -p 22,80,443 <TARGET_IP>
```

Observed results:

| Port    | State    | Service       |
| ------- | -------- | ------------- |
| 22/tcp  | Open     | SSH / OpenSSH |
| 80/tcp  | Open     | HTTP / Nginx  |
| 443/tcp | Filtered | HTTPS         |

The Nmap results were compared with the services detected locally on the Ubuntu server using ss.

---

## Skills Practiced

### Linux
- Ubuntu Server administration
- Systemd
- Service management
- SSH
- Network configuration
- Process and service inspection
- Basic troubleshooting

### Networking
- IPv4 addressing
- Subnetting
- Default gateways
- TCP/IP
- DNS
- ICMP
- TCP ports
- Network connectivity troubleshooting

### Security
- Firewall configuration
- UFW
- Port enumeration
- Service discovery
- Nmap
- Attack surface identification
- Basic network security concepts

### Web Infrastructure
- Nginx
- HTTP
- Service management
- Local and remote web service testing

## Tools
- VMware Workstation
- Ubuntu Server
- Kali Linux
- Nmap
- UFW
- Nginx
- OpenSSH
- Linux CLI

## Future Labs

This homelab will be expanded with additional security and infrastructure exercises:

- [ ] HTTPS and TLS configuration
- [ ] Nginx hardening
- [ ] SSH hardening
- [ ] Linux permissions and users
- [ ] Log analysis
- [ ] Wireshark network analysis
- [ ] Network traffic inspection
- [ ] Vulnerability scanning
- [ ] IDS/IPS
- [ ] Docker security
- [ ] Active Directory lab
- [ ] Cloud infrastructure
- [ ] AWS/Azure security fundamentals
- [ ] SIEM and security monitoring
- [ ] Incident response exercises

---

## Disclaimer

This lab is an isolated educational environment created for learning Linux administration, networking and cybersecurity.

All security testing is performed against systems owned and controlled by me.
