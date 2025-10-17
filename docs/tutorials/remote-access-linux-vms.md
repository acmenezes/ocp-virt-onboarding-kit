# Remote Access to Linux VMs

## Overview

This tutorial demonstrates various methods to access Linux virtual machines running on OpenShift Virtualization. Each method suits different use cases, from quick troubleshooting to production-grade access patterns.

## Access Methods Comparison

| Method | Use Case | Network Required | SSH Keys Required | Production Ready |
|--------|----------|------------------|-------------------|------------------|
| Console | Initial setup, troubleshooting | No | No | No |
| VNC | GUI access | No | No | No |
| virtctl SSH | Quick admin access | Yes | Yes | No |
| NodePort Service | External access | Yes | Yes | Limited |
| Port Forward | Development, testing | Yes | Yes | No |
| Secondary Network | Traditional infrastructure | Yes | Yes | Yes |

## Prerequisites

- OpenShift 4.17+ with OpenShift Virtualization operator installed
- CLI tools: `oc`, `virtctl`, `kubectl`
- At least one running Linux VM (we'll create one for this tutorial)
- Basic understanding of SSH and networking

## Setup: Deploy Test VM

Create a namespace and deploy a Fedora VM:

```bash
oc create namespace vm-access-demo
```

Create VM with SSH enabled:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: fedora-access-vm
  namespace: vm-access-demo
spec:
  dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1beta1
      kind: DataVolume
      metadata:
        name: fedora-access-vm-disk
      spec:
        sourceRef:
          kind: DataSource
          name: fedora
          namespace: openshift-virtualization-os-images
        storage:
          resources:
            requests:
              storage: 30Gi
  runStrategy: RerunOnFailure
  template:
    metadata:
      labels:
        kubevirt.io/domain: fedora-access-vm
    spec:
      domain:
        cpu:
          cores: 2
        devices:
          disks:
            - disk:
                bus: virtio
              name: rootdisk
            - disk:
                bus: virtio
              name: cloudinitdisk
          interfaces:
            - masquerade: {}
              name: default
          rng: {}
        machine:
          type: pc-q35-rhel9.4.0
        memory:
          guest: 4Gi
      networks:
        - name: default
          pod: {}
      volumes:
        - dataVolume:
            name: fedora-access-vm-disk
          name: rootdisk
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              user: fedora
              password: fedora123
              chpasswd: { expire: False}
              ssh_pwauth: True
              ssh_authorized_keys:
                - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC... your-public-key-here
          name: cloudinitdisk
```

Note: Replace `your-public-key-here` with your actual SSH public key, or use password authentication for testing.

Deploy the VM:

```bash
oc apply -f fedora-access-vm.yaml
```

Wait for the VM to be ready:

```bash
oc get vmi -n vm-access-demo
```

## Method 1: Console Access with virtctl

The most basic access method - direct console access to the VM.

### Access the Console

```bash
virtctl console fedora-access-vm -n vm-access-demo
```

### Exit the Console

Press `Ctrl+]` to exit.

### When to Use

- Initial VM setup and configuration
- No network connectivity troubleshooting
- VM doesn't respond to SSH
- Emergency access when network is broken

### Limitations

- No copy/paste support (limited)
- No scroll-back
- Text-only interface
- Single session at a time

## Method 2: VNC Access

For VMs with graphical desktop environments.

### Start VNC Session

```bash
virtctl vnc fedora-access-vm -n vm-access-demo
```

This launches a VNC viewer connecting to the VM's display.

### When to Use

- Accessing desktop Linux VMs
- Running GUI applications
- Installing software that requires GUI
- Troubleshooting display issues

### Requirements

- VNC viewer installed locally
- VM with desktop environment (GNOME, KDE, etc.)

## Method 3: SSH via virtctl

Quick SSH access through virtctl command.

### Prerequisites

- SSH key configured in VM (via cloud-init)
- Guest agent running in VM

### Connect via SSH

```bash
virtctl ssh fedora@fedora-access-vm -n vm-access-demo
```

### Skip Host Key Checking (for testing)

```bash
virtctl ssh fedora@fedora-access-vm -n vm-access-demo --local-ssh-opts="-o StrictHostKeyChecking=no"
```

### Verify Guest Agent

```bash
oc get vmi fedora-access-vm -n vm-access-demo -o jsonpath='{.status.guestOSInfo}' | jq .
```

Output should show guest OS information if agent is running.

### When to Use

- Quick administrative access
- Development and testing
- Temporary access needs
- When you have cluster access via oc/kubectl

### Limitations

- Requires cluster access
- Depends on guest agent
- Not suitable for production external access

## Method 4: Direct SSH via NodePort Service

Expose SSH port externally using a NodePort Service.

### Create NodePort Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fedora-ssh-nodeport
  namespace: vm-access-demo
spec:
  type: NodePort
  selector:
    kubevirt.io/domain: fedora-access-vm
  ports:
    - protocol: TCP
      port: 22
      targetPort: 22
      nodePort: 30022
```

