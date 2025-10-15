# Troubleshooting LVM Storage Operator

This guide provides comprehensive troubleshooting steps for the LVM Storage operator in OpenShift. Use these commands to diagnose and resolve common issues with LVM storage provisioning, volume groups, and persistent volume claims.

## Quick Health Check

Run this comprehensive health check to verify your LVM storage is working correctly:

```bash
# 1. Check LVMCluster status
oc get lvmcluster -n openshift-storage

# 2. Check all pods are running
oc get pods -n openshift-storage

# 3. Check StorageClass exists
oc get storageclass | grep lvms

# 4. Check volume group on node
oc debug node/<node-name> -- chroot /host vgs

# 5. Check thin pool
oc debug node/<node-name> -- chroot /host lvs
```

**Expected healthy output:**
- LVMCluster: `STATUS: Ready`
- Pods: All `Running` with no restarts
- StorageClass: `lvms-vg1` exists
- VG: Shows volume group with available space
- LV: Shows thin pool with low data/meta usage

## Verification Commands by Component

### 1. LVMCluster Status

#### Check Basic Status
```bash
oc get lvmcluster -n openshift-storage
```

**Healthy output:**
```
NAME         STATUS
lvmcluster   Ready
```

**Unhealthy indicators:**
- Status shows `NotReady`, `Degraded`, or `Error`
- No resources found

#### Check Detailed Status
```bash
oc get lvmcluster lvmcluster -n openshift-storage -o yaml
```

**What to look for:**
```yaml
status:
  conditions:
  - status: "True"
    type: ResourcesAvailable
    message: Reconciliation is complete and all the resources are available
  - status: "True"
    type: VolumeGroupsReady
    message: All the VGs are ready
  deviceClassStatuses:
  - name: vg1
    nodeStatus:
    - devices:
      - /dev/sda
      status: Ready
  ready: true
  state: Ready
```

**Key fields:**
- `status.conditions`: Should show `ResourcesAvailable: True` and `VolumeGroupsReady: True`
- `deviceClassStatuses[].nodeStatus[].status`: Should be `Ready`
- `deviceClassStatuses[].nodeStatus[].devices`: Should list your configured disk
- `ready: true` and `state: Ready`

#### View Device Discovery and Exclusions
```bash
oc get lvmcluster lvmcluster -n openshift-storage -o yaml | grep -A 50 "deviceClassStatuses"
```

**Example output:**
```yaml
deviceClassStatuses:
- name: vg1
  nodeStatus:
  - devices:
    - /dev/sda
    excluded:
    - name: /dev/sdc
      reasons:
      - /dev/sdc has children block devices and could not be considered
      - /dev/sdc is not part of the device selector
    status: Ready
```

This shows which devices were selected and why others were excluded.

### 2. Pod Status

#### Check All Pods
```bash
oc get pods -n openshift-storage
```

**Healthy output:**
```
NAME                             READY   STATUS    RESTARTS   AGE
lvms-operator-864b6975bb-kxt4t   1/1     Running   0          10m
vg-manager-xxxxx                 1/1     Running   0          5m
```

**Expected pods:**
- **lvms-operator**: Always present, manages the lifecycle of LVM storage
- **vg-manager**: One per node, manages volume groups on that node
- **topolvm-controller**: May not be present on Single Node OpenShift (SNO)

#### Check Pod Details
```bash
# Get pod details
oc describe pod -n openshift-storage <pod-name>

# Check specific pod logs
oc logs -n openshift-storage <pod-name>

# Check vg-manager logs (most useful for device issues)
oc logs -n openshift-storage daemonset/vg-manager
```

**Common log errors:**
- `"device is in use"` - Disk already mounted or used by system
- `"cannot wipe device"` - Need `forceWipeDevicesAndDestroyAllData: true`
- `"device not found"` - Wrong device path specified
- `"no devices found"` - Device selector doesn't match any available devices

#### Check Operator Installation
```bash
# Check ClusterServiceVersion
oc get csv -n openshift-storage

# Check subscription
oc get subscription -n openshift-storage

# Check install plan
oc get installplan -n openshift-storage
```

### 3. Storage Resources

#### Check StorageClass
```bash
# List all StorageClasses
oc get storageclass

# Get specific LVM StorageClass
oc get storageclass lvms-vg1 -o yaml
```

**Healthy output:**
```
NAME       PROVISIONER   RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
lvms-vg1   topolvm.io    Delete          WaitForFirstConsumer   true                   10m
```

