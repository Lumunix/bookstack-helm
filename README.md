# Bookstack Helm Chart

A Helm chart for deploying [BookStack](https://www.bookstackapp.com/) — a simple, self-hosted wiki platform — with MySQL on Kubernetes. Supports optional **Azure AD** (Entra ID) authentication and **SMTP** for email.

## Contents

- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Port-forward only (no host / no Ingress)](#port-forward-only-no-host--no-ingress)
- [Configuration](#configuration)
- [Azure AD (Entra ID) Setup](#azure-ad-entra-id-setup)
- [SMTP Setup](#smtp-setup)
- [SMTP with Azure (Communication Services)](#smtp-with-azure-communication-services)
- [Uninstall](#uninstall)

---

## Prerequisites

- Kubernetes cluster (1.24+)
- Helm 3.x
- **Ingress** (optional): NGINX Ingress Controller and cert-manager with a ClusterIssuer (e.g. `letsencrypt-prod`) if you use Ingress; omit if you use port-forward only
- (Optional) Azure AD app registration for SSO
- (Optional) SMTP server for password reset / notifications

---

## Quick Start

### Install from this repository (local chart)

```bash
helm upgrade --install <release_name> ./charts/bookstack --namespace=<namespace> --create-namespace \
  --set appHost=wiki.example.com \
  --set appKey=base64:VGhpc0lzQW5FeGFtcGxlQXBwS2V5Q2hhbmdlVGhpcyE= \
  --set azuread.enabled=false \
  --set smtp.enabled=false
```

### Install from the Helm repo (published charts)

```bash
helm repo add bookstack-helm https://lumunix.github.io/bookstack-helm/
helm repo update
helm upgrade --install <release_name> bookstack-helm/bookstack --namespace=<namespace> --create-namespace --values values.yaml
```

> **Important:** `appKey` must be set to a valid 32-character key (often base64-encoded). If missing or invalid, the container may fail and the app can show *"An unknown error occurred"*. Generate one with:  
> `php -r "echo 'base64:'.base64_encode(random_bytes(32));"`

---

## Port-forward only (no host / no Ingress)

You can run BookStack **without a host or Ingress** and access it only via `kubectl port-forward`:

1. Disable the Ingress and omit `appHost`.
2. Optionally set `appUrl` to the URL you use in the browser (so links and redirects work).

**Install example:**

```bash
helm upgrade --install my-wiki ./charts/bookstack --namespace=bookstack --create-namespace \
  --set appKey=base64:$(openssl rand -base64 32) \
  --set ingress.enabled=false \
  --set azuread.enabled=false \
  --set smtp.enabled=false
```

**Access:**

```bash
kubectl port-forward -n bookstack svc/my-wiki-bookstack 8080:8080
```

Then open **http://localhost:8080**. BookStack’s `APP_URL` defaults to `http://localhost:8080` when neither `appHost` nor `appUrl` is set, so redirects and links work. If you use a different local port, set `appUrl` accordingly, e.g. `--set appUrl=http://localhost:9000`.

**Switching to Ingress later:** You can enable a real host and Ingress anytime. Run `helm upgrade` with `ingress.enabled=true`, set `appHost` to your domain (and unset `appUrl` if you want the app to use `https://` + appHost). Add the new redirect URI in the Azure AD app registration (e.g. `https://wiki.yourdomain.com/oidc/callback`); you can keep the localhost URI or remove it. No need to reinstall.

---

## Configuration

| Parameter           | Description                                      | Default / Required   |
|--------------------|--------------------------------------------------|----------------------|
| `appHost`          | Public hostname (e.g. `wiki.example.com`). Required only when Ingress is enabled. | Optional |
| `appUrl`           | Full URL for links/redirects (e.g. `http://localhost:8080`). Overrides `https://` + appHost when set. | Optional; defaults to `http://localhost:8080` when no appHost |
| `appKey`           | Application encryption key (32 chars, often `base64:...`) | **Required** |
| `ingress.enabled`  | Create Ingress resource. Set `false` for port-forward only. | `true` |
| `allowHttp`        | Allow HTTP on Ingress (e.g. for redirect to HTTPS) | `false` |
| `storageType`      | BookStack storage driver (e.g. `local_secure`)   | `local_secure` |
| `azuread.*`        | Azure AD (Entra ID) SSO settings                 | See [Azure AD Setup](#azure-ad-entra-id-setup) |
| `smtp.*`           | SMTP settings for mail                            | See [SMTP Setup](#smtp-setup) |

Example custom `values.yaml` (with Ingress):

```yaml
appHost: wiki.example.com
appKey: base64:YourBase64Encoded32ByteKeyHere
ingress:
  enabled: true
allowHttp: false
storageType: local_secure

azuread:
  enabled: false
  # When enabled, set tenantId, appId, appSecret (see Azure AD section)

smtp:
  enabled: false
  # When enabled, set host, port, username, password, fromAddress, fromName
```

The chart deploys:

- **BookStack** (solidnerd/bookstack:25.12) with init containers for volume setup and DB wait
- **MySQL 5.7** for the database
- **PersistentVolumeClaims** for MySQL data, uploads, and storage
- **Ingress** (only when `ingress.enabled` is true; nginx, TLS via cert-manager)

---

## Azure AD (Entra ID) Setup

This chart can configure BookStack to use **Azure AD** (Microsoft Entra ID) for single sign-on. When enabled, the deployment gets `AZURE_TENANT`, `AZURE_APP_ID`, `AZURE_APP_SECRET`, and optional auto-register/confirm settings.

### 1. Register an application in Azure

1. Open [Azure Portal](https://portal.azure.com) → **Microsoft Entra ID** (Azure Active Directory).
2. Go to **App registrations** → **New registration**.
3. Set a name (e.g. "BookStack Wiki"), choose supported account types, and set **Redirect URI**:
   - Type: **Web**
   - **With Ingress (public host):** `https://<your-appHost>/oidc/callback`  
     Example: `https://wiki.example.com/oidc/callback`
   - **Port-forward only:** `http://localhost:8080/oidc/callback`  
     (use the port you forward to, e.g. `http://localhost:9000/oidc/callback` if you use port 9000). The redirect URI is required for Azure AD login in both cases; it must match the URL where users access BookStack.
4. Register and note:
   - **Application (client) ID** → use as `azuread.appId`
   - **Directory (tenant) ID** → use as `azuread.tenantId`

### 2. Create a client secret

1. In the app registration: **Certificates & secrets** → **New client secret**.
2. Add description, choose expiry, save.
3. Copy the **Value** (not the Secret ID) immediately — this is `azuread.appSecret`.

### 3. Configure the API permissions

1. **API permissions** → **Add a permission**.
2. **Microsoft Graph** → **Delegated**.
3. Add at least:
   - `openid`
   - `User.Read`
   - `email` (if you want email for auto-register/confirm)

### 4. Set Helm values

In `values.yaml` or via `--set`:

```yaml
azuread:
  enabled: true
  tenantId: "<your-tenant-id>"      # Directory (tenant) ID
  appId: "<your-application-id>"   # Application (client) ID
  appSecret: "<your-client-secret>" # Client secret value
```

The chart sets `AZURE_AUTO_REGISTER` and `AZURE_AUTO_CONFIRM_EMAIL` to `"true"` so users signing in via Azure AD are created and confirmed automatically. No extra env vars are required for that.

### 5. Security note

**Do not** commit `appSecret` (or `appKey`) to git. Prefer:

- Kubernetes **Secrets** and inject via a values file or `--set-file`, or
- A secret manager (e.g. External Secrets, Sealed Secrets) that injects into the deployment.

Example with existing secret (you would need to adjust the deployment template to use `valueFrom.secretKeyRef` for `AZURE_APP_SECRET` if you add that support; the chart currently expects the secret in values).

---

## SMTP Setup

To enable email (password reset, notifications), set SMTP in `values.yaml`:

```yaml
smtp:
  enabled: true
  host: smtp.example.com
  port: "587"           # or "25", "465" depending on provider
  username: your-smtp-user
  password: your-smtp-password
  fromAddress: noreply@example.com
  fromName: "My Wiki"
```

The chart sets `MAIL_ENCRYPTION` to `tls`. For other encryption or no TLS, the template would need to be extended (currently fixed in the chart).

---

## SMTP with Azure (Communication Services)

You can use **Azure Communication Services** (Email) as the SMTP provider. This uses Microsoft’s SMTP relay at `smtp.azurecomm.net` with Entra ID (Azure AD) authentication. It works well when you already use Azure AD for BookStack login.

### 1. Create Azure resources

1. In [Azure Portal](https://portal.azure.com), create an **Azure Communication Services** resource (if you don’t have one).
2. Create an **Azure Communication Email** resource and [provision a domain](https://learn.microsoft.com/en-us/azure/communication-services/quickstarts/email/create-email-communication-resource) (managed domain or [custom verified domain](https://learn.microsoft.com/en-us/azure/communication-services/quickstarts/email/add-azure-managed-domains)).
3. [Connect the Email resource](https://learn.microsoft.com/en-us/azure/communication-services/quickstarts/email/connect-email-communication-resource) to your Communication Services resource.

### 2. Create SMTP credentials (Entra ID app + role)

1. **Register an app** in **Microsoft Entra ID** → **App registrations** → **New registration**. Note the **Application (client) ID**.
2. **Create a client secret**: in the app → **Certificates & secrets** → **New client secret**. Copy the **Value** immediately — this is the SMTP password.
3. **Assign a role** to the app on the **Communication Services** resource:
   - Open the Communication Services resource → **Access control (IAM)** → **Add** → **Add role assignment**.
   - Role: **Communication and Email Service Owner** (or a [custom role](https://learn.microsoft.com/en-us/azure/communication-services/quickstarts/email/send-email-smtp/smtp-authentication) with `Microsoft.Communication/CommunicationServices/Read,Write` and `Microsoft.Communication/EmailServices/Write`).
   - Members: select your Entra application (service principal) → **Review + assign**.
4. **Create SMTP username** in the Communication Services resource:
   - Open the resource → **SMTP Usernames** → **+ Add SMTP Username**.
   - Select your Microsoft Entra application. Set the username (e.g. `bookstack-smtp` or an email-style address from your verified domain). Save.

### 3. Get the “From” address

Use a sender address from your **provisioned/verified domain**, for example:

- Managed domain: `donotreply@<your-resource-id>.azurecomm.net` (see **Provision domains** in the Email resource).
- Custom domain: `noreply@yourdomain.com` after adding and verifying the domain.

### 4. Helm values for Azure SMTP

Use these values so BookStack sends mail via Azure Communication Services:

```yaml
smtp:
  enabled: true
  host: smtp.azurecomm.net
  port: "587"                                    # Use 587 (TLS); 25 is also supported
  username: "<your-smtp-username>"               # The SMTP Username you created in step 2.4
  password: "<entra-app-client-secret>"          # The client secret from step 2.2 (keep secret!)
  fromAddress: "donotreply@xxxxxxxx.azurecomm.net"  # Or your custom verified domain address
  fromName: "My Wiki"
```

- **Username**: The SMTP Username you added under **SMTP Usernames** (not the Entra Application ID).
- **Password**: The Entra application **client secret** value.
- **fromAddress**: Must be from your Email resource’s provisioned or verified domain.

The chart uses `MAIL_ENCRYPTION: tls`, which matches Azure’s requirement for TLS on port 587.

**References:** [Send email with SMTP - Azure Communication Services](https://learn.microsoft.com/en-us/azure/communication-services/quickstarts/email/send-email-smtp/send-email-smtp), [SMTP authentication](https://learn.microsoft.com/en-us/azure/communication-services/quickstarts/email/send-email-smtp/smtp-authentication).

---

## Uninstall

```bash
helm uninstall <release_name> --namespace=<namespace>
```

Note: PVCs are not removed by default; delete them manually if you want to free storage.

---

## License

See [LICENSE](LICENSE) in this repository.
