# Argo CD (GitOps) application layout

An app-of-apps layout that lets Argo CD read this repository and own the
Kustomize stages that are safe to reconcile continuously. It adds Argo on top
of the existing procedures; it does not replace them. Read
[../../README.md](../../README.md) for the stage inventory and
[../../../docs/kustomize.md](../../../docs/kustomize.md) for the manual flow
this wraps.

## What Argo owns — and what it must not

| Application      | Stage source                        | Wave | Auto-sync | Prune | Self-heal |
| ---------------- | ----------------------------------- | ---- | --------- | ----- | --------- |
| `oce-operators`  | `overlays/test/operators`           | 0    | yes       | off   | yes       |
| `oce-database`   | `overlays/test/database`            | 1    | **no**    | off   | —         |
| `oce-admission`  | `overlays/test/admission`           | 1    | yes       | off   | yes       |
| `oce-openshell`  | `overlays/test/openshell`           | 2    | yes       | off   | yes       |
| `oce-routing`    | `overlays/test/routing`             | 3    | yes       | off   | yes       |

Not adopted, on purpose:

- `control-plane`, `control-ui`, `control-ui-access` — their manifests carry
  cluster-specific values that stay out of this repository.
- The three Helm releases (`oce`, `openshell-gateway`, `openshell-dashboard`).
  The OpenShell gateway requires a Helm post-renderer
  (`scripts/openshell-mtls-post-render`), which Argo CD Helm sources cannot
  run, and the OCE upgrade lifecycle (migrations, writer stop, fleet
  redeploy) belongs to the selected OCE release's helpers, per AGENTS.md.
- Secrets, database SQL bootstrap, `prepare-bootstrap-volume`,
  `grant-tenant-access`, `occ` CLI operations and image builds. These remain
  documented runbook steps.

Prune stays off everywhere for adoption; enable it per stage only after one
clean reconciliation cycle. No `resources-finalizer` is set on the
Applications, so deleting an Application never cascades to its resources.

## Files

```text
gitops-operator.yaml        OpenShift GitOps operator Subscription (bootstrap, by hand)
rbac.yaml                   Exact ClusterRole grants for the Argo controller
root.yaml                   Root Application (app of apps), applied once by hand
applications/base/          One Application per auto-synced stage, sync-wave annotated
applications/test/          OpenShift profile: Applications in openshift-gitops, plus
                            the manual-sync database Application
applications/local/         Verification profile: Applications in argocd, git:// source
verification/root-local.yaml Root Application for the local k3d verification
```

## Bootstrap on OpenShift (once, by hand)

```bash
# Verify the channel and pin the CSV current for the target cluster first.
oc -n openshift-marketplace get packagemanifest openshift-gitops-operator

oc apply -f manifests/openshift/argocd/gitops-operator.yaml

oc -n openshift-gitops wait --for=jsonpath='{.status.health}'=Healthy \
  argocd/openshift-gitops --timeout=5m

oc apply -f manifests/openshift/argocd/rbac.yaml

oc apply -f manifests/openshift/argocd/root.yaml
```

Argo then creates the stage Applications wave by wave: a wave's Applications
must be `Healthy` before the next wave is created, mirroring the stage order
in [../../../docs/kustomize.md](../../../docs/kustomize.md).

Adoption, not first application: on a cluster already deployed with the
runbooks, the stage manifests are identical to what `oc apply -k` created, so
each Application should appear `Synced` with an empty diff. Check
`argocd app diff <application>` (or the UI) before enabling anything beyond
the defaults above.

Optional: add a `PostgresCluster` health check so the database wave gate is
the cluster's actual readiness, through the ArgoCD CR:

```yaml
resourceCustomizations: |
  postgres-operator.crunchydata.com:
    PostgresCluster:
      health.lua: |
        hs = {}
        if obj.status == nil or obj.status.instances == nil then
          hs.status = "Progressing"
          return hs
        end
        for _, instance in ipairs(obj.status.instances) do
          if instance.readyReplicas ~= instance.replicas then
            hs.status = "Progressing"
            return hs
          end
        end
        hs.status = "Healthy"
        return hs
```

## Local pull-mode verification (k3d)

