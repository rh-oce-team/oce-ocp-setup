# Create a dedicated native OpenClaw Agent through OpenShell

Use the installed OCE runtime to create an Agent with the experimental native OpenClaw Harness.
Native dedicated support is included in OCE main through PR #1685. **PR #1982,
a custom OpenClaw build, and a controller rebuild are not required.** The standard
pinned runtime supports both Codex and native OpenClaw dedicated Agents.

Run from this setup repository with an OCC administrator service key and the
OpenShift operator permissions used by the additional-Agent recipe. Keep each
Agent's inputs outside Git. This recipe uses the OCC admin launcher as
`occ-workspace-files`; it does not create a Keycloak account or human login Route.
Existing Agents and their identity settings stay in place.

## 1. Check the installed runtime

Use the OCC connection and kubeconfig for the intended cluster. Save its current
Installation record in a protected directory, then require native worker support:

```bash
umask 077

# REQUIRED: existing protected cluster setup directory.
export OCC_SETUP_DIRECTORY='/absolute/path/to/cluster-setup'

# REQUIRED: authorized OCC administrator key for this installation.
export OCC_SERVICE_KEY_FILE='/absolute/path/to/occ-service-key.json'

export OCC_URL="https://$(yq -er '.spec.host' "$OCC_SETUP_DIRECTORY/console-route.yaml")"

bin/occ installation get -o json > "$OCC_SETUP_DIRECTORY/native-capabilities.json"

jq -e '.capabilities.nativeWorkers.support == "pinned-runtime"' \
  "$OCC_SETUP_DIRECTORY/native-capabilities.json"
```

Expect `true`. If it fails, stop and use [runtime preparation](native-harness-build.md).
Do not bypass this check with `runtime.nativeWorkerSupport: custom-image`.

Check the installed startup `installation.yaml` before creation. OpenShell's
Credential Gateway must allow `/usr/local/bin/node` in `binaries`, in addition
to any approved Codex binary. Native Harness resources must fit the configured
session capacity and Namespace quota. Our two-slot trial uses a 6 GiB Harness
memory limit and a 32 GiB Namespace memory quota; those are tested settings,
not universal minimums. The OpenShift setup uses `userNamespaces: false` for its
supported admission path. Preserve other Agents' grants, images, routing, and
resource settings. If a shared setting needs to change, complete the selected OCE
source's `docs/guides/deploy/production-installation-changes.md` before proceeding.
Adding another Agent to an already prepared Installation does not restart OCC.

## 2. Prepare the Agent and model credential

Start [additional-Agent creation](agents.md) at section 1. Before running its
commands, replace its name variables with these values, then complete sections
1–3 using the new Namespace and input directory:

```bash
export AGENT_NAME='native'

export AGENT_DISPLAY_NAME='Native'
```

Use an OpenAI API key authorized for `gpt-6-luna`. A Codex PAT is not a native
model credential. Keep the recipe's exact credential-source grant, image-pull
permissions, and Gateway SCC steps.

Before section 4, read the model metadata from the OpenClaw commit pinned by
the **installed OCE source**. A local OpenClaw checkout is used to read that
catalog, not to build another image. It must contain the pinned commit.

```bash
# REQUIRED: source checkout at the exact OCE commit deployed to this cluster.
export OCE_CHECKOUT='/absolute/path/to/installed-oce-checkout'

# REQUIRED: OpenClaw repository containing the installed runtime's pinned commit.
export OPENCLAW_CHECKOUT='/absolute/path/to/openclaw'

export OPENCLAW_COMMIT="$(sed -n 's/^ARG OPENCLAW_COMMIT=//p' "$OCE_CHECKOUT/deploy/runtime/Dockerfile")"

git -C "$OPENCLAW_CHECKOUT" show \
  "$OPENCLAW_COMMIT:extensions/openai/openclaw.plugin.json" |
  jq -e '[.modelCatalog.providers.openai.models[] | select(.id == "gpt-6-luna")] |
    if length == 1 then .[0] |
      {id,name,reasoning,input,contextWindow,maxTokens,thinkingLevelMap,
       cost:(.cost | {input,output,cacheRead,cacheWrite})}
    else error("Expected exactly one GPT-6-Luna entry in the pinned source") end' \
  > "$AGENT_INPUT_DIRECTORY/native-model.json"

jq . "$AGENT_INPUT_DIRECTORY/native-model.json"
```

Review the metadata. Cost fields are catalog values per million tokens; they do
not change provider charges. Build the native Configuration from the shared
example, removing the Codex plugin and selecting the OpenClaw runtime:

```bash
jq -e --slurpfile model "$AGENT_INPUT_DIRECTORY/native-model.json" '
  $model[0] as $m |
  if ($m.id == "gpt-6-luna" and
      ($m.contextWindow | type == "number" and . > 0 and floor == .) and
      ($m.maxTokens | type == "number" and . > 0 and floor == .) and
      ([$m.cost.input, $m.cost.output, $m.cost.cacheRead, $m.cost.cacheWrite] |
        all(type == "number" and . >= 0))) then
    del(.values.plugins) |
    .values.agents.defaults.model = "openai/gpt-6-luna" |
    .values.agents.defaults.models = {"openai/gpt-6-luna": {agentRuntime:{id:"openclaw"}}} |
    .values.models.providers = {openai:{
      baseUrl:"https://api.openai.com/v1", api:"openai-responses",
      apiKey:{source:"env",provider:"model",id:"OPENAI_API_KEY"}, models:[$m]}} |
    .values.secrets.providers.model = {source:"env",allowlist:["OPENAI_API_KEY"]}
  else error("Review the required native model metadata before creating the Configuration") end
' manifests/openshift/codex-luna-configuration.json \
  > "$AGENT_INPUT_DIRECTORY/native-configuration.json"

export CONFIGURATION_FILE="$AGENT_INPUT_DIRECTORY/native-configuration.json"
```

Continue [additional-Agent creation](agents.md#4-create-the-configuration-and-agent)
sections 4 and 5. The naming step sets both the OCC Agent name and the OpenClaw
assistant profile. Both Harness types use `executionMode: dedicated`;
`agentRuntime.id: openclaw` selects the native Harness.

Run only the deployment block in section 6. For this admin-only procedure, skip
its OpenShift OAuth Route and owner handoff. The model key is delivered to the
native Harness; the separate Gateway receives no inference key. This recipe
uses `openai/` models at `https://api.openai.com/v1`; custom endpoints need their
own supported configuration and qualification.

## 3. Enable the admin launcher and verify execution

Complete [the OCC admin launcher procedure](native-admin.md) for the new Agent.
If its global feature is already enabled, retain the installed settings and add
only this Agent's exact hostname, certificate, and allowed origin. Verify
certificate issuance before opening the link. No Keycloak client is needed.

Compute creates the required `dedicated-native` profile and selects
`inference: worker`. Do not add a competing profile or manually select a Cloud
Worker. Open the Control UI from the Agent's OCC page and require a complete
model response. Create missing initial workspace files through OCC, then ask
the Agent to read `/sandbox/enterprise/AGENTS.md`; check the same file in OCC.
A ready Pod alone does not prove inference or workspace access.

Retain the OCE and OpenClaw commits, runtime digest, Agent IDs, and model/workspace
results in the private Agent directory. Test another turn after a normal Gateway
replacement before claiming persistence proof. Leave existing Codex Agents intact.
