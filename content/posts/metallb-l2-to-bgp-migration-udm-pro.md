---
title: "The LoadBalancer That Never Worked: Migrating MetalLB from L2 to BGP"
date: 2026-07-30
draft: false
tags: ["metallb", "bgp", "kubernetes", "k3s", "homelab", "networking", "unifi", "argocd", "dns", "debugging"]
---

I started the day asking why one of my Frigate cameras was using more CPU than the others. I finished it having moved every LoadBalancer in my Kubernetes cluster onto BGP-routed addresses, because along the way I found a service that had been silently unreachable for two years.

This is the write-up of that migration: MetalLB from L2 mode to BGP, peered to a UniFi Dream Machine Pro. Mostly it's the six things that bit me, because those are the parts nobody puts in the tutorials.

## The two-year-old bug

While poking at Frigate I tried to hit its LoadBalancer address directly:

```
$ for p in 5000 1984 8554 8555; do nc -zv 172.30.0.1 $p; done
    all refused
```

Every port refused. Frigate worked fine — but only through the Traefik ingress. Its LoadBalancer had never worked at all.

The address was the tell. `172.30.0.1` is the `.1` of the node subnet: my UDM's own interface. MetalLB had auto-assigned my gateway's address to a Kubernetes service.

The confirmation was easy, and it's a useful trick:

```
$ ping 172.30.0.1     # replies
$ ping 172.30.0.5     # silent, and it's a working LoadBalancer
```

**A MetalLB address never answers ICMP.** MetalLB doesn't put the address on an interface; it just answers ARP for it and lets the node's DNAT rules do the rest. So a "LoadBalancer address" that replies to ping isn't a LoadBalancer address — something else owns it.

## How it happened

The pool was `172.30.0.0/16`. Reasonable-looking. Then I looked at a node:

```
$ ip -4 addr show eno1
    inet 172.30.0.11/15 ...          # /15, not /24
$ ip route | grep default
    default via 172.30.0.1
```

A **/15**. So `172.30.0.0`–`172.31.255.255` is one flat L2 segment, and it holds:

- the gateway at `.1`
- five DHCP-assigned nodes at `.11`–`.15`
- a NAS at `172.30.1.245`
- both MetalLB pools

MetalLB allocates the lowest free address in a pool and knows nothing about node addresses, gateways, DHCP scopes, or filers. It took `.1` because `.1` was first.

And it was going to keep going. Allocations so far were `.1` through `.8` plus `.10`, in exact ascending order. The next two services would have taken `.9` and then **`.11` — a node's own address**. `avoidBuggyIPs` doesn't help; it only skips `.0` and `.255`.

The immediate fix was to narrow the pool to the addresses actually in use plus a growth range well clear of the infrastructure. But the real fix was to stop sharing a segment with the gateway at all.

## Why BGP, and why the prefix matters

L2 mode requires the pool to be on-link with the nodes. BGP doesn't: you advertise a prefix and the router installs a route. That means the service range can live somewhere with nothing else in it.

I picked `10.44.0.0/16`, continuing the k3s numbering — `10.42` pods, `10.43` services, `10.44` load balancers.

The important property isn't the mnemonic, it's that **the prefix is off-segment**. If I'd carved the BGP pool out of the existing `/15`, the nodes would still be on-link for it and ARP would carry traffic *even if BGP were completely broken*. Every test would pass and I'd have learned nothing. Off-segment means a working address proves BGP is genuinely forwarding.

Verified it was free three ways before using it — no route from a node or the LAN, no ICMP reply, and no ARP presence probed from a node on the segment.

## The UDM Pro side

BGP on UniFi is a config file you upload (Settings → Routing → BGP), and it's plain FRR:

```
router bgp 64512
 bgp router-id 172.30.0.1
 bgp log-neighbor-changes
 no bgp ebgp-requires-policy
 !
 neighbor k8s peer-group
 neighbor k8s remote-as 64513
 neighbor k8s timers 10 30
 !
 neighbor 172.30.0.11 peer-group k8s     # one per node
 ...
 address-family ipv4 unicast
  neighbor k8s activate
  neighbor k8s prefix-list metallb-in in
  maximum-paths 5
 exit-address-family
exit
!
ip prefix-list metallb-in seq 10 permit 10.44.0.0/16 le 32
ip prefix-list metallb-in seq 99 deny any
```

Three things worth calling out:

**`no bgp ebgp-requires-policy` is not optional.** Modern FRR implements RFC 8212: without an explicit policy it establishes the session and then silently discards every route. The session shows up. Nothing works. You will not enjoy debugging that on a UDM.

