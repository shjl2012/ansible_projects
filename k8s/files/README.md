# Kubernetes Static Files

This directory contains static files to be copied to Kubernetes cluster nodes.

## File Types

- CNI plugin manifests (if not using templates)
- Kubernetes manifests (StorageClass, PersistentVolumes, etc.)
- Scripts for cluster operations
- Certificate files (if using custom certificates)
- Configuration files that don't require templating

## Usage in Playbooks

```yaml
- name: Copy static manifest
  copy:
    src: ../files/my-manifest.yml
    dest: /etc/kubernetes/manifests/my-manifest.yml
    owner: root
    group: root
    mode: '0644'
```

## Notes

- Do not commit sensitive files (certificates, keys) to version control
- Add sensitive files to `.gitignore`
- Use ansible-vault for encrypted files if needed
