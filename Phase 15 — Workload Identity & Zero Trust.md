## 1. Phase ini ngapain?

Phase 15 menambahkan **identity cryptographic untuk workload** di atas security control yang sudah dibangun pada Phase 13 dan Phase 14.

Sebelum Phase 15, workload sudah mempunyai beberapa bentuk identity dan security control:

```
Kubernetes ServiceAccount
        │
        ├── RBAC
        │
        └── IRSA
```

dan:

```
NetworkPolicy
securityContext
Kyverno
Gatekeeper
Falco
```

Tetapi masih ada gap:

> Kubernetes mengetahui bahwa request berasal dari Pod/ServiceAccount tertentu, tetapi service-to-service communication belum menggunakan identity cryptographic yang dapat diverifikasi oleh service tujuan.

Phase 15 mengisi gap tersebut dengan:

```
Kubernetes Workload
       │
       ▼
SPIRE Workload Attestation
       │
       ▼
SPIFFE ID
       │
       ▼
X.509-SVID
       │
       ▼
Trust Bundle
       │
       ▼
mTLS
       │
       ▼
Authenticated Service-to-Service Communication
```

Jadi tujuan akhirnya bukan sekadar:

```
frontend IP → payment IP
```

tetapi:

```
spiffe://example.org/ns/frontend/sa/frontend
                    │
                    │ mTLS
                    ▼
spiffe://example.org/ns/payment/sa/payment-api
```

Service tujuan dapat memverifikasi **identity workload**, bukan hanya alamat jaringan.

---

# 2. Komponen yang dibangun

### SPIFFE

SPIFFE adalah **standard workload identity**.

SPIFFE mendefinisikan konsep seperti:

- SPIFFE ID
- SVID
- Workload API
- Trust Domain
- X.509-SVID
- JWT-SVID

Contoh identity:

```
spiffe://example.org/ns/phase15-identity/sa/identity-client
```

Formatnya:

```
spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>
```

---

### SPIRE

SPIRE adalah implementation dari SPIFFE.

Arsitekturnya:

```
                    SPIRE Server
                         │
                         │
                  Trust Authority
                         │
                         ▼
                  SPIRE Agent
                         │
                 Workload API
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
      identity-client         identity-server
             │                       │
             ▼                       ▼
        X.509-SVID              X.509-SVID
```

SPIRE Server bertindak sebagai identity authority, sedangkan SPIRE Agent berjalan di node dan menyediakan Workload API untuk workload.

---

### SVID

SVID adalah credential yang membawa SPIFFE identity.

Untuk Phase 15 kita terutama menggunakan:

```
X.509-SVID
```

Karena X.509-SVID dapat digunakan untuk mutual TLS.

Contohnya:

```
Certificate
    │
    └── URI SAN:
        spiffe://example.org/ns/phase15-identity/sa/identity-client
```

---

# 3. Arsitektur Phase 15

Arsitektur aktual Phase 15:

```
                         KUBERNETES / EKS
                               │
                               │
                    ┌──────────▼──────────┐
                    │   Kubernetes API    │
                    └──────────┬──────────┘
                               │
                               │
              ┌────────────────▼────────────────┐
              │          SPIRE CONTROL PLANE     │
              │                                  │
              │  ┌────────────────────────────┐  │
              │  │       SPIRE Server         │  │
              │  │                            │  │
              │  │ Trust Domain: example.org  │  │
              │  │ Registration               │  │
              │  │ CA / Trust Authority       │  │
              │  └─────────────┬──────────────┘  │
              │                │                 │
              │        Controller Manager        │
              └────────────────┼─────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │    SPIRE Agent      │
                    │     DaemonSet       │
                    │                     │
                    │    Workload API     │
                    └──────────┬──────────┘
                               │
                         Unix Domain Socket
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
                 ▼                           ▼
        ┌─────────────────┐        ┌─────────────────┐
        │ identity-client │        │ identity-server │
        │                 │        │                 │
        │ ServiceAccount  │        │ ServiceAccount  │
        │       │         │        │       │         │
        │       ▼         │        │       ▼         │
        │    SVID         │        │    SVID         │
        └────────┬────────┘        └────────┬────────┘
                 │                          │
                 │       mTLS               │
                 └─────────────────────────►│
                                            │
                                    Identity verification
```

---

# 4. Hubungan dengan Phase 13

Phase 13 sudah membangun:

```
ServiceAccount
      │
      ├── RBAC
      │
      └── IRSA
```

dan:

```
NetworkPolicy
```

Phase 15 **tidak menggantikan** semua itu.

Justru kita mendapatkan layered identity:

```
                    WORKLOAD
                       │
             ┌─────────┴─────────┐
             │                   │
             ▼                   ▼
      Kubernetes Identity    SPIFFE Identity
             │                   │
       ServiceAccount         SVID
             │                   │
       RBAC / IRSA              │
                                 ▼
                              mTLS
                                 │
                                 ▼
                         Service Authorization
```

Contoh:

```
NetworkPolicy:
frontend → payment = ALLOW
```

belum berarti:

```
frontend → payment = automatically trusted
```

Payment service masih dapat memeriksa:

```
SPIFFE ID
```

sehingga security menjadi dua layer:

```
Network authorization
        +
Workload identity authorization
```

---

# 5. Hubungan dengan Phase 14

Phase 14 memberikan:

```
Kyverno
Gatekeeper
Falco
```

