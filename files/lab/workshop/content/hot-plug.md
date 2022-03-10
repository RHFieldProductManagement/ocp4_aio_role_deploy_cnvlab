### Background: Hot-plugging virtual disks
It is expected to have **dynamic reconfiguration** capabilities for VMs today, such as CPU/Memory/Storage/Network hot-plug/hot-unplug. Although these capabilities have been around for the traditional virtualisation platforms, it is a particularly challenging feature to implement in a **Kubernetes** platform because of the Kubernetes principle of **immutable pods**, where once deployed they are never modified. If something needs to be changed, you never do so directly on the Pod. Instead, you’ll build and deploy a new one that has all your needed changes baked in.

OpenShift Virtualization strives to have these dynamic reconfiguration capabilities for VMs although it's a Kubernetes-based platform. In the 4.9 release, hot-plugging virtual disks to a running virtual machine is supported as a [Technology Preview](https://access.redhat.com/support/offerings/techpreview) feature, so as a VM owner, you are able to attach and detach storage on demand.

### Exercise: Hot-plugging a virtual disk using the web console
In OpenShift Virtualization it's possible to hot-plug and hot-unplug virtual disks without stopping your virtual machine. This capability is helpful when you need to add storage to a running virtual machine without incurring down-time. When you hot-plug a virtual disk, you attach a virtual disk to a virtual machine instance while the virtual machine is running. When you hot-unplug a virtual disk, you detach a virtual disk from a virtual machine instance while the virtual machine is running. Only data volumes and persistent volume claims (PVCs) can be hot-plugged and hot-unplugged. You cannot hot-plug or hot-unplug *container* disks.

In this exercise, let's attach a new 5GB disk to one of our MongoDB database VM's by using the web console:
<table>
  <tr>
    <td>

1. Click **Workloads** → **Virtualization** from the side menu.
   
2. Click the **Virtual Machines** tab.
   
3. Select `mongodb-nationalparks` virtual machine to open its **Overview** screen.

4. On the **Disks** tab, click **Add Disk**.

5. In the **Add Disk** window, fill in the information for the virtual disk that you want to hot-plug. **Be sure to review the image provided to ensure you set the values correctly.**

6. Click **Add**.
   </td>
   <td><img src="img/hot-plug-disk-card.png" width="80%"/></td>
    </tr>
    </table>

Once you click Add to attach new disk, a new vm disk is automatically provisioned using the selected storage class, which is ceph-rbd in this exercise, and attached to the running virtual machine. You can see the new disk in the Disks tab of the virtual machine.

<img src="img/hot-plug-disk-list.png" width="80%"/></td>

To verify if the new 5GB disk is recognised and ready to use by the guest operating system, let's connect the console of our virtual machine and list block devices:

1. Click **Workloads** → **Virtualization** from the side menu.
   
2. Click the **Virtual Machines** tab.
   
3. Select `mongodb-nationalparks` virtual machine to open its **Overview** screen.

4. Navigate to the "**Console**" tab. You'll be able to login with "**redhat/openshift**", noting that you may have to click on the console window for it to capture your input.

5. Once you're in the virtual machine, run the *lsblk* command to list block devices recognised by the operating system.
```copy
sudo lsblk
```
<img src="img/hot-plug-disk-lsblk.png" width="50%"/></td>

> **NOTE**: This showed up as "**sda**", as the default interface is "**scsi**" - if we'd have chosen "virtio" this would have been a "**vd***" device.

### Exercise: Expand the VM's disk

OpenShift allows users to easily resize an existing PersistentVolumeClaim (PVC) objects. You no longer have to manually interact with the storage backend or delete and recreate PV and PVC objects to increase the size of a volume. Shrinking persistent volumes is not supported. In this exercise, let's resize our hot-plugged 5GB disk to 7GB by using the web console.

<table>
  <tr>
    <td>

1. Click **Workloads** → **Virtualization** from the side menu.
   
