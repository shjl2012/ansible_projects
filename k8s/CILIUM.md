# Cilium CNI Plugin Guide

This guide provides information on using and troubleshooting Cilium in your Kubernetes cluster.

## What is Cilium?

Cilium is an eBPF-based networking, observability, and security solution for cloud-native environments. It provides:

- **High-performance networking** using eBPF (kernel-level)
- **Advanced network policies** (L3/L4 and L7/HTTP)
- **Service mesh capabilities** (without sidecars)
- **Hubble observability** for network traffic visualization
- **Kube-proxy replacement** for better performance

## Installation

```bash
cd k8s
ansible-playbook playbooks/install-cni-cilium.yml -K
```

## Cilium CLI Commands

### Check Cilium Status

```bash
# Overall Cilium status
cilium status

# Detailed status
cilium status --verbose

# Wait for Cilium to be ready
cilium status --wait
```

### Verify Installation

```bash
# Check Cilium pods
kubectl get pods -n kube-system -l k8s-app=cilium

# Check Cilium operator
kubectl get pods -n kube-system -l name=cilium-operator

# Check all Cilium components
kubectl get all -n kube-system -l k8s.io/app=cilium
```

### Connectivity Testing

```bash
# Run connectivity test (takes 5-10 minutes)
cilium connectivity test

# Run specific connectivity tests
cilium connectivity test --test pod-to-pod
cilium connectivity test --test pod-to-service
```

### Hubble Observability

```bash
# Check Hubble status
cilium hubble port-forward &

# Observe network flows
hubble observe

# Observe flows from a specific pod
hubble observe --pod <pod-name>

# Observe flows to a specific service
hubble observe --service <service-name>

# Observe HTTP requests (L7)
hubble observe --protocol http
```

### Hubble UI

```bash
# Port-forward Hubble UI
kubectl port-forward -n kube-system svc/hubble-ui 12000:80

# Access at: http://localhost:12000
```

## Network Policy Examples

### Basic Pod-to-Pod Policy (L3/L4)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Cilium L7 HTTP Policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-specific-api-endpoints
spec:
  endpointSelector:
    matchLabels:
      app: api-server
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/v1/.*"
        - method: "POST"
          path: "/api/v1/orders"
```

### DNS-based Policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-external-api
spec:
  endpointSelector:
    matchLabels:
      app: my-app
  egress:
  - toFQDNs:
    - matchName: "api.example.com"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
```

## Performance Tuning

### Kube-proxy Replacement

Cilium can replace kube-proxy for better performance:

```bash
# Check if kube-proxy is disabled
kubectl get ds -n kube-system kube-proxy

# If kube-proxy exists, you can remove it (Cilium handles it)
kubectl delete ds -n kube-system kube-proxy
```

This is already configured in the playbook with `kubeProxyReplacement: strict`.

### Native Routing Mode

For best performance on cloud providers or bare metal with BGP:

Edit `group_vars/all.yml`:
```yaml
cilium_tunnel_mode: "disabled"  # Use native routing instead of VXLAN
```

This requires proper network routing configuration.

## Troubleshooting

### 1. Cilium Pods Not Running

```bash
# Check Cilium pod logs
kubectl logs -n kube-system -l k8s-app=cilium --tail=100

# Check Cilium operator logs
kubectl logs -n kube-system -l name=cilium-operator --tail=100

# Describe Cilium pods
kubectl describe pods -n kube-system -l k8s-app=cilium
```

### 2. Nodes Not Ready

```bash
# Check node status
kubectl get nodes

# Check Cilium status on all nodes
cilium status

# Check for CNI issues
kubectl get events -n kube-system --sort-by='.lastTimestamp'
```

### 3. Network Connectivity Issues

```bash
# Run connectivity test
cilium connectivity test

# Check endpoint health
cilium endpoint list

# Check BGP sessions (if using BGP)
cilium bgp peers
```

### 4. Hubble Not Working

```bash
# Check Hubble relay
kubectl get pods -n kube-system -l k8s-app=hubble-relay

# Check Hubble UI
kubectl get pods -n kube-system -l k8s-app=hubble-ui

# Restart Hubble relay
kubectl rollout restart deployment/hubble-relay -n kube-system
```

### 5. High Memory Usage

Cilium can use significant memory on large clusters:

```bash
# Check resource usage
kubectl top pods -n kube-system -l k8s-app=cilium

# Reduce Hubble buffer if needed (edit ConfigMap)
kubectl edit configmap cilium-config -n kube-system
# Set: hubble-flow-buffer-size: "512" (default is 4095)
```

## Monitoring and Metrics

### Prometheus Metrics

Cilium exposes Prometheus metrics:

```bash
# Port-forward to Cilium agent
kubectl port-forward -n kube-system ds/cilium 9090:9090

# Access metrics at: http://localhost:9090/metrics
```

### Grafana Dashboards

Cilium provides official Grafana dashboards:
- Cilium Agent Metrics
- Hubble Metrics
- Cilium Operator Metrics

Import from: https://grafana.com/grafana/dashboards/?search=cilium

## Upgrading Cilium

```bash
# Using Cilium CLI
cilium upgrade --version 1.16.0

# Or using Ansible (update cilium_version in group_vars/all.yml)
ansible-playbook playbooks/install-cni-cilium.yml -K
```

## Uninstalling Cilium

```bash
# Using Cilium CLI
cilium uninstall

# Manually remove resources
kubectl delete namespace cilium
kubectl delete ds -n kube-system cilium
kubectl delete deployment -n kube-system cilium-operator
```

## Advanced Features

### Service Mesh (without sidecars)

Cilium can provide service mesh features using eBPF:

```yaml
# Enable in group_vars/all.yml
cilium_enable_envoy: true
cilium_enable_l7_proxy: true
```

### Transparent Encryption

Enable encryption for all traffic:

```yaml
# Enable in group_vars/all.yml
cilium_enable_encryption: true
cilium_encryption_type: "wireguard"  # or "ipsec"
```

### Multi-cluster Networking

Connect multiple Kubernetes clusters:

```bash
cilium clustermesh enable
cilium clustermesh connect --destination-context cluster2
```

## Resources

- **Official Documentation**: https://docs.cilium.io/
- **Cilium Slack**: https://cilium.io/slack
- **GitHub**: https://github.com/cilium/cilium
- **Hubble Documentation**: https://docs.cilium.io/en/stable/gettingstarted/hubble/
- **Network Policy Editor**: https://editor.cilium.io/

## Configuration Reference

Current configuration in `group_vars/all.yml`:

- **Version**: {{ cilium_version }}
- **Tunnel Mode**: {{ cilium_tunnel_mode }}
- **Kube-proxy Replacement**: {{ cilium_kube_proxy_replacement }}
- **Hubble Enabled**: {{ cilium_enable_hubble }}
- **Hubble UI**: {{ cilium_hubble_ui }}
- **IPv4**: {{ cilium_enable_ipv4 }}
- **IPv6**: {{ cilium_enable_ipv6 }}