Maka lifecycle workload sekarang:

```
                    Pod Manifest
                         │
                         ▼
                 Kyverno / Gatekeeper
                         │
                   ALLOW / DENY
                         │
                         ▼
                    Kubernetes
                         │
                         ▼
                  ServiceAccount
                         │
                         ▼
                  SPIRE Attestation
                         │
                         ▼
                    SPIFFE ID
                         │
                         ▼
                      SVID
                         │
                         ▼
                       mTLS
                         │
                         ▼
                 Service-to-Service
                         │
                         ▼
                       Falco
                  Runtime Detection
```

Dengan demikian Phase 15 berada **di antara admission security dan application communication security**.

---

# 6. Trust Model

Trust model yang dibangun:

```
                    example.org
                        │
                  Trust Domain
                        │
             ┌──────────┴──────────┐
             │                     │
             ▼                     ▼
         client SVID           server SVID
             │                     │
             │                     │
             └─────────┬───────────┘
                       │
                      mTLS
                       │
                       ▼
                Mutual Identity
                Verification
```

Client memverifikasi:

```
"Apakah server benar-benar identity
 yang saya percaya?"
```

Server memverifikasi:

```
"Apakah client benar-benar workload
 yang saya izinkan?"
```

Jadi bukan sekadar:

```
CA trusted = connection trusted
```

tetapi:

```
Certificate trusted
        +
SPIFFE identity correct
        +
Authorization policy allows identity
        =
Communication allowed
```

---

# 7. Validation Matrix Phase 15

Berikut matrix berdasarkan hasil aktual yang lu kasih.

> **Catatan klasifikasi:** beberapa item yang lu label sebagai `FLOCi LIMITATION` secara teknis lebih tepat disebut **testing/implementation limitation**, bukan keterbatasan fundamental Floci. Contohnya `openssl` tidak tersedia di image workload bukan berarti Floci tidak mendukung mTLS. Untuk dokumentasi project, gue tetap mempertahankan kategori **FLOCi LIMITATION** sesuai format project lu, tetapi alasan teknisnya gue pertegas.

|No|Security Capability|Status Aktual|Alasan & Analisis Teknis|
|---|---|---|---|
|1|SPIRE Server|**PASS**|StatefulSet SPIRE Server berhasil di-deploy dan berjalan normal di namespace `spire`.|
|2|SPIRE Agent|**PASS**|DaemonSet SPIRE Agent berhasil berjalan pada node dan terhubung ke SPIRE Server.|
|3|Trust Domain|**PASS**|Trust domain `example.org` berhasil digunakan secara konsisten pada SPIRE environment.|
|4|Node Attestation|**PASS**|SPIRE Agent berhasil membuktikan identity node kepada SPIRE Server sebelum memperoleh trust.|
|5|Workload Attestation|**PASS**|SPIRE berhasil mengenali atribut Kubernetes workload berdasarkan ServiceAccount dan Namespace.|
|6|Workload Registration|**PASS**|Controller Manager berhasil memetakan workload Kubernetes ke SPIFFE identity berdasarkan registration/selector.|
|7|SPIFFE ID Issuance|**PASS**|SVID berhasil diterbitkan menggunakan mapping Namespace dan ServiceAccount.|
|8|Workload API|**PASS**|Workload dapat mengakses SPIFFE Workload API melalui Unix Domain Socket yang di-mount ke Pod.|
|9|X.509-SVID|**PASS**|Workload berhasil memperoleh credential berbasis X.509 dari SPIRE.|
|10|SPIFFE URI SAN|**PASS**|SPIFFE ID berhasil tertanam sebagai URI SAN pada X.509-SVID, misalnya `spiffe://example.org/...`.|
|11|Trust Bundle|**PASS**|Root CA/trust bundle tersedia sehingga SVID dapat diverifikasi terhadap trust domain.|
|12|Identity Uniqueness|**PASS**|`identity-client` dan `identity-server` memperoleh SPIFFE identity berbeda sesuai ServiceAccount masing-masing.|
|13|Unauthorized Identity Rejection|**PASS**|Workload yang tidak memenuhi registration/selector tidak dapat memperoleh SVID secara arbitrary.|
|14|SVID Rotation|**PASS**|Mekanisme short-lived credential renewal berjalan otomatis tanpa pengelolaan certificate secara manual.|
|15|mTLS|**FLOCi LIMITATION**|mTLS handshake tidak dapat divalidasi langsung melalui terminal karena workload testing image tidak menyediakan utility seperti `openssl`. Ini membatasi observability pengujian, bukan membuktikan bahwa SPIRE mTLS tidak didukung.|
|16|Correct Identity → Allowed|**FLOCi LIMITATION**|Identity issuance berhasil, tetapi authorization komunikasi menggunakan SVID belum dapat dibuktikan secara interaktif karena tidak tersedia application/client testing utility yang sesuai.|
|17|Wrong Identity → Denied|**FLOCi LIMITATION**|Negative mTLS identity test belum dapat dilakukan secara langsung karena workload tidak menyediakan utility untuk membangun dan memanipulasi koneksi TLS/SVID.|
|18|NetworkPolicy + SPIFFE|**PASS**|NetworkPolicy Kubernetes berhasil diterapkan dan tetap dapat digunakan bersama SPIFFE workload identity.|
|19|Network Allowed + Identity Denied|**FLOCi LIMITATION**|Skenario defense-in-depth ini membutuhkan aplikasi mTLS yang benar-benar memeriksa SPIFFE identity; belum dapat dibuktikan end-to-end pada environment pengujian.|
|20|Network Denied + Identity Valid|**PASS**|NetworkPolicy terbukti dapat memblokir koneksi walaupun workload memiliki identity yang valid. Ini membuktikan network authorization tetap menjadi independent security layer.|
|21|Agent Restart Recovery|**PASS**|SPIRE Agent berhasil restart dan kembali membangun koneksi dengan SPIRE Server.|
|22|Server Restart Recovery|**PASS**|SPIRE Server berhasil restart dan state dapat dipulihkan sehingga infrastructure identity kembali berfungsi.|
|23|Registration Persistence|**PASS**|Registration entry tetap tersedia setelah siklus restart infrastructure.|
|24|JWT-SVID|**FLOCi LIMITATION**|Implementasi Phase 15 berfokus pada X.509-SVID dan JWT-SVID tidak diaktifkan/divalidasi dalam workload test.|
|25|JWT Audience Enforcement|**FLOCi LIMITATION**|Karena JWT-SVID tidak diuji, validasi `aud` terhadap service tujuan juga belum dapat dilakukan.|
|26|Final E2E mTLS|**FLOCi LIMITATION**|Seluruh chain identity → SVID → trust validation → mTLS → authorization belum dapat divalidasi secara end-to-end karena tidak tersedia testing client/application yang sesuai di dalam workload.|

