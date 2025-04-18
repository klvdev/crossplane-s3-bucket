# 📘 Guia Completo: Provisionando Bucket S3 com Crossplane (sem Kratix)

Este guia descreve como criar um bucket S3 usando **Crossplane** com o **provider modular `provider-aws-s3` da Upbound**, partindo de um ambiente limpo com **KinD**.

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
apiVersion: aws.upbound.io/v1beta1
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

## 🪣 6. Criar Bucket S3

### Arquivo: `resources/s3-bucket.yaml`

```yaml
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: klebernetes-crossplane-bucket
spec:
  forProvider:
    region: us-east-1
  providerConfigRef:
    name: default
```

> ⚠️ Certifique-se de usar um nome **único global** (ex: prefixar com seu nome ou projeto)

```bash
kubectl apply -f resources/s3-bucket.yaml
```

---

## ✅ 7. Verificação final

```bash
kubectl get bucket.s3.aws.upbound.io
kubectl describe bucket klebernetes-crossplane-bucket
```

E acesse o [Console da AWS S3](https://s3.console.aws.amazon.com/s3) para confirmar a criação.

---

## 🧹 (Opcional) Cleanup

```bash
kubectl delete -f resources/s3-bucket.yaml
kubectl delete -f provider/providerconfig.yaml
kubectl delete secret aws-secret -n crossplane-system
kubectl delete -f provider/provider-aws-s3.yaml
kind delete cluster --name platform-cluster
```

---

Pronto! Agora você tem um repositório GitOps limpo para provisionar buckets S3 com Crossplane modular da Upbound.

Se quiser expandir para versionamento, CORS, tags ou site estático, posso te ajudar com os patches adicionais. ✅

