# Configuring Localnet Secondary Networks with VLAN using NetworkAttachmentDefinition in OpenShift Virtualization

This tutorial demonstrates how to configure localnet secondary networks with VLAN tagging using NetworkAttachmentDefinition (NAD) in OpenShift Virtualization. This approach provides direct access to physical network infrastructure with VLAN segmentation, enabling VMs to communicate with specific VLAN-tagged networks while maintaining pod network connectivity. This is ideal for integrating with existing VLAN-based network infrastructure or when network segmentation is required.

## Prerequisites

- **NMState Operator**: Must be installed and running in the cluster
- **NMState CR**: The nmstate instance must be created after the operator is running
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

![SNO diagram](./img/ocp-setup-diagrams-OCP%20virt%202nd%20Net.png)

## Step 1: Create VM Namespace

Create a namespace for VMs that will use the localnet secondary network with VLAN configuration.

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: vm-guests-vlan
EOF
```

## Step 2: Configure Bridge on Extra NIC (Optional)

This step shows how to create an OVS bridge on a dedicated physical NIC for localnet traffic. Skip to **Step 2A** if you want to use the default `br-ex` bridge instead.

### Option A: Create Custom OVS Bridge on Extra NIC

If you have an extra physical network interface (e.g., `ens224`, `eth1`, `enp11s0f0np0`) dedicated for VM networks, create an OVS bridge on that interface.

**First, identify your physical interface:**
```bash
# List available network interfaces on worker nodes
oc debug node/<worker-node-name> -- chroot /host ip link show
```

**Create the OVS bridge and attach the physical interface:**
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
      - name: enp11s0f0np0
        description: Physical interface for VLAN localnet
        type: ethernet
        state: up
        ipv4:
          enabled: false
        ipv6:
          enabled: false
      - name: br-vlan
        description: OVS bridge for VLAN traffic
        type: ovs-bridge
        state: up
        bridge:
          port:
            - name: enp11s0f0np0
EOF
```

**Configuration Details**:
- `enp11s0f0np0`: Replace with your actual physical interface name
- `br-vlan`: Custom bridge name (you can use any name)
- `ipv4/ipv6: enabled: false`: Physical interface doesn't need IP addressing
- The physical interface is added as a port to the OVS bridge

**Verify the bridge creation:**

> **Important**: OVS bridges won't show up in `ip addr show` or `ip link show` commands. This is normal! OVS bridges exist in the OVS database, not as regular Linux network devices. Use `ovs-vsctl` commands to verify them.

```bash
# Check the NodeNetworkConfigurationPolicy status
oc get nncp br-vlan-creation

# List all OVS bridges (THIS IS THE CORRECT WAY)
oc debug node/<worker-node-name> -- chroot /host ovs-vsctl list-br

# Verify detailed OVS configuration
oc debug node/<worker-node-name> -- chroot /host ovs-vsctl show
```

**Expected output from `ovs-vsctl list-br`:**
```
br-ex
br-int
br-vlan
```

**Expected output from `ovs-vsctl show` (relevant section):**
```
Bridge br-vlan
    Port enp11s0f0np0
        Interface enp11s0f0np0
            type: system
```

If you encounter any issues verifying the bridge, see the **[OVS Bridge Verification Troubleshooting Guide](../troubleshooting/ovs-bridge-verification.md)** for detailed troubleshooting steps with example outputs.

### Option B: Use Default br-ex Bridge

If you want to use the existing `br-ex` bridge (default OpenShift network), skip the bridge creation and proceed directly to Step 2A.

## Step 2A: Configure Bridge Mapping

Configure the bridge mapping that connects the logical network interface name `localnet-vlan` to your OVS bridge. This mapping tells OVN-Kubernetes which physical bridge interface handles localnet traffic with VLAN support.

**For custom bridge (br-vlan):**
```bash
oc apply -f - <<EOF
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: localnet-vlan-bridge-mapping
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

**For default bridge (br-ex):**
```bash
oc apply -f - <<EOF
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: localnet-vlan-bridge-mapping
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

**Configuration Details**:
- `localnet: localnet-vlan`: Logical network interface name (referenced later in NAD)
- `bridge: br-vlan` or `bridge: br-ex`: Physical bridge name
- This creates the mapping: `localnet-vlan` → `br-vlan` (or `br-ex`)

**Verify the bridge mapping:**
```bash
# Check the mapping configuration
oc get nncp localnet-vlan-bridge-mapping -o yaml

# Verify OVN bridge mappings on worker nodes
oc debug node/<worker-node-name> -- chroot /host ovs-vsctl get Open_vSwitch . external_ids:ovn-bridge-mappings
```

