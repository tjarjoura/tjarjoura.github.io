---
layout: post
title: "Demonstrating Kubernetes Race Conditions With A Custom Observability Tool"
date: 2026-04-17
---

I had always assumed that updating Kubernetes deployments was a zero downtime operation, but I recently discovered that isn't necessarily the case. 

We had a `traefik` proxy running inside of a Kubernetes cluster, acting as an API gateway in front of various backend services and we noticed that there would be occasional network errors during rolling restarts of the corresponding K8s Deployment. There were small windows of downtime during deploys. Granted this was small enough not to be noticed most of the time, but it flatly contradicted my previous expectation, and I wanted to understand why this was happening.

Some googling led me to this [Gruntwork blog post](https://www.gruntwork.io/blog/delaying-shutdown-to-wait-for-pod-deletion-propagation) and then this excerpt from [Kubernetes In Action](https://freecontent.manning.com/handling-client-requests-properly-with-kubernetes/) which gave a good explanation of the problem. 

To summarize -- the basic problem is that there are two processes that happen in parallel when a pod which is fronted by a Kubernetes service shuts down. The first is that SIGTERM is sent to the actual process running inside the pod container, which is handled according to whatever the application is coded to do. The second is that the pod's IP is removed from the list of endpoints belonging to the service, which is then picked up by `kube-proxy`, which _then_ removes the corresponding `iptables` rule for the pod's IP address. There is no built in mechanism for keeping these two in sync, so it can often happen that the process is terminated (and thus is not listening on its IP address) before the `iptables` rule for routing traffic to its IP address is removed.

This makes sense in theory -- Kubernetes deployments and services are managed by separate controllers that operate independently, so why should I have assumed they would magically sync up? Still, I wanted to really prove to myself that this was actually happening. There are a lot of moving parts here -- how could I possibly keep track of them at once?

The answer is that I needed a custom-built tool. And the realization I came to is that building this kind of tool is now possible in a way that just wasn't previously. The specific unlock was being able to use LLM-assisted coding to dramatically lower the tedium involved.

The first problem was to reliably reproduce the network errors. Because the window of downtime was so small, the error didn't always happen "organically". I needed to create some artificial traffic while performing the rolling restart in order to trigger the error.

I'm not ashamed to admit that in the past I would have written a bash loop that cURLs the traefik URL, opened up a separate terminal to trigger the rolling restart, and then tried to move my eyes back and forth fast enough to catch the error. Instead, this time I had Claude Code write a Go program that performs continuous HTTP requests against Traefik's URL, and in a separate goroutine triggers a rolling restart of the deployment. Then I had it log timestamped events for each operation. 

This isn't a hard problem, but it would have been tedious to code it all out by hand, and I probably would have decided it wasn't worth the effort for this "small" network blip.

I've recreated the environment for the purposes of this blog post. Here's an example of running the tool against a stock `k3d` cluster, hitting the built-in `traefik` proxy. I set up docker port-forwarding so it was exposed on `localhost:80`. I also modified it so there would be 2 replicas, and created an Ingress for a basic webpage at the path `/`.

What's interesting is that the availability gap on a single node k3d cluster is very small, much smaller than the gap I noticed on the "real" cluster, so I had to be pretty aggressive with the rate of HTTP requests. Here's an example run where it fires a request every 100ms:

```
+0.006s  [deployment-rollout]  restarting  deployment=traefik namespace=kube-system
+0.009s  [http]             response  duration=7.967706ms method=GET status=200 url=http://localhost:80
+0.037s  [deployment-rollout]  restarted  deployment=traefik namespace=kube-system
+0.113s  [http]             response  duration=3.19735ms method=GET status=200 url=http://localhost:80
  ...
+2.312s  [http]             response  duration=2.380564ms method=GET status=200 url=http://localhost:80
+2.377s  [pod-termination]  terminating  deletion_timestamp=2026-04-17T05:50:32-04:00 namespace=kube-system pod=traefik-6fdcf5b898-rwbfd pod_ip=10.42.0.83
+2.412s  [http]             error  error=Get "http://localhost:80": EOF method=GET url=http://localhost:80
+2.512s  [http]             response  duration=2.286649ms method=GET status=200 url=http://localhost:80
+2.614s  [http]             response  duration=4.442221ms method=GET status=200 url=http://localhost:80
```

You can see that it immediately triggers a restart of the k8s deployment. In the background this spins up a new pod, and after `2.377s` it terminates one of the old ones, and then the next request `35ms` later fails with EOF.  Presumably this was the gap between when the traefik process had closed its listening socket but traffic was still being forwarded to it. Then in another 100ms the requests start succeeding again. 

So with this I was able to reliably reproduce the error. 

Next I prompted Claude Code to "add endpointslice events" to the output of the tool. Now I could see, next to the logs for the HTTP requests and the rolling restart events, the precise timestamped moment when the pod's IP address was removed from the endpoint slice for the service.

```
+12.010s  [http]             response  duration=3.896189ms method=GET status=200 url=http://localhost:80
+12.309s  [http]             response  duration=3.026349ms method=GET status=200 url=http://localhost:80
+12.609s  [http]             response  duration=2.908761ms method=GET status=200 url=http://localhost:80
+12.856s  [endpointslice]    slice-modified  changes=[~10.42.0.99(ready)] slice=traefik-gmfgw
+12.884s  [pod-termination]  terminating  deletion_timestamp=2026-04-17T06:16:40-04:00 namespace=kube-system pod=traefik-75f59cb466-qjccp pod_ip=10.42.0.97
+12.909s  [http]             error  error=Get "http://localhost:80": EOF method=GET url=http://localhost:80
+12.914s  [endpointslice]    slice-modified  changes=[~10.42.0.97(terminating)] slice=traefik-gmfgw
+13.058s  [endpointslice]    slice-modified  changes=[-10.42.0.97(terminating)] slice=traefik-gmfgw
+13.863s  [pod-termination]  deleted  namespace=kube-system pod=traefik-75f59cb466-qjccp pod_ip=10.42.0.97
```

This was interesting, but the endpoint slices are just abstract API objects, in themselves they don't have any impact on the actual network packets flying around. What we really care about are the actual iptables rules, so that's what I decided to add next. 

Now this is really starting to get in the realm of things that would have not been worth the time without AI assistance. Without going into too much detail about iptables/nftables (that's worth its own post), the solution I landed on was to:
- run `nft monitor` to capture rule changes when `kube-proxy` makes them. Modern Linux distributions convert iptables -> nftables under the hood, so everything shows up under `nft monitor`.
- figure out which nftables chain belonged to which pod, by comparing the IP address being referenced inside of it to the IP address on the Pod's spec 
- do a diff with previous runs to look for any additions / removals of specific IP addresses. This is necessary because `kube-proxy` calls `iptables-restore`, which deletes and recreates everything on every change.
- report those changes as events to `stdout`

The algorithm is easy to describe, so I was able to explain to Claude Code how to do it, but coding it all out would have taken quite a bit of time. I was also able to clone the source code for both kube-proxy and iptables and point Claude Code at it for reference material when designing the implementation.

With this I was able to see the actual iptables changes on the same timeline as the pod termination events and the tcp network errors.

```
+12.610s  [http]             response  duration=2.260901ms method=GET status=200 url=http://localhost:80
+12.803s  [endpointslice]    slice-modified  changes=[~10.42.0.105(ready)] slice=traefik-gmfgw
+12.825s  [pod-termination]  terminating  deletion_timestamp=2026-04-17T06:30:28-04:00 namespace=kube-system pod=traefik-7b685bbcdd-cktdt pod_ip=10.42.0.102
+12.846s  [endpointslice]    slice-modified  changes=[~10.42.0.102(terminating)] slice=traefik-gmfgw
+12.910s  [http]             error  error=Get "http://localhost:80": EOF method=GET url=http://localhost:80
+13.010s  [endpointslice]    slice-modified  changes=[-10.42.0.102(terminating)] slice=traefik-gmfgw
+13.211s  [http]             response  duration=3.169981ms method=GET status=200 url=http://localhost:80
+13.566s  [nftables]         endpoint-added  pod=10.42.0.105
+13.566s  [nftables]         endpoint-removed  pod=10.42.0.102
```

This shows that the pod began terminating about 700ms before the corresponding iptables rule was removed, leaving a gap of downtime. Seeing the theory confirmed in real time like this was very cool. And I think this is only scratching the surface. We could keep digging by adding:
- `strace` captures for when the Traefik pod actually closed its listening socket or called `exit()`
- actual packet captures which could show which backend pod the packets were being routed to

Finally, seeing that the gap was consistently a few hundred milliseconds, the solution was to add a `preStop` lifecycle hook to the Traefik pod, to give the endpoints / iptables rules time to update before it stopped serving traffic:
```
lifecycle:
  preStop:
	exec:
	  command:
	  - /bin/sh
	  - -c
	  - sleep 10
```

Then the error is consistently avoided:
```
+20.944s  [pod-termination]  terminating  deletion_timestamp=2026-04-17T07:03:43-04:00 namespace=kube-system pod=traefik-54fcb567cc-4j6hk pod_ip=10.42.0.112
+20.972s  [endpointslice]    slice-modified  changes=[~10.42.0.112(terminating)] slice=traefik-gmfgw
+21.041s  [http]             response  duration=3.703407ms method=GET status=200 url=http://localhost:80
+21.342s  [http]             response  duration=3.984828ms method=GET status=200 url=http://localhost:80
  ...
+21.947s  [nftables]         endpoint-removed  pod=10.42.0.112
  ...
+31.120s  [endpointslice]    slice-modified  changes=[-10.42.0.112(terminating)] slice=traefik-gmfgw
+31.209s  [pod-termination]  deleted  namespace=kube-system pod=traefik-54fcb567cc-4j6hk pod_ip=10.42.0.112
+31.240s  [http]             response  duration=2.995166ms method=GET status=200 url=http://localhost:80
```

Now you can see that there is a big gap between the `terminating` and `deleted` events, more than big enough for the iptables changes to be applied, and so the HTTP errors are avoided.

There's a lot of discussion about LLMs atrophying your programming skills, giving you a shallower understanding of the system you're working on, and just generally reducing your competence as an engineer. In general, there may be some truth to that, but this case was the opposite experience. I was able to gain a much deeper and much more visceral understanding of a complex networking/distributed systems problem than would have been possible previously, at least on a reasonable timeframe.