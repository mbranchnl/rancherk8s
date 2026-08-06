
Description
-----------

This role can be used to deploy a rke2 cluster, the setup depends on the inventory you choose.
The role checks if you have enough servers for a quorum and deploy's the cluster. 
Deploying after the first node will be done sequencly to prevent issues with nodes joining at the exact same time.

KNOW ISSUES
-----------

- When bootstrapping a cluster with autoupgrade on true it will fail due to a unknown rancherk8s_version,
this is because the cluster is just bootstrapped. This will be fixed in future releases.

Gateway API CRDs
----------------

This role does not install or manage the Kubernetes Gateway API CRDs and does not enable Cilium's
Gateway API controller. Install and manage compatible CRDs after the RKE2 or K3s cluster is available,
for example through the Envoy Gateway installation managed by the GitOps layer. If Cilium's Gateway API
controller is enabled separately, its required CRDs must exist before that controller is enabled.

Role Variables
---------

| Variable                      | Required | Default        | Description                                                                  |
|------------------------------|----------|----------------|------------------------------------------------------------------------------|
| rancherk8s_version                 | yes      | v1.31.3+rke2r1| RKE2 version to install/upgrade to                                           |
| rancherk8s_node_type              | no       | server        | Node type: 'server' or 'agent'                                               |
| rancherk8s_service_account        | no       | admin         | Name of the service account to create                                        |
| rancherk8s_kubeconfig             | no       | /opt/rke2/rke2.yaml | Path to kubeconfig file                                               |
| rancherk8s_artifact_path          | no       | /tmp/         | Path for temporary artifacts                                                 |
| rancherk8s_cni                    | no       | cilium        | CNI plugin to use (e.g., 'cilium', 'canal')                                 |
| rancherk8s_allow_upgrade          | no       | true          | Allow cluster upgrades                                                       |
| rancherk8s_auto_upgrade           | no       | false         | Enable automatic upgrades when a newer version is specified                  |
| rancherk8s_auto_upgrade_grace_period | no    | 45           | Wait period (seconds) for API health check after upgrade                     |
| rancherk8s_backup_schedule        | no       | "0 8,20 * * *"| Cron schedule for etcd backups                                              |
| rancherk8s_backup_retention       | no       | "14"          | Number of days to retain backups                                            |
| rancherk8s_install_tools          | no       | true          | Install additional tools (like Flux)                                         |
| rancherk8s_flux_bootstrap         | no       | true          | Enable Flux GitOps toolkit installation                                      |
| rancherk8s_flux_bootstrap_provider| no       | github        | Git provider for Flux                                                        |
| rancherk8s_flux_bootstrap_token   | no       | -             | Authentication token for Git provider                                        |
| rancherk8s_flux_bootstrap_owner   | no       | -             | Repository owner for Flux                                                    |
| rancherk8s_flux_bootstrap_repo    | no       | fleet-infra   | Repository name for Flux                                                     |
| rancherk8s_flux_bootstrap_branch  | no       | main          | Git branch to use                                                            |
| rancherk8s_flux_bootstrap_path    | no       | ./clusters/my-cluster | Path within repository for cluster configuration                     |
| rancherk8s_flux_bootstrap_type    | no       | personal      | Repository type ('personal' or 'organization')                               |


Usage Examples
----------------

### Example Inventory
Create an inventory file with server and agent nodes:

```yaml
all:
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
    - rke2-ansible

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
   - Set rancherk8s_auto_upgrade: true
   - Run playbook

3. Adding custom labels:
   - Add labels in inventory as shown above
   - Run playbook to apply changes
