---
title: "Cloudflare Tunnel — Expose Agentopia Services via Zero Trust"
---

# Cloudflare Tunnel — Expose Agentopia Services via Zero Trust

Cloudflare Tunnel provides secure external access to Agentopia services running on k3s (server36) without opening inbound ports or configuring firewalls.

## Architecture

```
Browser → agentopia.creodeck.space
         → Cloudflare Edge (TLS termination, DDoS, caching)
         → Cloudflare Tunnel (QUIC, outbound-only from k3s)
         → cloudflared pod (kube-system namespace)
         → bot-config-api.agentopia.svc.cluster.local:80
```

Key properties:
- **Outbound-only**: cloudflared initiates connections to Cloudflare — no inbound ports needed on server36
- **TLS**: Cloudflare handles TLS certificates automatically (wildcard `*.creodeck.space`)
- **Local config mode**: Ingress rules defined in k8s ConfigMap (not Cloudflare dashboard)

## Current Setup

| Component | Location |
|---|---|
| cloudflared Deployment | `kube-system` namespace, server36 k3s |
| ConfigMap `cloudflared-config` | `kube-system` — contains `config.yaml` with ingress rules |
| Secret `cloudflared-credentials` | `kube-system` — tunnel credentials (derived from TUNNEL_TOKEN) |
| DNS CNAME | `agentopia.creodeck.space` → `b7a302f5-...cfargotunnel.com` (Cloudflare DNS) |

Tunnel ID: `b7a302f5-0a49-4544-9eec-cd4425e72fa3`

## Ingress Routes

Current routes (defined in ConfigMap `cloudflared-config`):

| Hostname | Service | Purpose |
|---|---|---|
| `agentopia.creodeck.space` | `http://bot-config-api.agentopia.svc.cluster.local:80` | Bot creation UI + API |
| (catch-all) | `http_status:404` | Reject unknown hostnames |

## How to Reproduce (from scratch)

### 1. Create Tunnel on Cloudflare

1. Go to [Cloudflare Zero Trust](https://one.dash.cloudflare.com) → Networks → Tunnels
2. Create a new tunnel, note the **TUNNEL_TOKEN**
3. Note the **Tunnel ID** (visible in tunnel details)

### 2. Deploy cloudflared on k3s

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
TUNNEL_ID="<your-tunnel-id>"
```

**Create credentials secret** (convert TUNNEL_TOKEN to credentials.json):

```bash
# Decode TUNNEL_TOKEN → credentials.json → k8s Secret
echo "<TUNNEL_TOKEN>" | base64 -d | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(json.dumps({'AccountTag': d['a'], 'TunnelID': d['t'], 'TunnelSecret': d['s']}))
" | kubectl create secret generic cloudflared-credentials \
    --namespace kube-system \
    --from-file=credentials.json=/dev/stdin
```

**Create config ConfigMap**:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudflared-config
  namespace: kube-system
data:
  config.yaml: |
    tunnel: ${TUNNEL_ID}
    credentials-file: /etc/cloudflared/credentials.json
    metrics: 0.0.0.0:2000
    no-autoupdate: true
    loglevel: info
    ingress:
      - hostname: agentopia.creodeck.space
        service: http://bot-config-api.agentopia.svc.cluster.local:80
      - service: http_status:404
EOF
```

**Deploy cloudflared**:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudflared
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:latest
          imagePullPolicy: Always
          args:
            - tunnel
            - --config
            - /etc/cloudflared/config.yaml
            - run
          volumeMounts:
            - name: config
              mountPath: /etc/cloudflared/config.yaml
              subPath: config.yaml
              readOnly: true
            - name: credentials
              mountPath: /etc/cloudflared/credentials.json
              subPath: credentials.json
              readOnly: true
          livenessProbe:
            httpGet:
              path: /ready
              port: 2000
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: 10m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
      volumes:
        - name: config
          configMap:
            name: cloudflared-config
        - name: credentials
          secret:
            secretName: cloudflared-credentials
EOF
```

### 3. Create DNS CNAME

In Cloudflare Dashboard (dash.cloudflare.com) → `creodeck.space` → DNS → Add Record:

| Type | Name | Target | Proxy |
|---|---|---|---|
| CNAME | `agentopia` | `<tunnel-id>.cfargotunnel.com` | Proxied (orange) |

### 4. Verify

```bash
# Health check
curl https://agentopia.creodeck.space/health
# → {"status":"ok","version":"2.0.0"}

# UI
open https://agentopia.creodeck.space/ui

# Check tunnel connections (should show 4 registered)
kubectl logs deploy/cloudflared -n kube-system --tail=10
```

## Adding New Routes

Edit the ConfigMap and restart:

```bash
kubectl edit configmap cloudflared-config -n kube-system
# Add new hostname under ingress: (above the catch-all)
#   - hostname: newservice.creodeck.space
#     service: http://new-service.agentopia.svc.cluster.local:8080

kubectl rollout restart deploy/cloudflared -n kube-system
```

Then add a CNAME DNS record for the new hostname → same tunnel ID `.cfargotunnel.com`.

## Gotchas

1. **Cloudflare "Hostname routes (Beta)"** under Connectors does NOT push ingress rules to cloudflared. Must use local config mode (ConfigMap) instead.
2. **Local config requires credentials.json**, not TUNNEL_TOKEN. Convert token using the script above.
3. **Catch-all rule required**: cloudflared config must end with `- service: http_status:404` as the last ingress entry.
4. **DNS propagation**: After adding CNAME, may take 1-5 minutes. Use `dig @1.1.1.1 <hostname>` to check Cloudflare DNS directly.
5. **Not ArgoCD-managed**: cloudflared lives in `kube-system` (outside agentopia namespace). Treated as cluster infrastructure, applied manually. If cluster is rebuilt, re-run steps above.
