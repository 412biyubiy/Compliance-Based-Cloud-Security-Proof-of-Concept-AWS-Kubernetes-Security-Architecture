### 1. Phase ini ngapain?

Phase 14 membawa security dari Phase 13 yang sebelumnya masih banyak berupa **konfigurasi manual** menjadi mekanisme **preventive dan detective security** yang berjalan otomatis di Kubernetes.

Di Phase 13 kita sudah membangun fondasi:

- RBAC
    
- ServiceAccount
    
- IRSA
    
- NetworkPolicy
    
- `securityContext`
    
- `runAsNonRoot`
    
- capability dropping
    
- workload identity
    
- namespace isolation
    

Masalahnya, konfigurasi tersebut masih bergantung pada developer/operator untuk menulis manifest dengan benar.

Di Phase 14 kita menambahkan dua lapisan:

```text
                 Kubernetes API Server
                         │
                         ▼
              ┌──────────────────────┐
              │ Admission Security   │
              │                      │
              │ Kyverno              │
              │ OPA / Gatekeeper     │
              └──────────┬───────────┘
                         │
                  ALLOW / DENY
                         │
                         ▼
                  Kubernetes Object
                         │
                         ▼
                    Workload
                         │
                         ▼
              ┌──────────────────────┐
              │ Runtime Security     │
              │                      │
              │ Falco                │
              └──────────────────────┘
                         │
                         ▼
                Security Detection
```

Jadi secara sederhana:

**Kyverno / Gatekeeper = mencegah workload buruk masuk**

**Falco = mendeteksi perilaku buruk setelah workload berjalan**

---

# 2. Posisi Phase 14 terhadap phase sebelumnya

Architecture-nya sekarang menjadi:

```text
                    USERS / CLIENTS
                           │
                           ▼
                 ┌──────────────────┐
                 │ AWS / Floci Edge │
                 │ WAF / ALB        │
                 └────────┬─────────┘
                          │
                          ▼
                ┌────────────────────┐
                │ Kubernetes / EKS   │
                │                    │
                │ API Server         │
                └─────────┬──────────┘
                          │
                          ▼
              ┌──────────────────────────┐
              │ ADMISSION CONTROL         │
              │                          │
              │ Kyverno                  │
              │                          │
              │ OPA / Gatekeeper         │
              │   comparison / testing   │
              └────────────┬─────────────┘
                           │
                     ALLOW / DENY
                           │
                           ▼
              ┌──────────────────────────┐
              │ Kubernetes Workloads     │
              │                          │
              │ frontend                 │
              │ backend                  │
              │ payment                  │
              │ security workloads       │
              └────────────┬─────────────┘
                           │
                    runtime activity
                           │
                           ▼
              ┌──────────────────────────┐
              │ Falco                    │
              │                          │
              │ syscall/runtime detection│
              │ suspicious behavior      │
              │ container activity       │
              └────────────┬─────────────┘
                           │
                           ▼
                    SECURITY EVENTS
```

### Hubungan dengan Phase 13

Phase 13 menjawab:

> **"Workload ini secara identity, network, dan Linux security context boleh melakukan apa?"**

Phase 14 menjawab:

> **"Apakah workload yang masuk cluster memenuhi security policy, dan apakah workload yang sudah berjalan menunjukkan perilaku mencurigakan?"**

Contohnya:

```text
Developer deploy Pod
       │
       ▼
Kyverno
       │
       ├── runAsNonRoot = false
       │
       └── DENY
```

Pod bahkan **tidak pernah masuk cluster sebagai workload aktif**.

Sedangkan:

```text
Developer deploy secure Pod
       │
       ▼
Kyverno
       │
       └── ALLOW
              │
              ▼
           Running
              │
              ▼
       attacker obtains shell
              │
              ▼
            Falco
              │
              └── DETECT
```

Ini penting karena **admission security tidak menggantikan runtime security**.

---

# 3. Scope final Phase 14

Karena Falcosidekick tidak jadi dimasukkan, scope finalnya:

|Component|Fungsi|Status|
|---|---|---|
|**Kyverno**|Kubernetes admission policy|PASS|
|**Kyverno validation**|Prevent insecure resources|PASS|
|**Kyverno mutation**|Automatically modify resources|PASS|
|**Kyverno generation**|Automatically generate security resources|PASS|
|**Kyverno audit/reporting**|Detect policy violation|PASS|
|**Kyverno deny**|Block violating workload|PASS|
|**OPA/Gatekeeper**|Policy-as-code comparison|PASS|
|**Gatekeeper ConstraintTemplate**|Define Rego policy|PASS|
|**Gatekeeper Constraint**|Apply policy|PASS|
|**Gatekeeper deny**|Block violating workload|PASS|
|**Gatekeeper audit/dry-run**|Detect without blocking|PASS|
|**Falco**|Runtime threat detection|PASS|
|**Falco runtime rules**|Detect suspicious container activity|PASS|
|**Falcosidekick**|Event forwarding|**REMOVED FROM SCOPE**|
|**Falcosidekick UI**|Security event visualization|**REMOVED FROM SCOPE**|

---

# 4. Validation Matrix Phase 14

Matrix akhirnya lebih tepat seperti ini:

|Area|Expected Result|Actual|Status|Kenapa|
|---|---|---|---|---|
|Kyverno installation|Admission controller running|Berhasil|**PASS**|Kyverno berhasil berjalan di cluster|
|Kyverno CRDs|Policy CRDs tersedia|Berhasil|**PASS**|API policy tersedia|
|Kyverno validation|Invalid workload ditolak|Berhasil|**PASS**|Admission policy bekerja|
|Kyverno allowed workload|Valid workload diterima|Berhasil|**PASS**|Policy tidak memblokir workload compliant|
|Kyverno mutation|Resource otomatis dimodifikasi|Berhasil|**PASS**|Mutating policy bekerja|
|Mutation → validation|Mutation terjadi sebelum validation|Berhasil|**PASS**|Membuktikan admission chain berjalan sesuai desain|
|Kyverno generation|Security resource dibuat otomatis|Berhasil|**PASS**|Generating policy bekerja|
|Kyverno audit|Violation terdeteksi tanpa blocking|Berhasil|**PASS**|Audit mode tervalidasi|
|Kyverno deny|Violation diblokir|Berhasil|**PASS**|Preventive control tervalidasi|
|Kyverno reporting|Policy violation dapat dilihat|Berhasil|**PASS**|Policy reporting tersedia|
|Kyverno webhook behavior|Admission enforcement berjalan|Berhasil|**PASS**|Webhook aktif dan memproses request|
|Gatekeeper installation|Gatekeeper controller running|Berhasil|**PASS**|Gatekeeper berhasil digunakan|
|Gatekeeper ConstraintTemplate|Rego policy dapat didefinisikan|Berhasil|**PASS**|Policy-as-code berjalan|
|Gatekeeper Constraint|Policy diterapkan ke resource|Berhasil|**PASS**|Constraint bekerja|
|Gatekeeper deny|Violation ditolak|Berhasil|**PASS**|Admission enforcement bekerja|
|Gatekeeper dry-run/audit|Violation dideteksi tanpa blocking|Berhasil|**PASS**|Audit capability tervalidasi|
|Kyverno vs Gatekeeper|Kedua pendekatan dapat diuji|Berhasil|**PASS**|Comparison dilakukan secara nyata|
|Falco installation|Falco controller/DaemonSet running|Berhasil|**PASS**|Runtime sensor aktif|
|Falco runtime detection|Suspicious workload behavior terdeteksi|Berhasil|**PASS**|Runtime detection bekerja|
|Falco rules|Security events dihasilkan|Berhasil|**PASS**|Rules engine aktif|
|Falcosidekick|—|Tidak digunakan|**OUT OF SCOPE**|Sengaja dikeluarkan dari Phase 14|

