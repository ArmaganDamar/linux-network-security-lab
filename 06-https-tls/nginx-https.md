# HTTPS / TLS Configuration

## Objective

Configure Nginx to serve HTTPS traffic using a self-signed TLS certificate and allow HTTPS traffic through the UFW firewall.

---

## Environment

- Server: Ubuntu Server
- Web Server: Nginx
- Security Testing: Kali Linux
- Virtualization: VMware Workstation
- Network: 192.168.153.0/24
- Server IP: 192.168.153.129

---

## 1. TLS Certificate

A self-signed TLS certificate was generated using OpenSSL.

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/nginx/ssl/lab.key \
-out /etc/nginx/ssl/lab.crt \
-subj "/C=TR/ST=Istanbul/L=Istanbul/O=Security-Lab/OU=Homelab/CN=192.168.153.129" \
-addext "subjectAltName=IP:192.168.153.129"
```

The certificate and private key were stored in:
```bash
/etc/nginx/ssl/lab.crt
/etc/nginx/ssl/lab.key
```
The private key is not included in this repository.

## 2. Nginx HTTPS Configuration

Nginx was configured to listen on TCP port 443.

```bash
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name 192.168.153.129;

    ssl_certificate /etc/nginx/ssl/lab.crt;
    ssl_certificate_key /etc/nginx/ssl/lab.key;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

The Nginx configuration was tested with:
```bash
sudo nginx -t
```

Result:
```bash
syntax is ok
test is successful
```

## 3. Firewall Configuration

HTTPS traffic was allowed through UFW:

```bash
sudo ufw allow 443/tcp
```
The resulting firewall configuration included:
| Port | Protocol | Service |
| ---- | -------- | ------- |
| 22   | TCP      | SSH     |
| 80   | TCP      | HTTP    |
| 443  | TCP      | HTTPS   |

## 4. Listening Services

Listening ports were verified using:
```bash
sudo ss -tulpn
```

Nginx was confirmed to be listening on:
```bash
0.0.0.0:80
0.0.0.0:443
[::]:80
[::]:443
```

## 5. HTTPS Testing

HTTPS was tested locally from the Ubuntu server:

```bash
curl -k https://192.168.153.129
```
The request successfully returned the custom Security Homelab HTML page.

## 6. Nmap Verification

The server was scanned from Kali Linux.

### Port Scan
```bash
nmap -p 22,80,443 192.168.153.129
```
Result:
| Port    | State | Service |
| ------- | ----- | ------- |
| 22/tcp  | Open  | SSH     |
| 80/tcp  | Open  | HTTP    |
| 443/tcp | Open  | HTTPS   |

Service Detection
```bash
nmap -sV -p 80,443 192.168.153.129
```
Detected services:
| Port    | Service  | Version      |
| ------- | -------- | ------------ |
| 80/tcp  | HTTP     | nginx 1.28.3 |
| 443/tcp | SSL/HTTP | nginx 1.28.3 |

## 7. TLS Certificate Verification

The TLS connection was inspected using:
```bash
openssl s_client -connect 192.168.153.129:443 -servername 192.168.153.129
```
The certificate was identified as self-signed.

Certificate properties:
- Subject: CN=192.168.153.129
- Organization: Security-Lab
- Organizational Unit: Homelab
- Key: RSA 2048-bit
- Signature: SHA256 with RSA
- Validity: 365 days

## 8. Troubleshooting

### 403 Forbidden

During the initial HTTPS test:
```bash
curl -k https://192.168.153.129
```

Nginx returned:
```bash
403 Forbidden
```
The HTTPS connection itself was working, but Nginx did not have the expected index.html file in /var/www/html.

A custom index.html page was created:
```bash
/var/www/html/index.html
```
After reloading Nginx, the HTTPS request successfully returned the page.

This demonstrated the difference between a network/service connectivity problem and a web server/application configuration problem.

## Result

HTTPS was successfully configured on the Ubuntu Server.

Final exposed services:
```bash
22/tcp  → SSH
80/tcp  → HTTP
443/tcp → HTTPS
```
The HTTPS service was verified from Kali Linux using Nmap and OpenSSL.







