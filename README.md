# OCE GitOps on OpenShift

This repository installs Argo CD on an OpenShift cluster and reconciles the
environment-specific OpenClaw Enterprise (OCE) platform stages. The generic
OpenShift deployment instructions, manifests, Helm chart, and upgrade lifecycle
live in the downstream fork,
[`red-hat-data-services/openclaw-enterprise`](https://github.com/red-hat-data-services/openclaw-enterprise),
under `deploy/rhoai/`.

Start with the fork's
[deployment guide](https://github.com/red-hat-data-services/openclaw-enterprise/blob/main/deploy/rhoai/README.md).
This repository covers only the GitOps layer.

## What this repository owns

- The OpenShift GitOps operator Subscription, Argo CD controller RBAC, the
  `oce-stages` AppProject, and the resource health checks.
- A root Application and one Application per platform stage: operators,
  database, admission, openshell, routing.
- The stage profile: the environment label and the placeholder PostgreSQL
  volume, layered on generic bases pinned to a fork commit.

Secrets, the `oce` Helm release and its upgrade lifecycle, SQL bootstrap,
`grant-tenant-access`, image builds, and the control-plane, control-ui and
control-ui-access stages stay in the fork runbook.

## Bootstrap a cluster

```bash
scripts/bootstrap-argocd --env stage --dry-run
scripts/bootstrap-argocd --env stage
```

See [the GitOps guide](docs/gitops.md) for what the script applies and the
ownership boundary.

## Consume a new fork revision

1. Read the fork's `deploy/rhoai/README.md` and select the commit to deploy.
2. Pin the stage profile:

   ```bash
   scripts/pin-fork <40-character commit SHA>
   ```

3. Prepare the cluster-specific inputs outside both checkouts, following the
   fork runbook.
4. Pause automatic sync, push the pin change, then resume with
   `scripts/promote-stage --pause` and `scripts/promote-stage --resume` so the
   stages apply in order. See [the GitOps guide](docs/gitops.md).

## Layout

```text
gitops/
├── bootstrap/              Operator, RBAC, AppProject, health checks
└── clusters/
    ├── stage/              Root Application and five stage Applications
    └── prod/               Placeholder
manifests/openshift/overlays/stage/
                            Stage profile on pinned fork bases
scripts/
├── bootstrap-argocd        Install Argo CD and register the root Application
└── pin-fork                Move the overlays to a fork commit
```

## Verification

```bash
oc kustomize gitops/bootstrap
oc kustomize gitops/clusters/stage/apps
python3 -m unittest discover -s tests
```

`oc kustomize manifests/openshift/overlays/stage` needs network access because
its bases come from the pinned fork commit.