**Key parameters:**
- `PROVISIONER: topolvm.io` - Correct provisioner
- `VOLUMEBINDINGMODE: WaitForFirstConsumer` - Efficient binding
- `ALLOWVOLUMEEXPANSION: true` - Volume expansion enabled

#### Check if StorageClass is Default
```bash
oc get storageclass lvms-vg1 -o jsonpath='{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}'
```

**Output:**
- `true` - This is the default StorageClass
- No output or `false` - Not the default

#### Check PersistentVolumes
```bash
# List all PVs
oc get pv

# Check PVs using LVM storage
oc get pv | grep topolvm
```

### 4. Volume Group Verification

#### Check Volume Group Status
```bash
oc debug node/<node-name> -- chroot /host vgs
```

**Healthy output:**
```
VG   #PV #LV #SN Attr   VSize   VFree  
vg1    1   1   0 wz--n- 558.37g <55.84g
```

**What to verify:**
- VG name matches your LVMCluster configuration (e.g., `vg1`)
- `VSize` shows total disk size
- `VFree` shows available space
- `Attr` should include `wz--n-` (writeable, resizable, normal)

#### Check Volume Group Details
```bash
oc debug node/<node-name> -- chroot /host vgdisplay vg1
```

**Example output:**
```
--- Volume group ---
VG Name               vg1
System ID             
Format                lvm2
Metadata Areas        1
Metadata Sequence No  2
VG Access             read/write
VG Status             resizable
MAX LV                0
Cur LV                1
Open LV               0
Max PV                0
Cur PV                1
Act PV                1
VG Size               558.37 GiB
PE Size               4.00 MiB
Total PE              142942
Alloc PE / Size       128517 / 502.04 GiB
Free  PE / Size       14425 / 56.35 GiB
VG UUID               abc123...
```

**Key indicators of health:**
- `VG Status: resizable`
- `VG Access: read/write`
- `Free PE / Size` shows available space

### 5. Thin Pool Verification

#### Check Thin Pool Status
```bash
oc debug node/<node-name> -- chroot /host lvs
```

**Healthy output:**
```
LV          VG  Attr       LSize    Pool Origin Data%  Meta%  
thin-pool-1 vg1 twi-a-tz-- <502.04g             0.00   6.76
```

**Attribute breakdown (twi-a-tz--):**
- `t` - thin pool
- `w` - writeable
- `i` - inherited
- `a` - active
- `t` - thin
- `z` - zero

**Data% and Meta% thresholds:**
- **Data%**: Shows how much thin pool data is used
  - `< 80%` - Healthy
  - `80-90%` - Monitor closely
  - `> 90%` - Consider expanding or cleaning up
- **Meta%**: Shows metadata usage
  - `< 80%` - Healthy
  - `> 80%` - May need metadata expansion

#### Check Thin Pool Details
```bash
oc debug node/<node-name> -- chroot /host lvdisplay vg1/thin-pool-1
```

### 6. Physical Volume Verification

#### Check Physical Volume
```bash
oc debug node/<node-name> -- chroot /host pvs
```

**Healthy output:**
```
PV         VG  Fmt  Attr PSize   PFree  
/dev/sda   vg1 lvm2 a--  558.37g <55.84g
```

**Attribute breakdown (a--):**
- `a` - allocatable
- `-` - not exported
- `-` - not missing

#### Check Physical Volume Details
```bash
oc debug node/<node-name> -- chroot /host pvdisplay /dev/sda
```

#### Verify Device Path
```bash
# Check device exists
oc debug node/<node-name> -- chroot /host ls -la /dev/sda

# Check device is a block device
oc debug node/<node-name> -- chroot /host lsblk /dev/sda

# Check device is not mounted
oc debug node/<node-name> -- chroot /host mount | grep /dev/sda
```

**If device is mounted:** It cannot be used for LVM storage. Unmount or choose a different device.

### 7. Disk and Device Information

#### List All Block Devices
```bash
oc debug node/<node-name> -- chroot /host lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
```

**Example output:**
```
NAME    SIZE TYPE FSTYPE      MOUNTPOINT MODEL
sda   558.4G disk LVM2_member            PERC H730 Adp
sdc   558.4G disk                        PERC H730 Adp
|-sdc1   1M part                        
|-sdc2 127M part vfat                   
|-sdc3 384M part ext4        /boot      
`-sdc4 557.9G part xfs        /sysroot
```

**How to identify usable disks:**
- No MOUNTPOINT - disk not in use
- No FSTYPE or shows `LVM2_member` (if wiping is intended)
- Has MOUNTPOINT - disk in use by system
- Has partitions (children) - may need wiping

#### Check Disk Serial Numbers
```bash
oc debug node/<node-name> -- chroot /host lsblk -d -o NAME,SIZE,SERIAL
```

Use this to ensure you're targeting the correct disk.

#### Check Device Persistent Paths
```bash
# By path
oc debug node/<node-name> -- chroot /host ls -la /dev/disk/by-path/ | grep sda

