In this section we're going to be configuring the networking for our environment. You're probably just wanting to create virtual machines, but let's just finish this one step and we'll understand a lot more about how networking works in OpenShift.

With OpenShift Virtualization (or more specifically, OpenShift in general - regardless of the workload type) we have a few different options for networking. We can just have our virtual machines be attached to the same pod networks that our containers would have access to, or we can configure more "real-world" virtualisation networking constructs like bridged networking, SR/IOV, and so on. It's also possible to have a combination of these, e.g. both pod networking and a bridged interface directly attached to a VM at the same time, using [Multus](https://github.com/k8snetworkplumbingwg/multus-cni), the default networking CNI in OpenShift 4.x. Multus allows multiple "sub-CNI" devices to be attached to a pod (regardless of whether a virtual machine is running there).

In this lab we're going to utilise multiple options - pod networking **and** a secondary network interface provided by a bridge on the underlying worker nodes (hypervisors). Each of the worker nodes has been configured with an additional, currently unused, network interface that is defined as `enp3s0`, and we'll create a bridge device, called `br1`, so we can attach our virtual machines to it - this network is actually the **same** L2 network as the one attached to `enp2s0`, so it's on the lab network ( `192.168.123.0/24`) as well.

The first step is to use the new Kubernetes NetworkManager state configuration to setup the underlying hosts to our liking. Recall that we can get the **current** state by requesting the `NetworkNodeState` (much of the following is snipped for brevity):


```execute-1
oc get nns/ocp4-worker1.aio.example.com -o yaml
```

This will display the NodeNetworkState in yaml format:

~~~yaml
apiVersion: nmstate.io/v1beta1
kind: NodeNetworkState
metadata:
  creationTimestamp: "2023-02-16T16:46:29Z"
  generation: 1
  name: ocp4-worker1.%node-network-domain%
(...)
    - accept-all-mac-addresses: false
      ethernet:
        auto-negotiation: false
      ethtool:
        feature:
          tx-generic-segmentation: true
          tx-tcp-segmentation: true
        ring:
          rx: 256
          tx: 256
      ipv4:
        address:
        - ip: 192.168.123.104
          prefix-length: 24
        auto-dns: true
        auto-gateway: true
        auto-route-table-id: 0
        auto-routes: true
        dhcp: true
        enabled: true
      ipv6:
        addr-gen-mode: eui64
        address:
        - ip: fe80::5054:ff:fe00:4
          prefix-length: 64
        auto-dns: true
        auto-gateway: true
        auto-route-table-id: 0
        auto-routes: true
        autoconf: true
        dhcp: true
        enabled: true
      lldp:
        enabled: false
      mac-address: "52:54:00:00:00:04"
      mtu: 1500
      name: enp2s0
      state: up
      type: ethernet
    - accept-all-mac-addresses: false
      ethernet:
        auto-negotiation: false
      ethtool:
        feature:
          tx-generic-segmentation: true
          tx-tcp-segmentation: true
        ring:
          rx: 256
          tx: 256
      ipv4:
        address: []
        dhcp: false
        enabled: false
      ipv6:
        address:
        - ip: fe80::5054:ff:fe00:104
          prefix-length: 64
        autoconf: false
        dhcp: false
        enabled: true
      lldp:
        enabled: false
      mac-address: "52:54:00:00:01:04"
      mtu: 1500
      name: enp3s0
      state: up
      type: ethernet
(...)
~~~

In there you'll (hopefully) spot the interface that we'd like to use to create a bridge, `enp3s0`, with DHCP being disabled and not in current use - there are no IP addresses associated to that network. DHCP is configured in the environment, but as part of the installation we forced this network to be disabled via [MachineConfig](https://github.com/RHFieldProductManagement/openshift-aio/blob/main/conf/k8s/97_workers_empty_enp3s0.yaml). We'll make the necessary changes to this interface shortly.

> **NOTE**: The first interface, `enp1s0 ` is used for PXE-based provisioning within the environment (not essential for baremetal deployments, but it's what we've used here), and `enp2s0` is being used for inter-OpenShift communication, including all of the pod networking via OpenShift SDN.


Now we can apply a new `NodeNetworkConfigurationPolicy` for our worker nodes to setup a desired state for `br1` via `enp3s0`, noting that in the `spec` we specify a `nodeSelector` to ensure that this **only** gets applied to our worker nodes; eventually allowing us to attach VM's to this bridge:

```execute-1
cat << EOF | oc apply -f -
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br1-enp3s0-policy-workers
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  desiredState:
    interfaces:
      - name: br1
        description: Linux bridge with enp3s0 as a port
        type: linux-bridge
        state: up
        ipv4:
          enabled: false
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: enp3s0
EOF
```

Check the output:

~~~bash
nodenetworkconfigurationpolicy.nmstate.io/br1-enp3s0-policy-workers created
~~~

Then enquire as to whether it was successfully applied:

```execute-1
oc get nnce
```

Check the status (it may take a few checks before all show as "**Available**", i.e. applied the requested configuration, it will go from "Pending" --> "Progressing" --> "Available"):

~~~bash
NAME                                                           STATUS      REASON
ocp4-worker1.%node-network-domain%.br1-enp3s0-policy-workers   Available   SuccessfullyConfigured
ocp4-worker2.%node-network-domain%.br1-enp3s0-policy-workers   Available   SuccessfullyConfigured
ocp4-worker3.%node-network-domain%.br1-enp3s0-policy-workers   Available   SuccessfullyConfigured
~~~

You can also request the status of the overall policy:

```execute-1
oc get nncp
```

Check the result, again we're expecting to see "**Available**":

~~~bash
NAME                        STATUS      REASON
br1-enp3s0-policy-workers   Available   SuccessfullyConfigured
~~~

We can also dive into the `NetworkNodeConfigurationPolicy` (**nncp**) a little further:


```execute-1
oc get nncp/br1-enp3s0-policy-workers -o yaml
```

You will see NetworkNodeConfigurationPolicy definition in yaml format:

~~~yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"nmstate.io/v1","kind":"NodeNetworkConfigurationPolicy","metadata":{"annotations":{},"name":"br1-enp3s0-policy-workers"},"spec":{"desiredState":{"interfaces":[{"bridge":{"options":{"stp":{"enabled":false}},"port":[{"name":"enp3s0"}]},"description":"Linux bridge with enp3s0 as a port","ipv4":{"enabled":false},"name":"br1","state":"up","type":"linux-bridge"}]},"nodeSelector":{"node-role.kubernetes.io/worker":""}}}
    nmstate.io/webhook-mutating-timestamp: "1676579506531529979"
  creationTimestamp: "2023-02-16T20:31:46Z"
  generation: 1
  name: br1-enp3s0-policy-workers
  resourceVersion: "1503181"
  uid: fff14e27-d550-4309-8150-b49994e1538a
