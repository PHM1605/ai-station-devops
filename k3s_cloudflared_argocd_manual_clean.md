# K3s + Cloudflare Tunnel Setup for `aistation.io.vn` and `argocd.aistation.io.vn`

This is a **manual reference** for setting up Cloudflare Tunnel with `cloudflared` in K3s using YAML **only** (no `.service`, no `tunnel.json` file). This document reflects the exact working path we used.

---

## ✅ Assumptions

- You already have:
  - A working Cloudflare account
  - Domain: `aistation.io.vn`
  - ArgoCD installed in `argocd` namespace
  - `k3s` installed on your server

---

## 🔐 Step 1: Authenticate Cloudflare Tunnel

```bash
cloudflared tunnel login
```

> This opens a browser. Select your domain (e.g., `aistation.io.vn`).
> It saves a `.cloudflared/cert.pem` file locally.

---

## 🚇 Step 2: Create the Tunnel (no `.json`, just token)

```bash
cloudflared tunnel create ai-station
```

- This creates a tunnel and stores it in `~/.cloudflared/ai-station.json`
- Get the tunnel **token**:

```bash
cloudflared tunnel token ai-station
```

---

## 📁 Step 3: K8s Secret (with TOKEN only)

No need for `.json` or `creds.json`. Just this:

```bash
kubectl create ns cloudflared

kubectl create secret generic cloudflared-token   --from-literal=token='<PASTE_YOUR_TOKEN_HERE>'   -n cloudflared
```

---

## 📦 Step 4: K8s Deployment (YAML-Based)

📄 Save as `infra/cloudflared/deploy.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: cloudflared
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
          args:
            - tunnel
            - --no-autoupdate
            - run
            - --token
            - $(TUNNEL_TOKEN)
          env:
            - name: TUNNEL_TOKEN
              valueFrom:
                secretKeyRef:
                  name: cloudflared-token
                  key: token
---
apiVersion: v1
kind: Service
metadata:
  name: cloudflared
  namespace: cloudflared
spec:
  selector:
    app: cloudflared
  ports:
    - port: 80
      targetPort: 80
```

Apply it:

```bash
kubectl apply -f infra/cloudflared/deploy.yaml
```

---

## 🌍 Step 5: Add Ingress Routes in Cloudflare Dashboard

Go to: `https://dash.cloudflare.com` → `Zero Trust` → `Access` → `Tunnels` → `ai-station` → **Configure** → `Public Hostname`

Add two routes:

| Hostname                    | Service URL                                                       |
|----------------------------|--------------------------------------------------------------------|
| `aistation.io.vn`          | `http://traefik.kube-system.svc.cluster.local:80`                 |
| `argocd.aistation.io.vn`   | `https://argocd-server.argocd.svc.cluster.local`                  |

No origin cert needed. Don’t set any `.json` or `.pem`.

---

## 🧪 Test ArgoCD

Open:

```
https://argocd.aistation.io.vn
```

Login:

- Username: `admin`
- Password: get via:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

---

## ✅ DONE

This setup works entirely through:

- `cloudflared` authenticated via `login`
- Tunnel started via `--token` from K8s Secret
- All public routes configured via Cloudflare dashboard

No systemd `.service`. No `.json` deployments.

---