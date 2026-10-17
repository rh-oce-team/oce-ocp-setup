# Move widget previews outside the OCC cookie domain

Use this step during [OCC admin launcher setup](native-admin.md) if the installed
preview domain overlaps its shared cookie domain. It changes the Installation's
preview domain, so every Agent with public preview access needs a replacement
endpoint and redeployment. Keep the old Routes until each replacement works.
Do not expose a renderer for an Agent that did not already have public previews.

The example uses `previews.<cluster-apps-domain>`, separate from
`occ.<cluster-apps-domain>`. Verify DNS resolves the nested hostname to the router.
The public exact-host certificate and private Envoy wildcard certificate serve
different listeners; retain both. The renderer serves public shell/assets only.
Private widget documents arrive through the authenticated Control UI.

## 1. Prepare the global preview change once

Keep the variables and candidate values from the launcher guide. This procedure
requires the existing namespaced private CA Issuer and default OpenShift router.

```bash
export PREVIEW_DOMAIN="previews.$(oc get ingress.config.openshift.io cluster -o jsonpath='{.spec.domain}')"

export PREVIEW_TLS_NAME="oce-preview-$(printf %s "$PREVIEW_DOMAIN" | shasum -a 256 | cut -c1-12)"

export OCC_INSTALLATION_SECRET="$(yq -er '.installation.secretName // "occ-installation-startup"' "$ADMIN_OUTPUT_DIRECTORY/values.yaml")"

export OCC_INSTALLATION_KEY="$(yq -er '.installation.key // "installation.yaml"' "$ADMIN_OUTPUT_DIRECTORY/values.yaml")"

oc -n openclaw-system get secret "$OCC_INSTALLATION_SECRET" -o json \
  | jq -er --arg key "$OCC_INSTALLATION_KEY" '.data[$key]' \
  | base64 --decode > "$ADMIN_OUTPUT_DIRECTORY/installation.before.yaml"

cp "$ADMIN_OUTPUT_DIRECTORY/installation.before.yaml" "$ADMIN_OUTPUT_DIRECTORY/installation.yaml"

export PRIVATE_GATEWAY="$(yq -er '.gatewayRouting.gatewayName // "oce-agent-gateways"' "$ADMIN_OUTPUT_DIRECTORY/values.yaml")"

export GATEWAY_LEAF_SECRET="$(oc -n openclaw-system get gateway "$PRIVATE_GATEWAY" -o json | jq -er '.spec.listeners[] | select(.name=="https") | .tls.certificateRefs[0].name')"

# This setup's Gateway Certificate and TLS Secret have matching names.
oc -n openclaw-system get certificate "$GATEWAY_LEAF_SECRET" -o json \
  > "$ADMIN_OUTPUT_DIRECTORY/gateway-certificate.json"

jq -e '(.spec.issuerRef.kind // "Issuer") == "Issuer"' \
  "$ADMIN_OUTPUT_DIRECTORY/gateway-certificate.json" > /dev/null

jq --arg name "$PREVIEW_TLS_NAME" --arg domain "$PREVIEW_DOMAIN" '
  {apiVersion:"cert-manager.io/v1",kind:"Certificate",
   metadata:{name:$name,namespace:"openclaw-system"},
   spec:{secretName:$name,dnsNames:[("*."+$domain)],issuerRef:.spec.issuerRef}}
' "$ADMIN_OUTPUT_DIRECTORY/gateway-certificate.json" \
  > "$ADMIN_OUTPUT_DIRECTORY/private-preview-certificate.json"

yq -i '.gatewayRouting.sandbox.domain = strenv(PREVIEW_DOMAIN) |
  .gatewayRouting.sandbox.tlsSecretName = strenv(PREVIEW_TLS_NAME)' \
  "$ADMIN_OUTPUT_DIRECTORY/values.yaml"

yq -i '.drivers.compute.configuration.gatewayRouting.sandbox.domain = strenv(PREVIEW_DOMAIN)' \
  "$ADMIN_OUTPUT_DIRECTORY/installation.yaml"

export INSTALLATION_CHECKSUM="$(shasum -a 256 "$ADMIN_OUTPUT_DIRECTORY/installation.yaml" | cut -d ' ' -f1)"

yq -i '.controlPlane.installationChecksum = strenv(INSTALLATION_CHECKSUM)' \
  "$ADMIN_OUTPUT_DIRECTORY/values.yaml"

scripts/validate-installation "$ADMIN_OUTPUT_DIRECTORY/installation.yaml"

oc apply -f "$ADMIN_OUTPUT_DIRECTORY/private-preview-certificate.json"

oc -n openclaw-system wait "certificate/$PREVIEW_TLS_NAME" \
  --for=condition=Ready --timeout=120s
```

