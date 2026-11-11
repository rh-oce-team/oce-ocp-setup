# Deploy OCE with Argo CD (GitOps)

This repository installs Argo CD on a cluster and reconciles the
environment-specific OCE platform stages. The generic OpenShift deployment
instructions, manifests, Helm chart, and upgrade lifecycle live in the
downstream fork,
[`red-hat-data-services/openclaw-enterprise`](https://github.com/red-hat-data-services/openclaw-enterprise),
under `deploy/rhoai/`.

Read the fork's [deployment guide](https://github.com/red-hat-data-services/openclaw-enterprise/blob/main/deploy/rhoai/README.md)
first. This guide covers only what this repository adds: the Argo CD bootstrap,
the stage profile, and the pinned fork revision Argo renders.

## What Argo owns

| Owned by Argo CD                                                       | Stays in the fork runbook                              |
| ---------------------------------------------------------------------- | ------------------------------------------------------ |
| `operators`, `database`, `admission`, `openshell`, `routing` stages    | `control-plane`, `control-ui`, `control-ui-access`     |
| The OpenShift GitOps operator Subscription, controller RBAC, AppProject | The `oce` Helm release and its upgrade lifecycle       |
| Stage drift correction (`selfHeal`)                                    | The `openshell-gateway`, `openshell-dashboard` and Envoy Helm releases |
|                                                                        | System Secrets, SQL bootstrap, `grant-tenant-access`, image builds, `occ` operations |

Argo CD renders Helm charts with `helm template`. It cannot run the OpenShell
gateway post-renderer, its `lookup()` guards are inert, and the OCE helpers
mutate the `oce` release (`attemptId`, `upgradeAttemptId`) while running
migrations and fleet redeploys. Owning that release from GitOps would
reimplement the lifecycle that AGENTS.md assigns to OCE, so it stays in the
fork runbook.

## Layout

```text
gitops/
├── bootstrap/              One-time cluster bootstrap (operator, RBAC, AppProject, health patch)
├── clusters/
│   ├── stage/              Root Application and one Application per platform stage
│   └── prod/               Placeholder; not deployable yet
manifests/openshift/overlays/stage/
                            Stage profile: five overlays on the pinned fork bases
scripts/
├── bootstrap-argocd        Install Argo CD and register the root Application
└── pin-fork                Move every stage overlay to a fork commit
```

Each stage overlay references the generic base in the fork as a Kustomize
remote base pinned to a commit, for example:

```yaml
resources:
  - github.com/red-hat-data-services/openclaw-enterprise//deploy/rhoai/manifests/openshift/base/operators?ref=<sha>
```

The generic manifests are not copied here. This repository owns only the
environment label and the stage sizing patch.

## Bootstrap a cluster

Review the operator channel first, then run the wrapper from the repository
root against the intended cluster:

```bash
oc -n openshift-marketplace get packagemanifest openshift-gitops-operator \
  -o jsonpath='{.status.channels[*].name}{"\n"}'

scripts/bootstrap-argocd --env stage --dry-run

scripts/bootstrap-argocd --env stage
```

The script applies the Subscription, polls for the default Argo CD instance and
waits for it to become Available, applies the RBAC and AppProject, patches the
resource health checks into the `ArgoCD` custom resource, and registers the
root Application. Rerunning it re-asserts the committed values, including the
whole `spec.resourceHealthChecks` list: fold any cluster-side health
customizations into `gitops/bootstrap/argocd-resource-health.patch.yaml` first.
It does not touch Secrets, the database, or OCE.

On a cluster already deployed with the fork runbook, Argo starts syncing as
soon as the Applications exist, so the committed manifests do not provide an
`argocd app diff` gate before the first sync. Compare the stage render with the
live objects first, or redeploy the stage cluster from this layout. Do not
bootstrap an adopted cluster until that comparison is clean.

## Consume a new fork revision

1. Read the fork's `deploy/rhoai/README.md` and select the commit to deploy.
   Record that commit and the image digests from its `resolve-images` step in
   the private release record.
2. Move the stage profile to that commit:

   ```bash
   scripts/pin-fork <40-character commit SHA>
   ```

3. Prepare the cluster-specific inputs the fork runbook lists, in a private
   directory outside both checkouts. The bootstrap Secrets, SQL bootstrap,
   `grant-tenant-access`, the `oce` Helm release, and the control-plane stages
   stay manual.
4. Render the new pin and confirm every object kind and namespace is covered by
   `gitops/bootstrap/app-project.yaml`. A new cluster-scoped kind blocks the
   whole sync wave, so fix the project before pushing.
5. Commit the overlay change and push. Argo reconciles the five stages from
   this repository and renders their bases from the pinned fork commit.

## Sync order and policy

| Application     | Wave | Stage        | Auto-sync | Gated by                     |
| --------------- | ---- | ------------ | --------- | ---------------------------- |
| `oce-operators` | 0    | `operators`  | yes       | Subscription health check    |
| `oce-database`  | 1    | `database`   | yes       | PostgresCluster health check |
| `oce-admission` | 1    | `admission`  | yes       | namespaces and SCC applied   |
| `oce-openshell` | 2    | `openshell`  | yes       | wave 1 Healthy               |
| `oce-routing`   | 3    | `routing`    | yes       | wave 2 Healthy               |

Prune is off everywhere: removing a resource from Git never deletes it from the
cluster until an operator does so by hand. The root gates each wave on the child
Applications' health (`argoproj.io/Application`), the Subscription check holds
wave 0 until every CSV is at its latest known version, and the retry backoff
remains the backstop when a healthy Subscription still has not registered its
CRDs.

## Revision and freeze

The Applications track this repository's `main`. The fork bases are pinned per
overlay by commit SHA; `scripts/pin-fork` is the only supported way to move
them. The AppProject's `sourceRepos` covers Application sources, not these
remote bases; the commit pin and the overlay review are the control.

Argo fetches each base from GitHub at render time. The pin makes the content
immutable, but a deleted, renamed, or private fork blocks every stage render
until the overlay is repointed or the base is mirrored. Keep a mirrored copy of
the pinned commit for incident response, and treat a fork availability event as
a stage incident.

Prod will track its own branch or tag once the prod cluster exists so a stage
merge cannot reach it.

## Stage data

`manifests/openshift/overlays/stage` commits only two deviations from the fork
base: an `oce-ocp-setup/environment: stage` label and a placeholder 10Gi
PostgreSQL volume. The label is metadata only; every NetworkPolicy selector
stays identical to the base, because the Pods those selectors target are
created outside these overlays.

These values are cluster-discovered and stay in `OCC_INPUT_DIRECTORY`, outside
both repositories: the Console Route hostname, the Kubernetes API and
PostgreSQL egress address inputs, the router CIDRs, the private gateway host,
image digests, and every Secret. See
[the stage profile](../manifests/openshift/overlays/stage/README.md).

## Deferred: charts from the downstream fork

Consuming the fork Helm charts through Argo CD multi-source is still a
follow-up: the chart path from the fork plus `valueFiles` from this repository
(`$values/...`). The dashboard chart is the first candidate. Do not add the
`oce` chart; see the ownership table.

## Version note

OpenShift GitOps publishes `gitops-X.Y` channels and has no `stable` channel;
`latest` installed 1.22.3 on OpenShift 4.22 for this stage deployment.
Multi-source Applications need Argo CD 2.6 or later, so every current channel
works for the fork-chart follow-up.

The stage environment was exercised on OpenShift 4.22 with OpenShift GitOps
1.22.3. All five Applications reached Synced/Healthy in wave order, and the root
became Healthy once Envoy Gateway accepted the GatewayClass. That run used the
previous local bases; the first sync after the remote-base switch is the
outstanding live verification, and it must confirm the repo-server can fetch the
pinned fork commit.

## Verification

The GitOps tree renders offline. The stage overlay needs network access because
its bases are fetched from the pinned fork commit:

```bash
oc kustomize gitops/bootstrap
oc kustomize gitops/clusters/stage/apps
python3 -m unittest discover -s tests
```

The tests assert the Application structure, the AppProject scope, the RBAC
verbs, the Lua health checks, the stage overlay shape, and the pinned revision.
