---
title: SR-IOV network operator improvements for OpenStack
date: 2022-07-13
url: /blog/sriov-network-operator-improvements-openstack/
categories:
  - openstack
  - containers
  - sriov
  - openshift
  - kubernetes
  - nfv
summary: Recent achievements in the SR-IOV Network Operator
cover:
  image: "images/tower.jpg"
---

Stay tuned on our recent achievements in the Kubernetes and OpenStack space when running Fast-Datapath applications.

<!--more-->

In the past months, the [Kubernetes Network Plumbing Working-Group](https://github.com/k8snetworkplumbingwg) added new features to the [SR-IOV Network Operator](https://github.com/k8snetworkplumbingwg/sriov-network-operator/) for the OpenStack platform.

If you’re not familiar with this [operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/), it helps Kubernetes cluster users deploy their workloads to be connected to [Fast Datapath](https://en.wikipedia.org/wiki/Fast_path) (FDP) networking resources. While the operator is named “[SR-IOV](https://en.wikipedia.org/wiki/Single-root_input/output_virtualization)”, we’ll see that it can also manage other types of connectivity.

In fact, the operator helps to provision and configure the [SR-IOV Network Device Plugin for Kubernetes](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin), which is in charge of discovering and advertising networking resources for FDP, mainly (but not exclusively) for SR-IOV Virtual Functions (VFs) and PCI Physical Functions (PFs) that are available on a Kubernetes host (usually a worker node).

The operator hides some complexity to achieve that and provides an easy user interface.


### OpenStack metadata support

The operator originally required [config-drives](https://docs.openstack.org/nova/latest/user/metadata.html#config-drives) to be enabled for the machines connected to the FDP networking, so it could read the OpenStack metadata and Network data.

We removed that requirement by adding support for reading that information from the [Nova metadata](https://docs.openstack.org/nova/latest/user/metadata.html#nova-metadata) service if no config-drive was used.

If your Kubernetes hosts have access to the Nova metadata URL, then you have nothing to do! Otherwise, you’ll need to make sure to create the machines with config-drive enabled.


### Enable VFIO with NOIOMMU

In virtual deployments of Kubernetes where the underlying virtualization platform (e.g. [QEMU](https://www.qemu.org/)) does not support a virtualized I/O memory management unit ([IOMMU](https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit)), the [VFIO](https://www.kernel.org/doc/html/latest/driver-api/vfio.html) PCI driver needs to be loaded with an option named  _enable_unsafe_noiommu_mode_ enabled. This option provides user-space I/O access to a direct memory access capable device without a IOMMU.

The operator is now loading the driver with the right arguments so the users don’t have to worry about it.


### DPDK

The operator was initially designed to work on Baremetal and not necessarily on virtualized platforms. However, when a virtualized Kubernetes host is connected to some network hardware using [DPDK](https://www.dpdk.org), the device is exposed as a [virtio](https://www.linux-kvm.org/page/Virtio) interface (seen as a VF by the operator) but to take advantage of DPDK, the device has to use the [VFIO-PCI driver](https://www.kernel.org/doc/html/latest/driver-api/vfio.html). We added support for detecting [vhost-user](https://qemu.readthedocs.io/en/latest/interop/vhost-user.html) interfaces that are connected to the specified Neutron network used for DPDK. Vhost-user is a module part of DPDK and it helps to run networking in the user-space. You can find more information [here](https://www.redhat.com/en/blog/how-vhost-user-came-being-virtio-networking-and-dpdk).

Here is an example of a SriovNetworkNodePolicy that can be used for Intel devices (you’ll need to change a few things if your device is Mellanox):


```
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: dpdk1
  namespace: openshift-sriov-network-operator
spec:
  deviceType: vfio-pci # change to netdevice if Mellanox
  nicSelector:
	netFilter: openstack/NetworkID:55a54d05-9ec1-4051-8adb-1b5a7be4f1b6
  nodeSelector:
	feature.node.kubernetes.io/network-sriov.capable: 'true'
  numVfs: 1
  priority: 99
  resourceName: dpdk1
  isRdma: false # set to true if Mellanox
```


You’ll need to configure the Network ID that matches your DPDK network in OpenStack.


### OVS Hardware Offload

[Open-vSwitch](https://www.openvswitch.org/) is CPU intensive, which affects system performance and prevents available bandwidth from being fully utilized.

Since OVS 2.8 a feature called OVS Hardware Offload is available. It improves performance significantly by offloading tasks to the hardware running the NIC. OpenStack has [full compatibility](https://docs.openstack.org/neutron/rocky/admin/config-ovs-offload.html) with this feature and the SR-IOV operator can now take advantage of it.

Here is an example of a SriovNetworkNodePolicy that can be used:


```
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: hwoffload1
  namespace: openshift-sriov-network-operator
spec:
  deviceType: netdevice
  nicSelector:
	netFilter: openstack/NetworkID:55a54d05-9ec1-4051-8adb-1b5a7be4f1b6
  nodeSelector:
	feature.node.kubernetes.io/network-sriov.capable: 'true'
  numVfs: 1
  priority: 99
  resourceName: hwoffload1
  isRdma: true
```


For now, we only support [certain types of devices](https://github.com/k8snetworkplumbingwg/sriov-network-operator/blob/master/deploy/configmap.yaml) from the Mellanox vendor.

Also, you’ll need to configure the Network ID that matches your offloaded network for OpenStack.


### Wrap-up

The SR-IOV Network operator was extended to support essential use-cases for OpenStack, so the workloads can be using FDP features. All the features are available in the upstream operator. If you’re an OpenShift user, it’ll be available to you in the 4.11 release and backported to 4.10 in the next zstream, so stay tuned!