## Step 3: Create NetworkAttachmentDefinition with VLAN

Create a NetworkAttachmentDefinition that configures localnet topology with VLAN tagging for secondary network connectivity.

```bash
oc apply -f - <<EOF
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: localnet-vlan-100
  namespace: vm-guests-vlan
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "localnet-vlan-100",
    "type": "ovn-k8s-cni-overlay",
    "topology": "localnet",
    "netAttachDefName": "vm-guests-vlan/localnet-vlan-100",
    "physicalNetworkName": "localnet-vlan",
    "vlanID": 100
  }'
EOF
```

**Configuration Details**:
- `physicalNetworkName: "localnet-vlan"`: **CRITICAL** - Must match the logical network name from Step 2A bridge mapping
- `vlanID: 100`: Specifies VLAN tag 100 for network segmentation
- `topology: "localnet"`: Uses localnet topology for direct physical network access
- **No IP management parameters**: Localnet topology with NAD does not support built-in IP address management. IP configuration must be handled entirely within the VM using cloud-init static configuration or external DHCP server on the VLAN network

**Important Notes**:
- The `physicalNetworkName` must match the `localnet` name configured in your bridge mapping (Step 2A)
- **IP Address Assignment**: With localnet topology, configure static IPs via cloud-init's `networkData` or rely on an external DHCP server running on the physical VLAN network
- OVN-Kubernetes does not provide IP address management for localnet secondary networks attached via NAD

## Step 4: Deploy VM with VLAN Network Connectivity

Create a VM that connects to both the pod network and the VLAN-tagged localnet secondary network.

```bash
oc apply -f - <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: fedora-vlan-vm
  namespace: vm-guests-vlan
  labels:
    app: fedora-vlan-vm
spec:
  runStrategy: Always
  dataVolumeTemplates:
  - metadata:
      name: fedora-vlan-volume
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
        app: fedora-vlan-vm
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
          - name: vlan_network
            bridge: {}
        resources:
          requests:
            memory: 2Gi
            cpu: 1
      networks:
      - name: default
        pod: {}
      - name: vlan_network
        multus:
          networkName: localnet-vlan-100
      volumes:
      - name: datavolumedisk
        dataVolume:
          name: fedora-vlan-volume
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
                <h1>Welcome to OpenShift Virtualization VLAN Network!</h1>
                <p>VLAN ID: 100</p>
                <p>Static IP: 192.168.100.10/24</p>
                <p>Gateway: 192.168.100.1</p>
              permissions: '0644'
            runcmd:
            - nohup python3 -m http.server 8080 --directory /root > /dev/null 2>&1 &
EOF
```

**Cloud-init Network Configuration Explained**:
- **Primary network (enp1s0)**: Not configured in `networkData` - defaults to DHCP from OpenShift pod network
- **enp2s0** (VLAN network/secondary): Configured with static IP `192.168.100.10/24`
  - **Critical**: Must use the actual interface name (`enp2s0`) not generic names like `eth0`, `eth1`
  - Uses compact YAML array syntax: `addresses: ["192.168.100.10/24"]`
  - Only the secondary interface needs explicit configuration in `networkData` for static IP assignment
  - Primary interface automatically gets DHCP from the pod network without explicit configuration

**Note**: KubeVirt VMs use predictable network interface names (e.g., `enp1s0`, `enp2s0`) based on PCI slot positions, not traditional `eth0`/`eth1` names. Always verify interface names match your VM's actual configuration.

## Step 5: Verify VLAN Network Connectivity

Access the VM console to verify dual network connectivity with VLAN configuration.

```bash
# Check the VM status
oc get vm -n vm-guests-vlan

# Check the running VM instance
oc get vmi -n vm-guests-vlan

# Access the VM console (credentials: fedora/fedora via cloud-init)
virtctl console -n vm-guests-vlan fedora-vlan-vm
```

Inside the VM console, verify the VLAN network configuration with static IP:

