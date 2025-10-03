# Configuring Localnet Secondary Networks with VLAN using ClusterUserDefinedNetwork in OpenShift Virtualization

This tutorial demonstrates how to configure localnet secondary networks with VLAN tagging using ClusterUserDefinedNetwork (CUDN) in OpenShift Virtualization. This approach provides direct access to physical network infrastructure with VLAN segmentation across multiple namespaces, enabling VMs to communicate with specific VLAN-tagged networks while maintaining pod network connectivity. CUDN offers centralized management and cross-namespace network availability, making it ideal for enterprise environments requiring consistent VLAN-based network infrastructure.

## Prerequisites

- **NMState Operator**: Must be installed and running in the cluster
- **NMState CR**: The nmstate instance must be created after the operator is running
- **Worker nodes**: The bridge mapping will be applied to worker nodes
- **VLAN Infrastructure**: Physical network infrastructure must support VLAN tagging
- **Switch Configuration**: Physical switch ports connected to OpenShift worker nodes must be configured in **trunk mode** to handle multiple VLAN-tagged traffic streams

Versions tested:
```
OCP 4.19
```

## Network Configuration Requirements

### Physical Switch Configuration: TRUNK MODE

The physical switch ports connected to OpenShift worker nodes **must be configured in trunk mode** for localnet VLAN configuration to work properly:

- **Trunk mode**: Allows switch ports to carry traffic for multiple VLANs
- **VLAN tagging**: Each VLAN is identified by its unique VLAN ID (e.g., 100, 200, 300)
- **Multiple networks**: Essential for supporting multiple VLAN-based secondary networks

## Step 1: Create VM Namespaces

Create namespaces for VMs that will use the localnet secondary network with VLAN configuration. CUDN allows the same network to be available across multiple namespaces.

```bash
# Create first namespace
oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: vm-guests-vlan-prod
EOF

# Create second namespace
oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: vm-guests-vlan-dev
EOF
```

## Step 2: Configure Bridge Mapping

Configure the bridge mapping that connects the logical network interface name `localnet-vlan` to the `br-ex` bridge interface on worker nodes. This mapping tells OVN-Kubernetes which physical bridge interface handles localnet traffic with VLAN support, enabling VM access to VLAN-tagged external networks through the underlying physical network infrastructure.

```bash
oc apply -f - <<EOF
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: cudn-localnet-vlan-bridge-mapping
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ''  
  desiredState:
    ovn:
      bridge-mappings:
      - localnet: localnet-vlan
        bridge: br-ex
        state: present
EOF
```

## Step 3: Create ClusterUserDefinedNetwork with VLAN

Create a ClusterUserDefinedNetwork that configures localnet topology with VLAN tagging for secondary network connectivity across multiple namespaces.

```bash
oc apply -f - <<EOF
apiVersion: k8s.ovn.org/v1
kind: ClusterUserDefinedNetwork
metadata:
  name: cudn-localnet-vlan-100
spec:
  namespaceSelector: 
    matchExpressions: 
    - key: kubernetes.io/metadata.name
      operator: In 
      values: ["vm-guests-vlan-prod", "vm-guests-vlan-dev"]
  network:
    topology: Localnet 
    localnet:
      role: Secondary 
      physicalNetworkName: localnet-vlan
      vlanID: 100
      ipam:
        mode: Disabled
EOF
```

**Configuration Details**:
- `vlanID: 100`: Specifies VLAN tag 100 for network segmentation
- `physicalNetworkName`: References the logical network interface name from bridge mapping
- `role: Secondary`: Defines this as a secondary network (VMs keep pod network as primary)
- `namespaceSelector`: Defines which namespaces can use this network
- **IPAM Disabled**: IP addresses require external DHCP server on the VLAN network or VMs need to be configured statically

## Step 4: Deploy VM with VLAN Network Connectivity

Create a VM in one of the configured namespaces that connects to both the pod network and the VLAN-tagged localnet secondary network.

