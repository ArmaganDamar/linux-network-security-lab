# Lab 07 - Nginx Hardening

## Objective

Harden an Nginx web server running on Ubuntu Server and verify the security configuration from a separate Kali Linux machine.

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

## Tasks Completed

- Enabled HTTPS on Nginx
- Generated a self-signed TLS certificate
- Disabled Nginx version disclosure
- Added security response headers
- Configured Content Security Policy
- Configured UFW firewall rules
- Verified the configuration from Kali Linux
- Performed HTTP header enumeration with Nmap

---

## 1. Nginx Version Disclosure

Nginx was configured with:

```nginx
server_tokens off;
```

Verification:
```bash
sudo nginx -T 2>/dev/null | grep -n "server_tokens"
```

Expected result:
```nginx
server_tokens off;
```
The HTTP response no longer exposes the Nginx version.

## 2. Security Headers

The following security headers were configured:
```nginx
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Content-Security-Policy "default-src 'self';" always;
```

### Purpose

X-Content-Type-Options

Prevents browsers from MIME-sniffing responses.
```bash
X-Content-Type-Options: nosniff
```

X-Frame-Options

Restricts how the page can be embedded in frames.
```bash
X-Frame-Options: SAMEORIGIN
```

Referrer-Policy

Controls the amount of referrer information sent with requests.
```bash
Referrer-Policy: strict-origin-when-cross-origin
```

Content-Security-Policy

Restricts resource loading to the same origin.
```bash
Content-Security-Policy: default-src 'self';
```

## 3. HTTPS Configuration

A self-signed TLS certificate was generated for the lab environment.

Certificate configuration:
```bash
CN=192.168.153.129
Subject Alternative Name:
IP:192.168.153.129
```

TLS verification was performed from Kali Linux using:
```bash
openssl s_client -connect 192.168.153.129:443 \
-servername 192.168.153.129
```
The certificate and TLS connection were successfully established.

## 4. Firewall Configuration

UFW was enabled and configured to allow:
```bash
22/tcp
80/tcp
443/tcp
```

Verification:
```bash
sudo ufw status numbered
```

## 5. Local Verification

The HTTPS service was tested locally:
```bash
curl -k -I https://192.168.153.129
```
The expected security headers were returned successfully.

## 6. Remote Verification From Kali

The server was tested from Kali Linux.

### HTTP Header Enumeration
```bash
nmap -p 443 --script http-headers 192.168.153.129
```

Nmap successfully identified:
```bash
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Referrer-Policy: strict-origin-when-cross-origin
Content-Security-Policy: default-src 'self';
```

Curl Verification
```bash
curl -k -I https://192.168.153.129
```
The same security headers were observed from the external lab client.

## 7. Results

The Nginx server was successfully hardened against several common web security issues.

The configuration was validated locally and independently from Kali Linux.

### Security Improvements
- Reduced server information disclosure
- Added MIME sniffing protection
- Added clickjacking protection
- Restricted referrer information
- Added Content Security Policy
- Enabled HTTPS
- Restricted network access with UFW
- Verified security controls from a separate host

### Skills Demonstrated
- Linux administration
- Nginx configuration
- Web server hardening
- HTTPS / TLS
- HTTP security headers
- UFW firewall
- Nmap
- OpenSSL
- Curl
- Security validation
- Network troubleshooting