```bash
# Check network interfaces - should see static IP on eth1 (VLAN interface)
ip addr show

# Expected output:
# eth0 (enp1s0): pod network with DHCP IP (e.g., 10.128.0.172)
# eth1 (enp2s0): VLAN network with static IP 192.168.100.10

# Check routes - only primary network has default route
ip route show

# Expected output example:
default via 10.128.0.1 dev enp1s0 proto dhcp src 10.128.0.172 metric 100 
10.128.0.0/14 dev enp1s0 proto kernel scope link src 10.128.0.172 metric 100
192.168.100.0/24 dev enp2s0 proto kernel scope link src 192.168.100.10

# Verify static IP is configured on VLAN interface
ip addr show enp2s0 | grep 192.168.100.10

# Test external connectivity (uses primary network metric 100)
ping -c 3 8.8.8.8

# Verify the web service is running
curl localhost:8080

# Test VLAN interface connectivity from another machine on the same VLAN
# From another VM or physical machine: curl 192.168.100.10:8080

# Test connectivity using specific interface
ping -I enp2s0 192.168.100.1

# Verify VLAN tagging (if tcpdump is available)
# This will show VLAN-tagged traffic on the secondary interface
sudo tcpdump -i enp2s0 -nn vlan 100
```

## Troubleshooting

### Common Issues

1. **VLAN Traffic Not Reaching VM**:
   - Verify physical switch configuration supports VLAN tagging
   - Check bridge mapping configuration on worker nodes
   - Ensure VLAN ID matches network infrastructure
   - Verify the physical NIC is connected and UP
   - Check that the OVS bridge includes the physical interface as a port

2. **IP Address Assignment Issues**:
   - Verify DHCP server is available on the VLAN network

3. **Network Connectivity Problems**:
   - Check that the physical network supports the configured VLAN
   - Test connectivity from physical network to VLAN subnet
   - Verify the physical switch port is in trunk mode

4. **Custom Bridge Creation Issues**:
   - See the **[OVS Bridge Verification Troubleshooting Guide](../troubleshooting/ovs-bridge-verification.md)** for detailed troubleshooting
   - Remember: OVS bridges won't show in `ip addr show` - use `ovs-vsctl list-br` instead
   - Ensure the physical interface name is correct
   - Verify NMState operator is running properly

5. **Bridge Mapping Not Working**:
   - Verify the bridge name in the mapping matches the created bridge
   - Check OVN bridge mappings: `ovs-vsctl get Open_vSwitch . external_ids:ovn-bridge-mappings`
   - Ensure the localnet name in the mapping matches the NAD configuration

6. **VM Pod Fails with "failed bridge mapping validation" Error**:
   - **Symptom**: VM pod fails with error: `failed to find OVN bridge-mapping for network: "localnet-vlan-100"`
   - **Cause**: Missing `physicalNetworkName` parameter in NetworkAttachmentDefinition
   - **Solution**: Ensure NAD includes `"physicalNetworkName": "localnet-vlan"` that matches the bridge mapping logical name from Step 2A
   - **Verification**: After updating NAD, recreate it with `oc delete` and `oc apply`

### Verification Commands

```bash
# Check NetworkAttachmentDefinition status
oc get network-attachment-definitions -n vm-guests-vlan

# Verify all NodeNetworkConfigurationPolicies
oc get nncp

# Check bridge creation status (if using custom bridge)
oc get nncp br-vlan-creation -o yaml

# Verify bridge mapping on worker nodes
oc get nncp localnet-vlan-bridge-mapping -o yaml

# Check physical interface status on worker node
oc debug node/<worker-node-name> -- chroot /host ip link show ens224

# Verify OVS bridge configuration
oc debug node/<worker-node-name> -- chroot /host ovs-vsctl show

# Check OVN bridge mappings
oc debug node/<worker-node-name> -- chroot /host ovs-vsctl get Open_vSwitch . external_ids:ovn-bridge-mappings

# List all ports on the custom bridge
oc debug node/<worker-node-name> -- chroot /host ovs-vsctl list-ports br-vlan

# Check VM network status
oc get vmi -n vm-guests-vlan -o yaml | grep -A 10 networks

# View OVN-Kubernetes logs for troubleshooting
oc logs -n openshift-ovn-kubernetes -l app=ovnkube-node

# Check NMState operator logs if bridge creation fails
oc logs -n openshift-nmstate -l app=kubernetes-nmstate-operator
```

## References

### Troubleshooting Guides
- **[OVS Bridge Verification Troubleshooting Guide](../troubleshooting/ovs-bridge-verification.md)** - Comprehensive guide for verifying and troubleshooting OVS bridge creation

### OpenShift Documentation
- [OpenShift - Understanding Multiple Networks](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/multiple_networks/understanding-multiple-networks)
- [Localnet Topology Configuration for OCP Virtualization](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/networking#virt-creating-secondary-localnet-udn_virt-connecting-vm-to-secondary-udn)
- [NMState Operator - Managing Node Network Configuration](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/networking_operators/k8s-nmstate-about-the-k8s-nmstate-operator)
- [OVS Bridge Configuration with NMState](https://nmstate.io/examples.html#interfaces-ovs-bridge)

