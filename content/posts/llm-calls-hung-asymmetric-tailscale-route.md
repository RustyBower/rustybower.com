---
title: "Hung Calls and 40 ms Errors: Two Ways My Local LLM Server Failed"
date: 2027-04-06
draft: false
categories: ["llm", "networking", "debugging"]
tags: ["tailscale", "llama-swap", "llama-cpp", "mac-studio"]
---

My cluster sends its LLM work to a Mac Studio on the LAN running [llama-swap](https://github.com/mostlygeek/llama-swap) in front of llama.cpp. Frigate uses it to describe camera alerts, Mealie uses it to parse recipes, and an agent uses it to summarise cluster health. Twice it has failed in ways where the obvious reading of the evidence pointed the wrong way.

## Failure one: the server logged 200, the client never got it

Wiring Mealie up to the Mac, I noticed that recipe imports sometimes hung until the client timed out. llama-swap's log for the same requests showed this:

```
200 POST /v1/chat/completions
```

The server finished, returned 200, and the reply never arrived.

### Narrowing it to one node

The pattern was that short requests worked and long ones didn't, but only sometimes. "Sometimes" turned out to mean "depending on which node the caller ran on". The test I settled on was a throwaway `curl` pod pinned to a node, asking for a long completion:

```bash
kubectl run llm-test --rm -it --restart=Never \
  --overrides='{"spec":{"nodeName":"node-e"}}' \
  --image=curlimages/curl -- \
  curl -s -m 180 -o /dev/null -w '%{http_code} %{time_total}s\n' \
  http://192.168.1.50:11434/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen3.8:27b","max_tokens":1500,"stream":false,
       "messages":[{"role":"user","content":"Write a long essay about bread."}]}'
```

From one node it returned `200 64s`. From another, the same request hung every time it stayed quiet for longer than somewhere between 9 and 35 seconds. Streamed responses and big fast downloads were fine, because they never go quiet. Frigate never saw the problem because it happens to run on the node that worked.

### The reply took a different road

On the Mac:

```
$ route get 192.168.2.15
   interface: utun4
```

`utun4` is the Tailscale tunnel. The NAS is a Tailscale subnet router, and among the routes it advertises is the whole subnet my Kubernetes nodes live on. The Mac had *accept routes* turned on, so it trusted that advertisement and sent replies to the nodes through Tailscale via the NAS, even though the requests had arrived directly over the LAN through the router.

Requests went one way and replies went the other. That's an asymmetric route, and it works right up until something on one of the two paths keeps per-connection state and loses track of a connection that has gone quiet.

### The fix

One setting on the Mac:

```bash
tailscale set --accept-routes=false
```

After that, `route get` for the node subnet shows `en0`, and a 61-second non-streamed request from the bad node returns 200. The Mac doesn't need the NAS's subnet routes. It's on the same LAN.

If it comes back, the check is:

```bash
tailscale debug prefs | grep RouteAll
```

## Failure two: "Compute error." in 40 milliseconds

The second failure looked like a bad request. Every call, from every client, to every model, started returning:

```
500 {"error":{"message":"Compute error.","type":"server_error"}}
```

That's llama-server reporting that `llama_decode` failed. The message reads like the server choked on the input, so I went looking at prompt templates and payloads. Replaying the exact payload with `curl` later returned 200, which ruled the request out.

The number that mattered was the latency. These 500s came back in about **40 ms**. A request that fails on its content gets far enough to tokenise and start decoding first. A 40 ms failure never got any compute at all, and on Apple Silicon that means the GPU was out of memory.

### A model that never unloaded

llama-swap loads models on demand and unloads them after an idle `ttl`. It also has groups. I keep three models in a `hot` group that's marked persistent, so the camera-alert model is always loaded and Frigate never waits for a load:

```yaml
groups:
  hot:
    persistent: true
    swap: false
    exclusive: false
    members:
      - ...
```

Every model outside `hot` is supposed to have a `ttl`. One didn't. Once anything asked for it, its 16 GB stayed loaded for good. With the three pinned models, that came to about 57 GB of a roughly 48 GB GPU budget on a 64 GB machine, and from then on *every* request failed, whichever model it asked for.

The fix was `ttl: 300` on that entry. The config now carries the rule as a comment:

```yaml
# Any entry outside `hot` MUST carry a ttl.
```

Checking for it is a single grep for model entries without a `ttl:` line. To see what's actually loaded right now:

```bash
curl -s localhost:11434/running
```

### The opposite trap

While fixing that, I found the reverse problem. I'd added `ttl: 300` to a model that *is* in `hot`, out of the same habit. A per-model `ttl` overrides the group's persistence, so that "always loaded" model unloaded after five idle minutes and paid a full load on the next request. Pinned models get no `ttl`, and everything else must have one.

### Find the caller by IP, not in Git

When a model wouldn't go idle, the quickest way to find what kept loading it was the source IP in llama-swap's log, not a search of my manifests. One caller keeps its model name in its own database, so grepping the repo found nothing at all.

## Caveats

- **I didn't trace why the asymmetric route hung only quiet connections.** Tailscale subnet routers normally SNAT, so replies arriving through the NAS should have come from the wrong source address and failed outright. They didn't: short calls completed. Something stateful along one of the two paths was dropping idle flows. Turning off accept-routes removed the asymmetry, so I stopped there.
- **I also can't explain why one node was unaffected.** It sits on the same subnet as the others. The pinned-pod test is what showed it, and I didn't dig further once the fix worked from every node.
- **The 40 ms rule of thumb is specific to this setup.** It holds for llama.cpp on Apple Silicon with these model sizes. A different backend could fail differently under memory pressure.
- **Long non-streamed calls are fragile anyway.** Streaming would have hidden the routing bug completely, which is good for users and bad for finding the problem. I'd rather find it.
