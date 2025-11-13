# 🚀 Hello mTLS World on AWS EKS

This repository provisions an **Amazon EKS cluster** using **Terraform** and deploys a simple
**Kubernetes NGINX application** secured with **mutual TLS (mTLS)** using **self-signed certificates**.

---

## 📘 Overview

| Layer | Tooling | Description |
|-------|----------|--------------|
| Infrastructure | Terraform + AWS | Creates VPC, Subnets, NAT, and EKS Cluster |
| Security | TLS + OpenSSL + Secrets | Generates self-signed CA, server & client certs |
| Application | Kubernetes + NGINX | Simple HTTPS “Hello World” service enforcing client-certificate verification |

---

## 🧱 Prerequisites

- [Terraform ≥ 1.5.0](https://developer.hashicorp.com/terraform/downloads)
- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- Valid AWS credentials (`aws configure`)
- macOS/Linux terminal (or WSL on Windows)

---

## 🗂 Directory Layout

```
.
├── eks/               # Terraform EKS-only cluster definition
│   ├── versions.tf
│   ├── providers.tf
│   ├── variables.tf
│   ├── vpc.tf
│   ├── eks.tf
│   └── outputs.tf
│
├── app/               # Kubernetes Hello mTLS World deployment
│   ├── certs/         # OpenSSL scripts + generated certs
│   └── hello-mtls.yaml
│
└── README.md
```

---

## ⚙️ 1. Provision the EKS Cluster

```bash
cd eks
terraform init
terraform plan
terraform apply -auto-approve
```

Once Terraform finishes, update your local kubeconfig:

```bash
aws eks update-kubeconfig   --region us-east-1   --name eks-demo-cluster
```

Verify access:
```bash
kubectl get nodes
```

---

## 🔐 2. Generate Self-Signed Certificates

Use OpenSSL to create a CA, a server cert for the service (`hello.default.svc.cluster.local`),
and a client cert for testing.

```bash
mkdir -p app/certs && cd app/certs

# Create CA
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650   -subj "/CN=mtls-demo-ca" -out ca.crt

# Server key & cert
cat > server.cnf <<'EOF'
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = req_ext
distinguished_name = dn
[ dn ]
CN = hello.app-mtls.svc.cluster.local
[ req_ext ]
subjectAltName = @alt_names
[ alt_names ]
DNS.1 = hello
DNS.2 = hello.app-mtls
DNS.3 = hello.app-mtls.svc
DNS.4 = hello.app-mtls.svc.cluster.local
EOF

openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -config server.cnf
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial   -out server.crt -days 825 -sha256 -extensions req_ext -extfile server.cnf

# Client cert
openssl genrsa -out client.key 2048
openssl req -new -key client.key -subj "/CN=test-client" -out client.csr
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial   -out client.crt -days 825 -sha256
```

---

## ☸️ 3. Deploy the Hello mTLS World App

Create Kubernetes namespace, secrets and the NGINX deployment.

```bash
kubectl create ns app-mtls

kubectl -n app-mtls create secret tls hello-tls   --cert=server.crt --key=server.key

kubectl -n app-mtls create secret generic hello-ca   --from-file=ca.crt=ca.crt
```

Apply the deployment manifest:
```bash
kubectl apply -f app/hello-mtls.yaml
kubectl get pods -l app=hello -n app-mtls
```

---

## 🧪 4. Test mTLS Inside the Cluster

Run a temporary curl pod:
```bash
kubectl -n app-mtls run curl --image=curlimages/curl:8.7.1 -it --restart=Never -- /bin/sh
```

Inside the pod, paste the certs (`ca.crt`, `client.crt`, `client.key`) under `/tmp/`.

### ❌ Without client cert
```bash
curl -vk --cacert /tmp/ca.crt https://hello:8443/
```
→ **Fails** (NGINX requires client auth)

### ✅ With client cert
```bash
curl -sk --cacert /tmp/ca.crt   --cert /tmp/client.crt --key /tmp/client.key   https://hello:8443/
```
→ Returns `hello, mTLS world`

---

## 🌍 5. (Optional) Test from Local Machine

Port-forward the service:
```bash
kubectl -n app-mtls port-forward svc/hello 8443:8443
```

Then, in another terminal:
```bash
curl -sk --cacert app/certs/ca.crt   --cert app/certs/client.crt --key app/certs/client.key   --resolve "hello.app-mtls.svc.cluster.local:8443:127.0.0.1"   https://hello.app-mtls.svc.cluster.local:8443/
```

---

## 🧹 Cleanup

```bash
kubectl -n app-mtls delete all -l app=hello
terraform destroy -auto-approve
```

---

## 🧾 Notes

- Adjust certificate SANs if you deploy in a different namespace or domain.
- For production, integrate with AWS ACM or cert-manager instead of self-signed certs.
- Limit public API endpoint access; use IAM roles and security groups for fine-grained control.

---

## 🧑‍💻 Author

**DevOps / Cloud Engineer:** _Satish Basani_  
Infra-as-Code | Kubernetes | Terraform | mTLS | AWS EKS
