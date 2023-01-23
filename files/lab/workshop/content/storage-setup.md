Now that we've got OpenShift Virtualization deployed, let's take a look at storage. First, switch back to the "**Terminal**" view in your lab environment. What you'll find is that OpenShift Data Foundation (ODF, although formerly known as OpenShift Container Storage/OCS - a software storage platform based on [Ceph](https://www.redhat.com/en/technologies/storage/ceph)) has been deployed for you:

```execute-1
oc get sc
```

This command should show the list of Storage Classes that are installed:

~~~bash
NAME                                    PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
localblock                              kubernetes.io/no-provisioner            Delete          WaitForFirstConsumer   false                  6h49m
ocs-storagecluster-ceph-rbd (default)   openshift-storage.rbd.csi.ceph.com      Delete          Immediate              true                   6h48m
ocs-storagecluster-ceph-rgw             openshift-storage.ceph.rook.io/bucket   Delete          Immediate              false                  6h48m
ocs-storagecluster-cephfs               openshift-storage.cephfs.csi.ceph.com   Delete          Immediate              true                   6h48m
openshift-storage.noobaa.io             openshift-storage.noobaa.io/obc         Delete          Immediate              false                  6h44m
~~~


Then check which version of the OCS operator is installed by executing the following:

```execute-1
oc get csv -n openshift-storage
```

Result should be similar to:

~~~bash
NAME                  DISPLAY                       VERSION   REPLACES              PHASE
mcg-operator.v4.9.3   NooBaa Operator               4.9.3     mcg-operator.v4.9.2   Succeeded
ocs-operator.v4.9.3   OpenShift Container Storage   4.9.3     ocs-operator.v4.9.2   Succeeded
odf-operator.v4.9.3   OpenShift Data Foundation     4.9.3     odf-operator.v4.9.2   Succeeded
~~~

We're going to use the already-deployed OCS/ODF based shared storage for the majority of the tasks, and depending on how much time you have left there's a bonus lab section at the end that sets up `hostpath` storage which uses the hypervisor's local disk(s), which can be useful when there's plenty of fast storage available to the host.

In the next few steps we're going to be creating a storage volume for a virtual machine that we'll create in a later lab section, and for this we'll pull in the contents of an existing disk image, so there's an operating system for our virtual machine to boot up. First, make sure you're in the default project:

```execute-1
oc project default
```

You should see:

~~~bash
Now using project "default" on server "https://172.30.0.1:443".
~~~

>**NOTE**: If you don't use the default project for the next few lab steps, it's likely that you'll run into some errors - some resources are scoped, i.e. aligned to a namespace, and others are not. Ensuring you're in the default namespace now will ensure that all of the coming lab steps should flow together.

Now let's create a new OCS-based Peristent Volume Claim (PVC) - a request for a dedicated storage volume that can be used for persistent storage with VM's and containerised applications. For this volume claim we will use a special annotation `cdi.kubevirt.io/storage.import.endpoint` which utilises the Kubernetes Containerized Data Importer (CDI).

