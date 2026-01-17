---
title: "Auto-Populating Pi-hole DNS from Kubernetes Ingresses"
date: 2026-01-17
draft: false
tags: ["kubernetes", "pihole", "dns", "external-dns", "homelab"]
---

Managing DNS records for a homelab with dozens of services is tedious. Every time you deploy something new, you have to remember to add a DNS record. I solved this by using external-dns to automatically create Pi-hole DNS entries from Kubernetes ingress resources.

## The Goal

When I create an ingress like this:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frigate
  annotations:
    external-dns.alpha.kubernetes.io/hostname: frigate.bowerha.us
spec:
  rules:
    - host: frigate.bowerha.us
      # ...
```

I want Pi-hole to automatically create a DNS record pointing `frigate.bowerha.us` to my ingress controller's IP. No manual steps, no forgetting to update DNS.

## Components

- **Pi-hole** - DNS server and ad blocker
- **external-dns** - Kubernetes controller that syncs DNS records
- **Kubernetes ingress-nginx** - Ingress controller with a known IP

## Setting Up external-dns

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: external-dns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      containers:
        - name: external-dns
          image: registry.k8s.io/external-dns/external-dns:v0.14.0
          args:
            - --source=ingress
            - --provider=pihole
            - --pihole-server=http://pihole-web.pihole.svc.cluster.local
            - --pihole-password=$(PIHOLE_PASSWORD)
            - --domain-filter=bowerha.us
            - --registry=noop
            - --policy=upsert-only
            - --interval=1m
          env:
            - name: PIHOLE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: pihole-password
                  key: password
```

Key arguments:
- `--source=ingress` - Watch ingress resources for DNS records
- `--provider=pihole` - Use Pi-hole as the DNS backend
- `--domain-filter=bowerha.us` - Only manage records for this domain
- `--policy=upsert-only` - Create/update but never delete records
- `--interval=1m` - Check for changes every minute

### Pi-hole Password Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pihole-password
  namespace: external-dns
type: Opaque
stringData:
  password: "your-pihole-admin-password"
```

### RBAC

external-dns needs permission to read ingresses:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
rules:
  - apiGroups: [""]
    resources: ["services", "endpoints", "pods"]
    verbs: ["get", "watch", "list"]
  - apiGroups: ["extensions", "networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "watch", "list"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
  - kind: ServiceAccount
    name: external-dns
    namespace: external-dns
```

## Annotating Ingresses

The key annotation is `external-dns.alpha.kubernetes.io/hostname`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: home-assistant
  namespace: homeassistant
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    external-dns.alpha.kubernetes.io/hostname: ha.bowerha.us
spec:
  ingressClassName: nginx
  rules:
    - host: ha.bowerha.us
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: home-assistant
                port:
                  number: 8123
  tls:
    - hosts:
        - ha.bowerha.us
      secretName: home-assistant-tls
```

Within a minute of applying this ingress, Pi-hole will have a local DNS record for `ha.bowerha.us` pointing to the ingress controller's LoadBalancer IP.

## How It Works

1. external-dns watches for ingress resources with the hostname annotation
2. It queries the ingress controller's service to find its external IP
3. It calls the Pi-hole API to create/update a Local DNS Record
4. Pi-hole serves that record to all clients on the network

The records appear in Pi-hole under **Local DNS > DNS Records**.

## Troubleshooting

### external-dns CrashLoopBackOff

Usually means it can't reach Pi-hole. Check:
- Is the Pi-hole service URL correct?
- Is Pi-hole actually running and responding on port 80?
- Is the password correct?

I had an issue where Pi-hole's FTL process got stuck after a volume issue. Restarting Pi-hole fixed external-dns.

### Records Not Appearing

Check external-dns logs:

```bash
kubectl logs -n external-dns deployment/external-dns
```

Look for:
- Connection errors to Pi-hole
- Domain filter mismatches
- Ingress resources missing the annotation

### Wrong IP Address

external-dns gets the IP from the ingress controller's service. If you're using MetalLB or a cloud LoadBalancer, make sure the service has an external IP assigned:

```bash
kubectl get svc -n ingress-nginx
```

## Making It Standard Practice

I added this to my team's deployment guidelines in `CLAUDE.md`:

```markdown
## DNS Management

When creating ingress resources for `bowerha.us` domains, always include
the external-dns annotation:

annotations:
  external-dns.alpha.kubernetes.io/hostname: myapp.bowerha.us

This automatically creates a Pi-hole DNS record pointing to the ingress
controller. No manual DNS configuration needed.
```

Now DNS is just part of the deployment - no separate step to remember.

## Result

What used to be a manual process:
1. Deploy app
2. Create ingress
3. Remember to add DNS
4. Log into Pi-hole
5. Add Local DNS record
6. Test that it works

Is now:
1. Deploy app with annotated ingress
2. Done

The DNS record appears automatically within a minute. When I tear down services, I use `--policy=upsert-only` so records persist (useful for debugging), but you could use `--policy=sync` for automatic cleanup.

This small automation removes friction from deploying new services and eliminates a whole category of "why can't I reach this?" debugging sessions.
