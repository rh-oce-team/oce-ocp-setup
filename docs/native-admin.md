# Enable the OCC admin Control UI launcher

Enable **Gateway administration → Open admin console** on an existing Agent's
OCC page. This is full native administration as `occ-workspace-files`, authorized
by the human's OCC session and exact Agent `administer` permission. It is separate
from the [Keycloak member login](team-login.md), which keeps individual identities.
Keep durable configuration in OCE; native edits can be overwritten on redeploy.

This recipe uses the installed OCE chart and images, password-based OCC login,
working private Gateway routing, cert-manager, and the default OpenShift ingress
controller. It does not enable external OIDC login to OCC itself. Do not apply it
during another Helm/Installation change. OCC restarts; Agent redeployment briefly
interrupts connections while retaining PVCs and data. Follow the selected OCE
checkout's `docs/guides/deploy/native-admin.md` for its version-specific contract.

## 1. Select the Agent and trusted domains

Run from this repository root with the existing OCC CLI connection and explicit
cluster kubeconfig. **Fill in the three paths/IDs below.** Use a new output directory
outside both checkouts. `OCC_URL` is your existing HTTPS Console origin.

```bash
umask 077

export OCE_CHECKOUT='/absolute/path/to/installed-oce-checkout'

export KUBECONFIG='/absolute/path/to/cluster/kubeconfig'

export OCC_NAMESPACE='ns_REPLACE'

export AGENT_ID='agt_REPLACE'

export ADMIN_OUTPUT_DIRECTORY="$HOME/.local/state/oce/native-admin/first-agent"

mkdir -m 700 "$ADMIN_OUTPUT_DIRECTORY"

oc whoami

oc whoami --show-server

bin/occ -o json installation get > "$ADMIN_OUTPUT_DIRECTORY/installation-record.json"

bin/occ -o json agent get "$AGENT_ID" > "$ADMIN_OUTPUT_DIRECTORY/agent.json"

export CONFIGURATION_ID="$(jq -er .configurationId "$ADMIN_OUTPUT_DIRECTORY/agent.json")"

bin/occ -o json configuration get "$CONFIGURATION_ID" > "$ADMIN_OUTPUT_DIRECTORY/configuration.before.json"

export CONSOLE_HOST="$(OCC_URL="$OCC_URL" python3 -c 'import os,urllib.parse; print(urllib.parse.urlsplit(os.environ["OCC_URL"]).hostname)')"

export ADMIN_DOMAIN="admin.$CONSOLE_HOST"

export COOKIE_DOMAIN="$CONSOLE_HOST"

export ADMIN_HOST="$(python3 - "$ADMIN_OUTPUT_DIRECTORY/installation-record.json" "$OCC_NAMESPACE" "$AGENT_ID" "$ADMIN_DOMAIN" <<'PYHOST'
import hashlib,json,sys
installation=json.load(open(sys.argv[1]))['id']
key='\0'.join([installation,sys.argv[2],sys.argv[3]])
print('agent-'+hashlib.sha256(key.encode()).hexdigest()[:32]+'.'+sys.argv[4])
PYHOST
)"

export ADMIN_ORIGIN="https://$ADMIN_HOST"
```

Use a previously unused admin domain for the initial rollout. Every hostname under
`COOKIE_DOMAIN` must be a trusted OCC endpoint: do not use the whole cluster apps
domain as the cookie parent. Verify DNS for the exact derived hostname resolves
to your ingress. The cluster wildcard certificate does not cover nested admin
hosts; the next step issues an exact-host certificate.

Review whether other Agents share this Configuration before updating it.

## 2. Prepare HTTPS ingress

These templates create a namespaced Let's Encrypt HTTP01 Issuer and exact-host
Certificate. Public DNS and ingress on port 80 must work for ACME validation.
Certificate names enter public certificate-transparency logs. Use your approved
certificate service instead if that is unsuitable. For the OpenShift cert-manager
version tested here, HTTP01 requires the controller feature gate
`ACMEHTTP01IngressPathTypeExact=false`; check the operator-managed controller
configuration if a solver Ingress is rejected. Do not bypass TLS verification.

```bash
export TLS_NAMESPACE='openclaw-system'

export ENDPOINT_NAME="occ-admin-$(printf %s "$AGENT_ID" | shasum -a 256 | cut -c1-12)"

export ENDPOINT_HOST="$ADMIN_HOST"

export ENDPOINT_SERVICE='openclaw-enterprise-api'

export ENDPOINT_PORT=8080

for name in issuer certificate ingress solver-network-policy; do
  envsubst < "manifests/native-admin/$name.yaml" \
    > "$ADMIN_OUTPUT_DIRECTORY/$name.yaml"
done

oc apply --dry-run=server -f "$ADMIN_OUTPUT_DIRECTORY/issuer.yaml" \
  -f "$ADMIN_OUTPUT_DIRECTORY/certificate.yaml" \
  -f "$ADMIN_OUTPUT_DIRECTORY/ingress.yaml" \
  -f "$ADMIN_OUTPUT_DIRECTORY/solver-network-policy.yaml"

oc apply -f "$ADMIN_OUTPUT_DIRECTORY/solver-network-policy.yaml" \
  -f "$ADMIN_OUTPUT_DIRECTORY/issuer.yaml" \
  -f "$ADMIN_OUTPUT_DIRECTORY/certificate.yaml" \
  -f "$ADMIN_OUTPUT_DIRECTORY/ingress.yaml"

oc -n "$TLS_NAMESPACE" wait "certificate/$ENDPOINT_NAME" \
  --for=condition=Ready --timeout=5m
```

