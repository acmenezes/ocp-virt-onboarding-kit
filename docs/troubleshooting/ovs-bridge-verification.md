# Troubleshooting OVS Bridge Creation for Localnet Networks

This guide helps you verify that OVS (Open vSwitch) bridges are correctly created and configured for localnet secondary networks in OpenShift Virtualization.

## Understanding OVS Bridges vs Linux Bridges

**Important**: OVS bridges behave differently from Linux bridges:

- **OVS bridges** are visible using `ovs-vsctl` commands
- **OVS bridges** may NOT appear as regular network interfaces in `ip addr show` or `ip link show`
- ℹ️ This is **normal and expected behavior** - OVS bridges exist in the OVS database, not as standard Linux network devices

## Verification Process

### Step 1: Check NodeNetworkConfigurationPolicy Status

First, verify that your NNCP was applied and configured successfully:

```bash
# List all NNCPs
oc get nncp

# Check specific NNCP status
oc get nncp br-vlan-creation
```

**Expected Output:**
```
NAME               STATUS      REASON
br-vlan-creation   Available   SuccessfullyConfigured
```

### Step 2: Get Detailed NNCP Information

Review the full NNCP configuration and status:

```bash
oc get nncp br-vlan-creation -o yaml
```

**Look for these key indicators in the output:**

```yaml
status:
  conditions:
  - lastHeartbeatTime: "2025-10-06T20:38:57Z"
    lastTransitionTime: "2025-10-06T20:38:57Z"
    message: 1/1 nodes successfully configured
    reason: SuccessfullyConfigured
    status: "True"
    type: Available
  - reason: SuccessfullyConfigured
    status: "False"
    type: Degraded
```

- `Available: True` with `SuccessfullyConfigured` = Configuration applied
- `Degraded: False` = No issues detected
- `message: X/X nodes successfully configured` = All nodes configured

### Step 3: Verify OVS Bridge Exists

**This is the correct way to check for OVS bridges:**

```bash
# Get your node name
oc get nodes

# List all OVS bridges
oc debug node/<node-name> -- chroot /host ovs-vsctl list-br
```

**Example Output:**
```
Starting pod/ocplab01-debug-k7bfc ...
To use host binaries, run `chroot /host`
br-ex
br-int
br-vlan

Removing debug pod ...
```

✅ If you see `br-vlan` in this list, your bridge was created successfully!

### Step 4: Verify Bridge Configuration Details

Check the complete OVS configuration to see bridge ports and interfaces:

```bash
oc debug node/<node-name> -- chroot /host ovs-vsctl show
```

**Example Output (relevant section):**
```
Bridge br-vlan
    Port enp11s0f0np0
        Interface enp11s0f0np0
            type: system
```

**What to verify:**
- Bridge name (`br-vlan`) appears in the output
- Your physical interface (e.g., `enp11s0f0np0`) is listed as a Port
- Interface type is `system` for physical interfaces

### Step 5: Verify Physical Interface Status

Ensure the physical interface is attached to the bridge and in UP state:

```bash
oc debug node/<node-name> -- chroot /host ip link show <interface-name>
```

**Example Output:**
```
4: enp11s0f0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master ovs-system state UP mode DEFAULT group default qlen 1000
    link/ether e8:eb:d3:13:06:16 brd ff:ff:ff:ff:ff:ff
```

**What to verify:**
- `UP,LOWER_UP` - interface is operational
- `master ovs-system` - interface is managed by OVS
- `state UP` - interface state is up

### Step 6: Why "ip addr show br-vlan" Fails

**This is NORMAL behavior:**

```bash
oc debug node/<node-name> -- chroot /host ip addr show br-vlan
```

**Expected Result:**
```
Device "br-vlan" does not exist.
```

**Why this happens:**
- OVS bridges exist in the OVS database, not as regular Linux network devices
- They won't appear in standard `ip` command outputs unless they have an internal interface
- This does NOT mean your bridge creation failed
- Always use `ovs-vsctl` commands to verify OVS bridges

