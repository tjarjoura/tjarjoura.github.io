---
layout: post
title: "Breaking A Kubernetes Service in Three Ways And Examining The Raw Packets"
date: 2026-04-24
---

A Kubernetes service is simple conceptually. You get a stable endpoint which load balances traffic across a pool of backend pods. Under the hood it's implemented by standard Linux kernel features -- DNAT iptables rules, virtual bridges, and veth pairs. There's no separate proxy process implementing the logic for a Kubernetes service, so when it breaks, the errors are just going to be standard Linux networking errors.

To demonstrate this, I'm going to intentionally break a Kubernetes service in three different ways, and examine the raw packet captures from each.

# Set Up
For the set up, I created a simple deployment which runs a single instance of `socat`, and I put a `NodePort` Kubernetes service in front of it. This is all running on a single node instance of  `k3d`, which means the service is ultimately exposed at `<k3d node>:<node port>`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
  labels:
    app: socat
  name: socat
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: socat
  template:
    metadata:
      labels:
        app: socat
    spec:
      containers:
      - args:
        - TCP-LISTEN:12345,fork,reuseaddr
        - EXEC:/bin/cat
        command:
        - socat
        image: alpine/socat:latest
        imagePullPolicy: Always
        name: socat
        ports:
        - containerPort: 12345
          protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: socat
  name: socat
  namespace: default
spec:
  ports:
  - nodePort: 31234
    port: 12345
    protocol: TCP
  type: NodePort
```

The `socat` process in the pod listens on port `12345`, and we want packets to the `k3d` node at port `31234` to be routed there.

As a quick demonstration of what this `socat` command does, it just echoes back whatever is sent to it on a TCP connection. I can `telnet` into the node port and get whatever I send echo'd back. Note the `127.0.0.1` is because I have the k3d docker container port forwarded to localhost on my MacBook host network:

```
tyler@MacBookPro sysobs % telnet 127.0.0.1 31234
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
hello
hello
world 
world
```

Under the hood, Kubernetes services are implemented by `EndpointSlice` resources, which are translated by `kube-proxy` into `iptables`/`netfilter` rules that the Linux Kernel can use directly. Essentially, these rules match packets for specific IP address / port tuples and then use network address translation to rewrite the destination address on the packet.

The `EndpointSlice` typically gets updated dynamically by a controller based on the `selector` field on the service, so you usually don't have to touch them. For the purposes of this post, I removed the `selector` field from the service and created the `EndpointSlice` manually.

# Happy Path
I'll set the correct pod IP and correct port in the `EndpointSlice` spec. From `kubectl get pod` we can see the pod's IP address:

```
socat-5c97445bc5-vw5dz       1/1     Running   0          23h     10.42.0.142   k3d-k3s-default-server-0   <none>           <none>
```

Which just needs to be set on the `EndpointSlice`, along with the correct port:

```
addressType: IPv4
apiVersion: discovery.k8s.io/v1
endpoints:
- addresses:
  - 10.42.0.142
  conditions:
    ready: true
kind: EndpointSlice
metadata:
  labels:
    app: socat
    kubernetes.io/service-name: socat
  name: socat
  namespace: default

ports:
- name: ""
  port: 12345
  protocol: TCP
```

This creates the following chain of IPTables rules:

```
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/socat" -m tcp --dport 31234 -j KUBE-EXT-6GHM6NOKDRAB634Q

...

-A KUBE-SEP-STAYFALK6DPGANUO -p tcp -m comment --comment "default/socat" -m tcp -j DNAT --to-destination 10.42.0.142:12345
```

I removed some of the intermediate rules to simplify, but basically -- this matches packets coming in to port `31234` (the `nodePort` on our service) and changes the destination address on them to `10.42.0.142:12345`. 

And if I attempt a TCP connection against the `NodePort` service, this is packet capture:
```
+0.401s  [pktcap]           capture-started  default/socat bpf="(ip and ((host 10.43.161.224 and port 12345) or ((net 10.42.0.0/24) and port 12345) or ((host 192.168.97.3) and port 31234))) or (arp and (host 10.42.0.142))" link=Linux SLL2
+3.519s  [pktcap]           tcp-packet  [eth0] 192.168.97.2:54398 -> 192.168.97.3:31234 SYN
+3.519s  [pktcap]           tcp-packet  [cni0] 10.42.0.1:10639 -> 10.42.0.142:12345 SYN
+3.519s  [pktcap]           tcp-packet  [veth0829e839] 10.42.0.1:10639 -> 10.42.0.142:12345 SYN
+3.519s  [pktcap]           tcp-packet  [veth0829e839] 10.42.0.142:12345 -> 10.42.0.1:10639 SYN,ACK
+3.519s  [pktcap]           tcp-packet  [cni0] 10.42.0.142:12345 -> 192.168.97.2:54398 SYN,ACK
+3.519s  [pktcap]           tcp-packet  [eth0] 192.168.97.3:31234 -> 192.168.97.2:54398 SYN,ACK
+3.519s  [pktcap]           tcp-packet  [eth0] 192.168.97.2:54398 -> 192.168.97.3:31234 ACK
+3.519s  [pktcap]           tcp-packet  [cni0] 10.42.0.1:10639 -> 10.42.0.142:12345 ACK
+3.519s  [pktcap]           tcp-packet  [veth0829e839] 10.42.0.1:10639 -> 10.42.0.142:12345 ACK
```

This is just a standard TCP handshake, but we see three different network interfaces being traversed for each packet. The first is `eth0`, which is the "external" interface of the `k3d` node. The packet is addressed to `192.168.97.3:31234`, which is the `k3d` node IP and node port.

Next we see the same `SYN` packet, this time on the `cni0` interface, the virtual bridge connecting every pod on the node, and the destination address has been changed to `10.42.0.142:12345`, which is the pod's IP and listening port. This is the direct effect of the DNAT iptables rule we looked at above, and the packet was routed to `cni0` because it now belonged to the `10.42.0.0/24` network. 

Notice also that the source address changed, which is another effect of the iptables rules that `kube-proxy` installs, which is then reversed on the return path.

Finally, each pod has its own `veth` pair which bridges the pod's network namespace to the host's network namespace. The host end of the pair is connected to the `cni0` virtual bridge, so the packet is re-transmitted to this interface.

When the server sends packets back to the client, the three interfaces are traversed in reverse order (`veth` -> `cni0` -> `eth0`), as expected.

# Break #1: Wrong Port
A simple way to break the service is to point at a port nothing in the pod is listening on. Since we can modify the `EndpointSlice` however we want, we can just change the port on it:
```
ports:
- name: ""
  port: 12346 # changed from 12345
  protocol: TCP
