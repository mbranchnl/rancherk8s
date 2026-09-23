
Table of Contents
------------------

- [Description](#description)
- [High Availability](#high-availability)
- [Install Method](#install-method)
- [Gateway API CRDs](#gateway-api-crds)
- [Role Variables](#role-variables)
  - [Core](#core)
  - [Pre-flight resource requirements](#pre-flight-resource-requirements)
  - [High Availability and Scheduling](#high-availability-and-scheduling)
  - [Firewall](#firewall)
  - [Storage](#storage)
  - [Networking / CNI](#networking--cni)
  - [Tooling](#tooling)
  - [Upgrades & Backups](#upgrades--backups)
  - [Flux GitOps](#flux-gitops)
- [Usage Examples](#usage-examples)
  - [Example Inventory](#example-inventory)
  - [Scenario Examples](#scenario-examples)
    - [1. Single node](#1-single-node)
    - [2. Three masters only (no dedicated workers)](#2-three-masters-only-no-dedicated-workers)
    - [3. Single master, multiple workers](#3-single-master-multiple-workers)
    - [4. Three masters (control-plane only), multiple workers](#4-three-masters-control-plane-only-multiple-workers)
    - [5. Three masters (schedulable), multiple workers](#5-three-masters-schedulable-multiple-workers)
  - [Example Playbook](#example-playbook)
  - [Rollout](#rollout)
  - [Post-Deployment](#post-deployment)
  - [Common Operations](#common-operations)

Description
-----------

This role deploys an RKE2 or K3s cluster (set via `rancherk8s_type`); the topology depends on
the inventory you choose. The role checks that you have enough servers for etcd quorum (an odd
number) before deploying. Secondary servers join one at a time, never in parallel, to prevent
etcd quorum issues.

Supported OS: RHEL-family 9+ (RHEL, Rocky, AlmaLinux, ...) and Ubuntu 22.04+ LTS.

High Availability
------------------

Multi-server (HA) clusters need `rancherk8s_api_endpoint`: a stable address for the control-plane
API. Either let this role manage it as a floating VIP (`rancherk8s_api_vip_enabled: true`,
kube-vip), or point it at an externally managed load balancer (leave that var false). Added to
tls-san automatically. Single-server clusters can leave it unset.

Install Method
--------------

Both k3s and RKE2 always install as a plain binary here - no OS package manager involved, on
either RHEL-family or Ubuntu. That means identical, fully Ansible-controlled install/upgrade
behavior on every supported distro, and no risk of `dnf`/`apt` silently upgrading the cluster
outside of this role's control. The one trade-off: on RHEL-family, RKE2's alternative RPM
install method pulls in the `rke2-selinux` policy package as a dependency, which the binary
install does not. If you run with SELinux enforcing, provision that policy yourself (e.g. via
your base image) before running this role - pre-flight warns if it detects this combination
unhandled.

Gateway API CRDs
----------------

This role does not install or manage the Kubernetes Gateway API CRDs and does not enable Cilium's
Gateway API controller. Install and manage compatible CRDs after the RKE2 or K3s cluster is available,
for example through the Envoy Gateway installation managed by the GitOps layer. If Cilium's Gateway API
controller is enabled separately, its required CRDs must exist before that controller is enabled.

Role Variables
--------------

### Core

| Variable                     | Required | Default                           | Description                                                   |
|-------------------------------|----------|------------------------------------|-----------------------------------------------------------------|
| rancherk8s_type               | yes      | -                                  | Distro to install: 'rke2' or 'k3s'. No default, must be set per inventory |
| rancherk8s_version             | yes      | v1.34.1+rke2r1                    | Version to install/upgrade to (bump this to trigger an upgrade) |
| rancherk8s_node_type           | no       | server                            | Node role: 'server' or 'agent'                                 |
| rancherk8s_service_account     | no       | admin                              | Name of the service account to create                          |
| rancherk8s_kubeconfig          | no       | /etc/rancher/\<type>/\<type>.yaml | Path to the local kubeconfig file                              |
| rancherk8s_fetch_kubeconfig    | no       | true                                | Fetch the primary server's kubeconfig to the control machine   |
| rancherk8s_cluster_name        | no       | ''                                  | Cluster/context/user name in the fetched kubeconfig and its filename (~/.kube/\<name>.yml). Empty falls back to the primary server's hostname |
| rancherk8s_artifact_path       | no       | /tmp/                              | Path for temporary install artifacts                            |

### Pre-flight resource requirements

| Variable                                     | Required | Default | Description                                                                 |
|-----------------------------------------------|----------|---------|-------------------------------------------------------------------------------|
| rancherk8s_min_cpu_cores                      | no       | 2       | Minimum CPU cores required per node                                          |
| rancherk8s_min_memory_mb                      | no       | 4096    | Minimum memory (MB) required per node                                        |
| rancherk8s_min_memory_margin_percent          | no       | 20      | Allowed shortfall below the memory minimum, as a percentage (guests under-report RAM) |
| rancherk8s_control_plane_only_min_cpu_cores   | no       | 2       | CPU floor used instead, on a `server` with `rancherk8s_server_schedulable: false` |
| rancherk8s_control_plane_only_min_memory_mb   | no       | 2048    | Memory floor used instead, on a `server` with `rancherk8s_server_schedulable: false` |

### High Availability and Scheduling

| Variable                       | Required | Default              | Description                                                    |
|----------------------------------|----------|-----------------------|--------------------------------------------------------------|
| rancherk8s_api_endpoint          | no*      | ''                    | VIP or external LB address for the control-plane API. *Required when more than one server is provisioned |
| rancherk8s_api_vip_enabled       | no       | false                 | Let this role manage rancherk8s_api_endpoint as a kube-vip VIP |
| rancherk8s_api_vip_version       | no       | v1.2.2                | kube-vip image tag, used when rancherk8s_api_vip_enabled is true |
| rancherk8s_api_vip_interface     | no       | primary server's NIC  | NIC kube-vip binds to for ARP on every control-plane node      |
| rancherk8s_server_schedulable    | no       | true                  | Whether `server` nodes run regular workloads. Set false to dedicate them to etcd/control-plane only - pair with `agent` nodes so the cluster has somewhere to schedule workloads |

### Firewall

| Variable                    | Required | Default | Description                                                       |
|-------------------------------|----------|---------|---------------------------------------------------------------------|
| rancherk8s_manage_firewalld   | no       | true    | Disable firewalld (it conflicts with CNI-managed iptables/eBPF)     |

### Storage

| Variable                              | Required | Default | Description                                                                 |
|----------------------------------------|----------|---------|-------------------------------------------------------------------------------|
| rancherk8s_default_local_storage_path  | no       | ''      | Path on the server node used by the built-in local-path-provisioner for the default 'local-path' StorageClass. Empty leaves it unset, falling back to the upstream k3s/rke2 default |
| rancherk8s_local_storage_provisioner_enabled | no | false   | Enable k3s's built-in 'local-storage' addon (local-path-provisioner); disabled by default, Flux provides it. No-op on rke2 |

### Networking / CNI

| Variable                       | Required | Default | Description                                                 |
|----------------------------------|----------|---------|----------------------------------------------------------------|
| rancherk8s_cni                   | no       | cilium  | CNI plugin to use (e.g., 'cilium', 'canal')                     |
| rancherk8s_cni_cilium_version     | no       | 1.18.7  | Cilium chart version                                            |
| rancherk8s_cni_cilium_autoupgrade | no       | false   | Auto-detect and use the latest Cilium version instead            |
| rancherk8s_cni_l2_enabled         | no       | true    | Enable Cilium L2 announcements                                   |
| rancherk8s_cni_operator_replicas  | no       | 1       | Cilium operator replica count                                    |
| rancherk8s_cni_gateway            | no       | true    | Reserve Cilium's Gateway API-adjacent settings (see Gateway API CRDs above - CRDs are not managed here) |

### Tooling

| Variable                | Required | Default  | Description                                                  |
|----------------------------|----------|----------|------------------------------------------------------------------|
| rancherk8s_install_tools   | no       | true     | Install additional tools (Flux, Helm, k9s, Cilium CLI)           |
| rancherk8s_helm_version    | no       | ''       | Helm version to install. Empty installs the latest release       |
| rancherk8s_k9s_version     | no       | v0.40.5  | k9s version to install. Empty installs the latest release        |

### Upgrades & Backups

| Variable                             | Required | Default         | Description                                                         |
|--------------------------------------|----------|-----------------|---------------------------------------------------------------------|
| rancherk8s_allow_upgrade             | no       | true            | Allow the role to run the upgrade path when a newer version is set  |
| rancherk8s_upgrade_drain_timeout     | no       | 300             | Seconds to wait for a node to drain before an upgrade fails         |
| rancherk8s_upgrade_agent_throttle    | no       | 1               | Agents upgraded concurrently (servers are always sequential)        |
| rancherk8s_backup_schedule           | no       | "0 8,20 * * *"  | Cron schedule for etcd snapshots                                    |
| rancherk8s_backup_retention          | no       | "14"            | Number of local snapshots to retain                                 |
| rancherk8s_backup_s3_enabled         | no       | false           | Also ship etcd snapshots to an S3-compatible bucket                 |
| rancherk8s_backup_s3_endpoint        | no       | ''              | S3 endpoint URL, required when rancherk8s_backup_s3_enabled is true |
| rancherk8s_backup_s3_bucket          | no       | ''              | S3 bucket name                                                      |
| rancherk8s_backup_s3_region          | no       | ''              | S3 region, optional depending on your provider                      |
| rancherk8s_backup_s3_folder          | no       | ''              | Optional folder/prefix within the bucket                            |
| rancherk8s_backup_s3_access_key      | no       | ''              | S3 access key (use vault-encrypted inventory)                       |
| rancherk8s_backup_s3_secret_key      | no       | ''              | S3 secret key (use vault-encrypted inventory)                       |

### Flux GitOps

| Variable                                 | Required | Default                     | Description                                                                                |
|------------------------------------------|----------|-----------------------------|--------------------------------------------------------------------------------------------|
| rancherk8s_flux_bootstrap                | no       | true                        | Enable Flux GitOps toolkit installation                                                    |
| rancherk8s_flux_interval                 | no       | 5m                          | Flux's sync interval                                                                       |
| rancherk8s_flux_bootstrap_provider       | no       | github                      | Git provider for Flux                                                                      |
| rancherk8s_flux_bootstrap_token          | no       | -                           | Authentication token for Git provider                                                      |
| rancherk8s_flux_bootstrap_owner          | no       | -                           | Repository owner for Flux                                                                  |
| rancherk8s_flux_bootstrap_repo           | no       | fleet-infra                 | Repository name for Flux                                                                   |
| rancherk8s_flux_bootstrap_branch         | no       | main                        | Git branch to use                                                                          |
| rancherk8s_flux_bootstrap_path           | no       | ./clusters/my-cluster       | Path within repository for cluster configuration                                           |
| rancherk8s_flux_bootstrap_type           | no       | personal                    | Repository type ('personal' or 'organization')                                             |
| rancherk8s_flux_sops_age_key_enabled     | no       | false                       | Push a SOPS age key into the cluster as a Secret for Flux's kustomize-controller           |
| rancherk8s_flux_sops_age_key_path        | no       | ~/.config/sops/age/keys.txt | Path to the age key file on the control machine, read at runtime (not stored in inventory) |
| rancherk8s_flux_sops_age_key_namespace   | no       | flux-system                 | Namespace to create the Secret in                                                          |
| rancherk8s_flux_sops_age_key_secret_name | no       | sops-age                    | Secret name, must match the Kustomization's decryption.secretRef                           |


Usage Examples
----------------

### Example Inventory
Node role is decided purely by each host's `rancherk8s_node_type` (defaults to `server`;
set `agent` explicitly on worker hosts). The role builds its own internal `server`/`agent`/
`server_primary`/`server_secondary` groups from that var during pre-flight - your
inventory's own group names are not read for this and can be anything you like (`masters`/
`workers`, `control-plane`/`workers`, ...). The scenario examples below happen to name
their inventory groups `server`/`agent` for readability, matching the role's own group
names, but that's a convention, not a requirement.

For larger inventories, split shared cluster vars into `group_vars/<group>.yml` instead of
inlining them under `all.vars`:

```yaml
# inventory/my-cluster.yml
my_cluster:
  hosts:
    rke2-server-01:
      ansible_host: 10.0.0.101
    rke2-server-02:
      ansible_host: 10.0.0.102
    rke2-server-03:
      ansible_host: 10.0.0.103
```

```yaml
# inventory/group_vars/my_cluster.yml
rancherk8s_type: 'k3s'
rancherk8s_cluster_name: 'my-cluster'   # ~/.kube/<name>.yml instead of the primary node's hostname
rancherk8s_api_endpoint: '10.0.0.50'
rancherk8s_api_vip_enabled: true
```

### Scenario Examples

Five common topologies, single-file inventory style (`all.vars` + `children`). Swap
`rancherk8s_type` for `rke2` where needed; everything else applies to both. Any inventory
with more than one `server` host needs `rancherk8s_api_endpoint` (see High Availability
above) - the role's pre-flight fails fast otherwise, and also rejects an even server
count (etcd quorum).

#### 1. Single node
One host, no HA endpoint needed - `server` is implied and the node runs everything
(control plane, etcd, workloads).

```yaml
all:
  vars:
    rancherk8s_type: 'k3s'
  children:
    server:
      hosts:
        node-01:
          ansible_host: 10.0.0.11
          ansible_user: root
```

#### 2. Three masters only (no dedicated workers)
HA control plane with no separate worker pool. `rancherk8s_server_schedulable` defaults
to `true`, so these three servers also run regular workloads - there's nowhere else for
pods to go without agents.

```yaml
all:
  vars:
    rancherk8s_type: 'k3s'
    rancherk8s_api_endpoint: '10.0.0.50'
    rancherk8s_api_vip_enabled: true
  children:
    server:
      hosts:
        node-01:
          ansible_host: 10.0.0.11
          ansible_user: root
        node-02:
          ansible_host: 10.0.0.12
          ansible_user: root
        node-03:
          ansible_host: 10.0.0.13
          ansible_user: root
```

#### 3. Single master, multiple workers
One server, no HA endpoint needed since there's only one server to join against.

```yaml
all:
  vars:
    rancherk8s_type: 'k3s'
  children:
    server:
      hosts:
        node-01:
          ansible_host: 10.0.0.11
          ansible_user: root
    agent:
      hosts:
        node-02:
          ansible_host: 10.0.0.21
          ansible_user: root
          rancherk8s_node_type: agent
        node-03:
          ansible_host: 10.0.0.22
          ansible_user: root
          rancherk8s_node_type: agent
        node-04:
          ansible_host: 10.0.0.23
          ansible_user: root
          rancherk8s_node_type: agent
```

#### 4. Three masters (control-plane only), multiple workers
`rancherk8s_server_schedulable: false` taints/cordons the servers (k3s: `node-taint`;
RKE2: `disable-scheduling`) so only etcd and the control plane run there - all regular
workloads land on the `agent` nodes. Pre-flight also refuses to provision a cluster
where every server is unschedulable and no agents are defined.

```yaml
all:
  vars:
    rancherk8s_type: 'k3s'
    rancherk8s_api_endpoint: '10.0.0.50'
    rancherk8s_api_vip_enabled: true
    rancherk8s_server_schedulable: false
  children:
    server:
      hosts:
        node-01:
          ansible_host: 10.0.0.11
          ansible_user: root
        node-02:
          ansible_host: 10.0.0.12
          ansible_user: root
        node-03:
          ansible_host: 10.0.0.13
          ansible_user: root
    agent:
      hosts:
        node-04:
          ansible_host: 10.0.0.21
          ansible_user: root
          rancherk8s_node_type: agent
        node-05:
          ansible_host: 10.0.0.22
          ansible_user: root
          rancherk8s_node_type: agent
        node-06:
          ansible_host: 10.0.0.23
          ansible_user: root
          rancherk8s_node_type: agent
```

#### 5. Three masters (schedulable), multiple workers
HA control plane where the three servers also run workloads alongside dedicated
agents - `rancherk8s_server_schedulable` left at its default (`true`).

```yaml
all:
  vars:
    rancherk8s_type: 'k3s'
    rancherk8s_api_endpoint: '10.0.0.50'
    rancherk8s_api_vip_enabled: true
  children:
    server:
      hosts:
        node-01:
          ansible_host: 10.0.0.11
          ansible_user: root
        node-02:
          ansible_host: 10.0.0.12
          ansible_user: root
        node-03:
          ansible_host: 10.0.0.13
          ansible_user: root
    agent:
      hosts:
        node-04:
          ansible_host: 10.0.0.21
          ansible_user: root
          rancherk8s_node_type: agent
        node-05:
          ansible_host: 10.0.0.22
          ansible_user: root
          rancherk8s_node_type: agent
        node-06:
          ansible_host: 10.0.0.23
          ansible_user: root
          rancherk8s_node_type: agent
```

### Example Playbook
```yaml
---
- name: Deploy RKE2 Cluster
  hosts: all
  roles:
    - rancherk8s
```

### Rollout
```shell
ansible-playbook -i inventory playbook.yml -l my_cluster --vault-password-file=~/.vault_pass
```

Kubeconfig is fetched to `~/.kube/<rancherk8s_cluster_name or primary node's hostname>.yml`.

### Post-Deployment
Retrieve the admin token for kubectl access:
```shell
kubectl get secret sa-admin-token -o jsonpath='{.data.*}' -n kube-system | base64 --decode
```

### Common Operations

1. Adding a new worker node:
   - Add node to inventory under [agent]
   - Run playbook

2. Upgrading the cluster:
   - Update rancherk8s_version in your variables
   - Run playbook - the role detects the version change and drains/upgrades nodes on its own

3. Adding custom labels:
   - Add labels in inventory as shown above
   - Run playbook to apply changes
