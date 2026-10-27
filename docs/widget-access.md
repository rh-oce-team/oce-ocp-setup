# How the team accesses OpenClaw dashboard widgets

This describes the OpenShift widget deployment used by this repository. It covers
the Dashboard panel inside an Agent's OpenClaw Control UI, rather than the
separate OpenShell administration dashboard.

## How to open a dashboard

1. Open the Agent's human Control UI URL and sign in through Keycloak. An
   authorized operator can also use the OCC admin launcher.
2. Select the session containing the saved dashboard.
3. Open its **Dashboard** panel. Expand the panel if the layout looks cramped.

The human route and admin launcher use different authenticated profiles. Session
access and widget grants can therefore differ. The browser can also remember a
different panel layout for each origin.

## What your browser connects to

There are two separate web endpoints:

| Endpoint | Purpose | Access |
| --- | --- | --- |
| Agent Control UI | Authenticated session access and delivery of private widget content | Keycloak login, or an authorized OCC admin launcher session |
| Widget renderer / preview hostname | Isolated rendering shell and assets used by the widget iframe | Public HTTPS endpoint; no separate OAuth login |

The request path is:

```text
Your browser
  |
  +-- HTTPS :443 --> authenticated Control UI
  |                    |
  |                    +-- supplies authorized session/widget content
  |
  +-- HTTPS :443 --> OpenShift router --> renderer service :8081
  |                                      |
  |                                      +-- isolated iframe shell/assets
  |
  +-- HTTPS --> external API, if the widget makes a permitted browser request
```

The authenticated Control UI initializes the iframe with the widget content.
Opening the preview hostname directly does not select or display a saved private
dashboard. The `/mcp-app-sandbox` path serves the renderer shell; it is not a
standalone dashboard sharing link.

**The externally exposed port is HTTPS 443 on the OpenShift router.** The renderer
normally listens on internal port **8081**, which is not published directly to
the internet. The custom NetworkPolicy permits the default OpenShift router pods
to reach that port. It does not authenticate individual viewers: public requests
reach the renderer through the router.

This recipe uses edge TLS termination: browser-to-router traffic is encrypted,
and router-to-renderer traffic uses HTTP inside the cluster. The administrative
Gateway listener on **8080** must never be the backend of the public renderer
route.

## Why this does not need more OpenShell policy rules

OpenShell governs operations performed by processes in the Agent's sandbox.
Loading a renderer is browser-to-service ingress. A widget's direct API fetch is
browser egress. Neither request is an Agent process making an outbound request
through OpenShell.

Browser-side widget API access instead depends on the widget's exact
network-origin grants, browser security restrictions such as CORS, and the target
API's authentication and authorization. A widget grant does not grant the Agent
network access or supply API credentials.

If an Agent command fetches data, or a widget interaction invokes an Agent tool,
the resulting sandbox operation remains subject to OpenShell policy and its
credential broker. GitHub operational credentials should remain brokered there;
do not copy them into widget code, documents, URLs, or browser storage.

## Security tradeoffs and boundaries

- **There is a public renderer surface.** Anyone who can reach its hostname can
  request the shell/assets. Its safety depends on the renderer serving only that
  limited surface. Gateway RPCs, Console pages, credentials, and private workspace
  files must remain unavailable through this listener. A hard-to-guess hostname
  is not an access control.
- **Private content still requires authenticated access.** Public renderer
  hosting does not make saved dashboards public. The Control UI must authorize
  access to the session before delivering its content. Share the authenticated
  Control UI link with teammates, rather than treating the preview URL as a
  private or shareable dashboard link.
- **Widgets execute content in the viewer's browser.** The isolated iframe,
  origin separation, and network grants are security boundaries. Grant only the
  external origins a widget needs. An allowed origin can receive requests and
  any data the widget sends to it; review the widget and requested origins before
  granting access to sensitive data.
- **OpenShell does not filter browser traffic.** Browser API calls use the
  viewer's network and applicable browser/API controls. OpenShell's Agent egress
  allow-list is not a browser egress allow-list. This matters when a team expects
  all dashboard data access to pass through the Agent's policy and audit path.
- **Keep authentication cookies away from the renderer.** The preview domain
  must be outside the OCC shared-cookie domain. A separate origin alone does not
  prevent a broadly scoped domain cookie from being sent. Follow the
  [separate-domain preview procedure](native-admin-previews.md).
- **The internal hop is unencrypted in this recipe.** Edge TLS protects the
  public hop, but does not provide end-to-end transport encryption. Use a
  supported alternative TLS design if the cluster's requirements include
  encryption between the router and renderer.

## What maintainers should verify

After deployment or an upgrade, verify the renderer shell loads at
`https://<preview-host>/mcp-app-sandbox`, while `/console/` and
`/v1/chat/completions` on that same host return **404**. These checks are smoke
tests, not a complete security assessment. Confirm the Route targets the isolated
renderer port and the ingress policy selects the intended router peers.

Then sign in to the actual Control UI, open an existing saved dashboard, and
interact with a widget. A successful shell response alone does not prove that
authentication, private content delivery, grants, or widget interactions work.

For installation, persistence, and rollback details, see
[Enable MCP Apps and dashboard widgets](widgets.md). This document describes the
deployment design; it does not replace checking the current cluster resources.
