# VM Remote Access Troubleshooting Guide

This guide provides comprehensive troubleshooting steps, verification commands, and solutions for issues related to remote access to Linux VMs in OpenShift Virtualization.

## Table of Contents
- [Quick Health Checks](#quick-health-checks)
- [Method 1: Console Access](#method-1-console-access)
- [Method 2: VNC Access](#method-2-vnc-access)
- [Method 3: SSH via virtctl](#method-3-ssh-via-virtctl)
- [Method 4: SSH via NodePort Service](#method-4-ssh-via-nodeport-service)
- [Method 5: SSH via LoadBalancer](#method-5-ssh-via-loadbalancer)
- [Method 6: SSH via Port Forward](#method-6-ssh-via-port-forward)
- [Method 7: Direct Access via Secondary Network](#method-7-direct-access-via-secondary-network)
- [Diagnostic Commands](#diagnostic-commands)

## Quick Health Checks

### Check VM Status
```bash
oc get vm <vm-name> -n <namespace>
oc get vmi <vm-name> -n <namespace>
```

Expected output for running VM:
```
NAME        AGE   STATUS    READY
my-vm       5m    Running   True
```

### Check VM Has IP Address
```bash
oc get vmi <vm-name> -n <namespace> -o jsonpath='{.status.interfaces[*].ipAddress}'
```

Should return at least one IP address.

### Check VM Pod is Running
```bash
oc get pods -n <namespace> -l kubevirt.io/domain=<vm-name>
```

Expected output:
```
NAME                       READY   STATUS    RESTARTS   AGE
virt-launcher-<vm>-xxxxx   3/3     Running   0          5m
```

### Check Guest Agent Status
```bash
oc get vmi <vm-name> -n <namespace> -o jsonpath='{.status.guestOSInfo.id}'
```

If empty, guest agent is not running (required for some access methods).

## Method 1: Console Access

### Issue: Cannot Connect to Console

**Symptoms**:
```bash
$ virtctl console my-vm -n my-namespace
Error: unable to connect to console
```

**Diagnosis**:
```bash
# Check VM is running
oc get vm my-vm -n my-namespace

# Check VMI exists
oc get vmi my-vm -n my-namespace

# Check virt-launcher pod
oc get pods -n my-namespace -l kubevirt.io/domain=my-vm
```

**Common Causes**:
- VM is not running
- VMI not created
- virt-launcher pod not running
- Network issues between client and API server

**Solutions**:
1. Start VM if stopped:
   ```bash
   virtctl start my-vm -n my-namespace
   ```

2. Wait for VMI to be ready:
   ```bash
   oc wait --for=condition=Ready vmi/my-vm -n my-namespace --timeout=60s
   ```

3. Check virt-launcher pod logs:
   ```bash
   oc logs -n my-namespace -l kubevirt.io/domain=my-vm
   ```

4. Verify virtctl version matches cluster:
   ```bash
   virtctl version
   oc get kubevirt -n openshift-cnv -o jsonpath='{.items[0].status.observedKubeVirtVersion}'
   ```

### Issue: Console Connected But No Login Prompt

**Symptoms**:
Console connects but shows blank screen or no prompt.

**Diagnosis**:
```bash
# Try sending Enter key
# Press Enter multiple times
```

**Common Causes**:
- VM still booting
- Serial console not configured in VM
- systemd-getty not running on ttyS0

**Solutions**:
1. Wait for VM to fully boot (check VMI phase):
   ```bash
   oc get vmi my-vm -n my-namespace -o jsonpath='{.status.phase}'
   ```

2. Check guest OS has serial console enabled:
   - For RHEL/Fedora: `systemctl status serial-getty@ttyS0.service`
   - Cloud images typically have this enabled by default

3. Try disconnect and reconnect:
   ```bash
   # Press Ctrl+] to disconnect
   virtctl console my-vm -n my-namespace
   ```

### Issue: Cannot Login at Console

**Symptoms**:
Login prompt appears but credentials don't work.

**Common Causes**:
- Wrong username/password
- Cloud-init SSH-only access configured
- Root login disabled

**Solutions**:
1. For cloud images, use cloud-init to set password:
   ```yaml
   cloudInitNoCloud:
     userData: |
       #cloud-config
       user: fedora
       password: mypassword
       chpasswd:
         expire: false
   ```

2. Or use SSH key authentication and access via console with key

3. Check cloud-init logs inside VM for user creation errors

## Method 2: VNC Access

### Issue: virtctl vnc Command Not Working

**Symptoms**:
```bash
$ virtctl vnc my-vm -n my-namespace
Error: VNC is not configured for this VM
```

**Diagnosis**:
```bash
# Check VM has VNC graphics device
oc get vm my-vm -n my-namespace -o yaml | grep -A 5 devices
```

**Common Causes**:
- No graphics device configured in VM spec
- Graphics device type is not VNC

**Solutions**:
1. Add VNC graphics device to VM:
   ```yaml
   spec:
     template:
       spec:
         domain:
           devices:
             inputs:
               - type: tablet
                 bus: usb
                 name: tablet
   ```

2. Ensure graphics is enabled (default for most VMs)

### Issue: VNC Viewer Cannot Connect

**Symptoms**:
virtctl vnc command starts but VNC viewer cannot connect.

**Diagnosis**:
```bash
# Check virtctl output for port number
virtctl vnc my-vm -n my-namespace
# Shows: "VNC connection on port 5900"

# Check if port is listening
netstat -tuln | grep 5900
```

**Common Causes**:
- Firewall blocking VNC port
- VNC viewer not installed or wrong viewer
- Port conflict

**Solutions**:
1. Install VNC viewer:
   ```bash
   # Linux
   sudo dnf install tigervnc

   # macOS
   brew install tiger-vnc
   ```

2. Connect to localhost:5900 or 127.0.0.1:5900

3. If port conflict, virtctl will use next available port (5901, 5902, etc.)

### Issue: VNC Connected But Blank Screen

**Symptoms**:
VNC connects but shows blank or black screen.

**Common Causes**:
- VM still booting
- No display manager running
- VM is console-only (no GUI)

**Solutions**:
1. Wait for VM to fully boot

2. Check if VM has GUI installed:
   ```bash
   # Inside VM
   systemctl status gdm  # For GNOME
   systemctl status lightdm  # For other DEs
   ```

3. For VMs without GUI, use console access instead

## Method 3: SSH via virtctl

### Issue: virtctl ssh Command Fails

**Symptoms**:
```bash
$ virtctl ssh fedora@my-vm -n my-namespace
Error: SSH is not available for this VM
```

**Diagnosis**:
```bash
# Check guest agent is running
oc get vmi my-vm -n my-namespace -o jsonpath='{.status.guestOSInfo}'

# Check VM interfaces
oc get vmi my-vm -n my-namespace -o jsonpath='{.status.interfaces}'
```

**Common Causes**:
- Guest agent not installed/running
- SSH service not running in VM
- No IP address assigned to VM
- virtctl-ssh-proxy not available

**Solutions**:
1. Install guest agent in VM:
   ```bash
   # Inside VM
   sudo dnf install qemu-guest-agent  # Fedora/RHEL
   sudo systemctl enable --now qemu-guest-agent
   ```

2. Install and enable SSH in VM:
   ```bash
   # Inside VM
   sudo dnf install openssh-server
   sudo systemctl enable --now sshd
   ```

3. Wait for guest agent to report IP:
   ```bash
   oc get vmi my-vm -n my-namespace -o jsonpath='{.status.interfaces[0].ipAddress}'
   ```

### Issue: virtctl ssh Connects But Authentication Fails

**Symptoms**:
```bash
$ virtctl ssh fedora@my-vm -n my-namespace
fedora@my-vm: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```

**Diagnosis**:
```bash
# Check SSH keys
ls -la ~/.ssh/

# Try with password authentication
virtctl ssh --local-ssh-opts="-o PreferredAuthentications=password" fedora@my-vm -n my-namespace
```

**Common Causes**:
- SSH public key not in VM's authorized_keys
- Wrong username
- SSH key passphrase required

**Solutions**:
1. Add SSH key via cloud-init:
   ```yaml
   cloudInitNoCloud:
     userData: |
       #cloud-config
       ssh_authorized_keys:
         - ssh-rsa AAAAB3NzaC1yc2EA... user@host
   ```

2. Or manually add key inside VM:
   ```bash
   mkdir -p ~/.ssh
   echo "ssh-rsa AAAAB3..." >> ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   ```

3. Check username - default for Fedora is `fedora`, RHEL is `cloud-user` or `ec2-user`

### Issue: virtctl ssh Very Slow

**Symptoms**:
Connection takes 30+ seconds to establish.

**Common Causes**:
- DNS reverse lookup timeout
- GSSAPI authentication attempt

**Solutions**:
Add SSH options to skip slow authentication methods:
```bash
virtctl ssh --local-ssh-opts="-o GSSAPIAuthentication=no -o UseDNS=no" fedora@my-vm -n my-namespace
```

## Method 4: SSH via NodePort Service

### Issue: NodePort Service Not Accessible

**Symptoms**:
```bash
$ ssh -p 30022 fedora@<node-ip>
ssh: connect to host <node-ip> port 30022: Connection refused
```

**Diagnosis**:
```bash
# Check service exists
oc get svc -n my-namespace

# Check service endpoints
oc get endpoints my-vm-ssh -n my-namespace

# Check NodePort assigned
oc get svc my-vm-ssh -n my-namespace -o jsonpath='{.spec.ports[0].nodePort}'
```

**Common Causes**:
- Service not created
- Wrong NodePort number
- VM pod not running or not ready
- Firewall blocking NodePort
- SSH not listening in VM

**Solutions**:
1. Verify Service configuration:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: my-vm-ssh
   spec:
     type: NodePort
     selector:
       kubevirt.io/domain: my-vm
     ports:
       - protocol: TCP
         port: 22
         targetPort: 22
         nodePort: 30022  # Optional, will be auto-assigned if omitted
   ```

2. Check selector matches VM:
   ```bash
   oc get vm my-vm -n my-namespace -o jsonpath='{.spec.template.metadata.labels}'
   ```

3. Verify SSH is running in VM:
   ```bash
   virtctl console my-vm -n my-namespace
   # Inside VM:
   systemctl status sshd
   ```

4. Check firewall rules on OpenShift nodes:
   ```bash
   # On node
   sudo firewall-cmd --list-ports
   sudo firewall-cmd --add-port=30022/tcp
   ```

### Issue: NodePort Works But Wrong VM

**Symptoms**:
SSH connects but to wrong VM or gets connection refused.

**Diagnosis**:
```bash
# Check service endpoints
oc describe svc my-vm-ssh -n my-namespace

# Look at Endpoints field - should show VM pod IP
```

**Common Causes**:
- Service selector matches multiple VMs
- Service selector doesn't match any VM

**Solutions**:
1. Use unique labels for service selector:
   ```yaml
   selector:
     kubevirt.io/domain: my-vm  # Unique per VM
     # OR
     app: my-specific-vm-app
   ```

2. Verify only one endpoint:
   ```bash
   oc get endpoints my-vm-ssh -n my-namespace -o yaml
   ```

### Issue: Cannot Determine Node IP

**Symptoms**:
Don't know which node IP to use for NodePort access.

**Solutions**:
```bash
# Get node IPs
oc get nodes -o wide

# Use any node's INTERNAL-IP or EXTERNAL-IP (if available)

# Or get specific node where VM is running
VM_NODE=$(oc get vmi my-vm -n my-namespace -o jsonpath='{.status.nodeName}')
oc get node $VM_NODE -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}'
```

## Method 5: SSH via LoadBalancer

### Issue: LoadBalancer Service Has No External IP

**Symptoms**:
```bash
$ oc get svc my-vm-ssh -n my-namespace
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
my-vm-ssh    LoadBalancer   172.30.x.x      <pending>     22:30022/TCP
```

**Diagnosis**:
```bash
# Check if LoadBalancer provider is available
oc get svc -A | grep LoadBalancer

# Check cluster has LoadBalancer support
oc describe svc my-vm-ssh -n my-namespace
```

**Common Causes**:
- No LoadBalancer provider configured (MetalLB, cloud provider)
- LoadBalancer quota exhausted
- LoadBalancer configuration error

**Solutions**:
1. For on-premise clusters, install [MetalLB](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/ingress_and_load_balancing/load-balancing-with-metallb).

2. For cloud deployments, ensure cloud provider integration is configured

3. Check LoadBalancer events:
   ```bash
   oc describe svc my-vm-ssh -n my-namespace
   ```

4. Alternative: Use NodePort instead of LoadBalancer

### Issue: LoadBalancer IP Not Reachable

**Symptoms**:
External IP assigned but SSH connection times out.

**Diagnosis**:
```bash
# Check LoadBalancer IP
LB_IP=$(oc get svc my-vm-ssh -n my-namespace -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Try ping
ping -c 3 $LB_IP

# Check from within cluster
oc run test-curl --image=curlimages/curl --rm -it --restart=Never -- curl -v telnet://$LB_IP:22
```

**Common Causes**:
- LoadBalancer IP not routable
- Firewall blocking traffic
- LoadBalancer misconfiguration
- SSH not running in VM

**Solutions**:
1. Verify LoadBalancer IP pool is correct for your network

2. Check firewall allows traffic to LoadBalancer IP

3. Verify SSH is running in VM (see Method 4 solutions)

## Method 6: SSH via Port Forward

### Issue: oc port-forward Fails to Start

**Symptoms**:
```bash
$ oc port-forward vmi/my-vm -n my-namespace 2222:22
Error: unable to forward port
```

**Diagnosis**:
```bash
# Check VMI exists
oc get vmi my-vm -n my-namespace

# Check local port not in use
netstat -tuln | grep 2222
lsof -i :2222
```

**Common Causes**:
- Local port already in use
- VMI not running
- Network policy blocking port-forward

**Solutions**:
1. Use different local port:
   ```bash
   oc port-forward vmi/my-vm -n my-namespace 12345:22
   ```

2. Kill process using the port:
   ```bash
   # Find process
   lsof -ti :2222 | xargs kill -9
   ```

3. Wait for VMI to be ready:
   ```bash
   oc wait --for=condition=Ready vmi/my-vm -n my-namespace
   ```

### Issue: Port Forward Established But SSH Fails

**Symptoms**:
```bash
$ oc port-forward vmi/my-vm -n my-namespace 2222:22 &
Forwarding from 127.0.0.1:2222 -> 22

$ ssh -p 2222 fedora@localhost
ssh: connect to host localhost port 2222: Connection refused
```

**Diagnosis**:
```bash
# Check SSH is listening in VM
virtctl console my-vm -n my-namespace
# Inside VM:
ss -tuln | grep :22
```

**Common Causes**:
- SSH not running in VM
- SSH listening on different port
- VM doesn't have SSH installed

**Solutions**:
1. Install and start SSH in VM:
   ```bash
   # Inside VM via console
   sudo dnf install openssh-server
   sudo systemctl enable --now sshd
   ```

2. Check SSH port:
   ```bash
   # Inside VM
   sudo ss -tuln | grep ssh
   ```

3. Check SSH configuration:
   ```bash
   # Inside VM
   sudo cat /etc/ssh/sshd_config | grep ^Port
   ```

### Issue: Port Forward Disconnects Frequently

**Symptoms**:
Port forward works but drops connection after few minutes.

**Common Causes**:
- Idle timeout
- API server connection timeout
- Network instability

**Solutions**:
1. Add keepalive to SSH connection:
   ```bash
   ssh -o ServerAliveInterval=60 -p 2222 fedora@localhost
   ```

2. Run port-forward in background with restart:
   ```bash
   while true; do
     oc port-forward vmi/my-vm -n my-namespace 2222:22
     sleep 5
   done &
   ```

## Method 7: Direct Access via Secondary Network

### Issue: Cannot Reach VM on Secondary Network

**Symptoms**:
```bash
$ ping 192.168.100.10
Request timeout
```

**Diagnosis**:
```bash
# Check VM has secondary interface configured
oc get vmi my-vm -n my-namespace -o jsonpath='{.status.interfaces}' | jq

# Check NAD exists
oc get network-attachment-definition -n my-namespace

# Check VM network configuration
oc get vm my-vm -n my-namespace -o jsonpath='{.spec.template.spec.networks}' | jq
```

**Common Causes**:
- NetworkAttachmentDefinition not configured
- VM doesn't have secondary network interface
- Secondary network IP not configured
- Physical network not accessible
- VLAN configuration error

**Solutions**:
1. Verify NAD is correctly configured:
   ```bash
   oc describe network-attachment-definition my-nad -n my-namespace
   ```

2. Verify VM has secondary network:
   ```yaml
   spec:
     template:
       spec:
         networks:
           - name: default
             pod: {}
           - name: secondary-net
             multus:
               networkName: my-nad
         domain:
           devices:
             interfaces:
               - name: default
                 masquerade: {}
               - name: secondary-net
                 bridge: {}
   ```

3. Check IP configuration in VM:
   ```bash
   virtctl console my-vm -n my-namespace
   # Inside VM:
   ip addr show enp2s0  # Secondary interface
   ```

4. Configure static IP via cloud-init (see cloud-init IP configuration tutorial)

5. For VLAN networks, see OVS bridge verification guide

### Issue: Secondary Network Interface Not Visible in VM

**Symptoms**:
Only primary interface (enp1s0) visible, no enp2s0.

**Diagnosis**:
```bash
# Inside VM
ip link show

# Check from OpenShift
oc get vmi my-vm -n my-namespace -o jsonpath='{.status.interfaces}' | jq
```

**Common Causes**:
- NAD doesn't exist in same namespace
- VM spec missing secondary network interface
- Multus CNI not working

**Solutions**:
1. Check NAD in correct namespace:
   ```bash
   oc get network-attachment-definition -n my-namespace
   ```

2. Verify VM spec has both network and interface defined

3. Check multus logs:
   ```bash
   oc logs -n openshift-multus -l app=multus
   ```

4. Check virt-launcher pod events:
   ```bash
   POD=$(oc get pods -n my-namespace -l kubevirt.io/domain=my-vm -o name)
   oc describe $POD -n my-namespace
   ```

## Diagnostic Commands

### Complete VM Access Health Check Script

```bash
#!/bin/bash
NAMESPACE="my-namespace"
VM_NAME="my-vm"

echo "=== VM Status ==="
oc get vm $VM_NAME -n $NAMESPACE
oc get vmi $VM_NAME -n $NAMESPACE

echo ""
echo "=== VM Pod Status ==="
oc get pods -n $NAMESPACE -l kubevirt.io/domain=$VM_NAME

echo ""
echo "=== VM IP Addresses ==="
oc get vmi $VM_NAME -n $NAMESPACE -o jsonpath='{.status.interfaces}' | jq

echo ""
echo "=== Guest Agent Status ==="
oc get vmi $VM_NAME -n $NAMESPACE -o jsonpath='{.status.guestOSInfo.id}'
echo ""

echo ""
echo "=== VM Networks ==="
oc get vm $VM_NAME -n $NAMESPACE -o jsonpath='{.spec.template.spec.networks}' | jq

echo ""
echo "=== Services for VM ==="
oc get svc -n $NAMESPACE -l kubevirt.io/domain=$VM_NAME

echo ""
echo "=== NetworkAttachmentDefinitions ==="
oc get network-attachment-definition -n $NAMESPACE

echo ""
echo "=== Recent Events ==="
oc get events -n $NAMESPACE --field-selector involvedObject.name=$VM_NAME --sort-by='.lastTimestamp' | tail -10
```

### From Inside VM - Network Diagnostic Script

```bash
#!/bin/bash
echo "=== Network Interfaces ==="
ip addr show

echo ""
echo "=== Routing Table ==="
ip route show

echo ""
echo "=== DNS Configuration ==="
cat /etc/resolv.conf

echo ""
echo "=== SSH Service Status ==="
systemctl status sshd

echo ""
echo "=== SSH Listening Ports ==="
ss -tuln | grep :22

echo ""
echo "=== Guest Agent Status ==="
systemctl status qemu-guest-agent

echo ""
echo "=== Firewall Status ==="
systemctl status firewalld 2>/dev/null || echo "Firewall not active"

echo ""
echo "=== Connectivity Test ==="
ping -c 2 8.8.8.8 && echo "External connectivity: OK" || echo "External connectivity: FAILED"
```

## Access Method Comparison - Troubleshooting Decision Tree

```
Need access to VM?
│
├─ VM not running? → virtctl start VM
│
├─ Just need console? → virtctl console
│   └─ Not working? → Check VM running, check virt-launcher pod
│
├─ Need GUI? → virtctl vnc
│   └─ Not working? → Check VNC graphics device, install VNC viewer
│
├─ Need SSH?
│   ├─ From workstation with virtctl? → virtctl ssh
│   │   └─ Not working? → Check guest agent, SSH keys, SSH service
│   │
│   ├─ Persistent access needed? → NodePort Service
│   │   └─ Not working? → Check service selector, NodePort number, node firewall
│   │
│   ├─ Public access needed? → LoadBalancer Service
│   │   └─ Not working? → Check LoadBalancer provider installed, IP assigned
│   │
│   ├─ Temporary access? → oc port-forward
│   │   └─ Not working? → Check local port free, VMI running, SSH in VM
│   │
│   └─ Direct network access? → Secondary Network
│       └─ Not working? → Check NAD, VM secondary interface, IP config, network routing
```

## Additional Resources

- [OpenShift Virtualization Networking](https://docs.openshift.com/container-platform/latest/virt/vm_networking/virt-about-vm-networking.html)
- [KubeVirt Network Access](https://kubevirt.io/user-guide/virtual_machines/accessing_virtual_machines/)
- [Virtctl CLI Reference](https://kubevirt.io/user-guide/operations/virtctl_client_tool/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [MetalLB](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/ingress_and_load_balancing/load-balancing-with-metallb)
- [Cloud-init IP Configuration Troubleshooting](./cloud-init-troubleshooting.md)
- [OVS Bridge Verification](./ovs-bridge-verification.md)

## Example Test: ocplab01 Cluster

Testing on SNO cluster (ocplab01):

```bash
$ oc get vmi fedora-access-vm -n vm-access-test -o jsonpath='{.status.interfaces}' | jq
[
  {
    "infoSource": "domain, guest-agent, multus-status",
    "interfaceName": "eth0",
    "ipAddress": "10.128.0.171",
    "ipAddresses": [
      "10.128.0.171",
      "fd02::62"
    ],
    "mac": "02:92:c2:00:00:08",
    "name": "default",
    "queueCount": 1
  }
]
```

Guest agent status:
```bash
$ oc get vmi fedora-access-vm -n vm-access-test -o jsonpath='{.status.guestOSInfo}' | jq
{
  "id": "fedora",
  "kernelRelease": "6.11.4-301.fc41.x86_64",
  "kernelVersion": "#1 SMP PREEMPT_DYNAMIC Thu Oct 10 16:16:06 UTC 2024",
  "name": "Fedora Linux",
  "prettyName": "Fedora Linux 41 (Cloud Edition)",
  "version": "41",
  "versionId": "41"
}
```

This confirms VM is accessible and guest agent is operational, enabling all SSH-based access methods.

