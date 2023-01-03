# In this lab

This lab provides the instructions to:

* [Overview](https://github.com/Pooriya-a/CalicoEnterprise-Networking-Training/blob/main/4.%20Kubernetes%20Services%20and%20CE%20Service%20Advertisement/README.md#overview)



## Overview

In Kubernetes, Ingress traffic refers to any traffic that is initiated from outside the cluster to the services running inside the Kubernetes cluster. Egress traffic is just the opposite, any traffic that is initiated from the pods from within the cluster to the IP endpoints that are located outside the cluster. Kubernetes provides a native ingress resource to manage and control Ingress traffic. However, there is no native Kubernetes Egress resource. So when pods need to connect to an endpoint outside the cluster, they do so using their own IP addresses by default. Considering that an application could be implemented through one or more pods and the fact that pods in Kubernetes are ephemeral, it is almost impossible to control the Kubernetes egress traffic from outside the cluster as the IP addresses are constantly changing. Egress gateways (EGWs) are pods that act as gateways for traffic leaving the cluster from certain client pods. The primary function of egress gateway is to configure the egress gateway client to have a particular and persistent source IP address when connecting to services outside the Kubernetes cluster.


After finshing this lab, you should gain a good understanding of how to deploy Calico Enterprise Egress Gateway and establish a permanent identity for the traffic that is leaving the cluster.

______________________________________________________________________________________________________________________________________________________________________

### 

The purpose of this lab is to demonstrate the funciationality of Egress Gateway. In this lab, we will:

8.1. Enable per namespace Egress Gateway support \
8.2. Deploy an Egress Gateway in *one* namespace \
8.3. Enable BGP with upstream routers (Bird) and advertise the Egress Gateway IP Pool \
8.4. Test and Verify the communication


1. Enable egress gateway support by patching FelixConfiguration to support egress gateway both per namespace and per pod.

```
kubectl patch felixconfiguration.p default --type='merge' -p '{"spec":{"egressIPSupport":"EnabledPerNamespaceOrPerPod"}}'
    
```
2. Egress gateways require the policy sync API to be enabled on Felix to implement symmetric routing. Run the following command to enable this configuration cluster-wide.

```
kubectl patch felixconfiguration.p default --type='merge' -p '{"spec":{"policySyncPathPrefix":"/var/run/nodeagent"}}'
    
```

3. Egress gateways use the IPPool resource for a particular application when it connects outside of the cluster. Run the following command to create the egress gateway IPPool.

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: egress-ippool-1
spec:
  cidr: 10.50.0.0/31
  blockSize: 32
  nodeSelector: "!all()"
EOF
```

4. Let's create a namespace and application, which will be using egress gateway to connect to resources outside the cluster.

```
kubectl apply -f -<<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: app1
  labels:
    tenant: tenant1
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-deployment
  namespace: app1
  labels:
    app: app1
    tenant: tenant1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1
      tenant: tenant1
  template:
    metadata:
      labels:
        app: app1
        tenant: tenant1
    spec:
      containers:
      - name: app1
        image: praqma/network-multitool
        env:
        - name: HTTP_PORT
          value: "1180"
        - name: HTTPS_PORT
          value: "11443"
        ports:
        - containerPort: 1180
          name: http-port
        - containerPort: 11443
          name: https-port
        resources:
          requests:
            cpu: "1m"
            memory: "20Mi"
          limits:
            cpu: "10m"
            memory: "20Mi"
EOF

```

5. Make sure the pods are running.

```
kubectl get pods -n app1

```

6. Egress gateway image needs to be downloaded onto the the nodes where egress gateway pods are deployed. We need to identify the pull secret that is needed for pulling Calico Enterprise images, and copy this into the namespace where you plan to create your egress gateways. Calico Enterprise by default uses a secret named `tigera-pull-secret`. Run the following command to copy `tigera-pull-secret` from `calico-system` namespace to the `app1` namespace.

```
kubectl create secret generic egress-pull-secret --from-file=.dockerconfigjson=/home/tigera/config.json --type=kubernetes.io/dockerconfigjson -n app1

```

7. Deploy the Egress gateway with the desired label to be used as selector by namespace and app workloads using this Egress Gateway. In this example, the label we are using is `egress-code: red`. Please also note the IPPool assigned to this Egress Gateway. In the `cni.projectcalico.org/ipv4pools` annotation, the IPPool can be specified either by its name (e.g. egress-ippool-1) or by its CIDR (e.g. 10.58.0.0/31).


```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: egress-gateway
  namespace: default
  labels:
    egress-code: red
spec:
  replicas: 1
  selector:
    matchLabels:
      egress-code: red
  template:
    metadata:
      annotations:
        cni.projectcalico.org/ipv4pools: "[\"10.10.10.0/31\"]"
      labels:
        egress-code: red
    spec:
      imagePullSecrets:
      - name: tigera-pull-secret
      nodeSelector:
        kubernetes.io/os: linux
      initContainers:
      - name: egress-gateway-init
        command: ["/init-gateway.sh"]
        image: quay.io/tigera/egress-gateway:v3.15.0
        env:
        # Use downward API to tell the pod its own IP address.
        - name: EGRESS_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        securityContext:
          privileged: true
      containers:
      - name: egress-gateway
        command: ["/start-gateway.sh"]
        image: quay.io/tigera/egress-gateway:v3.15.0
        env:
        # Optional: comma-delimited list of IP addresses to send ICMP pings to; if all probes fail, the egress
        # gateway will report non-ready.
        - name: ICMP_PROBE_IPS
          value: ""
        # Only used if ICMP_PROBE_IPS is non-empty: interval to send probes.
        - name: ICMP_PROBE_INTERVAL
          value: "5s"
        # Only used if ICMP_PROBE_IPS is non-empty: timeout before reporting non-ready if there are no successful 
        # ICMP probes.
        - name: ICMP_PROBE_TIMEOUT
          value: "15s"
        # Optional comma-delimited list of HTTP URLs to send periodic probes to; if all probes fail, the egress
        # gateway will report non-ready.
        - name: HTTP_PROBE_URLS
          value: ""
        # Only used if HTTP_PROBE_URL is non-empty: interval to send probes.
        - name: HTTP_PROBE_INTERVAL
          value: "10s"
        # Only used if HTTP_PROBE_URL is non-empty: timeout before reporting non-ready if there are no successful 
        # HTTP probes.
        - name: HTTP_PROBE_TIMEOUT
          value: "30s"
        # Port that the egress gateway serves its health reports.  Must match the readiness probe and health
        # port defined below.
        - name: HEALTH_PORT
          value: "8080"
        - name: EGRESS_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
        volumeMounts:
        - mountPath: /var/run
          name: policysync
        ports:
        - name: health
          containerPort: 8080
        readinessProbe:
          httpGet:
            path: /readiness
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 3
      terminationGracePeriodSeconds: 0
      volumes:
      - csi:
          driver: csi.tigera.io
        name: policysync
EOF

```

## 8.2.2. Connect the namespace to the gateways it should use

```
kubectl annotate ns app1 egress.projectcalico.org/selector='egress-code == "red"'
```

### 8.2.3. Verify both, the POD and Egress Gateway

```
kubectl get pods -n app1 -o wide 
```
```
tigera@bastion:~$ kubectl get pods -n app1 -o wide 
NAME                               READY   STATUS    RESTARTS   AGE   IP            NODE                                         NOMINATED NODE   READINESS GATES
app1-deployment-5bbfd76f9d-2zrdt   1/1     Running   0          7d    10.48.0.80    ip-10-0-1-30.ca-central-1.compute.internal   <none>           <none>
app1-deployment-5bbfd76f9d-pxlx8   1/1     Running   0          7d    10.48.0.203   ip-10-0-1-31.ca-central-1.compute.internal   <none>           <none>
egress-gateway-jc4wm               1/1     Running   0          40s   10.50.0.1     ip-10-0-1-30.ca-central-1.compute.internal   <none>           <none>
egress-gateway-n2zh2               1/1     Running   0          40s   10.50.0.0     ip-10-0-1-31.ca-central-1.compute.internal   <none>           <none>
```

## 8.3. BGP

### 8.3.1. BGP configuration on Calico

Deploy the needed BGP config, so we route our traffic to the bastion host through the egress gateway:

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgppeer-global-64512
spec:
  peerIP: 10.0.1.10
  asNumber: 64512
---
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  serviceClusterIPs:
  - cidr: 10.49.0.0/16
  communities:
  - name: bgp-large-community
    value: 64512:120
  prefixAdvertisements:
  - cidr: 10.50.0.0/31
    communities:
    - bgp-large-community
    - 64512:120
EOF
```

### 8.3.2. Check Calico Nodes connect to the bastion host

Our bastion host is simulating our ToR switch, and it should have BGP sessions established to all nodes:

```
sudo birdc show protocols
```

```
BIRD 1.6.8 ready.
name     proto    table    state  since       info
direct1  Direct   master   up     08:15:34    
kernel1  Kernel   master   up     08:15:34    
device1  Device   master   up     08:15:34    
control1 BGP      master   up     13:24:20    Established   
worker1  BGP      master   up     13:24:20    Established   
worker2  BGP      master   up     13:24:20    Established   
```

If you check the routes, you will see the edge gateway is reachable through the worker node where it has been deployed:

```
ip route
```
```
tigera@bastion:/var/run/bird$ ip route
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.49.0.0/16 proto bird 
        nexthop via 10.0.1.20 dev ens5 weight 1 
        nexthop via 10.0.1.30 dev ens5 weight 1 
        nexthop via 10.0.1.31 dev ens5 weight 1 
10.50.0.0/31 via 10.0.1.31 dev ens5 proto bird 
10.50.0.1 via 10.0.1.30 dev ens5 proto bird 
```

## 8.4. Verification

### 8.4.1. Test and verify Egress Gateway in action

Open a second browser to your lab (`<LABNAME>.lynx.tigera.ca`) if not already done so that we have an additional terminal to test the egress gateway.

On that terminal, start netcat in the bastion host to listen to an specific port:

```
netcat -nvlkp 7777
```

On the original terminal window, exec into any of the pods in the app1 namespace.

```
APP1_POD=$(kubectl get pod -n app1 --no-headers -o name | head -1) && echo $APP1_POD
```
```
kubectl exec -ti $APP1_POD -n app1 -- sh
```

And try to connect to the port in the bastion host.

```
nc -zv 10.0.1.10 7777
```

Type `exit` to exit out the pod terminal.

Go to the terminal that you ran the following netcat server on the bastion node. You should see an output saying you connected from the IP of one of the egress gateway pods to the netcat server.

```
$ sudo tcpdump -i ens5 port 7777
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens5, link-type EN10MB (Ethernet), capture size 262144 bytes
07:51:50.709295 IP ip-10-50-0-0.ca-central-1.compute.internal.41521 > ip-10-0-1-10.ca-central-1.compute.internal.7777: Flags [S], seq 1965935313, win 62727, options [mss 8961,sackOK,TS val 4198299208 ecr 0,nop,wscale 7], length 0
07:51:50.709346 IP ip-10-0-1-10.ca-central-1.compute.internal.7777 > ip-10-50-0-0.ca-central-1.compute.internal.41521: Flags [R.], seq 0, ack 1965935314, win 0, length 0
```

Stop the netcat listener process in the bastion host with `^C`.

### 8.4.2. Routing info on the Calico Node where App Workload is running

Login to worker node where the egress gateway and pods were deployed:

```
ssh worker2
```

Observe the routing policy is programmed for the the App workload POD IP

```
ip rule
```

```
0:      from all lookup local
100:    from 10.48.116.155 fwmark 0x80000/0x80000 lookup 250
32766:  from all lookup main
32767:  from all lookup default
```

Confirm that the policy is choosing the egress gateway as the next hop for any source traffic from App workload POD IP. 
#### Note: ensure to use the correct table number with the following command. In this case, the table number is 250.

```
ip route show table 250
```

```
default onlink 
        nexthop via 10.50.0.0 dev egress.calico weight 1 onlink 
        nexthop via 10.50.0.1 dev egress.calico weight 1 onlink
```

You can close the second browser tab with the terminal now as we will not use it for the rest of the labs.

## Conclusion

In this lab we observed how we can successfully setup egress gateway in a calico cluster and share routing information with external routing device over BGP.

