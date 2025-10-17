# Configuring IP Addresses with Cloud-init

## Overview

This tutorial demonstrates how to configure static and dynamic IP addresses for virtual machines in OpenShift Virtualization using cloud-init. Cloud-init is a standard method for initializing cloud instances with custom configurations, including network settings.

##What is Cloud-init?

Cloud-init is an industry-standard initialization tool that runs during VM boot to:
- Configure network interfaces
- Set hostnames
- Create users and SSH keys
- Run custom scripts
- Install packages

For networking, cloud-init supports both userData (generic configuration) and networkData (dedicated network configuration) sections.

## Prerequisites

- OpenShift 4.17+ with OpenShift Virtualization operator installed
- CLI tools: `oc`, `virtctl`, `kubectl`
- At least one configured network (primary pod network)
- For VLAN scenarios: secondary network configured (see localnet VLAN tutorial)
- Basic understanding of networking (IP, subnet, gateway, DNS)

## Interface Naming in KubeVirt

KubeVirt uses predictable network interface naming based on PCI slot positions:
- First interface: `enp1s0`
- Second interface: `enp2s0`
- Third interface: `enp3s0`

This differs from traditional `eth0`, `eth1` naming. Always use `enpXsY` format in cloud-init network configurations.

## Cloud-init Network Configuration Basics

### userData vs networkData

- **userData**: General cloud-init configuration (users, packages, scripts)
- **networkData**: Dedicated network interface configuration (preferred for networking)

### Configuration Format

OpenShift Virtualization uses cloud-init network configuration version 2 (netplan-compatible):

```yaml
networkData: |
  version: 2
  ethernets:
    enp1s0:
      dhcp4: true
```

### Precedence Order

1. networkData configuration (highest priority)
2. Datasource network configuration
3. Fallback to DHCP (default)

## Scenario 1: Single Interface with DHCP (Default)

The simplest configuration - VM uses DHCP on the primary pod network.

Create namespace:
```bash
oc create namespace cloud-init-demo
```

VM manifest without explicit network configuration:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: fedora-dhcp-vm
  namespace: cloud-init-demo
spec:
  dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1beta1
      kind: DataVolume
      metadata:
        name: fedora-dhcp-vm-disk
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
        kubevirt.io/domain: fedora-dhcp-vm
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
            name: fedora-dhcp-vm-disk
          name: rootdisk
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              user: fedora
              password: fedora123
              chpasswd: { expire: False }
              ssh_pwauth: True
          name: cloudinitdisk
```

Deploy and verify:

```bash
oc apply -f fedora-dhcp-vm.yaml
oc get vmi -n cloud-init-demo
```

Access the VM and check the IP:

```bash
virtctl console fedora-dhcp-vm -n cloud-init-demo
```

Inside the VM:
```bash
ip addr show enp1s0
```

The interface will have an IP from the pod network (typically 10.128.x.x range).

When to use: Development environments, simple deployments, when DHCP is sufficient.

## Scenario 2: Single Interface with Static IP

Configure a static IP on the primary interface. Note: This is not common for pod networks but useful for understanding the syntax.

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: fedora-static-vm
  namespace: cloud-init-demo
spec:
  dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1beta1
      kind: DataVolume
      metadata:
        name: fedora-static-vm-disk
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
        kubevirt.io/domain: fedora-static-vm
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
            name: fedora-static-vm-disk
          name: rootdisk
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              user: fedora
              password: fedora123
              chpasswd: { expire: False }
              ssh_pwauth: True
            networkData: |-
              version: 2
              ethernets:
                enp1s0:
                  dhcp4: false
                  dhcp6: false
                  addresses:
                    - 10.128.1.100/23
                  routes:
                    - to: default
                      via: 10.128.1.1
                  nameservers:
                    addresses:
                      - 8.8.8.8
                      - 8.8.4.4
          name: cloudinitdisk
```

Key points:
- `dhcp4: false` and `dhcp6: false` disable DHCP
- `addresses` sets the static IP with CIDR notation
- `routes` with `to: default` sets the default gateway
- `nameservers` configures DNS servers

## Scenario 3: Multiple Interfaces - Primary DHCP + Secondary Static

This is the most common real-world scenario. Primary interface uses DHCP, secondary interface (on VLAN or bridge) uses static IP.

Prerequisites: Ensure you have a secondary network configured (e.g., localnet VLAN 100).

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: fedora-multi-vm
  namespace: cloud-init-demo
spec:
  dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1beta1
      kind: DataVolume
      metadata:
        name: fedora-multi-vm-disk
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
        kubevirt.io/domain: fedora-multi-vm
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
            name: fedora-multi-vm-disk
          name: rootdisk
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              user: fedora
              password: fedora123
              chpasswd: { expire: False }
              ssh_pwauth: True
            networkData: |-
              version: 2
              ethernets:
                enp1s0:
                  dhcp4: true
                enp2s0:
                  dhcp4: false
                  dhcp6: false
                  addresses:
                    - 192.168.100.50/24
                  nameservers:
                    addresses:
                      - 192.168.100.1
          name: cloudinitdisk
```

Important notes:
- `enp1s0` (first interface) uses DHCP and gets the default route automatically
- `enp2s0` (second interface) has static IP without default route
- DNS servers on secondary interface are optional
- DO NOT set default route on secondary interface

Verification:

```bash
virtctl console fedora-multi-vm -n cloud-init-demo
```

Inside the VM:
```bash
# Check both interfaces
ip addr show

# Verify routing table (only one default route via enp1s0)
ip route show

