## Cred
```sh
username: S11ahmed.wft
password: "qVX2nvxhCpwp@#$"
```

## Install External Secrets Operator
```sh
helm repo add external-secrets https://charts.external-secrets.io
helm upgrade --install external-secrets external-secrets/external-secrets \
  --namespace external-secrets --create-namespace
```

## Install Reloader
```sh
helm repo add stakater https://stakater.github.io/stakater-charts
helm upgrade --install reloader stakater/reloader --namespace reloader --create-namespace
```


## ESO
```sh
# Add the external-secrets helm repository
helm repo add external-secrets https://charts.external-secrets.io

# Update helm repositories
helm repo update

# Install ESO version 0.17
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --version 0.17.0
```

## Create a secret
```sh
kubectl create secret generic vault-auth \
  --from-literal=username="existing-user-name" \
  --from-literal=password="existing-password" \
  --namespace=external-secrets 

kubectl create secret generic vault-auth \
  --from-literal=username="S11ahmed.wft" \
  --from-literal=password="qVX2nvxhCpwp@#$" \
  --namespace=external-secrets
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vault-auth
  namespace: external-secrets
type: Opaque
stringData:
  username: "existing-user-name"
  password: "existing-password"
```