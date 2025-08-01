# Configuring Layer 2 Primary Networks with Namespaced UDNs in OpenShift Virtualization

This tutorial shows you how to create a primary network using User Defined Networks (UDNs) with Layer 2 topology in OpenShift Virtualization. Instead of using the cluster's default network, UDNs let you define custom IP ranges and provide complete network isolation for applications within a specific namespace. This approach is ideal when applications need direct control over IP addressing or require custom networking requirements. The entire setup takes just 5 straightforward steps without requiring additional network attachment definitions.

Versions tested:
```
OCP 4.19
```

## Step 1: Create UDN-Enabled Namespace

Create a namespace with the UDN primary network label to enable User Defined Network functionality. The special label `k8s.ovn.org/primary-user-defined-network` tells OpenShift that this namespace will use a custom primary network instead of the cluster default. The namespace must be created and labeled before any workload deployment happens and cannot be changed once created.

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

Define the primary UDN with Layer2 topology and custom subnet configuration. This resource specifies the network characteristics including IP range, IPAM lifecycle, and primary role designation to override default cluster networking for pods in the namespace.

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

Create a virtual machine that utilizes the primary UDN configuration and runs a simple Python HTTP server for testing connectivity. The workload automatically receives an IP address from the 192.168.100.0/24 subnet via the UDN's IPAM and uses the UDN as its primary network interface.

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
            binding:
              name: l2bridge
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
            write_files:
            - path: /etc/resolv.conf
              content: |
                nameserver 172.30.0.10      # cluster DNS
                search udn-primary-vm-guests.svc.cluster.local svc.cluster.local
                options ndots:5            
            user: fedora
            password: fedora
            chpasswd: { expire: False }
            packages:
            - python3
            runcmd:
            - echo "<h1>Welcome to OpenShift Virtualization !!!</h1>" > /root/index.html
            - cd /root && nohup python3 -m http.server 80 > /dev/null 2>&1 &
EOF
```

## Step 4: Expose VM as a Service

Create a Kubernetes Service to expose the VM's Python HTTP server with a LoadBalancer.

Note: Cloud services like AWS automatically deploy a load balancer for this configuration. For bare metal deployments, consider using [MetalLB](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/ingress_and_load_balancing/load-balancing-with-metallb) as your load balancer.

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
  type: LoadBalancer
EOF
```

## Step 5: Verifying service availability

This test was performed using an AWS-based cluster with metal type instances that automatically deploys a load balancer and creates an external name. You can find the name in the external service IP:

```
oc get svc                                                                                                 
NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP                                                     
             PORT(S)        AGE                                                                                        
fedora-vm-web-service   LoadBalancer   172.30.72.140   aa32981df19ad4959ae9a82fb9990db9-1996929606.ca-central-1.elb.ama
zonaws.com   80:32291/TCP   45m   

```

Test the service using curl:
```
curl aa32981df19ad4959ae9a82fb9990db9-1996929606.ca-central-1.elb.amazonaws.com                            
<h1>Welcome to OpenShift Virtualization !!!</h1>    
```

## References

### OpenShift Documentation
- [OpenShift - Understanding Multiple Networks](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/multiple_networks/understanding-multiple-networks)
- [Configuring Primary Networks](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/multiple_networks/primary-networks)