# Kubernetes Cluster Playbooks

This directory contains Ansible playbooks for setting up and managing a Kubernetes cluster with **Kubernetes 1.31** and **Cilium 1.18.4 CNI**.

## Version Compatibility

- **Kubernetes**: 1.31 (latest stable)
- **Containerd**: 2.0.0+ or 1.7.20+ (installed via containerd.io package from Docker CE repository)
- **Cilium CNI**: 1.18.4 (supports Kubernetes up to 1.32)
- **Kubectl**: 1.31.0

## Prerequisites

Before running the playbooks:

1. **Rocky Linux 9** or compatible RHEL-based system
2. **SSH access** to all nodes with sudo privileges
3. **Ansible** installed on the control node
4. **Passwordless sudo** configured for the remote user (devops)
5. **Network connectivity** between all cluster nodes

## Playbook Execution Order

Run the playbooks in this strict sequence:

### 1. Prerequisites (`prerequisites.yml`)
Prepares all nodes for Kubernetes installation:
- Installs required packages (curl, wget, vim, git, python3-pip, **conntrack-tools**)
- Disables swap (required by Kubernetes)
- Configures SELinux to permissive mode
- Loads kernel modules (overlay, br_netfilter)
- Sets sysctl parameters for Kubernetes networking
- Adds Docker CE repository
- Installs **containerd.io** with CRI v1 support
- Configures containerd with systemd cgroup driver
- Sets vm.max_map_count=262144 on worker nodes (for Elasticsearch)

```bash
cd k8s
ansible-playbook playbooks/prerequisites.yml
```

**Important Notes:**
- The playbook installs `containerd.io` (not `containerd`) to get CRI v1 support required by Kubernetes 1.31
- Containerd config is regenerated to ensure CRI is properly enabled
- CRI version verification is performed automatically

### 2. Install Kubernetes (`install-kubernetes.yml`)
Installs Kubernetes components on all nodes:
- Adds official Kubernetes yum repository (v1.31)
- Installs kubelet, kubeadm, kubectl on control plane nodes
- Installs kubelet, kubeadm on worker nodes
- Enables kubelet service (starts after kubeadm init)
- Locks Kubernetes package versions

```bash
ansible-playbook playbooks/install-kubernetes.yml
```

### 3. Initialize Control Plane (`init-control-plane.yml`)
Initializes the first control plane node:
- Runs `kubeadm init` with cluster configuration
- Configures kubectl for root and devops users
- Generates join commands for workers and additional control planes
- Saves join commands to `/tmp/` on the Ansible control node

```bash
ansible-playbook playbooks/init-control-plane.yml
```

**Network Configuration:**
- Pod Network CIDR: `10.244.0.0/16`
- Service CIDR: `10.96.0.0/12` (Kubernetes default)
- API Server: Auto-detected (uses default IPv4 address)

### 4. Join Worker Nodes (`join-workers.yml`)
Joins worker nodes to the cluster:
- Reads join command from `/tmp/k8s-join-command.sh`
- Executes join command on all worker nodes
- Verifies nodes joined successfully

```bash
ansible-playbook playbooks/join-workers.yml
```

### 5. Join Additional Control Plane Nodes (`join-control-plane.yml`) - Optional
For high-availability clusters with multiple control plane nodes:
- Reads control plane join command from `/tmp/k8s-cp-join-command.sh`
- Joins additional control plane nodes (skips first node)
- Configures kubectl access

```bash
ansible-playbook playbooks/join-control-plane.yml
```

### 6. Install Cilium CNI (`install-cni-cilium.yml`)
Installs Cilium CNI plugin with advanced features:
- Downloads and installs Cilium CLI
- Installs Cilium 1.18.4 with configuration:
  - **Routing Mode**: vxlan (tunnel encapsulation)
  - **Kube-proxy Replacement**: Enabled (eBPF-based)
  - **Hubble Observability**: Enabled with UI
  - **Bandwidth Manager**: Enabled
  - **HA**: 2 operator replicas
- Waits for Cilium to be ready
- Verifies cluster nodes are Ready

```bash
ansible-playbook playbooks/install-cni-cilium.yml
```

**Cilium Features Enabled:**
- IPv4 networking (IPv6 disabled)
- VXLAN tunnel mode for pod-to-pod communication
- Strict kube-proxy replacement (eBPF-based load balancing)
- Hubble for network observability and monitoring
- Hubble UI for visual network inspection
- Bandwidth management for QoS