```

This changes the iptables rule to look like:
```
-A KUBE-SEP-XVRNL33HWWUQHJC7 -p tcp -m comment --comment "default/socat" -m tcp -j DNAT --to-destination 10.42.0.142:12346
```

Nothing is listening on this port, so let's see the packet capture:
```
+0.626s  [pktcap]           capture-started  default/socat bpf="(ip and ((host 10.43.161.224 and port 12345) or ((net 10.42.0.0/24) and port 12346) or ((host 192.168.97.3) and port 31234))) or (arp and (host 10.42.0.142))" link=Linux SLL2
+50.032s  [pktcap]           tcp-packet  [eth0] 192.168.97.2:58364 -> 192.168.97.3:31234 SYN
+50.032s  [pktcap]           tcp-packet  [cni0] 10.42.0.1:6999 -> 10.42.0.142:12346 SYN
+50.032s  [pktcap]           tcp-packet  [veth0829e839] 10.42.0.1:6999 -> 10.42.0.142:12346 SYN
+50.032s  [pktcap]           tcp-packet  [veth0829e839] 10.42.0.142:12346 -> 10.42.0.1:6999 ACK,RST
+50.032s  [pktcap]           tcp-packet  [cni0] 10.42.0.142:12346 -> 192.168.97.2:58364 ACK,RST
+50.032s  [pktcap]           tcp-packet  [eth0] 192.168.97.3:31234 -> 192.168.97.2:58364 ACK,RST
```

Since the IP address on the EndpointSlice is still correct, the packet still gets routed to the correct pod veth pair, but since nothing inside the pod's network namespace is listening on port `12346`, a TCP RST packet is sent back through the interfaces to the client.

This typically manifests on the client as a "connection refused" error.

# Break #2: Wrong IP Address
Again, we can just update the IP address on the EndpointSlice directly:
```
endpoints:
- addresses:
  - 10.42.0.143
  conditions:
    ready: true
```

Which changes the iptables rule to look like:
```
-A KUBE-SEP-L7Q23S57UQHHMQS4 -p tcp -m comment --comment "default/socat" -m tcp -j DNAT --to-destination 10.42.0.143:12345
```

Which results in the following packet capture:
```
+0.564s  [pktcap]           capture-started  default/socat bpf="(ip and ((host 10.43.161.224 and port 12345) or ((net 10.42.0.0/24) and port 12345) or ((host 192.168.97.3) and port 31234))) or (arp and (host 10.42.0.143))" link=Linux SLL2
+7.389s  [pktcap]           tcp-packet  [eth0] 192.168.97.2:47514 -> 192.168.97.3:31234 SYN
+7.389s  [pktcap]           arp-request  [cni0] arp who-has 10.42.0.143 tell 10.42.0.1
+7.389s  [pktcap]           arp-request  [veth0829e839] arp who-has 10.42.0.143 tell 10.42.0.1
+7.389s  [pktcap]           arp-request  [veth261c8b4a] arp who-has 10.42.0.143 tell 10.42.0.1
+7.389s  [pktcap]           arp-request  [vethb5026255] arp who-has 10.42.0.143 tell 10.42.0.1
+7.389s  [pktcap]           arp-request  [vethf88de69a] arp who-has 10.42.0.143 tell 10.42.0.1
+7.389s  [pktcap]           arp-request  [vethfd2cfe3e] arp who-has 10.42.0.143 tell 10.42.0.1
+7.389s  [pktcap]           arp-request  [veth03bb59f4] arp who-has 10.42.0.143 tell 10.42.0.1
+7.389s  [pktcap]           arp-request  [veth71d61d24] arp who-has 10.42.0.143 tell 10.42.0.1
+7.389s  [pktcap]           arp-request  [veth683aa91d] arp who-has 10.42.0.143 tell 10.42.0.1
```

After the packet's destination address changes, the Kernel attempts to discover the corresponding MAC address for this IP, and broadcasts an ARP `who-has` request on the pod bridge network. The `veth` pair for each pod on the node receives a copy of the ARP request, and since the IP address isn't attached to any of the pods, nobody is able to reply, and the TCP connection is unable to complete.

From the client you're not going to see an "ARP failed" error message, you'll probably just see it hanging while it retries the TCP handshake until it gives up with a timeout.

# Break #3: A Mix of Good and Bad IP Addresses
An `EndpointSlice` can contain multiple pod IP addresses, so let's put the correct address alongside the wrong one:
```
endpoints:
- addresses:
  - 10.42.0.143
  conditions:
    ready: true
