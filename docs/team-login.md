# Give Team Claw members their own Keycloak login

Use one Codex/OpenShell Agent with separate human profiles and shared sessions.
OCC Console authentication and other Agents stay unchanged.

**Experimental prerequisite:** [OCE PR #3049: explicit human Gateway identities](https://github.com/openclaw/openclaw-enterprise/pull/3049)
is open and **not yet merged** (verified October 21, 2026). It contains the
Gateway identity admission split from #2596. The separate OCC People-listing PR
is not required; OCC Console authentication stays unchanged. Before following
this recipe, deploy an OCE controller image that includes #3049, use its matching
CLI, and select a compatible OpenClaw runtime. An unmodified OCE main checkout
is insufficient while the PR remains unmerged. Checking out the PR locally does
not update the running controller; follow the [upgrade recipe](upgrades.md) first.

Reuse the existing Keycloak server with either a **new realm** or an existing
application realm. The existing-realm path below preserves current passwords
and creates only missing accounts. Local Keycloak accounts have their own passwords; matching an OpenShift username does not link
that account to GitHub or establish verified Git co-authorship. No OpenShift role
or OCC account is required to use this login. One Gateway remains one shared
workspace and tool-authority boundary.

The helpers create local accounts, an exact OIDC client, and protected proxy
inputs. They use the verified issuer and immutable subject for identity, an exact
membership group for proxy admission, and explicit per-person Gateway scopes.
Email is not an authorization input. Each initial password requires replacement.

## 1. Select the Agent and members

Run from this repository root. For a new Agent, follow sections 1–6 of the
[additional Agent recipe](agents.md), stopping after deployment succeeds and
before its OpenShift OAuth Route step. Continue here for Keycloak login instead;
skip the per-owner OCC handoff unless members also need OCC access.

Install Keycloak first using the [dashboard recipe](dashboard.md), or reuse an
existing server. Preserve that server and its database. Set every marked value;
keep all inputs outside Git.

```bash
umask 077

# REQUIRED: checkout including PR #3049, matching the installed controller and CLI.
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

## 2. Prepare realm accounts and the proxy

**Choose one account path.** For a new realm, the helper generates a separate
random temporary password for every member and refuses an existing realm or
saved login settings:

```bash
scripts/create-team-login-realm
```

**To reuse accounts in an existing application realm, run this block instead.**
It creates only missing enabled users; existing accounts and passwords stay
unchanged. Keep the temporary password file outside Git. All newly created
accounts must change that password at first login.

```bash
# REQUIRED: existing realm, and unused client/group names for this Agent.
export KEYCLOAK_REALM='existing-application-realm'

export KEYCLOAK_CLIENT_ID='oce-team-claw-browser'

export KEYCLOAK_GROUP_NAME='oce-team-claw-members'

export TEAM_INITIAL_PASSWORD_FILE="$TEAM_LOGIN_DIRECTORY/new-user-temporary-password"

read -r -s -p 'Temporary password for missing users: ' TEAM_INITIAL_PASSWORD

printf '\n'

printf '%s' "$TEAM_INITIAL_PASSWORD" > "$TEAM_INITIAL_PASSWORD_FILE"

unset TEAM_INITIAL_PASSWORD

scripts/prepare-team-login-users

export TEAM_USERS_FILE="$TEAM_LOGIN_DIRECTORY/normalized-users.json"

scripts/add-agent-keycloak-client
```

Use the normalized realm usernames in `TEAM_ADMIN_USERS_FILE`, too. The helper
refuses disabled or ambiguous accounts and never resets an existing password.
The client supplies its own username and membership claims and requests only
`openid`; the existing realm need not have a standard `profile` scope. Keep its
profile requirements and session policy; users may need to complete their
profile at login. The eight-hour session defaults below
apply to a new realm created by this recipe, not to an existing realm.

Both paths use the operator-created Keycloak administrator Secret through `oc`.
After a partial failure, inspect the saved inputs and realm before continuing;
do not recreate clients or accounts to replace credentials.

**Continue here after either account path:**

```bash
bin/occ configuration get "$CONFIGURATION_ID" -o json \
  > "$TEAM_LOGIN_DIRECTORY/current-configuration.json"

export CONFIGURATION_FILE="$TEAM_LOGIN_DIRECTORY/current-configuration.json"

scripts/prepare-team-login

oc apply --dry-run=server \
  -f "$TEAM_LOGIN_DIRECTORY/proxy-manifests-private.json"
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

### Session duration

New realms use an eight-hour SSO idle limit and eight-hour maximum session,
matching the proxy cookie. Access tokens last five minutes and the proxy attempts
refresh on HTTP requests after one minute. An open Control UI WebSocket alone
does not refresh the Keycloak session. Users must sign in again after the maximum
session expires; existing expired sessions are not revived by increasing limits.

For an older realm created by this recipe with a 30-minute idle limit, keep the
section 1 environment and run this once. It changes only that realm's idle limit;
review its eight-hour maximum session first. No passwords are reset.

```bash
python3 - <<'PYTHON'
import runpy
c = runpy.run_path("scripts/create-team-login-realm", run_name="session_update")
url = c["admin"] + "/" + c["realm"]
live = c["request"](url, token=c["token"])
assert live["ssoSessionMaxLifespan"] == 28800
c["request"](url, {"ssoSessionIdleTimeout": 28800}, "PUT", c["token"])
print("Realm idle limit updated to eight hours; existing passwords preserved.")
PYTHON
```

For another Agent using these same accounts, follow
[reuse existing Keycloak accounts](agent-keycloak-login.md).

## 3. Apply human scopes and preserve machine access

The prepared Configuration keeps existing model and workspace settings. Members
receive `operator.read` and `operator.write`; nominated administrators also
receive `operator.admin`. Preparation preserves existing device-approval scopes,
including admin approval required by the OCC launcher. Each person's explicit
identity scopes and role still restrict access. The default member role permits
collaboration in other people's sessions, subject to OpenClaw's session-sharing
rules. Gateway/model administration stays restricted.

**Define roles before assigning them.** First deploy the role definitions with
`maintainer` as the temporary default. This preserves the existing operator
access while the machine profile gets its explicit role. Keep the new public
proxy unapplied until the final member-default deployment succeeds.

```bash
jq --slurpfile login "$TEAM_LOGIN_DIRECTORY/configuration-keycloak.json" \
  '{values:.values} | .values.gateway.roles=$login[0].values.gateway.roles | .values.gateway.roles.default="maintainer"' \
  "$TEAM_LOGIN_DIRECTORY/current-configuration.json" \
  > "$TEAM_LOGIN_DIRECTORY/configuration-role-definitions.json"

bin/occ configuration update "$CONFIGURATION_ID" \
  --file "$TEAM_LOGIN_DIRECTORY/configuration-role-definitions.json"

bin/occ agent deploy "$AGENT_ID" -o json \
  > "$TEAM_LOGIN_DIRECTORY/role-definitions-deployment.json"

export ROLE_DEPLOYMENT_ID="$(jq -er .id "$TEAM_LOGIN_DIRECTORY/role-definitions-deployment.json")"

bin/occ agent deployment-status "$AGENT_ID" "$ROLE_DEPLOYMENT_ID"
```

Wait for `succeeded`. Then assign only the exact `occ-workspace-files` machine
profile the `maintainer` role through the Gateway's Pod-local operator RPC:

```bash
export GATEWAY_POD="$(oc -n "$GATEWAY_NAMESPACE" get pods \
  -l "openclaw.dev/agent=$AGENT_ID,openclaw.dev/workload-role=gateway" \
  -o jsonpath='{.items[0].metadata.name}')"

oc -n "$GATEWAY_NAMESPACE" exec "$GATEWAY_POD" -c gateway -- \
  node /app/openclaw.mjs gateway call users.list --json \
  > "$TEAM_LOGIN_DIRECTORY/profiles-before.json"

export MACHINE_PROFILE_ID="$(jq -er '.profiles | map(select(.displayName=="occ-workspace-files")) | if length==1 then .[0].id else error("Expected one machine profile") end' \
  "$TEAM_LOGIN_DIRECTORY/profiles-before.json")"

oc -n "$GATEWAY_NAMESPACE" exec "$GATEWAY_POD" -c gateway -- \
  node /app/openclaw.mjs gateway call users.setRole --json \
  --params "$(jq -nc --arg id "$MACHINE_PROFILE_ID" '{profileId:$id,role:"maintainer"}')"
```

Apply the prepared human scopes and `member` default:

```bash
bin/occ configuration update "$CONFIGURATION_ID" \
  --file "$TEAM_LOGIN_DIRECTORY/configuration-keycloak.json"

bin/occ agent deploy "$AGENT_ID" -o json \
  > "$TEAM_LOGIN_DIRECTORY/deployment-response.json"

export DEPLOYMENT_ID="$(jq -er .id "$TEAM_LOGIN_DIRECTORY/deployment-response.json")"

bin/occ agent deployment-status "$AGENT_ID" "$DEPLOYMENT_ID"
```

Require deployment status `succeeded`. Existing PVCs, profiles and sessions are
retained. The rollout restarts this Agent's Gateway and harness.

## 4. Complete first login and close the old route

After the final deployment succeeds, open the Keycloak proxy:

```bash
oc apply -f "$TEAM_LOGIN_DIRECTORY/proxy-manifests-private.json"

oc -n "$GATEWAY_NAMESPACE" rollout status deployment/oce-team-claw-oidc --timeout=3m

printf 'https://%s/new\n' "$TEAM_CLAW_HOST"

# Run locally for one recipient at a time; do not put passwords in chat or shared bundles.
# New-realm path only: read the generated password for one recipient.
jq -r --arg user 'alice' '.[$user]' "$TEAM_LOGIN_DIRECTORY/initial-passwords.json"
```

For existing-realm accounts, keep the current password. Deliver the protected
temporary password only to newly created users. Each person signs in with their
realm username, completes any required password/profile changes, and opens the
Control UI. Through the existing operator route, assign each nominated
administrator's new human profile the `maintainer` role. Members retain `member`.
Display names and avatars are personal; the assistant's name is separate.

Verify two human logins produce different profiles, can participate in a shared
session, and that a member cannot change Gateway settings. Confirm anonymous
requests with forged identity headers still redirect to Keycloak. Then remove
**this Agent's old shared-admin route and proxy**, which otherwise bypass the
new member permissions. Save its rendered manifest privately for operator recovery.

Skip this deletion for a new Agent that never had the old OAuth proxy.

```bash
oc -n "$GATEWAY_NAMESPACE" delete route,service,deployment \
  oce-agent-control-ui-oauth

oc -n "$GATEWAY_NAMESPACE" get route oce-team-claw-oidc
```

Do not delete another Agent's route or change the OCC Console login. Remove the
old proxy's unused ConfigMap, cookie Secret, ServiceAccount and NetworkPolicies
only after confirming their ownership in this Agent namespace. Keep OCC's private
machine route and its required NetworkPolicies.

Complete the [OCC admin launcher procedure](native-admin.md) for this Agent
as well. Keep this Keycloak Route and its member roles. The separate OCC launcher
uses the machine identity for authorized OCC administrators; it does not replace
individual teammate login. Repeat Keycloak preparation only after fetching the
latest Configuration so existing admin approval and origins stay intact.

## Membership changes and limits

Manage local users and this Agent's membership group in the selected Keycloak realm. Adding a
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
