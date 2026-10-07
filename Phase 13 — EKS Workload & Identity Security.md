## 13.0 Tujuan Phase

Phase 13 membangun security foundation pada workload Kubernetes yang berjalan di EKS/Floci.

Fokus utama phase ini bukan lagi membangun infrastructure AWS, tetapi memastikan workload Kubernetes memiliki:

- Namespace isolation
- ServiceAccount
- Projected ServiceAccount token
- RBAC least privilege
- NetworkPolicy ingress/egress
- Pod security hardening
- IRSA / IAM Roles for Service Accounts
- Validasi workload → AWS API
- Validasi workload tanpa IRSA
- Discovery dan validasi EKS Pod Identity
- Integrasi security workload dengan architecture phase sebelumnya

Secara sederhana, Phase 13 menjawab dua jenis pertanyaan:

1. "Apa yang boleh dilakukan workload terhadap Kubernetes?"
   → RBAC

2. "Apa yang boleh dilakukan workload terhadap AWS?"
   → IRSA / EKS Pod Identity

Sedangkan NetworkPolicy menjawab:

3. "Workload ini boleh berkomunikasi dengan workload mana?"
   → Kubernetes NetworkPolicy

Dan securityContext menjawab:

4. "Dengan privilege Linux seperti apa workload ini boleh berjalan?"
   → Pod/container securityContext


# 13.1 Arsitektur Phase

Architecture Phase 13:

                         AWS / Floci
                              │
             ┌────────────────┴────────────────┐
             │                                 │
           IAM                               EKS
             │                                 │
      ┌──────┴──────┐                    ┌─────┴─────┐
      │             │                    │           │
     IRSA      Pod Identity           RBAC    NetworkPolicy
      │             │                    │           │
      └─────────────┴────────────┬───────┴───────────┘
                                 │
                      Namespace: enterprise-security
                                 │
             ┌───────────────────┼────────────────────┐
             │                   │                    │
        ServiceAccount       Hardened Pod        NetworkPolicy
             │                   │                    │
       Projected Token      Non-root user       Ingress/Egress
             │                   │
             └──────────────┬────┘
                            │
                       Application
                            │
                 ┌──────────┴──────────┐
                 │                     │
          Kubernetes API          AWS API
                 │                     │
               RBAC              IAM / IRSA


Ideal architecture:

```text
User
 │
 ▼
AWS Infrastructure
 │
 ├── IAM
 │    └── IRSA / Pod Identity
 │
 └── EKS
      │
      └── Kubernetes
           │
           ├── Namespace
           ├── ServiceAccount
           ├── RBAC
           ├── NetworkPolicy
           ├── SecurityContext
           └── Workload
```

Phase 12 sebelumnya sebenarnya sudah menguji container supply chain dan ECR.

Namun karena Floci memiliki limitation pada ECR image push/pull dan EKS image pulling dari ECR, Phase 13 tidak menjadikan ECR sebagai dependency runtime.

Idealnya:

```text
Docker Image
     │
     ▼
   ECR
     │
     ▼
   EKS
     │
     ▼
 Secure Workload
```

Sedangkan pada environment Floci:

```text
Docker Image
     │
     ├── ECR
     │     └── FLOCi LIMITATION
     │
     └── Public Image / Local-compatible Image
               │
               ▼
              EKS
```

Dengan demikian kegagalan ECR pada Phase 12 tidak mencemari validasi security control Kubernetes pada Phase 13.

# 13.2 Validation Matrix Phase

|Area|Expected|Kenapa|
|---|---|---|
|EKS cluster access|PASS|Cluster merupakan foundation workload|
|Dedicated namespace|PASS|Membatasi scope workload|
|ServiceAccount|PASS|Identity workload Kubernetes|
|Projected token|PASS|Token diperlukan untuk workload identity|
|RBAC allow|PASS|Workload harus memiliki permission minimum|
|RBAC deny|PASS|Workload tidak boleh memiliki privilege berlebih|
|NetworkPolicy object|PASS|API NetworkPolicy tersedia|
|NetworkPolicy enforcement|PASS / FLOCi LIMITATION|Tergantung CNI/enforcement Floci|
|NetworkPolicy deny|PASS|Traffic tidak diizinkan harus ditolak|
|NetworkPolicy allow|PASS|Traffic legitimate harus tetap berjalan|
|Pod hardening|PASS|Mengurangi Linux privilege|
|OIDC Provider|PASS|Requirement IRSA|
|IRSA trust policy|PASS|IAM harus trust ServiceAccount tertentu|
|IRSA credential injection|PASS / FLOCi LIMITATION|Bergantung native Floci support|
|IRSA STS|PASS / FLOCi LIMITATION|Bergantung credential bridge Floci|
|IRSA AWS authorization|PASS / FLOCi LIMITATION|Membuktikan least privilege AWS|
|Non-IRSA workload|PASS|Tidak boleh otomatis mendapat application IAM role|
|Pod Identity API|PASS / FLOCi LIMITATION|Dicek melalui EKS API|
|Pod Identity association|PASS / FLOCi LIMITATION|Bergantung dukungan Floci|
|Pod Identity credential injection|PASS / FLOCi LIMITATION|Bergantung agent + EKS Auth|
|ECR → EKS runtime|FLOCi LIMITATION|Sudah terbukti pada Phase 12|
|End-to-end workload security|PASS / FLOCi LIMITATION|Gabungan seluruh control|

---

# Step 177. Prepare Phase 13 Workspace

## Tujuan

Membuat workspace untuk manifest, policy, trust policy, dan evidence Phase 13.

```bash
mkdir -p ~/phase13
cd ~/phase13
```

Validasi:

```bash
pwd
```

Expected:

```text
/home/floci/phase13
```

Kemudian:

```bash
echo "STEP 177: WORKSPACE PASS"
```

Expected:

```text
STEP 177: WORKSPACE PASS
```

---

# Step 178. Validate Existing EKS Security Foundation

## Tujuan

Sebelum membuat security control baru, kita validasi dulu cluster yang diwariskan dari Phase 3.

---

## 178.1. Cluster connectivity

