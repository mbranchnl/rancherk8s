
Description
-----------

This role can be used to deploy a rke2 cluster, the setup depends on the inventory you choose.
The role checks if you have enough servers for a quorum and deploy's the cluster. 
Deploying after the first node will be done sequencly to prevent issues with nodes joining at the exact same time.

KNOW ISSUES
-----------

- When bootstrapping a cluster with autoupgrade on true it will fail due to a unknown rke2_version,
this is because the cluster is just bootstrapped. This will be fixed in future releases.

Role Variables
---------

| Variable                      | Required | Default        | Description                                                                  |
|------------------------------|----------|----------------|------------------------------------------------------------------------------|
| rke2_version                 | yes      | v1.19.5+rke22 | RKE2 version to install/upgrade to                                           |
| server_args                  | no       | ""            | Additional server node parameters (e.g., "--tls-san=10.0.0.1")               |
| agent_args                   | no       | ""            | Additional agent node parameters                                              |
| rke2_tls_san_enabled        | no       | false         | Enable additional Subject Alternative Names (SANs)                           |
| tls_additional_san          | no       | ""            | Comma-separated list of additional SANs (IPs or hostnames)                   |
| rke2_auto_upgrade           | yes      | false         | Enable automatic upgrades when a newer version is specified                  |
| rke2_auto_upgrade_grace_period | yes    | 60           | Wait period (seconds) for API health check after upgrade                     |


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
   - Update rke2_version in your variables
   - Set rke2_auto_upgrade: true
   - Run playbook

3. Adding custom labels:
   - Add labels in inventory as shown above
   - Run playbook to apply changes
