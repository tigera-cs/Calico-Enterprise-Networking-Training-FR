## 2.1. Networking: Pod Connectivity - Basic Lab

This lab is the first of a series of labs exploring k8s networking concepts. This lab focuses on basic k8s networking from the POD and Host perspectives.   
In this lab, you will:
* Examine what the network looks like from the perspecitve of a pod (the pod network namespace)
* Examine what the network looks like from the perspecitve of the host (the host network namespace)

### Before you begin

The prerequisite to this lab is completing Lab1, which is installing Calico CNI and deploying the sample application Yaobank.


### Examine pod network namespace

We'll start by examining what the network looks like from the pod's point of view. Each pod get's its own Linux network namespace, which you can think of as giving it an isolated copy of the Linux networking stack. 


Let's find the name and location of the customer pod using the following command.

```
kubectl get pods -n yaobank -l app=customer -o wide
```
```
NAME                        READY   STATUS    RESTARTS   AGE     IP          NODE                                      NOMINATED NODE   READINESS GATES
customer-68d67b588d-w5zhr   1/1     Running   0          8m58s   10.48.0.7   ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
```

Note the node on which the pod is running on (`ip-10-0-1-30.eu-west-1.compute.internal` in this example.)

Use kubectl to exec into the pod so we can check the pod networking details. 

```
kubectl exec -ti -n yaobank $(kubectl get pods -n yaobank -l app=customer -o name) bash
```


First we will use `ip addr` to list the addresses and associated network interfaces that the pod sees.
```
ip addr
```
```
root@customer-68d67b588d-w5zhr:/app# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if13: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default 
    link/ether 8e:14:b1:d0:60:ad brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.48.0.7/32 scope global eth0
       valid_lft forever preferred_lft forever
```

The key things to note in this output are:
* There is a `lo` loopback interface with an IP address of `127.0.0.1`. This is the standard loopback interface that every network namespace has by default. You can think of it as `localhost` for the pod itself.
* There is an `eth0` interface which has the pods actual IP address, `10.48.0.7`. Notice this matches the IP address that `kubectl get pods` returned earlier.

Next let's look more closely at the interfaces using `ip link`.  We will use the `-c` option, which colours the output to make it easier to read.

```
ip -c link
```
```
root@customer-68d67b588d-w5zhr:/app# ip -c link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: eth0@if13: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default 
    link/ether 8e:14:b1:d0:60:ad brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

Look at the `eth0` part of the output. The key things to note are:
* The `eth0` interface is interface number 3 in the pod's network namespace.
* `eth0` is a link to the host network namespace (indicated by `link-netnsid 0`). i.e. It is the pod's side of the veth pair (virtual ethernet pair) that connects the pod to the host's networking.
* The `@if13` at the end of the interface name is the interface number of the other end of the veth pair within the host's network namespaces. In this example, interface number 13.  Remember this for later. We will take look at the other end of the veth pair shortly.

Finally, let's look at the routes the pod sees.

```
ip route
```
```
root@customer-68d67b588d-w5zhr:/app# ip route
default via 169.254.1.1 dev eth0 
169.254.1.1 dev eth0  scope link 
```
This shows that the pod's default route is out over the `eth0` interface. i.e. Anytime it wants to send traffic to anywhere other than itself, it will send the traffic over `eth0`.

We've finished our tour of the pod's view of the network, so we'll exit out of the exec to return to bastion host.

```
exit
```

### Examine the host's network namespace

We'll start by switching to the node where the customer pod is running. In our example earlier this was `ip-10-0-1-30.eu-west-1.compute.internal`, which refers to worker1. SSH into worker1. (Please note that the your customer pod might run on a different node. SSH into the worker node where customer pod is running in your lab)

```
ssh worker1
```

Now we're on the node hosting the customer pod. we'll examine the other end of the veth pair. In our example output earlier, the `@if13` indicated it should be interface number 13 in the host network namespace. (Your interface numbers may be different, but you should be able to follow along the same logic.)
```
ip -c link
```
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 02:58:11:57:a4:ef brd ff:ff:ff:ff:ff:ff
    altname enp0s5
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default 
    link/ether 02:42:7b:4d:bf:b8 brd ff:ff:ff:ff:ff:ff
7: calid1d8cf42e5f@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 1
9: calief1cea5cba1@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 0
11: cali9feb40e484d@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 2
12: cali0053d475d0d@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 3
13: calicd9cc437aea@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 4
```

Looking at interface number 13 in this example we see `calicd9cc437aea` which links to `@if3` in network namespace ID 4 (the customer pod's network namespace).  You may recall that interface 3 in the pod's network namespace was `eth0`, so this looks exactly as expected for the veth pair that connects the customer pod to the host network namespace.  

You can also see the host end of the veth pairs to other pods running on this node, all beginning with `cali`.


Let examine the node routing table. First let's remind ourselves of the `customer` pod's IP address.

```
kubectl get pods -n yaobank -l app=customer -o wide
````
```
NAME                        READY   STATUS    RESTARTS   AGE   IP          NODE                                      NOMINATED NODE   READINESS GATES
customer-68d67b588d-w5zhr   1/1     Running   0          53m   10.48.0.7   ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
```

Now lets look at the routes on the node, where the customer pod is running. SSH into the node.

```
ip route
```
```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.30 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.30 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.30 metric 100 
blackhole 10.48.0.0/26 proto bird 
10.48.0.1 dev calid1d8cf42e5f scope link 
10.48.0.3 dev calief1cea5cba1 scope link 
10.48.0.5 dev cali9feb40e484d scope link 
10.48.0.6 dev cali0053d475d0d scope link 
10.48.0.7 dev calicd9cc437aea scope link 
10.48.0.192/26 via 10.0.1.31 dev ens5 proto bird 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
```

In this example output, we can see the route to the customer pod's IP `10.48.0.7` is via the `calicd9cc437aea` interface, the host end of the veth pair for the customer pod. You can see similar routes for each of the IPs of the other pods hosted on this node. It's these routes that tell Linux where to send traffic that is destined to a local pod on the node.

We can also see several routes labelled `proto bird`. These are routes to pods on other nodes that Calico has learned over BGP. 

To understand these better, consider this route in the example output above `10.48.0.192/26 via 10.0.1.31 dev ens5 proto bird `.  It indicates pods with IP addresses falling within the `10.48.0.192/26` CIDR can be reached via `10.0.1.31` (which is worker2) through the `ens5` network interface (the host's main interface to the rest of the network). You should see similar routes in your output for each node.

Calico uses route aggregation to reduce the number of routes when possible. (e.g. `/26` in this example). The `/26` corresponds to the default block size that Calico IPAM (IP Address Management) allocates on demand as nodes need pod IP addresses. (If desired, the block size can be configured in Calico IPAM settings.)  

You can also see the `blackhole 10.48.0.0/26 proto bird` route. The `10.48.0.0/26` corresponds to the block of IPs that Calico IPAM allocated on demand for this node. This is the block from which each of the local pods got their IP addresses. The blackhole route tells Linux that if it can't find a more specific route for an individual IP in that block then it should discard the packet (rather than sending it out the default route to the network). You will only see traffic that hits this rule if something is trying to send traffic to a pod IP that doesn't exist, for example sending traffic to a recently deleted pod.

If Calico IPAM runs out of blocks to allocate to nodes, then it will use unused IPs from other nodes' blocks. These will be announced over BGP as more specific routes, so traffic to pods will always find its way to the right host.


We've finished our tour of the Customer pod's host's view of the network. Remember exit back to the bastion node.
```
exit
```

> __You have completed Lab2.1 and have by now a good understanding of Pod and Host networking.__

