# Creating Custom OS Images and Golden Images

## Overview

This tutorial demonstrates how to create custom golden images for OpenShift Virtualization. Golden images are pre-configured VM disk images that serve as templates for deploying multiple VMs with consistent software, configurations, and settings. This approach accelerates deployment, ensures consistency across environments, and reduces configuration drift.

## What are Golden Images?

Golden images are standardized, pre-configured VM images that contain:
- Base operating system
- Pre-installed software packages
- System configurations
- Security hardening
- Optimizations

Benefits of using golden images:
- Faster VM deployment (no need to install software after boot)
- Consistency across all VMs
- Reduced configuration errors
- Simplified maintenance and updates
- Version control for VM configurations

## Prerequisites

- OpenShift 4.17+ with OpenShift Virtualization operator installed
- CLI tools: `oc`, `virtctl`, `kubectl`
- Storage class configured (this tutorial uses `lvms-vg1`)
- Basic understanding of Linux system administration
- Basic understanding of cloud-init

## Golden Image Workflow

The process involves these steps:
1. Deploy a base VM from a standard OS image
2. Customize the VM (install software, configure settings)
3. Generalize the VM (remove machine-specific data)
4. Stop the VM and capture its disk
5. Create a DataSource for the golden image
6. Deploy new VMs from the golden image

## Step 1: Deploy Base VM

Create a namespace for the golden image workflow:

```bash
oc create namespace golden-images
```

Create a base Fedora VM that we'll customize:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: base-fedora-vm
  namespace: golden-images
spec:
  dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1beta1
      kind: DataVolume
      metadata:
        name: base-fedora-vm-disk
      spec:
        sourceRef:
          kind: DataSource
          name: fedora
          namespace: openshift-virtualization-os-images
        storage:
          resources:
            requests:
              storage: 30Gi
  runStrategy: RerunOnFailure
  template:
    metadata:
      labels:
        kubevirt.io/domain: base-fedora-vm
    spec:
      domain:
        cpu:
          cores: 2
        devices:
          disks:
            - disk:
                bus: virtio
              name: rootdisk
            - disk:
                bus: virtio
              name: cloudinitdisk
          interfaces:
            - masquerade: {}
              name: default
          rng: {}
        machine:
          type: pc-q35-rhel9.4.0
        memory:
          guest: 4Gi
      networks:
        - name: default
          pod: {}
      volumes:
        - dataVolume:
            name: base-fedora-vm-disk
          name: rootdisk
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              user: fedora
              password: fedora123
              chpasswd: { expire: False }
              ssh_pwauth: True
          name: cloudinitdisk
```

Save this as `base-fedora-vm.yaml` and deploy:

```bash
oc apply -f base-fedora-vm.yaml
```

Wait for the VM to be ready. This may take several minutes as the DataVolume clones from the source image:

```bash
oc get dv -n golden-images
```

Output showing clone progress:
```
NAME                  PHASE             PROGRESS   RESTARTS   AGE
base-fedora-vm-disk   CloneInProgress   45.2%                 2m
```

Once the DataVolume shows `Succeeded`, check the VM status:

```bash
oc get vmi -n golden-images
```

Output:
```
NAME             AGE   PHASE     IP             NODENAME   READY
base-fedora-vm   4m    Running   10.128.1.151   ocplab01   True
```

Note: Initial disk cloning typically takes 3-5 minutes depending on your storage performance.

## Step 2: Customize the VM

Access the VM console:

```bash
virtctl console base-fedora-vm -n golden-images
```

Login with username `fedora` and password `fedora123`.

Run the customization steps:

```bash
# Update system packages
sudo dnf update -y

# Install nginx web server and common tools
sudo dnf install -y nginx vim htop curl wget git

# Enable nginx to start on boot
sudo systemctl enable nginx

# Create a custom welcome page
sudo bash -c 'cat > /usr/share/nginx/html/index.html << EOF
<!DOCTYPE html>
<html>
<head>
    <title>Golden Image VM</title>
</head>
<body>
    <h1>VM from Golden Image</h1>
    <p>This VM was deployed from a golden image with pre-installed software.</p>
    <p>Nginx is pre-configured and ready to use.</p>
</body>
</html>
EOF'

# Configure firewall
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

## Step 3: Generalize the VM

Before capturing the image, we need to remove machine-specific data to make it reusable:

```bash
# Clean package cache
sudo dnf clean all

# Clean logs
sudo find /var/log -type f -exec truncate -s 0 {} \;

# Remove machine ID
sudo rm -f /etc/machine-id
sudo touch /etc/machine-id

# Clean cloud-init
sudo cloud-init clean --logs --seed

# Clean bash history
history -c
cat /dev/null > ~/.bash_history

# Sync and exit
sync
exit
```

Exit the console by pressing `Ctrl+]`.

## Step 4: Stop the VM

Stop the VM to capture its disk:

```bash
virtctl stop base-fedora-vm -n golden-images
```

Verify it's stopped:

```bash
oc get vm -n golden-images
```

Output:
```
NAME             AGE   STATUS    READY
base-fedora-vm   15m   Stopped   False
```

## Step 5: Create Golden Image DataSource

Now we'll create a DataSource that references the base VM's disk. This DataSource will be used to deploy new VMs.

Create the DataSource:

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataSource
metadata:
  name: fedora-nginx-golden
  namespace: golden-images
spec:
  source:
    pvc:
      name: base-fedora-vm-disk
      namespace: golden-images
