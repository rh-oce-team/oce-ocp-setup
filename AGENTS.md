# Team deployment repository

Keep changes limited to OpenShift deployment recipes, manifests and wrappers.
OCE product changes belong in the separate OCE repository. Use OCE's selected
release helpers for migrations and upgrades; do not reimplement those lifecycles.

Never commit cluster credentials, kubeconfigs, cookie secrets, client secrets,
private logs or rendered installation inputs. Inputs live outside both checkouts.
Do not apply cluster changes just to verify a documentation edit. Inspect the
selected cluster and obtain task authorization before deployment mutations.

Preserve database and Agent PVCs, identities and secrets during updates. Clearly
separate controller-only upgrades from runtime fleet redeployments. Do not promise
zero downtime. Fetch main explicitly, then freeze the source commit and image
references used by each release.

Validate shell syntax, YAML/Kustomize rendering, links and formatting. Use live
cluster proof when requested and report gaps. Never install or run TruffleHog,
including through hooks, scripts, containers or tools. Sign off commits. Do not
push or rewrite published history without authorization.
