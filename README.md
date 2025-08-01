# 📘 Guia Completo: Provisionando Bucket S3 com Crossplane + Argo CD

Este guia descreve como criar um bucket S3 usando **Crossplane** com o **provider modular `provider-aws-s3` da Upbound**, e gerenciar via **Argo CD** com GitOps.

---

## 📁 Estrutura de diretório sugerida (GitOps)

```
crossplane-s3-bucket/
├── provider/
│   ├── provider-aws-s3.yaml
│   └── providerconfig.yaml
├── secrets/
│   └── aws-credentials.txt
└── resources/
    └── s3-bucket.yaml
```

---

## 🧱 1. Criar cluster com KinD

```bash
kind create cluster --name platform-cluster
kubectl config use-context kind-platform-cluster
kubectl get nodes
```

---

## 📦 2. Instalar Crossplane via Helm

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace

kubectl get pods -n crossplane-system
```

---

## 🔐 3. Criar Secret com credenciais AWS

### Arquivo: `secrets/aws-credentials.txt`

```ini
[default]
aws_access_key_id=SUA_ACCESS_KEY_ID
aws_secret_access_key=SUA_SECRET_ACCESS_KEY
```

### Comando:

```bash
kubectl create secret generic aws-secret \
  -n crossplane-system \
  --from-file=creds=secrets/aws-credentials.txt
```

---

## 🔌 4. Instalar o provider S3

### Arquivo: `provider/provider-aws-s3.yaml`

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-s3
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v0.38.0
```

```bash
kubectl apply -f provider/provider-aws-s3.yaml
```

Aguarde o download e verificação:

```bash
kubectl get provider.pkg.crossplane.io
kubectl get crds | grep providerconfig
```

---

## ⚙️ 5. Criar ProviderConfig

### Arquivo: `provider/providerconfig.yaml`

```yaml
apiVersion: aws.s3.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-secret
      key: creds
```

```bash
kubectl apply -f provider/providerconfig.yaml
```

---

## 🪣 6. Criar Bucket S3 com Tag

### Arquivo: `resources/s3-bucket.yaml`

```yaml
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: klebernetes-crossplane-bucket
spec:
  forProvider:
    region: us-east-1
    tags:
      - key: environment
        value: devops-gitops
  providerConfigRef:
    name: default
```

> ⚠️ O nome do bucket deve ser único globalmente

```bash
kubectl apply -f resources/s3-bucket.yaml
```

---

## 🧭 7. Instalar Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd --watch
```

Expose temporário:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Acesse: https://localhost:8080

Recuperar senha:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo
```

---

## 🔗 8. Criar Application do Argo CD

### Arquivo: `argo-application.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-s3
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:klvdev/crossplane-s3-bucket.git
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

```bash
kubectl apply -f argo-application.yaml
```

---

## ✅ 9. Validar no Argo CD

- Acesse https://localhost:8080
- Login com `admin` + senha
- Veja o app `crossplane-s3`
- Clique em **Sync** se estiver `OutOfSync`

---

## 🔍 10. Verificar no cluster e na AWS

```bash
kubectl get bucket.s3.aws.upbound.io
kubectl describe bucket klebernetes-crossplane-bucket
```

Verifique as **Tags** no console do S3: https://s3.console.aws.amazon.com/s3

---

## 🧹 (Opcional) Cleanup

```bash
kubectl delete -f resources/s3-bucket.yaml
kubectl delete -f provider/providerconfig.yaml
kubectl delete secret aws-secret -n crossplane-system
kubectl delete -f provider/provider-aws-s3.yaml
kubectl delete ns argocd
kind delete cluster --name platform-cluster
```

---

Pronto! Agora você tem uma estrutura GitOps com Crossplane e Argo CD para provisionar buckets S3 na AWS com rastreabilidade total e sincronização automática. 

Se quiser o mesmo fluxo para Lambda + API Gateway, é só chamar. ✅