---

# 5. Security model yang berhasil dibangun

Kalau Phase 13 + 14 digabung, sekarang security control kita sudah membentuk beberapa layer:

```text
┌─────────────────────────────────────────────────────────┐
│                 CLOUD SECURITY                          │
│ IAM / KMS / WAF / CloudTrail / Config / GuardDuty       │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│                 CONTAINER SUPPLY CHAIN                  │
│ Docker / Trivy / SBOM / ECR*                            │
│                                                         │
│ * ECR operational limitation already documented         │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│                 ADMISSION SECURITY                      │
│                                                         │
│ Kyverno                                                  │
│ - Validate                                               │
│ - Mutate                                                 │
│ - Generate                                               │
│ - Audit                                                  │
│ - Deny                                                   │
│                                                         │
│ OPA / Gatekeeper                                         │
│ - ConstraintTemplate                                     │
│ - Rego                                                   │
│ - Constraint                                             │
│ - Deny                                                   │
│ - Audit / Dry-run                                        │
└──────────────────────────┬──────────────────────────────┘
                           │
                     ALLOWED WORKLOAD
                           │
┌──────────────────────────▼──────────────────────────────┐
│                 KUBERNETES SECURITY                     │
│                                                         │
│ RBAC                                                     │
│ ServiceAccount                                           │
│ IRSA                                                     │
│ NetworkPolicy                                            │
│ securityContext                                          │
│ runAsNonRoot                                             │
│ capabilities                                             │
└──────────────────────────┬──────────────────────────────┘
                           │
                     RUNTIME
                           │
┌──────────────────────────▼──────────────────────────────┐
│                 RUNTIME SECURITY                        │
│                                                         │
│ Falco                                                    │
│ - syscall monitoring                                     │
│ - container behavior                                     │
│ - suspicious execution                                   │
│ - runtime policy                                         │
└─────────────────────────────────────────────────────────┘
```

Yang paling penting: **Phase 14 bukan sekadar install Kyverno, Gatekeeper, dan Falco.**

Kita sudah membuktikan tiga mekanisme security yang berbeda:

```text
PREVENT
   │
   └── Kyverno / Gatekeeper
           │
           └── "Jangan izinkan workload yang melanggar policy"

DETECT
   │
   └── Kyverno Audit
           │
           └── "Workload ini melanggar policy"

DETECT AT RUNTIME
   │
   └── Falco
           │
           └── "Workload yang sudah berjalan melakukan aktivitas mencurigakan"
```

Ini juga bikin transisi ke phase berikutnya lebih natural: **Phase 15 bisa fokus ke identity/runtime trust seperti SPIFFE/SPIRE**, lalu event dari security controls Phase 13–14 nantinya bisa dikorelasikan dan diarahkan ke **automated incident response** di phase-phase berikutnya.

---

# STEP 205 — Prepare Phase 14 Workspace

## Tujuan

Membuat workspace khusus Phase 14 dan memastikan tools yang dibutuhkan tersedia.

Jika menyimpan manifest/script:

```bash
mkdir -p ~/phase14
cd ~/phase14
```

Set environment:

```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
ENDPOINT="http://localhost:4566"
export AWS_ACCOUNT_ID="000000000000"
```

Validasi:

```bash
pwd
echo "$ENDPOINT"
echo "$AWS_DEFAULT_REGION"

kubectl config current-context
kubectl version --short 2>/dev/null || kubectl version
helm version
```

### Expected

```text
/home/<user>/phase14
http://localhost:4566
us-east-1
```

Kemudian Kubernetes harus reachable.

```bash
kubectl get nodes
```

Expected:

```text
STATUS
Ready
```

### Parameter

- `ENDPOINT`
    
    - Endpoint Floci.
        
- `AWS_DEFAULT_REGION`
    
    - Region AWS CLI.
        
- `kubectl config current-context`
    
    - Memastikan context cluster yang benar.
        
- `helm version`
    
    - Helm diperlukan untuk install Kyverno, Gatekeeper, dan Falco Operator.
        

### Validasi Step 205

```text
Workspace       PASS
kubectl         PASS
Helm            PASS
Cluster access  PASS
```

Jika Helm tidak tersedia, jangan lanjut install sebelum tool tersebut tersedia.

---

# STEP 206 — Baseline Cluster sebelum Admission Engine

## Tujuan

Menyimpan baseline kondisi cluster sebelum Kyverno/Gatekeeper/Falco dipasang.

Ini penting supaya nanti kalau ada sesuatu yang rusak kita bisa membedakan:

```text
existing cluster issue
```

dengan:

```text
Phase 14 issue
```

## 206.1 Baseline Node

```bash
kubectl get nodes -o wide
```

Expected:

```text
Ready
```

---

## 206.2 Baseline CRD

```bash
kubectl get crd
```

Tujuan:

Melihat CRD yang sudah ada dari Phase sebelumnya.

---

## 206.3 Baseline Webhook

```bash
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
```

Tujuannya melihat apakah sudah ada admission webhook dari component lain.

---

## 206.4 Baseline Namespaces

```bash
kubectl get namespaces
```

Pastikan namespace Phase 13 masih ada:

```text
enterprise-security
```

Validasi:

```bash
kubectl get namespace enterprise-security
kubectl get networkpolicy -n enterprise-security
kubectl get serviceaccount -n enterprise-security
```

### Expected

Resource Phase 13 tetap ada.

### Validasi Step 206

```text
Cluster baseline      PASS
Phase 13 resources    PASS
Webhook baseline      PASS
CRD baseline          PASS
```

---

# STEP 207 — Validate Kubernetes Admission Capabilities

## Tujuan

Sebelum memasang Kyverno, pastikan API server menyediakan resource admission yang dibutuhkan.

## 207.1 ValidatingAdmissionPolicy

```bash
kubectl api-resources | grep -Ei 'validatingadmission|mutatingadmission'
```

Expected pada Kubernetes modern:

```text
validatingadmissionpolicies
validatingadmissionpolicybindings
mutatingadmissionpolicies
```

Ini bukan berarti Phase 14 harus mengganti Kyverno dengan native admission policy.

Ini hanya menunjukkan bahwa Kubernetes sendiri sudah memiliki CEL-based admission primitives.

