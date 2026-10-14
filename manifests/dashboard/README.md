# OpenShell dashboard on the OpenShift dev cluster

Use the [deployment recipe](../../docs/dashboard.md)
after the Codex/OpenShell installation. The dashboard runs in `openshell-system`;
Keycloak and its operator run in `openshell-auth`.

| File                            | Purpose                                                                                |
| ------------------------------- | -------------------------------------------------------------------------------------- |
| `keycloak-operator.yaml`        | Red Hat build of Keycloak 26.6.7, with manual InstallPlan approval                     |
| `database.sql`                  | Separate Keycloak database and owner on the existing dev PostgreSQL instance           |
| `keycloak.yaml`                 | Single Keycloak instance, database TLS, edge Route, and router-only HTTP ingress       |
| `realm-import.yaml`             | Dashboard client, roles, and one initial dashboard user; credentials reference Secrets |
| `dashboard-values.yaml`         | Dashboard 0.1.0 image pins, Keycloak role gate, Route, and restricted-v2 settings      |
| `gateway-oidc-values.yaml`      | Adds Keycloak token validation to the existing OpenShell gateway                       |
| `dashboard-gateway-access.yaml` | Permits only the dashboard Pods to reach gateway port 8080                             |

Render `${KEYCLOAK_HOST}` and `${DASHBOARD_HOST}` into private copies. Leave the
realm's credential placeholders intact for the Keycloak operator. Keep generated
passwords, cookie secrets, client secrets, and kubeconfigs outside this directory.

This is a development deployment. The existing private-network OpenShell
authentication mode remains enabled for OCC; adding dashboard OIDC does not
convert that mode into production authentication. No OCC or Agent account is
created or changed. Do not edit OCC-managed Sandbox/provider settings through
the dashboard: OCC remains their configuration owner.
