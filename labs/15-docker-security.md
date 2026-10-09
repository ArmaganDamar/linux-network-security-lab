# 15 - Docker Security

## Objective

Install Docker Engine on Ubuntu Server and perform practical security checks on container execution, privileges, filesystem access, network exposure, and resource limits.

The main objectives were:

- Install and verify Docker Engine and Docker Compose
- Inspect Docker daemon access and socket permissions
- Review Docker security options
- Compare root and non-root container execution
- Restrict container filesystem access
- Drop Linux capabilities
- Prevent privilege escalation through additional privileges
- Limit container resource consumption
- Restrict published ports to the localhost interface
- Verify container behavior and clean up test resources

---

## Lab Environment

| Component | Details |
|---|---|
| Host | Ubuntu Server |
| Container Engine | Docker Engine 29.9.0 |
| Docker Compose | 5.6.0 |
| Test Images | `hello-world`, `alpine:3.22`, `nginx:alpine` |
| Web Server Container | Nginx |
| Docker Socket | `/var/run/docker.sock` |
| Network Interface | `ens33` |
| Existing Firewall | UFW |
| Container Network Test | `127.0.0.1:8080` |

---

## 1. Docker Installation and Verification

Docker was not initially installed on the Ubuntu Server.

Docker Engine was installed using the official Docker APT repository, together with the Docker CLI, containerd, Buildx, and the Docker Compose plugin.

The installation was verified using:

```bash
sudo systemctl status docker --no-pager
```

The service status confirmed:

```text
Active: active (running)
```

The installed versions were checked with:

```bash
sudo docker --version
docker compose version
```

Observed versions:

```text
Docker Engine: 29.9.0
Docker Compose: v5.6.0
```

A test container was executed:

```bash
sudo docker run --rm hello-world
```

The container returned:

