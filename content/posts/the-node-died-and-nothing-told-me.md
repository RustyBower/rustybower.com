---
title: "The Node Died and Nothing Told Me: Two Outages on a Home K3s Cluster"
date: 2027-01-19
draft: false
categories: ["kubernetes", "homelab", "debugging"]
tags: ["k3s", "longhorn", "flannel", "keepalived"]
---

My home cluster is six small machines running K3s, with three control-plane nodes, Longhorn for storage, and Prometheus and Alertmanager for alerting. This autumn it lost a node twice. Neither outage paged me, and each one taught me something different about how the pieces fail together.

## Outage one: a dead node and eighteen silent hours

On August 28 at 20:09 UTC one worker went unreachable. No ping, no SSH, nothing on 6443. I found out the next afternoon, and not from an alert. I noticed pods that hadn't started.

Seven workloads were stuck in `ContainerCreating`: Pi-hole, Prometheus, Alertmanager, AppDaemon, a Minecraft server, Paperless's message broker and external-dns. Their six Longhorn volumes all sat in `attaching`, with `spec.nodeID` still pointing at the dead machine.

### Deleting the pods doesn't help

The obvious move is to force-delete the pods left on the dead node so their replacements can claim the volumes. I did that for all seven. Two minutes later, not one volume had moved.

What actually holds the volume is the Kubernetes `VolumeAttachment` object, and it belongs to kube-controller-manager's attach/detach controller, not to Longhorn. Kubernetes deliberately refuses to force-detach a volume from a node it can't reach, because it can't tell "dead" from "partitioned but still writing". Detaching an RWO volume from a node that is in fact still writing to it is how you corrupt a filesystem, so Kubernetes waits. It will wait forever.

Longhorn's `node-down-pod-deletion-policy` setting wouldn't have saved me either. All it automates is the pod deletion I'd just shown was useless.

### The out-of-service taint

The supported way to tell Kubernetes "this node is gone, I take responsibility" is the out-of-service taint:

```bash
kubectl taint node <dead-node> \
  node.kubernetes.io/out-of-service=nodeshutdown:NoExecute
```

Before applying it, I confirmed the node was really down and that it held no replicas of the affected volumes:

```bash
kubectl -n longhorn-system get replicas.longhorn.io -l longhornvolume=<pv>
```

The volumes detached and reattached within about ten seconds. All seven workloads were running about 70 seconds after the taint.

Two things not to do. First, **remove the taint before the node comes back.** It's `NoExecute`, so a node rejoining with it will evict everything scheduled onto it. Second, don't "fix" the stuck volumes by deleting the Node object. On K3s with embedded etcd, that also removes the node's etcd member, and if it's a control-plane node you've just turned a three-member cluster into two, where losing one more loses quorum.

### Why no alert fired

I had a `NodeNotReady` rule, and it was correct:

```yaml
- alert: NodeNotReady
  expr: kube_node_status_condition{condition="Ready", status="true"} == 0
  for: 5m
  labels:
    severity: critical
```

It never fired because Prometheus and Alertmanager were two of the seven stuck workloads. Each is a single replica with `strategy: Recreate`, backed by a Longhorn RWO volume. When the node they happened to be on died, they died with it, and their replacements waited on the same `VolumeAttachment` as everything else.

Alerting that runs inside the cluster can't be the only thing watching for the cluster losing a node. Some check has to live somewhere else.

### It wasn't SSD wear

The machines are old small-form-factor PCs on consumer SATA SSDs, and an earlier node death had been blamed on a failed SSD. I'd also estimated, from `/sys/block/sda/stat`, that the drives were wearing out fast. So my first theory was wear.

SMART said otherwise:

```
          life_left  P/E  NAND written  realloc  power_cycles  unexpected_power_loss  cmd_timeouts
node-a       87%      95     23.4 TB        0         2121              142                3386
node-b       81%     138     33.8 TB        0         1627               65                 860
```

My estimate had been off about fourfold. Block-layer counters count what the OS asked for, not what reached the NAND, and I'd also assumed nine years of service when the power-on hours showed about 5.3. If you want to know how worn an SSD is, read the drive's own NAND-write and erase-count attributes instead of extrapolating from `/proc`.

The column that stands out is **unexpected power loss**: 142 on the *least* worn drive, about 7% of its power cycles, alongside 3,386 command timeouts. That looks like power, not wear: a UPS, a shared circuit, or a bad SATA cable.

## Outage two: a kernel panic loop and a VIP flannel shouldn't have taken

