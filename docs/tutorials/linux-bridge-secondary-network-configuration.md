== Adding a Bridge as a Secondary Network Adapter

This quick-guide describes how to add a secondary Linux bridge adapter to a VM.
A Linux bridge acts as a network switch between pods running on the same node/host machine.
Optionally, a bridge interface can be linked to a physical host interface.
Doing this enables connections to external networks, which includes other pods/VMs running on other nodes/hosts.

NOTE: A Linux bridge network attachment definition (NAD) is the simplest way to connect VM(s) to a VLAN.

IMPORTANT: This guide assumes that VMs have already been created in OpenShift Virtualization prior to being modified.
Parts of this guide will require a namespace to be setup for the VMs which are being attached to a Linux bridge network.

The process is broken down into the following steps:

* <<install_nmstate_operator,Install NMState Operator>> onto the cluster.
** <<nmstate_operator_gui_procedure,OpenShift console procedure for NMState Operator>>
* <<create_linux_bridge_nncp,Create a Node Network Configuration Policy (NNCP) object>> to define the Linux bridge.
** <<create_nncp_prereqs,Prerequisites for creating a NNCP object>>
** <<create_nncp_cli_procedure,CLI procedure to create a NNCP object>>
** <<create_nncp_gui_procedure,OpenShift console procedure to create a NNCP object>>
* <<create_net_attach_def, Create a Network Attachment Definition (NAD)>> to provide Layer 2 networking to pods/VMs.
** <<create_nad_prereqs,Prerequisites for creating a NAD>>
** <<create_nad_cli_procedure,CLI procedure to create a NAD object>>
** <<create_nad_gui_procedure,OpenShift console procedure to create a NAD object>>
* <<configure_vm_nic, Configure the VM network interface connection (NIC)>> to connect the VM to the Linux bridge network.
** <<vm_nic_prereqs,Prerequisites for configuring the VM NIC>>
** <<vm_nic_gui_prcoedure,OpenShift console procedure to configure the VM NIC>>
** <<vm_nic_cli_procedure,CLI procedure to configure the VM NIC>>

=== Install the Kubernetes NMState Operator [[install_nmstate_operator]]
This operator is required to configure the Node Network Configuration Policy (NNCP) objects used to create new bridge network adapters on each node.
The NNCP actually creates the bridge interface on all (or specific) nodes/hosts in the cluster, which of course can be filtered using labels.
The bridge interface must exist on the nodes/hosts that the VMs are running on, before you can attach a secondary NIC to a pod/VM.

==== Procedure using the OpenShift console: [[nmstate_operator_gui_procedure]]
Login to the OpenShift admin console as any user with cluster-admin privileges (such as the default `kubeadmin` account).

On the navigation menu (left side), expand *Operators* then select *OperatorHub*.

Locate and select the *Kubernetes NMState Operator* from the grid. Use the search box to filter the operator listings by name.

Click *Install* to view the installation page, then click *Install* (accepting the defaults).

When the operator installation completes, select *View Operator*.

Under the Provided APIs list, click to *Create instance* of an NMState custom resource (CR).

Ensure the CR is named `nmstate` and click *Create*.

Wait until the web console displays the `Web console update is available` pop up message, and click *Refresh web console* on the bottom of the pop up (or wait a few moments and refresh).

NOTE: On later versions of OpenShift, you may instead be logged out of the OpenShift console automatically, and you must then re-authenticate.

From the navigation menu, expand *Networking* and check for the *NodeNetworkConfigurationPolicy* and *NodeNetworkState* items.
These should both exist before proceeding with <<create_linux_bridge_nncp,creating  NodeNetworkConfigurationPolicy (NNCP)>>.

==== Procedure using the CLI: [[nmstate_operator_cli_procedure]]
Before proceeding, authenticate to your OpenShift cluster's API as a cluster-admin.

Create the `openshift-nmstate` namespace:

----
oc create namespace openshift-nmstate
----

Using your favorite text editor, create the following Subscription object by copy/pasting the following yaml into a new buffer:

----
apiGroup: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: nmstate-operator
  namespace: openshift-nmstate
spec:
  channel: stable
  installPlanApproval: Automatic
  name: kubernetes-nmstate-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
----

Save the file as `k8s-nmstate-operator.subscription.yaml`, then create the subscription on the cluster:

----
oc create -f k8s-nmstate-operator.subscription.yaml -n openshift-nmstate
----

Wait until the operator has finished deploying. To check the current installation state, fetch the ClusterServiceVersion (`csv`) and check the install phase (last column):

----
oc get csv -n openshift-nmstate
NAME                                              DISPLAY                       VERSION               REPLACES                                          PHASE
kubernetes-nmstate-operator.4.18.0-202509101149   Kubernetes NMState Operator   4.18.0-202509101149   kubernetes-nmstate-operator.4.18.0-202508271223   Succeeded
----