Review the before/candidate diff privately. Preserve the existing sandbox listener,
port, ingress peers and all other Installation settings. The new domain must be
disjoint from `COOKIE_DOMAIN`; Helm rendering in the launcher guide enforces this.

## 2. Prepare each existing public renderer endpoint

**Set `GATEWAY_NAMESPACE` and `GATEWAY_SERVICE` to this Agent's existing Gateway.**
Repeat this section for every Agent whose renderer was public, using a separate
private output directory per Agent. Keep the original namespace/Agent ID paired
when deriving its hostname. These templates target the renderer on 8081, not the
administrative Gateway listener on 8080.

```bash
export GATEWAY_NAMESPACE='oce_REPLACE'

export GATEWAY_SERVICE='gateway-REPLACE'

export SANDBOX_PORT=8081

export WIDGET_HOST="$(python3 - "$OCC_NAMESPACE" "$AGENT_ID" "$PREVIEW_DOMAIN" <<'PYHOST'
import hashlib,sys
print('agent-'+hashlib.sha256((sys.argv[1]+'/'+sys.argv[2]).encode()).hexdigest()[:32]+'.'+sys.argv[3])
PYHOST
)"

export TLS_NAMESPACE="$GATEWAY_NAMESPACE"

export ENDPOINT_NAME="oce-preview-$(printf %s "$AGENT_ID" | shasum -a 256 | cut -c1-12)"

export ENDPOINT_HOST="$WIDGET_HOST"

export ENDPOINT_SERVICE="$GATEWAY_SERVICE"

export ENDPOINT_PORT="$SANDBOX_PORT"

mkdir -m 700 "$ADMIN_OUTPUT_DIRECTORY/preview"

for name in issuer certificate ingress solver-network-policy; do
  envsubst < "manifests/native-admin/$name.yaml" \
    > "$ADMIN_OUTPUT_DIRECTORY/preview/$name.yaml"
done

export WIDGET_ROUTE="$ENDPOINT_NAME"

envsubst < manifests/widgets/network-policy.yaml \
  > "$ADMIN_OUTPUT_DIRECTORY/preview/network-policy.yaml"

oc apply --dry-run=server -f "$ADMIN_OUTPUT_DIRECTORY/preview"

oc apply -f "$ADMIN_OUTPUT_DIRECTORY/preview"

oc -n "$GATEWAY_NAMESPACE" wait "certificate/$ENDPOINT_NAME" \
  --for=condition=Ready --timeout=5m
```

Retain the solver policy for certificate renewal. OpenShift ingress-to-route uses
the TLS Secret and follows certificate rotation. No OpenShell policy expansion is
needed; these are router-to-renderer ingress rules.

## 3. Update the startup document and return to the launcher guide

This patches only the existing startup Secret's selected data key, preserving its
Installation ID annotation. Finish the Helm upgrade from the launcher guide
immediately afterward; API and worker must read the matching candidate document.

```bash
jq -n --arg key "$OCC_INSTALLATION_KEY" \
  --rawfile document "$ADMIN_OUTPUT_DIRECTORY/installation.yaml" \
  '{data:{($key):($document|@base64)}}' \
  > "$ADMIN_OUTPUT_DIRECTORY/installation-patch.json"

oc -n openclaw-system patch secret "$OCC_INSTALLATION_SECRET" \
  --type=merge --patch-file "$ADMIN_OUTPUT_DIRECTORY/installation-patch.json"
```

Return to **Enable the OCC feature** in the launcher guide. After the Helm upgrade,
redeploy each affected Agent once and wait for `succeeded`. Then verify:

```bash
curl -fSs "https://$WIDGET_HOST/mcp-app-sandbox" > /dev/null

curl -sS -o /dev/null -w '%{http_code}\n' "https://$WIDGET_HOST/console/"

curl -sS -o /dev/null -w '%{http_code}\n' "https://$WIDGET_HOST/v1/chat/completions"
```

Expect shell success and `404` for both private paths. Open an existing saved
dashboard from the human Control UI and verify its widget renders and responds to
an interaction. A shell `200` alone is insufficient. Save the original Route
manifest privately, then delete only that superseded Route after browser proof.
Keep its TLS resources for rollback until the rollout is accepted. Retain the new
values and Installation as the protected cluster baseline for future upgrades.
