# Bookstack Helm Chart

## Local Installation

```sh
helm upgrade --install <release_name> . --namespace=<namespace> --create-namespace \
    --set appHost=<www.yourdomain.com> \
    --set appKey=base64:VGhpc0lzQW5FeGFtcGxlQXBwS2V5Q2hhbmdlVGhpcyE= \
    --set azuread.enabled=false \
    --set smtp.enabled=false
```

> [!IMPORTANT]  
> `appKey` must be set to 32-character key. Container would end up in error if not set. App would show `An unknown error occurred` if set to blank.
`

## Installation from Repository

```sh
helm repo add lumunix https://lumunix.github.io/helm-repo/
helm repo update
helm upgrade --install <release_name> lumunix/bookstack --namespace=<namespace> --values values.yaml
```
