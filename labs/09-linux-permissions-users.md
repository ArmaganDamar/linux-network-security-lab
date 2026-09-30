# 09 - Linux Permissions & Users

## Objective

Learn and demonstrate Linux user, group, file permission, directory permission, and ACL concepts through a controlled access management lab.

The main objectives were:

- Creating and managing Linux users and groups
- Understanding file and directory permissions
- Using `chmod`, `chown`, and `usermod`
- Understanding the difference between file and directory permissions
- Implementing the principle of least privilege
- Configuring user-specific permissions using ACLs

---

## Lab Environment

| Component | Details |
|---|---|
| Server | Ubuntu Server |
| Main User | `armagan` |
| Test Users | `developer`, `auditor` |
| Group | `webteam` |
| Lab Directory | `/srv/security-lab` |
| Tools | `chmod`, `chown`, `usermod`, `setfacl`, `getfacl`, `stat` |

---

## 1. Current User and Group Information

The current user and group configuration was inspected using:

```bash
whoami
id
groups
```

The main lab user was:
```bash
armagan
```
The `armagan` user has administrative privileges through the `sudo` group.

## 2. Users and Group Creation

A dedicated group was created for the lab:
```bash
sudo groupadd webteam
```

Two test users were created:
```bash
sudo useradd -m -s /bin/bash developer
sudo useradd -m -s /bin/bash auditor
```

The `developer` user was added to the `webteam` group:
```bash
sudo usermod -aG webteam developer
```

User and group membership were verified using:
```bash
id developer
id auditor
```

## 3. Lab Directory

A dedicated directory was created:
```bash
sudo mkdir -p /srv/security-lab
```

The ownership was configured as:
```bash
sudo chown root:webteam /srv/security-lab
```

The directory permissions were initially configured using:
```bash
sudo chmod 2775 /srv/security-lab
```
The `2`enables the setgid bit, causing newly created files within the directory to inherit the directory's group.

The directory was later restricted using:
```bash
sudo chmod 2770 /srv/security-lab
```

Final directory permissions:
```bash
drwxrws---
```

## 4. File Permissions

A test file was created by the `developer` user:
```bash
sudo -u developer bash -c 'echo "Security Lab Data" > /srv/security-lab/report.txt'
```

The file permissions were configured as:
```bash
sudo chmod 640 /srv/security-lab/report.txt
```

The resulting permissions were:
```bash
-rw-r----- 640 developer webteam report.txt
```

This means:
```bash
Owner  → read + write
Group  → read
Others → no access
```

The effective permissions were verified using:
```bash
stat -c '%A %a %U %G %n' /srv/security-lab/report.txt
```

## 5. Permission Testing
### Developer

The `developer` user was able to read the file:
```bash
sudo -u developer cat /srv/security-lab/report.txt
```

The `developer` user was also able to modify the file:
```bash
sudo -u developer bash -c 'echo "Developer update" >> /srv/security-lab/report.txt'
```

### Auditor

The `auditor`user initially did not belong to the `webteam` group and could not access the file.

After being added to the group:
```bash
sudo usermod -aG webteam auditor
```
the `auditor` user was able to read the file through the group's read permission.

However, the `auditor` user was not able to modify the file:
```bash
sudo -u auditor bash -c 'echo "Auditor update" >> /srv/security-lab/report.txt'
```
Result:
```bash
Permission denied
```

## 6. File Permissions vs Directory Permissions

A separate experiment was performed to demonstrate the difference between file and directory permissions.

A test directory was created:
```bash
sudo mkdir /srv/security-lab/permission-demo
sudo chown developer:webteam /srv/security-lab/permission-demo
sudo chmod 770 /srv/security-lab/permission-demo
```

A test file was created:
```bash
sudo -u developer bash -c 'echo "This is a permissions experiment." > /srv/security-lab/permission-demo/known.txt'
```

The file was configured with:
```bash
sudo chmod 640 /srv/security-lab/permission-demo/known.txt
```

The `auditor` user was given read permission to the file:
```bash
sudo setfacl -m u:auditor:r-- /srv/security-lab/permission-demo/known.txt
```

The directory was then configured so that `auditor` only had execute permission:
```bash
sudo setfacl -m u:auditor:--x /srv/security-lab/permission-demo
```

### Observed behavior

The `auditor` user could not list the directory:
```bash
sudo -u auditor ls /srv/security-lab/permission-demo
```

Result:
```bash
Permission denied
```
However, the `auditor` user could access the known file directly:
```bash
sudo -u auditor cat /srv/security-lab/permission-demo/known.txt
```

The file contents were successfully displayed.

This demonstrated that directory permissions and file permissions serve different purposes.


## 7. ACL Configuration

The `auditor` user was removed from the `webteam` group:
```bash
sudo gpasswd -d auditor webteam
```

The directory was restricted using:
```bash
sudo chmod 2770 /srv/security-lab
```

A user-specific ACL was then applied:
```bash
sudo setfacl -m u:auditor:r-x /srv/security-lab
```

The `report.txt` file was given a read-only ACL entry:
```bash
sudo setfacl -m u:auditor:r-- /srv/security-lab/report.txt
```

The resulting ACL configuration was verified using:
```bash
getfacl /srv/security-lab
```
and:
```bash
sudo getfacl /srv/security-lab/report.txt
```

## 8. Final Access Model

The final access model was designed as:
```bash
/srv/security-lab
│
├── developer → read / write
└── auditor   → read only
```
The `auditor` user was able to:

- Read `report.txt`
- Access the directory

The `auditor` user was not able to:

- Modify `report.txt`
- Create new files
- Write to existing files

This configuration demonstrates the principle of least privilege.



## Results

Observed results
| Test                           | Result                  |
| ------------------------------ | ----------------------- |
| User and group creation        | Successful              |
| `developer` added to `webteam` | Successful              |
| File ownership and permissions | Configured successfully |
| `developer` read access        | Successful              |
| `developer` write access       | Successful              |
| `auditor` read access          | Successful              |
| `auditor` write access         | Denied                  |
| `auditor` file creation        | Denied                  |
| Directory permission behavior  | Verified                |
| ACL configuration              | Successful              |
| Least privilege model          | Implemented             |

The experiments demonstrated the difference between file permissions and directory permissions and showed how ACLs can be used to provide user-specific access without granting unnecessary group privileges.

## Skills Demonstrated

### Linux Administration
- User management
- Group management
- File ownership
- File permissions
- Directory permissions
- `sudo`
- `chmod`
- `chown`
- `usermod`
- `gpasswd`
### Access Control
- Linux permission model
- `rwx` permissions
- Numeric permissions
- Setgid directories
- ACLs
- User-specific permissions
- Least privilege
### Troubleshooting
- Permission denied analysis
- File vs directory permission troubleshooting
- Access verification using different users

## Conclusion

The Linux permissions and users lab demonstrated how Linux controls access to files and directories through ownership, groups, traditional permissions, and ACLs.

A controlled access model was implemented where the `developer` user has read/write access while the `auditor` user is restricted to read-only access.

The configuration was verified through practical access tests using separate Linux users.



















































