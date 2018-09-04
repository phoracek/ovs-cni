# Demo

This demo will show you how to configure Open vSwitch bridges on your nodes,
create Kubernetes networks on top of them and use them by pods.

## Requirements

Before we start, make sure that you have your Kubernetes/OpenShift cluster
ready with OVS. In order to do that, you can follow guides of deployment on
[local cluster](deployment-on-local-cluster.md) or your [arbitrary
cluster](deployment-on-arbitrary-cluster.md).

## Create Open vSwitch bridge

First of all, we need to create OVS bridges on all nodes.

```shell
# on all nodes
ovs-vsctl add-br br1
```

This bridge on its own has no access to other nodes and therefore pods
connected to the same bridge on different nodes will not have connectivity to
one another. If you have just a single node, you can skip the rest of this
section and continue to [Creating network attachment
definition](#create-network-attachment-definition).

### Connect bridges using dedicated L2

The easiest way how to interconnect OVS bridge is using L2 connection on
dedicated interface (not used by node itself, without configured IP address).
If you have spare dedicated interface available, connect it as a port to OVS
bridge.

```shell
# on all nodes
ovs-vsctl add-port br1 eth1
```

### Connect bridges using shared L2

You can also share node management interface with OVS bridge. Before you do
that, please keep in mind, that you might lose your connectivity for a moment
as we will move IP address from the interface to the bridge. I would recommend
you doing this using console.

This process depends on your platform, following command is only an
illustrative example and it might break your system.

```shell
# on all nodes
## get original interface IP address
ip address show eth0
## flush original IP address
ip address flush eth0
## attach the interface to OVS bridge
ovs-vsctl add-port br1 eth0
## set the original IP address on the bridge
ip address add $ORIGINAL_INTERFACE_IP_ADDRESS dev br0
```

### Connect bridges using VXLAN

Lastly, we can connect OVS bridges using VXLAN tunnels. In order to do so,
create VXLAN port on OVS bridge per each remote node with `options:remote_ip`
set to the remote node IP address.

```shell
# on node01
REMOTE_IP="node02 IP address" ovs-vsctl add-port br1 vtep -- set Interface vxlan type=vxlan options:remote_ip=$REMOTE_IP

# on node02
REMOTE_IP="node01 IP address" ovs-vsctl add-port br1 vtep -- set Interface vxlan type=vxlan options:remote_ip=$REMOTE_IP
```

## Create Network Attachment Definition

With Multus, secondary networks are specified using
`NetworkAttachmentDefinition` objects. Such objects contain information that is
passed to ovs-cni plugin on nodes during pod configuration. Such networks can
be referenced from pod using their names. If this object, we specify name of
the bridge and optionaly a VLAN tag.

First, let's create a simple OVS network that will connect pods in trunk mode.

```shell
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: ovs-net-1
spec:
  config: '{
      "cniVersion": "0.3.1",
      "type": "ovs",
      "bridge": "br1"
    }'
EOF
```

Now create another network connected to the same bridge, this time with VLAN
tag specified.

```shell
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: ovs-net-2-vlan
spec:
  config: '{
      "cniVersion": "0.3.1",
      "type": "ovs",
      "bridge": "br1",
      "vlan": 100
    }'
EOF
```

## Attach a Pod to the network

Now when everything is ready, we can create a Pod and connect it to OVS
networks.

```shell
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: samplepod-1
  annotations:
    k8s.v1.cni.cncf.io/networks: ovs-net-1,ovs-net-2-vlan
spec:
  containers:
  - name: samplepod
    command: ["sleep", "99999"]
    image: alpine
EOF
```

We can login into the Pod to verify that secondary interfaces were created.

```shell
kubectl exec samplepod-1 -- ip link show
```

## Configure IP address

Open vSwitch CNI does not support IPAM yet. In order to test IP connectivity
between Pods, we need to configure IP manually.

Create second Pod, this time it will configure its own IP address. Note that in
order to configure IP address, this Pod must be in privileged mode.

```shell
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: samplepod-2
  annotations:
    k8s.v1.cni.cncf.io/networks: ovs-net-2-vlan
spec:
  containers:
  - name: samplepod
    command: ["sh", "-c", "ip address add 11.0.0.2/24 dev net1; sleep 99999"]
    image: alpine
    securityContext:
      privileged: true
EOF
```

Create another Pod so we have something to ping.

```shell
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: samplepod-3
  annotations:
    k8s.v1.cni.cncf.io/networks: ovs-net-2-vlan
spec:
  containers:
  - name: samplepod
    command: ["sh", "-c", "ip address add 11.0.0.3/24 dev net1; sleep 99999"]
    image: alpine
    securityContext:
      privileged: true
EOF
```

Once both Pods are up and running, we can try to ping from one to another.

```shell
kubectl exec -it samplepod-2 -- ping 11.0.0.3
```

## Scheduling

If your OVS bridges are available only on certain nodes, scheduling must be
involved. It is not yet natively implemented in ovs-cni, but you can use
Kubernetes node labels and pod label selectors to do so. If that is your case,
follow [scheduling guide](scheduling.md).