## Common Issues and Solutions

### Issue 1: Bridge Not Appearing in ovs-vsctl list-br

**Symptoms:**
- NNCP shows `Available: True` but bridge not in `ovs-vsctl list-br`

**Troubleshooting:**
```bash
# Check NNCP conditions
oc get nncp br-vlan-creation -o jsonpath='{.status.conditions[?(@.type=="Available")]}' | jq

# Check NMState operator logs
oc logs -n openshift-nmstate -l app=kubernetes-nmstate-operator

# Check if there are any node network states with issues
oc get nns -A
```

**Common causes:**
- Physical interface name mismatch (verify with `ip link show`)
- OVS not running properly on the node
- NMState operator issues

### Issue 2: Physical Interface Not Attached to Bridge

**Symptoms:**
- Bridge exists but `ovs-vsctl show` doesn't show your interface as a port

**Troubleshooting:**
```bash
# List all ports on the bridge
oc debug node/<node-name> -- chroot /host ovs-vsctl list-ports br-vlan

# Check interface status
oc debug node/<node-name> -- chroot /host ip link show <interface-name>

# Verify NNCP configuration has correct interface name
oc get nncp br-vlan-creation -o yaml | grep -A 5 "name: <interface-name>"
```

**Solutions:**
1. Verify physical interface name is correct in NNCP
2. Check that interface is not already in use by another bridge
3. Ensure interface exists and is available on the node

### Issue 3: Node Selector Mismatch

**Symptoms:**
- NNCP shows successful but bridge doesn't exist

**Troubleshooting:**
```bash
# Check node labels
oc get nodes --show-labels

# Check NNCP node selector
oc get nncp br-vlan-creation -o jsonpath='{.spec.nodeSelector}'
```

**Common issue:**
- Single Node OpenShift (SNO) nodes have `control-plane,master,worker` roles
- NNCP with `node-role.kubernetes.io/worker: ""` selector will still match
- Verify the NNCP status shows "X/X nodes successfully configured"

### Issue 4: Bridge Created But Not Working

**Symptoms:**
- Bridge exists, interface attached, but traffic not flowing

**Troubleshooting:**
```bash
# Check interface carrier status
oc debug node/<node-name> -- chroot /host cat /sys/class/net/<interface-name>/carrier

# Check for interface errors
oc debug node/<node-name> -- chroot /host ethtool -S <interface-name>

# Verify physical link is up
oc debug node/<node-name> -- chroot /host ethtool <interface-name> | grep "Link detected"
```

**Common causes:**
- Physical cable not connected
- Switch port disabled or misconfigured
- Interface speed/duplex mismatch

## Verifying Bridge Mappings for Localnet

After creating the OVS bridge, you need to verify that the bridge mapping to OVN-Kubernetes is working correctly.

### What Are Bridge Mappings?

Bridge mappings connect a **logical network name** (used in NetworkAttachmentDefinitions) to a **physical OVS bridge**. For example:
- Logical name: `localnet-vlan`
- Physical bridge: `br-vlan`
- Mapping: `localnet-vlan:br-vlan`

### Step 1: Verify Bridge Mapping NNCP Status

Check that the bridge mapping NNCP was applied successfully:

```bash
# List all NNCPs
oc get nncp | grep mapping

# Check specific bridge mapping NNCP
oc get nncp localnet-vlan-bridge-mapping
```

**Expected Output:**
```
localnet-vlan-bridge-mapping   Available   SuccessfullyConfigured
```

### Step 2: Get Detailed Bridge Mapping Configuration

```bash
oc get nncp localnet-vlan-bridge-mapping -o yaml
```

**Look for these key sections in the output:**

```yaml
spec:
  desiredState:
    ovn:
      bridge-mappings:
      - bridge: br-vlan
        localnet: localnet-vlan
        state: present
  nodeSelector:
    node-role.kubernetes.io/worker: ""
status:
  conditions:
  - message: 1/1 nodes successfully configured
    reason: SuccessfullyConfigured
    status: "True"
    type: Available
```