---

# 8. Interpretasi hasil Phase 15

Secara keseluruhan, hasilnya menunjukkan bahwa **identity infrastructure-nya berhasil**.

Bagian yang sudah benar-benar terbukti:

```
SPIRE Server
      ↓
SPIRE Agent
      ↓
Node Attestation
      ↓
Workload Attestation
      ↓
Workload Registration
      ↓
SPIFFE ID
      ↓
X.509-SVID
      ↓
URI SAN
      ↓
Trust Bundle
      ↓
SVID Rotation
```

Jadi bagian:

> **"Apakah Kubernetes workload bisa mendapatkan cryptographically verifiable identity dari SPIRE?"**

sudah **PASS**.

Yang belum terbukti adalah layer berikutnya:

```
SVID
 │
 ▼
mTLS
 │
 ▼
Identity Authorization
 │
 ▼
Service-to-Service Access
```

Karena testing environment belum memiliki application/client yang bisa melakukan mTLS secara langsung.

Ini penting dibedakan.

Kita **tidak boleh menyimpulkan**:

```
mTLS = gagal
```

karena yang sebenarnya terjadi:

```
SPIFFE/SPIRE identity issuance = terbukti

mTLS capability = belum dapat dibuktikan
                  dengan testing workload yang tersedia
```

---

# 9. Posisi Phase 15 terhadap keseluruhan project

Setelah Phase 15, security architecture project sudah menjadi:

```
                         ENTERPRISE APPLICATION
                                  │
                                  ▼
                        ┌──────────────────┐
                        │ Kubernetes / EKS │
                        └────────┬─────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
        PHASE 13             PHASE 14          PHASE 15
        Identity             Prevention         Zero Trust
              │              & Detection             │
              │                  │                   │
        ┌─────┴─────┐      ┌─────┴─────┐      ┌─────┴─────┐
        │           │      │           │      │           │
      RBAC        IRSA   Kyverno   Gatekeeper SPIFFE    SPIRE
        │           │      │           │        │           │
        │           │      └─────┬─────┘        │           │
        │           │            │              SVID        │
        │           │          Falco             │           │
        │           │            │              mTLS         │
        └───────────┴────────────┴──────────────┴───────────┘
                              │
                              ▼
                       DEFENSE IN DEPTH
```

Dan NetworkPolicy dari Phase 13 tetap berada sebagai network enforcement layer:

```
                 Request
                    │
                    ▼
             NetworkPolicy
                    │
             Network reachable?
                /        \
              NO          YES
              │            │
            DENY           ▼
                       SPIFFE/mTLS
                           │
                    Identity valid?
                       /       \
                     NO         YES
                     │           │
                   DENY         ALLOW
```

Ini yang membuat Phase 15 relevan dengan konsep **Zero Trust**: network reachability dan workload identity diperlakukan sebagai kontrol yang berbeda, bukan satu kontrol yang sama.

---

# 10. Gap yang tersisa setelah Phase 15

Hasil aktual juga menunjukkan satu gap yang jelas:

```
             IDENTITY PLANE
                  │
             ┌────▼────┐
             │ SPIRE   │
             │  PASS   │
             └────┬────┘
                  │
                SVID
                  │
                  ▼
          ┌───────────────┐
          │ mTLS test     │
          │               │
          │ NOT PROVEN    │
          └───────┬───────┘
                  │
                  ▼
       Service Authorization
                  │
                  ▼
        End-to-End Zero Trust
```

Jadi Phase 15 **sudah berhasil membangun workload identity plane**, tetapi belum membuktikan seluruh application communication plane.

Itu justru menjadi konteks yang bagus untuk phase berikutnya: identity yang sudah diterbitkan SPIRE dapat digunakan sebagai input untuk **authorization/service-to-service policy**, bukan sekadar certificate issuance.

Dengan demikian hasil Phase 15 tidak overclaim: **SPIFFE/SPIRE infrastructure dan workload identity sudah PASS, sedangkan end-to-end mTLS authorization tetap terdokumentasi sebagai limitation dari environment/testing stack.**

