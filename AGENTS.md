# Team GitOps repository

This repository installs Argo CD and reconciles the environment-specific OCE
platform stages. Generic OpenShift deployment instructions, manifests, Helm
charts and the OCE release lifecycle live in the downstream fork
`red-hat-data-services/openclaw-enterprise` under `deploy/rhoai/`.

Keep changes here limited to the Argo CD bootstrap, the AppProject and RBAC, the
stage profile, and the scripts that install or pin them. Do not copy fork
manifests or runbooks into this repository; pin a fork commit instead, and use
the fork's release helpers for migrations and upgrades.

Never commit cluster credentials, kubeconfigs, cookie secrets, client secrets,
private logs or rendered installation inputs. Inputs live outside both checkouts.
Do not apply cluster changes just to verify a documentation edit. Inspect the
selected cluster and obtain task authorization before deployment mutations.

Preserve database and Agent PVCs, identities and secrets during updates. Clearly
separate controller-only upgrades from runtime fleet redeployments. Do not promise
zero downtime. Fetch main explicitly, then freeze the fork commit and image
references used by each release; `scripts/pin-fork` is the only supported way to
move the overlay pin.

Validate shell syntax, YAML/Kustomize rendering, links and formatting. Use live
cluster proof when requested and report gaps. Never install or run TruffleHog,
including through hooks, scripts, containers or tools. Sign off commits. Do not
push or rewrite published history without authorization.