On November 16, Debian shipped kernel 6.1.0-55 and one of my control-plane nodes picked it up. It panic-looped: 33 boots on the 17th, six more on the 18th. Every crash dump in `/var/lib/systemd/pstore/` was the same:

```
BUG: kernel NULL pointer dereference ... ext4_mb_load_buddy_gfp
```

It happened in whatever process touched ext4 at the wrong moment, whether that was kworker, containerd, Longhorn or k3s itself. 6.1.0-56 fixed it a day later. I found out from the aftermath.

### What kured did with it

kured reboots nodes one at a time after updates. At 03:21 on the 18th it rebooted a healthy control-plane node. The crash-looping node took over the control plane's keepalived VIP, crashed, and rebooted at 03:22 *while holding it*. The healthy node came back at 03:25 and took the VIP back two seconds before k3s started.

By morning both nodes had registered the VIP, not their own IP, as their flannel VTEP address:

```bash
kubectl get node -o 'custom-columns=N:.metadata.name,F:.metadata.annotations.flannel\.alpha\.coreos\.com/public-ip'
```

Two nodes showing the same VIP is broken. Once the VIP moved, the node that had lost it was cut off from the pod network. Then kured drained a third node *onto* it. Longhorn detaches routed through that node's instance manager timed out, the drain deadlocked on the instance-manager PodDisruptionBudget, and about 25 pods went down. Prometheus and Alertmanager were among them again.

### Why flannel picked the VIP

I first assumed this was a boot-time race: whichever address was on the interface when k3s started. Reading the source showed it's worse than that. k3s uses the default-gateway interface and takes the first IPv4 address flannel's helper returns, and that helper sorts *permanent* addresses ahead of dynamic ones. keepalived adds the VIP as a permanent address, and the node's real address comes from DHCP, so it's dynamic. The VIP always won whenever it was present. It only looked like a race because the VIP isn't always there.

The obvious knobs didn't help. `flannel-iface` picks the interface, not the address on it. `node-ip` doesn't feed flannel's choice. `flannel-external-ip` is a cluster-wide server flag. Making the real address static would just tie the two in an unstable sort.

What did work is marking the VIP **deprecated**. In keepalived's config on both VIP holders:

```
virtual_ipaddress {
    192.168.1.9/24 preferred_lft 0
}
```

A deprecated address still receives traffic, so the Kubernetes API on the VIP keeps working, but the kernel won't use it as a source address and flannel skips it. To test it, I restarted k3s on the node *while* it held the VIP. It registered its own address, and a full pod-to-pod mesh test across every node passed.

kured also got two settings so one bad reboot can't immediately cause the next:

```yaml
# Hold the lock 30m after each reboot so the node rejoins and
# Longhorn rebuilds before the next drain.
- --lock-release-delay=30m
# Don't reboot while any alert is firing.
- --prometheus-url=http://prometheus.prometheus.svc.cluster.local:9090
- --alert-firing-only=true
```

### A bonus: hollow container images

While that node was crash-looping, it pulled a new version of Open WebUI. Afterwards the container exited 0 immediately with no logs. I blamed the release, rolled back, and told Renovate to skip it.

The release was fine. The image's unpacked snapshot on that node had every `.py` file and 3,783 of 5,775 frontend files at **zero bytes**. containerd verifies the compressed layer's digest when it downloads it, but nothing verifies the files after they're unpacked, and a node that panics in the ext4 allocator halfway through an unpack can leave a snapshot that looks complete and isn't. `crictl rmi` and a re-pull fixed it, and I reverted the rollback.

Before calling an image broken, download the layer from the registry and `tar -tvzf` it. If the files are full-sized there and empty on the node, the problem is the node.

## Caveats

- **The power theory isn't confirmed.** The unexpected-power-loss count is suggestive, but I haven't checked the UPS battery or the circuit yet. Kernel panics don't increment that counter, so the November crashes don't add evidence either way.
- **The keepalived fix lives outside Git.** It's host config on two machines. A rebuilt control-plane node needs `preferred_lft 0` added back, or the bug returns the first time that node boots holding the VIP.
- **kured's alert gate depends on Prometheus being up.** It stops a reboot cascade when the alerting stack is healthy, which is the case that matters most, but it doesn't solve the underlying problem: in-cluster alerting still can't report the loss of the node it runs on.
- **The out-of-service taint is only safe if the node is really dead.** If it's partitioned and still writing, the taint lets a second node mount the same volume. Check the node from more than one angle before applying it.
