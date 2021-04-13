# GPU Passthrough in OpenShift & Kubevirt

## Introduction

It is possible to support video card GPU passthrough using OpenShift and Kubevirt (upstream OpenShift Virtualization). The instructions below will help create a configuration that allows for running ONE virtual machine per physical host with a NVidia GPU card installed. 

## Prerequisites

This document will assume that you have already created a base OpenShift bare metal cluster with at least one baremetal host with a supported NVidia GPU card. These instructions were tested using a V100 card, but should work for other modern GPU cards.

## Configuring the bare metal GPU hosts for Passthrough

Once your OpenShift cluster is up and running: <br>
(1) Install NFD Operator and Create a Policy that enables NVDIA as device manufacturer. <br>
(2) Create and Apply Entitlements: oc create -f 0003-cluster-wide-machineconfigs.yaml  <br>
Ref: https://docs.nvidia.com/datacenter/kubernetes/openshift-on-gpu-install-guide/index.html<br>
(3) Install NVIDIA GPU Operator and Create ... <br>
(4) Install RH OCP Virt Operator and Setup Hyperconverged and HostPath Configs. <br>


## Setup Storage

You will need storage to run your virtual machines. If you have already configured persistent storage for your cluster you can skip these steps and move onto the [Create new VM](#create-new-vm) step. Notes below are for hostPath provisioner.


### HostPath Provisioner

Clone this: https://github.com/kubevirt/hostpath-provisioner.git

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
    path: "/var/hpvolumes" 
    useNamingPrefix: False
```

oc create -f hostpathprovisioner_cr.yaml -n kubevirt

OPTIONAL: set the hostpath-provisioner as the default storage class
set annotation on storage class "storageclass.kubernetes.io/is-default-class: true"

# Deploying a Windows VM

## Create new VM

We will start by uploading a ISO boot cd to our cluster. We will use the virtctl command to upload the ISO image. This requires that you installed the CDI operator in previous steps.
You can download the virtctl command from the [kubevirt github repo](https://github.com/kubevirt/kubevirt/releases)

```
$ oc new-project myvms
# Get the cdi route
$ oc -n cdi get routes
# update the url below to point to the output from the get routes command above
$ virtctl image-upload --uploadproxy-url=https://cdi-uploadproxy-cdi.apps.ocp4rhv.example.com dv iso-win10-dv --size=5Gi --image-path=/home/markd/en_windows_10_multiple_editions_x64_dvd_6846432.iso --insecure
```

Using the file "templates/win10vm1-pvc.yaml" create a PVC to store your virtual machine hard disk on, updating your required disk size and storageClass you want to use.

Create a VirtualMachine instance from yaml using file "templates/win10vm1.yaml". Be sure to update the device name under GPU for the name you gave your GPU in the kubevirt-cr configuration step.

From the console, power on the VM and follow standard install procedures for your Windows 10 OS. If you are using virtio for the hard disk controller you will need to follow the steps outlined [here](https://kubevirt.io/user-guide/#/creation/virtio-win?id=how-to-install-during-windows-install) but be sure to select the "w10" directory to load the proper drivers.
  
When the OS install is complete, open device manager and validate that the 3d device is identified.

## Windows 10 needs to be updated before the driver will install

Once you have a running VM the first thing you need to do is update the Windows OS to the latest release from Microsoft (run Windows Update until there are no updates to install.) This specifically applies to NVidia GPU cards, as the driver uses the WHQL model which was not supported in the original Windows 10 release.

For the hardware tested for this document (V100) the drivers are not auto-installed by Microsoft. You will need to goto the NVidia [driver website](https://www.nvidia.com/Download/index.aspx) and download the appropriate driver set.

### Accessing a Windows VM directly via RDP

If you enable RDP from within the WIndows VIrtual machine, it is possible to directly connect to that VM using RDP through the exposure of the RDP service via a NodePort service.

```
$ virtctl expose virtualmachine win10vm1 --name windows-app-server-rdp --port 3389 --target-port 3389 --type NodePort
$ oc get svc
NAME                     TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
windows-app-server-rdp   NodePort   172.30.44.89   <none>        3389:30239/TCP   9s
```

Now open your favorite RDP tool and connect to ANY worker node IP address on the highport number in the above output (eg. 30239)
