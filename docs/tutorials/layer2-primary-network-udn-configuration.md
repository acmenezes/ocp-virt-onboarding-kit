# Configuring Layer 2 Primary Networks with Namespaced UDNs in OpenShift 4.19

This tutorial shows you how to create a primary network using User Defined Networks (UDNs) with Layer 2 topology in OpenShift 4.19. Instead of using the cluster's default network, UDNs let you define custom IP ranges and provide complete network isolation for your applications within a specific namespace. This configuration handles east-west traffic between workloads within the cluster, not north-south traffic for external connectivity. This approach is ideal when your applications need direct control over IP addressing or when you want to separate workloads with custom networking requirements. The entire setup takes just 5 straightforward steps and works without requiring additional network attachment definitions.

## Step 1: Create UDN-Enabled Namespace

Create a namespace with the UDN primary network label to enable User Defined Network functionality. The special label `k8s.ovn.org/primary-user-defined-network` tells OpenShift that this namespace will use a custom primary network instead of the cluster default. The namespace must be created and labeled before any workload deployment happens. And it can't be changed once it's created.

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: udn-primary-vm-guests
  labels:
    k8s.ovn.org/primary-user-defined-network: ""
EOF
```

## Step 2: Configure the UserDefinedNetwork Resource

Define the primary UDN with Layer2 topology and custom subnet configuration. This resource specifies the network characteristics including IP range, IPAM lifecycle, and primary role designation that will override the default cluster networking for pods in the target namespace.

```bash
oc apply -f - <<EOF
apiVersion: k8s.ovn.org/v1
kind: UserDefinedNetwork
metadata:
  name: udn-primary
  namespace: udn-primary-vm-guests
spec:
  topology: Layer2
  layer2:
    role: Primary
    subnets: 
    - "192.168.100.0/24"
  ipam:
    lifecycle: Persistent
EOF
```

## Step 3: Deploy Workload with Primary UDN

Create a virtual machine that utilizes the primary UDN configuration and runs a sample web service for testing connectivity. The workload automatically receives a dynamically assigned IP address from the 192.168.100.0/24 subnet range via the UDN's IPAM and uses the UDN as its primary network interface instead of the cluster default network.

```bash
oc apply -f - <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: fedora-vm-with-udn
  namespace: udn-primary-vm-guests
  labels:
    app: fedora-web-vm
    network-type: udn-primary
spec:
  running: true
  dataVolumeTemplates:
  - metadata:
      name: fedora-udn-volume
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
        app: fedora-web-vm
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
        resources:
          requests:
            memory: 2Gi
            cpu: 1
      networks:
      - name: default
        pod: {}
      volumes:
      - name: datavolumedisk
        dataVolume:
          name: fedora-udn-volume
      - name: cloudinitdisk
        cloudInitNoCloud:
          userData: |
            #cloud-config
            user: fedora
            password: fedora
            chpasswd: { expire: False }
            packages:
            - nginx
            runcmd:
            - systemctl enable --now nginx
            - echo "<h1>Welcome to OpenShift Virtualization !!!</h1>" > /var/www/html/index.html
EOF
```

## Step 4: Expose VM as a Service

Create a Kubernetes Service to expose the VM's web server, making it accessible from other workloads within the cluster through the UDN network.

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: fedora-vm-web-service
  namespace: udn-primary-vm-guests
  labels:
    app: fedora-web-vm
spec:
  selector:
    app: fedora-web-vm
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
  type: ClusterIP
EOF
```

## Step 5: Verify Primary UDN Configuration and Service Connectivity

Validate that the primary UDN is functioning correctly by checking network status, service connectivity, and confirming the web service is accessible through the UDN network.

```bash
# Check UDN resource status
oc get userdefinednetwork -n udn-primary-vm-guests

# Verify workload network configuration
oc get pods -n udn-primary-vm-guests -o jsonpath="{.items[0].metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}" | jq

# Check the Service and test connectivity 
oc get service fedora-vm-web-service -n udn-primary-vm-guests

# Test service connectivity
oc debug node/$(oc get nodes -o name | head -1 | cut -d/ -f2) -- chroot /host curl http://fedora-vm-web-service.udn-primary-vm-guests.svc.cluster.local

# Alternative if node debug doesn't work with UDN: 
# oc run test --image=curlimages/curl --rm -i -n udn-primary-vm-guests -- curl http://fedora-vm-web-service.udn-primary-vm-guests.svc.cluster.local

# Test network connectivity from within the VM
virtctl console -n udn-primary-vm-guests fedora-vm-with-udn

# Inside the VM console, verify IP assignment and routing
ip addr show
ip route show

# Test nginx service locally within the VM
systemctl status nginx
curl localhost:80
curl 127.0.0.1:80

# Check what's actually in the nginx document root
cat /var/www/html/index.html

# Check if nginx is listening on port 80
ss -tulpn | grep :80

# If needed, restart nginx and test again
sudo systemctl restart nginx
curl localhost

# Note: The IP address is dynamically assigned from the 192.168.100.0/24 range
# The gateway IP (192.168.100.1) may not respond to ping in distributed router configurations
```

The primary UDN configuration is now complete. Workloads in the `udn-primary-vm-guests` namespace will use the custom 192.168.100.0/24 network as their primary interface, providing network isolation and custom IP addressing independent of the cluster default network.

This tutorial demonstrated how to deploy a VM with primary UDN networking, expose it as a service, and test connectivity using various debug pod approaches. Since ClusterIP services are only accessible from within the cluster, the debug pod methods shown above are essential for validating service connectivity in real-world scenarios.

## References

### OpenShift Documentation
- [OpenShift - Understanding Multiple Networks](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/multiple_networks/understanding-multiple-networks)
- [Configuring Primary Networks](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/multiple_networks/primary-networks)