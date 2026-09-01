# GitOps

Flux CD manages this cluster. Every change is a commit to this repo.

## Environment

| | |
|---|---|
| Cluster | kubeadm, 3 control-plane nodes (k8s-1/2/3) |
| Kubernetes | v1.35.8 |
| CNI | Cilium, pod CIDR 10.244.0.0/16 |
| API VIP | 10.0.0.25:6443, kube-vip |
| Flux | v2.9.5 |
| Host | ASUS NUC 15 Pro, Proxmox, 3 VMs |

## Repository layout

```
clusters/homelab/
├── flux-system/          # Flux's own manifests, written by bootstrap
│   ├── gotk-components.yaml
│   ├── gotk-sync.yaml
│   └── kustomization.yaml
└── apps/                 # Workload manifests
docs/
```

Flux watches `clusters/homelab` and reconciles every 60 seconds.

## Install

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
flux check --pre
```

```bash
export GITHUB_TOKEN=<fine-grained token: Contents RW, Administration RW>
export GITHUB_USER=stefan-bc

flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=homelab-gitops \
  --branch=main \
  --path=clusters/homelab \
  --personal
```

Bootstrap installs `source-controller`, `kustomize-controller`, `helm-controller` and `notification-controller` into `flux-system`, adds a deploy key to the repo, and commits its own manifests.

## Problem: controllers would not schedule

All four controllers stayed `Pending` after bootstrap.

```
0/3 nodes are available: 3 node(s) had untolerated taint(s).
```

The cluster has three control-plane nodes and no workers. Each carries `node-role.kubernetes.io/control-plane:NoSchedule`. The Flux controllers declare no matching toleration, so nothing could be placed.

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

Controllers scheduled immediately. All three nodes now run workloads alongside etcd and the API server. This is a single-host lab, so the isolation loss is accepted. Adding a dedicated worker is the correct fix and is deferred.

## Verification

Committed a `Namespace` manifest to `clusters/homelab/apps/`. No `kubectl apply` was run.

```
$ kubectl get ns demo
NAME   STATUS   AGE
demo   Active   5m36s
```

Deleted it out of band to test drift correction.

```
$ kubectl delete ns demo
$ flux reconcile kustomization flux-system --with-source
✔ applied revision main@sha1:410aee2
$ kubectl get ns demo
NAME   STATUS   AGE
demo   Active   1s
```

Git is authoritative. Manual changes are reverted on the next reconcile.

## Operating

```bash
flux get kustomizations                                  # sync status
flux get sources git                                     # repo status
flux reconcile kustomization flux-system --with-source   # force sync
flux logs --follow                                       # controller logs
```

## Constraints

Repository is public. No secrets are committed. CloudNativePG and cert-manager both need credentials, so SOPS or Sealed Secrets is a prerequisite for either.
