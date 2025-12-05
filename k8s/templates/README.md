# Kubernetes Configuration Templates

This directory contains Jinja2 templates for Kubernetes configuration files.

## Template Files

Templates should reference variables from `group_vars/` and use relative paths `../templates/` when referenced from playbooks.

## Common Templates

- `containerd-config.toml.j2` - containerd configuration
- `kubelet-config.yml.j2` - kubelet configuration
- `kubeadm-config.yml.j2` - kubeadm configuration for cluster init
- `calico.yml.j2` - Calico CNI configuration
- `storage-class.yml.j2` - StorageClass definitions
- `nfs-provisioner.yml.j2` - NFS provisioner configuration

## Usage in Playbooks

```yaml
- name: Configure containerd
  template:
    src: ../templates/containerd-config.toml.j2
    dest: /etc/containerd/config.toml
    owner: root
    group: root
    mode: '0644'
```
