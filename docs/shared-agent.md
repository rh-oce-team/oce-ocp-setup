# Create a shared Codex/OpenShell Agent with a custom model

Create one Agent that teammates access directly through OpenShift OAuth. They
can open and share sessions without OCC accounts. OpenClaw receives one shared
`occ-workspace-files` identity with Gateway administrator access. This recipe
does not provide private sessions, separate human profiles or verified Git
co-author attribution.

Run from this repository root after the existing OCE, OpenShell and private
Gateway routing installation is working. Use the installed OCE version's CLI.
An OCC administrator with OpenShift operator permissions creates the Agent.
CLI operations are attributed to the ServicePrincipal represented by the OCC
key; using the bootstrap key records `bootstrap-admin`, even when `oc whoami`
shows a personal OpenShift username.

The model endpoint must accept Bearer authentication and streaming OpenAI
Responses requests, including the tools advertised by the installed Codex version.
Chat Completions compatibility or a successful tool-free Responses request alone
is insufficient. One
Agent-owned provider credential serves all sessions and shares its billing and
limits. This recipe keeps Codex's existing `read-only` tool policy.

## 1. Choose the Agent and endpoint

Replace every marked value. Keep credentials outside the repository.

```bash
umask 077

# REQUIRED: checkout matching the installed OCE controller.
export OCE_CHECKOUT='/absolute/path/to/openclaw-enterprise'

# REQUIRED: private kubeconfig and authorized OCC service-key response file.
export KUBECONFIG='/private/path/to/kubeconfig'

export OCC_SERVICE_KEY_FILE='/private/path/to/occ-service-key.json'

export OCC_URL='https://occ.apps.example.test'

# REQUIRED: new DNS-style name; do not reuse an existing Agent directory.
export AGENT_NAME='team-claw'

# REQUIRED: Responses-compatible endpoint and its exact native model ID.
export OPENAI_BASE_URL='https://models.example.test/v1'

export MODEL_ID='vendor/model-name'

# REQUIRED: protected file containing only the model key, not KEY=value.
export MODEL_KEY_FILE='/private/path/to/model-key'

export AGENT_INPUT_DIRECTORY="$HOME/.local/state/openclaw-enterprise/agents/$AGENT_NAME"

export OCC_INPUT_DIRECTORY="$AGENT_INPUT_DIRECTORY"

export CONTROL_UI_ACCESS_MODE='cluster-admin'

scripts/build-occ

oc whoami

oc whoami --show-server

test -s "$MODEL_KEY_FILE"

test -s "$OCC_SERVICE_KEY_FILE"

test ! -e "$AGENT_INPUT_DIRECTORY/namespace-response.json"

install -d -m 700 "$AGENT_INPUT_DIRECTORY"
```

Use `cluster-admin` only when every intended member already has that cluster
role. For non-admin members choose `scoped`, then grant each member the exact
Service permission during the Control UI procedure. Neither choice grants OCC
Console access.

## 2. Create the Namespace and grant tenant access

Run this block without pausing between creation and grants. Namespace
provisioning has a convergence deadline.

```bash
bin/occ namespace create "$AGENT_NAME" -o json \
  > "$AGENT_INPUT_DIRECTORY/namespace-response.json"

export OCC_NAMESPACE="$(jq -er .id "$AGENT_INPUT_DIRECTORY/namespace-response.json")"

scripts/grant-tenant-access

export TENANT_NAMESPACE="$(oc get namespaces \
  -l "openclaw.dev/namespace=$OCC_NAMESPACE" -o jsonpath='{.items[0].metadata.name}')"

export GATEWAY_NAMESPACE="$TENANT_NAMESPACE"

bin/occ namespace get "$OCC_NAMESPACE"
```

Require Namespace status `ready` before continuing. Keep all generated inputs
if a command fails; inspect the existing resources instead of rerunning creation.

## 3. Register the model key and prepare the custom Configuration

The temporary request contains the real key. Delete it even if Secret creation
fails. The saved Secret response contains a reference, not the key.

