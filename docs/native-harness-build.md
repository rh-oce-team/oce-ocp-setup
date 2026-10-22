# Prepare the standard runtime for native OpenClaw

The standard OCE runtime supports native OpenClaw dedicated Agents through
OpenShell. **Do not build PR #1982 or a separate native image to add an Agent.**
PR #1685 merged the product support into main; #1982 supplies additional test
coverage. Use [native Agent creation](native-harness.md) directly when the
installed OCC reports `capabilities.nativeWorkers.support: "pinned-runtime"`.

## Reuse an installed runtime

Read the current capability using the installed OCC CLI and your authorized
connection:

```bash
bin/occ installation get -o json | jq -e \
  '.capabilities.nativeWorkers.support == "pinned-runtime"'
```

Expect `true`. Keep the existing verified runtime image digests and complete
[native Agent creation](native-harness.md). No build or fleet redeployment is
needed for another Agent.

## Upgrade an older Installation

If the capability is absent, select an OCE source commit that includes native
worker support. Preserve any reviewed deployment PRs required by existing
Agents, such as experimental human identity ingress. Those PRs are independent
of the native Harness choice.

Follow [the upgrade recipe](upgrades.md) to freeze the source, build and verify
the standard controller and runtime (resolve the published images only when no
unmerged PR is needed), back up the Installation, and run OCE's
coordinated upgrade helper. The helper updates the runtime fleet and briefly
interrupts Agents; this is an operator operation, not part of each Agent creation.
Recheck the capability after upgrade, then return to
[native Agent creation](native-harness.md).

## Custom source trials

A newer OpenClaw source is a separate compatibility trial. It is not required
for the pinned native runtime. Record both commits and qualify enrollment,
worker execution, credentials, and persistence before sharing that image with
existing Agents. PR #3041 adds persistence for newer OpenClaw client-store
layouts; check its merge status against the exact OCE source before selecting a
newer OpenClaw commit. Do not treat a successful image build as runtime proof,
set `nativeWorkerSupport: custom-image` to bypass qualification, or overwrite
shared image selections just to create one Agent.
