# Configuring Localnet Secondary Networks with ClusterUserDefinedNetwork in OpenShift Virtualization

This tutorial demonstrates how to configure localnet secondary networks using ClusterUserDefinedNetwork (CUDN) in OpenShift Virtualization. Localnet topology provides direct access to physical network infrastructure, enabling VMs to communicate with external networks while maintaining pod network connectivity. This approach is ideal for integrating with existing infrastructure or legacy systems requiring layer 2 connectivity.

## Prerequisites

- **NMState Operator**: Must be installed and running in the cluster
- **NMState CR**: The nmstate instance must be created after the operator is running
- **Worker nodes**: The bridge mapping will be applied to worker nodes

Versions tested:
```
OCP 4.19
```

## Step 1: Create VM Namespace

Create a namespace for VMs that will use the localnet secondary network.

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: vm-guests-localnet
EOF
```

## Step 2: Configure Bridge Mapping

Configure the bridge mapping that connects the logical physnet name to the `br-ex` bridge interface. This tells OVN-Kubernetes which physical interface handles localnet traffic.

```bash
oc apply -f - <<EOF
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: localnet-bridge-mapping
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ''  
  desiredState:
    ovn:
      bridge-mappings:
      - localnet: localnet1
        bridge: br-ex
        state: present
EOF
```

**Note**: This maps the logical `localnet1` name to the `br-ex` bridge on worker nodes, enabling VM access to external networks.

## Step 3: Create ClusterUserDefinedNetwork

Create a ClusterUserDefinedNetwork with localnet topology for secondary network connectivity to external networks.

```bash
oc apply -f - <<EOF
apiVersion: k8s.ovn.org/v1
kind: ClusterUserDefinedNetwork
metadata:
  name: cudn-localnet
spec:
  namespaceSelector: 
    matchExpressions: 
    - key: kubernetes.io/metadata.name
      operator: In 
      values: ["vm-guests-localnet"]
  network:
    topology: Localnet 
    localnet:
        role: Secondary 
        physicalNetworkName: localnet1 
        ipam:
          mode: Disabled
EOF
```

## Step 4: Deploy VM with Dual Network Connectivity

Create a VM that connects to both the pod network and the localnet secondary network for dual connectivity.

```bash
oc apply -f - <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: fedora-localnet-vm
  namespace: vm-guests-localnet
  labels:
    app: fedora-localnet-vm
spec:
  runStrategy: Always
  dataVolumeTemplates:
  - metadata:
      name: fedora-localnet-volume
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
        app: fedora-localnet-vm
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
          - name: secondary_localnet
            bridge: {}
        resources:
          requests:
            memory: 2Gi
            cpu: 1
      networks:
      - name: default
        pod: {}
      - name: secondary_localnet
        multus:
          networkName: cudn-localnet
      volumes:
      - name: datavolumedisk
        dataVolume:
          name: fedora-localnet-volume
      - name: cloudinitdisk
        cloudInitNoCloud:
          userData: |
            #cloud-config
            user: fedora
            password: fedora
            chpasswd: { expire: False }
            packages:
            - python3
            runcmd:
            - echo "<h1>Welcome to OpenShift Virtualization Localnet!</h1>" > /root/index.html
            - cd /root && nohup python3 -m http.server 8080 > /dev/null 2>&1 &
EOF
```

## Step 5: Verify Network Connectivity

Access the VM console to verify dual network connectivity.

```bash
# Check the VM status
oc get vm -n vm-guests-localnet

# Check the running VM instance
oc get vmi -n vm-guests-localnet

# Access the VM console (credentials: fedora/fedora via cloud-init)
virtctl console -n vm-guests-localnet fedora-localnet-vm
```

Inside the VM console, verify the network configuration:

```bash
# Check network interfaces - primary (pod network) and secondary (localnet)
ip addr show

# Check routes - two default gateways (lower metric prioritized)
# Primary network: metric 100, Secondary: metric 101
ip route show

# Expected output example:

default via 10.128.0.1 dev enp1s0 proto dhcp src 10.128.0.172 metric 100 
default via 192.168.2.1 dev enp2s0 proto dhcp src 192.168.2.128 metric 101 

# Test external connectivity
ping 8.8.8.8

# Verify the web service is running
curl localhost:8080

# Test localnet interface connectivity (replace <localnet-ip> with actual IP)
curl <localnet-ip>:8080
```
## References

### OpenShift Documentation
- [OpenShift - Understanding Multiple Networks](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/multiple_networks/understanding-multiple-networks)
- [Localnet Topology Configuration for OCP Virtualization](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/virtualization/networking#virt-creating-secondary-localnet-udn_virt-connecting-vm-to-secondary-udn)