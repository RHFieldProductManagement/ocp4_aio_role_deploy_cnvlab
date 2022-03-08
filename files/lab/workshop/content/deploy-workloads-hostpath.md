Now that we have the OCS instance running, let's do the same for the **hostpath** setup we created. Let's leverage the hostpath PVC that we created in a previous step - this is essentially the same as our OCS-based VM instance, except we reference the `rhel8-hostpath` PVC instead:


```execute-1
cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  annotations:
    name.os.template.kubevirt.io/rhel8: Red Hat Enterprise Linux 8.0
  labels:
    app: rhel8-server-hostpath
    kubevirt.io/os: rhel8
    os.template.kubevirt.io/rhel8: 'true'
    template.kubevirt.ui: openshift_rhel8-generic-small
    vm.kubevirt.io/template: openshift_rhel8-generic-small
    workload.template.kubevirt.io/generic: 'true'
  name: rhel8-server-hostpath
spec:
  running: true
  template:
    metadata:
      labels:
        vm.kubevirt.io/name: rhel8-server-hostpath
    spec:
      domain:
        cpu:
          cores: 1
          sockets: 1
          threads: 1
        devices:
          disks:
          - disk:
              bus: sata
            name: rhel8-hostpath
          interfaces:
          - bridge: {}
            model: e1000
            name: tuning-bridge-fixed
        firmware:
          uuid: 5d307ca9-b3ef-428c-8861-06e72d69f223
        machine:
          type: q35
        resources:
          requests:
            memory: 1024M
      evictionStrategy: LiveMigrate
      networks:
        - multus:
            networkName: tuning-bridge-fixed
          name: tuning-bridge-fixed
      terminationGracePeriodSeconds: 0
      volumes:
      - name: rhel8-hostpath
        persistentVolumeClaim:
          claimName: rhel8-hostpath
EOF
```

Check the VirtualMachine object is created:

~~~bash
virtualmachine.kubevirt.io/rhel8-server-hostpath created 
~~~

As before we can see the launcher pod built and run:

```execute-1
oc get pods
```

You will see:

~~~bash
NAME                                        READY   STATUS    RESTARTS   AGE
virt-launcher-rhel8-server-hostpath-ddmjm   0/1     Pending   0          4s
virt-launcher-rhel8-server-ocs-z5rmr        1/1     Running   0          30h
~~~

Now check the VMIs:

```execute-1
oc get vmi
```

And after a few seconds, this should launch...

~~~bash
NAME                    AGE   PHASE     IP               NODENAME                       READY
rhel8-server-hostpath   71s   Running   192.168.123.65   ocp4-worker2.%node-network-domain%   True
rhel8-server-ocs        30h   Running   192.168.123.64   ocp4-worker3.%node-network-domain%   True
~~~

Now look deeper:

```copy
oc describe pod/virt-launcher-rhel8-server-hostpath-9sqxm
```

And looking deeper we can see the hostpath claim we explored earlier being utilised, note the `Mounts` section for where, inside the pod, the `rhel8-hostpath` PVC is attached, and then below the PVC name:


~~~yaml
Name:         virt-launcher-rhel8-server-hostpath-9sqxm
Namespace:    default
Priority:     0
Node:         ocp4-worker2.%node-network-domain%/192.168.123.105
Start Time:   Tue, 09 Nov 2021 20:33:22 +0000
Labels:       kubevirt.io=virt-launcher
              kubevirt.io/created-by=7a19cb48-18bb-4d16-a7eb-4f7811675c17
              vm.kubevirt.io/name=rhel8-server-hostpath
(...)    
    Mounts:
      /var/run/kubevirt from public (rw)
      /var/run/kubevirt-ephemeral-disks from ephemeral-disks (rw)
      /var/run/kubevirt-private from private (rw)
      /var/run/kubevirt-private/vmi-disks/rhel8-hostpath from rhel8-hostpath (rw)
      /var/run/kubevirt/container-disks from container-disks (rw)
      /var/run/kubevirt/hotplug-disks from hotplug-disks (rw)
      /var/run/kubevirt/sockets from sockets (rw)
      /var/run/libvirt from libvirt-runtime (rw)
