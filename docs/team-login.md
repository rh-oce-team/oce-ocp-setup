# Give Team Claw members their own Keycloak login

Use one Codex/OpenShell Agent with separate human profiles and shared sessions.
OCC Console authentication and other Agents stay unchanged.

**Experimental prerequisite:** [OCE PR #2596: explicit Gateway identities and
native people in OCC](https://github.com/openclaw/openclaw-enterprise/pull/2596)
is open and **not yet merged** (verified October 16, 2026). Before following this
recipe, deploy an OCE controller image that includes that PR, use its matching
CLI, and select a compatible OpenClaw runtime. An unmodified OCE main checkout
is insufficient while the PR remains unmerged. Checking out the PR locally does
not update the running controller; follow the [upgrade recipe](upgrades.md) first.

Reuse the dashboard's Keycloak server with a **new realm**. Local Keycloak
accounts have their own passwords; matching an OpenShift username does not link
that account to GitHub or establish verified Git co-authorship. No OpenShift role
or OCC account is required to use this login. One Gateway remains one shared
workspace and tool-authority boundary.

The helpers create local accounts, an exact OIDC client, and protected proxy
inputs. They use the verified issuer and immutable subject for identity, an exact
membership group for proxy admission, and explicit per-person Gateway scopes.
Email is not an authorization input. Each initial password requires replacement.

## 1. Select the Agent and members

Run from this repository root. Install the existing Keycloak server first using
the [dashboard recipe](dashboard.md). Preserve that server and its database.
Set every marked value; keep all inputs outside Git.

```bash
umask 077

# REQUIRED: checkout including PR #2596, matching the installed controller and CLI.
export OCE_CHECKOUT='/absolute/path/to/openclaw-enterprise'

# REQUIRED: cluster and OCC administrator credentials.
export KUBECONFIG='/private/path/to/kubeconfig'

export OCC_URL='https://occ.apps.example.test'

export OCC_SERVICE_KEY_FILE='/private/path/to/occ-service-key.json'

# REQUIRED: existing Agent identifiers from its saved installation inputs.
export OCC_NAMESPACE='<OCC Namespace ID>'

export AGENT_ID='<Agent ID>'

export CONFIGURATION_ID='<Configuration ID>'

export GATEWAY_NAMESPACE='<tenant Kubernetes namespace>'

export GATEWAY_SERVICE='<gateway Service name>'

# REQUIRED: existing Keycloak HTTPS URL and new Agent login hostname.
export KEYCLOAK_URL='https://openshell-auth.apps.example.test'

export TEAM_CLAW_HOST='team-claw.apps.example.test'

export KEYCLOAK_NAMESPACE='openshell-auth'

export KEYCLOAK_ADMIN_SECRET='openshell-keycloak-initial-admin'

# Choose a new realm and private directory for this Agent.
export KEYCLOAK_REALM='team-claw'

export TEAM_LOGIN_DIRECTORY="$HOME/.local/state/openclaw-enterprise/team-login"

install -d -m 700 "$TEAM_LOGIN_DIRECTORY"

export TEAM_USERS_FILE="$TEAM_LOGIN_DIRECTORY/users.json"

export TEAM_ADMIN_USERS_FILE="$TEAM_LOGIN_DIRECTORY/administrators.json"

# REQUIRED: replace these lists with approved usernames. Administrators must be members.
printf '%s\n' '["alice", "bob"]' > "$TEAM_USERS_FILE"

printf '%s\n' '["alice"]' > "$TEAM_ADMIN_USERS_FILE"

oc whoami

oc whoami --show-server

scripts/build-occ
```

Cluster administrators are not automatically Gateway administrators. Review the
member list explicitly; exclude support and service accounts. A teammate need
not have logged into OpenShift to be included.

## 2. Create the new realm and prepare the proxy

The realm helper reads Keycloak's operator-created administrator Secret through
`oc`, authenticates to its HTTPS Admin API and creates the realm. It refuses an
existing realm or saved settings. On failure, keep its files and inspect the
partial setup rather than deleting users or rerunning with replacement passwords.

```bash
scripts/create-team-login-realm

bin/occ configuration get "$CONFIGURATION_ID" -o json \
  > "$TEAM_LOGIN_DIRECTORY/current-configuration.json"

export CONFIGURATION_FILE="$TEAM_LOGIN_DIRECTORY/current-configuration.json"

scripts/prepare-team-login

oc apply --dry-run=server \
  -f "$TEAM_LOGIN_DIRECTORY/proxy-manifests-private.json"

oc apply -f "$TEAM_LOGIN_DIRECTORY/proxy-manifests-private.json"

oc -n "$GATEWAY_NAMESPACE" rollout status deployment/oce-team-claw-oidc --timeout=3m
```

The proxy uses OAuth2 Proxy 7.15.5 pinned by amd64 digest, authorization-code login
with PKCE and nonce checks, and Secure/HttpOnly cookies. It overwrites the human
identity header and removes browser credentials and alternate identity headers.
The Route replaces incoming forwarded headers before OAuth2 Proxy preserves and
appends to the client address chain. Do not strip `X-Forwarded-For` between this
router and the Gateway: OpenClaw requires attributable proxy traffic. The HTTPS
Route uses the cluster certificate; the last hop from router to proxy
is HTTP, as in the existing OpenShift test recipes.

Its NetworkPolicy permits the exact Gateway, DNS, and router HTTPS endpoints.
DNS allows Service port 53 and OpenShift DNS Pod port 5353 after translation.
A Pod hostname alias directs Keycloak HTTPS to the router Service while retaining
the issuer hostname and certificate validation. Rerender after router/DNS Service
IP changes. Keep this proxy in the Gateway namespace; the generated ingress rule
selects that exact proxy. No OpenShell egress expansion is required.

## 3. Apply human scopes and preserve machine access

The prepared Configuration keeps existing model and workspace settings. Members
receive `operator.read` and `operator.write`; nominated administrators also
receive `operator.admin`. Device auto-approval excludes admin scopes. The default
member role permits collaboration in other people's sessions, subject to
OpenClaw's session-sharing rules. Gateway/model administration stays restricted.

```bash
bin/occ configuration update "$CONFIGURATION_ID" \
  --file "$TEAM_LOGIN_DIRECTORY/configuration-keycloak.json"

bin/occ agent deploy "$AGENT_ID" -o json \
  > "$TEAM_LOGIN_DIRECTORY/deployment-response.json"

export DEPLOYMENT_ID="$(jq -er .id "$TEAM_LOGIN_DIRECTORY/deployment-response.json")"

bin/occ agent deployment-status "$AGENT_ID" "$DEPLOYMENT_ID"
```

**Preserve the machine profile's role during the first rollout.** Identity scopes
alone do not override the new default member role. While the previous operator
route remains available, use its People administration to assign the
`occ-workspace-files` profile the `maintainer` role. If that route cannot connect,
use the Gateway's Pod-local operator-password RPC to call `users.list`, then
`users.setRole` with that exact machine profile ID and `role: "maintainer"`.
Do not assign this role to every person. Missing machine permissions can leave
OCC reconciliation retrying before its OpenShell Sandbox is created.

Require deployment status `succeeded`. Existing PVCs, profiles and sessions are
retained. The rollout restarts this Agent's Gateway and harness.

## 4. Complete first login and close the old route

```bash
printf 'https://%s/new\n' "$TEAM_CLAW_HOST"

# Run locally for one recipient at a time; do not put passwords in chat or shared bundles.
jq -r --arg user 'alice' '.[$user]' "$TEAM_LOGIN_DIRECTORY/initial-passwords.json"
```

Each person signs in with their username, replaces the temporary password and
opens the Control UI. Through the existing operator route, assign each nominated
administrator's new human profile the `maintainer` role. Members retain `member`.
Display names and avatars are personal; the assistant's name is separate.

Verify two human logins produce different profiles, can participate in a shared
session, and that a member cannot change Gateway settings. Confirm anonymous
requests with forged identity headers still redirect to Keycloak. Then remove
**this Agent's old shared-admin route and proxy**, which otherwise bypass the
new member permissions. Save its rendered manifest privately for operator recovery.

```bash
oc -n "$GATEWAY_NAMESPACE" delete route,service,deployment \
  oce-agent-control-ui-oauth

oc -n "$GATEWAY_NAMESPACE" get route oce-team-claw-oidc
```

Do not delete another Agent's route or change the OCC Console login. Remove the
old proxy's unused ConfigMap, cookie Secret, ServiceAccount and NetworkPolicies
only after confirming their ownership in this Agent namespace. Keep OCC's private
machine route and its required NetworkPolicies.

## Membership changes and limits

Manage local users and membership in the dedicated Keycloak realm. Adding a
member also requires adding its issuer-qualified subject to the Gateway allowlist
and identity scopes, then deploying that Configuration. Removing Gateway admission
requires updating both sides. Reusing a username after deletion creates a new
subject; never silently reuse the old grant.

Proxy logout clears its cookie; a Keycloak SSO session may still allow immediate
login. Account/group changes do not promise immediate closure of existing
WebSockets. Qualify revocation and connection expiry before treating this trial
as production access control.

Protected files include temporary passwords, client/cookie secrets, realm
bootstrap input and rendered Kubernetes Secrets. Share only configuration
summaries; deliver each initial password privately to its recipient. Keycloak
login does not resolve a model provider's Codex tool-compatibility limitations.