**The prefix-list is the actual safety net.** It confines the cluster to injecting `10.44.0.0/16` and nothing else, so a MetalLB misconfiguration can't push arbitrary routes into the home network. UniFi requires prefix-lists *after* the `router bgp` block.

**Set DHCP reservations for your nodes first.** Mine were on DHCP leases. Five `neighbor` statements pinned to addresses that could drift is a trap you set for your future self.

On the cluster side it's three CRDs — a pool, a `BGPAdvertisement`, and a `BGPPeer`. All five speakers came up:

```
"msg":"BGP session established"   x5
"event":"sessionUp"               x5
```

## Six things that bit me

### 1. Native mode rejects `keepaliveTime`

MetalLB v0.16 still ships the native BGP implementation alongside FRR and FRR-K8s. It works fine for a plain IPv4 eBGP session, but it's feature-frozen, and the webhook is blunt about it:

```
peer 172.30.0.1 has keepalive-time set on native bgp mode
```

`holdTime` is accepted, `keepaliveTime` isn't. No real loss — MetalLB derives keepalive as `holdTime/3`, and BGP negotiates the lower of the two hold times anyway, so the router's value governs. Native mode also has no BFD, which makes hold time your only failure detection.

### 2. Client-side apply silently ate a UDP port

My Frigate service declared `8555` on both TCP and UDP. The live object only ever had the TCP one — while ArgoCD cheerfully reported `Synced`.

`Service.spec.ports` carries two different merge semantics:

```
x-kubernetes-patch-merge-key:  port              ← client-side apply
x-kubernetes-list-map-keys:    [port, protocol]  ← server-side apply
```

Client-side apply — kubectl's default, and ArgoCD's — merges the port list on `port` **alone**. Two entries with port 8555 collide on that key and the second is silently discarded. The `last-applied-configuration` annotation kept listing the UDP port, which made it look like it had been applied.

I reproduced it deliberately: `kubectl apply` reported `configured` and still produced four ports; `kubectl apply --server-side` produced all five. Fix is `argocd.argoproj.io/sync-options: ServerSideApply=true`.

Any resource needing two entries that differ only by a non-merge-key field has this problem.

### 3. Two ways to pin an address, and you may not use both

MetalLB accepts `spec.loadBalancerIP` (deprecated) and the `metallb.io/loadBalancerIPs` annotation. If a service has both, it doesn't pick one:

```
Warning  AllocationFailed  service can not have both
         metallb.io/loadBalancerIPs and svc.Spec.LoadBalancerIP
```

It allocates **nothing**. My Paperless FTP service came up with no address at all because the base set the deprecated field and my overlay added the annotation.

Also worth knowing: the legacy `metallb.universe.tf/loadBalancerIPs` prefix is *still honoured* in v0.16.1. I assumed it was dead, concluded a service was unpinned, and briefly believed I'd created a hazard that didn't exist. Tested it with a throwaway service — it works fine. Don't write new config with it, but don't panic when you find it.

### 4. One Service can't hold two addresses of the same family

The obvious zero-downtime move — put both old and new addresses on one Service — doesn't work:

```
Failed to allocate IP: IPFamilyForAddresses: same address family
```

`loadBalancerIPs` is for dual-stack, not two IPv4 addresses. So I used a **parallel Service**: a second object selecting the same pods on the new address. Both serve at once, external references move one at a time, and then the old one goes. No flag day.

One trap in that pattern: for Mosquitto I nearly promoted the parallel service and deleted the original. That would have broken Home Assistant, whose broker is configured as `mosquitto.mosquitto.svc.cluster.local` — the cluster DNS name follows the Service *name*. The right move was to repoint the original and delete the parallel one.

### 5. external-dns `upsert-only` leaves litter, and repoints non-atomically

My external-dns runs `--policy=upsert-only`, so it never deletes records. Over time that leaves orphans pointing at addresses that stopped existing. I found three — `syncthing`, `glance`, `glance-test` — all from apps deleted long ago. When two names didn't follow a migration I assumed they were lagging; they had no Ingress at all.

More disruptive: it rewrites records **delete-then-add**. Watching a repoint of 36 records:

```
poll 1:  172.31.0.2 x36
poll 2:  172.31.0.2 x15          ← 21 names briefly did not exist
poll 3:  10.44.1.2 x34
```

For ~30 seconds those names returned NXDOMAIN. A phone that queried in that window cached the negative answer and "couldn't resolve" for a while after everything was fine. Worth knowing before you repoint anything critical.

