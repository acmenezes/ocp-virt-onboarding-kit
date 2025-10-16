# Configuring Localnet Secondary Networks with VLAN using NetworkAttachmentDefinition in OpenShift Virtualization

This tutorial demonstrates how to configure localnet secondary networks with VLAN tagging using NetworkAttachmentDefinition (NAD) in OpenShift Virtualization. This approach provides direct access to physical network infrastructure with VLAN segmentation, enabling VMs to communicate with specific VLAN-tagged networks while maintaining pod network connectivity. This is ideal for integrating with existing VLAN-based network infrastructure or when network segmentation is required.

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

## Step 2: Configure Bridge Mapping

Configure the bridge mapping that connects the logical network interface name `localnet-vlan` to the `br-ex` bridge interface on worker nodes. This mapping tells OVN-Kubernetes which physical bridge interface handles localnet traffic with VLAN support, enabling VM access to VLAN-tagged external networks through the underlying physical network infrastructure.

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
    "vlanID": 100,
    "subnets": "192.168.100.0/24",
    "excludeSubnets": "192.168.100.1/32"
  }'
EOF
```

**Configuration Details**:
- `vlanID: 100`: Specifies VLAN tag 100 for network segmentation
- `subnets`: Defines the expected IP subnet range for documentation and validation purposes
- `excludeSubnets`: Excludes gateway IP from being assigned (typically reserved for router/gateway)
- `topology: "localnet"`: Uses localnet topology for direct physical network access
- **No IPAM configuration**: IP addresses require external DHCP server on the VLAN network or VMs need to be configured statically

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
            - echo "<h1>Welcome to OpenShift Virtualization VLAN Network!</h1>" > /root/index.html
            - echo "<p>VLAN ID: 100</p>" >> /root/index.html
            - echo "<p>Network: 192.168.100.0/24</p>" >> /root/index.html
            - cd /root && nohup python3 -m http.server 8080 > /dev/null 2>&1 &
EOF
```

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

Inside the VM console, verify the VLAN network configuration:

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

### Verification Commands

```bash
# Check NetworkAttachmentDefinition status
oc get network-attachment-definitions -n vm-guests-vlan

# Verify bridge mapping on worker nodes
oc get nncp localnet-vlan-bridge-mapping -o yaml

# Check VM network status
oc get vmi -n vm-guests-vlan -o yaml | grep -A 10 networks

# View OVN-Kubernetes logs for troubleshooting
oc logs -n openshift-ovn-kubernetes -l app=ovnkube-node
```

## References

### OpenShift Documentation
- [OpenShift - Understanding Multiple Networks](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/multiple_networks/understanding-multiple-networks)
- [Localnet Topology Configuration for OCP Virtualization](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/networking#virt-creating-secondary-localnet-udn_virt-connecting-vm-to-secondary-udn)