Apply the service:

```bash
oc apply -f fedora-ssh-nodeport.yaml
```

### Verify the Service

```bash
oc get svc fedora-ssh-nodeport -n vm-access-demo
```

### Connect via SSH

Get your cluster node IP:

```bash
oc get nodes -o wide
```

Connect using the NodePort:

```bash
ssh fedora@<node-ip> -p 30022
```

### When to Use

- External access required
- Simple deployment without load balancer
- Development/testing clusters
- Quick external access setup

### Security Considerations

- Exposes SSH port on all cluster nodes
- Use strong authentication (SSH keys only)
- Consider firewall rules
- Not recommended for production
- Monitor for unauthorized access attempts

## Method 5: SSH via Port-Forward

Temporary SSH access using kubectl/oc port-forward.

### Find the VM Pod

```bash
oc get pods -n vm-access-demo
```

Output:
```
NAME                                   READY   STATUS    RESTARTS   AGE
virt-launcher-fedora-access-vm-xxxxx   2/2     Running   0          10m
```

### Forward SSH Port

```bash
oc port-forward -n vm-access-demo virt-launcher-fedora-access-vm-xxxxx 2222:22
```

### Connect via SSH (in another terminal)

```bash
ssh fedora@localhost -p 2222
```

### When to Use

- Quick temporary access
- Development and testing
- Troubleshooting specific VMs
- When NodePort/LoadBalancer unavailable

### Limitations

- Port-forward must stay active
- Single connection at a time
- Not suitable for production
- Requires active terminal session

## Method 6: Direct Network Access via Secondary Networks

For VMs on bridge or localnet networks, access directly via IP.

### Prerequisites

- VM attached to secondary network (bridge, localnet, VLAN)
- Network reachable from your location
- Static IP configured or DHCP with reserved address

### VM with Secondary Network

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: fedora-vlan-vm
  namespace: vm-access-demo
spec:
  dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1beta1
      kind: DataVolume
      metadata:
        name: fedora-vlan-vm-disk
      spec:
        sourceRef:
          kind: DataSource
          name: fedora
          namespace: openshift-virtualization-os-images
        storage:
          resources:
            requests:
              storage: 30Gi
  runStrategy: RerunOnFailure
  template:
    metadata:
      labels:
        kubevirt.io/domain: fedora-vlan-vm
    spec:
      domain:
        cpu:
          cores: 2
        devices:
          disks:
            - disk:
                bus: virtio
              name: rootdisk
            - disk:
                bus: virtio
              name: cloudinitdisk
          interfaces:
            - masquerade: {}
              name: default
            - bridge: {}
              name: vlan-network
          rng: {}
        machine:
          type: pc-q35-rhel9.4.0
        memory:
          guest: 4Gi
      networks:
        - name: default
          pod: {}
        - name: vlan-network
          multus:
            networkName: localnet-vlan-100
      volumes:
        - dataVolume:
            name: fedora-vlan-vm-disk
          name: rootdisk
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              user: fedora
              password: fedora123
              chpasswd: { expire: False }
              ssh_pwauth: True
              ssh_authorized_keys:
                - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC... your-public-key-here
            networkData: |-
              version: 2
              ethernets:
                enp1s0:
                  dhcp4: true
                enp2s0:
                  dhcp4: false
                  dhcp6: false
                  addresses:
                    - 192.168.100.100/24
          name: cloudinitdisk
```

### Connect via SSH

```bash
ssh fedora@192.168.100.100
```

### When to Use

- Production environments
- Traditional infrastructure patterns
- VMs needing persistent, predictable access
- Integration with existing networks
- Multiple VMs on same network segment

### Advantages

- Native network access
- No port-forwarding overhead
- Multiple simultaneous connections
- Standard network tooling works
- Fits traditional architecture

## SSH Key Configuration

### Generate SSH Key (if needed)

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/ocp-virt-key
```

