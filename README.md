# OCE setup on OpenShift

Deployment recipes, Helm inputs and Kustomize manifests for OpenClaw
Enterprise (OCE) on OpenShift. This repository contains no OCE application code.
It supports dedicated Codex Agents through OpenShell and an optional
Keycloak-protected OpenShell dashboard.

## Start here

Use a separate OCE checkout. Keep it on the release you intend to build or deploy;
fetching main selects a candidate, not permission to upgrade a running cluster.

```bash
# REQUIRED: absolute path to your OCE source checkout.
export OCE_CHECKOUT='/absolute/path/to/openclaw-enterprise'

# Run commands from this setup repository root.
scripts/build-occ
```

- **Fresh cluster:** [cluster preparation](docs/cluster.md), then [OCC installation](docs/install.md).
- **First Agent:** [private routing](docs/routing.md), [Agent creation](docs/agent.md), and [Control UI](docs/control-ui.md).
- **Another Agent:** [additional-Agent recipe](docs/agents.md), using the existing images.
- **Scoped owners:** [owner access](docs/owner-access.md); cluster-admin access remains an explicit alternative.
- **Dashboard:** [Keycloak and dashboard recipe](docs/dashboard.md).
- **Updating OCE:** [build and upgrade workflow](docs/upgrades.md).
- **Manifest layout:** [Kustomize stages](docs/kustomize.md).
- **Team Claw proposal:** [collaboration and identity plan](docs/team-claw-plan.md), pending agreement and implementation.

## Repositories and private state

`manifests/openshift/` contains cluster and Agent assets. `manifests/dashboard/`
contains the optional dashboard and identity-provider assets. `scripts/` prepares
inputs or invokes the selected OCE checkout's tools. OCE owns its Helm chart,
production examples, startup validator, migrations, bootstrap-volume helper and
upgrade workflow; use those from `OCE_CHECKOUT` rather than vendoring them here.

Keep kubeconfigs, credentials, rendered cluster inputs, source archives and
verification evidence outside both repositories. Share a configuration-only
bundle internally; supply secrets separately. Neither build commands nor fetching
main should silently alter installed images. Record source commits, chart versions
and image digests for every release.

The recipes were exercised on OpenShift 4.22.17, Kubernetes 1.35.8 and amd64,
with OpenShell 0.1.3, Crunchy 5.8.9 and single-instance PostgreSQL without backups.
This is a development setup, not a production qualification. Dashboard OIDC is
separate from OCC/Agent login. Existing OpenShell private-network authentication
remains development-only and must retain its network isolation.
