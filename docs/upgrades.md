# Build main and upgrade OCE

Run from this repository root with a dedicated, clean OCE checkout and protected
baseline files matching the live installation. Build from upstream main, then
freeze its commit and immutable image digests. Building never changes serving
images. Review upstream changes before applying them.

Controller-only upgrades stop OCC's API and worker for migrations and rollout,
but do not request Agent deployments. Console and OCC-proxied operations pause;
other affected runtime dependencies can interrupt Agents. Runtime upgrades also
redeploy every running Agent concurrently and can interrupt connections and turns.
Preserve database and Agent state; this is not a zero-downtime workflow.

## 1. Build the selected candidate

```bash
# REQUIRED: dedicated OCE release checkout, not another person's working tree.
export OCE_CHECKOUT='/absolute/path/to/openclaw-enterprise'

# REQUIRED: saved target-cluster connection and private installation files.
export KUBECONFIG='/absolute/path/to/kubeconfig'

export OCC_INPUT_DIRECTORY='/absolute/path/to/cluster-inputs'

# Choose controller for frequent OCC updates; runtime or both build those images.
scripts/build-images controller

oc -n openclaw-system get builds
```

Wait for the selected build to be Complete, then verify its architecture and
immutable digest. Use the same selection in both commands:

```bash
scripts/verify-images controller

source "$OCC_INPUT_DIRECTORY/images.env"

export OCC_REV="$(jq -er .oceCommit "$OCC_INPUT_DIRECTORY/runtime-source.json")"

# Move only the dedicated, clean release checkout to the exact built revision.
git -C "$OCE_CHECKOUT" switch --detach "$OCC_REV"

scripts/build-occ
```

The checkout must have its matching OCE workspace dependencies for startup
validation. Follow that checkout's dependency setup; do not reuse an incompatible
installed graph. OpenShell's approved Codex executable path must match the chosen
runtime image; review `manifests/openshift/openshell-installation-fragment.yaml`
when upstream changes its bundled Codex version.

## 2. Prepare the upgrade

Complete the selected OCE release's [upgrade checklist](https://github.com/openclaw/openclaw-enterprise/blob/main/docs/guides/deploy/upgrade-checklist.md)
before continuing. Compare protected inputs with live Helm values and the mounted
Installation Secret. Save a private baseline and back up PostgreSQL before
migrations; the dev database has no automatic backups. Back up Agent volumes
before runtime changes when recovery could require restoring state. Pause other
operators' Helm changes through completion or recovery.

Set your own protected OCC key:

```bash
export OCC_URL='https://<your OCC Console host>'

export OCC_SERVICE_KEY_FILE='/absolute/path/to/your-service-key.json'

export CONTEXT="$(oc config current-context)"
```

Bind the Installation Secret to the Installation returned by OCC. This annotation
is required by the upgrade helper. The command preserves an existing matching ID
and refuses to replace a different one:

```bash
export OCC_INSTALLATION_ID="$(bin/occ --output json installation get | jq -er .id)"

export OCC_INSTALLATION_SECRET="$(yq -er '.installation.secretName // "occ-installation-startup"' "$OCC_INPUT_DIRECTORY/values.yaml")"

existing_id="$(oc -n openclaw-system get secret "$OCC_INSTALLATION_SECRET" \
  -o go-template='{{ index .metadata.annotations "openclaw.dev/installation-id" }}')"

if test -z "$existing_id" || test "$existing_id" = '<no value>'; then
  oc -n openclaw-system annotate secret "$OCC_INSTALLATION_SECRET" \
    "openclaw.dev/installation-id=$OCC_INSTALLATION_ID"
elif test "$existing_id" != "$OCC_INSTALLATION_ID"; then
  echo 'Installation ID mismatch; stop and investigate before upgrading.' >&2
  false
fi
```

## 3. Upgrade the controller

Use a new evidence directory for each release. The helper creates it; do not
create it beforehand. OCC's API and worker stop during this command:

