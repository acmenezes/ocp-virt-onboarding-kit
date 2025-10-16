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

## Step 2: Create Custom OVS Bridge on Extra NIC

First, create a custom OVS bridge on an additional physical network interface (NIC) to isolate VLAN traffic from the primary cluster network.

```bash
oc apply -f - <<EOF
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br-vlan-creation
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ''
  desiredState:
    interfaces:
    - name: enp11s0f0np0  # Replace with your physical interface name
      type: ethernet
      state: up
      ipv4:
        enabled: false
      ipv6:
        enabled: false
      description: "Physical interface for VLAN localnet"
    - name: br-vlan
      type: ovs-bridge
      state: up
      bridge:
        port:
        - name: enp11s0f0np0
      description: "OVS bridge for VLAN traffic"
EOF
```

**Note**: Replace `enp11s0f0np0` with the name of your extra physical interface. You can find available interfaces by running:
```bash
oc debug node/<node-name> -- ip link show
```

Verify the bridge creation:
```bash
oc get nncp br-vlan-creation
# Expected output: STATUS: Available
```

## Step 3: Configure Bridge Mapping

Configure the bridge mapping that connects the logical network interface name `localnet-vlan` to the `br-vlan` OVS bridge on worker nodes. This mapping tells OVN-Kubernetes which physical bridge interface handles localnet traffic with VLAN support.

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
        bridge: br-vlan
        state: present
EOF
```

## Step 4: Create ClusterUserDefinedNetwork with VLAN

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
- `physicalNetworkName`: References the logical network interface name from bridge mapping (`localnet-vlan`)
- `role: Secondary`: Defines this as a secondary network (VMs keep pod network as primary)
- `namespaceSelector`: Defines which namespaces can use this network
- **IPAM Disabled**: IP addresses must be configured via cloud-init (static) or external DHCP server

## Step 5: Deploy VM with VLAN Network Connectivity and Static IP

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
          networkData: |
            version: 2
            ethernets:
              enp2s0:
                addresses: ["192.168.100.10/24"]
          userData: |
            #cloud-config
            user: fedora
            password: fedora
            chpasswd: { expire: False }
            packages:
            - python3
            - tcpdump
            - net-tools
            write_files:
            - path: /root/index.html
              content: |
                <h1>Welcome to OpenShift Virtualization CUDN VLAN Network!</h1>
                <p>VLAN ID: 100</p>
                <p>Static IP: 192.168.100.10/24</p>
                <p>Network: CUDN Localnet</p>
                <p>Namespace: vm-guests-vlan-prod</p>
              permissions: '0644'
            runcmd:
            - nohup python3 -m http.server 8080 --directory /root > /dev/null 2>&1 &
EOF
```

## Step 6: Deploy VM in Second Namespace

Demonstrate cross-namespace network availability by creating a VM in the second namespace using the same CUDN with a different static IP.

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
          networkData: |
            version: 2
            ethernets:
              enp2s0:
                addresses: ["192.168.100.20/24"]
          userData: |
            #cloud-config
            user: fedora
            password: fedora
            chpasswd: { expire: False }
            packages:
            - python3
            - tcpdump
            - net-tools
            write_files:
            - path: /root/index.html
              content: |
                <h1>Welcome to OpenShift Virtualization CUDN VLAN Network!</h1>
                <p>VLAN ID: 100</p>
                <p>Static IP: 192.168.100.20/24</p>
                <p>Network: CUDN Localnet</p>
                <p>Namespace: vm-guests-vlan-dev</p>
              permissions: '0644'
            runcmd:
            - nohup python3 -m http.server 8080 --directory /root > /dev/null 2>&1 &
EOF
```

## Step 7: Verify VLAN Network Connectivity

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

# Check routes
ip route show

# Expected output example (primary network via DHCP):
default via 10.128.0.1 dev enp1s0 proto dhcp src 10.128.0.174 metric 100 

# Test external connectivity
ping 8.8.8.8

# Verify the web service is running
curl localhost:8080

# Verify static IP assignment on VLAN interface
ip addr show enp2s0
# Expected: inet 192.168.100.10/24 (prod) or 192.168.100.20/24 (dev)

# Test VLAN interface connectivity between VMs
# From production VM (192.168.100.10) ping development VM:
ping 192.168.100.20

# Verify VLAN tagging (if tcpdump is available)
# This will show VLAN-tagged traffic on the secondary interface
sudo tcpdump -i enp2s0 -nn vlan 100
```

## Step 8: Test Cross-Namespace Connectivity

Verify that VMs in different namespaces can communicate over the shared CUDN VLAN network:

```bash
# From production VM (192.168.100.10), test connectivity to development VM
ping 192.168.100.20

# Test web service connectivity between namespaces
curl 192.168.100.20:8080

# From development VM (192.168.100.20), test connectivity to production VM
ping 192.168.100.10
curl 192.168.100.10:8080
```


## Troubleshooting

For comprehensive OVS bridge and bridge mapping troubleshooting, including CUDN-specific verification steps, see the [OVS Bridge Verification Troubleshooting Guide](../troubleshooting/ovs-bridge-verification.md#clusteruserdefinednetwork-cudn-verification).

### Common Issues

1. **VLAN Traffic Not Reaching VM**:
   - Verify physical switch configuration supports VLAN tagging
   - Check bridge mapping configuration on worker nodes
   - Ensure VLAN ID matches network infrastructure
   - Verify custom OVS bridge (`br-vlan`) is created: `oc get nncp br-vlan-creation`
   - Confirm physical interface is attached to bridge

2. **IP Address Assignment Issues**:
   - Verify static IP is configured correctly in cloud-init networkData
   - Check that the interface name matches (enp2s0 for second interface)
   - Verify cloud-init logs inside the VM: `cat /var/log/cloud-init.log`

3. **Network Connectivity Problems**:
   - Check that the physical network supports the configured VLAN
   - Test connectivity from physical network to VLAN subnet
   - Verify physical switch port is in trunk mode

4. **Cross-Namespace Issues**:
   - Verify namespace selector in CUDN configuration
   - Check that both namespaces are listed in the values array
   - Confirm NADs are auto-created in target namespaces

5. **CUDN-Specific Issues**:
   - Verify `physicalNetworkName` in CUDN matches bridge mapping logical name (`localnet-vlan`)
   - Check CUDN created NADs in selected namespaces: `oc get network-attachment-definitions -A | grep cudn-localnet-vlan-100`
   - Verify IPAM mode is set correctly (`Disabled` for static IPs via cloud-init)

### Verification Commands

```bash
# Check ClusterUserDefinedNetwork status
oc get clusteruserdefinednetwork

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