Once the operator has finished installing, create the `nmstate` object. Once again, in your favorite text editor, paste the following yaml:

----
apiVersion: nmstate.io/v1
kind: NMState
metadata:
  name: nmstate
spec: {}
----

NOTE: `nmstate` objects are cluster scoped, so a namespace is not required. Only one `nmstate` (also named `nmstate`) should exist on the cluster.

The `nmstate` object will trigger the NMState operator to install the `NodeNetworkConfigurationPolicy` and `NodeNetworkState` APIs (CRDs).
Once these APIs are available, proceed with <<create_linux_bridge_nncp,creating a linux bridge network device>>.

=== Create a Linux Bridge NodeNetworkConfigurationPolicy (NNCP) [[create_linux_bridge_nncp]]
The first step is to define the Linux bridge on the cluster nodes using a NodeNetworkConfigurationPolicy (NNCP).
NNCP objects are cluster-scoped, so they do not exist in a namespace.
You can, however control which node(s)/host(s) to which the NNCP applies using key/value labels.

IMPORTANT: If you skip the NNCP step and try to boot a VM with a secondary bridge adapter, it will fail to start.

==== Prerequisites: [[create_nncp_prereqs]]
You must have first installed the Kubernetes NMState Operator and created a `nmstate` CR after installation.

==== Procedure using the OpenShift console [[create_nncp_gui_procedure]]
To create a NodeNetworkConfigurationPolicy, make sure you're still logged into the OpenShift admin console as a cluster admin before continuing on.

From the console navigation menu (left pane), click *Networking* > *NodeNetworkConfigurationPolicy*.

In the web frame, you'll see a button to *Create NodeNetworkConfigurationPolicy* if no NNCP currently exists. Otherwise, click *Create* > *From Form*.

Complete the web form items as desired:

* *Policy Name*: Set this to `lnbr0-policy` for the sake of this example
* *Description* (optional): Add an optional description, for example `linux bridge lnbr0 with node port ens1`
* *Interface Name*: Set this to `lnbr0` for this example
* *Network state*: Leave *Up* selected 
* *Type*: Linux bridge
* *IP configuration* (optional): Check *IPv4* _only_ if you wish to set a static or DHCP address. Otherwise, leave this option unchecked (layer2-only).
* *Port* (optional): Enter the name of a hardware interface on the node, which will be used to connect the bridge network externally (including to pods/VMs on other nodes/hosts).
* *Enable STP* (optional): Check this if you want to enable spanning tree protocol for the bridge network.

Once the form is complete, click *Create*.

NOTE: As with `nmstate` objects, `nncp` objects are cluster-scoped, so the namespace is not required, although multiple `nncp` objects defining different networks can coexist on the cluster.

Once the NNCP is created, proceed with <<create_net_attach_def,creating a network attachment definition>>.

==== Procedure using the CLI: [[create_nncp_cli_procedure]]
Create a NodeNetworkConfigurationPolicy manifest, choosing from either a static, dhcp, or layer2-only (link-level) configuration for the bridge interface.

Launch a new/empty buffer in your favorite text editor, and name the file `lnbr0-policy.nncp.yaml`.

Use one of the example yaml templates that aligns with your use-case below, paste it into the editor and modify it as desired.

[NOTE]
====
.In each example:
* `lnbr0` (shorthand for `linux bridge 0`) is used as the interface name of the Linux bridge.
* `ens1` is the optional `port` adapter used to connect the bridge to the external network (including other nodes).
* `stp` (spanning tree protocol) is disabled in these examples.
* Whether configured as layer2, dhcp, or static, this affects the bridge interface addressing only, and does not affect pods/VMs.
====

.Static Configuration Example
----
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: lnbr0-policy
spec:
  desiredState:
    interfaces:
    - name: br0
      description: static linux bridge with ens1 as a port
      type: linux-bridge
      state: up
      ipv4:
        address:
        - ip: 192.168.1.89
          prefix-length: 24
        dhcp: false
        enabled: true
      bridge:
        options:
          stp:
            enabled: false
        port: ens1
----

.DHCP Configuration Example
----
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: lnbr0-policy
spec:
  desiredState:
    interfaces:
    - name: br0
      description: dhcp linux bridge with ens1 as a port
      type: linux-bridge
      state: up
      ipv4:
        dhcp: true
        enabled: true
      bridge:
        options:
          stp:
            enabled: false
        port: ens1
----

.Layer2-only Configuration Example
----
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: lnbr0-policy
spec:
  desiredState:
    interfaces:
    - name: lnbr0
      description: layer2-only linux bridge with ens1 as a port
      type: linux-bridge
      state: up
      bridge:
        options:
          stp:
            enabled: false
        port: ens1
