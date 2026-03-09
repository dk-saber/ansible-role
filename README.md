# Ansible Role: `kubernetes_setup`

An Ansible role that automates the deployment of a Kubernetes cluster using **kubeadm**, consisting of **1 control-plane node** and **2 worker nodes** by default. The role handles every step from system prerequisites to worker node registration, including container runtime setup (containerd) and Calico CNI installation.

---

## Requirements

- **OS**: Ubuntu (tested with recent LTS releases)
- **Ansible**: 2.10+
- **Privilege escalation**: `become: yes` is required (the playbook uses `root`)
- **SSH access**: passwordless SSH configured for the `devops` user (or the user defined in `ansible_user`)
- **Network connectivity**: all nodes must be able to reach the internet to download packages and container images

---

## Role Structure

```
kubernetes_setup/
├── defaults/
│   └── main.yml          # Default variables
├── handlers/
│   └── main.yml          # Handlers (restart containerd / kubelet)
├── tasks/
│   ├── main.yml          # Entry point – dispatches common, master, and node tasks
│   ├── common.yml        # Tasks applied to all nodes
│   ├── master.yml        # Control-plane initialisation and Calico deployment
│   └── nodes.yml         # Worker node join
└── templates/
    ├── config.toml.j2          # containerd configuration (SystemdCgroup)
    ├── kubeadm-config.yaml.j2  # kubeadm InitConfiguration + ClusterConfiguration
    └── calico.yaml.j2          # Calico CNI manifest (v3.25.0)
```

---

## Inventory

The role expects the following inventory groups:

| Group    | Description            | Default hosts                         |
|----------|------------------------|---------------------------------------|
| `master` | Control-plane node(s)  | `master-01` → `192.168.1.10`          |
| `node`   | Worker nodes           | `node-01` → `192.168.1.11`, `node-02` → `192.168.1.12` |

Example `hosts.ini`:

```ini
[master]
master-01 ansible_host=192.168.1.10

[node]
node-01   ansible_host=192.168.1.11
node-02   ansible_host=192.168.1.12

[all:vars]
ansible_user=devops
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

---

## Variables

All variables are defined in `defaults/main.yml` and can be overridden at playbook, inventory, or host level.

| Variable           | Default            | Description                                              |
|--------------------|--------------------|----------------------------------------------------------|
| `k8s_version`      | `"1.32"`           | Kubernetes minor version used for the apt repository     |
| `pod_network_cidr` | `"192.168.0.0/16"` | Pod network CIDR – must match Calico's default IPAM pool |
| `ansible_user`     | `"devops"`         | OS user for kubectl config ownership                     |

> **Note**: `pod_network_cidr` is passed to both `kubeadm` (ClusterConfiguration) and the Calico manifest. Do not change it unless you also update the Calico IP pool configuration accordingly.

---

## What the Role Does

### On all nodes (`common.yml`)

1. Disables swap permanently (required by Kubernetes).
2. Loads kernel modules `overlay` and `br_netfilter`.
3. Configures sysctl parameters for bridged networking and IP forwarding.
4. Adds each node's IP/hostname pair to `/etc/hosts`.
5. Installs **containerd** from the Docker apt repository and deploys the `config.toml` template (enabling `SystemdCgroup = true`).
6. Adds the Kubernetes apt repository for the specified `k8s_version`.
7. Installs `kubelet` and `kubeadm` on all nodes; installs `kubectl` **only on the master**.
8. Pins (`hold`) the versions of `kubelet`, `kubeadm`, and `kubectl` to prevent unintended upgrades.

### On the control-plane (`master.yml`)

1. Checks whether the cluster is already initialised (via `/etc/kubernetes/admin.conf`) to make the task idempotent.
2. Generates the `kubeadm-config.yaml` from the Jinja2 template and runs `kubeadm init`.
3. Creates the `.kube/config` directory and copies `admin.conf` for the `devops` user.
4. Deploys the **Calico** CNI by applying the generated manifest.
5. Generates a join token and stores the full join command in `hostvars` for workers to consume.

### On worker nodes (`nodes.yml`)

1. Checks whether the node has already joined (via `/etc/kubernetes/kubelet.conf`).
2. Executes the join command retrieved from `hostvars['master-01']`.

---

## Usage

### 1. Clone / copy the role

Place the `kubernetes_setup` directory under your Ansible `roles/` folder, or reference it directly in your project.

### 2. Configure the inventory

Edit `hosts.ini` to match your actual IP addresses and SSH user.

### 3. Run the playbook

```bash
ansible-playbook -i hosts.ini site.yml
```

The `site.yml` playbook applies the role to **all** hosts:

```yaml
- name: Kubernetes cluster deployment
  hosts: all
  become: yes
  roles:
    - kubernetes_setup
```

---

## Post-Deployment Verification

SSH to the master node and run:

```bash
kubectl get nodes
```

Expected output (nodes may take a minute to become `Ready` while Calico initialises):

```
NAME        STATUS   ROLES           AGE   VERSION
master-01   Ready    control-plane   5m    v1.32.x
node-01     Ready    <none>          3m    v1.32.x
node-02     Ready    <none>          3m    v1.32.x
```

---

## Customisation

### Change Kubernetes version

```yaml
# group_vars/all.yml
k8s_version: "1.31"
```

### Use a different pod CIDR

```yaml
pod_network_cidr: "10.244.0.0/16"
```

> If you change the CIDR, you must also update the `CALICO_IPV4POOL_CIDR` environment variable inside `calico.yaml.j2` (currently commented out) and ensure it does not overlap with your host network.

### Add more worker nodes

Simply add entries under the `[node]` group in `hosts.ini` — the role will join them automatically.

---

## Dependencies

None. All required packages are installed directly by the role.

---

## Tested With

| Component   | Version  |
|-------------|----------|
| Kubernetes  | 1.32     |
| containerd  | latest stable (Docker repo) |
| Calico CNI  | v3.25.0  |
| kubeadm API | v1beta4  |
| Ubuntu      | 22.04 / 24.04 |