```bash
oc apply -f - <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: fedora-cudn-vlan-vm
  namespace: vm-guests-vlan-prod
  labels:
    app: fedora-cudn-vlan-vm
spec:
  runStrategy: Always
  dataVolumeTemplates:
  - metadata:
      name: fedora-cudn-vlan-volume
    spec:
      pvc:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 33Gi
      sourceRef:
        kind: DataSource
        name: fedora
        namespace: openshift-virtualization-os-images
  template:
    metadata:
      labels:
        app: fedora-cudn-vlan-vm
    spec:
      domain:
        devices:
          disks:
          - name: datavolumedisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
          interfaces:
          - name: default
            bridge: {}
          - name: cudn_vlan_network
            bridge: {}
        resources:
          requests:
            memory: 2Gi
            cpu: 1
      networks:
      - name: default
        pod: {}
      - name: cudn_vlan_network
        multus:
          networkName: cudn-localnet-vlan-100
      volumes:
      - name: datavolumedisk
        dataVolume:
          name: fedora-cudn-vlan-volume
      - name: cloudinitdisk
        cloudInitNoCloud:
          userData: |
            #cloud-config
            user: fedora
            password: fedora
            chpasswd: { expire: False }
            packages:
            - python3
            - tcpdump
            - net-tools
            runcmd:
            - echo "<h1>Welcome to OpenShift Virtualization CUDN VLAN Network!</h1>" > /root/index.html
            - echo "<p>VLAN ID: 100</p>" >> /root/index.html
            - echo "<p>Network: CUDN Localnet</p>" >> /root/index.html
            - echo "<p>Namespace: vm-guests-vlan-prod</p>" >> /root/index.html
            - cd /root && nohup python3 -m http.server 8080 > /dev/null 2>&1 &
EOF
```

## Step 5: Deploy VM in Second Namespace

Demonstrate cross-namespace network availability by creating a VM in the second namespace using the same CUDN.

```bash
oc apply -f - <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: fedora-cudn-vlan-vm-dev
  namespace: vm-guests-vlan-dev
  labels:
    app: fedora-cudn-vlan-vm-dev
spec:
  runStrategy: Always
  dataVolumeTemplates:
  - metadata:
      name: fedora-cudn-vlan-volume-dev
    spec:
      pvc:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 33Gi
      sourceRef:
        kind: DataSource
        name: fedora
        namespace: openshift-virtualization-os-images
  template:
    metadata:
      labels:
        app: fedora-cudn-vlan-vm-dev
    spec:
      domain:
        devices:
          disks:
          - name: datavolumedisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
          interfaces:
          - name: default
            bridge: {}
          - name: cudn_vlan_network
            bridge: {}
        resources:
          requests:
            memory: 2Gi
            cpu: 1
      networks:
      - name: default
        pod: {}
      - name: cudn_vlan_network
        multus:
          networkName: cudn-localnet-vlan-100
      volumes:
      - name: datavolumedisk
        dataVolume:
          name: fedora-cudn-vlan-volume-dev
      - name: cloudinitdisk
        cloudInitNoCloud:
          userData: |
            #cloud-config
            user: fedora
            password: fedora
            chpasswd: { expire: False }
            packages:
            - python3
            - tcpdump
            - net-tools
            runcmd:
            - echo "<h1>Welcome to OpenShift Virtualization CUDN VLAN Network!</h1>" > /root/index.html
            - echo "<p>VLAN ID: 100</p>" >> /root/index.html
            - echo "<p>Network: CUDN Localnet</p>" >> /root/index.html
            - echo "<p>Namespace: vm-guests-vlan-dev</p>" >> /root/index.html
            - cd /root && nohup python3 -m http.server 8080 > /dev/null 2>&1 &
EOF
```

## Step 6: Verify VLAN Network Connectivity

Access the VM consoles to verify dual network connectivity with VLAN configuration across namespaces.

```bash
# Check VMs in production namespace
oc get vm -n vm-guests-vlan-prod
oc get vmi -n vm-guests-vlan-prod

# Check VMs in development namespace
oc get vm -n vm-guests-vlan-dev
oc get vmi -n vm-guests-vlan-dev

# Access production VM console (credentials: fedora/fedora via cloud-init)
virtctl console -n vm-guests-vlan-prod fedora-cudn-vlan-vm

# Access development VM console (credentials: fedora/fedora via cloud-init)
virtctl console -n vm-guests-vlan-dev fedora-cudn-vlan-vm-dev
```

Inside each VM console, verify the VLAN network configuration:

```bash
# Check network interfaces - primary (pod network) and secondary (VLAN network)
ip addr show

# Check routes - two default gateways (lower metric prioritized)
# Primary network: metric 100, Secondary VLAN: metric 101
ip route show

# Expected output example:
default via 10.128.0.1 dev enp1s0 proto dhcp src 10.128.0.172 metric 100 
default via 192.168.100.1 dev enp2s0 proto dhcp src 192.168.100.50 metric 101 

# Test external connectivity
ping 8.8.8.8

# Verify the web service is running
curl localhost:8080

# Test VLAN interface connectivity (replace <vlan-ip> with actual IP)
curl <vlan-ip>:8080

# Verify VLAN tagging (if tcpdump is available)
# This will show VLAN-tagged traffic on the secondary interface
sudo tcpdump -i enp2s0 -nn vlan 100
```

## Step 7: Test Cross-Namespace Connectivity

Verify that VMs in different namespaces can communicate over the shared CUDN VLAN network:

```bash
# From production VM, ping development VM (replace with actual VLAN IPs)
ping <dev-vm-vlan-ip>

# Test web service connectivity between namespaces
curl <dev-vm-vlan-ip>:8080
```

## Advanced CUDN VLAN Configuration

### Multiple VLAN Networks

Create additional CUDNs for different VLANs:

```bash
# VLAN 200 for production traffic
oc apply -f - <<EOF
apiVersion: k8s.ovn.org/v1
kind: ClusterUserDefinedNetwork
metadata:
  name: cudn-localnet-vlan-200
spec:
  namespaceSelector: 
    matchExpressions: 
    - key: kubernetes.io/metadata.name
      operator: In 
      values: ["vm-guests-vlan-prod"]
  network:
    topology: Localnet 
    localnet:
      role: Secondary 
      physicalNetworkName: localnet-vlan
      vlanID: 200
      ipam:
        mode: Disabled
EOF
```

### Static IP Configuration

For static IP assignment within the VLAN network:

```bash
oc apply -f - <<EOF
apiVersion: k8s.ovn.org/v1
kind: ClusterUserDefinedNetwork
metadata:
  name: cudn-localnet-vlan-static
spec:
  namespaceSelector: 
    matchExpressions: 
    - key: kubernetes.io/metadata.name
      operator: In 
      values: ["vm-guests-vlan-prod"]
  network:
    topology: Localnet 
    localnet:
      role: Secondary 
      physicalNetworkName: localnet-vlan
      vlanID: 300
      ipam:
        mode: Static
        staticIPAMConfig:
          addresses:
          - "192.168.300.0/24"
          excludeAddresses:
          - "192.168.300.1/32"
EOF
```

## Troubleshooting

### Common Issues

1. **VLAN Traffic Not Reaching VM**:
   - Verify physical switch configuration supports VLAN tagging
   - Check bridge mapping configuration on worker nodes
   - Ensure VLAN ID matches network infrastructure

2. **IP Address Assignment Issues**:
   - Verify DHCP server is available on the VLAN network

3. **Network Connectivity Problems**:
   - Check that the physical network supports the configured VLAN
   - Test connectivity from physical network to VLAN subnet

4. **Cross-Namespace Issues**:
   - Verify namespace selector in CUDN configuration
   - Check that both namespaces are listed in the values array

### Verification Commands

```bash
# Check ClusterUserDefinedNetwork status
oc get clusteuserdefinednetwork

# Verify bridge mapping on worker nodes
oc get nncp cudn-localnet-vlan-bridge-mapping -o yaml

# Check VM network status in both namespaces
oc get vmi -n vm-guests-vlan-prod -o yaml | grep -A 10 networks
oc get vmi -n vm-guests-vlan-dev -o yaml | grep -A 10 networks

# View OVN-Kubernetes logs for troubleshooting
oc logs -n openshift-ovn-kubernetes -l app=ovnkube-node

# Check CUDN network attachment across namespaces
oc get network-attachment-definitions -A | grep cudn-localnet-vlan-100
```

## CUDN vs NetworkAttachmentDefinition Comparison

| Feature | CUDN | NetworkAttachmentDefinition |
|---------|------|----------------------------|
| **Scope** | Cluster-wide, multi-namespace | Namespace-specific |
| **Management** | Centralized configuration | Per-namespace configuration |
| **VLAN Support** | Native vlanID parameter | Native vlanID parameter |
| **IP Management** | Static/Disabled modes | Static/DHCP via IPAM |
| **Use Case** | Enterprise, consistent networks | Application-specific networks |

## References

### OpenShift Documentation
- [OpenShift - Understanding Multiple Networks](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/multiple_networks/understanding-multiple-networks)
- [ClusterUserDefinedNetwork Configuration](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/networking#virt-creating-secondary-localnet-cudn_virt-connecting-vm-to-secondary-cudn)