```

Save as `fedora-golden-datasource.yaml` and apply:

```bash
oc apply -f fedora-golden-datasource.yaml
```

Verify the DataSource:

```bash
oc get datasource -n golden-images
```

Output:
```
NAME                   AGE
fedora-nginx-golden    5s
```

## Step 6: Deploy VMs from Golden Image

Now we can deploy multiple VMs from this golden image. Each VM will be cloned from the golden image.

Create the first VM:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: webserver-vm-1
  namespace: golden-images
spec:
  dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1beta1
      kind: DataVolume
      metadata:
        name: webserver-vm-1-disk
      spec:
        sourceRef:
          kind: DataSource
          name: fedora-nginx-golden
          namespace: golden-images
        storage:
          resources:
            requests:
              storage: 30Gi
  runStrategy: RerunOnFailure
  template:
    metadata:
      labels:
        kubevirt.io/domain: webserver-vm-1
    spec:
      domain:
        cpu:
          cores: 2
        devices:
          disks:
            - disk:
                bus: virtio
              name: rootdisk
            - disk:
                bus: virtio
              name: cloudinitdisk
          interfaces:
            - masquerade: {}
              name: default
          rng: {}
        machine:
          type: pc-q35-rhel9.4.0
        memory:
          guest: 4Gi
      networks:
        - name: default
          pod: {}
      volumes:
        - dataVolume:
            name: webserver-vm-1-disk
          name: rootdisk
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              hostname: webserver-vm-1
              fqdn: webserver-vm-1.example.com
          name: cloudinitdisk
```

Create a second VM:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: webserver-vm-2
  namespace: golden-images
spec:
  dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1beta1
      kind: DataVolume
      metadata:
        name: webserver-vm-2-disk
      spec:
        sourceRef:
          kind: DataSource
          name: fedora-nginx-golden
          namespace: golden-images
        storage:
          resources:
            requests:
              storage: 30Gi
  runStrategy: RerunOnFailure
  template:
    metadata:
      labels:
        kubevirt.io/domain: webserver-vm-2
    spec:
      domain:
        cpu:
          cores: 2
        devices:
          disks:
            - disk:
                bus: virtio
              name: rootdisk
            - disk:
                bus: virtio
              name: cloudinitdisk
          interfaces:
            - masquerade: {}
              name: default
          rng: {}
        machine:
          type: pc-q35-rhel9.4.0
        memory:
          guest: 4Gi
      networks:
        - name: default
          pod: {}
      volumes:
        - dataVolume:
            name: webserver-vm-2-disk
          name: rootdisk
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              hostname: webserver-vm-2
              fqdn: webserver-vm-2.example.com
          name: cloudinitdisk
```

Deploy both VMs:

```bash
oc apply -f webserver-vm-1.yaml
oc apply -f webserver-vm-2.yaml
```

Monitor the cloning process:

```bash
oc get dv -n golden-images
```

Output:
```
NAME                  PHASE             PROGRESS   RESTARTS   AGE
base-fedora-vm-disk   Succeeded         100.0%                15m
webserver-vm-1-disk   CloneInProgress   72.5%                 3m
webserver-vm-2-disk   CloneInProgress   71.8%                 3m
```

The DataVolumes will clone from the golden image. This typically takes 3-5 minutes per VM.

## Step 7: Verify the VMs

Check that both VMs are running:

```bash
oc get vmi -n golden-images
```

Output:
```
NAME             AGE   PHASE     IP             NODENAME   READY
webserver-vm-1   5m    Running   10.128.1.152   ocplab01   True
webserver-vm-2   5m    Running   10.128.1.153   ocplab01   True
```

Access one of the VMs and verify nginx is pre-installed:

```bash
virtctl console webserver-vm-1 -n golden-images
```

Check nginx status:

```bash
sudo systemctl status nginx
```

The service should be active (running).

Test the web server:

```bash
curl localhost
```

You should see the custom HTML page created during customization.

Exit the console with `Ctrl+]`.

## Version Management

To create new versions of your golden image:

1. Deploy a VM from the current golden image
2. Apply updates or changes
3. Generalize the VM
4. Create a new DataSource with version in the name (e.g., `fedora-nginx-golden-v2`)
5. Update VM deployments to reference the new version

Example DataSource naming:
- `fedora-nginx-golden-v1`
- `fedora-nginx-golden-v2`
- `fedora-nginx-golden-2025-10`

## Best Practices

### Image Creation
- Keep customizations minimal and focused
- Document all installed software and configurations
- Test the golden image thoroughly before production use
- Version your golden images
- Maintain a changelog for each version

### Generalization
- Always clean logs and temporary files
- Remove SSH host keys
- Remove machine-specific identifiers
- Clean cloud-init data
- Clear bash history

### Storage
- Use appropriate disk sizes for your use case
- Consider thin provisioning if available
- Monitor storage usage for golden images

### Security
- Apply security updates before creating golden images
- Remove any sensitive data or credentials
- Disable unnecessary services
- Configure firewall rules appropriately
- Use cloud-init for VM-specific secrets (SSH keys, passwords)

## Cleanup

To clean up the resources created in this tutorial:

```bash
# Delete VMs
oc delete vm webserver-vm-1 webserver-vm-2 base-fedora-vm -n golden-images

# Delete DataSource
oc delete datasource fedora-nginx-golden -n golden-images

# Delete namespace
oc delete namespace golden-images
```

## Summary

In this tutorial, you learned how to:
- Deploy and customize a base VM
- Generalize a VM for reuse as a golden image
- Create a DataSource for the golden image
- Deploy multiple VMs from the golden image
- Manage golden image versions

Golden images significantly improve VM deployment efficiency and consistency in OpenShift Virtualization environments.

## References

- [OpenShift Virtualization Documentation](https://docs.openshift.com/container-platform/latest/virt/about_virt/about-virt.html)
- [KubeVirt DataVolume Documentation](https://kubevirt.io/user-guide/operations/clone_api/)
- [Containerized Data Importer](https://github.com/kubevirt/containerized-data-importer)

