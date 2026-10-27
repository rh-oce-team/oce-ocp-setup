# Share a private Claw links directory

Generate a list of human login links and pin it in your private team Slack
channel. Actual Claw names, hostnames and the generated list stay outside this
public repository.

## Generate the directory

Select the intended cluster kubeconfig and confirm the server. The helper only
reads Routes named `oce-team-claw-oidc`, created by the [personal](agent-keycloak-login.md)
and [team](team-login.md) login recipes. It does not change cluster resources.

```bash
oc whoami --show-server

umask 077

# REQUIRED: private directory outside all source checkouts.
export CLAW_LINKS_DIRECTORY="$HOME/.local/state/oce-claw-links"

install -d -m 700 "$CLAW_LINKS_DIRECTORY"

scripts/claw-links > "$CLAW_LINKS_DIRECTORY/claw-links.txt"
```

By default each link is labeled with its hostname. To use memorable display
names, pass one `--name '<Kubernetes namespace>=<display name>'` per Claw. Find
the namespace in the output, then rerun with the desired labels:

```bash
scripts/claw-links --name 'oce-example=My Claw' \
  > "$CLAW_LINKS_DIRECTORY/claw-links.txt"
```

Open the private file, match each link to its intended Agent, and verify that
it reaches the human login and connects with your individual profile. Share the
text in your private Slack channel and pin the message. Regenerate it after
provisioning a Claw or changing its hostname, and update the pinned message.
The helper discovers configured URLs; it does not verify endpoint health or
membership. Review the list before sharing.

## Which link to use

Human links use your own Keycloak account and require membership in that Claw.
OCC's **Open admin console** is a separate administrative entry point using
the shared `occ-workspace-files` identity, authorized by your OCC permissions.

Choose a memorable hostname when provisioning a new Claw. Changing an existing
hostname also requires updating the OIDC client redirect URL and Gateway allowed
origin; editing a shared link alone does not move the endpoint.
