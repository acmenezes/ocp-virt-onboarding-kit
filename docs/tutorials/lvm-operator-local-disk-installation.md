# Installing LVM Storage Operator with Local Disk

This tutorial demonstrates how to install and configure the LVM Storage operator to use a spare local disk on OpenShift nodes. The LVM operator provides dynamic storage provisioning using Logical Volume Manager, ideal for Single Node OpenShift (SNO) or bare-metal clusters with local storage.

## Prerequisites

- **OpenShift 4.14+**: LVM Storage operator is supported
- **Local disk available**: At least one spare disk on cluster nodes
- **Cluster admin access**: Required to install operators and create storage classes
- **Disk can have existing data**: The operator can force-wipe disks using the `forceWipeDevicesAndDestroyAllData` parameter

Versions tested:
```
OCP 4.19
```

## Important Warning

⚠️ **Data Loss Warning**: Using the `forceWipeDevicesAndDestroyAllData: true` parameter will **permanently destroy all data** on the specified disk. Ensure you have backups and are using the correct disk device path before proceeding.

## Step 1: Identify Available Disks

Before configuring the LVM operator, identify which disk you want to use for storage.

```bash
# Get your node name
oc get nodes

# List all block devices on the node
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

**In this example:**
- `sda` - Spare disk with existing LVM data (this will be our target)
- `sdc` - System disk with OS partitions (DO NOT USE)

**Check for existing LVM usage:**
```bash
oc debug node/<node-name> -- chroot /host pvs
```

**Example output:**
```
PV         VG  Fmt  Attr PSize   PFree  
/dev/sda   vg1 lvm2 a--  558.37g <55.84g
```

This shows `/dev/sda` has existing LVM configuration that will be wiped when using `forceWipeDevicesAndDestroyAllData: true`.

**Get disk device path for persistent identification:**
```bash
oc debug node/<node-name> -- chroot /host ls -la /dev/disk/by-path/ | grep -E "pci.*scsi"
```

**Example output:**
```
lrwxrwxrwx. 1 root root 9 Oct 6 13:30 pci-0000:03:00.0-scsi-0:2:1:0 -> ../../sda
```

The persistent path `/dev/disk/by-path/pci-0000:03:00.0-scsi-0:2:1:0` points to `/dev/sda`.

## Step 2: Install LVM Storage Operator

Create the namespace and install the LVM Storage operator.

```bash
# Create namespace for LVM operator
oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-storage
EOF

# Create OperatorGroup
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-storage-operatorgroup
  namespace: openshift-storage
spec:
  targetNamespaces:
  - openshift-storage
EOF

# Create Subscription to install the operator
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: lvms-operator
  namespace: openshift-storage
spec:
  channel: stable-4.19
  name: lvms-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF
```

**Verify operator installation:**
```bash
# Wait for the operator to be installed (may take 2-3 minutes)
oc get csv -n openshift-storage

# Check operator pod is running
oc get pods -n openshift-storage
```

**Expected output (after operator installation, before LVMCluster):**
```
NAME                             READY   STATUS    RESTARTS   AGE
lvms-operator-864b6975bb-kxt4t   1/1     Running   0          2m
```

**Important**: At this stage, you should only see the `lvms-operator` pod. The `vg-manager` pod will be created automatically **after** you create the LVMCluster resource in Step 3.

**Expected output (after LVMCluster is created in Step 3):**
```
NAME                                  READY   STATUS    RESTARTS   AGE
lvms-operator-864b6975bb-kxt4t        1/1     Running   0          5m
vg-manager-xxxxx                      1/1     Running   0          1m
```

## Step 3: Create LVMCluster Configuration

Create an LVMCluster resource to configure the LVM storage backend. This is where you specify which disk to use and whether to force-wipe existing data.

### Configuration for Disk with Existing Data

If your disk has existing data or partitions (like our `sda` example), use `forceWipeDevicesAndDestroyAllData: true`:

⚠️ **Data Loss Warning**: Using the `forceWipeDevicesAndDestroyAllData: true` parameter will **permanently destroy all data** on the specified disk. Ensure you have backups and are using the correct disk device path before proceeding.


```bash
oc apply -f - <<EOF
apiVersion: lvm.topolvm.io/v1alpha1
kind: LVMCluster
metadata:
  name: lvmcluster
  namespace: openshift-storage
spec:
  storage:
    deviceClasses:
    - name: vg1
      default: true
      thinPoolConfig:
        name: thin-pool-1
        sizePercent: 90
        overprovisionRatio: 10
      nodeSelector:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-role.kubernetes.io/worker
            operator: In
            values:
            - ""
      deviceSelector:
        paths:
        - /dev/sda
        forceWipeDevicesAndDestroyAllData: true