- addresses:
  - 10.42.0.142
  conditions:
    ready: true
```

This creates iptables rules with a probabilistic load balancing effect:
```
-A KUBE-SVC-6GHM6NOKDRAB634Q -m comment --comment "default/socat -> 10.42.0.142:12345" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-STAYFALK6DPGANUO
-A KUBE-SVC-6GHM6NOKDRAB634Q -m comment --comment "default/socat -> 10.42.0.143:12345" -j KUBE-SEP-L7Q23S57UQHHMQS4
```

And now the packet capture shows only intermittent failures:
```
+20.003s  [tcp-probe]        io-error  127.0.0.1:57014 -> 127.0.0.1:31234 read: read tcp 127.0.0.1:57014->127.0.0.1:31234: i/o timeout
+20.064s  [pktcap]           arp-request  [cni0] arp who-has 10.42.0.143 tell 10.42.0.1
+20.064s  [pktcap]           arp-request  [veth0829e839] arp who-has 10.42.0.143 tell 10.42.0.1
+20.064s  [pktcap]           arp-request  [veth261c8b4a] arp who-has 10.42.0.143 tell 10.42.0.1
+20.064s  [pktcap]           arp-request  [vethb5026255] arp who-has 10.42.0.143 tell 10.42.0.1
+20.064s  [pktcap]           arp-request  [vethf88de69a] arp who-has 10.42.0.143 tell 10.42.0.1
+20.064s  [pktcap]           arp-request  [vethfd2cfe3e] arp who-has 10.42.0.143 tell 10.42.0.1
+20.064s  [pktcap]           arp-request  [veth03bb59f4] arp who-has 10.42.0.143 tell 10.42.0.1
+20.064s  [pktcap]           arp-request  [veth71d61d24] arp who-has 10.42.0.143 tell 10.42.0.1
+20.064s  [pktcap]           arp-request  [veth683aa91d] arp who-has 10.42.0.143 tell 10.42.0.1
+21.007s  [tcp-probe]        echo-ok  127.0.0.1:57015 -> 127.0.0.1:31234 connect=0ms echo=4ms (5B)
+21.005s  [pktcap]           tcp-packet  [eth0] 192.168.97.2:46896 -> 192.168.97.3:31234 SYN
+21.005s  [pktcap]           tcp-packet  [cni0] 10.42.0.1:29742 -> 10.42.0.142:12345 SYN
+21.005s  [pktcap]           tcp-packet  [veth0829e839] 10.42.0.1:29742 -> 10.42.0.142:12345 SYN
+21.005s  [pktcap]           tcp-packet  [veth0829e839] 10.42.0.142:12345 -> 10.42.0.1:29742 SYN,ACK
+21.005s  [pktcap]           tcp-packet  [cni0] 10.42.0.142:12345 -> 192.168.97.2:46896 SYN,ACK
+21.005s  [pktcap]           tcp-packet  [eth0] 192.168.97.3:31234 -> 192.168.97.2:46896 SYN,ACK
+21.005s  [pktcap]           tcp-packet  [eth0] 192.168.97.2:46896 -> 192.168.97.3:31234 ACK
+21.005s  [pktcap]           tcp-packet  [cni0] 10.42.0.1:29742 -> 10.42.0.142:12345 ACK
+21.005s  [pktcap]           tcp-packet  [veth0829e839] 10.42.0.1:29742 -> 10.42.0.142:12345 ACK
```

We get a mix of behavior from the Happy Path and Break #2.

As I showed in the previous post about race conditions, this can happen when a pod is terminated before `kube-proxy` has a chance to remove the iptables rule DNAT-ing traffic to it. Unlike the first two examples -- in practice nobody is going to intentionally modify EndpointSlices with the wrong IP addresses / ports -- this can very well happen in real world production scenarios. You'd observe general flakiness / intermittent connection failures around rolling deploys. 

# Takeaway
The experiments in this post clearly reveal that Kubernetes services are just an abstraction on top of the Linux kernel's `iptables`/`netfilter` subsystem. All of the errors didn't really have to do with the Kubernetes components themselves, but were just standard Linux networking errors you'd get any time you tried to access the wrong port or IP address. 