2. Click the **Virtual Machines** tab.
   
3. Select `mongodb-nationalparks` virtual machine to open its **Overview** screen.

4. On the **Disks** tab, click the **PVC name** of the `PersistingHotplug` disk.

5. In the **PersistentVolumeClaims** window, click **Actions** → **Expand PVC**

6. In the **Expand PersistentVolumeClaim** pop-up window, set the **Total Size** as **7 GiB**

7. Click **Expand**.
   
   </td>
   <td><img src="img/hot-plug-disk-expand-pvc.png" width="100%"/></td>
    </tr>
    </table>

Currently, an expanded disk's **new** size isn't automatically recognised by the OS. Instead this must be done at a lower level, which we will explain below. There is an [upstream feature](https://github.com/kubevirt/kubevirt/pull/5981) for this functionality in Kubernetes which is expected to be added in a future version of OpenShift. To verify if the guest operating system has recognised the disk expansion, let's connect the console of our virtual machine and list block devices again.

Size of the hot-plugged disk should still be listed as 5GB instead of 7GB.

1. Click **Workloads** → **Virtualization** from the side menu.
   
2. Click the **Virtual Machines** tab.
   
3. Select `mongodb-nationalparks` virtual machine to open its **Overview** screen.

4. Navigate to the "**Console**" tab. You'll be able to login with "**redhat/openshift**" (if you're not already logged in), noting that you may have to click on the console window for it to capture your input.

5. Once you're in the virtual machine, run the lsblk command to list block devices recognized by the operating system.
```execute
sudo lsblk
```
<img src="img/hot-plug-disk-lsblk.png" width="50%"/></td>

As we mentioned, the OS still sees this disk as 5GB. Let's fix this, but first a quick refresher from previous labs. As you'll recall OpenShift Virtualization creates one pod for each running virtual machine. This pod's primary container runs the virt-launcher. The main purpose of the virt-launcher Pod is to provide the cgroups and namespaces which will be used to host the VM process.
An instance of `libvirtd` is present in every VM pod. virt-launcher uses libvirtd to manage the life-cycle of the VM process.
Libvirt is an open-source API, daemon and management tool for managing platform virtualization including KVM, `virsh` is the most popular command line interface to interact with libvirt daemon `libvirtd`. 

In other words, you can manage KVM VM’s using `virsh` command line interface. To send the disk size change event we need to a guest operating system, we can execute the `virsh blockresize` command inside the virt-launcher pod of the virtual machine.

Let's do it!

First connect the terminal of the `virt-launcher` pod of our virtual machine and execute `virsh blockresize` command to notify the guest operating system so that it can recognize the disk size expansion.

1. Click **Workloads** → **Pods** from the side menu.
   
2. Select the Pod whose name starts with `virt-launcher-mongodb-nationalparks` its **Overview** screen.

3. Navigate to the "**Terminal**" tab. You may have to click on the console window for it to capture your input.

4. Once you're in the Pod's terminal, run the following commands to list block devices attached to the running virtual machine.

First list the running virtual machine and note it's Id.
```copy
virsh list
```
Which should show the following:

~~~bash
 Id   Name                                State
---------------------------------------------------
 1    backup-test_mongodb-nationalparks   running
~~~
Now list the block devices attached to the running virtual machine with `virsh domblklist` command:
```copy
virsh domblklist 1
```
This should list three volumes, the original root disk, the cloud-init disk, and our recently added hot-plug device:

~~~bash
 Target   Source
-----------------------------------------------------------------------------------------------------------
 vda      /dev/mongodb-nationalparks
 vdb      /var/run/kubevirt-ephemeral-disks/cloud-init-data/backup-test/mongodb-nationalparks/noCloud.iso
 sda      /var/run/kubevirt/hotplug-disks/disk-0
~~~
The name of the disk we have expanded should be disk-0. You can check the name of the disk on the **Disks** tab of the virtual machine if you are not sure. Once you identify the disk (which is `sda` in our example), then run the `virsh blockresize` command to notify the guest operating system that the disk is expanded to 7 GB.

