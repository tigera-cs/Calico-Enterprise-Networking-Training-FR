# In this lab

This lab provides the instructions to:

* [Overview](https://github.com/Pooriya-a/CalicoEnterprise-Networking-Training/blob/main/4.%20Kubernetes%20Services%20and%20CE%20Service%20Advertisement/README.md#overview)
* [Examine pod network connectivity across cluster nodes using CE VXLAN mode](https://github.com/Pooriya-a/CalicoEnterprise-Networking-Training/blob/main/3.%20Cross%20Node%20Connectivity/README.md#examine-pod-network-connectivity-across-cluster-nodes-using-ce-vxlan-mode)
* [Configure an externally routable IPPool](https://github.com/Pooriya-a/CalicoEnterprise-Networking-Training/blob/main/3.%20Cross%20Node%20Connectivity/README.md#configure-an-externally-routable-ippool)
* [Configure Calico Enterprise BGP Peering to connect with an upsteam router outside the cluster](https://github.com/Pooriya-a/CalicoEnterprise-Networking-Training/blob/main/3.%20Cross%20Node%20Connectivity/README.md#configure-calico-enterprise-bgp-peering-to-connect-with-an-upsteam-router-outside-the-cluster)
* [Configure a namespace to use an externally routable IP addresses](https://github.com/Pooriya-a/CalicoEnterprise-Networking-Training/blob/main/3.%20Cross%20Node%20Connectivity/README.md#configure-a-namespace-to-use-an-externally-routable-ip-addresses)

## Overview

Kubernetes is a dynamic environment and pods are not permanent resources. Pods are created and destroyed to match the desired state of your cluster. Since Pods and their IP addresses are ephemeral resources, Kubernetes provides a virtual entity called service as a permanent point of network communication between Pods. Once a service is created, it will stay there until deleted. Services provide a dns name and virtual IP address. In the context of Kubernetes service, pods are called endpoints. Kubernetes uses service controller, which it turn uses labels and selectors to keep the service endpoints update to date as pods are created and destroyed. 

By default, Kubernetes service cluster IPs are accessible only within the cluster and external access to the service requires a dedicated load balancer or ingress controller. Calico Enterprise enables you to advertise Kubernetes service IPs outside a cluster. Calico Enterprise supports advertising a service’s cluster IPs and external IPs. In cases where a service’s cluster IP is not routable, the service can be accessed using its external IP.

After finshing this lab, you should gain a good understanding of Kubernetes services and how to use Calico Enterprise to advertise Kubernetes services.




## 3.1. Kubernetes Services - Basic Lab

This is the 1st lab in a series of labs about k8s services. This lab explores the concepts related to k8s services and the drills down into the iptables chains that enable kube-proxy to deliver the service.

In this lab, you will:

* Examine kubernetes service
* Explore ClusterIP iptables rules
* Explore NodePort iptables rules

### Examine kubernetes services

We will run this lab from `worker1` so we can explore the iptables rules that kube-proxy has set up.

```
ssh worker1
```

Let's take a look at the services and pods in the `yaobank` kubernetes namespace.

```
kubectl get svc -n yaobank
```
```
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
customer   NodePort    10.49.206.189   <none>        80:30180/TCP   19m
database   ClusterIP   10.49.130.110   <none>        2379/TCP       19m
summary    ClusterIP   10.49.198.166   <none>        80/TCP         19m
```

We should have three services deployed. One `NodePort` service and two `ClusterIP` services.

Find the endpoints for each of the services.

```
kubectl get endpoints -n yaobank
```
```
NAME       ENDPOINTS                         AGE
customer   10.48.0.8:80                      20m
database   10.48.128.0:2379                  20m
summary    10.48.128.1:80,10.48.128.192:80   20m
```

List the pods in the yaobank namespace.

```
kubectl get pods -n yaobank -o wide
```

```
NAME                        READY   STATUS    RESTARTS   AGE   IP              NODE                                      NOMINATED NODE   READINESS GATES
customer-68d67b588d-hn95n   1/1     Running   0          21m   10.48.0.8       ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
database-769f6644c5-t925v   1/1     Running   0          21m   10.48.128.0     ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
summary-dc858dd7b-mt5gv     1/1     Running   0          21m   10.48.128.1     ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
summary-dc858dd7b-pkt6l     1/1     Running   0          21m   10.48.128.192   ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
```

You can see that the IP addresses listed as the service endpoints in the previous step map to the backing pods as expected. Each service is backed by one or more pods spread across the nodes in our cluster.

### Explore ClusterIP iptables rules

Let's explore the iptables rules that implement the `summary` service.

Find the service endpoints for `summary` `ClusterIP` service.

```
kubectl get endpoints -n yaobank summary
```

```
NAME      ENDPOINTS                         AGE
summary   10.48.128.1:80,10.48.128.192:80   24m
```

The `summary` service has two endpoints (`110.48.128.1` on port `80` AND `10.48.128.192` on port `80` in this example output). Starting from the `KUBE-SERVICES` iptables chain, we will traverse each chain until you get to the rule directing traffic to these endpoint IP addresses.

#### Examine the KUBE-SERVICE chain

```
sudo iptables -v --numeric --table nat --list KUBE-SERVICES | column -t
```

```
Chain  KUBE-SERVICES  (2                         references)
pkts   bytes          target                     prot         opt  in  out  source         destination
0      0              KUBE-MARK-MASQ             udp          --   *   *    !10.48.0.0/16  10.49.0.10     /*  kube-system/kube-dns:dns                                        cluster  IP          */     udp   dpt:53
0      0              KUBE-SVC-TCOU7JCQXEZGVUNU  udp          --   *   *    0.0.0.0/0      10.49.0.10     /*  kube-system/kube-dns:dns                                        cluster  IP          */     udp   dpt:53
0      0              KUBE-MARK-MASQ             tcp          --   *   *    !10.48.0.0/16  10.49.0.10     /*  kube-system/kube-dns:dns-tcp                                    cluster  IP          */     tcp   dpt:53
0      0              KUBE-SVC-ERIFXISQEP7F7OF4  tcp          --   *   *    0.0.0.0/0      10.49.0.10     /*  kube-system/kube-dns:dns-tcp                                    cluster  IP          */     tcp   dpt:53
0      0              KUBE-MARK-MASQ             tcp          --   *   *    !10.48.0.0/16  10.49.0.10     /*  kube-system/kube-dns:metrics                                    cluster  IP          */     tcp   dpt:9153
0      0              KUBE-SVC-JD5MR3NA4I4DYORP  tcp          --   *   *    0.0.0.0/0      10.49.0.10     /*  kube-system/kube-dns:metrics                                    cluster  IP          */     tcp   dpt:9153
0      0              KUBE-MARK-MASQ             tcp          --   *   *    !10.48.0.0/16  10.49.72.138   /*  ingress-nginx/ingress-nginx-controller-admission:https-webhook  cluster  IP          */     tcp   dpt:443
0      0              KUBE-SVC-EZYNCFY2F7N6OQA2  tcp          --   *   *    0.0.0.0/0      10.49.72.138   /*  ingress-nginx/ingress-nginx-controller-admission:https-webhook  cluster  IP          */     tcp   dpt:443
0      0              KUBE-MARK-MASQ             tcp          --   *   *    !10.48.0.0/16  10.49.206.189  /*  yaobank/customer:http                                           cluster  IP          */     tcp   dpt:80
0      0              KUBE-SVC-PX5FENG4GZJTCELT  tcp          --   *   *    0.0.0.0/0      10.49.206.189  /*  yaobank/customer:http                                           cluster  IP          */     tcp   dpt:80
0      0              KUBE-MARK-MASQ             tcp          --   *   *    !10.48.0.0/16  10.49.198.166  /*  yaobank/summary:http                                            cluster  IP          */     tcp   dpt:80
0      0              KUBE-SVC-OIQIZJVJK6E34BR4  tcp          --   *   *    0.0.0.0/0      10.49.198.166  /*  yaobank/summary:http                                            cluster  IP          */     tcp   dpt:80
0      0              KUBE-MARK-MASQ             tcp          --   *   *    !10.48.0.0/16  10.49.0.1      /*  default/kubernetes:https                                        cluster  IP          */     tcp   dpt:443
0      0              KUBE-SVC-NPX46M4PTMTKRN6Y  tcp          --   *   *    0.0.0.0/0      10.49.0.1      /*  default/kubernetes:https                                        cluster  IP          */     tcp   dpt:443
0      0              KUBE-MARK-MASQ             tcp          --   *   *    !10.48.0.0/16  10.49.149.29   /*  calico-system/calico-typha:calico-typha                         cluster  IP          */     tcp   dpt:5473
0      0              KUBE-SVC-RK657RLKDNVNU64O  tcp          --   *   *    0.0.0.0/0      10.49.149.29   /*  calico-system/calico-typha:calico-typha                         cluster  IP          */     tcp   dpt:5473
0      0              KUBE-MARK-MASQ             tcp          --   *   *    !10.48.0.0/16  10.49.97.116   /*  calico-system/calico-kube-controllers-metrics:metrics-port      cluster  IP          */     tcp   dpt:9094
0      0              KUBE-SVC-KQVGIOWQAVNMB2ZL  tcp          --   *   *    0.0.0.0/0      10.49.97.116   /*  calico-system/calico-kube-controllers-metrics:metrics-port      cluster  IP          */     tcp   dpt:9094
0      0              KUBE-MARK-MASQ             tcp          --   *   *    !10.48.0.0/16  10.49.138.47   /*  calico-apiserver/calico-api:apiserver                           cluster  IP          */     tcp   dpt:443
0      0              KUBE-SVC-I24EZXP75AX5E7TU  tcp          --   *   *    0.0.0.0/0      10.49.138.47   /*  calico-apiserver/calico-api:apiserver                           cluster  IP          */     tcp   dpt:443
0      0              KUBE-MARK-MASQ             tcp          --   *   *    !10.48.0.0/16  10.49.130.110  /*  yaobank/database:http                                           cluster  IP          */     tcp   dpt:2379
0      0              KUBE-SVC-AE2X4VPDA5SRYCA6  tcp          --   *   *    0.0.0.0/0      10.49.130.110  /*  yaobank/database:http                                           cluster  IP          */     tcp   dpt:2379
1502   90196          KUBE-NODEPORTS             all          --   *   *    0.0.0.0/0      0.0.0.0/0      /*  kubernetes                                                      service  nodeports;  NOTE:  this  must      be  the  last  rule  in  this  chain  */  ADDRTYPE  match  dst-type  LOCAL
```

Each iptables chain consists of a list of rules that are executed in order until a rule matches. The key columns/elements to note in this output are:
* `target` - which chain iptables will jump to if the rule matches
* `prot` - the protocol match criteria
* `source`, and `destination` - the source and destination IP address match criteria
* the comments that kube-proxy inculdes
* the additional match criteria at the end of each rule - e.g `dpt:80` that specifies the destination port match

#### KUBE-SERVICES -> KUBE-SVC-XXXXXXXXXXXXXXXX

Now let's look more closely at the rules for the `summary` service.

```
sudo iptables -v --numeric --table nat --list KUBE-SERVICES | grep -E summary | column -t
```

```
0  0  KUBE-MARK-MASQ             tcp  --  *  *  !10.48.0.0/16  10.49.198.166  /*  yaobank/summary:http  cluster  IP  */  tcp  dpt:80
0  0  KUBE-SVC-OIQIZJVJK6E34BR4  tcp  --  *  *  0.0.0.0/0      10.49.198.166  /*  yaobank/summary:http  cluster  IP  */  tcp  dpt:80
```

The second rule directs traffic destined for the summary service clusterIP (`10.49.198.166` in the example output) to the chain that load balances the service (KUBE-SVC-XXXXXXXXXXXXXXXX).

#### KUBE-SVC-XXXXXXXXXXXXXXXX -> KUBE-SEP-XXXXXXXXXXXXXXXX

`kube-proxy` in `iptables` mode uses an equal probability algorithm to load balance traffic between pods.  We currently have two `summary` pods.

Let's examine how this loadbalancing works. (Remember your chain name may be different than this example.)

```
sudo iptables -v --numeric --table nat --list KUBE-SVC-OIQIZJVJK6E34BR4 | column -t
```

```
Chain  KUBE-SVC-OIQIZJVJK6E34BR4  (1                         references)
pkts   bytes                      target                     prot         opt  in  out  source     destination
0      0                          KUBE-SEP-NCOI6JAXVX2D4BXJ  all          --   *   *    0.0.0.0/0  0.0.0.0/0    /*  yaobank/summary:http  */  statistic  mode  random  probability  0.50000000000
0      0                          KUBE-SEP-TVZHKMXMBG2BIHXS  all          --   *   *    0.0.0.0/0  0.0.0.0/0    /*  yaobank/summary:http  */
```

Notice that `kube-proxy` is using the `iptables` `statistic` module to set the probability for a packet to be randomly matched.  Make sure you scroll all the way to the right to see the full output.

The first rule directs traffic destined for the `summary` service to the chain that delivers packets to the first service endpoint (KUBE-SEP-NCOI6JAXVX2D4BXJ) with a probability of 0.50000000000. The second rule unconditionally directs to the second service endpoint chain (KUBE-SEP-TVZHKMXMBG2BIHXS). The result is that traffic is load balanced across the service endpoints equally (on average).

If there were 3 service endpoints, then the first chain match would be probability 0.33333333, the second probability 0.5, and the last unconditional. The result is each service endpoint receives a third of the traffic (on average).

#### KUBE-SEP-XXXXXXXXXXXXXXXX -> `summary` pod

Let's look at one of the service endpoint chains. (Remember your chain names may be different than this example.)
```
sudo iptables -v --numeric --table nat --list KUBE-SEP-NCOI6JAXVX2D4BXJ | column -t
```

```
Chain  KUBE-SEP-NCOI6JAXVX2D4BXJ  (1              references)
pkts   bytes                      target          prot         opt  in  out  source         destination
0      0                          KUBE-MARK-MASQ  all          --   *   *    10.48.128.192  0.0.0.0/0    /*  yaobank/summary:http  */
0      0                          DNAT            tcp          --   *   *    0.0.0.0/0      0.0.0.0/0    /*  yaobank/summary:http  */  tcp  to:10.48.128.192:80
```

The second rule performs the DNAT that changes the destination IP from the service's clusterIP to the IP address of the service endpoint backing pod (`10.48.128.192` in this example). After this, standard Linux routing can handle forwarding the packet like it would for any other packet.

#### Recap

You've just traced the kube-proxy iptables rules used to load balance traffic to `summary` pods exposed as a service of type `ClusterIP`.

In summary, for a packet being sent to a clusterIP:
* The KUBE-SERVICES chain matches on the clusterIP and jumps to the corresponding KUBE-SVC-XXXXXXXXXXXXXXXX chain.
* The KUBE-SVC-XXXXXXXXXXXXXXXX chain load balances the packet to a random service endpoint KUBE-SEP-XXXXXXXXXXXXXXXX chain.
* The KUBE-SEP-XXXXXXXXXXXXXXXX chain DNATs the packet so it will get routed to the service endpoint (backing pod).

### Explore NodePort iptables rules

Let's explore the iptables rules that implement the `customer` service.

Find the service endpoints for `customer` `NodePort` service.
```
kubectl get endpoints -n yaobank customer
```

```
NAME       ENDPOINTS      AGE
customer   10.48.0.8:80   42m
```

The `customer` service has one endpoint (`10.48.0.8` on port `80` in this example output). Starting from the `KUBE-SERVICES` iptables chain, we will traverse each chain until you get to the rule directing traffic to this endpoint IP address.

#### KUBE-SERVICES -> KUBE-NODEPORTS

The `KUBE-SERVICE` chain handles the matching for service types `ClusterIP` and `LoadBalancer`. At the end of `KUBE-SERVICE` chain, another custom chain `KUBE-NODEPORTS` will handle traffic for service type `NodePort`.

```
sudo iptables -v --numeric --table nat --list KUBE-SERVICES | grep KUBE-NODEPORTS | column -t
```

```
2448  147K  KUBE-NODEPORTS  all  --  *  *  0.0.0.0/0  0.0.0.0/0  /*  kubernetes  service  nodeports;  NOTE:  this  must  be  the  last  rule  in  this  chain  */  ADDRTYPE  match  dst-type  LOCAL
```

`match dst-type LOCAL` matches any packet with a local host IP as the destination. i.e. any address that is assigned to one of the host's interfaces.

#### KUBE-NODEPORTS -> KUBE-SVC-XXXXXXXXXXXXXXXX

```
sudo iptables -v --numeric --table nat --list KUBE-NODEPORTS | column -t
```

```
Chain  KUBE-NODEPORTS  (1                         references)
pkts   bytes           target                     prot         opt  in  out  source     destination
0      0               KUBE-MARK-MASQ             tcp          --   *   *    0.0.0.0/0  0.0.0.0/0    /*  yaobank/customer:http  */  tcp  dpt:30180
0      0               KUBE-SVC-PX5FENG4GZJTCELT  tcp          --   *   *    0.0.0.0/0  0.0.0.0/0    /*  yaobank/customer:http  */  tcp  dpt:30180
```

The second rule directs traffic destined for the `customer` service to the chain that load balances the service (KUBE-SVC-PX5FENG4GZJTCELT). `tcp dpt:30180` matches any packet with the destination port of tcp 30180 (the node port of the `customer` service).

#### KUBE-SVC-XXXXXXXXXXXXXXXX -> KUBE-SEP-XXXXXXXXXXXXXXXX
(Remember your chain name may be different than this example.)

```
sudo iptables -v --numeric --table nat --list KUBE-SVC-PX5FENG4GZJTCELT | column -t
```

```
Chain  KUBE-SVC-PX5FENG4GZJTCELT  (2                         references)
pkts   bytes                      target                     prot         opt  in  out  source     destination
0      0                          KUBE-SEP-M7DMW2CXWLD73RC3  all          --   *   *    0.0.0.0/0  0.0.0.0/0    /*  yaobank/customer:http  */
```

As we only have a single backing pod for the `customer` service, there is no loadbalancing to do, so there is a single rule that directs all traffic to the chain that delivers the packet to the service endpoint (KUBE-SEP-XXXXXXXXXXXXXXXX).

#### KUBE-SEP-XXXXXXXXXXXXXXXX -> `customer` endpoint
(Remember your chain name may be different than this example.)

```
sudo iptables -v --numeric --table nat --list KUBE-SEP-M7DMW2CXWLD73RC3 | column -t
```

```
Chain  KUBE-SEP-M7DMW2CXWLD73RC3  (1              references)
pkts   bytes                      target          prot         opt  in  out  source     destination
0      0                          KUBE-MARK-MASQ  all          --   *   *    10.48.0.8  0.0.0.0/0    /*  yaobank/customer:http  */
0      0                          DNAT            tcp          --   *   *    0.0.0.0/0  0.0.0.0/0    /*  yaobank/customer:http  */  tcp  to:10.48.0.8:80
```

This rule delivers the packet to the `customer` service endpoint.

The second rule performs the DNAT that changes the destination IP from the service's NodePort to the IP address of the service endpoint backing pod (`10.48.0.8` in this example). After this, standard Linux routing can handle forwarding the packet like it would for any other packet.

#### Recap

You've just traced the kube-proxy iptables rules used to load balance traffic to `customer` pods exposed as a service of type `NodePort`.

In summary, for a packet being sent to a NodePort:

* The end of the KUBE-SERVICES chain jumps to the KUBE-NODEPORTS chain
* The KUBE-NODEPORTS chaing matches on the NodePort and jumps to the corresponding KUBE-SVC-XXXXXXXXXXXXXXXX chain.
* The KUBE-SVC-XXXXXXXXXXXXXXXX chain load balances the packet to a random service endpoint KUBE-SEP-XXXXXXXXXXXXXXXX chain.
* The KUBE-SEP-XXXXXXXXXXXXXXXX chain DNATs the packet so it will get routed to the service endpoint (backing pod).

> __Congratulations! You have completed this la and you have by now a basic understanding of kubernetes services.__

