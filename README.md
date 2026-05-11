# Open5GS GitOps PoC with Argo CD

GitOps-managed deployment of an Open5GS 5G core network on Kubernetes, using Argo CD and Helm. Demonstrates patterns applicable to telco-grade CNF onboarding on Red Hat OpenShift.

## What's deployed

- **MongoDB** — subscriber database (Bitnami chart from OCI)
- **Open5GS** — 18 network functions: AMF, SMF, UPF, NRF, AUSF, UDM, UDR, PCF, NSSF, BSF, SCP, MME, HSS, PCRF, SGW-C, SGW-U (Gradiant chart from OCI)
- **Open5GS WebUI** — subscriber management (vendored chart from this repo)

All deployed on k3s, managed by Argo CD pulling from this Git repo.

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

## What this demonstrates

- **App-of-Apps** — one root Application bootstraps three children
- **Sync waves** — MongoDB before Open5GS before WebUI
- **OCI Helm charts** — modern distribution format
- **Vendored chart pattern** — WebUI chart patched in-repo because upstream hardcodes a deprecated image
- **Cross-chart integration** — `fullnameOverride` bridges MongoDB service name with Open5GS expectations
- **Self-heal + drift detection** — manual cluster changes auto-revert to Git state
- **Rollback via `git revert`** — same mechanism as deploy

## Notable engineering decisions

**Vendored WebUI chart.** Upstream chart hardcodes `bitnami/mongodb:4.4.1-debian-10-r39` as init container image. That tag was removed in Bitnami's 2025 BSI transition. Chart vendored and patched to use `bitnamilegacy/mongodb`.

**MongoDB `fullnameOverride`.** Open5GS hardcodes `mongodb://open5gs-mongodb/open5gs` as its DB URI. Bitnami chart defaults to a different service name. Resolved via values override, no chart forking.

## What would change for real telco CNFs

This is a PoC. Production telco deployment would add SR-IOV, PerformanceProfile, Multus, OLM-delivered operators, RHACM for fleet management, and OpenShift GitOps in place of upstream Argo CD. The GitOps mechanics shown here apply identically.

## Infrastructure

- k3s on a GCE `e2-standard-2` VM
- Argo CD UI on NodePort 30443
