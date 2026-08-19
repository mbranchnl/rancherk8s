
Description
-----------

This role deploys an RKE2 or K3s cluster (set via `rancherk8s_type`); the topology depends on
the inventory you choose. The role checks that you have enough servers for etcd quorum (an odd
number) before deploying. Secondary servers join one at a time, never in parallel, to prevent
etcd quorum issues.

Supported OS: RHEL-family 9+ (RHEL, Rocky, AlmaLinux, ...) and Ubuntu 22.04+ LTS.

KNOWN ISSUES
------------

- `rancherk8s_auto_upgrade` and `rancherk8s_auto_upgrade_grace_period` are not currently wired up
  to any task - setting them has no effect. To upgrade, bump `rancherk8s_version` and re-run the
  playbook; the role detects the version change on its own (see Common Operations below).

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
| rancherk8s_artifact_path       | no       | /tmp/                              | Path for temporary install artifacts                            |

### High Availability

| Variable                       | Required | Default              | Description                                                    |
|----------------------------------|----------|-----------------------|--------------------------------------------------------------|
| rancherk8s_api_endpoint          | no*      | ''                    | VIP or external LB address for the control-plane API. *Required when more than one server is provisioned |
| rancherk8s_api_vip_enabled       | no       | false                 | Let this role manage rancherk8s_api_endpoint as a kube-vip VIP |
| rancherk8s_api_vip_version       | no       | v1.2.2                | kube-vip image tag, used when rancherk8s_api_vip_enabled is true |
| rancherk8s_api_vip_interface     | no       | primary server's NIC  | NIC kube-vip binds to for ARP on every control-plane node      |

### Firewall

| Variable                    | Required | Default | Description                                                       |
|-------------------------------|----------|---------|---------------------------------------------------------------------|
| rancherk8s_manage_firewalld   | no       | true    | Disable firewalld (it conflicts with CNI-managed iptables/eBPF)     |

### Storage

| Variable                              | Required | Default | Description                                                                 |
|----------------------------------------|----------|---------|-------------------------------------------------------------------------------|
| rancherk8s_default_local_storage_path  | no       | ''      | Path on the server node used by the built-in local-path-provisioner for the default 'local-path' StorageClass. Empty leaves it unset, falling back to the upstream k3s/rke2 default |

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
Create an inventory file with server and agent nodes:

```yaml
all:
  vars:
    rancherk8s_type: 'k3s' # or 'rke2', no default - required
    # Required once you have more than one server (see High Availability above).
    rancherk8s_api_endpoint: '10.0.0.50'
    rancherk8s_api_vip_enabled: true
  children:
    server:
      hosts:
        rke2-server-01:
          ansible_user: root
        rke2-server-02:
          ansible_user: root
        rke2-server-03:
          ansible_user: root
    agent:
      hosts:
        rke2-agent-01:
          ansible_user: root
          rancherk8s_node_type: agent # defaults to 'server' otherwise
          labels:
            - role=worker
            - environment=prod
```

### Example Playbook
```yaml
---
- name: Deploy RKE2 Cluster
  hosts: all
  roles:
    - rancherk8s
```

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
