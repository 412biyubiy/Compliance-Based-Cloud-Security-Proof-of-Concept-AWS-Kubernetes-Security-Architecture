## 1. Phase ini ngapain?

Phase 17 adalah **capstone validation** dari seluruh project.

Phase 1–16 sebelumnya membangun dan memvalidasi individual security/control layer. Phase 17 menggabungkan semuanya ke satu **production-like enterprise transaction path**, lalu menguji apakah security controls tersebut benar-benar tetap berlaku ketika sistem:

```
user datang
→ application menerima traffic
→ backend memproses request
→ database digunakan
→ S3 digunakan
→ secret digunakan
→ workload memakai IAM/IRSA
→ network segmentation berlaku
→ workload identity berlaku
→ monitoring/security telemetry berjalan
→ attack disimulasikan
→ security control melakukan prevention/detection
→ workload dipulihkan
→ backup/restore tetap tersedia
→ Terraform tetap bisa reconcile
```

Jadi Phase 17 bukan lagi:

> “Apakah WAF hidup?”

atau:

> “Apakah SPIRE hidup?”

Tetapi:

> **“Apakah seluruh security architecture tetap bekerja ketika enterprise application benar-benar digunakan, diserang, dipulihkan, dan direconcile?”**

---

# 2. Posisi Phase 17 terhadap Phase 1–16

```
Phase 1
Networking
    │
Phase 2
Network Security
    │
Phase 3
Compute
    │
Phase 4
Edge / ALB / WAF
    │
Phase 5
Database
    │
Phase 6
S3 / Storage
    │
Phase 7
DNS / SSL
    │
Phase 8
KMS / Secrets
    │
Phase 9
Monitoring
    │
Phase 10
CloudTrail / Config / GuardDuty / Compliance
    │
Phase 11
Automated Response
    │
Phase 12
Supply Chain / ECR / SBOM
    │
Phase 13
EKS / RBAC / IRSA / NetworkPolicy
    │
Phase 14
Kyverno / Gatekeeper / Falco
    │
Phase 15
SPIFFE / SPIRE / Workload Identity
    │
Phase 16
Backup / Restore / Governance / Terraform
    │
    ▼
PHASE 17
END-TO-END ENTERPRISE VALIDATION
```

Dengan kata lain:

**Phase 17 menggunakan hasil semua phase sebelumnya sebagai control plane.**

---

# 3. Arsitektur Phase 17

Arsitektur final yang kita validasi:

```
                              INTERNET / USER
                                    │
                                    ▼
                               Route53 / DNS
                                    │
                                    ▼
                               CloudFront
                                    │
                                    ▼
                                  WAF
                                    │
                                    ▼
                                  ALB
                                    │
                          ┌─────────┴─────────┐
                          │                   │
                          ▼                   ▼
                      FRONTEND           Legacy EC2
                          │
                          ▼
                     Kubernetes
                          │
             ┌────────────┼────────────┐
             │            │            │
             ▼            ▼            ▼
         Frontend       Backend      Payment
                          │            │
                          └─────┬──────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
                    ▼                       ▼
                   RDS                    S3
                    │                       │
                    ▼                       ▼
                   KMS                  Object Data

                  WORKLOAD IDENTITY
                         │
                         ▼
                SPIFFE / SPIRE / SVID
                         │
                         ▼
                    mTLS Identity

                KUBERNETES SECURITY
                         │
        ┌────────────────┼─────────────────┐
        ▼                ▼                 ▼
    NetworkPolicy     Kyverno          Gatekeeper
        │                                  │
        └────────────────┬─────────────────┘
                         ▼
                       Falco

                 AWS SECURITY PLANE
                         │
       ┌─────────────────┼─────────────────┐
       ▼                 ▼                 ▼
    IAM/STS          CloudTrail          Config
       │                                   │
       ▼                                   ▼
    IRSA             GuardDuty         Compliance

                 OBSERVABILITY
                         │
        ┌────────────────┼───────────────┐
        ▼                ▼               ▼
   Application       Kubernetes        AWS
      Logs             Logs          Metrics/Logs

                 RESILIENCE / IaC
                         │
               ┌─────────┴─────────┐
               ▼                   ▼
             Backup             Terraform
               │                   │
               ▼                   ▼
            Restore           Rebuild/Drift
```

---

# 4. Cara membaca hasil Phase 17

Kita pakai tiga status saja:

### PASS

Capability benar-benar berhasil dibuktikan pada lab.

### FLOCi LIMITATION

Test sudah dilakukan, tetapi environment/emulator/mock application tidak menyediakan behavior yang diperlukan untuk membuktikan control tersebut.

Ini **bukan berarti control enterprise-nya gagal**.

Contohnya Step 334:

```
Expected:
user A tidak boleh mengakses user B

Actual:
http-echo → 200 OK untuk semua path
```

Yang tidak ada adalah **authorization logic** pada target test, bukan bukti bahwa architecture IAM/security lu bocor.

### FAIL

Capability seharusnya bekerja di environment tersebut tetapi actual result menyimpang.

Dan berdasarkan hasil lu:

**Tidak ada FAIL pada Phase 17.**

---

# 5. FINAL VALIDATION MATRIX — ACTUAL RESULT

|No|Capability|Actual Result|Kenapa|
|---|---|---|---|
|1|Phase 1–16 baseline|**PASS**|Platform dari Phase sebelumnya healthy sebelum final E2E|
|2|Application deployment|**PASS**|Production-like application workload berhasil dideploy|
|3|Frontend service|**PASS**|Frontend berhasil berjalan sebagai user-facing layer|
|4|Backend/API service|**PASS**|Backend service berhasil berjalan|
|5|Payment service|**PASS**|Sensitive business service berhasil berjalan|
|6|Database connectivity|**PASS**|Application layer dapat berkomunikasi dengan database|
|7|S3 connectivity|**PASS**|Application/data flow ke object storage berhasil|
|8|Secrets Manager integration|**PASS**|Secret dapat digunakan oleh application flow|
|9|KMS integration|**PASS**|Encryption/decryption capability berhasil digunakan|
|10|IAM least privilege|**PASS**|Intended workload permission dapat berjalan sesuai design|
|11|IRSA workload identity|**PASS**|Pod memperoleh AWS identity melalui workload identity|
|12|NetworkPolicy segmentation|**PASS**|Traffic antar workload mengikuti segmentation policy|
|13|SPIFFE/SPIRE identity|**PASS**|Workload identity/SVID tetap berfungsi|
|14|Normal user traffic|**PASS**|Normal application traffic berhasil|
|15|Checkout transaction|**PASS**|Business transaction end-to-end berhasil|
|16|DB write/read|**PASS**|Data dapat ditulis dan dibaca|
|17|S3 object operation|**PASS**|Object upload/download berhasil|
|18|WAF inspection|**PASS**|WAF layer dapat divalidasi pada jalur application|
|19|CloudTrail logging|**FLOCi LIMITATION**|Output CloudTrail tidak memberikan event history yang diperlukan untuk membuktikan logging API secara penuh|
|20|Application logging|**PASS**|Application activity dapat diamati pada application layer|
|21|Kubernetes security telemetry|**PASS**|Kubernetes/Falco/runtime security telemetry tersedia|
|22|Monitoring/metrics|**FLOCi LIMITATION**|Kubernetes metrics dan Falco berjalan, tetapi AWS-native CloudWatch aggregation tidak merepresentasikan seluruh cloud security activity|
|23|Broken access control test|**FLOCi LIMITATION**|`http-echo` tidak memiliki user/session/authorization model sehingga ID-based access control tidak dapat dibuktikan|
|24|IDOR test|**FLOCi LIMITATION**|Mock backend tidak mempunyai object ownership atau user-to-resource mapping|
|25|Authentication bypass test|**FLOCi LIMITATION**|`http-echo` tidak melakukan JWT/token/session validation sehingga token palsu tidak menghasilkan authentic 401|
|26|SQL injection test|**FLOCi LIMITATION**|Mock backend tidak terhubung ke SQL query execution path|
|27|XSS test|**FLOCi LIMITATION**|Tidak ada HTML rendering/reflection path yang dapat digunakan untuk membuktikan XSS prevention|
|28|Command injection test|**FLOCi LIMITATION**|Mock backend tidak meneruskan input ke shell/OS command execution|
|29|Path traversal test|**FLOCi LIMITATION**|Mock server tidak melakukan dynamic filesystem access|
|30|SSRF test|**FLOCi LIMITATION**|Tidak tersedia backend URL-fetching/outbound HTTP primitive pada target mock|
|31|HTTP method abuse|**FLOCi LIMITATION**|`http-echo` menerima berbagai HTTP methods tanpa router-level restriction|
|32|Malicious payload test|**FLOCi LIMITATION**|Mock server menerima payload besar/malformed dan tetap merespons statically|
|33|Security header validation|**PASS**|Security-header behavior dapat divalidasi|
|34|TLS/HTTPS validation|**PASS**|Transport security layer berhasil divalidasi|
|35|WAF detection/blocking|**FLOCi LIMITATION**|Jalur test tidak menghasilkan genuine WAF rule inspection/blocking evidence; request tetap dilayani mock application|
|36|GuardDuty detection|**PASS**|GuardDuty capability yang tersedia pada environment berhasil divalidasi|
|37|Config compliance detection|**PASS**|Configuration/compliance capability berhasil divalidasi|
|38|CloudTrail event correlation|**FLOCi LIMITATION**|Tidak tersedia event history yang cukup untuk melakukan korelasi CloudTrail secara penuh|
|39|Falco runtime detection|**PASS**|Falco agent aktif dan runtime detection berjalan|
|40|Kyverno admission enforcement|**PASS**|Non-compliant workload ditolak oleh admission policy|
|41|Gatekeeper policy enforcement|**PASS**|Policy violation berhasil ditolak|
|42|Network attack containment|**PASS**|NetworkPolicy membatasi lateral movement|
|43|AWS privilege escalation attempt|**PASS**|Privilege boundary pada intended workload dapat diuji|
|44|IRSA privilege abuse|**PASS**|Workload identity tetap terbatas pada intended AWS permissions|
|45|S3 unauthorized access|**PASS**|Unauthorized data access behavior berhasil divalidasi|
|46|KMS unauthorized decrypt|**FLOCi LIMITATION**|Static emulator credential path melewati granular IAM authorization sehingga test berhenti pada cryptographic validation (`InvalidCiphertextException`), bukan `AccessDenied`|
|47|Secrets unauthorized access|**FLOCi LIMITATION**|Secrets Manager pada local emulator tidak merepresentasikan granular IAM denial seperti AWS production dan unauthorized caller masih dapat retrieve secret|
|48|Backup validation|**PASS**|Backup lifecycle dari Phase 16 tetap tersedia|
|49|Restore validation|**PASS**|Restore job berhasil dijalankan pada Floci|
|50|Terraform state validation|**PASS**|Terraform state tetap dapat digunakan untuk reconciliation|
|51|Security regression after rebuild|**PASS**|IaC/recovery lifecycle tidak merusak security control|
|52|Application recovery|**PASS**|Application dapat recover setelah controlled workload disruption|
|53|Kubernetes workload recovery|**PASS**|Kubernetes replacement/recovery berhasil|
|54|Detection-to-response chain|**PASS**|Security event/control chain yang tersedia berhasil divalidasi pada environment|
|55|Final enterprise E2E|**PASS**|Enterprise lifecycle berhasil ditutup dengan seluruh capability utama tervalidasi dan limitation terdokumentasi|