(...)
Volumes:
  rhel8-hostpath:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  rhel8-hostpath
    ReadOnly:   false
(...)
~~~

If we take a peek inside of that pod we can dig down further, you'll notice that it's largely the same output as the OCS step above.

Open a bash shell in the pod:

```copy
oc exec -it virt-launcher-rhel8-server-hostpath-9sqxm bash
```

Then execute:


```execute-1
virsh domblklist default_rhel8-server-hostpath
```

But the mount point is obviously not over OCS:

~~~bash
 Target   Source
-----------------------------------------------------------------------
 sda      /var/run/kubevirt-private/vmi-disks/rhel8-hostpath/disk.img
~~~

The problem here is that if you try and run the `mount` command to see how this is attached:

```execute-1
mount | grep rhel8
```

It will only show the whole filesystem being mounted into the pod:

~~~bash
/dev/vda4 on /run/kubevirt-private/vmi-disks/rhel8-hostpath type xfs (rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,prjquota)
~~~

However, we can see how this has been mapped in by Kubernetes by looking at the pod configuration on the host. 

Now exit from the pod:

```execute-1
exit
```

First, recall which host your virtual machine is running on:

```execute-1
oc get vmi/rhel8-server-hostpath
```

~~~bash
NAME                    AGE     PHASE     IP               NODENAME                       READY
rhel8-server-hostpath   6m43s   Running   192.168.123.65   ocp4-worker2.%node-network-domain%   True
~~~

And get the name of the launcher pod:

```execute-1
oc get pods | awk '/hostpath/ {print $1;}'
```

This is the launcher pod:

~~~bash
virt-launcher-rhel8-server-hostpath-9sqxm
~~~

> **NOTE**: In your environment, your virtual machine may be running on worker1, simply adjust the following instructions to suit your configuration.

Next, we need to get the container ID from the pod:

```copy
oc describe pod virt-launcher-rhel8-server-hostpath-9sqxm | awk -F// '/Container ID/ {print $2;}'
```

You should see something similar below:

~~~bash
ba06cc69e0dbe376dfa0c8c72f0ab5513f31ab9a7803dd0102e858c94df55744
~~~

> **NOTE**: Make a note (copy it) of this container ID as we'll use it in the next step

Now we can check on the worker node itself, remembering to adjust these commands for the worker that your hostpath based VM is running on, and the container ID from the above:

```copy
oc describe pod virt-launcher-rhel8-server-hostpath-9sqxm | grep "Node:"
```

Then you should see the worker node that the pod is running on:

~~~bash
Node:         ocp4-worker2.%node-network-domain%/192.168.123.105
~~~


Open a debug pod on the node in question, yours may be a different node, adjust to suit your environment:


```execute-1
oc debug node/ocp4-worker2.%node-network-domain%
```

~~~bash
Starting pod/ocp4-worker2aioexamplecom-debug ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.123.105
If you don't see a command prompt, try pressing enter.
~~~

```execute-1
chroot /host
```

Then inspect the container by using the container ID from the previous step: 

```copy
crictl inspect ba06cc69e0dbe376dfa0c8c72f0ab5513f31ab9a7803dd0102e858c94df55744 | grep -A4 rhel8-hostpath
```

Examine the *hostPath*:

~~~json
        "containerPath": "/var/run/kubevirt-private/vmi-disks/rhel8-hostpath",
        "hostPath": "/var/hpvolumes/pvc-0f3ff50d-12b1-4ad6-8beb-be742a6e674a",
        "propagation": "PROPAGATION_PRIVATE",
        "readonly": false,
        "selinuxRelabel": false
(...)
~~~

Here you can see that the container has been configured to have a `hostPath` from `/var/hpvolumes` mapped to the expected path inside of the container where the libvirt definition is pointing to. Don't forget to exit (twice) before proceeding:

Exit the debug shell(s) before proceeding:

```execute-1
exit
```

```execute-1
exit
```

Execute `oc whoami` here just makes sure you're in the right place:

```execute-1
oc whoami
```

You should see:

~~~bash
system:serviceaccount:workbook:cnv
~~~