# Test secondary network
ping 192.168.100.1
```

## Scenario 4: VLAN Interface with Static IP

Similar to Scenario 3 but specifically for VLAN networks.

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: fedora-vlan-vm
  namespace: cloud-init-demo
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
              name: vlan-100
          rng: {}
        machine:
          type: pc-q35-rhel9.4.0
        memory:
          guest: 4Gi
      networks:
        - name: default
          pod: {}
        - name: vlan-100
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
              write_files:
                - path: /var/www/html/index.html
                  content: |
                    <html>
                    <body>
                      <h1>VM on VLAN 100</h1>
                      <p>IP: 192.168.100.50</p>
                    </body>
                    </html>
                  permissions: '0644'
              runcmd:
                - dnf install -y python3
                - cd /var/www/html && python3 -m http.server 80 &
            networkData: |-
              version: 2
              ethernets:
                enp1s0:
                  dhcp4: true
                enp2s0:
                  dhcp4: false
                  dhcp6: false
                  addresses:
                    - 192.168.100.50/24
          name: cloudinitdisk
```

This example also demonstrates:
- Using `write_files` to create content
- Using `runcmd` to start a simple web server
- Static IP on VLAN interface

## Verification Commands

After deploying any scenario, use these commands to verify:

### Check VM Status

```bash
# Check VirtualMachine
oc get vm -n cloud-init-demo

# Check VirtualMachineInstance
oc get vmi -n cloud-init-demo

# Get detailed VMI info
oc describe vmi <vm-name> -n cloud-init-demo
```

### Access VM Console

```bash
virtctl console <vm-name> -n cloud-init-demo
```

Exit console with `Ctrl+]`.

### Inside the VM

```bash
# Show all interfaces
ip addr show

# Show routing table
ip route show

# Show DNS configuration
cat /etc/resolv.conf

# Test connectivity
ping -c 3 8.8.8.8
ping -c 3 google.com

# Check cloud-init status
cloud-init status

# View cloud-init logs
sudo cat /var/log/cloud-init.log
sudo cat /var/log/cloud-init-output.log

# Check applied network configuration
nmcli device show
nmcli connection show
```

## Common Issues and Solutions

### Issue: Interface names don't match

**Problem**: Used `eth0` instead of `enp1s0` in networkData.

**Solution**: Always use KubeVirt's predictable naming: `enp1s0`, `enp2s0`, etc.

### Issue: VM has no connectivity on secondary interface

**Problem**: Forgot to disable DHCP on static interface.

**Solution**: Always set `dhcp4: false` and `dhcp6: false` when using static IPs.

### Issue: Multiple default routes

**Problem**: Set default route on both interfaces.

**Solution**: Only set default route on primary interface (usually enp1s0). Secondary interfaces should not have default routes.

### Issue: Cloud-init configuration not applied

**Problem**: Syntax error in YAML or cloud-init configuration.

**Solution**: 
- Check cloud-init logs: `/var/log/cloud-init.log`
- Verify YAML indentation
- Use `cloud-init status` to check for errors

### Issue: DNS not resolving on secondary network

**Problem**: Nameservers not configured or wrong nameservers.

**Solution**: Add appropriate nameservers in the networkData for the interface. If secondary interface is for isolated network, may not need external DNS.

For detailed troubleshooting, see the [Cloud-init Troubleshooting Guide](../troubleshooting/cloud-init-troubleshooting.md).

## Best Practices

### Network Configuration
- Use networkData for all network interface configurations
- Always disable DHCP explicitly when using static IPs
- Only configure default route on primary interface
- Use predictable interface naming (enpXsY)
- Test configurations in dev environment first

### Cloud-init Usage
- Keep configurations simple and readable
- Use write_files for complex file content
- Avoid complex shell scripts in runcmd
- Validate YAML syntax before deploying
- Check cloud-init logs when troubleshooting

### IP Address Planning
- Document IP assignments for all VMs
- Use consistent IP ranges for different purposes
- Avoid IP conflicts by maintaining IP inventory
- Consider using DHCP reservations for dynamic scenarios

### Security
- Don't hardcode passwords in production
- Use SSH keys instead of passwords
- Restrict access to appropriate networks
- Configure firewall rules via cloud-init if needed

## When to Use Each Approach

### Use DHCP when:
- Quick development and testing
- IP address doesn't need to be predictable
- DHCP server is available and reliable
- VM is ephemeral

### Use Static IP when:
- VM hosts a service that needs consistent addressing
- Integration with external systems requires fixed IP
- DHCP is not available on the network
- Production environments requiring predictability

### Use Multiple Interfaces when:
- Separating management and data traffic
- Connecting to multiple networks (pod network + VLAN)
- Implementing network segmentation
- Compliance requirements for network isolation

## Cleanup

To remove all resources created in this tutorial:

```bash
# Delete VMs
oc delete vm fedora-dhcp-vm fedora-static-vm fedora-multi-vm fedora-vlan-vm -n cloud-init-demo

# Delete namespace
oc delete namespace cloud-init-demo
```

## Summary

In this tutorial, you learned:
- How cloud-init network configuration works in OpenShift Virtualization
- KubeVirt's predictable network interface naming
- How to configure DHCP and static IP addresses
- How to configure multiple network interfaces
- How to verify network configuration
- Best practices for cloud-init networking
- Common issues and solutions

## References

- [Cloud-init Documentation](https://cloudinit.readthedocs.io/)
- [Cloud-init Network Configuration](https://cloudinit.readthedocs.io/en/latest/reference/network-config.html)
- [OpenShift Virtualization Documentation](https://docs.openshift.com/container-platform/latest/virt/about_virt/about-virt.html)
- [KubeVirt Documentation](https://kubevirt.io/user-guide/)