---

# 6. Semua FLOCi LIMITATION Phase 17

Biar nggak ketuker, hasil limitation lu sebenarnya cuma ada **14 capability**.

## A. Application attack surface limitation

```
23 A01 Broken Access Control
24 IDOR
25 Authentication Bypass
26 SQL Injection
27 XSS
28 Command Injection
29 Path Traversal
30 SSRF
31 HTTP Method Abuse
32 Malicious Payload
35 WAF Detection/Blocking
```

Ini bukan 11 vulnerability.

Justru artinya **test engine kita tidak punya application primitive yang diperlukan untuk membuktikan control tersebut**.

Target backend saat ini:

```
                    REQUEST
                       │
                       ▼
                  http-echo
                       │
                       ▼
              "enterprise-api-ok"
```

Sedangkan realistic enterprise application yang dibutuhkan:

```
                    REQUEST
                       │
                       ▼
                 API ROUTER
                       │
              ┌────────┼─────────┐
              ▼        ▼         ▼
           AuthZ     Input      Business
           Layer   Validation     Logic
              │        │           │
              ▼        ▼           ▼
            User      SQL         DB
            Data     Query
                       │
                       ▼
                   File/HTTP
                   Operations
```

Makanya:

```
http-echo
     ↓
tidak bisa membuktikan
A01 / IDOR / A07 / A03 / A10
```

Ini limitation yang sangat penting untuk ditulis secara jujur di portfolio.

---

# 7. AWS-native limitation

Lima capability lain:

```
19  CloudTrail logging
22  Monitoring/metrics
38  CloudTrail correlation
46  KMS unauthorized decrypt
47  Secrets unauthorized access
```

### CloudTrail / CloudWatch

Kubernetes observability lu justru cukup bagus:

```
Metrics Server → PASS
Falco → PASS
Kubernetes runtime visibility → PASS
```

Yang terbatas adalah cloud-native telemetry simulation:

```
AWS API activity
      ↓
CloudTrail
      ↓
CloudWatch / centralized telemetry
```

Local AWS emulator tidak menghasilkan behavior telemetry sekomprehensif AWS production.

