# telco-platform-gitops

GitOps repo for a 5G Standalone core ([Open5GS](https://open5gs.org)) on Kubernetes. ArgoCD
syncs everything here to the cluster, and the Open5GS Operator reconciles the custom resources
into running network functions (AMF, SMF, UPF, PCF, NRF, AUSF, UDM, UDR, NSSF, BSF, SCP, plus
MongoDB). A network slice or subscriber change is a commit to this repo.

## What's here

| Path | What it does |
|---|---|
| `apps/observability-app.yaml` | ArgoCD `Application` for `kube-prometheus-stack` (Prometheus, Grafana, Alertmanager). `serviceMonitorSelectorNilUsesHelmValues: false` so Prometheus picks up ServiceMonitors from every namespace |
| `apps/open5gs-operator-app.yaml` | ArgoCD `Application` for the Gradiant `open5gs-operator` Helm chart, pinned to `1.0.7` |
| `apps/open5gs-cr-app.yaml` | ArgoCD `Application` that syncs `overlays/local` from this repo |
| `base/open5gs-cr/` | The `Open5GS` custom resource: PLMN `999/70`, TAC `0001`, two slices (`SST 1 / SD 0x111111`, `SST 2 / SD 0x222222`), ServiceMonitors on AMF, PCF, UPF |
| `base/open5gs-subscribers/` | An `Open5GSUser` subscriber on the SST 2 slice |
| `base/open5gs-operator-metrics/` | `ServiceMonitor` for the Operator's controller-runtime metrics + `ClusterRoleBinding` letting Prometheus through kube-rbac-proxy |
| `overlays/local/` | Composes the three bases for the local cluster and labels them `environment: local` |

## Running it

Needs a Kubernetes cluster (built on Docker Desktop's Kubernetes) with ArgoCD installed.

```bash
git clone https://github.com/kingswanzy2020/telco-platform-gitops.git
cd telco-platform-gitops

# 1. Observability first: the Operator needs the ServiceMonitor CRD to exist
kubectl apply -f apps/observability-app.yaml
argocd app sync observability

# 2. The Operator
kubectl apply -f apps/open5gs-operator-app.yaml
argocd app sync open5gs-operator

# 3. The 5G core itself (CRs + subscriber + operator metrics)
kubectl apply -f apps/open5gs-cr-app.yaml
argocd app sync open5gs-cr

kubectl get pods -n open5gs
```

The Applications don't enable automated sync, so each change is synced explicitly. Add
`syncPolicy.automated` if you want ArgoCD to apply commits on its own.

**Fork first if you want to push changes.** `apps/open5gs-cr-app.yaml` points `repoURL` at this
repo, so change it to your fork.

### A note on the subscriber credentials

`base/open5gs-subscribers/open5gs-user-sample.yaml` holds a `key` and `opc`. These are the
standard Open5GS/UERANSIM **test values**, on the `999/70` test PLMN, and don't belong to any
real SIM. Don't commit real subscriber keys to Git: use a SealedSecret or an external secret
store instead.

## Write-up

Full write-up — animated architecture diagram, the Git-commit-to-new-network-slice flow, and
the CRD schema-pruning bug behind a permanent OutOfSync — lives in my portfolio repo:
**[Projects / kubernetes / gitops-5g-core](https://github.com/kingswanzy2020/Projects/tree/main/kubernetes/gitops-5g-core)**.
