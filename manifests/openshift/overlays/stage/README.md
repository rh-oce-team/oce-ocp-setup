# Stage cluster profile

The environment-specific overlay for the five stages Argo CD reconciles from
`gitops/clusters/stage`. The generic manifests live in the downstream fork; each
stage here references them as a Kustomize remote base pinned by commit. Move
every base with:

```bash
scripts/pin-fork <40-character commit SHA>
```

The committed values are the live stage profile. The
`oce-ocp-setup/environment: stage` label and the 5Gi PostgreSQL volume are the
only deviations from the fork base; the volume matches the live cluster, and a
storage change must be deliberate.

| Stage        | Argo Application  | Wave | Namespace              | Deviation                |
| ------------ | ----------------- | ---- | ---------------------- | ------------------------ |
| `operators`  | `oce-operators`   | 0    | `openclaw-postgres`    | environment label        |
| `database`   | `oce-database`    | 1    | `openclaw-postgres`    | environment label, 5Gi   |
| `admission`  | `oce-admission`   | 1    | `openclaw-system`      | environment label        |
| `openshell`  | `oce-openshell`   | 2    | `openshell-system`     | environment label        |
| `routing`    | `oce-routing`     | 3    | `envoy-gateway-system` | environment label        |

`control-plane`, `control-ui`, and `control-ui-access` stay in the fork runbook.

## Why `labels` and not `commonLabels`

The `labels` transformer with `includeSelectors: false` and
`includeTemplates: false` adds the environment label to object metadata only.
`commonLabels` would also inject it into every NetworkPolicy `podSelector` and
`namespaceSelector`. The Pods those selectors target are created outside these
overlays (the `oce` Helm release, OCE-managed workspace Pods, and the OpenShell
gateway release), so the selectors would match nothing and the policies would
deny the traffic they exist to allow.

## Values that are not committed

Cluster-specific values stay in the private input directory described in the
fork runbook: hostnames, API and PostgreSQL egress addresses, router CIDRs, image
digests, and every Secret.