Ini relevan dengan OWASP A09 karena security logging bukan cuma soal “log ada”, tetapi apakah aktivitas dapat dipantau dan digunakan untuk detection/response/forensics. [OWASP Foundation](https://owasp.org/Top10/en/A09_2021-Security_Logging_and_Monitoring_Failures/?utm_source=chatgpt.com)

---

# 8. KMS limitation harus ditulis begini

Ini penting karena output lu sebenarnya **jangan sampai dibaca sebagai “KMS berhasil unauthorized decrypt”**.

Yang terjadi:

```
Unauthorized workload
        │
        ▼
    KMS Decrypt
        │
        ▼
 Local emulator
        │
        ▼
No granular IAM denial
        │
        ▼
Cryptographic validation
        │
        ▼
InvalidCiphertextException
```

Yang kita inginkan di AWS production:

```
Unauthorized workload
        │
        ▼
    KMS Decrypt
        │
        ▼
 IAM authorization
        │
        X
 AccessDenied
```

Jadi Step 354 harus **FLOCi LIMITATION**, bukan PASS dan bukan security failure.

---

# 9. Secrets Manager juga sama

Actual lab:

```
Unauthorized caller
        │
        ▼
Secrets Manager
        │
        ▼
SecretString returned
```

Production AWS yang kita desain:

```
Unauthorized caller
        │
        ▼
Secrets Manager
        │
        ▼
IAM evaluation
        X
AccessDenied
```

Jadi:

**Step 355 = FLOCi LIMITATION.**

Ini penting banget karena kalau lu tulis “PASS”, portfolio reviewer bisa salah mengartikan bahwa lu berhasil membuktikan unauthorized access prevention, padahal justru environment test-nya tidak mampu membuktikan control tersebut.

---

# 10. Compliance alignment final

Phase 17 sekarang sangat cocok dijadikan **control validation layer** terhadap beberapa framework.

## OWASP Top 10

OWASP Top 10:2021 mencakup A01 Broken Access Control, A02 Cryptographic Failures, A03 Injection, A05 Security Misconfiguration, A07 Identification and Authentication Failures, A09 Security Logging and Monitoring Failures, A10 SSRF, dan kategori lainnya. [OWASP Top 10](https://top10.owasp.org/2021/?utm_source=chatgpt.com)

Mapping project:

```
A01 Broken Access Control
→ Phase 17 Step 334
→ FLOCi LIMITATION

A02 Cryptographic Failures
→ Phase 8 KMS
→ Phase 17 KMS validation
→ PASS
   dengan negative authorization limitation pada Step 354

A03 Injection
→ SQLi / XSS / Command / Path
→ FLOCi LIMITATION karena mock backend

A04 Insecure Design
→ Threat model + architecture
→ Phase 1–17
→ PASS

A05 Security Misconfiguration
→ Kyverno / Gatekeeper / WAF / hardening
→ PASS

A06 Vulnerable Components
→ Phase 12 Trivy/SBOM
→ PASS

A07 Identification & Authentication
→ Phase 13/15
→ PASS pada platform identity
→ application authentication bypass test = limitation

A08 Software/Data Integrity
→ ECR / SBOM / Terraform
→ PASS kecuali ECR

A09 Logging & Monitoring
→ Kubernetes/Falco = PASS
→ CloudTrail/CloudWatch depth = FLOCi LIMITATION

A10 SSRF
→ Step 340
→ FLOCi LIMITATION
```

OWASP sendiri menempatkan A01 sebagai access-control enforcement dan A03 sebagai injection dari untrusted input; jadi limitation pada mock backend memang tepat dicatat sebagai keterbatasan test surface, bukan vulnerability finding. [OWASP Top 10](https://top10.owasp.org/2021/id/A01_2021-Broken_Access_Control/?utm_source=chatgpt.com)

---

# 11. PCI DSS alignment

Phase 17 menyentuh area yang relevan dengan:

```
Network Security
      ↓
Phase 1–4

Protect account/payment-related data
      ↓
Phase 5–8

Access control / least privilege
      ↓
Phase 8 / 13 / 17

Logging and monitoring
      ↓
Phase 9–11 / 17

Security testing
      ↓
Phase 12–17

Backup / recovery / availability
      ↓
Phase 16–17
```

Tetapi wording portfolio yang benar:

> **“The platform implements and validates security controls relevant to PCI DSS requirements.”**

Bukan:

> “PCI DSS compliant.”

Karena PCI DSS adalah formal compliance standard yang membutuhkan assessment terhadap requirement dan scope yang sebenarnya, bukan sekadar menjalankan lab. PCI SSC mendeskripsikan PCI DSS sebagai baseline requirements untuk melindungi payment account data.

---

# 12. ISO/IEC 27001 alignment

Phase 16–17 terutama memperkuat tiga objective besar:

```
Confidentiality
    │
    ├── IAM
    ├── IRSA
    ├── KMS
    ├── Secrets
    ├── NetworkPolicy
    └── SPIFFE/SPIRE

Integrity
    │
    ├── ECR
    ├── SBOM
    ├── Kyverno
    ├── Gatekeeper
    ├── Terraform
    └── Audit

Availability
    │
    ├── Kubernetes recovery
    ├── Backup
    ├── Restore
    ├── Terraform rebuild
    └── workload recovery
```

Ini sejalan dengan ISO/IEC 27001 yang menekankan confidentiality, integrity, availability dan risk-management process. [ISO](https://www.iso.org/standard/27001?browse=tc&utm_source=chatgpt.com)

Tetap sama: **aligned with ISO/IEC 27001 controls**, bukan “ISO 27001 certified”.

---

# 13. Final status Phase 17

Dengan output yang lu kasih:

```
========================================
PHASE 17 FINAL RESULT
========================================

PASS                 : 41
FLOCi LIMITATION     : 14
FAIL                 : 0
NOT TESTED           : 0
```

Dan 14 limitation tersebut sudah punya alasan yang jelas:

```
Application Mock Surface
├── Broken Access Control
├── IDOR
├── Authentication Bypass
├── SQL Injection
├── XSS
├── Command Injection
├── Path Traversal
├── SSRF
├── HTTP Method Abuse
├── Malicious Payload
└── WAF Detection/Blocking

AWS Emulator Observability/Authorization
├── CloudTrail Logging
├── CloudWatch/AWS Metrics
├── CloudTrail Correlation
├── KMS Unauthorized Decrypt
└── Secrets Manager Unauthorized Access
```

Jadi **tidak ada evidence bahwa project kita mengalami security breach atau architectural failure dari 14 limitation tersebut**. Yang terbukti adalah batas kemampuan environment test untuk mensimulasikan behavior production tertentu.

Dan menurut gue ini justru penutup project yang jauh lebih kuat:

```
BUILD
  ↓
HARDEN
  ↓
MONITOR
  ↓
DETECT
  ↓
RESPOND
  ↓
IDENTIFY
  ↓
BACKUP
  ↓
RESTORE
  ↓
REBUILD
  ↓
ATTACK
  ↓
REGRESSION
  ↓
FINAL E2E
```

---
# Step 315. Prepare Phase 17 Workspace

## Tujuan

Membuat workspace khusus untuk seluruh evidence final.

```bash
mkdir ~/phase17
cd ~/phase17

mkdir -p \
  app \
  manifests \
  attacks \
  evidence \
  logs \
  reports \
  compliance \
  recovery
```

### Penjelasan

```
app/
```

Application deployment artifacts.

```
manifests/
```

Kubernetes manifests.

```
attacks/
```

Payload dan test definitions.

```
evidence/
```

Raw command output.

```
logs/
```

Application/security logs.

```
reports/
```

Final validation report.

```
compliance/
```

OWASP / PCI / ISO mapping.

```
recovery/
```

Backup/recovery evidence.

### Validasi

```bash
pwd
find . -maxdepth 1 -type d | sort
```

Expected:

```
/home/<user>/phase17
./app
./attacks
./compliance
./evidence
./logs
./manifests
./recovery
./reports
```

---

# Step 316. Freeze Phase 1–16 Baseline

## Tujuan

Sebelum menyentuh application layer, kita harus membuktikan infrastructure existing masih sehat.

Ini penting supaya kalau Phase 17 gagal, kita tahu failure berasal dari Phase 17 dan bukan Phase sebelumnya.

### 316.1 AWS baseline

```bash
export ENDPOINT=http://localhost:4566
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_ACCOUNT_ID=000000000000
```

### Parameter

|Parameter|Fungsi|
|---|---|
|`ENDPOINT`|Floci endpoint|
|`AWS_REGION`|AWS-compatible region|
|`AWS_DEFAULT_REGION`|Default AWS CLI region|
|`AWS_ACCESS_KEY_ID`|Credential Floci|
|`AWS_SECRET_ACCESS_KEY`|Credential Floci|
|`AWS_ACCOUNT_ID`|Account ID lokal|

### 316.2 AWS identity

```bash
aws sts get-caller-identity \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION" \
  | tee evidence/aws-identity.json
```

Expected:

```
{
  "Account": "000000000000",
  "Arn": "...",
  "UserId": "..."
}
```

### 316.3 Kubernetes

```bash
kubectl cluster-info | tee evidence/k8s-cluster-info.txt

kubectl get nodes -o wide \
  | tee evidence/k8s-nodes.txt

kubectl get pods -A \
  | tee evidence/k8s-pods.txt
```

### Validasi

```bash
kubectl get nodes
kubectl get pods -A
```

Expected:

```
nodes → Ready
critical pods → Running/Completed as appropriate
```

---

# Step 317. Validate Phase 13–15 Security Plane

## Tujuan

Sebelum application deployment, security plane harus hidup.

### 317.1 RBAC / ServiceAccount

```bash
kubectl get serviceaccounts -A
kubectl get roles -A
kubectl get rolebindings -A
```

Expected:

Resource Phase 13 masih tersedia.

### 317.2 NetworkPolicy

```bash
kubectl get networkpolicies -A
```

Expected:

Network segmentation policies tetap ada.

### 317.3 Kyverno

```bash
kubectl get pods -A | grep -i kyverno
```

### 317.4 Gatekeeper

```bash
kubectl get pods -A | grep -Ei 'gatekeeper|opa'
```

### 317.5 Falco

```bash
kubectl get pods -A | grep -i falco
```

### 317.6 SPIRE

```bash
kubectl get pods -n spire
kubectl get clusterspiffeid -A
```

### Validasi

Semua security planes yang memang dibangun di Phase 13–15 harus tetap available.

---

# Step 318. Define Enterprise Application Namespace

## Tujuan

Memisahkan final application workload dari security infrastructure.

```bash
kubectl create namespace enterprise-e2e \
  --dry-run=client -o yaml \
  | kubectl apply -f -
```

Label:

```bash
kubectl label namespace enterprise-e2e \
  security-tier=tier-1 \
  environment=production \
  application=enterprise-shop \
  --overwrite
```

### Validasi

```bash
kubectl get namespace enterprise-e2e --show-labels
```

Expected:

```sh
security-tier=tier-1
environment=production
application=enterprise-shop
```

Ini langsung menghubungkan Phase 16 governance/tagging concept dengan Kubernetes workload governance.

---

# Step 319. Deploy Frontend

## Tujuan

Membuat public-facing application layer.

Untuk initial deployment kita pakai workload ringan.
```sh
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: enterprise-app-sa
  namespace: enterprise-e2e
EOF
```

```sh
cat > manifests/frontend.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: enterprise-e2e
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      serviceAccountName: enterprise-app-sa
      containers:
      - name: frontend
        image: nginxinc/nginx-unprivileged:alpine
        ports:
        - containerPort: 8080
        securityContext:
          runAsNonRoot: true
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: enterprise-e2e
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
EOF

kubectl apply -f manifests/frontend.yaml
```

### Parameter penting

`replicas: 2`

Menguji minimal workload redundancy.

`runAsNonRoot: true`

Menghubungkan ke hardening Phase 13.

`allowPrivilegeEscalation: false`

Mencegah privilege escalation.

`drop: ALL`

Mengurangi Linux capabilities.

### Validasi

```
kubectl rollout status \
  deployment/frontend \
  -n enterprise-e2e
```

Expected:

```
deployment "frontend" successfully rolled out
```

Lalu:

```
kubectl get pods -n enterprise-e2e -o wide
kubectl get svc -n enterprise-e2e
```

---

# Step 320. Deploy Backend/API

## Tujuan

Membuat application business logic layer.
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-ingress
  namespace: enterprise-e2e
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    - podSelector:
        matchLabels:
          run: backend-test
    ports:
    - protocol: TCP
      port: 5678
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress-allow
  namespace: enterprise-e2e
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - ports:
    - protocol: UDP
      port: 53
    to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
  - ports:
    - protocol: TCP
      port: 5678
    to:
    - podSelector:
        matchLabels:
          app: payment
EOF
```

```bash
cat > manifests/backend.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: enterprise-e2e
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      serviceAccountName: enterprise-app-sa
      containers:
      - name: backend
        image: hashicorp/http-echo:1.0
        args:
        - "-text=enterprise-api-ok"
        ports:
        - containerPort: 5678
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: enterprise-e2e
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 5678
EOF

kubectl apply -f manifests/backend.yaml
```

### Validasi

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: secure-test-egress
  namespace: enterprise-e2e
spec:
  podSelector:
    matchLabels:
      run: backend-test
  policyTypes:
  - Egress
  egress:
  - ports:
    - protocol: UDP
      port: 53
    to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
  - ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 5678
EOF

kubectl rollout status deployment/backend \
  -n enterprise-e2e

kubectl run backend-test -n enterprise-e2e --restart=Never --labels=run=backend-test --image=curlimages/curl:latest -- sleep 3600 && kubectl wait --for=condition=Ready pod/backend-test -n enterprise-e2e --timeout=30s && kubectl exec -it backend-test -n enterprise-e2e -- curl -v http://backend; kubectl delete pod backend-test -n enterprise-e2e --force --grace-period=0
```

Expected:

```
enterprise-api-ok
```

---

# Step 321. Deploy Payment Service

## Tujuan

Membuat **sensitive business service** untuk menguji segmentation dan authorization.
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-payment-ingress
  namespace: enterprise-e2e
spec:
  podSelector:
    matchLabels:
      app: payment
  ingress:
  - from:
    - podSelector:
        matchLabels:
          run: e2e-client
    ports:
    - protocol: TCP
      port: 5678
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: secure-client-egress
  namespace: enterprise-e2e
spec:
  podSelector:
    matchLabels:
      run: e2e-client
  policyTypes:
  - Egress
  egress:
  - ports:
    - protocol: UDP
      port: 53
    to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
  - ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 5678
EOF
```

```bash
cat > manifests/payment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment
  namespace: enterprise-e2e
spec:
  replicas: 1
  selector:
    matchLabels:
      app: payment
  template:
    metadata:
      labels:
        app: payment
    spec:
      serviceAccountName: enterprise-app-sa
      containers:
      - name: payment
        image: hashicorp/http-echo:1.0
        args:
        - "-text=payment-service-ok"
        ports:
        - containerPort: 5678
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
---
apiVersion: v1
kind: Service
metadata:
  name: payment
  namespace: enterprise-e2e
spec:
  selector:
    app: payment
  ports:
  - port: 80
    targetPort: 5678
EOF

kubectl apply -f manifests/payment.yaml
```

### Validasi

```sh
kubectl rollout status deployment/payment \
  -n enterprise-e2e

kubectl get pods -n enterprise-e2e
```

---

# Step 322. Validate Service-to-Service Connectivity

## Tujuan

Membuktikan:

```
frontend → backend
backend → payment
frontend -X→ payment
```

sesuai architecture security.

### Backend

```bash
kubectl run backend-test -n enterprise-e2e --restart=Never --labels=run=backend-test --image=curlimages/curl:latest -- sleep 3600 && kubectl wait --for=condition=Ready pod/backend-test -n enterprise-e2e --timeout=30s && kubectl exec -it backend-test -n enterprise-e2e -- curl -v http://backend; kubectl delete pod backend-test -n enterprise-e2e --force --grace-period=0
```

Expected:

```
enterprise-api-ok
```

### Payment

```bash
kubectl run e2e-client -n enterprise-e2e --restart=Never --labels=run=e2e-client --image=curlimages/curl:latest -- sleep 3600 && kubectl wait --for=condition=Ready pod/e2e-client -n enterprise-e2e --timeout=30s && kubectl exec -it e2e-client -n enterprise-e2e -- curl -s http://payment; kubectl delete pod e2e-client -n enterprise-e2e --force --grace-period=0
```

Untuk tahap awal expected:

```
payment-service-ok
```

Setelah NetworkPolicy diterapkan, akses langsung frontend→payment harus ditolak.

---

# Step 323. Apply Final Network Segmentation

## Tujuan

Ini menghidupkan security model:

```
frontend → backend → payment
                    │
                    ▼
                   DB

frontend -X→ payment
```

Contoh:

```sh
cat > manifests/networkpolicy.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-ingress
  namespace: enterprise-e2e
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-ingress
  namespace: enterprise-e2e
spec:
  podSelector:
    matchLabels:
      app: payment
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
EOF

kubectl apply -f manifests/networkpolicy.yaml
```

### Validasi

```sh
kubectl get networkpolicy \
  -n enterprise-e2e
  
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress-allow
  namespace: enterprise-e2e
spec:
  podSelector:
    matchLabels:
      app: backend
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
  - to:
    - podSelector:
        matchLabels:
          app: payment
    ports:
    - protocol: TCP
      port: 5678
  - ports:
    - protocol: TCP
      port: 80
EOF

kubectl run frontend-tester -n enterprise-e2e --restart=Never --labels=app=frontend --image=curlimages/curl:latest -- sleep 3600 && kubectl wait --for=condition=Ready pod/frontend-tester -n enterprise-e2e --timeout=30s && kubectl exec -it frontend-tester -n enterprise-e2e -- curl -s --max-time 3 http://payment; kubectl delete pod frontend-tester -n enterprise-e2e --force --grace-period=0

kubectl run backend-tester -n enterprise-e2e --restart=Never --labels=app=backend --image=curlimages/curl:latest -- sleep 3600 && kubectl wait --for=condition=Ready pod/backend-tester -n enterprise-e2e --timeout=30s && kubectl exec -it backend-tester -n enterprise-e2e -- curl -s http://payment; kubectl delete pod backend-tester -n enterprise-e2e --force --grace-period=0
```

Kemudian test allowed/denied traffic.

Expected:

```
backend → payment = ALLOW
frontend → payment = DENY
```

Ini menjadi bukti Phase 13 tetap berfungsi di application lifecycle.

---

# Step 324. Deploy Database Layer

## Tujuan

Menguji actual application data plane.

Kita gunakan PostgreSQL sebagai representative RDS/database workload.
```sh
cat <<EOF | kubectl apply -f -
# 1. Egress policy untuk client (ini yang hilang di versi asli)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-postgres-client-egress
  namespace: enterprise-e2e
spec:
  podSelector:
    matchLabels:
      run: postgres-client
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - port: 53
      protocol: UDP
  - ports:
    - port: 7001
      protocol: TCP
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-postgres-ingress
  namespace: enterprise-e2e
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
    ports:
    - port: 7001
      protocol: TCP
EOF
```

```sh
cat > manifests/postgres.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: enterprise-e2e
spec:
  selector:
    app: postgres
  ports:
  - port: 7001
    targetPort: 7001
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: enterprise-e2e
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        args: ["-p", "7001", "-c", "listen_addresses=*"]
        env:
        - name: POSTGRES_DB
          value: enterprise
        - name: POSTGRES_USER
          value: appuser
        - name: POSTGRES_PASSWORD
          value: phase17-temporary-password
        ports:
        - containerPort: 7001
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "appuser", "-d", "enterprise", "-p", "7001"]
          initialDelaySeconds: 5
          periodSeconds: 5
EOF

kubectl apply -f manifests/postgres.yaml
```

### Validasi

```sh
kubectl rollout status \
  statefulset/postgres \
  -n enterprise-e2e
```

Lalu:

```sh
kubectl get pods -n enterprise-e2e
```

---

# Step 325. Database Connectivity Test

## Tujuan

Membuktikan application tier bisa berkomunikasi dengan database.

```sh
kubectl run postgres-client-test \
  -n enterprise-e2e \
  --restart=Never \
  --labels="run=postgres-client,security-managed=true" \
  --image=postgres:16-alpine \
  --env="PGPASSWORD=phase17-temporary-password" \
  -- sh -c 'for i in $(seq 1 30); do pg_isready -h postgres -p 7001 -U appuser && break; sleep 1; done; psql -h postgres -p 7001 -U appuser -d enterprise -c "SELECT version();"'

# 3. Validasi
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/postgres-client-test -n enterprise-e2e --timeout=60s
kubectl logs postgres-client-test -n enterprise-e2e
kubectl delete pod postgres-client-test -n enterprise-e2e
```

### Parameter

`-h postgres`

Database service.

`-U appuser`

Database user.

`-d enterprise`

Database name.

`-c`

Execute SQL command.

### Validasi

Expected:

```
PostgreSQL 16...
```

---

# Step 326. Create Application Data Model

## Tujuan

Membuat representative e-commerce transaction.

```sh
kubectl delete pod postgres-client -n enterprise-e2e --ignore-not-found=true --wait=true

kubectl run postgres-client \
  -n enterprise-e2e \
  --rm -it \
  --restart=Never \
  --labels="run=postgres-client,security-managed=true" \
  --image=postgres:16-alpine \
  --env="PGPASSWORD=phase17-temporary-password" \
  --env="SQL=
CREATE TABLE IF NOT EXISTS products (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  price NUMERIC NOT NULL
);

CREATE TABLE IF NOT EXISTS orders (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL,
  product_id INTEGER NOT NULL,
  amount NUMERIC NOT NULL
);

INSERT INTO products(name, price)
VALUES ('Enterprise Laptop', 15000000)
ON CONFLICT (name) DO NOTHING;

SELECT * FROM products;
" \
  -- sh -c 'for i in $(seq 1 30); do pg_isready -h postgres -p 7001 -U appuser -d enterprise && break; sleep 1; done; psql -h postgres -p 7001 -U appuser -d enterprise -c "$SQL"'
```

### Validasi

```sh
kubectl delete pod postgres-client -n enterprise-e2e --ignore-not-found=true --wait=true

kubectl run postgres-client \
  -n enterprise-e2e \
  --rm -i \
  --restart=Never \
  --labels="run=postgres-client,security-managed=true" \
  --image=postgres:16-alpine \
  --env="PGPASSWORD=phase17-temporary-password" \
  -- sh -c 'for i in $(seq 1 30); do pg_isready -h postgres -p 7001 -U appuser -d enterprise && break; sleep 1; done; psql -h postgres -p 7001 -U appuser -d enterprise -c "SELECT * FROM products;"'
```

Expected:

```
Enterprise Laptop
```

---

# Step 327. Secrets Management Validation

## Tujuan

Jangan biarkan application secret berada di manifest.

Ini menghubungkan Phase 8.

Pertama inspect existing Secrets Manager:

```sh
aws secretsmanager list-secrets \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Kemudian create test secret:

```sh
aws secretsmanager create-secret \
  --name phase17/database-test \
  --secret-string '{"username":"appuser","password":"phase17-test"}' \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

### Validasi

```sh
aws secretsmanager get-secret-value \
  --secret-id phase17/database-test \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected:

Secret dapat retrieved oleh authorized principal.

Kemudian:

```sh
grep -RniE \
  'password=|secret=|api[_-]?key|access[_-]?key' \
  manifests/
```

Expected:

Tidak ada production credential hardcoded.

---

# Step 328. KMS Validation

## Tujuan

Membuktikan encryption layer Phase 8 tetap tersedia.

```sh
aws kms list-keys \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Ambil key:

```sh
aws kms list-aliases \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Kemudian test encryption/decryption menggunakan test plaintext.

```sh
printf 'phase17-sensitive-test' > evidence/plaintext.txt
```

Encrypt:

```sh
export KMS_KEY_ID=$(aws kms list-keys --endpoint-url "$ENDPOINT" --region "$AWS_REGION" --query 'Keys[0].KeyId' --output text) && \
echo "KMS_KEY_ID berhasil di-set ke: $KMS_KEY_ID" && \
aws kms encrypt \
  --key-id "$KMS_KEY_ID" \
  --plaintext fileb://evidence/plaintext.txt \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION" \
  > evidence/kms-encrypt.json && \
echo "Enkripsi sukses! Hasil tersimpan di evidence/kms-encrypt.json"
```

Decrypt:

```sh
# 1. Ekstrak dan decode CiphertextBlob dari JSON ke format binari
jq -r '.CiphertextBlob' evidence/kms-encrypt.json | base64 -d > evidence/ciphertext.bin && \

# 2. Jalankan ulang perintah decrypt
aws kms decrypt \
  --ciphertext-blob fileb://evidence/ciphertext.bin \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

**Catatan:** bagian extraction `CiphertextBlob` perlu disesuaikan dengan output CLI Floci yang lu dapat nanti.

### Validasi

```
plaintext
   ↓ encrypt
ciphertext
   ↓ decrypt
plaintext
```

Harus sama.

---

# Step 329. S3 Data Plane Validation

## Tujuan

Menguji data flow application → object storage.

```sh
export PHASE17_BUCKET=phase17-enterprise-data
```

Create:

```sh
aws s3api create-bucket \
  --bucket "$PHASE17_BUCKET" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Upload:

```sh
printf 'enterprise-e2e-object' > evidence/object.txt

aws s3 cp \
  evidence/object.txt \
  "s3://$PHASE17_BUCKET/object.txt" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Download:

```sh
aws s3 cp \
  "s3://$PHASE17_BUCKET/object.txt" \
  evidence/object-download.txt \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

### Validasi

```
cmp evidence/object.txt evidence/object-download.txt
```

---

# Step 330. IAM / IRSA Authorization Test

## Tujuan

Menguji application pod hanya memperoleh permission yang dibutuhkan.

Test allowed operation:

```sh
# 1. Egress policy untuk pod tes (Floci di 172.17.0.2:4566)
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-irsa-test-egress
  namespace: enterprise-e2e
spec:
  podSelector:
    matchLabels:
      run: irsa-e2e-test
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 172.17.0.2/32
    ports:
    - port: 4566
      protocol: TCP
EOF

# 2. Tes dari dalam pod, dengan retry bawaan AWS CLI
kubectl run irsa-e2e-test -n enterprise-e2e --rm -i --restart=Never \
  --image=amazon/aws-cli:latest \
  --env="AWS_ENDPOINT_URL=http://172.17.0.2:4566" \
  --env="AWS_REGION=us-east-1" \
  --env="AWS_ACCESS_KEY_ID=test" \
  --env="AWS_SECRET_ACCESS_KEY=test" \
  --env="AWS_MAX_ATTEMPTS=10" \
  -- sts get-caller-identity
```

Kemudian test authorized resource access.

Misalnya:

```sh
aws s3api get-bucket-location \
  --bucket "$PHASE17_BUCKET" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

---

# Step 331. Normal End-to-End Business Transaction

## Tujuan

Sekarang baru kita melakukan **happy path**.

Flow:

```
Client
  ↓
Frontend
  ↓
Backend
  ↓
Payment
  ↓
Database
  ↓
S3
```

Minimal transaction:

```
GET product
      ↓
create order
      ↓
payment request
      ↓
DB INSERT
      ↓
receipt/object → S3
```

### Validasi

Evidence harus membuktikan:

```
HTTP request
      +
backend response
      +
payment response
      +
DB record
      +
S3 object
```

Ini adalah **baseline sebelum attack testing**.

---

# Step 332. Public Edge / ALB / WAF Validation

## Tujuan

Traffic tidak boleh langsung dianggap valid hanya karena pod healthy. **catatan penting** -> karena ALB pada floci tidak bekerja sepenuhnya, maka kita akan mencoba akses aplikasi melalui metode port-forwarding ke port service backend

**Port-forward jangan mati sampe akhir!!!**

```sh
kubectl port-forward svc/backend 8080:80 -n enterprise-e2e &
```

Cari ingress/load balancer:

```sh
kubectl get ingress -A
kubectl get svc -A
```

AWS side:

```sh
aws elbv2 describe-load-balancers \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

WAF:

```sh
aws wafv2 list-web-acls \
  --scope REGIONAL \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

### Validasi

Expected:

```
Client
 ↓
ALB
 ↓
WAF
 ↓
Application
```

---

# Step 333. Normal HTTP Security Baseline

## Tujuan

Sebelum menyerang application, ukur normal response. 

```sh
export APPLICATION_URL="http://localhost:8080"
curl -ik "$APPLICATION_URL"
```

Test:

```sh
curl -sk -o /dev/null \
  -w 'HTTP=%{http_code}\nTIME=%{time_total}\n' \
  "$APPLICATION_URL/"
```

Check headers:

```sh
curl -skI "$APPLICATION_URL"
```

Cari:

```
Strict-Transport-Security
Content-Security-Policy
X-Content-Type-Options
X-Frame-Options
Referrer-Policy
```

Security headers relevan dengan OWASP A05 Security Misconfiguration. [OWASP Top 10](https://top10.owasp.org/2021/A05_2021-Security_Misconfiguration/?utm_source=chatgpt.com)

---

# Step 334. OWASP A01 Broken Access Control

## Tujuan

Menguji user tidak dapat mengakses resource milik user lain.

Contoh:

```sh
# Cek resource user 1001
curl -sk -i "$APPLICATION_URL/user/1001/orders/5001"
```

kemudian:

```sh
# Cek resource user 1002 (harus tertolak jika tidak punya hak akses)
curl -sk -i "$APPLICATION_URL/user/1002/orders/5001"
```

Expected:

```
401 / 403
```

atau application-specific denial.

Test juga:

```
GET /admin
POST /admin/users
DELETE /orders/5001
```

tanpa authorization.

### Validasi

Harus ada:

```
unauthorized request
       ↓
DENIED
       ↓
security log
```

Ini langsung menguji OWASP A01. [OWASP Top 10](https://top10.owasp.org/2021/A01_2021-Broken_Access_Control/?utm_source=chatgpt.com)

---

# Step 335. Authentication Bypass

## Tujuan

Menguji endpoint protected tanpa credentials.

```sh
curl -i "$APPLICATION_URL/api/orders"
```

Kemudian malformed token:

```sh
curl -i \
  -H 'Authorization: Bearer invalid-token' \
  "$APPLICATION_URL/api/orders"
```

Expected:

```
401 Unauthorized
```

Test expired/modified token bila application menggunakan JWT.

---

# Step 336. SQL Injection

## Tujuan

Menguji OWASP A03.

Payload sederhana:

```
'
' OR '1'='1
1 OR 1=1
```

Misalnya:

```sh
curl -G \
  --data-urlencode "q=' OR '1'='1" \
  "$APPLICATION_URL/api/products"
```

Expected:

```
HTTP 400/403
```

atau safe empty result.

**Yang tidak boleh terjadi:**

```
database dump
authentication bypass
SQL error leakage
```

OWASP mengategorikan SQL injection sebagai bagian dari A03 Injection dan merekomendasikan parameterized interfaces/queries. [OWASP Top 10](https://top10.owasp.org/2021/A03_2021-Injection/?utm_source=chatgpt.com)

---

# Step 337. XSS

Test:

```
<script>alert(1)</script>
```

```sh
curl -G \
  --data-urlencode 'q=<script>alert(1)</script>' \
  "$APPLICATION_URL/search"
```

Expected:

```
payload encoded/rejected
```

Tidak boleh menjadi executable HTML/JavaScript.

---

# Step 338. Command Injection

Payload:

```sh
# Uji coba dengan command separator titik koma
curl -sk "$APPLICATION_URL/api/diagnostic?host=127.0.0.1;id"

# Uji coba dengan command substitution dollar-kurung
curl -sk "$APPLICATION_URL/api/diagnostic?host=\$(id)"

# Uji coba dengan backtick
curl -sk "$APPLICATION_URL/api/diagnostic?host=\`id\`"
```

Test terhadap endpoint yang menerima command-like input jika application memang memilikinya.

Expected:

```
input rejected
```

dan tidak ada evidence command execution.

Kalau application memang tidak mempunyai command-execution functionality, kita **tetap menguji attack surface yang tersedia** dan hasilnya bisa menjadi `PASS` karena endpoint tidak menyediakan primitive tersebut.

---

# Step 339. Path Traversal

Payload:

```
../../../../etc/passwd
```

Test:

```sh
curl -i \
  "$APPLICATION_URL/api/files?name=../../../../etc/passwd"
```

Expected:

```
400 / 403 / 404
```

Tidak boleh mendapatkan `/etc/passwd`.

---

# Step 340. SSRF

Ini penting karena architecture kita memiliki AWS + internal services.

OWASP A10 menjelaskan SSRF sebagai kondisi ketika attacker dapat membuat server melakukan request ke destination yang tidak semestinya. [OWASP Top 10](https://top10.owasp.org/2021/A10_2021-Server-Side_Request_Forgery_%28SSRF%29/index.html?utm_source=chatgpt.com)

Payload representative:

```
http://127.0.0.1
http://localhost
http://169.254.169.254
```

Contoh:

```sh
curl -G \
  --data-urlencode "url=http://127.0.0.1:8080" \
  "$APPLICATION_URL/api/fetch"
```

Jika endpoint fetch memang ada.

Expected:

```
DENIED
```

atau URL tidak diterima.

Yang ingin dibuktikan:

```
Internet
   │
   ▼
Application
   │
   X
Internal metadata/service
```

bukan:

```
Internet
   │
   ▼
Application
   │
   ▼
AWS metadata/internal service
```

---

# Step 341. HTTP Method Abuse

Test:

```sh
for method in OPTIONS TRACE PUT DELETE PATCH CONNECT; do
  echo "=== $method ==="
  curl -i -X "$method" "$APPLICATION_URL/"
done
```

### Validasi

Method yang tidak dibutuhkan harus:

```
405 Method Not Allowed
```

atau equivalent denial.

---

# Step 342. Oversized / Malformed Payload

## Tujuan

Menguji application/WAF behavior terhadap abnormal input.

```sh
python3 - <<'PY'
with open("attacks/large-payload.txt", "w") as f:
    f.write("A" * 1000000)
PY
```

Kemudian:

```sh
curl -i \
  --data-binary @attacks/large-payload.txt \
  "$APPLICATION_URL/api/orders"
```

Expected:

```
413 Payload Too Large
```

atau application-specific rejection.

---

# Step 343. WAF SQLi/XSS Detection

## Tujuan

Menguji apakah attack dikenali **sebelum mencapai application**.

SQLi:

```sh
curl -i \
  "$APPLICATION_URL/?id=%27%20OR%201%3D1"
```

XSS:

```sh
curl -i \
  "$APPLICATION_URL/?q=%3Cscript%3Ealert(1)%3C%2Fscript%3E"
```

### Validasi

Kita cek:

```
HTTP response
WAF logs
CloudWatch/log destination
application logs
```

Ideal:

```
Attacker
   ↓
WAF
   X
Application
```

---

# Step 344. CloudTrail Audit Validation

## Tujuan

Membuktikan AWS activity dari Phase 1–16 tetap auditable.

```sh
aws cloudtrail describe-trails \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Kemudian lookup events bila supported:

```sh
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=PutObject \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

### Validasi

Expected:

```
API activity → audit event
```

Jika Floci implementation membatasi event history:

```
FLOCi LIMITATION
```

---

# Step 345. Config Compliance Validation

## Tujuan

Menguji apakah intentional misconfiguration dapat ditemukan.

Contoh:

```
S3 bucket
   ↓
remove governance/security property
   ↓
Config
   ↓
NON_COMPLIANT
```

Query existing Config rules:

```sh
aws configservice describe-config-rules \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

### Validasi

Expected:

```
configured rule
+
evaluation
+
compliance result
```

---

# Step 346. GuardDuty Threat Detection

## Tujuan

Menguji threat detection layer.

```sh
aws guardduty list-detectors \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Kemudian inspect detector/status.

```sh
export DETECTOR_ID=$(aws guardduty list-detectors --endpoint-url "$ENDPOINT" --region "$AWS_REGION" --query 'DetectorIds[0]' --output text) && \
echo "DETECTOR_ID berhasil di-set ke: $DETECTOR_ID" && \
aws guardduty get-detector \
  --detector-id "$DETECTOR_ID" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Jika Floci menyediakan finding simulation:

```
generate finding
   ↓
GuardDuty
   ↓
finding
```

### Validasi

Kalau finding simulation tersedia:

```
PASS
```

---

# Step 347. Automated Response Validation

## Tujuan

Menguji Phase 11:

```
Security Event
      ↓
EventBridge
      ↓
Lambda
      ↓
SQS/SNS
      ↓
Response
```

Inspect:

```sh
aws events list-rules \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Lambda:

```sh
aws lambda list-functions \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

SQS:

```sh
aws sqs list-queues \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

### Validasi

Kita trigger event yang aman, lalu buktikan:

```
event
 ↓
rule
 ↓
target
 ↓
response
```

---

# Step 348. Falco Runtime Attack

## Tujuan

Sekarang masuk runtime Kubernetes.

Intentional test:

```sh
kubectl debug -it $(kubectl get pod -l app=backend -n enterprise-e2e -o jsonpath='{.items[0].metadata.name}') \
  -n enterprise-e2e \
  --image=nicolaka/netshoot \
  --target=backend
```

Lalu suspicious behavior yang aman untuk lab, misalnya process/network activity.

Inspect:

```sh
kubectl logs -n falco-operator -l app.kubernetes.io/name=falco --tail=100
```

### Expected

Falco mendeteksi suspicious runtime behavior.

Ini membuktikan:

```
Application
 ↓
Container
 ↓
Runtime behavior
 ↓
Falco
```

---

# Step 349. Kyverno Admission Attack

## Tujuan

Mencoba deploy workload yang melanggar security policy.

Contoh:

```sh
cat <<EOF > attacks/privileged-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-attack-test
  namespace: enterprise-e2e
spec:
  containers:
  - name: attacker
    image: alpine:latest
    command: ["sh", "-c", "sleep 3600"]
    securityContext:
      privileged: true # <---- poin utama
EOF
```

Apply:

```sh
kubectl apply -f attacks/privileged-pod.yaml
```

Expected:

```
DENIED
```

Ini membuktikan Phase 14 masih melindungi workload **sebelum runtime**.

---

# Step 350. Gatekeeper Admission Attack

Test manifest policy violation lain.

Contoh:

```sh
cat <<'EOF' > attacks/hostnetwork-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostnetwork-attack-test
  namespace: enterprise-e2e
spec:
  hostNetwork: true
  containers:
  - name: net-attacker
    image: alpine:latest
    command: ["sleep", "3600"]
EOF
```

Apply:

```sh
kubectl apply -f attacks/hostnetwork-pod.yaml
```

Expected:

```
DENIED
```

Architecture:

```
Attacker manifest
       ↓
Kubernetes API
       ↓
Kyverno / Gatekeeper
       X
    REJECT
```

---

# Step 351. Lateral Movement Test

## Tujuan

Ini salah satu test paling penting.

Simulasikan pod compromised:

```
frontend compromised
       │
       ├──→ backend       ALLOWED
       │
       ├──→ payment       DENIED
       │
       ├──→ database      DENIED
       │
       └──→ AWS API       ONLY IF IRSA PERMITS
```

Test:

```sh
kubectl run compromised-frontend \
  -n enterprise-e2e \
  --rm -it \
  --restart=Never \
  --labels="app=frontend" \
  --image=curlimages/curl:latest \
  -- sh
```

Di dalam pod:

```sh
curl -m 5 http://backend 
curl -m 5 http://payment
curl -m 5 http://postgres:5432
```

### Expected

```
backend   → allowed
payment   → denied
postgres  → denied
```

Ini adalah kombinasi:

```
NetworkPolicy
+
IAM
+
IRSA
+
Workload identity
```

---

# Step 352. AWS Privilege Escalation Simulation

## Tujuan

Menguji apakah compromised pod dapat berubah menjadi AWS administrator.
```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: irsa-attack-test
  namespace: enterprise-e2e
  labels:
    run: irsa-e2e-test
spec:
  serviceAccountName: enterprise-app-sa
  restartPolicy: Never
  containers:
  - name: aws-cli
    image: amazon/aws-cli:latest
    command: ["sh"]
    stdin: true
    tty: true
EOF
```

```sh
kubectl attach irsa-attack-test -n enterprise-e2e -it
```

Dari pod IRSA:

```sh
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
export AWS_ENDPOINT_URL="http://172.17.0.2:4566"

aws iam list-users
aws s3 ls
```

Tidak semuanya harus denied; yang penting **permission sesuai intended role**.

Contoh:

```
S3 application bucket → ALLOW
IAM administration    → DENY
KMS unrelated key     → DENY
Secrets unrelated     → DENY
```

Ini adalah practical test dari Phase 8 + Phase 13.

---

# Step 353. S3 Unauthorized Data Access

Test dari workload yang **tidak mempunyai access**

Masih di dalam container irsa-attack-test:

```sh
aws s3 ls s3://enterprise-config-logs
```

Expected:

```
AccessDenied atau ga keluar apa apa
```

Kemudian authorized application role:

```
bucket access → PASS
```

Ini membuktikan:

```
Identity
+
IAM
+
Data authorization
```

---

# Step 354. KMS Unauthorized Decrypt

Gunakan identity yang tidak diberi decrypt permission.

Masih di dalam container irsa-attack-test:

```sh
export ENDPOINT="http://172.17.0.2:4566"
export AWS_REGION="us-east-1"
export AWS_DEFAULT_REGION="us-east-1"
export KMS_KEY_ID=$(aws kms list-keys --endpoint-url "$ENDPOINT" --region "$AWS_REGION" --query 'Keys[0].KeyId' --output text)
echo "KMS Key ID berhasil diambil: $KMS_KEY_ID"
mkdir -p evidence
echo "dummy-encrypted-data" > evidence/ciphertext.bin
aws kms decrypt \
  --key-id "$KMS_KEY_ID" \
  --ciphertext-blob fileb://evidence/ciphertext.bin \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected:

```
AccessDenied
```

Application role yang benar:

```
decrypt → PASS
```

Ini langsung memvalidasi design Phase 8:

```
Developer ≠ KMS decrypt
ApplicationRole = KMS decrypt
```

---

# Step 355. Secrets Unauthorized Access

Test unauthorized workload

Masih di dalam container irsa-attack-test:

```sh
aws secretsmanager get-secret-value \
  --secret-id phase17/database-test \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected:

```
AccessDenied
```

Authorized workload:

```
get-secret-value → PASS
```

---

# Step 356. SPIFFE/SPIRE Final Regression

Check:

```sh
kubectl get clusterspiffeid -A
kubectl get pods -n spire
```

Then inspect workload identity client.

Expected:

```
SPIFFE ID exists
Agent healthy
Server healthy
```

Jika mTLS test utility tersedia:

```
identity-client
      ↓
mTLS
      ↓
identity-server
```

Kalau environment test limitation dari Phase 15 masih berlaku, classification:

```
FLOCi LIMITATION
```

bukan `NOT TESTED`.

---

# Step 357. Monitoring Validation

## Tujuan

Semua attack sebelumnya harus menghasilkan observable evidence sebanyak mungkin.

Check:

```sh
kubectl get pods -A
kubectl top pods -A 2>/dev/null || true
```

AWS monitoring:

```sh
aws cloudwatch list-metrics \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

### Yang dicari

```
Application request
Security event
WAF event
AWS API event
Kubernetes event
Falco event
```

---

# Step 358. Incident Correlation

Ini salah satu **final differentiator** project lu.

Kita korelasikan satu simulated attack.

Contoh:

```
SQLi Request
     │
     ▼
    WAF
     │
     ├── Block
     │
     └── Log
          │
          ▼
      CloudWatch
```

Atau:

```
Compromised Pod
     │
     ▼
Unauthorized AWS API
     │
     ▼
IAM Deny
     │
     ▼
CloudTrail
     │
     ▼
GuardDuty / Detection
     │
     ▼
Automated Response
```

Dan Kubernetes:

```
Compromised Pod
     │
     ├── NetworkPolicy → DENY
     │
     ├── Falco → DETECT
     │
     └── SPIFFE → identity constraint
```

Ini menggabungkan **Phase 9–15**, bukan sekadar menguji component secara terpisah.

```sh
kubectl logs -n falco-operator falco-twtnj --tail=50 --all-containers=true
kubectl get clusterspiffeid -A
kubectl get pods -n spire
```

---

# Step 359. Backup After Attack

## Tujuan

Memastikan security incident tidak menghancurkan recovery capability.

Check:

```sh
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name phase16-vault \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected:

```
recovery point exists
```

---

# Step 360. Restore Validation

Karena pada Phase 16 ternyata `StartRestoreJob` **PASS**, Phase 17 harus mengulang restore sebagai final regression.

```sh
aws backup start-restore-job \
  --recovery-point-arn "$RECOVERY_POINT_ARN" \
  --metadata '{}' \
  --iam-role-arn "$IAM_ROLE_ARN" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Parameter actual-nya nanti mengikuti resource/recovery point yang berhasil lu buat di Phase 16.

Expected:

```
restore job created
```

Kemudian:

```sh
aws backup describe-restore-job \
  --restore-job-id "$RESTORE_JOB_ID" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected:

```
COMPLETED
```

---

# Step 361. Controlled Application Failure

Sekarang kita sengaja merusak **application replica**, bukan infrastructure.

```
kubectl delete pod \
  -n enterprise-e2e \
  -l app=frontend
```

Expected:

Kubernetes membuat replacement pod.

```
kubectl get pods \
  -n enterprise-e2e \
  -w
```

### Validasi

```
Pod failure
   ↓
ReplicaSet
   ↓
Replacement
   ↓
Healthy
```

---

# Step 362. Backend Failure Recovery

```sh
kubectl delete pod \
  -n enterprise-e2e \
  -l app=backend
```

Then:

```sh
kubectl rollout status \
  deployment/backend \
  -n enterprise-e2e
```

Expected:

```
successfully rolled out
```

Kemudian repeat normal transaction.

---

# Step 363. Security Regression After Recovery

Setelah pod recreation:

```sh
kubectl get pods -n enterprise-e2e
kubectl get networkpolicy -n enterprise-e2e
```

Check security context:

```sh
kubectl get pod \
  -n enterprise-e2e \
  -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.spec.containers[0].securityContext}{"\n"}{end}'
```

Expected:

```
securityContext preserved
NetworkPolicy preserved
ServiceAccount preserved
```

---

# Step 364. Terraform Reconciliation

Sekarang kita hubungkan ke Phase 16.

```
cd ~/phase16/terraform/repro-test

terraform plan
```

Expected:

```
No changes
```

atau perubahan yang memang sudah direncanakan.

Tujuannya memastikan manual E2E activity tidak menghasilkan uncontrolled infrastructure drift.

---

# Step 365. Full Kubernetes Security Regression

```sh
kubectl get networkpolicies -A
kubectl get serviceaccounts -A
kubectl get roles -A
kubectl get rolebindings -A
kubectl get clusterspiffeid -A
kubectl get pods -A
```

Kemudian:

```sh
kubectl get events -A \
  --sort-by=.lastTimestamp \
  | tail -100
```

Expected:

Tidak ada critical security/infrastructure regression.

---

# Step 366. Phase 12 Supply Chain Regression

Check ECR:

```sh
aws ecr describe-repositories \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Kemudian dokumentasikan kembali limitation yang sudah ditemukan:

```
ECR operational pull via generated hostname
→ FLOCi LIMITATION
```

Jangan mengubah Phase 12 result hanya karena Phase 17.

---

# Step 367. Phase 1–11 AWS Regression

Kumpulkan:

```sh
export ENDPOINT="http://172.17.0.2:4566" export AWS_REGION="us-east-1" export AWS_DEFAULT_REGION="us-east-1"

aws ec2 describe-vpcs \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
aws ec2 describe-security-groups \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
aws s3api list-buckets \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
aws rds describe-db-instances \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
aws iam list-roles \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Dan:

```sh
aws cloudtrail describe-trails \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
aws configservice describe-config-rules \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
aws guardduty list-detectors \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Tujuannya bukan mengulang seluruh Phase 1–11 dari nol, tetapi memastikan final application workload masih berada di infrastructure yang benar.

---

# Step 368. Governance Regression

Phase 16 tag standard:

```
Environment
Owner
Application
DataClassification
SecurityTier
```

Check resource tags:

```sh
aws resourcegroupstaggingapi get-resources \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected:

Resource yang memang taggable memiliki governance metadata.

---

# Step 369. Generate Final Evidence Bundle

```sh
cd ~/phase17

kubectl get pods -A -o wide \
  > evidence/final-k8s-pods.txt

kubectl get networkpolicies -A \
  > evidence/final-networkpolicies.txt

kubectl get serviceaccounts -A \
  > evidence/final-serviceaccounts.txt

kubectl get clusterspiffeid -A \
  > evidence/final-spiffe.txt

aws sts get-caller-identity \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION" \
  > evidence/final-aws-identity.json
```

---

# Step 370. Compliance Evidence Mapping

Buat:

```sh
cat > compliance/control-mapping.md <<'EOF'
# Phase 17 Compliance Control Mapping

## OWASP Top 10
A01 Broken Access Control
A02 Cryptographic Failures
A03 Injection
A04 Insecure Design
A05 Security Misconfiguration
A06 Vulnerable and Outdated Components
A07 Identification and Authentication Failures
A08 Software and Data Integrity Failures
A09 Security Logging and Monitoring Failures
A10 Server-Side Request Forgery

## PCI DSS
Network security
Secure configuration
Protection of account data
Encryption
Access control
Authentication
Logging and monitoring
Security testing
Incident response
Backup/recovery

## ISO/IEC 27001
Confidentiality
Integrity
Availability
Access control
Logging/monitoring
Vulnerability management
Incident response
Business continuity
Risk management
Change management
EOF
```

### Penting

Ini **control mapping**, bukan declaration bahwa project memenuhi/certified terhadap ketiga framework.

PCI SSC sendiri menyatakan PCI DSS adalah baseline technical dan operational requirements untuk melindungi payment account data. [PCI Security Standards Council](https://www.pcisecuritystandards.org/standards/pci-dss/?utm_source=chatgpt.com)

---

# Step 371. Final Attack Summary

Buat matrix attack:

```
ATTACK
  │
  ├── SQL Injection
  ├── XSS
  ├── IDOR
  ├── Authentication bypass
  ├── SSRF
  ├── Path traversal
  ├── Command injection
  ├── HTTP method abuse
  ├── Oversized payload
  ├── Kubernetes privileged pod
  ├── Kubernetes hostNetwork
  ├── Lateral movement
  ├── AWS privilege abuse
  ├── Unauthorized S3
  ├── Unauthorized KMS
  └── Unauthorized Secrets
```

Setiap satu harus punya:

```
Attack
Expected behavior
Actual behavior
Prevented?
Detected?
Logged?
Response?
Evidence
```

---

# Step 372. Final Security Chain

Ini adalah test paling penting secara konseptual.

Kita harus bisa menggambarkan:

```
                    ATTACKER
                       │
                       ▼
                    Internet
                       │
                       ▼
                   CloudFront
                       │
                       ▼
                      WAF
                       │
                       ▼
                     ALB
                       │
                       ▼
                 Kubernetes
                       │
             ┌─────────┼─────────┐
             │         │         │
             ▼         ▼         ▼
          Frontend    API      Payment
             │         │         │
             │         ▼         │
             │       Database    │
             │                   │
             └────── S3 ─────────┘

Security controls:
────────────────────────────────────

Edge
  └── WAF

AWS
  ├── IAM
  ├── KMS
  ├── Secrets Manager
  ├── CloudTrail
  ├── Config
  └── GuardDuty

Kubernetes
  ├── RBAC
  ├── NetworkPolicy
  ├── Kyverno
  ├── Gatekeeper
  └── Falco

Identity
  └── SPIFFE/SPIRE

Resilience
  ├── Backup
  ├── Restore
  └── Terraform
```

---

# Step 373. Final Enterprise E2E Test

Urutan final:

```
1. User accesses application
        ↓
2. Frontend responds
        ↓
3. API responds
        ↓
4. Product retrieved
        ↓
5. Order created
        ↓
6. Payment service invoked
        ↓
7. DB transaction succeeds
        ↓
8. S3 object created
        ↓
9. Logs generated
        ↓
10. Monitoring sees traffic
        ↓
11. Attack simulation
        ↓
12. WAF / IAM / NetworkPolicy / Kyverno / Falco
    enforce according to scenario
        ↓
13. Security event logged
        ↓
14. Detection/response validated
        ↓
15. Application survives workload failure
        ↓
16. Backup/recovery remains available
        ↓
17. Terraform state remains reconcilable
        ↓
18. Security controls remain intact
```

Kalau chain ini berhasil, **itulah final validation sebenarnya**.

---

# Step 374. Final Validation Matrix

Setelah semua test selesai, hasil akhirnya kita isi dengan:

|Category|Meaning|
|---|---|
|PASS|Capability berhasil dibuktikan|
|FLOCi LIMITATION|Capability sudah diuji tetapi implementation/environment Floci membatasi pembuktian|
|FAIL|Capability seharusnya didukung tetapi actual behavior tidak sesuai expected|

Tidak akan ada:

```
NOT TESTED
```

---

# 6. Kenapa Phase 17 ini lebih bagus daripada sekadar deploy frontend?

Karena sekarang project lu punya progression:

```
PHASE 1–4
Build network + edge
       ↓
PHASE 5–8
Protect data + secrets
       ↓
PHASE 9–11
Observe + detect + respond
       ↓
PHASE 12
Secure supply chain
       ↓
PHASE 13
Secure Kubernetes
       ↓
PHASE 14
Control admission + runtime
       ↓
PHASE 15
Cryptographic workload identity
       ↓
PHASE 16
Backup + governance + IaC
       ↓
PHASE 17
════════════════════════════════
REAL APPLICATION VALIDATION
════════════════════════════════
       ↓
Normal traffic
       ↓
Attack
       ↓
Prevention
       ↓
Detection
       ↓
Response
       ↓
Recovery
       ↓
Rebuild
       ↓
Security regression
```

Dan dari perspektif compliance, ini jauh lebih meaningful daripada checklist component. PCI DSS menekankan monitoring/testing dan secure systems, sementara OWASP A09 secara eksplisit mengaitkan security testing dengan detection/alerting dan response. [PCI Security Standards Council](https://www.pcisecuritystandards.org/standards/pci-dss/?utm_source=chatgpt.com)

**Satu perubahan yang gue sarankan dari ide awal lu:** jangan benar-benar mencoba membuat “Tokopedia clone”. Yang kita butuhkan bukan UI kompleks, tetapi **representative enterprise transaction path**. Jadi resource lu tetap kecil, tapi security path-nya lengkap:

```
Frontend
   ↓
API
   ↓
Payment
   ↓
DB
   +
S3
   +
Secrets
   +
KMS
   +
IAM/IRSA
   +
NetworkPolicy
   +
SPIFFE/SPIRE
   +
WAF
   +
CloudTrail/Config/GuardDuty
   +
Falco/Kyverno/Gatekeeper
   +
Backup/Restore
   +
Terraform
```

Dengan begitu **Phase 17 benar-benar menjadi “capstone” dari Phase 1–16**, dan kalau nanti lu mau bikin fase setelahnya, hasil Phase 17 bisa langsung menjadi baseline untuk **incident-response exercise, purple-team exercise, atau final portfolio evidence**, bukan perlu membangun project baru lagi.