```bash
kubectl cluster-info
```

Expected:

```text
Kubernetes control plane ...
```

---

## 178.2. Kubernetes version

```bash
kubectl version
```

Expected:

```text
Client Version: ...
Server Version: ...
```

---

## 178.3. Node status

```bash
kubectl get nodes -o wide
```

Expected:

```text
STATUS
Ready
```

---

## 178.4. Detect CNI

```bash
kubectl get pods \
  -A \
  -o wide | grep -Ei 'cni|calico|cilium|flannel|aws-node'
```

Tujuannya **bukan sekadar mencari pod**, tetapi menentukan siapa yang sebenarnya melakukan network enforcement.

Possible result:

```text
aws-node
```

atau:

```text
calico
```

atau:

```text
cilium
```

atau:

```text
kube-flannel
```

Ini sangat penting untuk interpretasi NetworkPolicy.

---

## 178.5. Detect network policy capability

```bash
kubectl api-resources | grep -i networkpolicy
```

Expected:

```text
networkpolicies
```

Kemudian:

```bash
kubectl get crd | grep -Ei 'networkpolicy|cilium|calico'
```

Ini membantu mengetahui apakah ada policy engine tambahan.

---

## ValidationStep 178

```bash
echo "=== EKS ==="
kubectl cluster-info

echo
echo "=== NODES ==="
kubectl get nodes

echo
echo "=== NETWORK COMPONENTS ==="
kubectl get pods -A -o wide | \
  grep -Ei 'cni|calico|cilium|flannel|aws-node' || true

echo
echo "=== NETWORK POLICY API ==="
kubectl api-resources | grep -i networkpolicy

echo
echo "STEP 178: EKS FOUNDATION VALIDATION COMPLETE"
```

Expected minimum:

```text
Cluster reachable
Node Ready
NetworkPolicy API exists
```

---

# Step 179. Create Dedicated Security Namespaces

## Tujuan

Membangun boundary Kubernetes.

Kita gunakan:

```text
enterprise-security
enterprise-security-untrusted
```

Tujuannya untuk membuktikan namespace isolation.

```sh
kubectl create namespace enterprise-security
kubectl create namespace enterprise-security-untrusted
```

Kalau sudah ada:

```bash
kubectl get namespace \
  enterprise-security \
  enterprise-security-untrusted
```

Expected:

```text
enterprise-security
enterprise-security-untrusted
```

---

## 179.1. Label namespace

```bash
kubectl label namespace enterprise-security \
  security-tier=trusted \
  --overwrite

kubectl label namespace enterprise-security-untrusted \
  security-tier=untrusted \
  --overwrite
```

### Parameter

`label`

Menambahkan Kubernetes metadata.

`--overwrite`

Mengizinkan label yang sudah ada diubah.

Validasi:

```bash
kubectl get namespace \
  enterprise-security \
  enterprise-security-untrusted \
  --show-labels
```

Expected:

```text
enterprise-security ... security-tier=trusted
enterprise-security-untrusted ... security-tier=untrusted
```

---

## Final validation

```bash
kubectl get ns \
  enterprise-security \
  enterprise-security-untrusted
```

Status:

**PASS** jika namespace berhasil dibuat.

---

# Step 180. Create Dedicated ServiceAccounts

## Tujuan

ServiceAccount adalah identity Kubernetes workload.

Kita buat:

```text
enterprise-security
└── workload-sa

enterprise-security-untrusted
└── untrusted-sa
```

```bash
kubectl create serviceaccount \
  workload-sa \
  -n enterprise-security

kubectl create serviceaccount \
  untrusted-sa \
  -n enterprise-security-untrusted
```

Validasi:

```bash
kubectl get serviceaccount \
  -n enterprise-security

kubectl get serviceaccount \
  -n enterprise-security-untrusted
```

Expected:

```text
workload-sa
```

dan:

```text
untrusted-sa
```

---

## 180.1. Inspect token behavior

```bash
kubectl get serviceaccount workload-sa \
  -n enterprise-security \
  -o yaml
```

Kemudian:

```bash
kubectl create token \
  workload-sa \
  -n enterprise-security
```

### Tujuan

Kubernetes modern menggunakan bound ServiceAccount tokens yang bersifat time/audience-bound. AWS juga mendokumentasikan bahwa bound ServiceAccount token digunakan sebagai bagian dari IRSA. ([AWS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/service-accounts.html?utm_source=chatgpt.com "Grant Kubernetes workloads access to AWS using Kubernetes Service Accounts - Amazon EKS"))

Expected:

```text
eyJhbGciOiJSUzI1NiIs...
```

JWT.

Validasi:

```bash
TOKEN=$(kubectl create token \
  workload-sa \
  -n enterprise-security)

test -n "$TOKEN" \
  && echo "SERVICE ACCOUNT TOKEN: PASS"
```

Status:

**PASS**

---

# Step 181. Create Least-Privilege Kubernetes RBAC

## Tujuan

Menguji prinsip:

> Workload tidak otomatis boleh membaca seluruh Kubernetes API.

Kita buat Role yang hanya boleh membaca ConfigMap.

Buat file:

```bash
cat > rbac.yaml <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: workload-config-reader
  namespace: enterprise-security
rules:
- apiGroups: [""]
  resources:
  - configmaps
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: workload-config-reader
  namespace: enterprise-security
subjects:
- kind: ServiceAccount
  name: workload-sa
  namespace: enterprise-security
roleRef:
  kind: Role
  name: workload-config-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

Apply:

```bash
kubectl apply -f rbac.yaml
```

### Penjelasan

`Role`

Permission hanya berlaku di namespace tersebut.

`resources: configmaps`

Workload hanya mengakses ConfigMap.

`verbs`

- `get`
    
- `list`
    
- `watch`
    

Tidak ada:

```text
create
update
delete
```

`RoleBinding`

Menghubungkan Role dengan ServiceAccount.

---

## Validation

```bash
kubectl get role \
  -n enterprise-security

kubectl get rolebinding \
  -n enterprise-security
