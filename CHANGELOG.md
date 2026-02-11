# Changelog

All notable changes to this project are documented in this file.

## [1.3.5] - 2026-02-11
### Title
Add Ingress OIDC sample values file

### Description
- Added `charts/bookstack/values-test-oidc.yaml` for public host/Ingress deployments using OIDC as primary authentication (Azure-only login) with SMTP disabled.
- Updated `README.md` and `CONFIGURATION.md` to reference both Ingress and port-forward OIDC sample values files.

## [1.3.4] - 2026-02-11
### Title
Add OIDC primary authentication support

### Description
- Added `oidc` chart configuration and BookStack deployment env wiring for primary OIDC authentication (`AUTH_METHOD=oidc`), including auto-initiate and issuer discovery options.
- Added a guard to prevent enabling both `oidc.enabled` and `azuread.enabled` simultaneously.
- Added `charts/bookstack/values-test-portforward-oidc.yaml` and updated `README.md`/`CONFIGURATION.md` with OIDC Azure-only login setup and secret key guidance.

## [1.3.3] - 2026-02-11
### Title
Changelog automation and values test updates

### Description
- Added release changelog integration in GitHub Actions so release title and notes are sourced from this file for each chart version.
- Added `service.port` to both sample values test files to keep examples aligned with the configurable service port feature.
- Added initial changelog history entries for recent chart releases.

## [1.3.2] - 2026-02-11
### Title
Configurable service port

### Description
- Added configurable `service.port` support in chart templates for Service and Ingress backend routing.
- Documented the new parameter in `README.md` and `CONFIGURATION.md`.

## [1.3.1] - 2026-02-11
### Title
Port-forward Azure AD sample and docs refresh

### Description
- Added `values-test-portforward-azuread.yaml` with SMTP disabled and Kubernetes Secret based Azure AD setup.
- Updated README examples to include the new port-forward Azure AD values test file.
