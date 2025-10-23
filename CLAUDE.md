# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a GitOps repository for managing a Kubernetes cluster using Flux CD. The cluster runs k3s on Ubuntu nodes and uses a declarative, Git-driven approach for infrastructure and application deployment.

## Common Commands

### Task Commands
All commands use `task` (Taskfile). The main Taskfile is at the root, with included task files in `.taskfiles/`.

**Ansible Operations:**
- `task ansible:deps` - Upgrade Ansible galaxy dependencies
- `task ansible:list` - List all hosts
- `task ansible:playbook:ubuntu-prepare` - Prepare k8s nodes for k3s
- `task ansible:playbook:k3s-install` - Install k3s on nodes
- `task ansible:playbook:k3s-nuke` - Remove k3s from nodes
- `task ansible:playbook:ubuntu-upgrade` - Upgrade OS on all nodes
- `task ansible:adhoc:ping` - Ping all hosts
- `task ansible:adhoc:reboot` - Reboot all k8s nodes

**Flux/Cluster Operations:**
- `task flux:sync` - Sync flux-system with Git repository
- `task cluster:verify` - Verify flux prerequisites
- `task cluster:install` - Install Flux into the cluster
- `task cluster:reconcile` - Force Flux to pull changes from Git
- `task cluster:hr-restart` - Restart all failed Helm releases
- `task cluster:nodes` - List cluster nodes
- `task cluster:pods` - List all pods
- `task cluster:kustomizations` - List all kustomizations
- `task cluster:helmreleases` - List all helm releases
- `task cluster:resources` - Gather common resources (useful for support)

### Environment Variables
- `KUBECONFIG` is set to `{{.PROJECT_DIR}}/provision/kubeconfig` in Taskfile.yml

## Architecture

### Flux CD GitOps Structure

The repository follows a hierarchical Flux CD structure under `cluster/`:

1. **base/** - Flux entrypoint
   - Contains the main `flux-system` Kustomization pointing to this directory
   - Defines `core` and `apps` Kustomizations
   - Contains `cluster-settings.yaml` (ConfigMap) and `cluster-secrets.sops.yaml` (Secret) for variable substitution
   - `flux-system/charts/` contains HelmRepository and GitRepository source definitions

2. **core/** - Critical infrastructure (prune: false)
   - Namespaces are defined here
   - Essential services like Longhorn storage
   - Applied before apps, never pruned by Flux

3. **apps/** - User applications (prune: true)
   - Organized by namespace (default, home, media, monitoring, data, networking, etc.)
   - Each app typically has a `ks.yaml` (Kustomization) that points to its `app/` subdirectory
   - Apps can have multiple Kustomizations with dependencies (e.g., cert-manager and cert-manager-issuers)

### Kustomization Pattern

Applications follow this structure:
```
cluster/apps/<namespace>/<app-name>/
├── ks.yaml                    # Flux Kustomization(s) pointing to subdirs
├── app/
│   ├── kustomization.yaml     # Kustomize resources list
│   ├── helmrelease.yaml       # Helm chart deployment (if using Helm)
│   └── *.yaml                 # Other k8s resources
```

### Secret Management

- SOPS is used for encrypting secrets (files ending in `.sops.yaml`)
- Flux decrypts secrets using the `sops-gpg` secret in `flux-system` namespace
- The Age key for SOPS is expected at `~/.config/sops/age/keys.txt`
- Variable substitution is performed using `cluster-settings` ConfigMap and `cluster-secrets` Secret

### Dependencies

- Kustomizations can specify `dependsOn` to ensure proper ordering
- Health checks verify resources are ready before dependent resources are applied
- Example: `cert-manager-issuers` depends on `cert-manager`

## Provisioning

### Ansible
- Inventory: `provision/ansible/inventory/hosts.yml`
- Host variables in `provision/ansible/inventory/host_vars/` (SOPS encrypted)
- Group variables in `provision/ansible/inventory/group_vars/`
- Playbooks handle Ubuntu preparation, k3s installation, and cluster teardown

### Initial Setup Requirements
Per setup.md, nodes need:
- `open-iscsi nfs-common exfat-fuse exfat-utils` packages
- Multipath configuration for Longhorn (blacklist `devnode "^sd[a-z0-9]+"`)

## Key Technologies

- **Flux CD**: GitOps continuous delivery
- **k3s**: Lightweight Kubernetes distribution
- **Cilium**: CNI with Gateway API support (replacing Ingress)
- **Longhorn**: Distributed block storage
- **cert-manager**: TLS certificate management
- **CloudNative-PG**: PostgreSQL operator
- **SOPS**: Secret encryption with Age/GPG
- **Kustomize**: Kubernetes configuration management
- **Helm**: Package manager (via Flux HelmRelease)

## Important Notes

- The main branch is `main` (Git repo: https://github.com/salisk/k8s-gitops)
- Flux watches the `main` branch and reconciles every 10m (flux-system), 30m (apps)
- Core resources have `prune: false` to prevent accidental deletion
- App resources have `prune: true` to remove resources not tracked in Git
- Variable substitution is applied to all Kustomizations via patches in `apps.yaml`
