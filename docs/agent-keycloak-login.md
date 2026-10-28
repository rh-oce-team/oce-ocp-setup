# Give a personal Agent an existing Keycloak login

Reuse a person's existing Keycloak account without resetting its password.
This creates a separate OIDC client and membership group for one Agent. The
[team-login prerequisites](team-login.md) still apply, including the experimental
OCE controller and compatible runtime. OCC authentication stays unchanged.

## 1. Select the existing Agent and realm

Use the environment from the team-login recipe for cluster credentials, OCC,
Keycloak and the selected Agent. Set `OCC_NAMESPACE`, `AGENT_ID`,
`CONFIGURATION_ID`, `GATEWAY_NAMESPACE` and `GATEWAY_SERVICE` to this Agent's
values. Choose an unused login hostname and a new protected directory. This
recipe creates an additional route; retain the old one until login succeeds.

```bash
# REQUIRED: existing realm, new client/group, new hostname and private directory.
export KEYCLOAK_REALM='team-claw'

export KEYCLOAK_CLIENT_ID='personal-agent-browser'

export KEYCLOAK_GROUP_NAME='personal-agent-members'

export TEAM_CLAW_HOST='personal-agent.apps.REPLACE_ME'

export TEAM_LOGIN_DIRECTORY="$HOME/.local/state/oce-agents/REPLACE_ME/keycloak"

install -d -m 700 "$TEAM_LOGIN_DIRECTORY"

export TEAM_USERS_FILE="$TEAM_LOGIN_DIRECTORY/users.json"

export TEAM_ADMIN_USERS_FILE="$TEAM_LOGIN_DIRECTORY/administrators.json"

# REQUIRED: the same existing realm username in both lists for an owner-only Agent.
printf '%s\n' '["REPLACE_ME"]' > "$TEAM_USERS_FILE"

printf '%s\n' '["REPLACE_ME"]' > "$TEAM_ADMIN_USERS_FILE"

scripts/add-agent-keycloak-client

bin/occ configuration get "$CONFIGURATION_ID" -o json \
  > "$TEAM_LOGIN_DIRECTORY/current-configuration.json"

export CONFIGURATION_FILE="$TEAM_LOGIN_DIRECTORY/current-configuration.json"

scripts/prepare-team-login

scripts/configure-team-login-scopes
```

The helper refuses existing clients, groups or saved settings. After a partial
failure, inspect the saved inputs and realm rather than recreating accounts or
credentials. Membership in this Agent's group is separate from team membership;
Gateway admission also checks the exact issuer-qualified subject.

## 2. Deploy an owner-only Gateway

Only the nominated owner and OCC machine identity are admitted. Both need full
Gateway access through their explicit identity grants. Omit `gateway.roles` for
the current trusted-team Codex workaround described in
[the team-login guide](team-login.md#3-apply-human-scopes-and-preserve-machine-access).
Personal profiles, names, avatars and session ownership remain separate.

```bash
jq 'del(.values.gateway.roles)' \
  "$TEAM_LOGIN_DIRECTORY/configuration-keycloak.json" \
  > "$TEAM_LOGIN_DIRECTORY/configuration-owner.json"

oc apply --dry-run=server -f "$TEAM_LOGIN_DIRECTORY/proxy-manifests-private.json"

oc apply -f "$TEAM_LOGIN_DIRECTORY/proxy-manifests-private.json"

oc -n "$GATEWAY_NAMESPACE" rollout status deployment/oce-team-claw-oidc --timeout=3m

bin/occ configuration update "$CONFIGURATION_ID" \
  --file "$TEAM_LOGIN_DIRECTORY/configuration-owner.json"

export DEPLOYMENT_ID="$(bin/occ agent deploy "$AGENT_ID" -o json | jq -r .id)"

bin/occ agent deployment-status "$AGENT_ID" "$DEPLOYMENT_ID"

printf 'https://%s/new\n' "$TEAM_CLAW_HOST"
```

Require deployment status `succeeded`. Sign in with the existing Keycloak username
and password. A first human login creates a distinct Gateway profile; it does not
rename or merge the existing `occ-workspace-files` profile or transfer ownership
of its old sessions. Model credentials, Agent identity and workspace stay intact.

Confirm the new profile and expected session access, then close this Agent's old
shared-admin route/proxy using the team-login guide's ownership checks. For a
hostname-preserving migration, stage the proxy and Gateway first, then update the
existing Route's Service target to the new proxy; do not create two Routes with
the same hostname.

Regenerate the [private Claw links directory](claw-links.md) and share it with
the owner so they can find their human login again. Keep actual names and URLs
out of this public repository.
