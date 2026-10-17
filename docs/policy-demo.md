# Demo a denied request, approve its policy, and retry

**Experimental prerequisite:** build and deploy an OCE controller containing
[PR #2599](https://github.com/openclaw/openclaw-enterprise/pull/2599), which is open
and not yet merged (verified October 16, 2026). Use dedicated Codex with OpenShell,
a working model credential, and an OCC administrator login. The Agent's Control UI
and OCC Console are separate browser tabs.

## Prepare the Installation once

The Installation delegation ceiling is the operator's limit on what Agent policy
editors may approve. An empty ceiling rejects every addition, even for an
administrator. It grants no network access by itself.

The example ceiling permits four exact executable/hostname pairs, on port 443.
It does not allow other Python versions, other executables or arbitrary Google
subdomains. Check the resolved interpreter inside your image; this trial used
`/usr/bin/python3.11`.

```bash
# REQUIRED: protected inputs for the deployed OCC installation.
export OCC_SETUP_DIRECTORY='/private/path/to/cluster-setup'

cp "$OCC_SETUP_DIRECTORY/installation.yaml" \
  "$OCC_SETUP_DIRECTORY/installation.yaml.before-policy-demo"

# Run from this setup repository root.
export DEMO_CEILING_FILE="$PWD/manifests/agents/policy-demo-ceiling.json"

yq -i '.sandboxNetworkPolicyCeiling = load(strenv(DEMO_CEILING_FILE))' \
  "$OCC_SETUP_DIRECTORY/installation.yaml"
```

Review the four entries before replacing any existing ceiling; retain other
approved entries when your Installation already delegates access. Apply the
edited file using OCE's
[Installation-change procedure](https://github.com/openclaw/openclaw-enterprise/blob/main/docs/guides/deploy/production-installation-changes.md).
Run it from the installed controller's source checkout. Keep the controller and
runtime image references unchanged. The checksum update restarts OCC's API and
worker; it does not redeploy the Agents.

In OCC, open the Agent, select its current running version, and choose **Policy
denials**. Under **Agent network policy**, enable policy editing if needed. Confirm
that **Installation delegation ceiling** now lists the demo destinations.
Enabling editing requires Installation and Agent administration; editing additions
also requires Agent `manage_policy` and SandboxPolicy `update` permissions.

## Example 1: Python fetches Example Domain

Start with no Python → `example.com` addition. Ask the Agent in its Control UI:

```text
Use only /usr/bin/python3.11 and Python's urllib.request to fetch
https://example.com/. Print the HTTP status and whether the response contains
"Example Domain". Use a 15-second timeout. Do not use curl, a browser, another
interpreter, proxies or a fallback. If network policy blocks the request,
report the denial and stop.
```

In the OCC **Policy denials** tab:

1. Find the denial for `/usr/bin/python3.11` and `example.com:443`.
2. Under **Policy suggestions**, choose **Review suggestion** for that exact
   executable, hostname and port. Unrelated model-client suggestions are not part
   of this demo. If the provider has not produced a suggestion, add that exact
   rule manually under **Proposed network rules**.
3. Select **Review changes → Save and apply to current revision**.
4. Wait for application state **applied** and its provider version. **Save** alone
   does not change the running sandbox.
5. Ask the Agent:

```text
Retry the exact Python request now. Use the same interpreter and URL, with no
fallback. Print the HTTP status and whether "Example Domain" is present.
```

Expect HTTP `200` and a matching page title/content. Live policy application needs
no image build or Agent redeployment.

## Two more examples

Repeat the same deny → review → save-and-apply → retry sequence for each.

**Google robots file:** use the final hostname explicitly to avoid introducing
an extra redirect destination.

```text
Use only /usr/bin/python3.11 and urllib.request to GET
https://www.google.com/robots.txt with a 15-second timeout. Print the HTTP status
and first three lines. Do not switch tools or follow a fallback. If policy blocks
the request, report the denial and stop.
```

Approve Python → `www.google.com:443`, then repeat the request. Starting at
`https://google.com/` can redirect to `www.google.com`; the apex grant does not
authorize that second hostname. The example ceiling includes both so each exact
suggestion can be approved when demonstrating redirects intentionally.

**GitHub API with curl:** the Installation's existing git rule does not grant
curl access to the GitHub API.

```text
Use only /usr/bin/curl to GET https://api.github.com/zen with a 15-second timeout.
Print the response and HTTP status. Do not use git, Python, a browser, proxies
or a fallback. If policy blocks it, report the denial and stop.
```

Approve curl → `api.github.com:443`, then repeat. Expect an HTTP response from the
GitHub API; its unauthenticated rate limits still apply.

## Reset for another presentation

Remove only the addition used by your demo, then **Review changes → Save and
apply to current revision** and wait for **applied**. Retry the original prompt to
show denial again. Preserve other approved additions and read-only defaults.
Old provider suggestions may remain visible; their presence does not mean access
is granted. Use the current policy and confirmed application state as evidence.

A ceiling error means the complete executable/host/port request does not fit the
operator's ceiling. Check the resolved binary, exact hostname, redirected hosts
and requested port range. A saved policy with a failed or pending application
has not established enforcement; inspect the application result before retrying.