### Step 3: Verify OVN Bridge Mappings on the Node

This is the **critical verification** - check that OVN-Kubernetes has the bridge mapping configured:

```bash
oc debug node/<node-name> -- chroot /host ovs-vsctl get Open_vSwitch . external_ids:ovn-bridge-mappings
```

**Expected Output:**
```
"localnet-vlan:br-vlan,physnet:br-ex"
```

**What this means:**
- `localnet-vlan:br-vlan` = Your custom mapping ✅
- `physnet:br-ex` = Default OpenShift mapping ✅
- Multiple mappings are comma-separated

### Step 4: Verify Complete OVS External IDs

Get all OVS external IDs to see the full configuration:

```bash
oc debug node/<node-name> -- chroot /host ovs-vsctl list Open_vSwitch | grep -A 2 external_ids
```

**Example Output:**
```
external_ids        : {hostname=ocplab01, ovn-bridge-mappings="localnet-vlan:br-vlan,physnet:br-ex", ovn-bridge-remote-probe-interval="0", ovn-enable-lflow-cache="true", ovn-encap-ip="192.168.2.241", ovn-encap-type=geneve, ovn-is-interconn="true", ovn-memlimit-lflow-cache-kb="1048576", ovn-monitor-all="true", ovn-ofctrl-wait-before-clear="0", ovn-remote="unix:/var/run/ovn/ovnsb_db.sock", ovn-remote-probe-interval="180000", ovn-set-local-ip="true", rundir="/var/run/openvswitch", system-id="1b41fc3f-f597-4cd9-b508-e8247c209d97"}
```

**Verify:**
- ✅ `ovn-bridge-mappings` includes your mapping
- ✅ `hostname` matches your node
- ✅ `ovn-encap-type=geneve` is present

### Step 5: Verify Physical Interface is Attached

Confirm the physical interface is attached to the bridge:

```bash
oc debug node/<node-name> -- chroot /host ovs-vsctl list-ports br-vlan
```

**Expected Output:**
```
enp11s0f0np0
```

This confirms your physical interface is a port on the bridge and ready to carry traffic.

## Bridge Mapping Traffic Flow

Understanding how traffic flows through bridge mappings:

```
┌─────────────────────────────────────────────────────────────────┐
│ Virtual Machine                                                  │
│   └─> eth1 (secondary network interface)                        │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ NetworkAttachmentDefinition                                      │
│   - name: localnet-vlan-100                                      │
│   - topology: localnet                                           │
│   - vlanID: 100                                                  │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ OVN-Kubernetes (looks up bridge mapping)                         │
│   localnet-vlan → br-vlan                                        │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ OVS Bridge: br-vlan                                              │
│   - Adds VLAN tag 100                                            │
│   - Routes to physical interface                                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Physical Interface: enp11s0f0np0                                 │
│   - Sends VLAN-tagged traffic to physical network               │
└─────────────────────────────────────────────────────────────────┘
```

## Troubleshooting Bridge Mapping Issues

### Issue 1: Bridge Mapping Not in OVN External IDs

**Symptoms:**
- NNCP shows successful but `ovs-vsctl get Open_vSwitch . external_ids:ovn-bridge-mappings` doesn't include your mapping

**Troubleshooting:**
```bash
# Check NNCP was applied
oc get nncp localnet-vlan-bridge-mapping -o yaml

# Restart NMState operator
oc delete pod -n openshift-nmstate -l app=kubernetes-nmstate-operator

# Check NMState handler logs
oc logs -n openshift-nmstate -l component=kubernetes-nmstate-handler
```

**Common causes:**
- NNCP applied before bridge was created
- NMState handler not running properly
- OVS service issues

**Solution:**
Delete and recreate the bridge mapping NNCP after confirming the bridge exists.

### Issue 2: Wrong Bridge Name in Mapping

**Symptoms:**
- Bridge mapping exists but uses wrong bridge name

