# BookStack Helm Chart – Configuration & Other Setups

This document covers **optional configuration** and **alternative setups** (e.g. using values instead of Kubernetes Secrets, Azure AD app registration, SMTP, port-forward only).  
**Preferred method:** [README.md](README.md) — install using **Kubernetes Secrets** for all sensitive values.

## Contents

- [Prerequisites](#prerequisites)
- [Alternative: Using values instead of Secrets](#alternative-using-values-instead-of-secrets)
- [Port-forward only (no host / no Ingress)](#port-forward-only-no-host--no-ingress)
- [Configuration reference](#configuration-reference)
- [Azure AD (Entra ID) app registration](#azure-ad-entra-id-app-registration)
- [SMTP setup (inline values)](#smtp-setup-inline-values)
- [SMTP with Azure Communication Services](#smtp-with-azure-communication-services)
- [Switching to Ingress later](#switching-to-ingress-later)
- [Uninstall](#uninstall)

---

## Prerequisites

- Kubernetes cluster (1.24+)
- Helm 3.x
- **Ingress** (optional): NGINX Ingress Controller and cert-manager with a ClusterIssuer (e.g. `letsencrypt-prod`) if you use Ingress; omit if you use port-forward only
- (Optional) Azure AD app registration for SSO
- (Optional) SMTP server for email

---

## Alternative: Using values instead of Secrets

You can pass sensitive values via `values.yaml` or `--set` instead of Kubernetes Secrets. This is **not recommended** for production (secrets may end up in history or config files).

**Quick start (inline values, with Ingress):**

```bash
helm upgrade --install <release_name> ./charts/bookstack --namespace=<namespace> --create-namespace \
  --set appHost=wiki.example.com \
  --set appKey=base64:$(openssl rand -base64 32) \
  --set azuread.enabled=false \
  --set smtp.enabled=false
```

**Quick start from Helm repo:**

```bash
helm repo add bookstack-helm https://lumunix.github.io/bookstack-helm/
helm repo update
helm upgrade --install <release_name> bookstack-helm/bookstack --namespace=<namespace> --create-namespace --values values.yaml
```

`appKey` is required when not using Secrets. Generate one: `openssl rand -base64 32` (use with `base64:` prefix in BookStack).

For production, use the [Kubernetes Secrets method in the README](README.md#quick-start).

---

## Port-forward only (no host / no Ingress)

You can run BookStack without a host or Ingress and access it only via `kubectl port-forward`.

1. Disable Ingress: `ingress.enabled: false`.
2. Omit `appHost`; optionally set `appUrl` to the URL you use in the browser (e.g. `http://localhost:8080`).

**With Kubernetes Secrets (preferred):**

```bash
kubectl create namespace bookstack
kubectl create secret generic bookstack-secrets -n bookstack --from-literal=app-key="base64:$(openssl rand -base64 32)"
helm upgrade --install my-wiki ./charts/bookstack -n bookstack \
  --set kubernetesSecret.name=bookstack-secrets \
  --set ingress.enabled=false \
  --set azuread.enabled=false \
  --set smtp.enabled=false
```

**Access:**

```bash
kubectl port-forward -n bookstack svc/my-wiki-bookstack 8080:8080
```

Open **http://localhost:8080**. If you use a different port, set `appUrl` accordingly (e.g. `http://localhost:9000`).

---

## Configuration reference

| Parameter           | Description                                      | Default / Required   |
|--------------------|--------------------------------------------------|----------------------|
| `appHost`          | Public hostname. Required only when Ingress is enabled. | Optional |
| `appUrl`           | Full URL for links/redirects. Overrides `https://` + appHost when set. | Optional |
| `appKey`           | Application encryption key. **Required** only when not using [kubernetesSecret](README.md#quick-start). | — |
| `kubernetesSecret.name` | Use a Kubernetes Secret for sensitive values (fixed key names). **Preferred.** See [README](README.md). | — |
| `ingress.enabled`  | Create Ingress. Set `false` for port-forward only. | `true` |
| `service.port`     | Service port exposed by ClusterIP and used by Ingress backend. | `8080` |
| `allowHttp`        | Allow HTTP on Ingress. | `false` |
| `storageType`      | BookStack storage driver. | `local_secure` |
| `azuread.*`        | Azure AD SSO. When using Secrets, put tenantId/appId/appSecret in the Secret; see [README](README.md). | — |
| `smtp.*`           | SMTP for email. When using Secrets, put password in the Secret; see [README](README.md). | — |

The chart deploys: BookStack (solidnerd/bookstack:25.12), MySQL 5.7, PVCs for data/uploads/storage, and optionally an Ingress (nginx + cert-manager).

---

## Azure AD (Entra ID) app registration

To use Azure AD for login, register an app and either put credentials in a [Kubernetes Secret](README.md#which-secrets-are-required-for-which-functionality) (preferred) or in values (see [Alternative: using values](#alternative-using-values-instead-of-secrets)).

### 1. Register an application in Azure

1. Open [Azure Portal](https://portal.azure.com) → **Microsoft Entra ID** → **App registrations** → **New registration**.
2. Set a name (e.g. "BookStack Wiki"), choose supported account types, and set **Redirect URI**:
   - Type: **Web**
   - **With Ingress (public host):** `https://<your-appHost>/oidc/callback` (e.g. `https://wiki.example.com/oidc/callback`)
   - **Port-forward only:** `http://localhost:8080/oidc/callback` (use the port you forward to)
3. Register and note **Application (client) ID** and **Directory (tenant) ID**.

### 2. Create a client secret

In the app: **Certificates & secrets** → **New client secret**. Copy the **Value** immediately — this is the client secret (store it in a Kubernetes Secret; see [README](README.md)).

### 3. API permissions

**API permissions** → **Add a permission** → **Microsoft Graph** → **Delegated**. Add: `openid`, `User.Read`, `email` (for auto-register/confirm).

### 4. Use with this chart

- **Preferred:** Put `azure-tenant-id`, `azure-app-id`, `azure-app-secret` in your Kubernetes Secret and set `azuread.enabled: true`. See [README](README.md).
- **Alternative:** Set `azuread.enabled: true` and `tenantId`, `appId`, `appSecret` in values (not recommended for production).

The chart sets `AZURE_AUTO_REGISTER` and `AZURE_AUTO_CONFIRM_EMAIL` to `true` automatically.

---

## SMTP setup (inline values)

To enable email with inline values (password in values), set in `values.yaml`:

```yaml
smtp:
  enabled: true
  host: smtp.example.com
  port: "587"
  username: your-smtp-user
  password: your-smtp-password
  fromAddress: noreply@example.com
  fromName: "My Wiki"
```

**Preferred:** Store the password in a Kubernetes Secret with key `smtp-password` and set `kubernetesSecret.name`; omit `smtp.password` from values. See [README](README.md).

The chart sets `MAIL_ENCRYPTION` to `tls`.

---

## SMTP with Azure Communication Services

You can use **Azure Communication Services** (Email) as the SMTP provider (`smtp.azurecomm.net`, port 587, TLS).

### 1. Create Azure resources

1. Create an **Azure Communication Services** resource.
2. Create an **Azure Communication Email** resource and [provision a domain](https://learn.microsoft.com/en-us/azure/communication-services/quickstarts/email/create-email-communication-resource).
3. [Connect the Email resource](https://learn.microsoft.com/en-us/azure/communication-services/quickstarts/email/connect-email-communication-resource) to the Communication Services resource.

### 2. SMTP credentials (Entra ID app + role)

1. Register an app in **Microsoft Entra ID**; create a **client secret** (this will be the SMTP password — store it in your Kubernetes Secret as `smtp-password`).
2. On the **Communication Services** resource: **Access control (IAM)** → **Add role assignment** → Role **Communication and Email Service Owner** → Members: your Entra application.
3. In the Communication Services resource: **SMTP Usernames** → **+ Add SMTP Username** → select your Entra app, set username.

### 3. Helm configuration

Use a Secret for the password (key `smtp-password`) and set in values:

```yaml
smtp:
  enabled: true
  host: smtp.azurecomm.net
  port: "587"
  username: "<your-smtp-username>"    # From SMTP Usernames in Azure
  # password from Kubernetes Secret (smtp-password)
  fromAddress: "donotreply@xxxxxxxx.azurecomm.net"   # From your provisioned domain
  fromName: "My Wiki"
```

References: [Send email with SMTP - Azure Communication Services](https://learn.microsoft.com/en-us/azure/communication-services/quickstarts/email/send-email-smtp/send-email-smtp), [SMTP authentication](https://learn.microsoft.com/en-us/azure/communication-services/quickstarts/email/send-email-smtp/smtp-authentication).

---

## Switching to Ingress later

If you started with port-forward only, you can enable Ingress later:

1. `helm upgrade` with `ingress.enabled=true` and `appHost=<your-domain>`.
2. Unset or remove `appUrl` if you want the app to use `https://` + appHost.
3. In the Azure AD app registration, add redirect URI `https://<your-domain>/oidc/callback` (you can keep or remove the localhost URI).

No reinstall needed.

---

## Uninstall

```bash
helm uninstall <release_name> --namespace=<namespace>
```

PVCs are not removed by default; delete them manually to free storage.
