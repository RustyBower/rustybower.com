---
title: "A Tailscale Search Domain in Every Pod, and the One Binary It Broke"
date: 2027-01-26
draft: false
categories: ["kubernetes", "networking", "debugging"]
tags: ["tailscale", "coredns", "resolv-conf", "ads-b"]
---

The morning after a round of node reboots, the Flightradar24 feeder on my cluster stopped feeding. Its log was a single line, repeated:

```
Could not connect feed! (result -1, fd -1)
```

That included connecting to the readsb aggregator *in the same namespace*. The pod was running and nothing had changed in Git. A shell in the same pod could resolve and reach everything fine:

```
$ kubectl exec deploy/fr24 -- getent hosts feed.flightradar24.com
...resolves...
```

So the network worked and the process looked stuck.

## What had changed: one extra search domain

The thing that had changed was each pod's `/etc/resolv.conf`. After the reboots, the search list on pods scheduled to the rebooted nodes looked like this:

```
search adsb.svc.cluster.local svc.cluster.local cluster.local tailXXXX.ts.net example.com
nameserver 10.43.0.10
options ndots:5
```

`tailXXXX.ts.net` is my tailnet's MagicDNS domain. The nodes run Tailscale, and Tailscale rewrites the host's `/etc/resolv.conf` to add its own search domain. kubelet builds each pod's `resolv.conf` from the host's search list at the moment the pod starts, so whatever Tailscale had put there was copied into every pod created afterwards. The one node that hadn't rebooted didn't have it.

## Why only fr24feed noticed

Every other app on those nodes kept working. fr24feed is different because it's a statically linked binary with its own DNS resolver. It doesn't use glibc's resolver the way `getent`, `curl` and most of the other tools in the same pod do. It even announces this at startup:

```
DNS mode: PING
```

With the default `ndots:5`, any name with fewer than five dots is tried against each search domain first, so `feed.flightradar24.com` becomes `feed.flightradar24.com.adsb.svc.cluster.local`, then `...svc.cluster.local`, and so on through the Tailscale domain before the name is ever tried as-is. glibc works through that list fine. fr24feed's resolver didn't, and the extra domain was enough to break every lookup it made.

## The fix

A pod-level `dnsConfig` that drops `ndots` to 1:

```yaml
spec:
  template:
    spec:
      dnsConfig:
        options:
          - name: ndots
            value: "1"
```

With `ndots:1`, any name containing a dot is tried as an absolute name first, so external hostnames resolve on the first query and the search list barely comes into play. The feeder reconnected as soon as the new pod came up.

## It comes and goes

What makes this nasty is that it isn't stable. A month later, the pods in that same namespace disagree with each other depending on when they started:

| Pod | Node | Started | Tailscale domain in search list |
|---|---|---|---|
| piaware | node-b | Nov 18 09:32 | yes |
| ultrafeeder | node-e | Nov 19 09:02 | no |
| adsb2mqtt | node-c | Nov 19 14:17 | yes |
| fr24 | node-c | Nov 20 09:56 | no |

Two pods on the same node, started a day apart, with different DNS configuration. The host's `/etc/resolv.conf` on that node had last been rewritten a few hours before I checked, and the Tailscale domain wasn't in it at that moment. Something on the host, the DHCP client or Tailscale itself, keeps rewriting the file, and each pod captures whichever version is there when it starts.

So the failure follows pod start times, not deploys. An app can work for weeks and break after an unrelated reschedule, then fix itself on the next one.

## How to spot it

If one app suddenly can't resolve names while a shell in the same pod can, compare the search lists:

```bash
kubectl get pods -A -o name | while read p; do
  ns=${p%%/*}
  echo "$p $(kubectl exec -n "$ns" "$p" -- grep search /etc/resolv.conf 2>/dev/null)"
done | grep ts.net
```

Any pod showing a `ts.net` domain picked up the host's Tailscale DNS.

## Caveats

- **`ndots:1` is a workaround on one app.** The fix at the source is `tailscale set --accept-dns=false` on the Kubernetes nodes, so Tailscale stops touching the host's `resolv.conf`. I haven't done that yet. The nodes still have MagicDNS enabled.
- **I don't know exactly why fr24feed's resolver fails.** The extra search domain is what changed, and `ndots:1` fixed it, but I haven't traced what the binary does with a five-entry search list. A limit on search domains, mishandling NXDOMAIN, or a timeout budget used up by the extra queries would all produce what I saw.
- **`ndots:1` changes behaviour for short names.** Names with a dot, like `service.namespace`, are now tried as absolute names first, which costs one failed external lookup before falling back to the search list. That's fine for a feeder that mostly talks to the internet, but it's not something to apply to every workload by default.
- **Any other static or musl-based binary on these nodes is exposed to the same thing.** fr24feed is just the one that happened to break.