# By ID
oc debug node/<node-name> -- chroot /host ls -la /dev/disk/by-id/ | grep sda
```

## Common Issues and Solutions

### Issue 1: LVMCluster Shows "NotReady"

**Symptoms:**
```
NAME         STATUS
lvmcluster   NotReady
```

**Diagnosis:**
```bash
# Check detailed status
oc get lvmcluster lvmcluster -n openshift-storage -o yaml | grep -A 20 "conditions"

# Check vg-manager logs
oc logs -n openshift-storage daemonset/vg-manager
```

**Common causes and solutions:**

1. **Device not found or wrong path**
   ```bash
   # Verify device exists
   oc debug node/<node-name> -- chroot /host ls -la /dev/sda
   ```
   **Solution:** Update LVMCluster with correct device path

2. **Device has existing data/filesystem**
   ```bash
   # Check for existing filesystem
   oc debug node/<node-name> -- chroot /host lsblk -f /dev/sda
   ```
   **Solution:** Add `forceWipeDevicesAndDestroyAllData: true` to deviceSelector

3. **Device is in use**
   ```bash
   # Check if mounted
   oc debug node/<node-name> -- chroot /host mount | grep /dev/sda
   ```
   **Solution:** Unmount device or use different disk

### Issue 2: No Volume Group Created

**Symptoms:**
- LVMCluster exists but `vgs` shows no volume group
- Device not showing in deviceClassStatuses

**Diagnosis:**
```bash
# Check if disk was wiped
oc debug node/<node-name> -- chroot /host pvs

# Check vg-manager logs for errors
oc logs -n openshift-storage daemonset/vg-manager --tail=100

# Check device visibility
oc debug node/<node-name> -- chroot /host lsblk -o NAME,SIZE,TYPE,FSTYPE
```

**Solutions:**

1. **Device not being selected**
   - Verify deviceSelector paths match actual device
   - Check device is not excluded (see deviceClassStatuses.excluded)

2. **Permissions issues**
   - Ensure vg-manager pod has necessary privileges
   - Check SELinux is not blocking access

3. **Device wipe failed**
   - Set `forceWipeDevicesAndDestroyAllData: true`
   - Manually wipe device and restart operator

### Issue 3: PVC Stuck in Pending

**Symptoms:**
```
NAME            STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-pvc        Pending                                      lvms-vg1       5m
```

**Diagnosis:**
```bash
# Check PVC events
oc describe pvc <pvc-name>

# Check if pod using PVC exists
oc get pods --all-namespaces -o wide | grep <pvc-name>

# Check available space in VG
oc debug node/<node-name> -- chroot /host vgs
```

**Common causes:**

1. **WaitForFirstConsumer - No pod scheduled yet**
   - **Normal behavior**: PVC stays Pending until a pod using it is scheduled
   - **Solution**: Create a pod that uses the PVC

2. **Insufficient space**
   ```bash
   # Check VG free space
   oc debug node/<node-name> -- chroot /host vgdisplay vg1 | grep "Free"
   ```
   **Solution:** Delete unused PVs or expand volume group

3. **Thin pool full**
   ```bash
   # Check thin pool usage
   oc debug node/<node-name> -- chroot /host lvs | grep thin-pool
   ```
   **Solution:** Clean up volumes or increase thin pool size

4. **Node selector mismatch**
   ```bash
   # Check pod node selector vs LVMCluster node selector
   oc get pod <pod-name> -o yaml | grep nodeSelector
   oc get lvmcluster lvmcluster -n openshift-storage -o yaml | grep -A 5 nodeSelector
   ```

### Issue 4: Thin Pool Data% or Meta% High

**Symptoms:**
```
LV          VG  Attr       LSize    Data%  Meta%
thin-pool-1 vg1 twi-a-tz-- 502.04g  92.00  85.00
```

**Diagnosis:**
```bash
# Check thin pool details
oc debug node/<node-name> -- chroot /host lvs -a | grep thin-pool

# List all thin volumes
oc debug node/<node-name> -- chroot /host lvs -a -o lv_name,lv_size,data_percent,pool_lv | grep thin-pool