----

Apply the NNCP yaml file that was just created to the cluster:

----
oc create -f lnbr0-policy.nncp.yaml
----

After a few moments, each applicable node in the cluster will have a new bridge interface named `lnbr0` with the configured connection state.

To confirm that the `lnbr0` interface exists on the node(s), you can either view the `NodeNetworkState` of the cluster, or start a debug session on a node.

To view the `NodeNetworkState` for all nodes in the cluster, fetch the `nns` resource, and optionally `grep` for the bridge interface:

----
oc describe nns | grep lnbr0
----

If you prefer, open a debug session on a node to see things yourself:

----
oc debug node/worker0.example.com
----

Once you have a shell prompt on the node, use the `ip` tool to fetch the status of the bridge interface:

----
ip link show lnbr0
----

NOTE: As with `nmstate` objects, `nncp` objects are cluster-scoped, so the namespace is not required, although multiple `nncp` objects can coexist on the cluster.

Once the existence of the Linux bridge interface is confirmed on the node(s), proceed with <<create_net_attach_def,creating a network attachment definition>>.


=== Creating a Linux Bridge Network Attachment Definition (NAD) [[create_net_attach_def]]
The next step is to create a Network Attachment Definition to provides Layer 2 networking to your pods/VMs.
The NAD is what allows pods/VMs within a specific project/namespace to connect to a secondary network.

==== Prerequisites [[create_nad_prereqs]]
While not strictly required to create a network attachment definition, a NNCP defining the bridge interface must exist on the cluster before you attach a pod/VM to a bridge network. 
Neglecting to do so will prevent any VM with a bridge-based NIC from booting.

IMPORTANT: A namespace containing VMs must either exist or be created before proceeding.

==== Procedure using the CLI: [[create_nad_cli_procedure]]
It is necessary to change to the project/namespace of the VMs before proceeding (replace `bridge-demo` with the namespace for where the VMs reside):

----
oc project bridge-demo
----

Launch an empty file/buffer in your favorite text editor and paste in the following yaml code, changing as needed:

----
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: lnbr0-nad
  namespace: bridge-demo
spec:
  config: '{
    "name": "lnbr0-nad",
    "cniVersion": "0.3.1",
    "type": "bridge",
    "bridge": "lnbr0",
    "ipam": {},
    "vlan": 1,
    "macspoofchk": false,
    "disableContainerInterface": false,
    "portIsolation": false
  }'
----

Each field from the above example is explained below (optional fields are shown with their respective default values):

* *metadata.name*: The K8s name of the NAD
* *metadata.namespace*: This should match the namespace that the pods/VMs are running in (`bridge-demo` in this example).
* *spec.config*: This field accepts a json string value with the following sub-fields:
** *"name"*: The name of the configuration, which should ideally match `metadata.name`
** *"cniVersion"*: CNI Version must currently be `"0.3.1"`.
** *"type"* (required): Type must be set to `"bridge"` to use a Linux bridge interface.
** *"bridge"* (required): The name of the bridge interface defined on nodes/hosts via the NNCP definition.
** *"ipam"* (unsupported): This feature is not supported for virtualization, so it must be empty (layer2 only).
** *"vlan"* (optional): The desired VLAN ID of the interface (defaults to `1`).
** *"macspoofchk"* (optional): Enables mac spoof check, limiting traffic originating from the container to the mac address of the interface (defaults to `false`).
** *"disableContainerInterface"* (optional): Disables the interface (veth peer) within the container, which disables IPAM. This does not affect IPAM usage within OpenShift Virtualization since it's not supported (defaults to `false`).
** *"portIsolation"* (optional): Set isolation on the host interface. This prevents containers from communicating with each other, enforcing communication only with the host or through the gateway. (defaults to `false`)

[IMPORTANT]
====
* Configuring IP address management (IPAM) in a network attachment definition for VMs is not supported, so the `"ipam": {}` definition must be empty.
* The NAD must exist in the same namespace as the pods/VMs.
* OpenShift Virtualization does not support https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_and_managing_networking/configuring-a-network-bond#upstream-switch-configuration-depending-on-the-bonding-modes[Linux bridge bonding modes] 0, 5, and 6.
====

Create the network attachment definition on OpenShift using the yaml file:

----
oc create -f network-attachment-definition.yaml
----

Verify that the NAD was successfully created:

----
oc get network-attachment-definition lnbr0-nad
----

With two of the three components in place, the last step is to <<configure_vm_nic,add a new virtual network adapter to the VM(s)>>.

==== Procedure using the OpenShift console: [[create_nad_console_procedure]]

From the web console, select *Networking* > *NetworkAttachmentDefinitions*.

In the main web frame, use the pull down menu to select the *Project* that the VMs reside in.