The `applications/local` profile and the `overlays/local` stage subsets verify
the whole mechanism on a disposable plain-Kubernetes cluster: Argo pulls this
repository over the git protocol from the local host and applies the portable
stage subsets in wave order. No OpenShift-specific API (OLM, SCC, Routes) and
no cluster credentials are involved.

Prerequisites: `k3d`, `kubectl`, `docker`, `git`. Pin the Argo CD and
Gateway API versions to reviewed releases; the commands below record the
versions exercised at the time of writing.

**1. Platform prep.** Create the cluster and install Argo CD and the Gateway
API CRDs (on OpenShift the platform supplies the Gateway API CRDs; locally
they are prep, like the Ingress Operator provides them). Use server-side
apply for Argo CD: the ApplicationSet CRD exceeds the client-side apply
annotation limit on current Kubernetes. Exercised with Argo CD v3.6.1,
Gateway API v1.7.0 and k3s v1.35.9:

```bash
k3d cluster create oce-argocd

kubectl create namespace argocd

kubectl apply --server-side --force-conflicts -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/v3.6.1/manifests/install.yaml

kubectl -n argocd rollout status deploy/argocd-server --timeout=5m

kubectl apply -f \
  https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.7.0/standard-install.yaml
```

Host quirks seen during verification, in case they recur: a host whose root
user exhausted the inotify instance limit (`fs.inotify.max_user_instances`)
starves containerd's CRI watcher inside new k3d containers (`too many open
files`), and docker's generated node DNS pointed at a resolver that was not
listening, so kubelet image pulls failed until `/etc/resolv.conf` inside the
node container was pointed at a working nameserver.

**2. Serve the branch.** Argo must reach the repository from inside the k3d
network, so serve a bare clone through `git daemon` on the host;
`host.k3d.internal` resolves to the host from inside the cluster:

```bash
verification=$HOME/.local/state/oce-argocd-verification
install -d -m 700 "$verification"

git clone --bare . "$verification/oce-ocp-setup.git"

git -C "$verification/oce-ocp-setup.git" daemon \
  --base-path="$verification" --export-all --reuseaddr --port=9418 &
```

The clone's HEAD is the checked-out branch; `applications/local` tracks
`HEAD`, so pushing or re-cloning updates what Argo pulls.

**3. Apply the root and watch the waves.**

```bash
kubectl apply -f manifests/openshift/argocd/verification/root-local.yaml

kubectl get applications -n argocd -w
```

Expected progression: `oce-operators` (namespaces) becomes
`Synced/Healthy`, then `oce-admission`, then `oce-openshell` (its
`openclaw-system` NetworkPolicy is admitted only because the admission wave
already created that namespace), then `oce-routing` (namespace and RBAC; the
local overlay drops the GatewayClass, whose health waits for the Envoy
controller the runbooks install), and finally `oce-root` reports
`Synced/Healthy`. Prune is off by design: a resource removed from the
repository stays in the cluster until it is deleted once by hand, and the
Application reports `OutOfSync` until then.

**4. Verify the resources and self-heal.**

```bash
kubectl get namespaces \
  openclaw-postgres cert-manager-operator openclaw-system \
  openshell-system envoy-gateway-system

kubectl -n openshell-system get networkpolicy
kubectl get clusterrole oce-openshell-workspace-writer
kubectl get runtimeclass openshell-sandbox
kubectl get role,rolebinding -n openclaw-system
kubectl get role,rolebinding -n envoy-gateway-system

# Delete one resource out of band; self-heal must restore it.
kubectl -n openshell-system delete networkpolicy openshell-gateway-ingress
kubectl -n openshell-system get networkpolicy openshell-gateway-ingress
```

**5. Verify pull mode.** Commit a change to a `overlays/local` stage and
update the bare clone; Argo refreshes the repository within three minutes
(no webhook) and applies it:

```bash
git -C "$verification/oce-ocp-setup.git" fetch -f origin \
  "$(git rev-parse --abbrev-ref HEAD):$(git rev-parse --abbrev-ref HEAD)"
```

**Teardown.**

```bash
k3d cluster delete oce-argocd
kill %1   # the git daemon
```

The verification touches only the disposable k3d cluster and the private
directory above; it never contacts the deployment cluster or applies
OpenShift-only resources.