```

Expected:

```text
workload-config-reader
```

dan:

```text
workload-config-reader
```

Kemudian authorization test:

```bash
kubectl auth can-i \
  get configmaps \
  --as=system:serviceaccount:enterprise-security:workload-sa \
  -n enterprise-security
```

Expected:

```text
yes
```

Test forbidden operation:

```bash
kubectl auth can-i \
  delete pods \
  --as=system:serviceaccount:enterprise-security:workload-sa \
  -n enterprise-security
```

Expected:

```text
no
```

Ini salah satu test paling penting di Phase 13.

---

# Step 182. Validate RBAC Positive and Negative Paths

## Tujuan

Bukan hanya membuktikan permission ada.

Kita juga harus membuktikan permission **tidak berlebihan**.

---

## 182.1. Allowed

```bash
kubectl auth can-i \
  get configmaps \
  --as=system:serviceaccount:enterprise-security:workload-sa \
  -n enterprise-security
```

Expected:

```text
yes
```

---

## 182.2. Forbidden pod deletion

```bash
kubectl auth can-i \
  delete pods \
  --as=system:serviceaccount:enterprise-security:workload-sa \
  -n enterprise-security
```

Expected:

```text
no
```

---

## 182.3. Forbidden secrets

```bash
kubectl auth can-i \
  get secrets \
  --as=system:serviceaccount:enterprise-security:workload-sa \
  -n enterprise-security
```

Expected:

```text
no
```

---

## 182.4. Forbidden nodes

```bash
kubectl auth can-i \
  get nodes \
  --as=system:serviceaccount:enterprise-security:workload-sa \
  -n enterprise-security
```

Expected:

```text
no
```

---

## 182.5. ClusterRole escalation test

```bash
kubectl auth can-i \
  create clusterroles \
  --as=system:serviceaccount:enterprise-security:workload-sa
```

Expected:

```text
no
```

---

## Final validation

```bash
echo "GET CONFIGMAPS:"
kubectl auth can-i \
  get configmaps \
  --as=system:serviceaccount:enterprise-security:workload-sa \
  -n enterprise-security

echo "DELETE PODS:"
kubectl auth can-i \
  delete pods \
  --as=system:serviceaccount:enterprise-security:workload-sa \
  -n enterprise-security

echo "GET SECRETS:"
kubectl auth can-i \
  get secrets \
  --as=system:serviceaccount:enterprise-security:workload-sa \
  -n enterprise-security

echo "GET NODES:"
kubectl auth can-i \
  get nodes \
  --as=system:serviceaccount:enterprise-security:workload-sa

echo "CREATE CLUSTERROLE:"
kubectl auth can-i \
  create clusterroles \
  --as=system:serviceaccount:enterprise-security:workload-sa
```

Expected:

```text
GET CONFIGMAPS:
yes

DELETE PODS:
no

GET SECRETS:
no

GET NODES:
no

CREATE CLUSTERROLE:
no
```

Status:

**PASS**

---

# Step 183. Create NetworkPolicy Default Deny

## Tujuan

Menerapkan prinsip:

```text
Default:
DENY

Exception:
ALLOW explicitly
```

AWS EKS best practices juga merekomendasikan memulai NetworkPolicy dengan default-deny kemudian menambahkan allow rules sesuai kebutuhan. ([AWS Documentation](https://docs.aws.amazon.com/eks/latest/best-practices/network-security.html?utm_source=chatgpt.com "Network security - Amazon EKS"))

Buat:

```bash
cat > networkpolicy-default-deny.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: enterprise-security
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
```

Apply:

```bash
kubectl apply \
  -f networkpolicy-default-deny.yaml
```

### Penjelasan

```yaml
podSelector: {}
```

Memilih semua Pod di namespace.

```yaml
policyTypes:
- Ingress
- Egress
```

Mengaktifkan policy untuk:

```text
incoming
outgoing
```

Validasi:

```bash
kubectl get networkpolicy \
  -n enterprise-security
```

Expected:

```text
default-deny
```

**Penting:** object berhasil dibuat belum berarti traffic benar-benar diblokir. Enforcement akan diuji diStep 184–185.

---

# Step 184. Test NetworkPolicy Ingress Isolation

## Tujuan

Menguji behavior sebenarnya.

Kita buat server Pod dan client Pod.

```bash
kubectl run network-server \
  -n enterprise-security \
  --image=nginx:alpine \
  --labels=app=network-server \
  --port=80
```

Client:

```bash
kubectl run network-client \
  -n enterprise-security \
  --image=curlimages/curl:latest \
  --command -- \
  sleep 3600
```

Tunggu:

```bash
kubectl wait \
  --for=condition=Ready \
  pod/network-server \
  -n enterprise-security \
  --timeout=120s
```

dan:

```bash
kubectl wait \
  --for=condition=Ready \
  pod/network-client \
  -n enterprise-security \
  --timeout=120s
```

---

## 184.1. Get server IP

```bash
SERVER_IP=$(kubectl get pod \
  network-server \
  -n enterprise-security \
  -o jsonpath='{.status.podIP}')

echo "$SERVER_IP"
```

Expected:

```text
10.x.x.x
```

---

## 184.2. Test before allow rule

```bash
kubectl exec \
  -n enterprise-security \
  network-client \
  -- curl \
  --connect-timeout 5 \
  "http://$SERVER_IP"
```

Expected ideal:

```text
timeout
```

karena default deny.

Kalau justru:

```text
<html>
...
```

berarti NetworkPolicy object ada tetapi enforcement tidak berjalan.

Itu akan kita klasifikasikan:

**FLOCi LIMITATION**

bukan PASS.

---

# Step 185. Add Explicit NetworkPolicy Allow

## Tujuan

Membuktikan policy tidak sekadar “deny everything”, tetapi bisa memberikan access secara least privilege.

```bash
# Buat NetworkPolicy 2-Arah (Ingress Server + Egress Client)
cat > networkpolicy-allow-client.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-network-server-ingress
  namespace: enterprise-security
spec:
  podSelector:
    matchLabels:
      app: network-server
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: network-client
    ports:
    - protocol: TCP
      port: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-network-client-egress
  namespace: enterprise-security