**Verification:**
```bash
# Check what bridges exist
oc debug node/<node-name> -- chroot /host ovs-vsctl list-br

# Check what mapping references
oc get nncp localnet-vlan-bridge-mapping -o jsonpath='{.spec.desiredState.ovn.bridge-mappings[0].bridge}'
```

**Solution:**
Ensure the bridge name in the mapping NNCP matches the actual OVS bridge name exactly.

### Issue 3: Logical Name Mismatch

**Symptoms:**
- VMs can't connect to network even though bridge mapping exists

**Verification:**
```bash
# Check logical name in bridge mapping
oc get nncp localnet-vlan-bridge-mapping -o jsonpath='{.spec.desiredState.ovn.bridge-mappings[0].localnet}'

# Check NetworkAttachmentDefinition or CUDN references the same name
oc get network-attachment-definitions <nad-name> -n <namespace> -o yaml
```

**Solution:**
The logical network name in the bridge mapping must match the name used in your NetworkAttachmentDefinition or ClusterUserDefinedNetwork configuration.

### Issue 4: Multiple Mappings Conflict

**Symptoms:**
- Multiple bridge mappings configured but traffic goes to wrong bridge

**Verification:**
```bash
# List all bridge mappings
oc debug node/<node-name> -- chroot /host ovs-vsctl get Open_vSwitch . external_ids:ovn-bridge-mappings
```

**Expected format:**
```
"mapping1:bridge1,mapping2:bridge2,mapping3:bridge3"
```

**Solution:**
Ensure each logical network name is unique and maps to the correct bridge. OVN will use the first matching mapping.

## Complete Verification Checklist

Use this checklist to verify your OVS bridge and bridge mappings are correctly configured:

### Bridge Creation
- [ ] NNCP status shows `Available: True` with `SuccessfullyConfigured`
- [ ] NNCP message shows all nodes successfully configured
- [ ] `ovs-vsctl list-br` includes your bridge name
- [ ] `ovs-vsctl show` displays bridge with physical interface as port
- [ ] Physical interface shows `UP,LOWER_UP` status
- [ ] Physical interface shows `master ovs-system`
- [ ] Physical link has carrier (cable connected)
- [ ] No errors in NMState operator logs

### Bridge Mapping
- [ ] Bridge mapping NNCP shows `Available: True`
- [ ] `ovs-vsctl get Open_vSwitch . external_ids:ovn-bridge-mappings` includes your mapping
- [ ] Mapping format is correct: `logical-name:bridge-name`
- [ ] Logical network name matches what you'll use in NAD/CUDN
- [ ] Bridge name in mapping matches actual OVS bridge
- [ ] Physical interface is attached to the bridge (`ovs-vsctl list-ports`)
- [ ] No conflicts with other bridge mappings

## Additional Verification Commands

### Check OVS Service Status
```bash
oc debug node/<node-name> -- chroot /host systemctl status openvswitch
```

### View OVS Logs
```bash
oc debug node/<node-name> -- chroot /host journalctl -u openvswitch -n 50
```

### List All OVS Interfaces
```bash
oc debug node/<node-name> -- chroot /host ovs-vsctl list interface
```

### Check OVS Database
```bash
oc debug node/<node-name> -- chroot /host ovs-vsctl list bridge
```

### Monitor OVS Bridge Traffic (Optional)
```bash
# View ports and their statistics
oc debug node/<node-name> -- chroot /host ovs-ofctl dump-ports br-vlan
```

## Localnet VLAN Network Troubleshooting

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
   - **Solution**: Ensure NAD includes `"physicalNetworkName": "localnet-vlan"` that matches the bridge mapping logical name from bridge mapping configuration
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

- [NMState Documentation](https://nmstate.io/)
- [Open vSwitch Documentation](http://www.openvswitch.org/support/dist-docs/)
- [OpenShift NMState Operator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/networking_operators/k8s-nmstate-about-the-k8s-nmstate-operator)

