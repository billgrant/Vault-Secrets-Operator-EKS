# Vault-Secrets-Operator-EKS

## Tutorial on setting up the Vault Secret Operator with an External Vault.

This is modified version of the tutorial [Deploy the Kubernetes Vault Secrets Operator with HCP Vault Dedicated]([Deploy the Kubernetes Vault Secrets Operator with HCP Vault Dedicated](https://developer.hashicorp.com/vault/tutorials/cloud-ops/kubernetes-vso-hcp-vault). It also uses some commands from [gautambaghel
/vault-secrets-operator-demo](https://github.com/gautambaghel/vault-secrets-operator-demo/tree/main).

** Note your EKS cluster and Vault Server must have network connectivity and be allowed to speak to each other

The example vault server address used in this tutorial **https://external-vault.example.net:8200**
This tutorial was tested with Vault Enterprise 15.5 and Kubernetes 1.29 on EKS
One other note the Vault namespace used is operator. If you not using names spaces you will set the namesspace to root in the configuration

1. Enable the KV secret engine

```shell
vault secrets enable -version=2 -path=secret kv
```
2. Create a secret at path secret/exampleapp/config with a username and password.

```shell
vault kv put secret/exampleapp/config username='static-user' password='static-pass'
```

3. Create a Kubernetes service account named vault-auth with a service account token. This token is used by Vault to authenticate with the Kubernetes API.

```shell
kubectl create -f - <<EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-auth
---
apiVersion: v1
kind: Secret
metadata:
  name: vault-auth
  annotations:
    kubernetes.io/service-account.name: vault-auth
type: kubernetes.io/service-account-token
---
EOF
```

4. Create a role for the vault-auth service account to permit access to the Kubernetes API.

```shell
kubectl create -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: role-tokenreview-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: vault-auth
    namespace: default
EOF
```

5. Retrieve the vault-auth secret and store it as an environment variable.

```
VAULTAUTH_SECRET=$(kubectl get secret vault-auth -o json | jq -r '.data') \
    && echo $VAULTAUTH_SECRET
```
6. Decode the ca.crt certificate and store it as an environment variable.

```
K8S_CA_CRT=$(echo $VAULTAUTH_SECRET | jq -r '."ca.crt"' | base64 -d)
```

7. Decode the token and store it as an environment variable.

```
VAULTAUTH_TOKEN=$(echo $VAULTAUTH_SECRET | jq -r '.token' | base64 -d)
```

8. Set the EKS cluster URL
```shell
export K8S_URL=$(kubectl config view --raw --minify --flatten \
   -o jsonpath='{.clusters[].cluster.server}')
```

9. Enable the Kubernetes auth method.

```shell
vault auth enable kubernetes
```

10. Configure the Kubernetes auth method to connect to the Kubernetes API using the vault-auth service account token.

```
vault write auth/kubernetes/config \
 token_reviewer_jwt=$VAULTAUTH_TOKEN \
 kubernetes_host=$K8S_URL \
 kubernetes_ca_cert=$K8S_CA_CRT
```

```shell
vault policy write exampleapp-read - << EOF
path "secret/data/exampleapp/config" {
  capabilities = ["read"]
}
EOF
```

11. Create a role for the Kubernetes auth method and include the exampleapp-read Vault policy.

```shell
vault write auth/kubernetes/role/exampleapp \
bound_service_account_names=vault-auth \
bound_service_account_namespaces=default \
policies=default,exampleapp-read \
ttl=1h
```
12. Install and update the HashiCorp Helm repository

```
helm repo add hashicorp https://helm.releases.hashicorp.com \
    && helm repo update
```

13. Install the Vault Secrets Operator.

```
helm install vault-secrets-operator hashicorp/vault-secrets-operator \
    --namespace vault-secrets-operator \
    --create-namespace \
```

14. Create a connection to Vault Dedicated.

```
kubectl create -f - <<EOF
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  namespace: default
  name: vault-connection
spec:
  # address to the Vault server.
  address: https://external-vault.example.net:8200
---
EOF
```

**Make sure to set address to your vault server address that the EKS cluster can speak to.**

15. Verify the configuration.

```shell
kubectl describe vaultconnection.secrets.hashicorp.com/vault-connection
```

16. Configure authentication for the Vault Secrets Operator controller.

```shell
kubectl create -f - <<EOF
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: vault-auth
spec:
  vaultConnectionRef: vault-connection
  method: kubernetes
  mount: kubernetes
  kubernetes:
    role: exampleapp
    serviceAccount: vault-auth
  namespace: "operator" #Vault Dedicated (enterprise)
---
EOF
```

17. Configure the Vault Secrets Operator to read from the secret KV v2 mount at the exampleapp/config path.

```shell
kubectl create -f - <<EOF
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: vault-static-secret
spec:
  vaultAuthRef: vault-auth
  namespace: "operator" #Vault Dedicated (Enterprise)
  mount: secret
  type: kv-v2
  path:  exampleapp/config
# version: 2
  refreshAfter: 300s
  destination:
    create: true
    name: vso-handled
---
EOF
```

18. Verify the Kubernetes secret was created.

```shell
kubectl get secrets
```

19. Read the Kubernetes secret value and decode the base64 encoded strings.

```
kubectl get secret vso-handled -o json | jq ".data | map_values(@base64d)"
```

20. Change the secret value

```shell
vault kv put secret/exampleapp/config username='static-user-changed' password='static-pass-changed'
```

21. Wait a few seconds and check again to see the new values

```
kubectl get secret vso-handled -o json | jq ".data | map_values(@base64d)"
```
