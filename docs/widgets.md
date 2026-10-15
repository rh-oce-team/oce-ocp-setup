# Enable MCP Apps and dashboard widgets

Run this optional recipe after an Agent's [Control UI](control-ui.md) works.
It uses OCE's supported widget sandbox settings and a separate OpenShift edge
Route to the Gateway's isolated renderer port. Browser traffic uses HTTPS;
the router-to-renderer hop uses HTTP inside the cluster. The NetworkPolicy
admits only the default OpenShift router peers on that port. Choose a different
TLS design when your environment requires encryption on that hop.

The preview hostname is externally reachable without OAuth because it serves
public sandbox shell/renderer assets only. Private widget documents still arrive
through the authenticated Control UI. The preview listener must not serve Gateway
RPCs, Console pages, credentials, or private workspace data.

This recipe assumes this repository's `oce` Helm release in `openclaw-system`,
private Gateway routing, the default OpenShift ingress controller, a namespaced
CA Issuer, and a dedicated Codex Agent. It uses the cluster's existing apps wildcard
DNS and router certificate. Shared-domain OCC cookies require a different preview
domain; the helper refuses that configuration. Stop concurrent Helm/Installation
changes while applying this update. OCC restarts briefly; deploying the Agent
interrupts its connections while retaining its PVCs.

## 1. Prepare private inputs

Keep the cluster login, OCC connection and selected Agent variables from the
[additional-Agent recipe](agents.md). Run from this repository root. Use an OCE
checkout matching the installed controller revision; this procedure changes no
image selections.

```bash
umask 077

: "${OCE_CHECKOUT:?Set the installed OCE source checkout}"

: "${OCC_INPUT_DIRECTORY:?Set the private Agent directory outside Git}"

: "${OCC_NAMESPACE:?Set the OCE Namespace ID, not the Kubernetes namespace}"

: "${AGENT_ID:?Set the existing Agent ID}"

bin/occ agent get "$AGENT_ID"

oc whoami

oc whoami --show-server

# Use a NEW directory. The helper refuses to overwrite rollback inputs.
export WIDGET_OUTPUT_DIRECTORY="$OCC_INPUT_DIRECTORY/widget-preview"

scripts/prepare-widget-preview

scripts/validate-installation "$WIDGET_OUTPUT_DIRECTORY/installation.yaml"

helm template oce "$OCE_CHECKOUT/deploy/helm/openclaw-enterprise" \
  -n openclaw-system -f "$WIDGET_OUTPUT_DIRECTORY/values.yaml" \
  > "$WIDGET_OUTPUT_DIRECTORY/helm-rendered.yaml"

oc kustomize "$WIDGET_OUTPUT_DIRECTORY" \
  > "$WIDGET_OUTPUT_DIRECTORY/resources-rendered.yaml"

oc apply --dry-run=server -k "$WIDGET_OUTPUT_DIRECTORY"
```

Review the private candidate values and Installation against their `before-*`
copies. Only the preview settings and managed Installation checksum should change.
Review `route.yaml` and `network-policy.yaml`: their backend must be this Agent's
Gateway sandbox port, normally `8081`, never its administrative port `8080`.
The helper prepares files only; it does not deploy anything. Keep every output
private, including the live Installation copy.

## 2. Enable the renderer configuration

OCE's preview configuration also provisions its separate private Envoy sandbox
listener and wildcard certificate. Retain these Compute-owned resources. For
browser access, this recipe uses the OpenShift Route directly to the isolated
renderer: the OpenShift 4.22 re-encrypt router did not supply the SNI required by
the hostname-restricted Envoy TLS listener in our trial.