# Check which PVs are consuming space
oc get pv -o custom-columns=NAME:.metadata.name,SIZE:.spec.capacity.storage,STORAGECLASS:.spec.storageClassName | grep lvms
```

**Solutions:**

1. **Delete unused PVCs**
   ```bash
   # Find PVCs not in use
   oc get pvc --all-namespaces

   # Delete PVC
   oc delete pvc <pvc-name> -n <namespace>
   ```

2. **Expand thin pool** (if VG has free space)
   ```bash
   # Check VG free space
   oc debug node/<node-name> -- chroot /host vgs

   # Extend thin pool (if needed manually)
   # Usually handled automatically by LVM operator
   ```

3. **Adjust overprovisioning ratio**
   - Edit LVMCluster to reduce `overprovisionRatio`

### Issue 5: Device Wipe Failed

**Symptoms:**
- LVMCluster created but device not being used
- Logs show "cannot wipe device" or "device busy"

**Diagnosis:**
```bash
# Check current filesystem/partition
oc debug node/<node-name> -- chroot /host lsblk -f /dev/sda

# Check for LVM signatures
oc debug node/<node-name> -- chroot /host pvs | grep /dev/sda

# Check if device is open
oc debug node/<node-name> -- chroot /host lsof | grep /dev/sda
```

**Solution:**
```bash
# Ensure forceWipeDevicesAndDestroyAllData is set
oc edit lvmcluster lvmcluster -n openshift-storage

# Add under deviceSelector:
forceWipeDevicesAndDestroyAllData: true
```

### Issue 6: vg-manager Pod CrashLooping

**Symptoms:**
```
NAME             READY   STATUS             RESTARTS   AGE
vg-manager-xxx   0/1     CrashLoopBackOff   5          3m
```

**Diagnosis:**
```bash
# Check pod logs
oc logs -n openshift-storage <vg-manager-pod> --previous

# Check pod events
oc describe pod -n openshift-storage <vg-manager-pod>

# Check node resources
oc describe node <node-name>
```

**Common causes:**

1. **Device permissions**
   - vg-manager may lack permissions to access device
   - Check SELinux denials in node logs

2. **Missing device**
   - Device path in config doesn't exist
   - Device removed or changed name

3. **Resource constraints**
   - Node running out of memory or CPU
   - Check node capacity

### Issue 7: StorageClass Not Default

**Symptoms:**
- PVCs without explicit storageClassName fail to bind
- Warning: "no default deviceClass was specified"

**Diagnosis:**
```bash
# Check if any StorageClass is default
oc get storageclass | grep default

# Check lvms StorageClass annotations
oc get storageclass lvms-vg1 -o yaml | grep is-default-class
```

**Solutions:**

1. **Set LVM StorageClass as default**
   ```bash
   oc patch storageclass lvms-vg1 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
   ```

2. **Or set default in LVMCluster**
   ```bash
   oc edit lvmcluster lvmcluster -n openshift-storage

   # Add under deviceClasses:
   - name: vg1
     default: true
   ```

## Comprehensive Diagnostic Script

Save this as a shell script to quickly diagnose LVM storage issues:

```bash
#!/bin/bash

NAMESPACE="openshift-storage"
NODE_NAME="<your-node-name>"

echo "=== LVM Storage Diagnostic Report ==="
echo ""

echo "1. LVMCluster Status:"
oc get lvmcluster -n $NAMESPACE
echo ""

echo "2. Pods Status:"
oc get pods -n $NAMESPACE
echo ""

echo "3. StorageClass:"
oc get storageclass | grep lvms
echo ""

echo "4. PVs:"
oc get pv | grep topolvm
echo ""

echo "5. Volume Groups:"
oc debug node/$NODE_NAME -- chroot /host vgs
echo ""

echo "6. Logical Volumes:"
oc debug node/$NODE_NAME -- chroot /host lvs
echo ""

echo "7. Physical Volumes:"
oc debug node/$NODE_NAME -- chroot /host pvs
echo ""

echo "8. Block Devices:"
oc debug node/$NODE_NAME -- chroot /host lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
echo ""

echo "9. Recent vg-manager logs:"
oc logs -n $NAMESPACE daemonset/vg-manager --tail=20
echo ""

echo "=== End of Diagnostic Report ==="
```

### Related Guides
- [LVM Operator Local Disk Installation Tutorial](../tutorials/lvm-operator-local-disk-installation.md) - Installation guide with detailed configuration

