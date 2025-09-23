# OpenShift Virtualization Onboarding Kit

A bite-size approach reference project for deploying virtualization workloads for those who are just starting with OpenShift.

**WARNING: This project DOES NOT replace the official OpenShift Virtualization training, which is highly recommended, nor the official documentation. Instead it is supposed to serve as a complement and a facilitator to partner and customer engineers just getting started.**

## Prerequisites

- OpenShift 4.17+ cluster with OpenShift Virtualization operator
- CLI tools: `oc`, `kubectl`, `virtctl` 
- Optional: `kustomize`, `helm`, `ansible` depending on chosen approach

### Quick Start Guides (WIP):

#### Setting up the basics:

- Setting UP Local Environment and Tools for OCP virtualization
- [Virtctl basics](docs/tutorials/virtctl-basic-overview.md)

#### Working with files, images and templates

- Transferring files to and from VMs
- Creating and Using your own custom VM images
- Shrinking large images
- Converting images to qcow2 format
- Creating your own custom template on OCP Virtualization

#### Configuring network for special purposes

- [Setting a Primary User Defined Network for your VMs](docs/tutorials/layer2-primary-network-udn-configuration.md)
- Configuring secondary networks with layer 2 topology
- [Setting up your VMs on the same network as your nodes \(localnet\)](docs/tutorials/localnet-secondary-network-configuration.md)
- Setting up your VMs on VLANs (localnet)
- Setting up your VMs on VLANs (Linux Bridges)
- Setting VMs on Hardware Accelerated networks (SR-IOv)
- Configuring IP addresses through cloud-init

#### Configuring storage for OCP virtualization

- [Configuring a Default Storage Class for OCP Virtualization](docs/tutorials/configure-default-storage-class.md)
- Creating Storage Profiles for OCP virtualization
- Using local storage with HostPath provisioner
- Creating Storage Classes
- Using local Storage with LVM operator

## License
This project is licensed under the MIT License - see the LICENSE file for details.