> **NOTE**: CDI is a utility to import, upload, and clone virtual machine images for OpenShift Virtualization. The CDI controller watches for this annotation on the PVC and if found it starts a process to import, upload, or clone. When the annotation is detected the `CDI` controller starts a pod which imports the image from that URL. Cloning and uploading follow a similar process. Read more about the Containerized Data Importer [here](https://github.com/kubevirt/containerized-data-importer).

Basically we are asking OpenShift to create this Persistent Volume Claim and use the image in the endpoint to fill it. In this case we use `"http://192.168.123.100:81/rhel8-kvm.img"` in the annotation to ensure that upon instantiation of the PV it is populated with the contents of a CentOS 8 (although it's labelled "rhel8") KVM image which can be later used as a backing store for a virtual machine.

In addition to triggering the CDI utility we also specify the storage class that OCS/ODF uses (`ocs-storagecluster-ceph-rbd`) which will dynamically create the PV in the backend Ceph storage platform. Finally note the `requests` section - we are asking for a 40GB volume size.

OK, let's create the PVC with all this included:


```execute-1 
cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "rhel8-ocs"
  labels:
    app: containerized-data-importer
  annotations:
    cdi.kubevirt.io/storage.import.endpoint: "http://192.168.123.100:81/rhel8-kvm.img"
spec:
  volumeMode: Block
  storageClassName: ocs-storagecluster-ceph-rbd
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 40Gi
EOF
```

This should create our new PVC:

~~~bash
persistentvolumeclaim/rhel8-ocs created
~~~

Once created, CDI triggers the importer pod automatically to take care of the conversion for you:

```execute-1 
oc get pods
```

You should see the importer pod as below:

~~~bash
NAME                   READY   STATUS              RESTARTS   AGE
importer-rhel8-ocs     0/1     ContainerCreating   0          1s
~~~

Watch the logs and you can see the process, it may initially give an error about the pod waiting to start, you can retry after a few seconds:

```execute-1 
oc logs importer-rhel8-ocs -f
```

You will see the log output as below:

~~~bash
I1103 17:33:42.409423       1 importer.go:52] Starting importer
I1103 17:33:42.442150       1 importer.go:135] begin import process
I1103 17:33:42.447139       1 data-processor.go:329] Calculating available size
I1103 17:33:42.448349       1 data-processor.go:337] Checking out block volume size.
I1103 17:33:42.448380       1 data-processor.go:349] Request image size not empty.
I1103 17:33:42.448395       1 data-processor.go:354] Target size 40Gi.
I1103 17:33:42.448977       1 nbdkit.go:269] Waiting for nbdkit PID.
I1103 17:33:42.949247       1 nbdkit.go:290] nbdkit ready.
I1103 17:33:42.949288       1 data-processor.go:232] New phase: Convert
I1103 17:33:42.949328       1 data-processor.go:238] Validating image
I1103 17:33:42.969690       1 qemu.go:250] 0.00
I1103 17:33:47.145392       1 qemu.go:250] 1.02
I1103 17:33:53.728302       1 qemu.go:250] 2.03
I1103 17:33:55.924329       1 qemu.go:250] 3.05
I1103 17:33:58.014054       1 qemu.go:250] 4.06
(...)
I0317 11:46:56.253155       1 prometheus.go:69] 98.24
I0317 11:46:57.253350       1 prometheus.go:69] 100.00
I0317 11:47:00.195494       1 data-processor.go:205] New phase: Resize
I0317 11:47:00.524989       1 data-processor.go:268] Expanding image size to: 40Gi
I0317 11:47:00.878420       1 data-processor.go:205] New phase: Complete
~~~

If you're quick, you can Ctrl-C this watch (don't worry, it won't kill the import, you're just watching its logs), and view the structure of the importer pod to get some of the configuration it's using:

```execute-1 
 oc describe pod $(oc get pods | awk '/importer/ {print $1;}')
```

~~~yaml
Name:         importer-rhel8-ocs
Namespace:    default
Priority:     0
Node:         ocp4-worker1.%node-network-domain%/192.168.123.104
Start Time:   Wed, 03 Nov 2021 17:32:59 +0000
Labels:       app=containerized-data-importer
              app.kubernetes.io/component=storage
              app.kubernetes.io/managed-by=cdi-controller
              app.kubernetes.io/part-of=hyperconverged-cluster
              app.kubernetes.io/version=v4.9.0
              cdi.kubevirt.io=importer
(...)
    Environment:
      IMPORTER_SOURCE:               http
      IMPORTER_ENDPOINT:             http://192.168.123.100:81/rhel8-kvm.img
      IMPORTER_CONTENTTYPE:          kubevirt
      IMPORTER_IMAGE_SIZE:           40Gi
      OWNER_UID:                     11b007c9-293f-46a6-b1e2-e3a59b66ec4b
      FILESYSTEM_OVERHEAD:           0
      INSECURE_TLS:                  false
(...)
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-q6d4s (ro)
    Devices:
      /dev/cdi-block-volume from cdi-data-vol
(...)
Volumes:
  cdi-data-vol:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  rhel8-ocs
    ReadOnly:   false
(...)
~~~

Here we can see the importer settings we requested through our claims, such as `IMPORTER_SOURCE`, `IMPORTER_ENDPOINT`, and `IMPORTER_IMAGE_SIZE`. You can check PVC readiness by executing the command below:

```execute-1 
oc get pvc
```

Once this process has completed you'll notice that your PVC is ready to use:

~~~bash
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                  AGE
rhel8-ocs   Bound    pvc-1a4ea2c5-937c-486d-932c-b2b7d161ec0e   40Gi       RWX            ocs-storagecluster-ceph-rbd   50s
~~~