### 6. `load-balancer-cleanup` zombies

Five separate Services refused to delete, some terminating since the previous year, still holding their addresses and erroring in the controller every 17 minutes. All stuck on the `service.kubernetes.io/load-balancer-cleanup` finalizer, several orphaned in namespaces that had been force-deleted out from under them — which makes them unpatchable:

```
$ kubectl -n deepstack patch svc deepstack-service ...
Error from server (NotFound): namespaces "deepstack" not found
```

The fix is to recreate the namespace so the object becomes addressable again, clear the finalizer, then delete the namespace:

```
kubectl create ns deepstack
kubectl -n deepstack patch svc deepstack-service --type=merge \
  -p '{"metadata":{"finalizers":null}}'
kubectl delete ns deepstack
```

If a Service won't die, this finalizer is the first thing to check.

## Static or dynamic?

The pools split into auto-assigned and pinned, and the test I settled on is:

> Does anything **outside Kubernetes** reference this address?

Because an auto-assigned address changes when the Service is recreated, and whatever held it breaks silently.

Applying that honestly, almost nothing qualified as dynamic. Only three services had zero references anywhere and were reached purely by name through the ingress. Everything else — the broker, the ingress controller, Plex, the DNS server — needed pinning.

One I got wrong at first: Music Assistant looked like a browser-only app until I read its ports. It exposes `streamserver:8097`, `streams:8098` and `lms:3483` — it hands players URLs built from its own address and is discovered by Squeezebox clients. Definitively static. **Read the ports before classifying.**

## Verification, and the mistake I made twice

The single most useful technique: capture on the **node's interface**, not in the pod. kube-proxy DNATs the destination before the pod sees it, so a pod cannot tell which address a client used. On the node, pre-DNAT, you see the original:

```
tcpdump -i any -n -Q in "dst host 172.31.0.254 and port 53"
```

That's how I proved nothing was still using an old address before retiring it, and how I proved the remote cluster's CoreDNS had actually picked up a config change — resolution succeeded either way, because both addresses answered, so the only real evidence was which one the packets went to.

**Always send a canary query first.** A broken filter and a genuinely idle address produce identical output. If you haven't proven the capture works, silence means nothing. I built that check into the script as a hard failure.

And the mistake, which I made twice: **I verified reachability from one subnet and generalised.** First I declared a whole `/16` unreachable after testing two addresses in it. Later I confirmed the new prefix worked from my workstation and called it done — then found my network has *eight* client VLANs, and the one my workstation sits in has an `Internal → Internal Any/Any` rule while the IoT VLAN is default-deny with explicit allows. My testing had proven nothing about the VLAN that mattered.

BGP installs a route. **A route is not permission.** The firewall rules referenced the old addresses explicitly and all needed updating:

```
Allow Pihole DNS   IoT → 172.31.0.254:53
Allow MQTT         IoT → 172.31.0.9:1883
Port Forward Plex  WAN → 172.31.0.4:32400
```

The DNS one was the dangerous one: paired with a `Block External DNS` rule, an IoT device that renewed its lease would have had no resolver *and* no fallback.

## Config is layered, and the top layer lies

My NAS kept querying the old DNS server after I'd fixed it. Host config was unambiguous:

```
/etc/resolv.conf              nameserver 10.44.1.254
midclt network.configuration  nameserver1 10.44.1.254
```

The culprit was a Docker container running with host networking, holding a **copy** of `/etc/resolv.conf` taken at creation — three weeks earlier. Updating the host didn't touch it. Neither did the Tailscale admin UI, because that container has `accept-dns` off (`CorpDNS: false`), so Tailscale deliberately doesn't manage its resolver. A container restart fixed it in seconds.

"The host config is correct" is necessary, not sufficient. Check the layer that's actually sending the packets.

## Was it worth it?

For the stated goal — stop MetalLB handing out infrastructure addresses — the pool narrowing alone would have done it. BGP was the bigger swing, and it bought:

- Service addresses in a range with nothing else in it, so the whole class of
collision is gone by construction
- ECMP across all five nodes (`announcing from node "kube03"` *and* `"kube05"`
for the same service)
- Failure detection in ~30s instead of ~3min, via hold timers

The cost is that half the config now lives on an appliance, outside Git, where a firmware update could reset it. For a homelab that's an acceptable trade. If your router doesn't do BGP, narrow your pools and sleep fine.

The thing I'd tell past-me isn't about BGP at all:

**Test from the network that matters, not the one you're sitting on.** I learned that twice in one day.
