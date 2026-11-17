# Ansible Role: `github_runner_exporter`

An Ansible role to deploy and configure [`rezizter/github-runner-prometheus-exporter`](https://github.com/rezizter/github-runner-prometheus-exporter) as a hardened `systemd` service across major Linux distributions, including **Amazon Linux 2023**, **RHEL / Rocky / AlmaLinux**, **Ubuntu**, and **Debian**.

## Features

* **Cross-Platform & Multi-Arch Support:** Supports `x86_64` (`amd64`) and `aarch64` (`arm64`) on Amazon Linux 2023, Enterprise Linux (8/9), Ubuntu (22.04/24.04), and Debian (12/13).
* **Flexible Installation Methods:** Build directly from source using an isolated Go toolchain (`source` mode) or download prebuilt release binaries (`binary` mode).
* **Security Hardening:** Runs under a dedicated unprivileged system user (`github-exporter`) with strict `systemd` sandboxing (`NoNewPrivileges`, `ProtectSystem=strict`, `PrivateTmp`, etc.).
* **CI Tested:** Verified with Molecule and Docker across all target OS families via GitHub Actions.

---

## Requirements

* **Ansible:** `>= 2.14`
* **Target Host:** Linux system using `systemd` with internet access to GitHub (and `go.dev` if installing from source).
* **Privileges:** Root or `sudo` (`become: true`) access on target nodes.

---

## Role Variables

Available variables and their default values (defined in `defaults/main.yml`):

| Variable | Default | Description |
| :--- | :--- | :--- |
| `github_runner_exporter_install_method` | `"source"` | Installation mode: `"source"` (compiles via `go build`) or `"binary"` (downloads GitHub release archive). |
| `github_runner_exporter_repo_url` | `"https://github.com/rezizter/github-runner-prometheus-exporter.git"` | Git repository URL used when building from source. |
| `github_runner_exporter_version` | `"main"` | Git branch, tag, or commit hash (for `source`), or release tag (for `binary`). |
| `github_runner_exporter_go_version` | `"1.23.4"` | Go version installed to `/usr/local/go` when building from source. |
| `github_runner_exporter_user` | `"github-exporter"` | System user running the exporter service. |
| `github_runner_exporter_group` | `"github-exporter"` | System group for the exporter user and config directory. |
| `github_runner_exporter_install_dir` | `"/usr/local/bin"` | Directory where the compiled/extracted binary is placed. |
| `github_runner_exporter_config_dir` | `"/etc/github-runner-exporter"` | Directory storing `exporter.env`. |
| `github_runner_exporter_src_dir` | `"/usr/local/src/github-runner-prometheus-exporter"` | Directory where the repository is cloned when building from source. |
| `github_runner_exporter_service_name` | `"github-runner-exporter"` | Name of the `systemd` unit. |
| `github_runner_exporter_service_enabled` | `true` | Whether the service starts automatically on boot. |
| `github_runner_exporter_service_state` | `"started"` | Desired state of the `systemd` service. |
| `github_runner_exporter_cli_args` | `""` | Additional command-line flags appended to `ExecStart`. |
| `github_runner_exporter_env_vars` | `{ PORT: "9877" }` | Dictionary of environment variables written to `/etc/github-runner-exporter/exporter.env`. |

---

## How to Run the Role

### 1. Install the Role

Add the role to your `requirements.yml` file:

```yaml
---
roles:
  - name: github_runner_exporter
    src: https://github.com/<your-org>/ansible-role-github-runner-exporter.git
    version: main
```

Install it using `ansible-galaxy`:

```bash
ansible-galaxy install -r requirements.yml
```

*(Alternatively, place the role directory directly inside your project's `roles/github_runner_exporter/` folder.)*

### 2. Create an Inventory File (`inventory.ini`)

```ini
[monitoring_nodes]
al2023-node.example.com ansible_user=ec2-user
ubuntu-node.example.com ansible_user=ubuntu
```

### 3. Create a Playbook (`deploy-exporter.yml`)

Pass your GitHub Personal Access Token (PAT) and organization/repository targets using `github_runner_exporter_env_vars` (ideally encrypted via Ansible Vault):

```yaml
---
- name: Deploy GitHub Runner Prometheus Exporter
  hosts: monitoring_nodes
  become: true

  vars:
    github_runner_exporter_install_method: "source"
    github_runner_exporter_version: "main"
    github_runner_exporter_env_vars:
      PORT: "9101"
      GITHUB_TOKEN: "{{ vault_github_token }}"
      GITHUB_ORG: "my-github-org"
      # GITHUB_REPO: "my-github-org/my-repo"

  roles:
    - role: github_runner_exporter
```

### 4. Execute the Playbook

Run Ansible against your inventory:

```bash
ansible-playbook -i inventory.ini deploy-exporter.yml --ask-vault-pass
```

### 5. Verify Deployment

Once the playbook finishes, confirm the service is active and exposing Prometheus metrics:

```bash
# Check systemd service status
sudo systemctl status github-runner-exporter

# Query the metrics endpoint
curl -s http://localhost:9101/metrics | head -n 20
```

---

## Local Testing with Molecule

This role includes a Molecule test suite using Docker containers with `systemd` enabled.

### Prerequisites

```bash
pip install ansible-core molecule molecule-plugins[docker] docker yamllint ansible-lint
```

### Run Tests

```bash
# Test against Amazon Linux 2023 (default)
molecule test

# Test against a specific distribution
MOLECULE_DISTRO=geerlingguy/docker-ubuntu2404-ansible:latest molecule test
MOLECULE_DISTRO=geerlingguy/docker-rockylinux9-ansible:latest molecule test
MOLECULE_DISTRO=geerlingguy/docker-debian12-ansible:latest molecule test
```

---

## License

GPL-3.0 license
