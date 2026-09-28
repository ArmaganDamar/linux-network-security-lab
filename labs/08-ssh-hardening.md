# 08 - SSH Hardening

## Objective

Harden the SSH service on the Ubuntu Server by:

- Enabling SSH key-based authentication
- Disabling password-based SSH authentication
- Preventing direct root password login
- Verifying the configuration from a Kali Linux client

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

## 1. SSH Configuration Review

The active SSH configuration was inspected using:

```bash
sudo sshd -T | grep -E '^(passwordauthentication|pubkeyauthentication|permitrootlogin)'
```

| Setting                  | Initial Value       |
| ------------------------ | ------------------- |
| `PermitRootLogin`        | `prohibit-password` |
| `PubkeyAuthentication`   | `yes`               |
| `PasswordAuthentication` | `yes`               |

Password authentication was still enabled and required hardening.


## 2. SSH Hardening Configuration

A dedicated SSH hardening configuration file was created:
```bash
sudo nano /etc/ssh/sshd_config.d/00-hardening.conf
```

The following configuration was applied:
```bash
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin prohibit-password
```

The configuration was then validated:
```bash
sudo sshd -t
```
No syntax errors were reported.

The SSH service was reloaded:
```bash
sudo systemctl reload ssh
```

## 3. Configuration Verification:

The effective SSH configuration was verified
```bash
sudo sshd -T | grep -E '^(passwordauthentication|pubkeyauthentication|permitrootlogin)'
```

Result:
```bash
permitrootlogin prohibit-password
pubkeyauthentication yes
passwordauthentication no
```
The final configuration confirms that:

- Public key authentication is enabled.
- Password-based authentication is disabled.
- Root password authentication is prohibited.

## 4. SSH Key Authentication

An ED25519 key pair was generated on the Kali Linux client:
```bash
ssh-keygen -t ed25519
```

The public key was copied to the Ubuntu Server:
```bash
ssh-copy-id armagan@192.168.153.129
```

The key-based connection was then tested:
```bash
ssh armagan@192.168.153.129
```
The connection was established successfully without entering the user's password.

## 5. Password Authentication Test

Password authentication was explicitly tested from Kali Linux:
```bash
ssh -o PreferredAuthentications=password \
-o PubkeyAuthentication=no \
armagan@192.168.153.129
```

Result:
```bash
armagan@192.168.153.129: Permission denied (publickey).
```
This confirmed that password-based SSH authentication was disabled successfully.

## 6. SSH Service Verification

The SSH service was verified after applying the configuration:
```bash
sudo systemctl status ssh
```
The service remained active and running.

The configuration was also validated with:
```bash
sudo sshd -t
```
No configuration errors were reported.

---

## Results

| Security Control | Result |
|---|---|
| SSH service | Active and running |
| Public key authentication | Enabled |
| Password authentication | Disabled |
| Root password login | Prohibited |
| ED25519 authentication | Successful |
| SSH configuration syntax | Valid |

The effective SSH configuration confirmed that `PasswordAuthentication` is set to `no` and `PubkeyAuthentication` is set to `yes`.

A key-based SSH connection from Kali Linux to the Ubuntu Server was established successfully without entering a password.

A password-only authentication attempt was also tested and rejected with:

```text
Permission denied (publickey).
```

## Conclusion

SSH hardening was successfully completed on the Ubuntu Server.

Password-based SSH authentication was disabled and replaced with ED25519 public key authentication. The configuration was validated locally and tested remotely from Kali Linux.

The final SSH configuration provides a stronger authentication mechanism while maintaining remote administrative access.













































