# Caddy Web Server Setup Guide

This directory contains the Kubernetes manifests to deploy the [Caddy](https://caddyserver.com/) web server on top of the KIND cluster created using the steps in [../kind/README.md](../kind/README.md).

Manifests included:

| File | Kind | Purpose |
|---|---|---|
| `namespace.yml` | Namespace | Creates the `dev` namespace that Caddy is deployed into |
| `deployment.yml` | Deployment | Caddy deployment (2 replicas, probes, resource limits) |
| `service.yml` | Service (ClusterIP) | Exposes the Deployment's pods on port 80 inside the cluster |

## Concepts: Deployment vs Service

**Deployment** — A Deployment manages a set of identical Pods for you. You describe the desired state (container image, how many replicas, resource limits, health checks) and the Deployment controller keeps that state true: restarting crashed Pods, recreating Pods lost when a node dies, and rolling out updates without downtime. In `deployment.yml`, `caddy-deployment` keeps 2 replicas of the `caddy:2-alpine` container running at all times, with each Pod labeled `app: caddy`.

**Service** — Pods are ephemeral: every time one is recreated it gets a new IP address, so nothing else in the cluster can reliably talk to it directly. A Service gives a single stable address (a ClusterIP and DNS name) that automatically load-balances traffic across whichever Pods currently match its label selector, no matter how many times those Pods come and go. In `service.yml`, `caddy-service` selects any Pod labeled `app: caddy` and forwards traffic on port 80 to them.

**How they relate** — The Deployment creates and owns the Pods; the Service doesn't know the Deployment exists at all — it only watches for Pods matching its selector (`app: caddy`) and load-balances across whatever it finds. That shared label is the only link between the two objects. So when the Deployment scales, restarts, or replaces Pods, the Service picks up the change automatically, with zero manual reconfiguration.

> Prerequisite: make sure the KIND cluster is already up and `kubectl` is pointing at it before continuing here.
>
> ```bash
> kind get clusters
> kubectl get nodes
> ```

## 1. Create the Namespace

```bash
kubectl apply -f namespace.yml
kubectl get namespace dev
```

All the resources below are created in the `dev` namespace (set via `metadata.namespace` in each manifest).

## 2. Deploy Caddy (Deployment + Service)

Apply the Deployment:

```bash
kubectl apply -f deployment.yml
kubectl get deployment caddy-deployment -n dev
kubectl get pods -n dev -l app=caddy
```

Apply the Service:

```bash
kubectl apply -f service.yml
kubectl get svc caddy-service -n dev
```

## 3. Accessing Caddy

Use `kubectl port-forward` to reach the Service from your machine:

```bash
kubectl port-forward svc/caddy-service 8080:80 -n dev
```

Then open:

```bash
http://localhost:8080
```

If you're on an EC2 box, tunnel it from your Mac the same way as the dashboard step:

```bash
ssh -i your-key.pem -L 8080:127.0.0.1:8080 ubuntu@<ec2-public-ip>
```

### Accessing directly via the EC2 public IP (no SSH tunnel)

If you'd rather browse straight to the EC2 instance's public IP instead of tunneling, bind the port-forward to all interfaces on the EC2 box:

```bash
kubectl port-forward --address 0.0.0.0 svc/caddy-service 8080:80 -n dev
```

Then open port `8080` in the EC2 instance's security group (allow inbound TCP from your IP), and browse to:

```bash
http://<ec2-public-ip>:8080
```

Note this only works while the `kubectl port-forward` command keeps running in that terminal — use `tmux`/`screen` if you want it to survive closing your SSH session.

## 4. Verifying and Troubleshooting

```bash
kubectl get pods -n dev -l app=caddy -o wide
kubectl describe deployment caddy-deployment -n dev
kubectl logs -n dev -l app=caddy
```

## 5. Cleanup

```bash
kubectl delete -f service.yml
kubectl delete -f deployment.yml
kubectl delete -f namespace.yml
```

## 6. Notes

- The manifests use the official `caddy:2-alpine` image, which ships with a default `Caddyfile` serving a static page on port 80 — no extra config is required to see it running.
- To serve your own site or reverse-proxy config, mount a custom `Caddyfile` via a `ConfigMap` into `/etc/caddy/Caddyfile` on the Deployment's pod template.
- Deleting the `dev` namespace (`kubectl delete -f namespace.yml`) also removes everything inside it (Deployment, Service, pods), so the individual deletes above are only needed if you want to keep the namespace around.
