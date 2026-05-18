# k8-secrets-orchestrator
## Table of Contents
- [Abstract](#abstract)
- [Security Disclaimer](#security-disclaimer)
- [Repository Structure](#repository-structure)
- [Installation](#installation)
  - [1. Deploy Vault](#1-deploy-vault-on-cluster)
  - [2. Initialize and unseal Vault](#2-initialize-and-unseal-vault)
  - [3. Enable KV engine and put some secrets](#3-enable-kv-engine-and-put-some-secrets)
  - [4. Create Kubernetes auth for Vault](#4-create-kubernetes-auth-for-vault)
  - [5. Create Vault policy and role for ESO](#5-create-vault-policy-and-role-for-eso)
  - [6. Install External Secrets Operator](#6-install-external-secrets-operator)
  - [7. Connect ESO to Vault ](#7-connect-eso-to-vault)
  - [8. Create the synced Secret in desired namespace ](#8-create-the-synced-secret-in-desired-namespace)
  - [9. Install Reloader for automatic restarts](#9-install-reloader-for-automatic-restarts)
  - [10. Update Helm chart](#10-update-helm-chart)

## Abstract
There are 10 containers, that require cyclic maintenance. All of them have the same (from logic point of view) goal, but 5 of them are using very similar configuration unlike the rest of them which are operating on completely different variables and settings. With provided solutions, every change regarding credentials is detected and forces a reboot of affected pods. Non-sensitive data is stored in _values.yaml_ but secrets are meant to be stored in Vault.  

## Security Disclaimer
There are currently several workarounds that may impact security in a production environment. The goal of this solution is to demonstrate a more robust and secure approach to secrets and configuration management in small environments, but it is not yet flawless. For example, Vault is currently configured without encrypted connections (tlsDisable = true). Additionally, key rotation policies and proper Vault access controls (including sealing procedures) have not yet been fully defined. Please keep these limitations in mind during implementation.

## Repository Structure
```text
├── .gitignore
├── README.md
├── app
|     ├── Dockerfile
|     ├── app.py
├── clustersecretstore-vault.yaml
├── etl-chart
|     ├── Chart.yaml
|     ├── templates
|     |     ├── Deployment.yaml
|     |     ├── etl-app-externalsecret.yaml
|     ├── values.yaml
├── etl-eso-policy.hcl
├── vault-values.yaml

```

## Installation
### 1. Deploy Vault on cluster
Prepare a Helm repo and a namespace
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

kubectl create namespace vault
```
Create vault configuration
```yaml
#vault-values.yaml
global:
  enabled: true
  tlsDisable: true

injector:
  enabled: false

server:
  standalone:
    enabled: true
    config: |
      ui = true

      listener "tcp" {
        address = "[::]:8200"
        cluster_address = "[::]:8201"
        tls_disable = 1
      }

      storage "file" {
        path = "/vault/data"
      }

  ha:
    enabled: false

  dataStorage:
    enabled: true
    size: 10Gi
    storageClass: local-path

ui:
  enabled: true
```
Install Vault:

```bash
helm install vault hashicorp/vault -n vault -f vault-values.yaml
```
### 2. Initialize and unseal Vault
Initialization:
```bash
kubectl exec -n vault vault-0 -- vault operator init -key-shares=1 -key-threshold=1
```
Output contains:
- Unseal key
- Initial root key

Unseal node:
```bash
kubectl exec -n vault vault-0 -- vault operator unseal <UNSEAL_KEY>
```
Login and verify:
```bash
kubectl exec -it -n vault vault-0 -- sh
vault login <ROOT_KEY>
vault status
exit
```

Verify if Vault is exposed:
```bash
kubectl run curl-test --rm -it --restart=Never --image=curlimages/curl -n external-secrets -- \
  curl -s http://vault.vault.svc:8200/v1/sys/health
```

### 3. Enable KV engine and put some secrets
```bash
kubectl exec -it -n vault vault-0 -- sh
vault login <ROOT_KEY>

vault secrets enable -path=secret kv-v2

vault kv put secret/integration/etl-app \
  db_password='SuperStrongPassword123!' \
  aws_default_region='eu-central-1' \
  http_proxy='http://proxy.internal:3128'

vault kv get secret/integration/etl-app
exit
```

### 4. Create Kubernetes auth for Vault
```bash
kubectl create namespace vault-auth || true
kubectl create serviceaccount vault-auth -n vault-auth || true

kubectl create clusterrolebinding vault-auth-binding \
  --clusterrole=system:auth-delegator \
  --serviceaccount=vault-auth:vault-auth
```

Setup these variables:
```bash
SA_JWT_TOKEN=$(kubectl create token vault-auth -n vault-auth)
K8S_HOST="https://kubernetes.default.svc:443"
SA_CA_CRT=$(kubectl get secret -n vault $(kubectl get sa default -n vault -o jsonpath='{.secrets[0].name}') -o jsonpath="{.data.ca\.crt}" | base64 -d)
```
Configure Vault
```bash
kubectl exec -it -n vault vault-0 -- sh
vault login <ROOT_KEY>

vault auth enable kubernetes

vault write auth/kubernetes/config \
  token_reviewer_jwt="$SA_JWT_TOKEN" \
  kubernetes_host="$K8S_HOST" \
  kubernetes_ca_cert="$SA_CA_CRT"
exit

```

### 5. Create Vault policy and role for ESO

It's advised to create separate namespace for ESO
```bash
kubectl create namespace external-secrets
```

Create policy file _etl-eso-policy.hcl_
```bash
#etl-eso-policy.hcl
path "secret/data/integration/etl-app" {
  capabilities = ["read"]
}

path "secret/metadata/integration/etl-app" {
  capabilities = ["read", "list"]
}

```

Copy created policy and create role:
```bash
kubectl cp etl-eso-policy.hcl vault/$(kubectl get pod -n vault -l app.kubernetes.io/name=vault -o jsonpath='{.items[0].metadata.name}'):/tmp/etl-eso-policy.hcl

kubectl exec -it -n vault vault-0 -- sh
vault login <ROOT_KEY>

vault policy write etl-eso-policy /tmp/etl-eso-policy.hcl

vault write auth/kubernetes/role/etl-eso-role \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=etl-eso-policy \
  ttl=1h
exit
```

### 6. Install External Secrets Operator
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  -n external-secrets
```

### 7. Connect ESO to Vault 
Create _clustersecretstore-vault.yaml_
```bash
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://vault.vault.svc:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "etl-eso-role"
          serviceAccountRef:
            name: "external-secrets"
            namespace: "external-secrets"
```
And apply it:
```bash
kubectl apply -f clustersecretstore-vault.yaml
```

### 8. Create the synced Secret in desired namespace 
For this use case I used namespace _integration_.
Creating namespace and ServiceAccount:
```bash
kubectl create namespace integration
kubectl create serviceaccount etl-sa -n integration 
```
Then create ExternalSecrets which will be updating or creating a K8s Secret from Vault data.

```bash
#etl-app-externalsecret.yaml
{{- $seen := dict }}
{{- range .Values.pods }}
{{- if not (hasKey $seen .secretName) }}
{{- $_ := set $seen .secretName true }}
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: {{ .secretName }}
  namespace: {{ $.Release.Namespace }}
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: {{ .secretName }}
    creationPolicy: Owner
  data:
    - secretKey: secret_key
      remoteRef:
        key: {{ .vaultKey | quote }}
        property: secret_key
{{- end }}
{{- end }}
```
Apply it:
```bash
kubectl apply -f etl-app-externalsecret.yaml
```

### 9. Install Reloader for automatic restarts
```bash
helm repo add stakater https://stakater.github.io/stakater-charts
helm repo update

kubectl create namespace reloader
helm install reloader stakater/reloader -n reloader
```

### 10. Update Helm chart
```bash
#deployment.yaml
# templates/deployment.yaml
{{- range .Values.pods }}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .name }}
  namespace: {{ $.Release.Namespace }}
  labels:
    app: etl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{ .name }}
  template:
    metadata:
      labels:
        app: {{ .name }}
      annotations: 
        reloader.stakater.com/auto: "true"
        reloader.stakater.com/rollout-strategy: "restart"
    spec:
      serviceAccountName: etl-sa
      containers:
      - name: etl-container
        image: 192.168.0.21:30500/etl-app:3.0.0
        imagePullPolicy: Always
        env:
        - name: HTTP_PROXY
          value: {{ $.Values.global.httpProxy | quote }}
        - name: AWS_DEFAULT_REGION
          value: {{ $.Values.global.awsDefault | quote }}
        - name: BKTNAME
          value: {{ .bktName | quote }}
        - name: DBNAME
          value: {{ .dbName | quote }}
        - name: SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: {{ .secretName }}
              key: secret_key
        volumeMounts:
        - name: log-volume
          mountPath: /app/logs
      volumes:
      - name: log-volume
        hostPath:
          path: /tmp/logs/{{ .name }}
---
{{- end }}
```

You also have to create _values.yaml_ to store your variables.

Then deploy configuratrion into desired namespace:
```bash
helm upgrade --install etl-app ./etl-chart -n integration
```



