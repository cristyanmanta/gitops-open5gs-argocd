# Open5GS GitOps PoC

GitOps-managed 5G core deployment using Argo CD, Helm, and k3s.

## Bootstrap

On the target cluster (with Argo CD already installed):

```bash
kubectl apply -f bootstrap/root-app.yaml
```

That single command bootstraps everything else. Argo CD reads `apps/` and creates
each workload Application in dependency order via sync waves.

## Architecture

- **mongodb** (sync wave 0) — subscriber database
- **open5gs** (sync wave 1) — 5G core network functions (AMF, SMF, UPF, etc.)
- **open5gs-webui** (sync wave 2) — subscriber management UI

## How to change things

Edit the values inline in `apps/<workload>.yaml`, commit, push. Argo CD
auto-syncs within 3 minutes.