EOF
```

### Configuration for Clean/New Disk

If your disk is clean with no existing data or partitions:

```bash
oc apply -f - <<EOF
apiVersion: lvm.topolvm.io/v1alpha1
kind: LVMCluster
metadata:
  name: lvmcluster
  namespace: openshift-storage
spec:
  storage:
    deviceClasses:
    - name: vg1
      default: true
      thinPoolConfig:
        name: thin-pool-1
        sizePercent: 90
        overprovisionRatio: 10
      nodeSelector:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-role.kubernetes.io/worker
            operator: In
            values:
            - ""
      deviceSelector:
        paths:
        - /dev/sda
EOF
```

**Configuration Parameters Explained:**

- **`name: vg1`**: Volume group name (will be created on the disk)
- **`default: true`**: Makes this deviceClass the default for the cluster
  - **Purpose**: Prevents warning about no default deviceClass
  - **Effect**: The generated StorageClass will be available without explicitly specifying it in PVCs
  - **Note**: If you have multiple deviceClasses, only one should be marked as default
- **`thinPoolConfig`**: Configures thin provisioning for efficient storage use
  - `sizePercent: 90`: Use 90% of disk space for the thin pool
  - `overprovisionRatio: 10`: Allow 10x overprovisioning (provision more storage than physically available)
- **`nodeSelector`**: Which nodes to configure (worker nodes in this example)
- **`deviceSelector.paths`**: Specific disk device paths to use
  - Can use `/dev/sda`, `/dev/disk/by-path/...`, or `/dev/disk/by-id/...`
- **`forceWipeDevicesAndDestroyAllData: true`**: ⚠️ **CRITICAL PARAMETER**
  - **Purpose**: Wipes all existing data and partitions on the disk
  - **Use when**: Disk has existing filesystems, partitions, or LVM configuration
  - **Effect**: Complete data destruction - no recovery possible
  - **Omit when**: Disk is already clean/empty

## Step 4: Verify LVMCluster Status

Run a quick health check to verify the LVMCluster is ready:

```bash
# Check LVMCluster status
oc get lvmcluster -n openshift-storage

# Verify volume group was created
oc debug node/<node-name> -- chroot /host vgs

# Verify thin pool was created
oc debug node/<node-name> -- chroot /host lvs
```

**Expected healthy output:**
- LVMCluster shows `STATUS: Ready`
- VG `vg1` appears with appropriate size
- Thin pool `thin-pool-1` is active with low data/meta usage

For detailed verification steps, expected outputs, and troubleshooting, see the **[LVM Storage Troubleshooting Guide](../troubleshooting/lvm-storage-troubleshooting.md)**.

## Step 5: Set Default StorageClass

The LVM operator automatically creates a StorageClass named `lvms-vg1`, but it **does not set it as the default**. Making it the default StorageClass is essential for OpenShift Virtualization to automatically provision storage for VM disks and DataVolumes.

### Verify StorageClass Exists

```bash
oc get storageclass
```

**Expected output:**
```
NAME       PROVISIONER   RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
lvms-vg1   topolvm.io    Delete          WaitForFirstConsumer   true                   5m
```

**Note**: No `(default)` annotation is shown - this StorageClass is **not yet** the default.

### Set as Default StorageClass

Make the LVM StorageClass the default for the cluster:

```bash
oc patch storageclass lvms-vg1 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Verify Default Status

```bash
oc describe sc | grep IsDefaultClass
```

**Expected output after setting default:**
```
NAME                 PROVISIONER   RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
lvms-vg1 (default)   topolvm.io    Delete          WaitForFirstConsumer   true                   6m
```

The `(default)` annotation confirms this is now the default StorageClass.

### Why This is Important for OpenShift Virtualization

Setting a default StorageClass is **critical** for OpenShift Virtualization:

#### **1. DataSource OS Images Require Default Storage**
OpenShift Virtualization's DataSource OS images (Fedora, RHEL, CentOS Stream, Windows) need a default StorageClass to automatically create boot source volumes:

```bash
# Check available OS images
oc get datasources -n openshift-virtualization-os-images
```

Without a default StorageClass, these DataSources will remain in a pending state and cannot be used for VM creation.

#### **2. Automatic PVC Creation for VMs**
When creating VMs using `dataVolumeTemplates`, if no `storageClassName` is specified, the default StorageClass is used:

