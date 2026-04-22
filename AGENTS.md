# Agents

## Project

Ansible collection (`k3s.orchestration`) for deploying a K3s Kubernetes cluster. The repo root IS the collection root.

## Requirements

- Control node: `ansible-core >= 2.15.0`
- External collections (install with `ansible-galaxy collection install -r collections/requirements.yml`):
  - `community.general >= 7.0.0`
  - `ansible.posix >= 1.5.0`
  - `community.kubernetes`

## Key Commands

```bash
# Lint
yamllint .
ansible-lint

# Build collection artifact
ansible-galaxy collection build

# Deploy cluster
ansible-playbook playbooks/site.yml -i inventory.yml

# Upgrade cluster (update k3s_version in inventory first)
ansible-playbook playbooks/upgrade.yml -i inventory.yml
```

## Directory Layout

```
roles/           # Ansible roles (prereq, airgap, raspberrypi, k3s_server, k3s_agent, k3s_upgrade)
playbooks/       # Playbooks (site.yml, upgrade.yml, reset.yml, reboot.yml)
k8s/             # K8s manifests and Helm values (cilium/, helm/)
  cilium/        # Cilium CNI (uses kustomize + Helm)
  helm/          # Helm values for Prometheus, Minecraft, RCON
```

## site.yml installs more than K3s

After provisioning nodes, it also installs:
- kube-prometheus-stack (monitoring) via Helm
- Cilium CNI via kustomize + Helm
- Minecraft server via Helm

## CI

- Lint: `yamllint .` then `ansible-lint`
- Build: `ansible-galaxy collection build` → produces `k3s-orchestration-*.tar.gz`
- Run on: push to master (build), PR + push (lint)

## Inventory

Copy `inventory-sample.yml` to `inventory.yml` and configure:
- `k3s_version` (required)
- `token` (required; generate with `openssl rand -base64 64`)
- `api_endpoint` (defaults to first server host)
- `server` group = control plane, `agent` group = workers
- Multiple servers in `server` group → HA mode (embedded etcd, odd number required)

## Local Testing

Vagrant (`vagrant up`) provisions 5 nodes: 3 servers + 2 agents on `10.10.10.100-104`. Default provider is `libvirt`, `virtualbox` is fallback.

## YAML Conventions

- Max line length: 180 (warn in yamllint, ignore in ansible-lint)
- `truthy` values must be `true`/`false` (not `yes`/`no`)
- Braces max 1 space inside
- Octal values must use explicit `0o` prefix