spec:
  podSelector:
    matchLabels:
      role: network-client
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: network-server
    ports:
    - protocol: TCP
      port: 80
EOF
```

Label client:

```bash
kubectl label pod \
  network-client \
  -n enterprise-security \
  role=network-client \
  --overwrite
```

Apply:

```bash
kubectl apply \
  -f networkpolicy-allow-client.yaml
```

Test:

```bash
kubectl exec \
  -n enterprise-security \
  network-client \
  -- curl \
  --connect-timeout 5 \
  "http://$SERVER_IP"
```

Expected:

```text
Welcome to nginx!
```

atau HTML nginx.

Sekarang buat pod dari namespace lain:

```bash
kubectl run network-attacker \
  -n enterprise-security-untrusted \
  --image=curlimages/curl:latest \
  --command -- \
  sleep 3600
```

Wait:

```bash
kubectl wait \
  --for=condition=Ready \
  pod/network-attacker \
  -n enterprise-security-untrusted \
  --timeout=120s
```

Test:

```bash
kubectl exec \
  -n enterprise-security-untrusted \
  network-attacker \
  -- curl \
  --connect-timeout 5 \
  "http://$SERVER_IP"
```

Expected:

```text
timeout
```

Jadi kita membuktikan:

```text
same namespace + correct label
        → ALLOW

different namespace
        → DENY
```

Ini adalah test yang jauh lebih bernilai daripada sekadar:

```bash
kubectl get networkpolicy
```

---

# Step 186. Test Egress and DNS Policy

## Tujuan

Default deny sebelumnya juga memblokir egress.

Kita harus menguji DNS secara eksplisit.

Check CoreDNS:

```bash
kubectl get pods \
  -n kube-system \
  -l k8s-app=kube-dns
```

Kemudian test DNS dari client:

```bash
kubectl exec \
  -n enterprise-security \
  network-client \
  -- nslookup kubernetes.default.svc.cluster.local
```

Expected ideal setelah default deny:

```text
timeout
```

karena DNS egress belum di-allow.

Sekarang add DNS exception:

```bash
cat > networkpolicy-allow-dns.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: enterprise-security
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
EOF
```

Apply:

```bash
kubectl apply \
  -f networkpolicy-allow-dns.yaml
```

Test:

```bash
kubectl exec \
  -n enterprise-security \
  network-client \
  -- nslookup kubernetes.default.svc.cluster.local
```

Expected:

```text
Server:
Address:

Name: kubernetes.default.svc.cluster.local
Address: ...
```

---

## 186.1. Test unauthorized egress

Coba akses external destination:

```bash
kubectl exec \
  -n enterprise-security \
  network-client \
  -- curl \
  --connect-timeout 5 \
  https://example.com
```

Expected ideal:

```text
timeout
```

Karena kita hanya allow DNS, bukan arbitrary internet egress.

Kalau external access tetap berhasil, berarti egress enforcement tidak berjalan.

Classification:

**FLOCi LIMITATION** jika policy engine/CNI di environment Floci tidak enforce.

---

# Step 187. Validate Pod SecurityContext

## Tujuan

Network identity saja belum cukup.

Container juga harus:

```text
non-root
no privilege escalation
drop capabilities
```

Buat test Pod:

```bash
cat > secure-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secure-workload
  namespace: enterprise-security
spec:
  serviceAccountName: workload-sa
  containers:
  - name: app
    image: nginxinc/nginx-unprivileged:alpine
    ports:
    - containerPort: 8080
    securityContext:
      runAsNonRoot: true
      runAsUser: 101
      runAsGroup: 101
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      readOnlyRootFilesystem: false
EOF
```

Apply:

```bash
kubectl apply -f secure-pod.yaml
```

Validate:

```bash
kubectl get pod \
  secure-workload \
  -n enterprise-security
```

Expected:

```text
Running
```

Check UID:

```bash
kubectl exec \
  -n enterprise-security \
  secure-workload \
  -- id
```

Expected:

```text
uid=101(...)
```

Check security context:

```bash
kubectl get pod \
  secure-workload \
  -n enterprise-security \
  -o jsonpath='{.spec.containers[0].securityContext}'
```

Expected:

```text
runAsNonRoot=true
runAsUser=101
allowPrivilegeEscalation=false
capabilities.drop=ALL
```

Status:

**PASS** jika konfigurasi diterima dan container berjalan sebagai non-root.

---

# Step 188. Validate ServiceAccount Isolation

## Tujuan

Membuktikan workload menggunakan ServiceAccount yang kita tentukan, bukan default ServiceAccount.

```bash
kubectl get pod \
  secure-workload \
  -n enterprise-security \
  -o jsonpath='{.spec.serviceAccountName}{"\n"}'
```

Expected:

```text
workload-sa
```

Kemudian:

```bash
kubectl get pod \
  secure-workload \
  -n enterprise-security \
  -o jsonpath='{.spec.automountServiceAccountToken}{"\n"}'
```

Expected:

```text
true
```

Karena workload ini memang membutuhkan Kubernetes identity.

Untuk workload yang **tidak membutuhkan Kubernetes API identity**, best practice berikutnya adalah:

```yaml
automountServiceAccountToken: false
```

Nanti ini akan kita gunakan dalam policy admission Phase 14.

---

# Step 189. Validate Existing IRSA OIDC Foundation

## Tujuan

Phase sebelumnya sudah memiliki OIDC/IRSA:

```text
OIDC issuer:
https://oidc.eks.us-east-1.amazonaws.com/id/3117390CF240449DA5D851635496DE9D
```

Sekarang kita validasi ulang dari perspektif Phase 13.

Set:

```bash
export OIDC_ID="3117390CF240449DA5D851635496DE9D"
export OIDC_PROVIDER="oidc.eks.us-east-1.amazonaws.com/id/$OIDC_ID"
# 2. Daftarkan OIDC Provider Phase 3 ke IAM Mock Floci 
aws --endpoint-url="$ENDPOINT" iam create-open-id-connect-provider --url "https://$OIDC_PROVIDER" --client-id-list "sts.amazonaws.com" --thumbprint-list "9e99a48a9960b14926bb7f3b02e22da2b0ab7280"
# 3. Verifikasi ketersediaan OIDC Provider di IAM 
aws --endpoint-url="$ENDPOINT" iam list-open-id-connect-providers 
# 4. Inspeksi & Validasi Trust Policy pada Role EnterpriseAppIRSARole 
aws --endpoint-url="$ENDPOINT" iam get-role --role-name EnterpriseAppIRSARole --query 'Role.AssumeRolePolicyDocument'
```

Check IAM OIDC provider:

```bash
aws --endpoint-url="$ENDPOINT" \
  iam list-open-id-connect-providers