---

# Step 247. Prepare Phase 15 Workspace & Baseline

## Tujuan

Membuat workspace khusus Phase 15 dan memastikan state dari Phase 13–14 masih sehat.

Ini penting karena Phase 15 harus berdiri di atas:

- EKS/k3s Floci
    
- ServiceAccount
    
- namespace
    
- NetworkPolicy
    
- admission security
    
- Falco
    

yang sudah dibuat sebelumnya.

### 247.1 Buat workspace

```bash
mkdir ~/phase15
cd ~/phase15
```

### Parameter

- `mkdir` → membuat directory.
    
- `~/phase15` → directory Phase 15 di home user.
    
- `cd` → masuk ke directory tersebut.
    

### 247.2 Set environment

```bash
export ENDPOINT=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export CLUSTER_NAME=enterprise-eks
export SPIFFE_TRUST_DOMAIN=example.org
```

Parameter:

- `ENDPOINT` → Floci API endpoint.
    
- `AWS_DEFAULT_REGION` → region AWS simulation.
    
- `CLUSTER_NAME` → nama EKS cluster yang digunakan project.
    
- `SPIFFE_TRUST_DOMAIN` → root namespace cryptographic identity kita.
    

Trust domain `example.org` tidak harus benar-benar menjadi DNS domain. SPIFFE menjelaskan bahwa trust domain merupakan root trust untuk identity dan tidak harus memiliki DNS infrastructure. ([Spiffe](https://spiffe.io/docs/latest/deploying/configuring/?utm_source=chatgpt.com "Configuring SPIRE | SPIFFE"))

### 247.3 Validasi

```bash
echo "$ENDPOINT"
echo "$AWS_DEFAULT_REGION"
echo "$CLUSTER_NAME"
echo "$SPIFFE_TRUST_DOMAIN"

kubectl cluster-info
kubectl get nodes -o wide
```

Expected:

```text
http://localhost:4566
us-east-1
enterprise-eks
example.org
```

dan Kubernetes API/node berada dalam kondisi `Ready`.

### Validation

|Test|Expected|
|---|---|
|Floci endpoint variable|PASS|
|Region variable|PASS|
|Cluster variable|PASS|
|Kubernetes API|PASS|
|Node Ready|PASS|

---

# Step 248. Baseline Existing Security Controls

## Tujuan

Memastikan Phase 15 tidak merusak security control Phase 13–14.

### 248.1 Check namespace

```bash
kubectl get namespace
```

### 248.2 Check ServiceAccount

```bash
kubectl get serviceaccount -A
```

### 248.3 Check NetworkPolicy

```bash
kubectl get networkpolicy -A
```

### 248.4 Check Kyverno

```bash
kubectl get pods -A | grep -i kyverno
```

### 248.5 Check Gatekeeper

```bash
kubectl get pods -A | grep -i gatekeeper
```

### 248.6 Check Falco

```bash
kubectl get pods -A | grep -i falco
```

### Validasi

Yang kita cari bukan nama pod tertentu, tetapi bukti bahwa security stack Phase 13–14 masih tersedia.

```text
ServiceAccount      → harus tersedia
NetworkPolicy       → harus tersedia
Kyverno             → harus healthy jika masih dipertahankan
Gatekeeper          → sesuai hasil Phase 14
Falco               → harus healthy
```

Kalau salah satu komponen Phase 14 memang sudah dihapus sebagai bagian cleanup, **jangan dianggap failure**.

---

# Step 249. Check Kubernetes Version & SPIRE Compatibility

## Tujuan

Memastikan versi Kubernetes yang digunakan Floci cocok untuk eksperimen SPIRE.

SPIFFE quickstart Kubernetes saat ini mendokumentasikan pengujian pada Kubernetes 1.29–1.34. Cluster Floci kita harus kita cek aktualnya sebelum instalasi. ([Spiffe](https://spiffe.io/docs/latest/try/getting-started-k8s/?utm_source=chatgpt.com "Quickstart for Kubernetes | SPIFFE"))

### 249.1 Kubernetes version

```bash
kubectl version
```

### 249.2 Node information

```bash
kubectl get nodes -o wide
```

### 249.3 Detailed node runtime

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" | kernel="}{.status.nodeInfo.kernelVersion}{" | runtime="}{.status.nodeInfo.containerRuntimeVersion}{"\n"}{end}'
```

### Penjelasan parameter

`jsonpath` mengambil field tertentu dari object Kubernetes sehingga kita tidak perlu membaca seluruh `kubectl describe node`.

Yang kita ambil:

```text
.metadata.name
.status.nodeInfo.kernelVersion
.status.nodeInfo.containerRuntimeVersion
```

### Validasi

Expected:

```text
node-name | kernel=... | runtime=...
```

Kalau versi Kubernetes berada di luar range quickstart, **belum otomatis failure**; kita lanjutkan karena compatibility harus dibuktikan secara runtime.

---

# Step 250. Create SPIRE Namespace

## Tujuan

Memisahkan control-plane identity infrastructure dari application namespace.

Architecture:

```text
spire
├── SPIRE Server
├── SPIRE Agent
└── SPIRE Controller Manager
```

Sedangkan workload tetap:

```text
enterprise-app
enterprise-security
phase15-identity
```

### 250.1 Create namespace

```bash
kubectl create namespace spire --dry-run=client -o yaml | kubectl apply -f -
```

Parameter:

- `create namespace spire` → membuat namespace.
    
- `--dry-run=client` → tidak langsung membuat object.
    
- `-o yaml` → menghasilkan manifest.
    
- `kubectl apply -f -` → menerapkan YAML dari stdin.
    

### Validasi

```bash
kubectl get namespace spire
```

Expected:

```text
spire    Active
```

---

# Step 251. Install SPIRE CRDs

## Tujuan

SPIRE versi modern menyediakan Helm chart hardened untuk deployment Kubernetes. Official SPIFFE documentation menyediakan quick installation melalui chart `spire-crds` dan `spire`. ([Spiffe](https://spiffe.io/docs/latest/spire-helm-charts-hardened-about/installation/?utm_source=chatgpt.com "Installation | SPIFFE"))

Kita install CRDs terlebih dahulu supaya Kubernetes memahami object SPIRE.

### 251.1 Add repository

```bash
helm repo add spiffe-hardened https://spiffe.github.io/helm-charts-hardened/
helm repo update
```

### Parameter

- `helm repo add` → mendaftarkan Helm repository.
    
- `spiffe-hardened` → nama lokal repository.
    
- URL → official SPIFFE hardened Helm repository.
    
- `helm repo update` → mengambil index chart terbaru.
    

### 251.2 Install CRDs

```bash
helm upgrade --install \
  --create-namespace \
  -n spire \
  spire-crds \
  spire-crds \
  --repo https://spiffe.github.io/helm-charts-hardened/
```

Parameter penting:

- `upgrade --install` → install kalau belum ada, upgrade kalau sudah ada.
    
- `--create-namespace` → membuat namespace jika belum ada.
    
- `-n spire` → target namespace.
    
- `spire-crds` pertama → Helm release name.
    
- `spire-crds` kedua → chart name.
    
- `--repo` → source chart.
    

### Validasi

```bash
kubectl get crd | grep -Ei 'spire|spiffe'
```

Expected ada CRD yang berkaitan dengan:

```text
clusterspiffeids
clusterstaticentries
...
```

Nama CRD aktual mengikuti chart version yang terpasang.

---

# Step 252. Install SPIRE Stack

## Tujuan

Deploy komponen utama:

```text
SPIRE Server
SPIRE Agent
Controller Manager
```

SPIRE Server menjadi identity authority, sementara Agent berjalan pada node dan menyediakan Workload API kepada workload. ([Spiffe](https://spiffe.io/docs/latest/spire-about/spire-concepts/?utm_source=chatgpt.com "SPIRE Concepts | SPIFFE"))

### 252.1 Install

```bash
helm upgrade --install -n spire spire spire \
  --repo https://spiffe.github.io/helm-charts-hardened/ \
  --set global.spiffeCSI.enabled=false \
  --set spiffe-csi-driver.enabled=false \
  --set spiffe-oidc-discovery-provider.spiffeCSI.enabled=false
```
### 252.2 Ubah sumber volume CSI menjadi HostPath socket lokal secara langsung
```bash
# (Catatan: Perintah ini langsung mengganti volume index pertama yang tadinya CSI menjadi HostPath socket agent yang valid).
kubectl patch deployment spire-spiffe-oidc-discovery-provider -n spire --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/volumes/0", "value": {"name": "spiffe-workload-api", "hostPath": {"path": "/run/spire/agent-sockets", "type": "DirectoryOrCreate"}}}
]'
```
### Validasi

```bash
kubectl get pods -n spire -o wide
```

Expected architecture:

```text
spire-server-xxxxx       Running
spire-agent-xxxxx        Running
```
Agent seharusnya berbentuk DaemonSet sehingga jumlah Agent mengikuti jumlah node. Ini memang model deployment SPIRE di Kubernetes. ([Spiffe](https://spiffe.io/docs/latest/try/getting-started-k8s/?utm_source=chatgpt.com "Quickstart for Kubernetes | SPIFFE"))

---

# Step 253. Inspect SPIRE Components

## Tujuan

Jangan langsung menganggap Helm `deployed` berarti SPIRE benar-benar berfungsi.

Kita cek:

```text
Deployment
StatefulSet
DaemonSet
Service
CRD
```

### 253.1 Workloads

```bash
kubectl get deployment,statefulset,daemonset -n spire
```

### 253.2 Services

```bash
kubectl get svc -n spire
```

### 253.3 Controller

```bash
kubectl get pods -n spire -o wide
```

### 253.4 Logs Server

```bash
kubectl logs -n spire statefulset/spire-server --tail=100
```

### 253.5 Logs Agent

```bash
kubectl logs -n spire daemonset/spire-agent --tail=100
```

### Validasi

Cari indikasi:

```text
server started
agent started
attestation successful
workload API available
```

Error seperti:

```text
connection refused
failed node attestation
unable to connect to server
```

harus kita investigasi sebelum lanjut.

---

# Step 254. Verify SPIRE Trust Domain

## Tujuan

Memastikan seluruh SPIRE components menggunakan trust domain:

```text
example.org
```

### 254.1 Inspect configuration

```bash
kubectl get configmap -n spire -o yaml | grep -n -i "trust_domain"
```
### Validasi

Expected:

```text
43:          "trust_domain": "example.org"
268:          "trust_domain": "example.org"
308:          "trust_domain": "example.org"
```

atau konfigurasi equivalente dari Helm chart.

SPIRE Server dan Agent harus menggunakan trust domain yang sama. ([Spiffe](https://spiffe.io/docs/latest/deploying/configuring/?utm_source=chatgpt.com "Configuring SPIRE | SPIFFE"))

---

# Step 255. Verify Node Attestation

## Tujuan

Ini adalah salah satu bagian terpenting.

Sebelum SPIRE dapat percaya kepada workload, SPIRE perlu percaya kepada **Agent yang menjalankan workload tersebut**.

Modelnya:

```text
Kubernetes Node
      │
      ▼
SPIRE Agent
      │
      │ node attestation
      ▼
SPIRE Server
      │
      └── trusted agent
```

SPIRE menggunakan node attestation dan workload attestation sebagai bagian dari identity issuance. ([Spiffe](https://spiffe.io/docs/latest/spire-about/spire-concepts/?utm_source=chatgpt.com "SPIRE Concepts | SPIFFE"))

### 255.1 Inspect Agent

```bash
kubectl get pods -n spire -o wide | grep spire-agent
```
### 255.2 Inspect logs

```bash
kubectl logs -n spire daemonset/spire-agent --tail=200
```

### Validasi

Yang ingin kita buktikan:

```text
Agent successfully connects to Server
Agent is attested
Agent receives identity
```
---

# Step 256. Inspect SPIRE Registration Model

## Tujuan

Memahami mapping:

```text
Kubernetes attributes
        │
        ▼
SPIRE selectors
        │
        ▼
SPIFFE ID
```

SPIRE registration entry terdiri dari:

- SPIFFE ID
    
- selectors
    
- parent ID
    

dan selector digunakan untuk menentukan workload mana yang berhak memperoleh identity tersebut. ([Spiffe](https://spiffe.io/docs/latest/deploying/registering/?utm_source=chatgpt.com "Registering workloads | SPIFFE"))

### 256.1 List entries

```bash
kubectl exec -n spire statefulset/spire-server -- \
  /opt/spire/bin/spire-server entry show
```
### Validasi

Kita ingin melihat registration entries untuk:

```text
SPIRE Agent
default identity
```

atau workload identity yang dihasilkan controller manager.

---

# Step 257. Inspect SPIFFE Identity CRDs

## Tujuan

Chart modern dapat menggunakan SPIRE Controller Manager dan `ClusterSPIFFEID` untuk mengelola identity melalui Kubernetes CRD. Dokumentasi official menjelaskan bahwa chart dapat membuat default ClusterSPIFFEID yang memetakan workload berdasarkan namespace dan ServiceAccount. ([Spiffe](https://spiffe.io/docs/latest/spire-helm-charts-hardened-about/identifiers/?utm_source=chatgpt.com "Identifiers | SPIFFE"))

### 257.1 List CRDs

```bash
kubectl api-resources | grep -Ei 'spiffe|spire'
```

### 257.2 List ClusterSPIFFEID

```bash
kubectl get clusterspiffeid -A
```

### 257.3 Inspect

```bash
kubectl get clusterspiffeid -A -o yaml
```

### Validasi

Kita ingin mendapatkan mapping semacam:

```text
Kubernetes namespace
        +
ServiceAccount
        ↓
SPIFFE ID
```

Contoh default SPIRE:

```bash
# spiffeIDTemplate: spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }} -> outputnya
spiffe://example.org/ns/<namespace>/sa/<serviceaccount>
```

Pattern tersebut memang merupakan default identity pattern pada hardened Helm chart. ([Spiffe](https://spiffe.io/docs/latest/spire-helm-charts-hardened-about/identifiers/?utm_source=chatgpt.com "Identifiers | SPIFFE"))

---

# Step 258. Create Dedicated Identity Namespace

## Tujuan

Kita jangan langsung menguji pada workload production-style Phase 13.

Buat isolation namespace:

```text
phase15-identity
```

### 258.1 Namespace

```bash
kubectl create namespace phase15-identity
```

### 258.2 ServiceAccount

```bash
kubectl create serviceaccount identity-client \
  -n phase15-identity

kubectl create serviceaccount identity-server \
  -n phase15-identity
```

### Validasi

```bash
kubectl get serviceaccount -n phase15-identity
```

Expected:

```text
identity-client
identity-server
```

---

# Step 259. Create Workload Identity Mapping

## Tujuan

Kita ingin identity yang deterministic:

```text
identity-client
    ↓
spiffe://example.org/ns/phase15-identity/sa/identity-client

identity-server
    ↓
spiffe://example.org/ns/phase15-identity/sa/identity-server
```

Ini jauh lebih meaningful daripada sekadar mendapatkan random certificate.

### 259.1 Inspect existing default mapping

```bash
kubectl get clusterspiffeid -A -o yaml
```

Kalau default identity sudah mencakup seluruh namespace, kita dapat memanfaatkannya.

### 259.2 Verify expected identity mapping

```bash
kubectl get clusterspiffeid -A
```

Jika controller menggunakan default:

```text
phase15-identity
+
identity-client
```

harus dapat menghasilkan:

```text
spiffe://domain/ns/phase15-identity/sa/identity-client
```

### Validasi

**PASS** apabila workload identity dapat dipetakan secara deterministic dari:

```text
namespace + ServiceAccount
```

tanpa menyimpan private key/certificate secara manual di Secret.

Ini salah satu tujuan utama SPIFFE Workload API. Workload memperoleh identity saat runtime melalui API tersebut. ([Spiffe](https://spiffe.io/docs/latest/spiffe-specs/spiffe_workload_api/?utm_source=chatgpt.com "SPIFFE Workload API | SPIFFE"))

---

# Step 260. Deploy SPIFFE Workload

## Tujuan

Sekarang kita benar-benar membuat workload yang meminta identity.
### 260.1 Edit ConfigMap & Restart SPIRE Agent DaemonSet
```text
untuk mengubah bagian ->
WorkloadAttestor "k8s" {
        plugin_data {
          skip_kubelet_verification = true
          use_new_container_locator = true
        }
      }

jadi ->
WorkloadAttestor "k8s" {
        plugin_data {
          skip_kubelet_verification = true
          use_new_container_locator = false
        }
      }
```     
```bash
kubectl get configmap spire-agent -n spire -o yaml | \
  sed 's/"use_new_container_locator": true/"use_new_container_locator": false/g' | \
  kubectl apply -f - && \
kubectl rollout restart daemonset/spire-agent -n spire
```
### 260.2 Create workload

```bash
cat > identity-workload.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: identity-client
  namespace: phase15-identity
  labels:
    app: identity-client
spec:
  serviceAccountName: identity-client
  containers:
  - name: app
    image: ghcr.io/spiffe/spire-agent:1.15.3
    command:
    - /opt/spire/bin/spire-agent
    - api
    - watch
    - -socketPath
    - /run/spire/sockets/spire-agent.sock
    volumeMounts:
    - name: spiffe-workload-api
      mountPath: /run/spire/sockets
      readOnly: true
  volumes:
  - name: spiffe-workload-api
    hostPath:
      path: /run/spire/agent-sockets
      type: Directory
EOF
```

### Penjelasan

```text
serviceAccountName: identity-client
```

adalah bagian penting.

SPIRE Kubernetes workload attestation dapat menggunakan Kubernetes workload attributes seperti namespace dan ServiceAccount untuk menentukan identity. ([Spiffe](https://spiffe.io/docs/latest/try/getting-started-k8s/?utm_source=chatgpt.com "Quickstart for Kubernetes | SPIFFE"))

### 260.3 Apply

```bash
kubectl apply -f identity-workload.yaml
```

### 260.4 Validate

```bash
kubectl get pod identity-client \
  -n phase15-identity \
  -o wide
```

Expected:

```text
Running
```

---

# Step 261. Verify Workload API Exposure

## Tujuan

Sekarang kita membuktikan bahwa workload tidak perlu memiliki:

```text
/private-key.pem
/certificate.pem
```

yang kita generate sendiri.

Workload mendapatkan identity dari SPIRE Agent melalui **Workload API**.

SPIFFE Workload API biasanya exposed melalui local Unix Domain Socket dan menyediakan X.509-SVID serta trust bundle. ([Spiffe](https://spiffe.io/docs/latest/spiffe-specs/spiffe/?utm_source=chatgpt.com "Secure Production Identity Framework for Everyone | SPIFFE"))

### 261.1 Inspect pod mounts

```bash
kubectl get pod identity-client \
  -n phase15-identity \
  -o yaml
```

Cari:

```text
spire
workload-api
socket
csi
```

### 261.2 Inspect volume mounts

```bash
kubectl get pod identity-client \
  -n phase15-identity \
  -o jsonpath='{.spec.containers[*].volumeMounts}'
```

### Validasi

Expected:

```text
SPIFFE Workload API / SVID access mechanism
```

---

# Step 262. Obtain X.509-SVID

## Tujuan

Ini milestone utama Phase 15.

Kita ingin membuktikan:

```text
Workload
   │
   │ Workload API
   ▼
SPIRE Agent
   │
   ▼
X.509-SVID
```

SVID harus mengandung SPIFFE ID sebagai URI SAN. X.509-SVID specification mensyaratkan SPIFFE ID berada pada URI SAN. ([Spiffe](https://spiffe.io/docs/latest/spiffe-specs/x509-svid/?utm_source=chatgpt.com "X509-SVID | SPIFFE"))

### Validasi certificate

Jika SVID tersedia sebagai file:

```bash
kubectl exec -it identity-client -n phase15-identity -c app -- \
  /opt/spire/bin/spire-agent api fetch -socketPath /run/spire/sockets/spire-agent.sock
```

Kemudian cari:

```text
X509v3 Subject Alternative Name
    URI:spiffe://example.org/...
```

Jika image workload tidak memiliki `openssl`, gunakan container debugging/helper yang sesuai dengan mekanisme SVID yang terpasang.

### Expected

Harus ada:

```text
URI:spiffe://example.org/ns/phase15-identity/sa/identity-client
```

### Validation

```text
Certificate exists
        +
Valid X.509 certificate
        +
SPIFFE URI SAN
        +
Correct trust domain
```

= **PASS**

---

# Step 263. Verify Trust Bundle

## Tujuan

SVID saja belum cukup.

Workload juga perlu tahu:

> "CA mana yang saya percaya?"

SPIFFE Workload API memberikan trust bundle bersama X.509-SVID. ([Spiffe](https://spiffe.io/docs/latest/spiffe-specs/spiffe_workload_api/?utm_source=chatgpt.com "SPIFFE Workload API | SPIFFE"))

Architecture:

```text
identity-client
      │
      ├── X.509-SVID
      │
      └── Trust Bundle
               │
               ▼
        verify identity-server
```

### Validasi

```bash
kubectl exec -n spire spire-server-0 -- /opt/spire/bin/spire-server bundle show > ca.pem && \
openssl x509 -in ca.pem -text -noout
```

Expected:

```text
CA certificate
valid trust root
```

---

# Step 264. Verify Identity of Second Workload

## Tujuan

Membuktikan bahwa identity **unik per workload**, bukan satu certificate dipakai semua workload.

Deploy:

```text
identity-server
```

dengan ServiceAccount berbeda.

```bash
cat <<'EOF' > identity-server.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: identity-server
  namespace: phase15-identity
---
apiVersion: v1
kind: Pod
metadata:
  name: identity-server
  namespace: phase15-identity
  labels:
    app: identity-server
spec:
  serviceAccountName: identity-server
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: spire-workload-api
      mountPath: /run/spire/sockets
      readOnly: true
  volumes:
  - name: spire-workload-api
    hostPath:
      path: /run/spire/sockets
      type: Directory
EOF

kubectl apply -f identity-server.yaml
```

### Validasi

```bash
kubectl get pods -n phase15-identity -o wide
```

### Kemudian verify SVID server.
```bash
kubectl logs -n spire daemonset/spire-agent --tail=50 | grep identity-server
```

Expected identity berbeda:

```text
client:
spiffe://example.org/ns/phase15-identity/sa/identity-client

server:
spiffe://example.org/ns/phase15-identity/sa/identity-server
```

---

# Step 265. Test Identity Isolation

## Tujuan

Sekarang kita test negative case.

Kita ingin membuktikan:

> workload tidak bisa memperoleh arbitrary SPIFFE identity hanya dengan mengubah ServiceAccount/manifest.

### Test

Buat workload baru dengan ServiceAccount berbeda:

```bash
kubectl create serviceaccount unauthorized \
  -n phase15-identity
```

Kemudian deploy workload:

```text
unauthorized
```

dan lihat identity yang diberikan.

### Expected

Workload hanya mendapatkan identity yang sesuai registration/selector.

Tidak boleh bisa meminta:

```text
spiffe://example.org/ns/payment/sa/payment-api
```

secara arbitrary.

### Validation

```text
selector matches registered identity
        ↓
SVID issued

selector does not match
        ↓
identity not issued
```

Ini membuktikan bahwa SPIFFE identity bukan sekadar string yang ditulis pada manifest.

Registration entry memetakan SPIFFE ID dengan selectors yang harus dimiliki workload sebelum identity diberikan. ([Spiffe](https://spiffe.io/docs/latest/deploying/registering/?utm_source=chatgpt.com "Registering workloads | SPIFFE"))

---

# Step 266. Verify X.509-SVID Rotation

## Tujuan

Membuktikan bahwa credential bukan static secret jangka panjang.

SPIRE menyediakan short-lived key/certificate melalui Workload API dan workload dapat menerima identity yang diperbarui secara otomatis. ([Spiffe](https://spiffe.io/docs/latest/deploying/svids/?utm_source=chatgpt.com "Working with SVIDs | SPIFFE"))

### 266.1 Record certificate metadata

```bash
kubectl exec -it identity-client -n phase15-identity -c app -- \
  /opt/spire/bin/spire-agent api fetch -socketPath /run/spire/sockets/spire-agent.sock | grep -E "SPIFFE ID|Valid"
```

Catat:

```text
serial
notBefore
notAfter
```

### 266.2 Tunggu sampai rotation event

Setelah interval renewal yang digunakan deployment:

```bash
kubectl exec -it identity-client -n phase15-identity -c app -- \
  /opt/spire/bin/spire-agent api fetch -socketPath /run/spire/sockets/spire-agent.sock | grep -E "SPIFFE ID|Valid"
```

### Expected

Certificate serial/validity berubah tanpa:

```text
manual secret replacement
manual private key deployment
pod redeployment
```

### Validation

Kalau rotation berjalan:

**PASS**

---

# Step 267. Combine SPIFFE Identity + NetworkPolicy

## Tujuan

Sekarang kita masukkan security control Phase 13.

Target:

```text
NetworkPolicy
        +
mTLS
        +
SPIFFE identity
```

---

## 267.1 Allow client → server

```bash
cat <<'EOF' > mtls-networkpolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-mtls-client
  namespace: phase15-identity
spec:
  podSelector:
    matchLabels:
      app: identity-server
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: identity-client
    ports:
    - protocol: TCP
      port: 8443
EOF

kubectl apply -f mtls-networkpolicy.yaml
```

### Validation

```bash
kubectl get networkpolicy \
  -n phase15-identity
```

---
# Step 268. Final Phase 15 Validation

Jalankan:

```bash
# Periksa status pod di semua namespace terkait
kubectl get pods -n spire
kubectl get pods -n phase15-identity -o wide

# Verifikasi aturan CRD dan NetworkPolicy aktif
kubectl get clusterspiffeid -A
kubectl get networkpolicy -n phase15-identity

# Periksa entri registrasi di SPIRE Server
kubectl exec -n spire spire-server-0 -- \
  /opt/spire/bin/spire-server entry show

# Pantau log sirkulasi SVID dari identity-client via API watch
kubectl logs -n phase15-identity identity-client --tail=50
```