Click *Create Network Attachment Definition*.

Type `lnbr0-nad` as the Policy name, and optionally provide a description.

Scroll down to Policy Interface(s) and complete the form as follows:

* *Interface Name*: `lnbr0`
* *Type*: `Linux Bridge`
* *IP configuration* (unsupported): check *IPv4* and optionally provide an IP address (such as 192.168.1.100) and desired prefix length, or select *DHCP*
* *Port* (optional): enter a network interface name which exists on the node(s) (such as `ens1`) to use as the physical bridge adapter.
* *VLAN tag number* (optional): if VLAN use is desired, enter the ID number in the VLAN Tag Number field. 
* *MAC spoof check* (optional): enables MAC spoof filtering, which provides security by allowing only a single MAC address to exit the pod.

Click *Create*.

[NOTE]
====
* Only a subset of the supported configuration options for network attachment definitions are available when using the OpenShift Console.
* For more advanced options & customization from the OpenShift console, you can use the YAML editor tab when creating or modifying a network attachment definition.
* The node must support nftables and the nft binary must be deployed to enable MAC spoof check.
====

You should see the network attachment definition show up in the list once it's created. 

=== Configuring the VM Network Interface [[configure_vm_nic]]
Finally, you configure the virtual machine to use the newly created Linux bridge network attachment definition.

==== Prerequisites [[vm_nic_prereqs]]
The NMState operator, along with the `nmstate`, `nncp` and `network-attachment-definition` resources, should all exist in the cluster (the latter in the respective namespace of the VMs) before proceeding.

The VMs must also already exist in OpenShift Virtualzation before proceeding. The VMs do not have to be running, but changes to running VMs will require restarting (or live migrating) before taking effect.

==== Procedure using the OpenShift Console: [[vm_nic_console_procedure]]
Navigate to *Virtualization* > *VirtualMachines*.

Click on a VM to view its VirtualMachine details page.

On the *Configuration* tab, then click the *Network* tab.

Click *Add network interface*.

Enter the interface Name and select the network attachment definition (`lnbr0-nad` in this example) from the Network list.

Click *Save*.

Restart or live migrate the VM to apply the changes.

.Networking fields for VM interface configuration:
* *Name*: Name for the network interface controller.
* *Model*: Model of the network interface controller (supported values are e1000e and virtio).
* *Network*: List of available network attachment definitions.
* *Type*: Binding method (select bridge for a Linux bridge network).
* *MAC Address*: MAC address for the network interface controller. If not specified, one is assigned automatically.

==== Procedure using the CLI: [[vm_nic_cli_procedure]]

Start by fetching the existing VM configuration in yaml format. In the following example, we use a VM named `bridge-example-0` in the namespace `bridge-example`, and an output file named `bridge-example.vm.yaml`:

----
oc get vm bridge-example-0 -n bridge-example -o yaml > bridge-example.vm.yaml
----

Edit the file you've just created. Add the VM bridge interface and the NAD binding to the VM yaml configuration via the `spec.template.spec.domain.devices.interfaces` and `spec.template.spec.networks` sections.

----
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: bridge-example-0
  namespace: bridge-example
spec:
  ...
  template:
    ...
    spec:
      ...
      domain:
        ...
        devices:
          ...
          interfaces:
          - macAddress: 'ff:00:ba:ff:00:ba'
            masquerade: {}
            name: default
          - bridge: {}
            macAddress: 'ba:00:ff:ba:00:ff'
            model: virtio
            name: nic-linux-bridge-0
      ...
      networks:
      - name: default
        pod: {}
      - multus:
          networkName: nad-lnbr0
        name: nic-linux-bridge-0
  ...
----

Reviewing the two primary fields in the above example:

* *spec.template.spec.domain.devices.interfaces*:
** *`bridge`*: an empty definition `{}` here is sufficient.
** *`macAddress`*: You specify a mac address or leave this field empty and it will auto-generate.
** *`model`*: `virtio` (virtualized hardware) is chosen over `e1000` (emulated adapter) for performance gains in Linux.
** *`name`*: An arbitrary identifier which gets referred to later in the `network` section.
* *spec.template.spec.networks*:
** *`name`*: This is the name of the bridge interface as defined in the `interfaces` field.
** *`multus.networkName`*: The name of the NAD providing pods/VMs access to the bridge network.

Once you've edited and saved the yaml file, apply the VM configuration to the cluster:

----
oc apply -f bridge-example.vm.yaml
----

NOTE: If you edited a running virtual machine, you must restart or live migrate it for the changes to take effect.

The Linux bridge should now be visible from the VM (after a restart, as mentioned). You can use the `nmcli` tool or Windows Settings/Control Panel inside the container to configure the IP address of the bridge interface.