```bash
scripts/upgrade \
  --kubeconfig "$KUBECONFIG" --context "$CONTEXT" \
  --namespace openclaw-system --release oce \
  --values "$OCC_INPUT_DIRECTORY/values.yaml" \
  --installation "$OCC_INPUT_DIRECTORY/installation.yaml" \
  --controller-image "$CONTROLLER_IMAGE" \
  --source-revision "$OCC_REV" \
  --evidence-dir "$OCC_INPUT_DIRECTORY/upgrade-$OCC_REV-controller"
```

The helper preflights startup, stops writers, runs Helm migrations and checks
rollout. It neither creates backups nor proves model execution. Verify a real
model turn afterward and compare Agent revisions, PVC identities and workspace
hashes with the baseline.

The helper updates the `--values` and `--installation` files with the selected
images. Share those refreshed files with teammates after verification; older
copies will fail the next upgrade's live-state comparison.

For a runtime release, build and verify `runtime`, then use `--runtime-image
"$RUNTIME_IMAGE"` instead of `--controller-image`. Adding both image flags changes
both components. Check the upstream helper's current prerequisites for the
selected release, including repository brokers when enabled.

**Do not run `scripts/prepare-inputs`, repeat bootstrap-volume preparation, or
recreate system Secrets during an upgrade.** Those commands prepare a fresh
installation and reset routing fields. Baseline values and Installation files
must match live state; use separately reviewed candidate files for settings
changes. Preserve the dashboard's OpenShell OIDC overlay on later gateway
upgrades. OCC upgrades do not upgrade Keycloak or OpenShell automatically.

For a stopped or partially applied release, follow the upstream
[recovery procedure](https://github.com/openclaw/openclaw-enterprise/blob/main/docs/guides/deploy/production-upgrade-recovery.md)
instead of starting a second upgrade. No push or deployment runs automatically
when this repository or OCE main is fetched.

## Upgrade OpenShell separately

OpenShell has its own Helm release and SQLite PVC. Back up that database
consistently and retain its encryption/JWT Secrets before changing versions.
Keep the live values, including the dashboard OIDC overlay; applying only the
fresh-install values would discard that overlay. Agent PVCs and credentials stay
in place, but connections can pause while the gateway restarts.

Download and verify the selected charts using the
[OpenShell chart preparation](cluster.md#stage-openshell-separately), then run:

```bash
# REQUIRED: a new private directory for this OpenShell upgrade.
export OPENSHELL_UPGRADE_DIRECTORY="$OCC_INPUT_DIRECTORY/openshell-0.1.4-upgrade"

install -d -m 700 "$OPENSHELL_UPGRADE_DIRECTORY"

helm get values openshell-gateway -n openshell-system -o yaml \
  > "$OPENSHELL_UPGRADE_DIRECTORY/values-before.yaml"

cp "$OPENSHELL_UPGRADE_DIRECTORY/values-before.yaml" \
  "$OPENSHELL_UPGRADE_DIRECTORY/values.yaml"

export OPENSHELL_IMAGE_PINS="$PWD/manifests/openshift/openshell-gateway-values.yaml"

yq -i '.gateway.image = load(strenv(OPENSHELL_IMAGE_PINS)).gateway.image |
  .sandboxRuntime.image = load(strenv(OPENSHELL_IMAGE_PINS)).sandboxRuntime.image |
  .supervisor.image = load(strenv(OPENSHELL_IMAGE_PINS)).supervisor.image' \
  "$OPENSHELL_UPGRADE_DIRECTORY/values.yaml"

helm upgrade openshell-gateway \
  "$OCC_INPUT_DIRECTORY/openshell-charts/helm-chart-0.1.4.tgz" \
  --namespace openshell-system \
  -f "$OPENSHELL_UPGRADE_DIRECTORY/values.yaml" \
  --wait --timeout 7m

oc -n openshell-system rollout status statefulset/openshell-gateway --timeout=5m
```

Existing Sandboxes retain their supervisor image until restarted or replaced.
Deploy each intended Agent through OCC to create its new-version Sandbox; keep
its existing Configuration and credential sources. Verify a model turn, tool
credentials and the dashboard afterward. Share the refreshed protected gateway
values with maintainers separately from Secrets.
