# Vault-Secrets-Operator-EKS

## Tutorial on setting up the Vault Secret Operator with an External Vault.

This is modified version of the tutorial [Deploy the Kubernetes Vault Secrets Operator with HCP Vault Dedicated]([Deploy the Kubernetes Vault Secrets Operator with HCP Vault Dedicated](https://developer.hashicorp.com/vault/tutorials/cloud-ops/kubernetes-vso-hcp-vault). It also uses some commands from [gautambaghel
/vault-secrets-operator-demo](https://github.com/gautambaghel/vault-secrets-operator-demo/tree/main).

** Note your EKS cluster and Vault Server must have network connectivity and be allowed to speak to each other

The example vault server address used in this tutorial **https://external-vault.example.net:8200**
This tutorial was tested with Vault Enterprise 15.5 and Kubernetes 1.29 on EKS
One other note the Vault namespace used is operator. If you not using names spaces you will set the namesspace to root in the configuration

1. Enable the KV secret engines

```shell
vault secrets enable -version=2 -path=app1 kv
```

```shell
vault secrets enable -version=2 -path=app2 kv
```

2. Create a secret at the path app1/secret1/webuser with a username and password.

```shell
vault kv put app1/secret1/webuser username='web-user' password='web-pass'
```

3. Create a secret at the path app2/secret1/dbuser with a username and password. 

```shell
vault kv put app2/secret1/dbuser username='db-user' password='db-pass'
```

4. Create a Kubernetes service account named vault-auth with a service account token. This token is used by Vault to authenticate with the Kubernetes API.

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

5. Create a role for the vault-auth service account to permit access to the Kubernetes API.

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

6. Retrieve the vault-auth secret and store it as an environment variable.

```
VAULTAUTH_SECRET=$(kubectl get secret vault-auth -o json | jq -r '.data') \
    && echo $VAULTAUTH_SECRET
```
7. Decode the ca.crt certificate and store it as an environment variable.

```
K8S_CA_CRT=$(echo $VAULTAUTH_SECRET | jq -r '."ca.crt"' | base64 -d)
```

8. Decode the token and store it as an environment variable.

```
VAULTAUTH_TOKEN=$(echo $VAULTAUTH_SECRET | jq -r '.token' | base64 -d)
```

9. Set the EKS cluster URL
```shell
export K8S_URL=$(kubectl config view --raw --minify --flatten \
   -o jsonpath='{.clusters[].cluster.server}')
```

10. Enable the Kubernetes auth method.

```shell
vault auth enable kubernetes
```

11. Configure the Kubernetes auth method to connect to the Kubernetes API using the vault-auth service account token.

```
vault write auth/kubernetes/config \
 token_reviewer_jwt=$VAULTAUTH_TOKEN \
 kubernetes_host=$K8S_URL \
 kubernetes_ca_cert=$K8S_CA_CRT
```

```shell
vault policy write apps-read - << EOF
path "app1/data/secret1/webuser" {
  capabilities = ["read"]
}

path "app2/data/secret1/dbuser" {
  capabilities = ["read"]
}
EOF
```

12. Create a role for the Kubernetes auth method and include the apps Vault policy.

```shell
vault write auth/kubernetes/role/apps \
bound_service_account_names=vault-auth \
bound_service_account_namespaces=default \
policies=default,apps-read \
ttl=1h
```
13. Install and update the HashiCorp Helm repository

```
helm repo add hashicorp https://helm.releases.hashicorp.com \
    && helm repo update
```

14. Install the Vault Secrets Operator.

```
helm install vault-secrets-operator hashicorp/vault-secrets-operator \
    --namespace vault-secrets-operator \
    --create-namespace \
```

15. Create a connection to Vault Dedicated.

```shell
kubectl create -f - <<EOF
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  namespace: default
  name: vault-connection
spec:
  # address to the Vault server.
  address: $VAULT_ADDR
---
EOF
```

**Make sure to set address to your vault server address that the EKS cluster can speak to.**

16. Verify the configuration.

```shell
kubectl describe vaultconnection.secrets.hashicorp.com/vault-connection
```

17. Configure authentication for the Vault Secrets Operator controller.

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
    role: apps
    serviceAccount: vault-auth
  namespace: "admin" #Vault Dedicated only
---
EOF
```

18. Configure the Vault Secrets Operator to read from the secret KV v2 mount at the exampleapp/config path.

```shell
kubectl create -f - <<EOF
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: vault-app1-webuser
spec:
  vaultAuthRef: vault-auth
  namespace: "admin" #Vault Dedicated only
  mount: app1
  type: kv-v2
  path:  secret1/webuser
# version: 2
  refreshAfter: 300s
  destination:
    create: true
    name: vso-handled-app1
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: vault-app2-dbuser
spec:
  vaultAuthRef: vault-auth
  namespace: "admin" #Vault Dedicated only
  mount: app2
  type: kv-v2
  path:  secret1/dbuser
# version: 2
  refreshAfter: 300s
  destination:
    create: true
    name: vso-handled-app2
EOF
```

19. Verify the Kubernetes secret was created.

```shell
kubectl get secrets
```

20. Read the Kubernetes secret value and decode the base64 encoded strings.

```
kubectl get secret vso-handled-app1 -o json | jq ".data | map_values(@base64d)"
```

```
kubectl get secret vso-handled-app1 -o json | jq ".data | map_values(@base64d)"
```