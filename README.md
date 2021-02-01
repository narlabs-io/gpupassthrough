# GPU Passthrough in OpenShift & Kubevirt

It is possible to support video card GPU passthrough using OpenShift and Kubevirt (upstream OpenShift Virtualization). The instructions below will help create a configuration that allows for running ONE virtual machine per physical host with a NVidia GPU card installed. This is an unsupported setup, but will give you an idea of what functions and features are coming to OpenShift Virtualization in the coming releases.

## Prerequisites

This document will assume that you have already created a base OpenShift bare metal cluster with at least one baremetal host with a supported NVidia GPU card. These instructions were tested using a V100 card, but should work for other modern GPU cards. See [Installing a cluster on bare metal](https://docs.openshift.com/container-platform/4.6/installing/installing_bare_metal/installing-bare-metal.html) for details on getting a bare metal cluster up and running.

## Configuring the bare metal GPU hosts for Passthrough

Once your OpenShift cluster is up and running, we need to create some configuration changes to disable loading the default NVidia kernel driver. We will need to identify the hardware IDs for each card in each bare metal host. We will start by logging into each worker node with a GPU and record the PCI ID:

```
oc get nodes
# find the node name of the hardware with GPUs
oc debug node/<node name>
```

Once the debug node is up, we need to run a lspci:

```
$ chroot /host
$ lspci
04:00.0 VGA compatible controller [0300]: NVIDIA Corporation GK208B [GeForce GT 710] [10de:128b] (rev a1)
04:00.1 Audio device [0403]: NVIDIA Corporation GK208 HDMI/DP Audio Controller [10de:0e0f] (rev a1)
09:00.0 3D controller [0302]: NVIDIA Corporation GV100GL [Tesla V100 PCIe 16GB] [10de:1db4] (rev a1)
```

In the above example, the PCI id we are looking for is "10de:1db4". Record this information for each card type you plan to use.

### Creating vfioConfig for pci passthrough

Now that we have identified all the PCI ids that identify the cards we want to run in passthrough, we need to configure our OpenShift cluster to specifically assign the "vfio" driver to the card. To do this we will apply a machineConfig to our cluster that specifically assigns the vfio driver to the associated GPU card.

We are going to create a base64 encoded string which we will put in our machineConfig file. To do this, run the following command with the VendorID(s) that you want to put into passthrough mode. Be sure to put the vendor IDs in using CAPITAL letters only:

```
$ echo "options vfio-pci ids=10DE:1DB4" | base64
b3B0aW9ucyB2ZmlvLXBjaSBpZHM9MTBERToxREI0Cg==
```

Using the base64 output from above create a new file called "vfioConfig.yaml" with the following contents:

```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 55-vfio-gpu
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
    config:
      ignition:
        version: 3.1.0
      storage:
        files:
          - contents:
              source: >-
                data:text/plain;charset=utf-8;base64,b3B0aW9ucyB2ZmlvLXBjaSBpZHM9MTBkZToxZGI0Cg==
            mode: 0644
            overwrite: true
            path: /etc/modprobe.d/gpu-vfio.conf
          - contents:
              source: 'data:,vfio-pci'
            mode: 0644
            overwrite: true
            path: /etc/modules-load.d/gpu-vfio.conf
```

Be sure to update the contents source with the base64 string you got from the prior step.

Log into your cluster with the oc command and then apply the vfioConfig.yaml file to your cluster:

NOTE: This will reboot all your worker nodes one at a time.

```
$ oc login <cluster name>
$ oc create -f vfioConfig.yaml
$ oc get machineconfigpool
# WAIT for worker to have all machinecounts updated
```

## Install upstream kubevirt

GPU Passthrough to virtualized machines is not yet supported in OpenShift Virutalization. It will be supported with an upcoming release. In the mean time, in order to try out this functionality we will leverage the upstream "kubevirt" project to enable PCI passthrough.

### Install Kubevirt from upstream 

```
# Pick an upstream version of KubeVirt to install
$ export RELEASE=v0.37.1
# Deploy the KubeVirt operator
$ kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml
```

We will create a custom kubevirt-cr to deploy kubevirt into our cluster. This will allow us to enable the GPU feature gate as well as enable the kubevirt device manager to handle the GPU device we are going to pass through. Create kubevirt-cr.yaml with contents below. For each card type create a new pciVendorSelector section. The resourceName is arbitrary, but should reflect the card you are connecting to, to simplify things down the line.

```
---
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  certificateRotateStrategy: {}
  configuration:
    developerConfiguration:
      featureGates:
      - GPU
      - LiveMigration
    permittedHostDevices:
      pciHostDevices:
      - pciVendorSelector: "10DE:1DB4"
        resourceName: "nvidia.com/V100"
        externalResourceProvider: true
  customizeComponents: {}
  imagePullPolicy: IfNotPresent
```

Now we will apply the configuration and wait for the install/configuration to complete:

```
$ kubectl apply -f kubevirt-cr.yaml
# wait until all KubeVirt components are up
$ kubectl -n kubevirt wait kv kubevirt --for condition=Available
```

To validate that the kubevirt device manager has successfully configured/recognized the cards, run the following oc command and look for the "resourceName" to be listed in the "Capacity" section and the "Allocatable" section as shown below:

```
$ oc describe node/node6.ocp4rhv.example.com
Name:               node6.ocp4rhv.example.com
Roles:              worker
Labels:             beta.kubernetes.io/arch=amd64
...
Capacity:
  cpu:                            32
  memory:                         131924860Ki
  nvidia.com/V100:                1
  pods:                           250
Allocatable:
  cpu:                            31500m
  memory:                         130773884Ki
  nvidia.com/V100:                1
  pods:                           250
...
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource                       Requests     Limits
  --------                       --------     ------
  cpu                            404m (1%)    0 (0%)
  memory                         1520Mi (1%)  512Mi (0%)
  ephemeral-storage              0 (0%)       0 (0%)
  hugepages-1Gi                  0 (0%)       0 (0%)
  hugepages-2Mi                  0 (0%)       0 (0%)
  devices.kubevirt.io/kvm        0            0
  devices.kubevirt.io/tun        0            0
  devices.kubevirt.io/vhost-net  0            0
  nvidia.com/V100                0            0
```

### Install CDI Importer

The Containerized Data Importer (CDI) is used to help make virtual machine creation from ISO files easier. We will install the CDI operator directly from github using the steps below: 

```
$ export VERSION=$(curl -s https://github.com/kubevirt/containerized-data-importer/releases/latest | grep -o "v[0-9]\.[0-9]*\.[0-9]*")
$ kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-operator.yaml
$ kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-cr.yaml
```

## Setup Storage 

You will need storage to run your vm from. This can be done with any CSI storage provider. The notes below are for setting up NFS or hostPath provisioner.

### NFS Client Storage

Clone this: https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

$ oc new-project nfs-provisioner
$ NAMESPACE=`oc project -q`
$ sed -i'' "s/namespace:.*/namespace: $NAMESPACE/g" ./deploy/rbac.yaml
$ oc create -f deploy/rbac.yaml
$ oc create role use-scc-hostmount-anyuid --verb=use --resource=scc --resource-name=hostmount-anyuid -n $NAMESPACE
$ oc adm policy add-role-to-user use-scc-hostmount-anyuid system:serviceaccount:$NAMESPACE:nfs-client-provisioner

### HostPath Provisoner

use oc debug node/<node name>
- chroot /host
- mkdir /var/hpvolumes

create a machineconfig.yaml file with the following contents:
```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 50-set-selinux-for-hostpath-provisioner
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 3.1.0
    systemd:
      units:
        - contents: |
            [Unit]
            Description=Set SELinux chcon for hostpath provisioner
            Before=kubelet.service

            [Service]
            ExecStart=/usr/bin/chcon -Rt container_file_t /var/hpvolumes

            [Install]
            WantedBy=multi-user.target
          enabled: true
          name: hostpath-provisioner.service
```

create hostpathprovisioner_cr.yaml

```
apiVersion: hostpathprovisioner.kubevirt.io/v1beta1
kind: HostPathProvisioner
metadata:
  name: hostpath-provisioner
spec:
  imagePullPolicy: IfNotPresent
  pathConfig:
    path: "/var/localstorage" 
    useNamingPrefix: "false"
```

oc create -f hostpathprovisioner_cr.yaml -n openshift-cnv

OPTIONAL: set the hostpath-provisioner as the default storage class
set annotation on storage class "storageclass.kubernetes.io/is-default-class: true"

# Deploying a Windows VM

## Create new VM

virtctl image-upload --uploadproxy-url=https://cdi-uploadproxy-openshift-cnv.apps.ocp4rhv.example.com dv iso-win10-dv --size=4Gi --image-path=/home/markd/en_windows_10_multiple_editions_x64_dvd_6846432.iso --insecure

Using the file "templates/win10vm1-pvc.yaml" create a PVC to store your virtual machine hard disk on, updating your required disk size and storageClass you want to use.

Create a VirtualMachine instance from yaml using file "templates/win10vm1.yaml". Be sure to update the device name under GPU for the name you gave your GPU in the kubevirt-cr configuration step.

From the console, power on the VM and follow standard install procedures for your Windows 10 OS. If you are using virtio for the hard disk controller you will need to follow the steps outlined [here](https://kubevirt.io/user-guide/#/creation/virtio-win?id=how-to-install-during-windows-install) but be sure to select the "w10" directory to load the proper drivers.
  
When the OS install is complete, open device manager and validate that the 3d device is identified.

## Windows 10 needs to be updated before the driver will install

Once you have a running VM the first thing you need to do is update the Windows OS to the latest release from Microsoft (run Windows Update until there are no updates to install.) This specifically applies to NVidia GPU cards, as the driver uses the WHQL model which was not supported in the original Windows 10 release.

For the hardware tested for this document (V100) the drivers are not auto-installed by Microsoft. You will need to goto the NVidia [driver website](https://www.nvidia.com/Download/index.aspx) and download the appropriate driver set.

### Accessing a Windows VM directly via RDP

If you enable RDP from within the WIndows VIrtual machine, it is possible to directly connect to that VM using RDP through the exposure of the RDP service via a NodePort service.

`virtctl expose virtualmachine win10vm1 --name windows-app-server-rdp --port 3389 --target-port 3389 --type NodePort`