```bash
oc apply -f "$WIDGET_OUTPUT_DIRECTORY/certificate.yaml"

oc -n openclaw-system wait certificate/oce-widget-preview-tls \
  --for=condition=Ready --timeout=120s

export OCC_INSTALLATION_SECRET="$(yq -er \
  '.installation.secretName // "occ-installation-startup"' \
  "$WIDGET_OUTPUT_DIRECTORY/values.yaml")"

export OCC_INSTALLATION_KEY="$(yq -er \
  '.installation.key // "installation.yaml"' \
  "$WIDGET_OUTPUT_DIRECTORY/values.yaml")"

jq -n --arg key "$OCC_INSTALLATION_KEY" \
  --rawfile document "$WIDGET_OUTPUT_DIRECTORY/installation.yaml" \
  '{data:{($key):($document|@base64)}}' \
  > "$WIDGET_OUTPUT_DIRECTORY/installation-patch.json"

oc -n openclaw-system patch secret "$OCC_INSTALLATION_SECRET" \
  --type=merge --patch-file "$WIDGET_OUTPUT_DIRECTORY/installation-patch.json"

helm upgrade oce "$OCE_CHECKOUT/deploy/helm/openclaw-enterprise" \
  -n openclaw-system -f "$WIDGET_OUTPUT_DIRECTORY/values.yaml" \
  --wait --timeout 5m
```

The checksum restarts API and worker with the new startup configuration. The patch
preserves the Secret's Installation ID annotation. If Helm fails, inspect and
recover that release before continuing. The upstream
[Installation-change procedure](https://github.com/openclaw/openclaw-enterprise/blob/main/docs/guides/deploy/production-installation-changes.md)
explains rollback using the saved `before-*` files and their recalculated checksum.

## 3. Deploy the Agent and expose its renderer

OCE generates the new sandbox origin in the Agent's next revision. The Deployment
may take several minutes while retained RWO volumes move. Do not repeat deployment
while that revision is running or its outcome is uncertain.

```bash
bin/occ -o json agent deploy "$AGENT_ID" \
  > "$WIDGET_OUTPUT_DIRECTORY/deployment.json"

export REVISION_ID="$(jq -er .id "$WIDGET_OUTPUT_DIRECTORY/deployment.json")"

bin/occ agent deployment-status "$AGENT_ID" "$REVISION_ID"
```

Repeat the status read until it reports `succeeded`. Stop and inspect a `failed`
result. Then apply the prepared resources:

```bash
oc apply -k "$WIDGET_OUTPUT_DIRECTORY"

export WIDGET_HOST="$(jq -er .host "$WIDGET_OUTPUT_DIRECTORY/metadata.json")"

curl -fSs "https://$WIDGET_HOST/mcp-app-sandbox" > /dev/null

curl -sS -o /dev/null -w '%{http_code}\n' "https://$WIDGET_HOST/console/"

curl -sS -o /dev/null -w '%{http_code}\n' "https://$WIDGET_HOST/v1/chat/completions"
```

Expect shell success and `404` for both private paths. Reopen the Agent Control UI,
open the session's Dashboard panel, and verify a real widget renders and its
interaction works. A shell `200` alone is not browser proof. An older inline Canvas
preview can be unavailable after redeployment even when the saved dashboard widget
works; inspect the pinned board before recreating content.

No OpenShell egress expansion is needed for renderer hosting. Widget API fetches
run in the viewer's browser and require the widget's exact network-origin grants.
Agent-side shell/API requests remain subject to OpenShell policy separately.

## Retain the working configuration

After verification, refresh your protected cluster `values.yaml` and
`installation.yaml` from this output directory, and share the updated configuration
with the cluster maintainers. Keep backups and credentials separate. Later OCE
upgrades must preserve these preview settings. New Agents use the enabled
Compute configuration, but each needs its own prepared Route and ingress policy.
Existing Agents acquire the generated origin when explicitly redeployed.

To disable public access to one renderer, delete only its Route and custom policy
using the prepared `route.yaml` and `network-policy.yaml`. Do not delete shared
certificate resources or PVCs. This does not undo the cluster preview setting;
coordinate a separate Installation change if removing that for all Agents.