```

Expected provider:

```text
arn:aws:iam::000000000000:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/3117390CF240449DA5D851635496DE9D
```

---

## 189.1. Inspect existing IRSA role

```bash
aws --endpoint-url="$ENDPOINT" \
  iam get-role \
  --role-name EnterpriseAppIRSARole
```

Cari:

```text
AssumeRolePolicyDocument
```

Ideal trust policy mempunyai:

```text
Federated:
arn:aws:iam::000000000000:oidc-provider/...
```

dan:

```text
sts:AssumeRoleWithWebIdentity
```

IRSA memang menggunakan OIDC token dari Kubernetes yang ditukar ke temporary credentials melalui STS `AssumeRoleWithWebIdentity`. ([AWS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html?utm_source=chatgpt.com "IAM roles for service accounts - Amazon EKS"))

Validation:

```bash
aws --endpoint-url="$ENDPOINT" \
  iam get-role \
  --role-name EnterpriseAppIRSARole \
  --query 'Role.AssumeRolePolicyDocument'
```

Status:

**PASS** jika trust relationship sesuai.

---

# Step 190. Configure IRSA ServiceAccount

## Tujuan

Menghubungkan Kubernetes ServiceAccount:

```text
workload-sa
```

dengan:

```text
EnterpriseAppIRSARole
```

Annotation:

```yaml
eks.amazonaws.com/role-arn
```

Buat:

```bash
kubectl annotate serviceaccount \
  workload-sa \
  -n enterprise-security \
  eks.amazonaws.com/role-arn=arn:aws:iam::000000000000:role/EnterpriseAppIRSARole \
  --overwrite
```

Validate:

```bash
kubectl get serviceaccount \
  workload-sa \
  -n enterprise-security \
  -o yaml
```

Expected:

```yaml
annotations:
  eks.amazonaws.com/role-arn: arn:aws:iam::000000000000:role/EnterpriseAppIRSARole
```

IRSA memang menggunakan annotation role ARN pada ServiceAccount. ([AWS Documentation](https://docs.aws.amazon.com/eks/latest/eksctl/iamserviceaccounts.html?utm_source=chatgpt.com "IAM Roles for Service Accounts - Eksctl User Guide"))

Status:

**PASS** jika annotation berhasil.

---

# Step 191. Validate IRSA Web Identity Injection

## Tujuan

Sekarang kita tidak cukup memeriksa annotation.

Kita harus melihat apakah Pod benar-benar menerima:

```text
AWS_ROLE_ARN
AWS_WEB_IDENTITY_TOKEN_FILE
```

```bash
cat > allow-irsa-test-egress.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-irsa-test-egress
  namespace: enterprise-security
spec:
  podSelector:
    matchLabels:
      app: irsa-test
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 172.17.0.2/32
    ports:
    - protocol: TCP
      port: 4566
EOF

kubectl apply -f allow-irsa-test-egress.yaml
```
Buat Pod:
```bash
cat > irsa-test.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: irsa-test
  namespace: enterprise-security
  labels:
    app: irsa-test
spec:
  serviceAccountName: workload-sa
  hostAliases:
  - ip: "172.17.0.2"
    hostnames:
    - "floci-endpoint"
  containers:
  - name: aws
    image: amazon/aws-cli:latest
    command: ["sleep", "3600"]
    env:
    - name: AWS_ROLE_ARN
      value: "arn:aws:iam::000000000000:role/EnterpriseAppIRSARole"
    - name: AWS_WEB_IDENTITY_TOKEN_FILE
      value: "/var/run/secrets/eks.amazonaws.com/serviceaccount/token"
    - name: AWS_DEFAULT_REGION
      value: "us-east-1"
    - name: AWS_ENDPOINT_URL
      value: "http://floci-endpoint:4566"
    - name: AWS_ENDPOINT_URL_STS
      value: "http://floci-endpoint:4566"
    volumeMounts:
    - name: aws-iam-token
      mountPath: "/var/run/secrets/eks.amazonaws.com/serviceaccount"
      readOnly: true
  volumes:
  - name: aws-iam-token
    projected:
      sources:
      - serviceAccountToken:
          audience: "sts.amazonaws.com"
          expirationSeconds: 86400
          path: "token"
EOF
```

Apply:

```bash
kubectl apply -f irsa-test.yaml
```

Wait:

```bash
kubectl wait \
  --for=condition=Ready \
  pod/irsa-test \
  -n enterprise-security \
  --timeout=180s
```

Check:

```bash
kubectl exec \
  -n enterprise-security \
  irsa-test \
  -- env | grep '^AWS_'
```

Expected ideal:

```text
AWS_ROLE_ARN=arn:aws:iam::000000000000:role/EnterpriseAppIRSARole
AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

Kemudian:

```bash
kubectl exec \
  -n enterprise-security \
  irsa-test \
  -- ls -l \
  /var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

Expected:

```text
token
```

---
# Step 192. Validate STS Identity from IRSA Pod

## Tujuan

Ini adalah test paling penting untuk IRSA.

```bash
kubectl exec \
  -n enterprise-security \
  irsa-test \
  -- aws sts get-caller-identity