```text
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

This confirmed that Docker could download an image, create a container, execute it, and return its output.

---

## 2. Docker User and Socket Permissions

The current user and group memberships were inspected:

```bash
id
getent group docker
```

The `docker` group existed, but the `armagan` user was not a member.

The Docker socket permissions were inspected using:

```bash
ls -l /var/run/docker.sock
```

Observed permissions:

```text
srw-rw---- 1 root docker ... /var/run/docker.sock
```

The socket was owned by `root:docker` and had permissions `660`.

This means that root and members of the Docker group can access the Docker socket, subject to the host's permission configuration.

The current user was not added to the Docker group during this lab because membership can grant effectively root-equivalent control over the host.

---

## 3. Docker Security Options

The Docker security options were inspected using:

```bash
sudo docker info --format '{{json .SecurityOptions}}'
```

Observed options included:

```text
apparmor
seccomp
cgroupns
```

The output identified the default AppArmor profile and the built-in seccomp profile.

These mechanisms provide additional controls over container processes and their interaction with the host kernel.

Their presence does not, by itself, guarantee that every container is securely configured; container-specific settings and host configuration must also be considered.

---

## 4. Container Inventory

The current container and image inventory was inspected:

```bash
sudo docker ps -a
sudo docker image ls
```

Initially, no containers were listed.

The `hello-world` image was available after the installation test. Additional images were downloaded for the container security exercises.

---

## 5. Default Root Execution

A container was started using Alpine Linux:

```bash
sudo docker run --rm alpine:3.22 id
```

The output included:

```text
uid=0(root) gid=0(root)
```

This demonstrated that the command ran as root inside the container by default.

Container root and host root are distinct execution contexts, but running as root inside a container can increase risk if other isolation controls are weak or a container escape vulnerability is exploited.

For workloads that do not require root, using a non-root container user is generally preferable.

---

## 6. Non-Root Container Execution

The same image was executed using a non-root user ID:

```bash
sudo docker run --rm --user 1000:1000 alpine:3.22 id
```

Observed output:

```text
uid=1000 gid=1000
```

This confirmed that Docker allowed the process to run with the specified non-root UID and GID.

The test demonstrated how container privileges can be reduced by explicitly defining the runtime user.

The numeric UID and GID do not necessarily correspond to named accounts inside the image; they establish the process identity used by the container.

---

## 7. Read-Only Filesystem and Privilege Restrictions

A restricted container was launched with the following options:

```bash
sudo docker run --rm --read-only --cap-drop=ALL --security-opt=no-new-privileges --tmpfs /tmp:rw,noexec,nosuid,size=16m,mode=1777 alpine:3.22 sh -c 'id; touch /tmp/test && echo "tmpfs write: OK"; touch /etc/docker-lab-test || echo "root filesystem write blocked as expected"'
```

The test produced:

```text
uid=0(root) gid=0(root)
tmpfs write: OK
touch: /etc/docker-lab-test: Read-only file system
root filesystem write blocked as expected
```

The results confirmed:

- The container could write to its temporary `/tmp` filesystem.
- The container could not write to its read-only root filesystem.
- All Linux capabilities were dropped through `--cap-drop=ALL`.
- The `no-new-privileges` security option was enabled.

This demonstrated that container restrictions can reduce the available operations even when a process runs as root inside the container.

The test verified the read-only filesystem behavior directly. The individual security effects of capability removal and `no-new-privileges` were configured but were not separately tested through privilege-escalation scenarios.

---

## 8. Nginx Container and Resource Limits

An Nginx container was deployed with a localhost-only port binding and resource restrictions:

```bash
sudo docker run -d --name docker-lab-nginx -p 127.0.0.1:8080:80 --memory=128m --cpus=0.5 --pids-limit=100 nginx:alpine
```

The configuration was inspected using:

```bash
sudo docker inspect --format 'Ports={{json .HostConfig.PortBindings}} Memory={{.HostConfig.Memory}} NanoCpus={{.HostConfig.NanoCpus}} PidsLimit={{.HostConfig.PidsLimit}} Privileged={{.HostConfig.Privileged}}' docker-lab-nginx
```

Observed configuration:

```text
Ports={"80/tcp":[{"HostIp":"127.0.0.1","HostPort":"8080"}]}
Memory=134217728
NanoCpus=500000000
PidsLimit=100
Privileged=false
```

These values correspond to:

| Setting | Value | Purpose |
|---|---|---|
| Published port | `127.0.0.1:8080 → 80` | Restricts the published host endpoint to localhost |
| Memory limit | 128 MiB | Limits container memory consumption |
| CPU limit | 0.5 CPU | Limits container CPU allocation |
| PID limit | 100 | Limits the number of processes and threads counted by the PID controller |
| Privileged mode | `false` | Avoids privileged container mode |

The localhost-only binding reduces network exposure compared with publishing the service on all host interfaces.

Docker port publishing and host firewall behavior should still be assessed separately for any deployment exposed to other machines.

---

## 9. Container Service Verification

The Nginx container was verified using:

```bash
curl -I http://127.0.0.1:8080
```

The service returned:

```text
HTTP/1.1 200 OK
Server: nginx/1.31.6
Content-Type: text/html
```

This confirmed that the container was running, the published port was accessible through the host's loopback interface, and Nginx responded successfully.

---

## 10. Container Cleanup

The temporary Nginx container was removed:

```bash
sudo docker rm -f docker-lab-nginx
```

The final inventory was checked using:

```bash
sudo docker ps -a
```

The command returned an empty container list.

The Nginx test container was therefore successfully removed after testing.

---

## Results

| Assessment Area | Result |
|---|---|
| Docker Engine installation | Completed |
| Docker Compose verification | Completed |
| Docker service verification | Successful |
| `hello-world` container test | Successful |
| Docker socket permission review | Completed |
| Docker group membership review | Completed |
| Docker security options review | Completed |
| Default root execution test | Completed |
| Non-root execution test | Successful |
| Read-only filesystem test | Successful |
| Temporary filesystem write test | Successful |
| Linux capability restrictions | Configured |
| `no-new-privileges` | Configured |
| Localhost-only port binding | Verified |
| Memory and CPU limits | Verified |
| PID limit | Verified |
| Nginx container response | HTTP 200 OK |
| Test container cleanup | Completed |

---

## Security Analysis

This lab demonstrated that container security requires more than simply running an application inside Docker.

The Docker socket and group permissions were reviewed because Docker daemon access can grant powerful control over the host. Root and non-root execution were compared, and a restricted container demonstrated the effect of a read-only root filesystem with a writable temporary filesystem.

The Nginx container was configured with CPU, memory, and PID limits. Its service was published only on the host's loopback interface, reducing unnecessary network exposure.

The exercises also highlighted the importance of applying multiple security controls together rather than relying on a single setting.

---

## Skills Demonstrated

### Docker Administration

- Docker Engine installation
- Docker Compose verification
- Image and container management
- Docker daemon inspection
- Container lifecycle management

### Container Security

- Docker socket permission review
- Docker group security
- Root and non-root execution
- Read-only filesystem configuration
- Linux capability restrictions
- `no-new-privileges`
- AppArmor and seccomp awareness

### Network and Resource Security

- Localhost-only port publishing
- Container port mapping inspection
- Memory limits
- CPU limits
- PID limits
- Service connectivity verification

### Security Testing

- Runtime configuration inspection
- Restriction verification
- Container behavior testing
- Resource configuration validation
- Test environment cleanup

---

## Conclusion

The Docker Security lab provided practical experience installing Docker Engine and applying basic container hardening controls.

The exercises demonstrated the difference between root and non-root execution, verified read-only filesystem behavior, and configured capability restrictions and `no-new-privileges`.

An Nginx container was also deployed with localhost-only port exposure and explicit CPU, memory, and PID limits. The service returned HTTP 200 OK, and the temporary container was removed after testing.

This lab established a foundation for further container security work, including image vulnerability scanning, Docker networking, secrets management, and container runtime monitoring.
