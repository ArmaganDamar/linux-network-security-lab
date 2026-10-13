# Lab 17 — Infrastructure as Code with OpenTofu and Docker

## Objective

Learn the core Infrastructure as Code (IaC) workflow by defining, planning, deploying, validating, and removing a local Nginx container with OpenTofu. The lab uses the existing Ubuntu Server and Docker Engine to practice cloud-style provisioning without creating paid cloud resources.

> **Scope:** This lab uses local Docker only. It does not provision or configure resources in Microsoft Azure.

## Environment

- **Operating system:** Ubuntu Server 24.04.1 LTS
- **OpenTofu:** v1.13.1
- **Docker Engine:** 29.9.0
- **Docker provider:** `kreuzwerker/docker` v4.6.0
- **Container image:** `nginx:alpine`
- **Host binding:** `127.0.0.1:8081` → container port `80`
- **Project directory:** `/opt/opentofu-docker-lab`

## Architecture

```text
main.tf
  |
  v
OpenTofu
  |
  v
kreuzwerker/docker provider v4.6.0
  |
  v
Local Docker Engine
  |-- nginx:alpine image (kept locally)
  `-- opentofu-lab-nginx container
        `-- 127.0.0.1:8081 -> container port 80
```

Binding the published port to `127.0.0.1` restricts the service to the Ubuntu host's loopback interface instead of exposing it on all host interfaces.

## Configuration

File: `/opt/opentofu-docker-lab/main.tf`

```hcl
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "4.6.0"
    }
  }
}

provider "docker" {
  host = "unix:///var/run/docker.sock"
}

resource "docker_image" "nginx" {
  name         = "nginx:alpine"
  keep_locally = true
}

resource "docker_container" "web" {
  name  = "opentofu-lab-nginx"
  image = docker_image.nginx.image_id

  ports {
    internal = 80
    external = 8081
    ip       = "127.0.0.1"
  }

  memory      = 128
  memory_swap = 128
  privileged  = false
  restart     = "no"
}

output "nginx_url" {
  value = "http://127.0.0.1:8081"
}
```

The provider version is pinned to `4.6.0` so future initializations use the selected version. OpenTofu created `terraform.lock.hcl` during initialization. The `pids_limit` argument was tested but removed after provider v4.6.0 reported it as unsupported for this resource.

## Procedure

### 1. Install and verify OpenTofu

OpenTofu was installed on Ubuntu using the official Debian installation script. The installed version was verified with:

```bash
tofu version
```

Observed version: `OpenTofu v1.13.1`.

### 2. Initialize the project

```bash
sudo tofu -chdir=/opt/opentofu-docker-lab init
```

**Result:** Initialization succeeded and installed `kreuzwerker/docker` v4.6.0. OpenTofu generated `terraform.lock.hcl` to record provider selection.

### 3. Check formatting

```bash
sudo tofu fmt -check /opt/opentofu-docker-lab/main.tf
```

**Result:** The command completed without a formatting error.

### 4. Review the execution plan

```bash
sudo tofu -chdir=/opt/opentofu-docker-lab plan
```

Plan summary:

```text
Plan: 2 to add, 0 to change, 0 to destroy.
```

The planned resources were the local `nginx:alpine` image and the `opentofu-lab-nginx` container.

### 5. Apply the configuration

```bash
sudo tofu -chdir=/opt/opentofu-docker-lab apply
```

After reviewing the plan, `yes` was entered at the confirmation prompt.

**Result:**

```text
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
nginx_url = "http://127.0.0.1:8081"
```

### 6. Validate Nginx

```bash
curl -I http://127.0.0.1:8081
sudo docker ps
```

**Observed results:**

- The HTTP request returned `HTTP/1.1 200 OK`.
- The `opentofu-lab-nginx` container was running.
- Docker showed the published port as `127.0.0.1:8081->80/tcp`.

### 7. Destroy the managed resources

```bash
sudo tofu -chdir=/opt/opentofu-docker-lab destroy
```

After reviewing the destroy plan, `yes` was entered at the confirmation prompt.

**Result:**

```text
Destroy complete! Resources: 2 destroyed.
```

Confirm that the container was removed and the image remained locally available:

```bash
sudo docker ps -a --filter name=opentofu-lab-nginx
sudo docker images nginx
```

**Observed results:**

- No `opentofu-lab-nginx` container was listed.
- The `nginx:alpine` image remained available locally, consistent with `keep_locally = true`.

## Security Notes

- The container port was bound to `127.0.0.1`, not `0.0.0.0` or the Ubuntu VM's LAN address.
- `privileged = false` was configured.
- The container memory limit was configured as 128 MB; `memory_swap = 128` sets the total memory-plus-swap limit to 128 MB for this provider schema.
- `pids_limit` was not retained because the pinned provider rejected it as an unsupported argument.
- OpenTofu was run with `sudo` to access `/var/run/docker.sock`. Docker socket access is highly privileged and can provide effective root-level control of the host. This approach is suitable only for a controlled lab; review provider versions and do not grant untrusted users access to the Docker socket.
- No Azure account or cloud resources were used, so this exercise did not create Azure resource charges.

## Lessons Learned

- IaC describes a desired state in configuration rather than relying only on manual commands.
- `init`, `fmt`, `plan`, `apply`, service validation, and `destroy` form a repeatable workflow.
- A provider translates the configuration into operations against a target platform; here, that platform was the local Docker Engine.
- Provider schemas are version-specific, so arguments must be checked against the pinned version.
- The configuration and `terraform.lock.hcl` support reproducibility. Review state and other generated files before committing them to a public repository; do not commit secrets.

## References

- [OpenTofu documentation](https://opentofu.org/docs/)
- [OpenTofu Debian/Ubuntu installation](https://opentofu.org/docs/v1.13/intro/install/deb/)
- [Docker provider v4.6.0 — container resource](https://registry.terraform.io/providers/kreuzwerker/docker/4.6.0/docs/resources/container)
- [Docker provider v4.6.0 — image resource](https://registry.terraform.io/providers/kreuzwerker/docker/4.6.0/docs/resources/image)