```

Expected ideal:

```json
{
    "UserId": "...",
    "Account": "000000000000",
    "Arn": "arn:aws:sts::000000000000:assumed-role/EnterpriseAppIRSARole/..."
}
```

Kalau berhasil:

**IRSA END-TO-END = PASS**

Kalau:

```text
NoCredentials
```

atau web identity injection tidak terjadi:

**FLOCi LIMITATION** jika native Floci EKS IRSA integration tidak menyediakan credential bridge.

Ini konsisten dengan evidence dari Phase sebelumnya, di mana explicit STS exchange bekerja tetapi automatic WebIdentity credential chain di Pod mengalami limitation.

---

# Step 193. Validate Least-Privilege AWS Access

## Tujuan

IRSA tidak cukup hanya menghasilkan credentials.

Kita harus membuktikan permission IAM-nya.

Pertama:

```bash
kubectl exec \
  -n enterprise-security \
  irsa-test \
  -- aws sts get-caller-identity
```

Kemudian coba API yang memang diperbolehkan role.

Misalnya jika role memiliki access ke S3:

```bash
kubectl exec -n enterprise-security irsa-test -- aws s3api list-buckets
```

Lalu coba operation yang seharusnya tidak diizinkan, misalnya:

```bash
kubectl exec -n enterprise-security irsa-test -- aws iam list-users
```

Expected:

```text
Allowed operation
    → success

Unauthorized operation
    → AccessDenied
```

Kalau AWS CLI tidak mendapat credentials karena Floci WebIdentity limitation, status:

**FLOCi LIMITATION**

Jangan menganggap:

```text
NoCredentials
```

sama dengan:

```text
AccessDenied
```

Itu dua failure berbeda.

---

# Step 194. Test Unauthorized ServiceAccount

## Tujuan

Membuktikan Pod yang tidak menggunakan IRSA ServiceAccount tidak otomatis mendapatkan IAM role application.

Buat Pod:

```bash
cat > no-irsa-test.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: no-irsa-test
  namespace: enterprise-security
spec:
  serviceAccountName: default
  containers:
  - name: aws
    image: amazon/aws-cli:latest
    command:
    - /bin/sh
    - -c
    - |
      aws sts get-caller-identity || true
      sleep 3600
EOF
```

Apply:

```bash
kubectl apply -f no-irsa-test.yaml
```

Wait:

```bash
kubectl wait \
  --for=condition=Ready \
  pod/no-irsa-test \
  -n enterprise-security \
  --timeout=180s
```

Check:

```bash
kubectl exec \
  -n enterprise-security \
  no-irsa-test \
  -- env | grep '^AWS_ROLE_ARN' || true
```

Expected:

```text
no output
```

Kemudian:

```bash
kubectl exec \
  -n enterprise-security \
  no-irsa-test \
  -- aws sts get-caller-identity
```

Expected ideal:

```text
NoCredentials
```

atau tidak mendapatkan `EnterpriseAppIRSARole`.

Ini membuktikan identity isolation.

---

# Step 195. Detect EKS Pod Identity Capability

## Tujuan

Sekarang kita test mechanism kedua:

```text
EKS Pod Identity
```

Berbeda dengan IRSA, Pod Identity tidak menggunakan annotation pada ServiceAccount. Association dibuat melalui EKS dan membutuhkan Pod Identity Agent pada node. ([AWS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html?utm_source=chatgpt.com "Learn how EKS Pod Identity grants pods access to AWS services - Amazon EKS"))

Check EKS API:

```bash
aws --endpoint-url="$ENDPOINT" \
  eks list-pod-identity-associations \
  --cluster-name "$CLUSTER_NAME"
```

Jika API tersedia:

```text
associations
```

Jika Floci belum mendukung:

```text
UnknownOperation
```

atau:

```text
UnsupportedOperation
```

→ **FLOCi LIMITATION**

---

## 195.1. Check Pod Identity Agent

```bash
kubectl get daemonset \
  -A | grep -i pod-identity
```

Ideal AWS:

```text
eks-pod-identity-agent
```

Kemudian:

```bash
kubectl get pods \
  -A | grep -i pod-identity
```

Expected ideal:

```text
eks-pod-identity-agent
```

Pod Identity Agent memang diperlukan untuk EKS Pod Identity. ([AWS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html?utm_source=chatgpt.com "Learn how EKS Pod Identity grants pods access to AWS services - Amazon EKS"))

---

# Step 196. Test EKS Pod Identity Association

## Tujuan

Jika Floci mendukung API-nya, kita buat association:

```text
enterprise-security
        │
        ▼
workload-podidentity
        │
        ▼
IAM Role
```

Buat ServiceAccount:

```bash
kubectl create serviceaccount \
  podidentity-sa \
  -n enterprise-security
```

Buat association:

```bash
aws --endpoint-url="$ENDPOINT" \
  eks create-pod-identity-association \
  --cluster-name "$CLUSTER_NAME" \
  --namespace enterprise-security \
  --service-account podidentity-sa \
  --role-arn arn:aws:iam::000000000000:role/EnterpriseAppIRSARole
```

### Parameter

`--cluster-name`

Cluster target.

`--namespace`

Namespace ServiceAccount.

`--service-account`

ServiceAccount target.

`--role-arn`

IAM role yang diberikan kepada Pod.

Ideal AWS juga mensyaratkan IAM role trust terhadap principal:

```text
pods.eks.amazonaws.com
```

dan Pod Identity Agent berjalan pada node. ([AWS Documentation](https://docs.aws.amazon.com/eks/latest/best-practices/identity-and-access-management.html?utm_source=chatgpt.com "Identity and Access Management - Amazon EKS"))

Kalau API tidak didukung:

**FLOCi LIMITATION**

---

# Step 197. Validate Pod Identity Association

JikaStep 196 berhasil:

```bash
aws --endpoint-url="$ENDPOINT" \
  eks list-pod-identity-associations \
  --cluster-name "$CLUSTER_NAME"
```

Expected:

```text
enterprise-security
podidentity-sa
EnterpriseAppIRSARole
```

Kemudian:

```bash
aws --endpoint-url="$ENDPOINT" \
  eks describe-pod-identity-association \
  --cluster-name "$CLUSTER_NAME" \
  --association-id "<ASSOCIATION_ID>"
```

Expected:

```text
associationArn
associationId
clusterName
namespace
serviceAccount
roleArn
```

Kalau association API tidak tersedia:

**FLOCi LIMITATION**

---

# Step 198. Validate Pod Identity Credential Injection

## Tujuan

Buat workload menggunakan ServiceAccount Pod Identity.

```bash
cat > podidentity-test.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: podidentity-test
  namespace: enterprise-security
