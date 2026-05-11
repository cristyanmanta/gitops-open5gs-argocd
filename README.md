# Open5GS GitOps PoC with Argo CD

GitOps-managed deployment of an Open5GS 5G core network on Kubernetes, using Argo CD and Helm. Demonstrates patterns applicable to telco-grade CNF onboarding on Red Hat OpenShift.

**Live demo:**
- Argo CD UI → [https://gitops.cristyanmanta.dev](https://gitops.cristyanmanta.dev)
- Open5GS WebUI → [https://open5gs.cristyanmanta.dev](https://open5gs.cristyanmanta.dev)

## What's deployed

- **MongoDB** — subscriber database (Bitnami chart, OCI)
- **Open5GS** — 18 network functions: AMF, SMF, UPF, NRF, AUSF, UDM, UDR, PCF, NSSF, BSF, SCP, MME, HSS, PCRF, SGW-C, SGW-U (Gradiant chart, OCI)
- **Open5GS WebUI** — subscriber management UI (vendored chart in this repo)

All running on a single-node k3s cluster on GCP, managed end-to-end by Argo CD pulling from this Git repo.

## Repo structure

```
.
├── bootstrap/root-app.yaml      # Apply once to bootstrap everything
├── apps/                        # Argo CD Applications (one per workload)
│   ├── mongodb.yaml
│   ├── open5gs.yaml
│   └── open5gs-webui.yaml
└── charts/
    └── open5gs-webui/           # Vendored chart with patched init image
```

## Bootstrap

```bash
kubectl apply -f bootstrap/root-app.yaml
```

That's the only manual step. Argo CD reads `apps/` and deploys everything in sync-wave order: MongoDB → Open5GS → WebUI.

## Making changes

Edit any file in `apps/`, commit, push. Argo CD syncs within ~3 minutes.

```bash
git add apps/
git commit -m "Your change"
git push origin main
```

## Architecture

```
Browser
   │
   │ HTTPS (Cloudflare-managed cert)
   ▼
Cloudflare edge (TLS mode)
   │
   │ HTTP
   ▼
Caddy reverse proxy (port 80, host-level)
   │
   ├── gitops.cristyanmanta.dev   ──▶  argocd-server NodePort :30443
   └── open5gs.cristyanmanta.dev  ──▶  open5gs-webui NodePort :30999
```

## What this demonstrates

- **App-of-Apps pattern** — one root Application bootstraps three children
- **Sync waves** — MongoDB before Open5GS before WebUI
- **OCI Helm charts** — modern distribution format from Docker Hub
- **Vendored chart pattern** — WebUI chart patched in-repo because upstream hardcodes a deprecated image
- **Cross-chart integration** — `fullnameOverride` bridges MongoDB service name with Open5GS expectations
- **Self-heal + drift detection** — manual cluster changes auto-revert to Git state
- **Rollback via `git revert`** — same mechanism as deploy

## Notable engineering decisions

**Vendored WebUI chart.** Upstream chart hardcodes `bitnami/mongodb:4.4.1-debian-10-r39` as init container image. That tag was removed in Bitnami's 2025 BSI transition. Chart vendored to `charts/open5gs-webui/` and patched to use `bitnamilegacy/mongodb:4.4.15`.

**MongoDB `fullnameOverride`.** Open5GS hardcodes `mongodb://open5gs-mongodb/open5gs` as its DB URI. Bitnami chart defaults to a different service name. Resolved via values override, no chart forking.

**Disabled k3s's bundled Traefik.** k3s ships with Traefik as the default Ingress controller. For this PoC, routing is handled by Caddy on the host (simpler for 2 hostnames). Removing the Traefik LoadBalancer Service was necessary to free port 80 for Caddy. In production OpenShift, this role would be filled by OpenShift Routes with edge TLS termination.

## What would change for real telco CNFs

This is a PoC with a research-grade 5G core. Production telco deployment would add:

- **SR-IOV** for line-rate data plane
- **PerformanceProfile + Node Tuning Operator** for CPU isolation, hugepages, NUMA pinning
- **Multus CNI** for multi-NIC pods (N2/N3/N4/N6 on separate networks)
- **Operator-delivered CNFs** via OLM/OperatorHub instead of raw Helm
- **OpenShift GitOps** (productised Argo CD) and **OpenShift Routes** in place of upstream Argo CD and Caddy
- **RHACM** for fleet-wide policy and multi-cluster placement
- **cert-manager** for declarative TLS automation
- **Lawful intercept**, charging integration, signalling firewalls — not implemented in Open5GS

The GitOps mechanics shown here (sync waves, declarative state, drift detection, vendored charts where needed) apply identically. Only the workload complexity and platform-level operators differ.

## Infrastructure

- k3s on a GCE `e2-standard-2` VM 
- Caddy reverse proxy + Cloudflare proxied DNS for hostname routing and TLS
  