This same configuration should be reflected when asking OpenShift for a list of *all* persistent volumes (`PV`):

```execute-1 
oc get pv
```

Noting that there will be some additional PV's that are used with OpenShift Data Foundation as well as the image registry: 

~~~bash
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                           STORAGECLASS                  REASON   AGE
local-pv-6751e4c7                          100Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-0-data-1g87z7   localblock                             52m
local-pv-759d6092                          100Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-1-data-1kks6c   localblock                             52m
local-pv-911d74ba                          100Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-1-data-0f9khs   localblock                             52m
local-pv-9d4e1eb0                          100Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-2-data-0hvt5v   localblock                             52m
local-pv-be875671                          100Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-0-data-0vlwls   localblock                             52m
local-pv-e2c7f8a9                          100Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-2-data-17kkwt   localblock                             52m
pvc-1a4ea2c5-937c-486d-932c-b2b7d161ec0e   40Gi       RWX            Delete           Bound    default/rhel8-ocs                               ocs-storagecluster-ceph-rbd            77s
pvc-ee63c434-9d72-4bc4-b224-a7d5ae7aa9b9   100Gi      RWX            Delete           Bound    openshift-image-registry/ocs-imgreg             ocs-storagecluster-cephfs              45m
pvc-f8f83266-dfb2-463b-8707-d4514566eb67   50Gi       RWO            Delete           Bound    openshift-storage/db-noobaa-db-pg-0             ocs-storagecluster-ceph-rbd            46m
~~~

Let's take a look on the Ceph-backed storage system for more information about the image. We can do this by matching the PVC to the underlying RBD image. First describe the persistent volume to get the UUID of the image name by matching the ID of the PV for `rhel8-ocs` in the output above. You'll have to manually enter these values for your environment. In the example above that value is `pvc-1a4ea2c5-937c-486d-932c-b2b7d161ec0e` :


```execute-1
CSI_VOL=$(oc describe pv/`oc get pv | grep default/rhel8-ocs \ 
| awk '{print $1}'` | grep imageName | sed 's/ //g' \ 
| awk -F "=" '{print $2}')
```

Let's make sure that the variable indeed contains the CSI volume ID: 

```execute-1
echo $CSI_VOL
```

This gives us the imageName we need. In the above output that is `csi-vol-70d062c5-408f-11ec-a2b0-0a580a830025`

Now, in the pod's terminal we can use the `rbd` command to inspect the image, noting we must specify the pool name "*ocs-storagecluster-cephblockpool*" as this is a "block" type of PV:


```
oc exec -it -n openshift-storage $(oc get pods -n openshift-storage \ 
| awk '/tools/ {print $1;}') -- rbd info $CSI_VOL \ 
--pool=ocs-storagecluster-cephblockpool
```

This will display information about the image:

~~~yaml
rbd image 'csi-vol-70d062c5-408f-11ec-a2b0-0a580a830025':
	size 40 GiB in 10240 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 115b595bde9a
	block_name_prefix: rbd_data.115b595bde9a
	format: 2
	features: layering
	op_features: 
	flags: 
	create_timestamp: Mon Nov  8 12:28:56 2021
	access_timestamp: Mon Nov  8 12:28:56 2021
	modify_timestamp: Mon Nov  8 12:28:56 2021
~~~


Then execute an `rbd disk-usage` request against the same image to see the disk usage; don't forget to specifiy the correct pool:

```
oc exec -it -n openshift-storage $(oc get pods -n openshift-storage \ 
| awk '/tools/ {print $1;}') --  rbd disk-usage $CSI_VOL --pool=ocs-storagecluster-cephblockpool
```

This will display the usage:

~~~bash
NAME                                         PROVISIONED USED
csi-vol-70d062c5-408f-11ec-a2b0-0a580a830025      40 GiB 8.7 GiB
~~~

That's it! Don't forget to exit from the pod when done.

```execute-1 
exit
```

You'll see that this image matches the correct size, corresponds to the PV that we requested be created for us, and has consumed approximately ~9GB of the disk, reflecting that we've cloned a copy of CentOS8 into our persistent volume via the data importer tool. We'll use this persistent volume to spawn a virtual machine in a later step.

For now, let's move on and check out some networking. Click on "**Networking Setup**" below to move to the next lab section.
