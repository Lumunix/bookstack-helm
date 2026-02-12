# BookStack Helm Chart

Helm chart for [BookStack](https://www.bookstackapp.com/) with MySQL on Kubernetes. Supports optional **OIDC** (primary SSO, including Azure Entra ID), optional **Azure AD social login**, and **SMTP** for email.

**Preferred method:** store all sensitive values (app key, OIDC/Azure credentials, SMTP password) in a **Kubernetes Secret** and reference it via `kubernetesSecret`. Other setups (inline values, port-forward only, OIDC/Azure/SMTP details) are in **[CONFIGURATION.md](CONFIGURATION.md)**. Unsupported hacks (e.g. Mermaid Viewer) are in **[HACKS.md](HACKS.md)**.

## Contents

- [Prerequisites](#prerequisites)
- [Quick Start (Kubernetes Secrets)](#quick-start-kubernetes-secrets)
- [Secrets required per functionality](#secrets-required-per-functionality)
- [Configuration](#configuration)
- [Other setups](#other-setups)
- [Hacks](#hacks)
- [Changelog](#changelog)
- [Uninstall](#uninstall)

---

## Prerequisites

- Kubernetes 1.24+, Helm 3.x
- Optional: NGINX Ingress + cert-manager if you use Ingress; otherwise you can use [port-forward](CONFIGURATION.md#port-forward-only-no-host--no-ingress)

---

## Quick Start (Kubernetes Secrets)

1. **Create a Secret** in the same namespace as the release with the keys you need (see [Secrets required per functionality](#secrets-required-per-functionality)).

2. **Install or upgrade** the chart with `kubernetesSecret.name` set to that Secret. Do not set `appKey` (or Azure/SMTP secrets) in values when they come from the Secret.

**Minimal (no Azure AD, no SMTP):**

```bash
kubectl create namespace bookstack
kubectl create secret generic bookstack-secrets -n bookstack \
  --from-literal=app-key="base64:$(openssl rand -base64 32)"

helm repo add bookstack-helm https://lumunix.github.io/bookstack-helm/
helm repo update
helm upgrade --install my-wiki bookstack-helm/bookstack -n bookstack --create-namespace \
  --set kubernetesSecret.name=bookstack-secrets \
  --set ingress.enabled=false
```

**Access via port-forward:**

```bash
kubectl port-forward -n bookstack svc/my-wiki-bookstack 8080:8080
# Open http://localhost:8080
```

**With Ingress (public host):**

```bash
# Create Secret with at least app-key (and Azure/SMTP keys if you enable those)
kubectl create secret generic bookstack-secrets -n bookstack \
  --from-literal=app-key="base64:$(openssl rand -base64 32)"

helm upgrade --install my-wiki bookstack-helm/bookstack -n bookstack --create-namespace \
  --set kubernetesSecret.name=bookstack-secrets \
  --set appHost=wiki.example.com
```

**From local chart:**

```bash
helm upgrade --install my-wiki ./charts/bookstack -n bookstack --create-namespace \
  -f values.yaml
```

Example values files using Kubernetes Secret:
- **Ingress/public host + OIDC/Azure-only login (no SMTP):** **[charts/bookstack/values-test-oidc.yaml](charts/bookstack/values-test-oidc.yaml)** (`-f charts/bookstack/values-test-oidc.yaml`)
- **Azure AD + SMTP:** **[charts/bookstack/values-test.yaml](charts/bookstack/values-test.yaml)** (`-f charts/bookstack/values-test.yaml`)
- **Port-forward only + Azure AD (no SMTP):** **[charts/bookstack/values-test-portforward-azuread.yaml](charts/bookstack/values-test-portforward-azuread.yaml)** (`-f charts/bookstack/values-test-portforward-azuread.yaml`)
- **Port-forward only + OIDC/Azure-only login (no SMTP):** **[charts/bookstack/values-test-portforward-oidc.yaml](charts/bookstack/values-test-portforward-oidc.yaml)** (`-f charts/bookstack/values-test-portforward-oidc.yaml`)

Ensure the Secret exists before install/upgrade; otherwise the Pod will not start until it is created.

---

## Secrets required per functionality

Include **only** the secret keys for the options you enable. The chart expects these exact key names inside the Secret.

| Functionality       | Secret keys (default names) |
|---------------------|-----------------------------|
| **Base (all)**      | `app-key` — BookStack app key (e.g. `base64:...`) |
| **OIDC only**       | `app-key` + `oidc-client-secret` (recommended) |
| **Azure AD only**   | `app-key` + `azure-tenant-id` + `azure-app-id` + `azure-app-secret` |
| **SMTP only**       | `app-key` + `smtp-password` |
| **Azure AD + SMTP** | `app-key` + `azure-tenant-id` + `azure-app-id` + `azure-app-secret` + `smtp-password` |

**Key reference:**

| Secret key (default) | Description        | When used |
|----------------------|--------------------|-----------|
| `app-key`            | BookStack APP_KEY  | Always (when using kubernetesSecret) |
| `oidc-client-secret` | OIDC client secret | `oidc.enabled: true` |
| `azure-tenant-id`    | Azure AD tenant ID | `azuread.enabled: true` |
| `azure-app-id`       | Azure AD app ID    | `azuread.enabled: true` |
| `azure-app-secret`   | Azure AD secret    | `azuread.enabled: true` |
| `smtp-password`      | SMTP password      | `smtp.enabled: true` |

**Example – OIDC only (Azure-only login, no SMTP):**

```bash
kubectl create secret generic bookstack-secrets -n bookstack \
  --from-literal=app-key='base64:YOUR_APP_KEY' \
  --from-literal=oidc-client-secret='YOUR_OIDC_CLIENT_SECRET'
```

```yaml
kubernetesSecret:
  name: bookstack-secrets
oidc:
  enabled: true
  issuer: "https://login.microsoftonline.com/<tenant-id>/v2.0"
  clientId: "YOUR_APP_ID"
  # clientSecret from Secret key: oidc-client-secret
  name: "Azure AD"
  autoInitiate: true
azuread:
  enabled: false
smtp:
  enabled: false
```

**Example – Azure AD only (no SMTP):**

```bash
kubectl create secret generic bookstack-secrets -n bookstack \
  --from-literal=app-key='base64:YOUR_APP_KEY' \
  --from-literal=azure-tenant-id='YOUR_TENANT_ID' \
  --from-literal=azure-app-id='YOUR_APP_ID' \
  --from-literal=azure-app-secret='YOUR_CLIENT_SECRET'
```

```yaml
# values.yaml (or --set)
kubernetesSecret:
  name: bookstack-secrets
azuread:
  enabled: true
smtp:
  enabled: false
```

**Example – Azure AD + SMTP:**

```bash
kubectl create secret generic bookstack-secrets -n bookstack \
  --from-literal=app-key='base64:YOUR_APP_KEY' \
  --from-literal=azure-tenant-id='YOUR_TENANT_ID' \
  --from-literal=azure-app-id='YOUR_APP_ID' \
  --from-literal=azure-app-secret='YOUR_CLIENT_SECRET' \
  --from-literal=smtp-password='YOUR_SMTP_PASSWORD'
```

```yaml
kubernetesSecret:
  name: bookstack-secrets
azuread:
  enabled: true
smtp:
  enabled: true
  host: smtp.example.com
  port: "587"
  username: myuser
  fromAddress: noreply@example.com
  fromName: "My Wiki"
```

---

## Configuration

See sample values files:
- **Ingress/public host + OIDC/Azure-only login (no SMTP):** **[charts/bookstack/values-test-oidc.yaml](charts/bookstack/values-test-oidc.yaml)**
- **Azure AD + SMTP:** **[charts/bookstack/values-test.yaml](charts/bookstack/values-test.yaml)**
- **Port-forward only + Azure AD (no SMTP):** **[charts/bookstack/values-test-portforward-azuread.yaml](charts/bookstack/values-test-portforward-azuread.yaml)**
- **Port-forward only + OIDC/Azure-only login (no SMTP):** **[charts/bookstack/values-test-portforward-oidc.yaml](charts/bookstack/values-test-portforward-oidc.yaml)**

| Parameter           | Description |
|--------------------|-------------|
| `kubernetesSecret.name` | **Preferred.** Name of the Secret holding app-key (and optionally Azure/SMTP keys). |
| `appHost`          | Public hostname; required when Ingress is enabled. |
| `appUrl`           | Full URL for links/redirects (e.g. port-forward: `http://localhost:8080`). |
| `ingress.enabled`  | Create Ingress. Set `false` for port-forward only. Default: `true`. |
| `service.port`     | ClusterIP Service port used by Ingress backend. Default: `8080`. |
| `mermaidViewer.enabled` | Enable Mermaid Viewer hack for interactive diagrams. See [HACKS.md](HACKS.md#mermaid-viewer). Default: `false`. |
| `oidc.enabled`      | Enable OIDC as primary authentication (`AUTH_METHOD=oidc`), which replaces local email/password login. |
| `azuread.enabled`   | Enable Azure AD login. Credentials from Secret or values; see [CONFIGURATION.md](CONFIGURATION.md). |
| `smtp.enabled`      | Enable SMTP. Password from Secret or values; see [CONFIGURATION.md](CONFIGURATION.md). |

More options and defaults: see [CONFIGURATION.md – Configuration reference](CONFIGURATION.md#configuration-reference).

---

## Other setups

The following are documented in **[CONFIGURATION.md](CONFIGURATION.md)**:

- **Using values instead of Secrets** — passing `appKey` and other secrets via values or `--set` (not recommended for production)
- **Port-forward only** — no Ingress, no host; access via `kubectl port-forward`
- **OIDC (primary auth)** — Azure-only login via OIDC (`AUTH_METHOD=oidc`)
- **Azure AD app registration** — step-by-step in Azure Portal (redirect URI, client secret, API permissions)
- **SMTP setup** — host, port, from address; inline or via Secret
- **SMTP with Azure Communication Services** — using `smtp.azurecomm.net`
- **Switching to Ingress later** — from port-forward to a public host

---

## Hacks

**[HACKS.md](HACKS.md)** documents unsupported BookStack customizations built into this chart (e.g. [Mermaid Viewer](HACKS.md#mermaid-viewer) for interactive diagrams). Apply at your own risk.

---

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for versioned release titles and descriptions.

---

## Uninstall

```bash
helm uninstall <release_name> --namespace=<namespace>
```

PVCs are not deleted by default; remove them manually if needed.

---

## License

See [LICENSE](LICENSE).
