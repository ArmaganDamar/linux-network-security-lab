# Lab 17 — Infrastructure as Code with OpenTofu and Docker

## Objective

Practice Infrastructure as Code (IaC) workflows by defining, planning, deploying, validating, and destroying a local container with OpenTofu. This lab uses a local Docker Engine to learn the workflow without provisioning paid cloud resources.

> **Scope note:** This is a local IaC exercise inspired by cloud infrastructure workflows. It does **not** create or configure resources in Microsoft Azure.

## Environment

- OS: Ubuntu Server 24.04.1 LTS
- OpenTofu: v1.13.1
- Docker Engine: 29.9.0
- Docker provider: `kreuzwerker/docker` v4.6.0
- Container image: `nginx:alpine`
- Published endpoint: `127.0.0.1:8081` → container port `80`

## Architecture

```text
OpenTofu configuration (main.tf)
            |
            v
Docker provider v4.6.0
            |
            v
Local Docker Engine
            |
            +-- nginx:alpine image (kept locally)
            |
            +-- opentofu-lab-nginx container
                    |
                    +-- 127.0.0.1:8081 -> container:80
```

The host-side binding uses `127.0.0.1`, so the published Nginx port is intended to be reachable from the Ubuntu host itself, not directly from other network hosts.

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

The Docker provider version is pinned to `4.6.0`. The attempted `pids_limit` argument was removed after the provider reported it as unsupported by this resource schema.

## Procedure

### 1. Initialize the project

```bash
sudo tofu -chdir=/opt/opentofu-docker-lab init
```

Result: OpenTofu successfully initialized and installed `kreuzwerker/docker` v4.6.0. It also created `terraform.lock.hcl` to record the provider selection.

### 2. Validate formatting

```bash
sudo tofu fmt -check /opt/opentofu-docker-lab/main.tf
```

Result: the command completed without a formatting error.

### 3. Review the execution plan

```bash
sudo tofu -chdir=/opt/opentofu-docker-lab plan
```

Plan summary:

```text
Plan: 2 to add, 0 to change, 0 to destroy.
```

The two managed resources were the Docker image and the Nginx container.

### 4. Apply the configuration

```bash
sudo tofu -chdir=/opt/opentofu-docker-lab apply
```

After reviewing the plan, `yes` was entered at the confirmation prompt.

Result:

```text
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
nginx_url = "http://127.0.0.1:8081"
```

### 5. Verify the service

```bash
curl -I http://127.0.0.1:8081
sudo docker ps
```

Observed results:

- HTTP returned `HTTP/1.1 200 OK`.
- The `opentofu-lab-nginx` container was running.
- Docker showed the port binding as `127.0.0.1:8081->80/tcp`.

### 6. Destroy managed resources

```bash
sudo tofu -chdir=/opt/opentofu-docker-lab destroy
```

After reviewing the destroy plan, `yes` was entered at the confirmation prompt.

Result:

```text
Destroy complete! Resources: 2 destroyed.
```

Verification commands:

```bash
sudo docker ps -a --filter name=opentofu-lab-nginx
sudo docker images nginx
```

Observed results:

- No `opentofu-lab-nginx` container was listed.
- The local `nginx:alpine` image remained available, consistent with `keep_locally = true`.

## Security Notes

- The container port was published on `127.0.0.1`, rather than all host interfaces.
- `privileged = false` was set.
- The container memory limit was set to 128 MB; `memory_swap = 128` sets the total memory-plus-swap limit to 128 MB for this provider schema.
- The `pids_limit` argument was not used because provider v4.6.0 rejected it as unsupported.
- OpenTofu was run with `sudo` because it interacted with `/var/run/docker.sock`. Access to the Docker socket is highly privileged and can effectively grant root-level control of the host. Use this pattern only in a controlled lab, review provider code and pin provider versions, and do not give untrusted users access to the Docker socket.
- No Azure account or cloud resources were used in this lab, so it did not incur Azure resource charges.

## Lessons Learned

- OpenTofu describes the desired state in code and uses a plan before making changes.
- A provider translates IaC configuration into operations against the target platform—in this case, the local Docker Engine.
- `init`, `fmt`, `plan`, `apply`, service validation, and `destroy` form a repeatable IaC workflow.
- Provider schemas matter: arguments must be supported by the pinned provider version.
- State and the lock file are important for reproducibility. Keep the project configuration and `terraform.lock.hcl` under version control; do not commit sensitive files or state without reviewing their contents.

## References

- [OpenTofu documentation](https://opentofu.org/docs/)
- [OpenTofu Debian/Ubuntu installation](https://opentofu.org/docs/v1.13/intro/install/deb/)
- [Docker provider v4.6.0 — container resource](https://registry.terraform.io/providers/kreuzwerker/docker/4.6.0/docs/resources/container)
- [Docker provider v4.6.0 — image resource](https://registry.terraform.io/providers/kreuzwerker/docker/4.6.0/docs/resources/image)