```bash
set -o pipefail

python3 - <<'PY'
import json, os, pathlib
key = pathlib.Path(os.environ['MODEL_KEY_FILE']).read_text().rstrip('\r\n')
if not key or '\n' in key or '\r' in key:
    raise SystemExit('Expected one nonempty API key')
path = pathlib.Path(os.environ['OCC_INPUT_DIRECTORY']) / 'model-secret.json'
with path.open('x') as stream:
    json.dump({'name': os.environ['AGENT_NAME'] + '-model-key', 'value': key}, stream)
PY

if bin/occ secret create --file "$AGENT_INPUT_DIRECTORY/model-secret.json" -o json \
  > "$AGENT_INPUT_DIRECTORY/model-secret-response.json"; then
  rm -f "$AGENT_INPUT_DIRECTORY/model-secret.json"
else
  rm -f "$AGENT_INPUT_DIRECTORY/model-secret.json"
  echo 'Secret creation failed; stop here.' >&2
  false
fi

export MODEL_SECRET_RESPONSE_FILE="$AGENT_INPUT_DIRECTORY/model-secret-response.json"

scripts/prepare-shared-agent

bin/occ credential-source create --file "$AGENT_INPUT_DIRECTORY/credential-source.json" \
  -o json > "$AGENT_INPUT_DIRECTORY/credential-source-response.json"

export SOURCE_ID="$(jq -er .id "$AGENT_INPUT_DIRECTORY/credential-source-response.json")"

bin/occ credential-source get "$SOURCE_ID"

bin/occ configuration create --file "$AGENT_INPUT_DIRECTORY/configuration.json" \
  -o json > "$AGENT_INPUT_DIRECTORY/configuration-response.json"

export CONFIGURATION_ID="$(jq -er .id "$AGENT_INPUT_DIRECTORY/configuration-response.json")"
```

Require source state and Gateway status `ready`. The helper prepares files only.
It reuses the existing Codex Configuration and the custom-source manifest, sets
the selected model and assistant display name from `AGENT_NAME`, and keeps the
Gateway model URL at fail-closed
`http://127.0.0.1:9`. Codex receives the endpoint through the credential source.
The assistant name is managed by OCE and survives redeployment. The internal
OpenClaw agent ID remains `main`; the shared signed-in user identity is unchanged.
No provider key goes into the Configuration. To change endpoint later, register
a new immutable source and update the Agent's binding.

## 4. Create the Agent

```bash
jq -n --arg name "$AGENT_NAME" --arg configuration "$CONFIGURATION_ID" \
  --arg source "$SOURCE_ID" '{
    name:$name, configurationId:$configuration, executionMode:"dedicated",
    harnessAuth:{method:"credential_source", sourceId:$source},
    credentialSources:[{sourceId:$source}]
  }' > "$AGENT_INPUT_DIRECTORY/agent.json"

bin/occ agent create --file "$AGENT_INPUT_DIRECTORY/agent.json" -o json \
  > "$AGENT_INPUT_DIRECTORY/agent-response.json"

export AGENT_ID="$(jq -er .id "$AGENT_INPUT_DIRECTORY/agent-response.json")"

export AGENT_PRINCIPAL="$(jq -er .servicePrincipalId "$AGENT_INPUT_DIRECTORY/agent-response.json")"

export AGENT_SA="agent-$(printf '%s' "$AGENT_ID" | openssl dgst -sha256 | awk '{print substr($NF,1,12)}')"

export GATEWAY_SA="gateway-${AGENT_SA#agent-}"
```

## 5. Grant runtime access, deploy and open the Control UI

Complete **sections 5 and 6 only** of the
[additional-Agent recipe](agents.md#5-grant-the-exact-credential-source-and-runtime-access),
retaining the variables above. Those steps grant the exact model-source, image
pull and Gateway SCC permissions, then deploy the Agent. Section 6 directs you
to the required OAuth Route procedure. Finish that procedure and require the
final Agent revision to report `succeeded`.

Skip that recipe's section 7 when teammates need only the Control UI. They do
not need OCC accounts or Agent owner grants. Share the `/new` URL from the OAuth
Route procedure and verify each teammate's OpenShift login. Each authorized
member receives shared Gateway administration, not a restricted member role.

Complete section 8's workspace-file check and a real model turn in the Control
UI. An HTTP 200 from the provider alone does not prove Codex execution.

A GitHub bot account is separate from both model authentication and teammate
login. Naming an Agent after an account does not authenticate it to GitHub.
GitHub tool credentials, repository permissions and verified human co-authors
are not configured by this recipe.

## If the provider rejects tools

A deployed revision can be healthy even when its model calls fail. An endpoint
backed by vLLM was observed returning HTTP 400 during a real Codex turn:
`Custom tools with a grammar format are not supported; only free-form text custom tools are accepted.`
The same endpoint completed a tool-free streaming Responses request.

This is a provider/Codex tool-format compatibility failure, not evidence of
failed OpenShift OAuth or missing OpenShell egress. Do not widen network access
or disable sandbox security to fix it. Use a provider/model that supports the
installed Codex tool schema, or resolve the compatibility before handing off
the Agent. A custom base URL does not guarantee every model backend works.