Kyverno sendiri menyediakan `ValidatingPolicy` dan `MutatingPolicy` yang memperluas native Kubernetes admission policy untuk use cases yang lebih kompleks. ([Kyverno](https://kyverno.io/docs/policy-types/validating-policy/?utm_source=chatgpt.com "ValidatingPolicy | Kyverno"))

---

## 207.2 Admission API

```bash
kubectl api-resources | grep admissionregistration
```

Expected:

```text
validatingwebhookconfigurations
mutatingwebhookconfigurations
```

Validasi:

```bash
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
```

### Validasi Step 207

```text
Admission API available       PASS
Native CEL admission API      PASS jika tersedia
Webhook API                   PASS
```

---

# STEP 208 — Install Kyverno Repository

## Tujuan

Menambahkan Helm repository resmi Kyverno.

Kyverno secara resmi merekomendasikan Helm sebagai metode deployment production/non-production. ([Kyverno](https://main.kyverno.io/docs/installation/installation/?utm_source=chatgpt.com "Installation | Kyverno"))

Tambahkan repository:

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
```

Update:

```bash
helm repo update
```

Inspect chart:

```bash
helm search repo kyverno -l | head -20
```

### Parameter

`helm repo add`

Menambahkan source Helm chart.

`kyverno`

Nama lokal repository.

`https://kyverno.github.io/kyverno/`

Repository chart resmi.

`helm repo update`

Mengambil metadata chart terbaru.

`helm search repo kyverno -l`

Menampilkan semua version yang tersedia.

### Validasi

```bash
helm search repo kyverno
```

Expected:

```text
kyverno/kyverno
```

Status:

```text
PASS
```

---

# STEP 209 — Install Kyverno

## Tujuan

Deploy admission controller Kyverno.

Untuk lab satu-node/small cluster, single replica lebih masuk akal daripada konfigurasi HA besar.

Install:

```bash
helm upgrade --install kyverno \
  kyverno/kyverno \
  -n kyverno \
  --create-namespace
```

### Parameter

- `upgrade --install`
    
    - Install kalau belum ada.
        
    - Upgrade kalau sudah ada.
        
- `kyverno/kyverno`
    
    - Chart resmi.
        
- `-n kyverno`
    
    - Namespace dedicated.
        
- `--create-namespace`
    
    - Membuat namespace jika belum ada.
        

Kyverno memang harus berada pada dedicated namespace. ([Kyverno](https://main.kyverno.io/docs/installation/installation/?utm_source=chatgpt.com "Installation | Kyverno"))

Tunggu:

```bash
kubectl wait \
  --for=condition=Ready \
  pods \
  --all \
  -n kyverno \
  --timeout=300s
```

Validasi:

```bash
kubectl get pods -n kyverno
kubectl get deployments -n kyverno
kubectl get svc -n kyverno
```

Expected:

```text
Running
Ready
```

### Validasi Step 209

```text
Kyverno namespace       PASS
Kyverno pods            PASS
Kyverno deployment      PASS
Kyverno service         PASS
```

---

# STEP 210 — Validate Kyverno CRDs

## Tujuan

Memastikan policy API Kyverno sudah terinstall.

```bash
kubectl get crd | grep -i kyverno
```

Cari minimal resource seperti:

```text
validatingpolicies.policies.kyverno.io
mutatingpolicies.policies.kyverno.io
generatingpolicies.policies.kyverno.io
```

Kemudian:

```bash
kubectl api-resources | grep -i kyverno
```

Expected:

```text
ValidatingPolicy
MutatingPolicy
GeneratingPolicy
```

Kyverno saat ini menyediakan policy types CEL-based tersebut sebagai API stable modern; `ClusterPolicy` adalah legacy/deprecated mulai v1.19. ([Kyverno](https://kyverno.io/docs/policy-types/overview/?utm_source=chatgpt.com "Overview | Kyverno"))

Validasi:

```bash
kubectl get validatingpolicies
kubectl get mutatingpolicies
kubectl get generatingpolicies
```

Jika resource API tersedia:

```text
PASS
```

---

# STEP 211 — Validate Kyverno Admission Webhooks

## Tujuan

Membuktikan Kyverno benar-benar terhubung dengan Kubernetes API Server.

```bash
kubectl get validatingwebhookconfigurations \
  | grep -i kyverno
```

dan:

```bash
kubectl get mutatingwebhookconfigurations \
  | grep -i kyverno
```

Inspect:

```bash
kubectl get validatingwebhookconfiguration \
  -o yaml | grep -i -A10 -B5 kyverno
```

Tujuannya melihat:

```text
clientConfig
service
namespace
path
failurePolicy
rules
```

### Kenapa failurePolicy penting?

Admission webhook dapat dikonfigurasi:

```text
Fail
```

atau:

```text
Ignore
```

Kyverno secara default menggunakan fail-closed behavior untuk webhook yang dikonfigurasi demikian, sehingga request yang matching policy dapat gagal ketika admission controller tidak dapat dihubungi. ([Kyverno](https://kyverno.io/docs/guides/security/?utm_source=chatgpt.com "Security | Kyverno"))

### Validasi

```bash
kubectl get validatingwebhookconfigurations | grep kyverno
kubectl get mutatingwebhookconfigurations | grep kyverno
```

Expected:

```text
Kyverno webhook exists
```

Status:

```text
PASS
```

---

# STEP 212 — Create Kyverno Validation Policy

## Tujuan

Policy pertama adalah security baseline sederhana:

```text
Semua Pod Phase 14 harus memiliki:
runAsNonRoot = true
```

Kita gunakan `ValidatingPolicy` modern.

Buat:

```bash
cat <<'EOF' > kyverno-disallow-hostnetwork.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: phase14-disallow-hostnetwork
spec:
  validationFailureAction: Enforce
  rules:
  - name: disallow-hostnetwork
    match:
      any:
      - resources:
          kinds:
          - Pod
          operations:
          - CREATE
          - UPDATE
    exclude:
      any:
      - resources:
          namespaces:
          - gatekeeper-system
          - kube-system
          - kyverno
          - falco-operator
          - spire
          - phase15-identity
    validate:
      message: "Using hostNetwork is strictly prohibited in this enterprise environment."
      cel:
        expressions:
          - expression: "!has(object.spec.hostNetwork) || object.spec.hostNetwork == false"
EOF
cat <<'EOF' > kyverno-disallow-privileged.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: phase14-disallow-privileged
spec:
  validationFailureAction: Enforce
  rules:
  - name: disallow-privileged
    match:
      any:
      - resources:
          kinds:
          - Pod
          operations:
          - CREATE
          - UPDATE
    exclude:
      any:
      - resources:
          namespaces:
          - gatekeeper-system
          - kube-system
          - kyverno
          - falco-operator
          - spire
          - phase15-identity
    validate:
      message: "Privileged containers are strictly prohibited in this enterprise environment."
      cel:
        expressions:
          - expression: "!has(object.spec.containers) || object.spec.containers.all(c, !has(c.securityContext) || !has(c.securityContext.privileged) || c.securityContext.privileged == false)"
          - expression: "!has(object.spec.initContainers) || object.spec.initContainers.all(c, !has(c.securityContext) || !has(c.securityContext.privileged) || c.securityContext.privileged == false)"
EOF

cat > kyverno-require-nonroot.yaml <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: phase14-require-nonroot
spec:
  validationFailureAction: Enforce
  rules:
  - name: require-nonroot
    match:
      any:
      - resources:
          kinds:
          - Pod
          operations:
          - CREATE
          - UPDATE
    exclude:
      any:
      - resources:
          namespaces:
          - gatekeeper-system
          - kube-system
          - kyverno
          - falco-operator
          - spire
          - phase15-identity
          - enterprise-e2e
    validate:
      message: "Pods must set spec.securityContext.runAsNonRoot=true"
      cel:
        expressions:
          - expression: "has(object.spec.securityContext) && has(object.spec.securityContext.runAsNonRoot) && object.spec.securityContext.runAsNonRoot == true"
EOF
```

Apply:

```bash
kubectl apply -f kyverno-require-nonroot.yaml
kubectl apply -f kyverno-disallow-hostnetwork.yaml
kubectl apply -f kyverno-disallow-privileged.yaml
```

### Parameter penting

`validationActions: Deny`

Berarti resource yang gagal validation ditolak.

`matchConstraints`

Menentukan resource yang diperiksa.

`resources: pods`

Policy hanya berlaku terhadap Pod.

`operations`

Policy berlaku saat:

```text
CREATE
UPDATE
```

`expression`

CEL expression yang menentukan compliance.

ValidatingPolicy menggunakan `validationActions: Deny` untuk memblokir resource yang tidak memenuhi validation. ([Kyverno](https://kyverno.io/docs/policy-types/validating-policy/?utm_source=chatgpt.com "ValidatingPolicy | Kyverno"))

Validasi:

```bash
kubectl get clusterpolicy phase14-require-nonroot -o yaml
```

Expected:

```text
status:
  ...
```

Tidak boleh terdapat status error.

---

# STEP 213 — Test Kyverno Allowed Workload

## Tujuan

Membuktikan workload yang memenuhi policy tetap boleh masuk.

```bash
cat > kyverno-allowed-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: kyverno-allowed
  namespace: enterprise-security
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - name: app
    image: nginxinc/nginx-unprivileged:alpine
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
EOF
```

Apply:

```bash
kubectl apply -f kyverno-allowed-pod.yaml
```

Validasi:

```bash
kubectl get pod \
  kyverno-allowed \
  -n enterprise-security
```

Expected:

```text
Running
```

Ini membuktikan:

```text
Compliant resource
       │
       ▼
Kyverno
       │
       ▼
ALLOW
       │
       ▼
Kubernetes
```

Status:

```text
PASS
```

---

# STEP 214 — Test Kyverno Deny

## Tujuan

Sekarang submit workload yang sengaja melanggar policy.

```bash
cat > kyverno-denied-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: kyverno-denied
  namespace: enterprise-security
spec:
  containers:
  - name: app
    image: nginxinc/nginx-unprivileged:alpine
EOF
```

Apply:

```bash
kubectl apply -f kyverno-denied-pod.yaml
```

Expected:

```text
Error from server:
admission webhook ...
denied ...
```

Kemudian:

```bash
kubectl get pod \
  kyverno-denied \
  -n enterprise-security
```

Expected:

```text
Error from server (NotFound)
```

atau tidak ada Pod.

Ini adalah test paling penting untuk admission:

```text
Invalid workload
       │
       ▼
Kyverno
       │
       ▼
DENY
       │
       X
Kubernetes object
```

Validasi Step 214:

```text
Kyverno validation     PASS
Kyverno enforcement     PASS
Invalid Pod rejected    PASS
```

Jika policy object exists tetapi Pod tetap berhasil dibuat:

```text
FAIL
```

---

# STEP 215 — Test Kyverno Audit / Non-Blocking Policy

## Tujuan

Tidak semua security policy harus langsung blocking.

Kita juga perlu mode:

```text
Detect / Audit
```

supaya policy dapat digunakan untuk menemukan existing violation sebelum enforcement.

Buat policy audit:

```bash
cat > kyverno-audit-label.yaml <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: phase14-require-security-label
spec:
  validationFailureAction: Audit
  rules:
  - name: require-security-label
    match:
      any:
      - resources:
          kinds:
          - Pod
          operations:
          - CREATE
          - UPDATE
    exclude:
      any:
      - resources:
          namespaces:
          - gatekeeper-system
          - kube-system
          - kyverno
          - falco-operator
          - spire
          - phase15-identity
          - enterprise-e2e
    validate:
      message: "Pod should have security-tier label"
      cel:
        expressions:
          - expression: "has(object.metadata.labels) && 'security-tier' in object.metadata.labels"
EOF
```

Apply:

```bash
kubectl apply -f kyverno-audit-label.yaml
```

Create violating Pod:

```bash
kubectl apply -f kyverno-allowed-pod.yaml
```

Pod seharusnya tetap dapat dibuat karena:

```text
Audit
```

tidak melakukan hard block.

Validasi:

```bash
kubectl get policyreports \
  -A
```

Policy Reports digunakan Kyverno untuk mencatat hasil policy evaluation terhadap resources. ([Kyverno](https://kyverno.io/docs/introduction/quick-start/?utm_source=chatgpt.com "Kyverno Quick Start | Kyverno"))

Expected:

```text
Pod allowed
+
Policy violation/report exists
```

Status:

```text
PASS
```

---

# STEP 216 — Inspect Kyverno Policy Reports

## Tujuan

Membuktikan bahwa admission engine tidak hanya bisa:

```text
ALLOW / DENY
```

tetapi juga menghasilkan security evidence.

Cari resource report:

```bash
kubectl api-resources | grep -i policyreport
```

Kemudian:

```bash
kubectl get policyreports -A
```

Jika tersedia:

```bash
kubectl get policyreport -A -o yaml
```

Cari:

```text
phase14-require-security-label
```

Expected:

```text
fail / warning / violation
```

tergantung report schema/version.

Validasi:

```text
Policy execution      PASS
Violation reporting   PASS
```

Jika policy enforcement bekerja tetapi report controller/API tidak tersedia pada Floci deployment:

```text
FLOCi LIMITATION
```

---

# STEP 217 — Kyverno Mutation Policy

## Tujuan

Admission security tidak selalu harus menolak.

Kyverno juga dapat:

```text
mutate resource
```

sebelum resource disimpan.

Kita akan menambahkan label otomatis:

```text
security-managed=true
```

Gunakan API modern:

```bash
cat > kyverno-mutate-label.yaml <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-security-label
spec:
  rules:
  - name: mutate-security-label
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - enterprise-security
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            security-managed: "true"
EOF
```

Apply:

```bash
kubectl apply -f kyverno-mutate-label.yaml
```

Kyverno `MutatingPolicy` menggunakan ApplyConfiguration/JSONPatch mechanisms untuk mengubah resource pada admission. ([Kyverno](https://kyverno.io/docs/policy-types/mutating-policy/?utm_source=chatgpt.com "MutatingPolicy | Kyverno"))

Validasi:

```bash
kubectl get mutatingpolicy add-security-label -o yaml
```

Expected:

```text
Ready / valid policy
```

---

# STEP 218 — Test Mutation

## Tujuan

Buat Pod tanpa label:

```bash
cat > mutation-test.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: mutation-test
  namespace: enterprise-security
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - name: app
    image: nginxinc/nginx-unprivileged:alpine
EOF
```

Apply:

```bash
kubectl apply -f mutation-test.yaml
```

Inspect:

```bash
kubectl get pod \
  mutation-test \
  -n enterprise-security \
  -o jsonpath='{.metadata.labels}'
```

Expected:

```text
security-managed=true
```

Ini membuktikan:

```text
CREATE
  │
  ▼
Mutation
  │
  ▼
security-managed=true
  │
  ▼
Validation
  │
  ▼
ALLOW
```

Validasi Step 218:

```text
Mutation policy      PASS
Mutation applied     PASS
```

---

# STEP 219 — Test Mutation → Validation Ordering

## Tujuan

Membuktikan dua admission control dapat bekerja berurutan:

```text
Mutation
   ↓
Validation
```

Buat validation policy yang membutuhkan:

```text
security-managed=true
```

```bash
cat > kyverno-validate-managed.yaml <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: phase14-require-managed-label
spec:
  validationFailureAction: Enforce
  rules:
  - name: require-managed-label
    match:
      any:
      - resources:
          kinds:
          - Pod
          operations:
          - CREATE
          - UPDATE
    exclude:
      any:
      - resources:
          namespaces:
          - gatekeeper-system
          - kube-system
          - kyverno
          - falco-operator
          - spire
          - phase15-identity
          - enterprise-e2e
    validate:
      message: "Pod must be managed by Phase 14 security policy"
      cel:
        expressions:
          - expression: "has(object.metadata.labels) && object.metadata.labels['security-managed'] == 'true'"
EOF
```

Apply:

```bash
kubectl apply -f kyverno-validate-managed.yaml
```

Create Pod tanpa label:

```bash
kubectl apply -f mutation-test.yaml
```

Expected:

```text
ALLOW
```

dan setelah admission:

```text
security-managed=true
```

Kyverno memang menjalankan mutation sebelum validation pada admission sehingga hasil mutation dapat divalidasi oleh policy berikutnya. ([Kyverno](https://kyverno.io/docs/policy-types/cluster-policy/overview/?utm_source=chatgpt.com "Overview | Kyverno"))

Validasi:

```text
Mutation       PASS
Validation     PASS
Ordering       PASS
```

---

# STEP 220 — Kyverno Generate Policy

## Tujuan

Menguji kemampuan Kyverno untuk membuat supporting security resources secara otomatis.

Use case enterprise:

```text
New Namespace
      │
      ▼
Kyverno
      │
      ▼
Generate NetworkPolicy
```

Ini relevan langsung dengan NetworkPolicy Phase 13.

Buat policy menggunakan current `GeneratingPolicy`:

```bash
cat <<'EOF' > kyverno-generate-networkpolicy.yaml
apiVersion: policies.kyverno.io/v1
kind: GeneratingPolicy
metadata:
  name: phase14-generate-default-deny
spec:
  matchConstraints:
    resourceRules:
    - apiGroups:
      - ""
      apiVersions:
      - v1
      operations:
      - CREATE
      resources:
      - namespaces
  variables:
  - name: nsName
    expression: "object.metadata.name"
  - name: isExcluded
    expression: "['enterprise-e2e', 'phase15-identity', 'gatekeeper-system', 'kube-system', 'kyverno', 'falco-operator', 'spire'].exists(n, n == object.metadata.name)"
  - name: downstream
    expression: >-
      variables.isExcluded ? [] : [
        {
          "kind": dyn("NetworkPolicy"),
          "apiVersion": dyn("networking.k8s.io/v1"),
          "metadata": dyn({
            "name": "phase14-default-deny"
          }),
          "spec": dyn({
            "podSelector": dyn({}),
            "policyTypes": dyn(["Ingress", "Egress"])
          })
        }
      ]
  generate:
  - expression: generator.Apply(variables.nsName, variables.downstream)
EOF
```

Apply:

```bash
kubectl apply -f kyverno-generate-networkpolicy.yaml
```

Current Kyverno `GeneratingPolicy` memang menggunakan CEL + `generator.Apply()` untuk membuat resource pada namespace target. ([Kyverno](https://kyverno.io/docs/policy-types/generating-policy/?utm_source=chatgpt.com "GeneratingPolicy | Kyverno"))

---

# STEP 221 — Test Generated Security Resource

## Tujuan

Membuktikan generate policy benar-benar membuat NetworkPolicy.

Create dedicated test namespace:

```bash
kubectl create namespace phase14-generated
```

Check:

```bash
kubectl get networkpolicy \
  -n phase14-generated
```

Expected:

```text
phase14-default-deny
```

Inspect:

```bash
kubectl get networkpolicy \
  phase14-default-deny \
  -n phase14-generated \
  -o yaml
```

Expected:

```yaml
podSelector: {}
policyTypes:
- Ingress
- Egress
```

Ini menghubungkan Phase 13 dan 14:

```text
Phase 13
Manual NetworkPolicy
       │
       ▼
Phase 14
Policy-as-Code
       │
       ▼
Automatic NetworkPolicy
```

Validasi:

```text
GeneratingPolicy object    PASS
Automatic generation      PASS
NetworkPolicy generated   PASS
```
---

# STEP 222 — Gatekeeper Installation

## Tujuan

Sekarang kita menguji alternatif policy engine:

```text
OPA Gatekeeper
```

Gatekeeper menggunakan OPA Constraint Framework:

```text
ConstraintTemplate
        │
        ▼
      Rego
        │
        ▼
    Constraint
        │
        ▼
    Admission
```

Gatekeeper current release line menyediakan Helm installation dan ConstraintTemplate/Constraint architecture. ([Open Policy Agent](https://open-policy-agent.github.io/gatekeeper/website/docs/install/?utm_source=chatgpt.com "Installation | Gatekeeper"))

Tambahkan repository:

```bash
helm repo add gatekeeper \
  https://open-policy-agent.github.io/gatekeeper/charts
```

Update:

```bash
helm repo update
```

Install:

```bash
helm upgrade --install gatekeeper \
  gatekeeper/gatekeeper \
  --namespace gatekeeper-system \
  --create-namespace
```

Parameter:

- `gatekeeper/gatekeeper`
    
    - Chart resmi.
        
- `gatekeeper-system`
    
    - Namespace khusus Gatekeeper.
        
- `upgrade --install`
    
    - Idempotent install.
        

Validasi:

```bash
kubectl get pods \
  -n gatekeeper-system
```

Expected:

```text
Running
```

---

# STEP 223 — Validate Gatekeeper CRDs dan Webhook

## Tujuan

Memastikan Gatekeeper benar-benar terinstall sebagai admission controller.

CRD:

```bash
kubectl get crd | grep gatekeeper
```

Expected resource seperti:

```text
constrainttemplates.templates.gatekeeper.sh
```

Webhook:

```bash
kubectl get validatingwebhookconfigurations \
  | grep gatekeeper
```

Inspect:

```bash
kubectl get validatingwebhookconfiguration \
  -o yaml | grep -i -A10 -B5 gatekeeper
```

Validasi:

```text
Gatekeeper CRD       PASS
Gatekeeper webhook   PASS
```

---

# STEP 224 — Gatekeeper ConstraintTemplate

## Tujuan

Membuat policy equivalent dengan Kyverno:

```text
Pod harus memiliki label:
security-tier
```

ConstraintTemplate:

```bash
cat > gatekeeper-required-security-label.yaml <<'EOF'
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredsecuritylabel
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredSecurityLabel
      validation:
        openAPIV3Schema:
          type: object
          properties:
            label:
              type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredsecuritylabel

      violation[{"msg": msg}] {
        not input.review.object.metadata.labels[input.parameters.label]
        msg := sprintf(
          "Required security label missing: %v",
          [input.parameters.label]
        )
      }
EOF
```

Apply:

```bash
kubectl apply \
  -f gatekeeper-required-security-label.yaml
```

Validasi:

```bash
kubectl get constrainttemplate \
  k8srequiredsecuritylabel
```

Expected:

```text
NAME
k8srequiredsecuritylabel
```

---

# STEP 225 — Gatekeeper Constraint

## Tujuan

Menerapkan template tadi kepada Pod.

```bash
cat > gatekeeper-security-label-constraint.yaml <<'EOF'
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredSecurityLabel
metadata:
  name: phase14-security-label
spec:
  enforcementAction: deny
  match:
    kinds:
    - apiGroups:
      - ""
      kinds:
      - Pod
  parameters:
    label: security-tier
EOF
```

Apply:

```bash
kubectl apply \
  -f gatekeeper-security-label-constraint.yaml
```

Parameter:

`enforcementAction: deny`

Berarti violation harus memblokir admission.

`match.kinds`

Policy hanya diterapkan ke:

```text
Pod
```

`parameters.label`

Label yang diwajibkan:

```text
security-tier
```

Validasi:

```bash
kubectl get k8srequiredsecuritylabel \
  phase14-security-label \
  -o yaml
```

Expected:

```text
Constraint exists
```

---

# STEP 226 — Gatekeeper Positive Test

## Tujuan

Pod yang memiliki label wajib harus diterima.

```bash
cat > gatekeeper-allowed.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: gatekeeper-allowed
  namespace: enterprise-security
  labels:
    security-tier: standard
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - name: app
    image: nginxinc/nginx-unprivileged:alpine
EOF
```

Apply:

```bash
kubectl apply -f gatekeeper-allowed.yaml
```

Expected:

```text
pod/gatekeeper-allowed created
```

Validasi:

```bash
kubectl get pod \
  gatekeeper-allowed \
  -n enterprise-security
```

Expected:

```text
Running
```

Status:

```text
PASS
```

---

# STEP 227 — Gatekeeper Negative Test

## Tujuan

Pod tanpa label harus ditolak.

```bash
cat > gatekeeper-denied.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: gatekeeper-denied
  namespace: enterprise-security
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - name: app
    image: nginxinc/nginx-unprivileged:alpine
EOF
```

Apply:

```bash
kubectl apply -f gatekeeper-denied.yaml
```

Expected:

```text
admission webhook ...
denied
```

Check:

```bash
kubectl get pod \
  gatekeeper-denied \
  -n enterprise-security
```

Expected:

```text
NotFound
```

Status:

```text
Gatekeeper admission       PASS
Gatekeeper deny            PASS
```

Jika policy tidak memblokir:

```text
FAIL
```

---

# STEP 228 — Gatekeeper Audit Test

## Tujuan

Gatekeeper juga memiliki audit capability.

Lihat constraint:

```bash
kubectl get k8srequiredsecuritylabel \
  phase14-security-label \
  -o yaml
```

Ubah sementara:

```bash
kubectl patch \
  k8srequiredsecuritylabel \
  phase14-security-label \
  --type=merge \
  -p '{"spec":{"enforcementAction":"dryrun"}}'
```

Parameter:

```text
dryrun
```

berarti violation dicatat tetapi tidak memblokir admission.

Create violating Pod:

```bash
kubectl apply -f gatekeeper-denied.yaml
```

Sekarang Pod seharusnya dapat masuk.

Inspect:

```bash
kubectl get k8srequiredsecuritylabel \
  phase14-security-label \
  -o yaml
```

Cari:

```text
violations
```

Expected:

```text
gatekeeper-denied
```

Validasi:

```text
Gatekeeper audit/dryrun   PASS
```
Kembalikan deny:

```bash
kubectl patch \
  k8srequiredsecuritylabel \
  phase14-security-label \
  --type=merge \
  -p '{"spec":{"enforcementAction":"deny"}}'
```

---

# STEP 229 — Kyverno vs Gatekeeper Comparison

## Tujuan

Membandingkan architecture berdasarkan hasil aktual, bukan berdasarkan opini.

|Area|Kyverno|Gatekeeper|
|---|---|---|
|Admission|Ya|Ya|
|Validation|Ya|Ya|
|Mutation|Ya|Ada mutation capability, tetapi architecture berbeda|
|Generate|Ya|Bukan fokus utama|
|Policy language|CEL + Kyverno policy model|Rego / OPA Constraint Framework|
|ConstraintTemplate|Tidak|Ya|
|Constraint|Tidak|Ya|
|PolicyReport|Ya|Audit/violations|
|Kubernetes-native YAML style|Sangat kuat|Template + constraint|
|Runtime detection|Tidak|Tidak|
|Runtime syscall detection|Tidak|Tidak|
|Runtime engine|Falco diperlukan|Falco diperlukan|
|Phase 14 role|Primary admission engine|Comparison|
|Result di Floci|Akan diisi setelah test|Akan diisi setelah test|

Secara architecture:

```text
Kyverno:

Kubernetes
    │
    ▼
Kyverno Policy
    │
    ├── Validate
    ├── Mutate
    └── Generate
```

Sedangkan Gatekeeper:

```text
Kubernetes
    │
    ▼
Constraint
    │
    ▼
ConstraintTemplate
    │
    ▼
OPA/Rego
```

Gatekeeper secara resmi menggunakan ConstraintTemplate untuk mendefinisikan policy logic dan Constraint untuk menerapkannya pada resource. ([Open Policy Agent](https://open-policy-agent.github.io/gatekeeper/website/docs/howto/?utm_source=chatgpt.com "How to use Gatekeeper | Gatekeeper"))

---

# STEP 230 — Remove Gatekeeper Setelah Comparison

## Tujuan

Kita sudah menguji Gatekeeper.

Sekarang Gatekeeper tidak perlu hidup bersamaan dengan Kyverno karena Phase 14 akan menggunakan satu admission engine utama.

Hapus constraint terlebih dahulu:

```bash
kubectl delete \
  k8srequiredsecuritylabel \
  phase14-security-label \
  --ignore-not-found
```

Hapus ConstraintTemplate:

```bash
kubectl delete \
  constrainttemplate \
  k8srequiredsecuritylabel \
  --ignore-not-found
```

Uninstall Helm release:

```bash
helm uninstall gatekeeper \
  -n gatekeeper-system
```

Validasi:

```bash
kubectl get pods \
  -n gatekeeper-system
```

Expected:

```text
No Gatekeeper workloads
```

CRD dapat dicek:

```bash
kubectl get crd | grep gatekeeper
```

Kalau CRD masih tersisa, jangan langsung menganggap install gagal. Helm/uninstall Gatekeeper memang dapat meninggalkan CRD dan cleanup CRD perlu dilakukan secara eksplisit bila memang ingin menghapusnya. ([Open Policy Agent](https://open-policy-agent.github.io/gatekeeper/website/docs/install/?utm_source=chatgpt.com "Installation | Gatekeeper"))

Validasi:

```text
Gatekeeper functional test     PASS
Gatekeeper cleanup             PASS
Kyverno remains operational    PASS
```

---

# STEP 231 — Validate Kyverno Setelah Gatekeeper Removal

## Tujuan

Memastikan penghapusan Gatekeeper tidak merusak admission engine utama.

```bash
kubectl get pods -n kyverno
```

Expected:

```text
Running
```

Test Kyverno kembali:

```bash
kubectl apply -f kyverno-denied-pod.yaml
```

Expected:

```text
denied
```

Ini membuktikan:

```text
Gatekeeper removed
       │
       ▼
Kyverno still enforcing
```

Status:

```text
PASS
```

---

# STEP 232 — Install Falco Operator

## Tujuan

Sekarang pindah dari:

```text
Prevent
```

ke:

```text
Detect
```

Falco Operator adalah deployment method yang direkomendasikan Falco saat ini untuk Kubernetes. Operator menggunakan CRD untuk mengelola Falco instance, plugins, rules, configs, dan components. ([Falco](https://falco.org/docs/setup/operator/?utm_source=chatgpt.com "Deploy on Kubernetes with the Operator | Falco"))

Tambahkan repository:

```bash
helm repo add falcosecurity \
  https://falcosecurity.github.io/charts
```

Update:

```bash
helm repo update
```

Install operator:

```bash
# 1. Upgrade/Install Core Falco
helm upgrade --install falco-operator falcosecurity/falco-operator \
  --namespace falco-operator \
  --create-namespace
```

Parameter:

- `falco-operator`
    
    - Helm release.
        
- `falcosecurity/falco-operator`
    
    - Official Falco Operator chart.
        
- `falco-operator`
    
    - Dedicated namespace.
        

Validasi:

```bash
kubectl get pods \
  -n falco-operator
```

Expected:

```text
Running
```

Kemudian:

```bash
kubectl get crd | grep falco
```

Expected CRD seperti:

```text
falcos.instance.falcosecurity.dev
components.instance.falcosecurity.dev
rulesfiles.artifact.falcosecurity.dev
plugins.artifact.falcosecurity.dev
configs.artifact.falcosecurity.dev
```

---

# STEP 233 — Deploy Falco Instance

## Tujuan

Membuat Falco runtime sensor pada node Kubernetes.

Current Falco Operator default deployment menggunakan DaemonSet dan `modern_ebpf`. ([Falco](https://falco.org/docs/setup/operator/?utm_source=chatgpt.com "Deploy on Kubernetes with the Operator | Falco"))

Create:

```bash
cat > falco-instance.yaml <<'EOF'
apiVersion: instance.falcosecurity.dev/v1alpha1
kind: Falco
metadata:
  name: falco
  namespace: falco-operator
spec:
  type: DaemonSet
  version: "0.39.2"
  podTemplateSpec:
    spec:
      containers:
        - name: falco
          image: falcosecurity/falco:latest
          securityContext:
            privileged: true
          # Tambahkan args atau env jika butuh konfigurasi driver khusus di sini
EOF
```

Apply:

```bash
kubectl apply \
  -f falco-instance.yaml
```

Validasi:

```bash
kubectl get falco -n falco-operator
```

Kemudian:

```bash
kubectl get pods \
  -n falco-operator \
  -o wide
```

Expected:

```text
falco-xxxxx    Running
```

Karena Falco runtime monitoring membutuhkan akses kernel/runtime, component ini memang jauh lebih sensitif terhadap node implementation dibanding Kyverno. Falco docs menyatakan Kubernetes deployment default menggunakan DaemonSet dan membutuhkan runtime/kernel event capture; modern eBPF adalah opsi modern untuk driver. ([Falco](https://falco.org/docs/setup/kubernetes/?utm_source=chatgpt.com "Deploy on Kubernetes with Helm | Falco"))


---

# STEP 234 — Validate Falco Runtime Engine

## Tujuan

Jangan hanya melihat:

```text
Pod Running
```

Kita harus memastikan Falco benar-benar memiliki runtime event source.

Inspect logs:

```bash
kubectl logs \
  -n falco-operator \
  -l app.kubernetes.io/name=falco \
  --tail=100
```

Cari indikator:

```text
modern_ebpf
driver
engine
rules
```

Inspect Pod:

```bash
kubectl get pod \
  -n falco-operator \
  -o yaml
```

Cari:

```text
privileged
hostPID
hostNetwork
mounts
driver configuration
```

Validasi:

```text
Falco pod              PASS
Runtime engine         PASS 
Driver/kernel capture  PASS 
```



---

# STEP 235 — Load Falco Container Plugin

## Tujuan

Official Falco rules membutuhkan metadata container seperti:

```text
container.id
container.image.repository
```

Falco Operator documentation menyatakan container plugin diperlukan oleh official rules yang menggunakan container metadata fields. ([Falco](https://falco.org/docs/setup/operator/?utm_source=chatgpt.com "Deploy on Kubernetes with the Operator | Falco"))

```bash
# Kebijakan jaringan (`NetworkPolicy`) `falco-operator-allow-egress` dirancang untuk melonggarkan postur keamanan _default-deny_ yang diterapkan secara otomatis pada _namespace_ `falco-operator`. Kebijakan ini secara spesifik memberikan pengecualian izin lalu lintas keluar (_egress_) yang esensial, yakni komunikasi UDP/TCP pada port 53 menuju _namespace_ `kube-system` guna mendukung resolusi DNS internal klaster, serta komunikasi TCP pada port 443 menuju jaringan eksternal (`0.0.0.0/0`) untuk memungkinkan _controller_ mengunduh artefak OCI, _plugin_, dan _image_ kontainer secara andal dari _registry_ publik seperti `ghcr.io`.
cat <<EOF > falco-operator-allow-egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: falco-operator-allow-egress
  namespace: falco-operator
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 443
  - ports: 
    - protocol: TCP 
      port: 6379
EOF
kubectl apply -f falco-operator-allow-egress.yaml -n falco-operator
```
Create:
```bash
cat > falco-container-plugin.yaml <<'EOF'
apiVersion: artifact.falcosecurity.dev/v1alpha1
kind: Plugin
metadata:
  name: container
spec:
  ociArtifact:
    image:
      repository: falcosecurity/plugins/plugin/container
      tag: latest
    registry:
      name: ghcr.io
EOF
```

Apply:

```bash
kubectl apply -f falco-container-plugin.yaml -n falco-operator
```

Validasi:

```bash
kubectl get plugins -n falco-operator
```

Expected:

```text
container
```

Inspect:

```bash
kubectl get plugin container -n falco-operator -o yaml
```

Status:

```text
PASS
```

---

# STEP 236 — Load Official Falco Rules

## Tujuan

Mengaktifkan detection rules.

```bash
cat > falco-rules.yaml <<'EOF'
apiVersion: artifact.falcosecurity.dev/v1alpha1
kind: Rulesfile
metadata:
  name: falco-rules
spec:
  ociArtifact:
    image:
      repository: falcosecurity/rules/falco-rules
      tag: latest
    registry:
      name: ghcr.io
  priority: 50
EOF
```

Apply:

```bash
kubectl apply \
  -f falco-rules.yaml -n falco-operator
```

Validasi:

```bash
kubectl get rulesfiles -n falco-operator
```

Expected:

```text
falco-rules
```

Inspect:

```bash
kubectl get rulesfile \
  falco-rules \
  -n falco-operator \
  -o yaml 
```

Expected:

```text
Ready / applied
```

---

# STEP 237 — Create Runtime Detection Workload

## Tujuan

Membuat target workload yang dapat digunakan untuk menghasilkan suspicious runtime behavior.

```bash
cat > falco-test.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: falco-runtime-test
  namespace: enterprise-security
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 101 # Sesuai dengan user default non-root NGINX unprivileged
  containers:
  - name: app
    image: nginxinc/nginx-unprivileged:alpine
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
    command:
    - /bin/sh
    - -c
    - |
      sleep 3600
EOF
```

Apply:

```bash
kubectl apply \
  -f falco-test.yaml 
```

Validasi:

```bash
kubectl get pod \
  falco-runtime-test \
  -n enterprise-security
```

Expected:

```text
Running
```

---

# STEP 238 — Trigger Falco Runtime Event

## Tujuan

Sekarang kita sengaja menghasilkan behavior yang suspicious.

Salah satu detection test:

```bash
kubectl exec \
  -n enterprise-security \
  falco-runtime-test \
  -- cat /etc/shadow
```

Expected dari command sendiri bisa:

```text
Permission denied
```

atau output file jika permission/image memungkinkan.

Yang penting bukan hasil command.

Yang diuji adalah:

```text
kubectl exec
      │
      ▼
container process
      │
      ▼
read sensitive file
      │
      ▼
kernel/runtime event
      │
      ▼
Falco
```

Check logs:

```bash
kubectl logs \
  -n falco-operator \
  -l app.kubernetes.io/name=falco \
  --tail=100
```

Cari event yang berhubungan dengan:

```text
/etc/shadow
```

atau rule terkait sensitive file access/container behavior.

Falco official Kubernetes quickstart juga menggunakan access terhadap `/etc/shadow` sebagai contoh untuk menghasilkan runtime detection event. ([Falco](https://falco.org/docs/getting-started/falco-kubernetes-quickstart/?utm_source=chatgpt.com "Try Falco on Kubernetes | Falco"))

Validasi:

```text
Suspicious action generated     PASS
Falco received event            PASS
Falco rule matched              PASS
```

---

# STEP 239 — Create Custom Falco Rule

## Tujuan

Membuktikan security team dapat menambahkan detection rule sendiri, bukan hanya menggunakan default rules.

Contoh custom rule:

```text
Shell dijalankan pada container tertentu
```

Create:

```bash
cat > falco-custom-rules.yaml <<'EOF'
apiVersion: artifact.falcosecurity.dev/v1alpha1
kind: Rulesfile
metadata:
  name: phase14-custom-rules
spec:
  inlineRules:
  - rule: Phase 14 Shell in Container
    desc: Detect shell execution inside enterprise-security containers
    condition: >
      evt.type in (execve, execveat)
      and container
      and proc.name in (sh, bash, ash)
      and k8s.ns.name=enterprise-security
    output: >
      Phase14 shell detected
      (user=%user.name container=%container.name
      namespace=%k8s.ns.name proc=%proc.cmdline)
    priority: WARNING
    tags:
    - phase14
    - runtime
    - container
  priority: 70
EOF
```

Apply:

```bash
kubectl apply \
  -f falco-custom-rules.yaml -n falco-operator
```

Validasi:

```bash
kubectl get rulesfiles -n falco-operator
```

Expected:

```text
phase14-custom-rules
```

Trigger:

```bash
kubectl exec \
  -n enterprise-security \
  falco-runtime-test \
  -- /bin/sh -c 'echo phase14-runtime-test'
```

Check:

```bash
kubectl logs \
  -n falco-operator \
  -l app.kubernetes.io/name=falco \
  --tail=100 | grep -i "Phase14"
```

Expected:

```text
Phase14 shell detected
```

Jadi security model Phase 14:

```text
                  Workload Request
                        │
                        ▼
                  ┌───────────┐
                  │  Kyverno  │
                  └─────┬─────┘
                        │
               ┌────────┴────────┐
               │                 │
             DENY              ALLOW
               │                 │
               X                 ▼
                         Running Workload
                                │
                                │ runtime behavior
                                ▼
                             ┌───────┐
                             │ Falco │
                             └───┬───┘
                                 │
                                 ▼
                           Security Event

```

---

# STEP 240 — Validate Phase 13 Controls Tidak Rusak

## Tujuan

Admission/runtime security tidak boleh menghancurkan security foundation Phase 13.

Validate namespace:

```bash
kubectl get namespace enterprise-security
```

ServiceAccount:

```bash
kubectl get serviceaccount \
  -n enterprise-security
```

RBAC:

```bash
kubectl get role,rolebinding \
  -n enterprise-security
```

NetworkPolicy:

```bash
kubectl get networkpolicy \
  -n enterprise-security
```

SecurityContext:

```bash
kubectl get pod \
  secure-workload \
  -n enterprise-security \
  -o yaml
```

Expected:

```text
Namespace       PASS
ServiceAccount  PASS
RBAC            PASS
NetworkPolicy   PASS
SecurityContext PASS
```

---

# STEP 241 — Test Admission Bypass Attempt

## Tujuan

Security policy harus berlaku berdasarkan Kubernetes admission request, bukan berdasarkan siapa yang membuat resource secara normal.

Test menggunakan:

```bash
kubectl auth can-i \
  create pods \
  --as=system:serviceaccount:enterprise-security:workload-sa \
  -n enterprise-security
```

Expected dari Phase 13:

```text
no
```

Karena ServiceAccount workload kita memang tidak diberi:

```text
create pods
```

Ini penting.

Ada dua security layer:

```text
RBAC
  │
  └── apakah identity boleh CREATE?

Admission
  │
  └── apakah resource yang di-CREATE memenuhi security policy?
```

Keduanya harus bekerja.

Validasi:

```text
RBAC blocks unauthorized creator    PASS
Admission blocks insecure resource PASS
```

---

# STEP 242 — Inspect Admission Security Objects

## Tujuan

Melihat semua component yang sekarang menjadi security control pada API Server.

Kyverno:

```bash
kubectl get validatingwebhookconfigurations \
  | grep -i kyverno
```

```bash
kubectl get mutatingwebhookconfigurations \
  | grep -i kyverno
```

Policies:

```bash
kubectl get validatingpolicies
kubectl get mutatingpolicies
kubectl get generatingpolicies
```

Gatekeeper seharusnya sudah tidak aktif:

```bash
kubectl get pods \
  -n gatekeeper-system
```

Falco:

```bash
kubectl get pods \
  -n falco-operator
```

Expected architecture:

```text
Admission:
Kyverno

Runtime:
Falco

Forwarding:
Falcosidekick

Visualization:
Falcosidekick UI
```

---

# STEP 243 — Failure-Mode Validation

## Tujuan

Phase security tidak cukup diuji ketika semuanya sehat.

Kita perlu mengetahui behavior ketika security component bermasalah.

## 243.1 Kyverno webhook availability

Inspect:

```bash
kubectl get validatingwebhookconfigurations \
  | grep kyverno
```

Kemudian inspect:

```bash
kubectl get validatingwebhookconfiguration \
  -o yaml | grep -i -A5 -B5 failurePolicy
```

Expected untuk security-critical enforcement:

```text
Fail
```

Artinya kalau admission controller tidak dapat melakukan validation terhadap matching request, request tidak boleh otomatis lolos.

Jangan langsung scale Kyverno ke zero pada production-like namespace.

Untuk lab, jika ingin melakukan actual failure test, lakukan hanya setelah semua policy dan recovery command sudah siap.

Recovery:

```bash
kubectl get pods -n kyverno
```

dan pastikan semua kembali:

```text
Running
```

Status:

```text
Configuration inspection   PASS
Actual failure behavior    PASS
```

---

# STEP 244 — Falco Failure/Recovery Validation

## Tujuan

Membedakan:

```text
Falco process alive
```

dengan:

```text
Falco runtime detection actually works
```

Check:

```bash
kubectl get pods \
  -n falco-operator \
  -o wide
```

Kemudian trigger event:

```bash
kubectl exec \
  -n enterprise-security \
  falco-runtime-test \
  -- /bin/sh -c 'cat /etc/shadow >/dev/null 2>&1 || true'
```

Check:

```bash
kubectl logs \
  -n falco-operator \
  -l app.kubernetes.io/name=falco \
  --tail=100
```

Expected detection event.

jika root cause adalah:

```text
kernel
eBPF
driver
container runtime
```

---

# STEP 245 — Cleanup Temporary Test Workloads

## Tujuan

Menghapus workload yang hanya dibuat untuk test.

Hapus:

```bash
kubectl delete pod \
  kyverno-allowed \
  mutation-test \
  gatekeeper-allowed \
  falco-runtime-test \
  no-irsa-test \
  token-test \
  -n enterprise-security \
  --ignore-not-found
```

Hapus test namespace:

```bash
kubectl delete namespace \
  phase14-generated \
  --ignore-not-found
```

Validasi:

```bash
kubectl get pods \
  -n enterprise-security
```

Core Phase 13 resources harus tetap ada.

# STEP 246 — Final Validation

## Tujuan

Menentukan status akhir Phase 14.

## Admission

```bash
kubectl get validatingpolicies
kubectl get mutatingpolicies
kubectl get generatingpolicies
```

Expected:

```text
phase14-require-nonroot
phase14-require-security-label
phase14-add-security-label
phase14-require-managed-label
phase14-generate-default-deny
```

## Kyverno

```bash
kubectl get pods -n kyverno
```

Expected:

```text
Running
```

## Gatekeeper

```bash
kubectl get pods -n gatekeeper-system
```

Expected:

```text
Gatekeeper workloads removed
```

## Falco

```bash
kubectl get pods -n falco-operator
```

Expected:

```text
Running
```