spec:
  serviceAccountName: podidentity-sa
  containers:
  - name: aws
    image: amazon/aws-cli:2
    command:
    - /bin/sh
    - -c
    - |
      env | grep '^AWS_' || true
      sleep 3600
EOF
```

Apply:

```bash
kubectl apply \
  -f podidentity-test.yaml
```

Check:

```bash
kubectl exec \
  -n enterprise-security \
  podidentity-test \
  -- env | grep '^AWS_'
```

Ideal:

```text
AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE=...
AWS_CONTAINER_CREDENTIALS_FULL_URI=...
```

atau environment variables sesuai Pod Identity credential provider mechanism.

Pod Identity menggunakan EKS Auth API dan agent untuk memberikan temporary credentials kepada workload. ([AWS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-how-it-works.html?utm_source=chatgpt.com "Understand how EKS Pod Identity works - Amazon EKS"))

---

# Step 199. Validate Pod Identity AWS API Access

```bash
kubectl exec \
  -n enterprise-security \
  podidentity-test \
  -- aws sts get-caller-identity
```

Expected ideal:

```json
{
  "Account": "000000000000",
  "Arn": "arn:aws:sts::000000000000:assumed-role/EnterpriseAppIRSARole/..."
}
```

Jika:

```text
NoCredentials
```

atau agent/API tidak tersedia:

**FLOCi LIMITATION**

---

# Step 200. Compare IRSA vs Pod Identity

## Tujuan

Membuktikan dua mekanisme identity secara eksplisit.

|Property|IRSA|EKS Pod Identity|
|---|---|---|
|Kubernetes ServiceAccount|Ya|Ya|
|IAM role|Ya|Ya|
|OIDC provider|Ya|Tidak diperlukan untuk Pod Identity|
|ServiceAccount annotation|Ya|Tidak|
|EKS Pod Identity Agent|Tidak|Ya|
|STS WebIdentity|Ya|Tidak sebagai mekanisme utama|
|EKS Auth API|Tidak|Ya|
|Credential injection|WebIdentity|Pod Identity Agent|
|Floci expectation|mungkin limitation|sangat mungkin limitation|
|AWS ideal|PASS|PASS|

IRSA menggunakan OIDC dan `AssumeRoleWithWebIdentity`, sedangkan Pod Identity memakai EKS Auth API + Pod Identity Agent. ([AWS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html?utm_source=chatgpt.com "IAM roles for service accounts - Amazon EKS"))

Validation command:

```bash
echo "=== IRSA ==="

kubectl get serviceaccount \
  workload-sa \
  -n enterprise-security \
  -o yaml | grep role-arn || true

echo
echo "=== POD IDENTITY ==="

aws --endpoint-url="$ENDPOINT" \
  eks list-pod-identity-associations \
  --cluster-name "$CLUSTER_NAME" \
  2>&1
```

---

# Step 201. Validate Namespace + RBAC + NetworkPolicy Integration

## Tujuan

Sekarang kita tidak lagi menguji component satu-satu.

Kita test security boundary lengkap:

```text
Namespace
   │
   ├── ServiceAccount
   │
   ├── RBAC
   │
   ├── NetworkPolicy
   │
   └── SecurityContext
```

Check:

```bash
kubectl get namespace enterprise-security
```

```bash
kubectl get serviceaccount \
  workload-sa \
  -n enterprise-security
```

```bash
kubectl get role \
  workload-config-reader \
  -n enterprise-security
```

```bash
kubectl get rolebinding \
  workload-config-reader \
  -n enterprise-security
```

```bash
kubectl get networkpolicy \
  -n enterprise-security
```

```bash
kubectl get pod \
  secure-workload \
  -n enterprise-security
```

Expected:

```text
Namespace             PASS
ServiceAccount        PASS
RBAC Role             PASS
RoleBinding           PASS
NetworkPolicy         PASS
Secure workload       PASS
```

---

# Step 202. Validate Cross-Namespace Isolation

## Tujuan

Menguji apakah workload `untrusted` bisa mengakses resource security.

RBAC:

```bash
kubectl auth can-i \
  get configmaps \
  --as=system:serviceaccount:enterprise-security-untrusted:untrusted-sa \
  -n enterprise-security
```

Expected:

```text
no
```

Network:

```bash
kubectl exec \
  -n enterprise-security-untrusted \
  network-attacker \
  -- curl \
  --connect-timeout 5 \
  "http://$SERVER_IP"
```

Expected:

```text
timeout
```

Jadi:

```text
untrusted namespace
       │
       ├── Kubernetes API access → DENY
       │
       └── protected workload → DENY
```

Ini merupakan isolation test end-to-end.

---

# Step 203. Final Cleanup of Temporary Test Workloads

## Tujuan

Menghapus workload testing tanpa menghapus security foundation.

Delete:

```bash
kubectl delete pod \
  network-server \
  network-client \
  network-attacker \
  secure-workload \
  irsa-test \
  no-irsa-test \
  podidentity-test \
  -n enterprise-security \
  --ignore-not-found
```

Delete temporary ServiceAccount:

```bash
kubectl delete serviceaccount \
  podidentity-sa \
  -n enterprise-security \
  --ignore-not-found
```

Delete test policies:

```bash
kubectl delete networkpolicy \
  default-deny \
  allow-network-client \
  allow-dns \
  -n enterprise-security \
  --ignore-not-found
```

**Catatan:** jangan hapus `workload-sa`, Role, dan RoleBinding karena itu bagian dari security foundation Phase 13.

Validasi:

```bash
kubectl get pods \
  -n enterprise-security
```

Expected:

```text
No temporary test workloads
```

---

# Step 204. Phase 13 Final Validation

Ini command final yang nanti paling berguna setelah lu kirim hasil Floci.

```bash
echo "=================================================="
echo " PHASE 13 FINAL VALIDATION"
echo "=================================================="

echo
echo "[1] EKS:"
kubectl get nodes