```copy
virsh blockresize 1 sda 7g
```
This should return the following if successful:

~~~bash
Block device 'sda' is resized
~~~
After executing the `virsh blockresize` command, verify by listing block devices recognised by the operating system again in the virtual machine console (return to the VM list, select the national parks VM, and then the "Console" tab; you should still be logged in):
```execute
sudo lsblk
```
<img src="img/hot-plug-disk-lsblk-expanded.png" width="50%"/></td>

### Exercise: Hot-unplugging a virtual disk using the web console
It's possible to hot-**un**plug virtual disks when you want to remove them without stopping your virtual machine or virtual machine instance. This capability is helpful when you need to remove storage from a running virtual machine without incurring down-time. When you hot-unplug a virtual disk, you detach a virtual disk from a virtual machine instance while the virtual machine is running. Only data volumes and persistent volume claims (PVCs) can be hot-unplugged. In this exercise, let's detach the disk that we have hot-plugged in the previous exercise from our MongoDB database VM by using the web console:

<table>
  <tr>
    <td>

1. Click **Workloads** → **Virtualization** from the side menu.
   
2. Click the **Virtual Machines** tab.
   
3. Select `mongodb-nationalparks` virtual machine to open its **Overview** screen.

4. Click the **Disks** tab. The page displays a list of disks attached to the virtual machine.

5. Click the Options menu <img src="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABsAAAAjCAIAAADqn+bCAAAACXBIWXMAAA7EAAAOxAGVKw4bAAAA+0lEQVRIie2WMQqEMBBFJ47gUXRBLyBYqbUXULCx9CR2XsAb6AlUEM9kpckW7obdZhwWYWHXX/3i8TPJZEKEUgpOlXFu3JX4V4kmB2qaZhgGKSUiZlkWxzEBC84N9zxv27bdO47Tti0Bs3at4wBgXVca/lJnfN/XPggCGmadIwAsywIAiGhZFk1ydy2EYJKgGCqK4vZUVVU0zKpxnmftp2mi4S/1GhG1N82DMWNNYVmW4zgqpRAxTVMa5t4evlg11nXd9/1eY57nSZIQMKtG13WllLu3bbvrOgJmdUbHwfur8Xniqw6Hh5UYRdGDNowwDA+WvP4UV+JPJ94B1gKUWcTOCT0AAAAASUVORK5CYII=" alt="kebab" title="Options menu"> of the **disk** named `disk-0` which is marked as `PersistingHotplug`

6. Click **Delete**.
   
7. In the confirmation pop-up window, click **Detach** to hot-unplug the disk.
   </td>
   <td><img src="img/hot-plug-disk-detach.png" width="100%"/></td>
    </tr>
    </table>

Once you click **Detach** to hot-unplug the disk, it's detached from the running virtual machine and guest operating system automatically recognises the event. To verify if the 7G disk removal is recognised by the guest operating system, let's connect the console of our virtual machine and list block devices once again. The disk should no longer be listed by the guest operating system.

1. Click **Workloads** → **Virtualization** from the side menu.
   
2. Click the **Virtual Machines** tab.
   
3. Select `mongodb-nationalparks` virtual machine to open its **Overview** screen.

4. Navigate to the "**Console**" tab. You'll be able to login with "**redhat/openshift**", noting that you may have to click on the console window for it to capture your input.

5. Once you're in the virtual machine, run the lsblk command to list block devices recognised by the operating system.
```execute
sudo lsblk
```
<img src="img/hot-plug-disk-lsblk-removed.png" width="50%"/></td>

That's it for hot-plugging and expanding virtual disks - we've hot-plugged a new 5GB disk to our MongoDB database virtual machine using OpenShift web console, expanded its size to 7GB and finally hot-unplugged it from our virtual machine. To move onto the next section of the lab, click "**Network Isolation**" below.