### Add Key to Cloud-init

Include in the VM's userData section:

```yaml
ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC... your-key-here
```

### Disable Password Authentication

For production, disable password auth:

```yaml
#cloud-config
ssh_pwauth: False
disable_root: true
```

## Verification Commands

### Check VM Status

```bash
# VirtualMachine
oc get vm -n vm-access-demo

# VirtualMachineInstance
oc get vmi -n vm-access-demo

# Detailed VM info
oc describe vmi fedora-access-vm -n vm-access-demo
```

### Check SSH Service in VM

Via console:

```bash
virtctl console fedora-access-vm -n vm-access-demo
```

Inside the VM:
```bash
# Check SSH service
sudo systemctl status sshd

# Check SSH is listening
sudo ss -tlnp | grep :22

# Check firewall
sudo firewall-cmd --list-all
```

### Test Network Connectivity

```bash
# From inside VM
ping -c 3 8.8.8.8

# From outside (if on same network)
ping -c 3 192.168.100.100
```

## Common Issues and Solutions

### Issue: virtctl console shows no login prompt

**Problem**: VM not fully booted or display issue.

**Solution**: Wait longer, check VM status, verify VM is running.

### Issue: SSH connection refused

**Problem**: SSH service not running or firewall blocking.

**Solution**:
- Check sshd status: `sudo systemctl status sshd`
- Start if needed: `sudo systemctl start sshd`
- Check firewall: `sudo firewall-cmd --list-services`

### Issue: virtctl ssh fails with guest agent error

**Problem**: Guest agent not installed or not running.

**Solution**: Install qemu-guest-agent package in the VM:
```bash
sudo dnf install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

### Issue: NodePort not accessible

**Problem**: Firewall on cluster nodes blocking port.

**Solution**: Check node firewall rules, ensure NodePort range is open.

### Issue: Can't connect to secondary network IP

**Problem**: Network not reachable or routing issue.

**Solution**: Verify network configuration, check routes, ensure VLAN/bridge is properly configured.

For detailed troubleshooting, see the [VM Remote Access Troubleshooting Guide](../troubleshooting/vm-remote-access-troubleshooting.md).

## Security Best Practices

### Authentication
- Use SSH keys only, disable password authentication
- Rotate SSH keys regularly
- Use different keys for different environments
- Implement key management policies

### Network Security
- Limit NodePort exposure to specific IPs if possible
- Use network policies to restrict VM access
- Configure VM firewall appropriately
- Monitor access logs regularly

### Access Control
- Implement RBAC for virtctl access
- Use separate namespaces for different environments
- Document who has access to what VMs
- Regular access audits

## When to Use Each Method

### Console/VNC
- Initial setup
- Troubleshooting
- GUI applications
- Emergency access

### virtctl SSH
- Quick admin tasks
- Development work
- Testing
- When you have cluster access

### NodePort
- Testing external access
- Dev/test environments
- Quick external access setup
- Non-production workloads

### Port-Forward
- Temporary debugging
- One-off access
- Development testing
- When other methods unavailable

### Secondary Network
- Production environments
- Traditional infrastructure
- Multiple VMs needing access
- Integration with existing networks

## Cleanup

Remove all resources created in this tutorial:

```bash
# Delete VMs
oc delete vm fedora-access-vm fedora-vlan-vm -n vm-access-demo

# Delete Service
oc delete svc fedora-ssh-nodeport -n vm-access-demo

# Delete namespace
oc delete namespace vm-access-demo
```

## Summary

In this tutorial, you learned:
- Multiple methods to access Linux VMs in OpenShift Virtualization
- When to use each access method
- How to configure SSH keys via cloud-init
- How to expose VMs externally using Services
- How to use port-forwarding for temporary access
- How to access VMs on secondary networks
- Security best practices for VM access
- Common issues and troubleshooting steps

## References

- [OpenShift Virtualization Documentation](https://docs.openshift.com/container-platform/latest/virt/about_virt/about-virt.html)
- [virtctl Documentation](https://kubevirt.io/user-guide/operations/virtctl_client_tool/)
- [KubeVirt User Guide](https://kubevirt.io/user-guide/)
- [SSH Best Practices](https://www.ssh.com/academy/ssh/best-practices)