echo
echo "[2] Namespaces:"
kubectl get ns \
  enterprise-security \
  enterprise-security-untrusted

echo
echo "[3] ServiceAccounts:"
kubectl get sa \
  -n enterprise-security

echo
echo "[4] RBAC:"
kubectl get role,rolebinding \
  -n enterprise-security

echo
echo "[5] RBAC Allowed:"
kubectl auth can-i \
  get configmaps \
  --as=system:serviceaccount:enterprise-security:workload-sa \
  -n enterprise-security

echo
echo "[6] RBAC Forbidden:"
kubectl auth can-i \
  delete pods \
  --as=system:serviceaccount:enterprise-security:workload-sa \
  -n enterprise-security

echo
echo "[7] NetworkPolicy:"
kubectl get networkpolicy \
  -n enterprise-security

echo
echo "[8] CNI:"
kubectl get pods \
  -A -o wide | \
  grep -Ei 'cni|calico|cilium|flannel|aws-node' || true

echo
echo "[9] IRSA Annotation:"
kubectl get sa workload-sa \
  -n enterprise-security \
  -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}{"\n"}'

echo
echo "[10] IAM OIDC Providers:"
aws --endpoint-url="$ENDPOINT" \
  iam list-open-id-connect-providers

echo
echo "[11] IRSA Role:"
aws --endpoint-url="$ENDPOINT" \
  iam get-role \
  --role-name EnterpriseAppIRSARole \
  --query 'Role.AssumeRolePolicyDocument'

echo
echo "[12] Pod Identity Associations:"
aws --endpoint-url="$ENDPOINT" \
  eks list-pod-identity-associations \
  --cluster-name "$CLUSTER_NAME"

echo
echo "[13] Pod Identity Agent:"
kubectl get daemonset \
  -A | grep -i pod-identity || true

echo
echo "=================================================="
echo " PHASE 13 VALIDATION COMPLETE"
echo "=================================================="
```

---

# Expected Final Matrix Phase 13

Untuk sementara, **sebelum output Floci lu masuk**, matrix yang kita jadikan baseline:

|Security Layer|Expected|
|---|---|
|Existing EKS|PASS|
|Node readiness|PASS|
|Namespace isolation|PASS|
|ServiceAccount|PASS|
|Bound ServiceAccount token|PASS|
|Least-privilege RBAC|PASS|
|RBAC positive authorization|PASS|
|RBAC negative authorization|PASS|
|Default-deny NetworkPolicy object|PASS|
|NetworkPolicy ingress enforcement|PASS / FLOCi LIMITATION|
|NetworkPolicy ingress allow rule|PASS / FLOCi LIMITATION|
|Cross-namespace network isolation|PASS / FLOCi LIMITATION|
|NetworkPolicy egress enforcement|PASS / FLOCi LIMITATION|
|DNS exception|PASS / FLOCi LIMITATION|
|External egress denial|PASS / FLOCi LIMITATION|
|Non-root workload|PASS|
|Privilege escalation disabled|PASS|
|Linux capabilities dropped|PASS|
|IRSA OIDC provider|PASS|
|IRSA IAM trust policy|PASS|
|IRSA ServiceAccount mapping|PASS|
|IRSA token injection|PASS / FLOCi LIMITATION|
|IRSA STS exchange|PASS / FLOCi LIMITATION|
|IRSA AWS identity|PASS / FLOCi LIMITATION|
|IRSA least privilege|PASS / FLOCi LIMITATION|
|Unauthorized SA isolation|PASS / FLOCi LIMITATION|
|Pod Identity API|PASS / FLOCi LIMITATION|
|Pod Identity Agent|PASS / FLOCi LIMITATION|
|Pod Identity association|PASS / FLOCi LIMITATION|
|Pod Identity credential injection|PASS / FLOCi LIMITATION|
|Pod Identity STS identity|PASS / FLOCi LIMITATION|
|Pod Identity least privilege|PASS / FLOCi LIMITATION|
|Namespace + RBAC + NetworkPolicy integration|PASS / FLOCi LIMITATION|
|Kubernetes → AWS identity integration|PASS / FLOCi LIMITATION|

### Yang paling penting dari Phase 13

Kita **jangan menyamakan tiga hal ini**:

```text
1. Kubernetes RBAC
       ↓
   "Pod boleh ngapain ke Kubernetes API?"

2. NetworkPolicy
       ↓
   "Pod boleh bicara ke Pod mana?"

3. IRSA / Pod Identity
       ↓
   "Pod boleh ngapain ke AWS API?"
```

Itulah security model yang mau kita bangun:

```text
                     WORKLOAD
                        │
          ┌─────────────┼─────────────┐
          │             │             │
          ▼             ▼             ▼
        RBAC       NetworkPolicy   IAM Identity
          │             │             │
          ▼             ▼             ▼
 Kubernetes API      Pod/Network     AWS API
 permissions         permissions     permissions
```

Dan Phase 14 nanti baru masuk akal untuk menaruh **Kyverno/OPA** di atas semua ini:

```text
                 Developer
                     │
                     ▼
                   EKS
                     │
              ┌──────┴──────┐
              │ Admission   │
              │ Controller  │
              └──────┬──────┘
                     │
              Kyverno / OPA
                     │
       ┌─────────────┼─────────────┐
       │             │             │
   Non-root      No privileged   Image policy
   required       container      / digest
       │             │             │
       └─────────────┼─────────────┘
                     ▼
                Kubernetes
                     │
                     ▼
             Phase 13 controls
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
         RBAC    NetworkPolicy  IAM
                                │
                         IRSA / Pod Identity
```

Jadi **jangan jalankan Phase 14 dulu**. Jalankan Phase 13 dari **177 sampai 204**, terutama bagian **178, 184–186, 189–199**. Dari output itulah kita bisa bedakan dengan presisi mana yang benar-benar didukung Floci dan mana yang harus masuk `FLOCi LIMITATION`, terutama **NetworkPolicy, IRSA automatic injection, dan EKS Pod Identity**. ([AWS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html?utm_source=chatgpt.com "IAM roles for service accounts - Amazon EKS"))