Keep the solver policy for renewals. The admin Ingress targets the OCC API on
8080, never the Gateway or renderer. Preserve the existing router-to-API ingress
policy from the OCC installation.

## 3. Enable the OCC feature

For the first Agent, save and edit the live values. For additional Agents, retain
these installed global settings; repeat the per-Agent certificate, origin and
deployment steps.

```bash
helm -n openclaw-system get values oce -o yaml \
  > "$ADMIN_OUTPUT_DIRECTORY/values.before.yaml"

cp "$ADMIN_OUTPUT_DIRECTORY/values.before.yaml" "$ADMIN_OUTPUT_DIRECTORY/values.yaml"

yq -i '.agentNativeAdmin.enabled = true |
  .agentNativeAdmin.domain = strenv(ADMIN_DOMAIN) |
  .agentNativeAdmin.sharedCookieDomain = strenv(COOKIE_DOMAIN)' \
  "$ADMIN_OUTPUT_DIRECTORY/values.yaml"
```

**If widget previews are enabled:** their domain must be outside the OCC cookie
parent, with neither domain containing the other. An existing `apps.cluster…`
preview domain overlaps `occ.apps.cluster…` and must move before the Helm upgrade.
Follow [move previews outside the cookie domain](native-admin-previews.md) now,
then return with the candidate values, Installation and certificates prepared.

```bash
helm template oce "$OCE_CHECKOUT/deploy/helm/openclaw-enterprise" \
  -n openclaw-system -f "$ADMIN_OUTPUT_DIRECTORY/values.yaml" \
  > "$ADMIN_OUTPUT_DIRECTORY/helm-rendered.yaml"

helm upgrade oce "$OCE_CHECKOUT/deploy/helm/openclaw-enterprise" \
  -n openclaw-system -f "$ADMIN_OUTPUT_DIRECTORY/values.yaml" \
  --wait --timeout 5m
```

Expect release `deployed` and healthy API/worker Pods. Preserve image selections.
Resolve any failed Helm release before continuing.

## 4. Merge the Agent origin and deploy

The following merge preserves existing identities, origins, model settings and
Secret references. Omitted Secret bindings stay unchanged. Review explicit UI or
device-approval opt-outs with the Configuration owner before enabling them.

```bash
jq --arg origin "$ADMIN_ORIGIN" '
  {expectedGeneration:.generation, values:.values} |
  .values.gateway.auth.mode = "trusted-proxy" |
  .values.gateway.auth.trustedProxy.userHeader = "x-occ-identity" |
  .values.gateway.auth.trustedProxy.allowUsers |= ((. // []) + ["occ-workspace-files"] | unique) |
  .values.gateway.auth.identityScopes["occ-workspace-files"] |= ((. // []) + ["operator.admin"] | unique) |
  .values.gateway.auth.trustedProxy.deviceAutoApprove = {enabled:true,scopes:["operator.read","operator.write"]} |
  .values.gateway.controlUi.enabled = true |
  .values.gateway.controlUi.allowedOrigins |= ((. // []) + [$origin] | unique)
' "$ADMIN_OUTPUT_DIRECTORY/configuration.before.json" \
  > "$ADMIN_OUTPUT_DIRECTORY/configuration-update.json"

bin/occ configuration update "$CONFIGURATION_ID" \
  --file "$ADMIN_OUTPUT_DIRECTORY/configuration-update.json"

bin/occ -o json agent deploy "$AGENT_ID" \
  > "$ADMIN_OUTPUT_DIRECTORY/deployment.json"

export REVISION_ID="$(jq -er .id "$ADMIN_OUTPUT_DIRECTORY/deployment.json")"

bin/occ agent deployment-status "$AGENT_ID" "$REVISION_ID"
```

Repeat only the status read until `succeeded`; investigate `failed` or prolonged
pending work before admitting another revision. Preserve `gateway.publicOrigin`
when keeping a separate human-facing Keycloak route.

## 5. Verify and retain

Sign out of OCC and sign in again to replace the old host-only session cookie.
On the Agent page, select **Refresh access**, then **Open admin console**. Expect
its exact admin hostname and a connected Control UI. The displayed profile may
have a customized name; the proxy authenticates as `occ-workspace-files`.
Verify the existing Keycloak route still connects with your individual profile.
If previews moved, open a saved dashboard and interact with a widget before
retiring its old Route. Anonymous requests to the admin host must be denied or
sent to OCC login; they must never open an authenticated Gateway session.

After verification, copy the candidate `values.yaml` and any changed
`installation.yaml` into your protected cluster baseline used for upgrades.
Retain the before files and rendered ingress resources outside Git.
To withdraw one launcher endpoint, delete its prepared `ingress.yaml` only;
keep Keycloak routes and PVCs. To undo the global feature or preview domain,
restore both saved Helm values and startup Installation together through the
selected OCE checkout's `docs/guides/deploy/production-installation-changes.md`.