**Access Hubble UI:**
```bash
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
# Open browser: http://localhost:12000
```

### 7. Setup CLI Server (`setup-cli-server.yml`) - Optional
Configures a dedicated management server outside the cluster:
- Installs kubectl 1.31.0
- Installs Helm 3.13.3
- Installs Cilium CLI
- Configures bash completion and kubectl alias (k=kubectl)
- Attempts to auto-copy kubeconfig from control plane

```bash
ansible-playbook playbooks/setup-cli-server.yml
```

## Important Configuration Notes

### Containerd Runtime
- Uses `containerd.io` package from Docker CE repository (provides v2.1.5+)
- **Not** the basic `containerd` package which lacks CRI v1 support
- Configured with systemd cgroup driver (required for Kubernetes 1.31)
- Config file regenerated on every prerequisites run to ensure CRI is enabled

### Cilium 1.18.4 Breaking Changes
Cilium 1.15+ introduced breaking changes from earlier versions:

1. **`tunnel` parameter removed** → Use `routingMode` instead
   - Old: `--set tunnel=vxlan`
   - New: `--set routingMode=vxlan`

2. **`kubeProxyReplacement` now requires boolean** → Changed from string values
   - Old: `--set kubeProxyReplacement=strict`
   - New: `--set kubeProxyReplacement=true`

3. **Boolean parameters** → Must be lowercase strings (true/false, not True/False)
   - Playbook automatically converts using `| lower` Jinja2 filter

### Namespace for Cilium
Cilium installs to the **kube-system** namespace (not cilium-system):
- Pods: `kubectl get pods -n kube-system -l k8s-app=cilium`
- DaemonSet: `kubectl get daemonset cilium -n kube-system`
- Services: `kubectl get svc -n kube-system hubble-ui`

## Troubleshooting

### Issue: CRI Error - "unknown service runtime.v1.RuntimeService"

**Cause**: Containerd installed without CRI support or config not properly generated.

**Solution**: Run prerequisites playbook again - it will regenerate containerd config:
```bash
ansible-playbook playbooks/prerequisites.yml
```

Or manually fix:
```bash
sudo systemctl stop containerd
sudo rm -f /etc/containerd/config.toml
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd

# Verify CRI is working:
sudo crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock version
```

### Issue: Missing conntrack during kubeadm init

**Cause**: conntrack-tools package not installed.

**Solution**: Install manually:
```bash
sudo dnf install -y conntrack-tools
```

Or re-run prerequisites playbook (already includes conntrack-tools).

### Issue: Cilium install fails with "tunnel was deprecated"

**Cause**: Using old Cilium parameter names with Cilium 1.15+.

**Solution**: Already fixed in playbook - uses `routingMode` instead of `tunnel`.

### Issue: Cilium install fails with "kubeProxyReplacement must be true or false"

**Cause**: Cilium 1.15+ requires boolean values, not strings like "strict".

**Solution**: Already fixed in playbook - converts to boolean true/false.

## Verification Commands

After installation, verify the cluster:

```bash
# Check nodes are Ready
kubectl get nodes

# Check all pods are Running
kubectl get pods -A

# Check Cilium status
cilium status

# Check Cilium connectivity
cilium connectivity test

# View Cilium pods
kubectl get pods -n kube-system -l k8s-app=cilium

# Check containerd version
sudo crictl version
```

## Configuration Variables

Key variables in `group_vars/all.yml`:

```yaml
kubernetes_version: "1.31"
cilium_version: "1.18.4"
pod_network_cidr: "10.244.0.0/16"
service_cidr: "10.96.0.0/12"
cilium_tunnel_mode: "vxlan"
cilium_kube_proxy_replacement: "strict"  # Converted to boolean true
cilium_enable_hubble: true
cilium_hubble_ui: true
```

## Additional Resources

- [Kubernetes 1.31 Release Notes](https://kubernetes.io/blog/2024/08/13/kubernetes-v1-31-release/)
- [Cilium 1.18 Documentation](https://docs.cilium.io/en/v1.18/)
- [Containerd Installation Guide](https://docs.docker.com/engine/install/rhel/)
- [Kubeadm Cluster Setup](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