spec:
  desiredState:
    interfaces:
    - bridge:
        options:
          stp:
            enabled: false
        port:
        - name: enp3s0
      description: Linux bridge with enp3s0 as a port
      ipv4:
        enabled: false
      name: br1
      state: up
      type: linux-bridge
  nodeSelector:
    node-role.kubernetes.io/worker: ""
status:
  conditions:
  - lastHeartbeatTime: "2023-02-16T20:32:13Z"
    lastTransitionTime: "2023-02-16T20:32:13Z"
    message: 3/3 nodes successfully configured
    reason: SuccessfullyConfigured
    status: "True"
    type: Available
  - lastHeartbeatTime: "2023-02-16T20:32:13Z"
    lastTransitionTime: "2023-02-16T20:32:13Z"
    reason: SuccessfullyConfigured
    status: "False"
    type: Degraded
  - lastHeartbeatTime: "2023-02-16T20:32:13Z"
    lastTransitionTime: "2023-02-16T20:32:13Z"
    reason: ConfigurationProgressing
    status: "False"
    type: Progressing
  lastUnavailableNodeCountUpdate: "2023-02-16T20:32:13Z"
~~~

If you'd like to fully verify that this has been successfully configured on the host, we can do that easily via the `oc debug node` command (pick any of your workers):

```execute-1
oc debug node/ocp4-worker1.%node-network-domain%
```

Wait for the debug pod to start:

~~~bash
Starting pod/ocp4-worker1aioexamplecom-debug ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.123.104
If you don't see a command prompt, try pressing enter.
~~~

Now move into the host directory so we can access host binaries/filesystems:

```execute-1
chroot /host
```

Then check the status of our **enp3s0** device:

```execute-1
ip link show dev enp3s0
```

This should show the following, with the key bit showing "**master br1**", so we know it has been consumed into our new bridge:
~~~bash
4: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master br1 state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:00:01:04 brd ff:ff:ff:ff:ff:ff
~~~

Now exit from debug pod, **remembering to execute this twice** to return back to our lab workbook:

```execute-1
exit
```

It should say the following once you've exited the debug pod:

~~~bash
Removing debug pod ...
~~~

Ensure you are in the correct shell:

```execute-1
oc whoami
```

You should see *cnv* service account as previously:
~~~bash
system:serviceaccount:workbook:cnv
~~~


As you can see, *enp3s0* is attached to "*br1*" and has the MAC address of the underlying "physical" adaptor.

Now that the "physical" networking is configured on the underlying worker nodes, we need to then define a `NetworkAttachmentDefinition` so that when we want to use this bridge, OpenShift and OpenShift Virtualization know how to attach into it. We need to ensure that we're in the right project first, i.e. "**default**", ignore if it complains about already being in that project, it's just good practice to check:

```execute-1
oc project default
```

Let's create the `NetworkAttachmentDefinition`, this associates the bridge we just defined with a logical name, known here as '**tuning-bridge-fixed**':


```execute-1
cat << EOF | oc apply -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: tuning-bridge-fixed
  annotations:
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/br1
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "groot",
    "plugins": [
      {
        "type": "cnv-bridge",
        "bridge": "br1"
      },
      {
        "type": "tuning"
      }
    ]
  }'
EOF
```


It should verify the NetworkAttchmentDefinition has been created:

~~~bash
networkattachmentdefinition.k8s.cni.cncf.io/tuning-bridge-fixed created
~~~

> **NOTE**: The important flags to recognise here are the **type**, being **cnv-bridge** which is a specific implementation that links in-VM interfaces to a counterpart on the underlying host for **full-passthrough** of networking. Also note that there is no **IPAM** listed - we don't want the CNI to manage the network address allocation for us instead the network we want to attach to has DHCP enabled.

That's it! We're good to go. Next step is to deploy a virtual machine! Click on "**Deploy Workloads**" to continue with the lab.