```yaml
dataVolumeTemplates:
- metadata:
    name: fedora-volume
  spec:
    pvc:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 30Gi
    sourceRef:
      kind: DataSource
      name: fedora
      namespace: openshift-virtualization-os-images
```

This automatically uses the default StorageClass (`lvms-vg1`) without explicitly specifying it.

#### **3. Simplified VM Creation**
With a default StorageClass:
- VM templates work out-of-the-box
- DataSource cloning happens automatically
- No need to specify `storageClassName` in every PVC
- Consistent storage provisioning across the cluster

#### **4. Boot Source Preparation**
The OpenShift Virtualization operator can automatically prepare boot sources for common OS images when a default StorageClass is available:

```bash
# Check boot source preparation status
oc get DataImportCron -n openshift-virtualization-os-images
```

### Alternative: Explicit StorageClass in VMs

If you choose **not** to set a default StorageClass, you must explicitly specify it in every VM definition:

```yaml
dataVolumeTemplates:
- metadata:
    name: fedora-volume
  spec:
    pvc:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 30Gi
      storageClassName: lvms-vg1  # Must be explicit
    sourceRef:
      kind: DataSource
      name: fedora
      namespace: openshift-virtualization-os-images
```

**Recommendation**: Always set a default StorageClass for OpenShift Virtualization clusters to simplify operations and enable automatic boot source provisioning.

## Step 6: Test Storage Provisioning

Create a test PVC to verify storage provisioning works:

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lvms-test-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: lvms-vg1
EOF
```

**Create a test pod to use the PVC:**
```bash
oc apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: lvms-test-pod
  namespace: default
spec:
  containers:
  - name: test
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: lvms-test-pvc
EOF
```

**Verify PVC is bound:**
```bash
oc get pvc lvms-test-pvc

# Should show STATUS as "Bound"
```

**Verify pod is running and can write to storage:**
```bash
# Wait for pod to start
oc wait --for=condition=Ready pod/lvms-test-pod --timeout=120s

# Test writing to the volume
oc exec lvms-test-pod -- sh -c "echo 'LVM Storage Test' > /data/test.txt && cat /data/test.txt"

# Expected output: LVM Storage Test
```

**Clean up test resources:**
```bash
oc delete pod lvms-test-pod
oc delete pvc lvms-test-pvc
```

## Troubleshooting

If you encounter any issues during installation or operation of LVM storage, refer to the comprehensive **[LVM Storage Troubleshooting Guide](../troubleshooting/lvm-storage-troubleshooting.md)**.

The troubleshooting guide covers:
- Complete verification commands with expected outputs
- Common issues and their solutions
- Device discovery and wipe problems
- PVC provisioning issues
- Thin pool management
- Comprehensive diagnostic procedures

## Understanding forceWipeDevicesAndDestroyAllData

### What It Does

The `forceWipeDevicesAndDestroyAllData: true` parameter:

1. **Removes partition tables**: Wipes GPT, MBR, and any partition information
2. **Destroys filesystems**: Removes ext4, xfs, btrfs, and other filesystem signatures
3. **Clears LVM metadata**: Removes existing physical volumes, volume groups, and logical volumes
4. **Zeroes beginning of disk**: Writes zeros to the first few megabytes to ensure clean state
5. **Forces reuse**: Allows the LVM operator to take ownership of the disk

### When to Use It

**Use `forceWipeDevicesAndDestroyAllData: true` when:**
- Disk has existing partitions or filesystems
- Disk was previously used for LVM (has PV/VG/LV)
- Disk has data you want to destroy
- You're repurposing a disk from another use case

**Don't use it when:**
- Disk is brand new and unformatted
- You're unsure which disk to use (verify first!)
- Disk contains data you need to keep

### Safety Checks Before Using

Always verify the disk device path before enabling this parameter:

```bash
# 1. Verify the disk path
oc debug node/<node-name> -- chroot /host lsblk -o NAME,SIZE,TYPE,MOUNTPOINT

# 2. Ensure disk is NOT mounted
oc debug node/<node-name> -- chroot /host mount | grep /dev/sda

# 3. Double-check you're not using the system disk
oc debug node/<node-name> -- chroot /host df -h | grep sda

# 4. Review disk serial number
oc debug node/<node-name> -- chroot /host lsblk -d -o NAME,SERIAL /dev/sda
```

## References

### OpenShift Documentation
- [Persistent Storage Using LVM Storage](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/storage/persistent-storage-using-local-storage#persistent-storage-using-lvms)
- [Understanding Persistent Storage](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/storage/understanding-persistent-storage#storage-classes_understanding-persistent-storage)

