# Configuring Localnet Secondary Networks with User Defined Networks in OpenShift Virtualization

This tutorial shows you how to create a localnet secondary network using UserDefinedNetwork resources in OpenShift Virtualization. Localnet topology provides direct access to the underlying physical network infrastructure without VLANs, enabling virtual machines to communicate directly with external networks and services while maintaining their primary pod network connectivity. This approach is ideal when you need VMs to have direct access to existing network infrastructure or when integrating with legacy systems that require layer 2 connectivity. The entire setup takes just 4 straightforward steps and provides seamless integration with your existing network infrastructure.

Versions tested:
```
OCP 4.19
```

## Step 1: Create VM Namespace

Create a dedicated namespace for hosting virtual machines that will use the localnet secondary network configuration. This namespace will contain all the VM workloads and network resources needed for localnet connectivity.

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: vm-guests-localnet
EOF
```

## Step 2: Create UserDefinedNetwork with Localnet Topology

Define the localnet UserDefinedNetwork that provides secondary network connectivity to VMs. This configuration creates a localnet topology that enables direct access to external physical networks while maintaining isolation from other networks.

```bash
oc apply -f - <<EOF
apiVersion: k8s.ovn.org/v1
kind: UserDefinedNetwork
metadata:
  name: localnet-secondary
  namespace: vm-guests-localnet
spec:
  topology: Localnet
  localnet:
    role: Secondary
  namespaceSelector:
    matchLabels:
      name: vm-guests-localnet
EOF
```

## Step 3: Deploy VM with Localnet Secondary Network

Create a virtual machine that connects to both the default pod network and the localnet secondary network. The VM will have dual connectivity - pod network for Kubernetes services and localnet for direct external network access.

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
  running: true
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
      annotations:
        k8s.v1.cni.cncf.io/networks: vm-guests-localnet/localnet-secondary
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
          - name: localnet
            bridge: {}
        resources:
          requests:
            memory: 2Gi
            cpu: 1
      networks:
      - name: default
        pod: {}
      - name: localnet
        multus:
          networkName: vm-guests-localnet/localnet-secondary
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

## Step 4: Verify Localnet Secondary Network Connectivity

Test the localnet configuration by accessing the VM console and verifying dual network connectivity. The VM should have access to both the pod network for Kubernetes services and the localnet for direct external network access.

```bash
# Check the VM status
oc get vm -n vm-guests-localnet

# Check the running VM instance
oc get vmi -n vm-guests-localnet

# Access the VM console
virtctl console -n vm-guests-localnet fedora-localnet-vm
```

Inside the VM console, verify the network configuration:

```bash
# Check network interfaces and IP addresses
ip addr show

# Check routes - should show external gateway
ip route show

# Test external connectivity
ping 8.8.8.8

# Test DNS resolution
nslookup google.com

# Verify the web service is running
curl localhost:8080

# Check connectivity on both interfaces
# eth0 should be the pod network interface
# eth1 should be the localnet interface

# Verify pod network connectivity (Kubernetes services)
curl kubernetes.default.svc.cluster.local

# Test localnet interface connectivity to external networks
# Replace <localnet-ip> with the actual IP assigned to eth1
curl <localnet-ip>:8080
```

To test external access to the VM's web service, you can access it using:
- The pod network through Kubernetes Services (internal cluster access)
- The localnet interface IP directly from external networks on the same physical segment

## References

### OpenShift Documentation
- [OpenShift - Understanding Multiple Networks](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/multiple_networks/understanding-multiple-networks)
- [Configuring Additional Networks](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/multiple_networks/configuring-additional-network-types)
- [UserDefinedNetwork CR](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/multiple_networks/primary_networks/about-user-defined-networks)
- [Creating Secondary Networks on OVN-Kubernetes](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/multiple_networks/secondary_networks/creating-secondary-nwt-ovnk)
- [OpenShift Virtualization Networking](https://docs.redhat.com/en/documentation/openshift_virtualization/4.19/html/networking/index)

### Kubernetes APIs
- [NetworkAttachmentDefinition API](https://github.com/k8snetworkplumbingwg/network-attachment-definition-client)
- [KubeVirt VirtualMachine API](https://kubevirt.io/api-reference/main/definitions.html#_v1_virtualmachine)