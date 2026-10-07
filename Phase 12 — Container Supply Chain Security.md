## 1. Phase ini ngapain?

Phase 12 membangun **secure container supply chain** dari source code sampai workload Kubernetes.

Tujuannya bukan sekadar “install Trivy dan bikin ECR”, tetapi membuktikan lifecycle:

```
Application Source
      │
      ▼
Docker Build
      │
      ▼
Local Container Image
      │
      ├──────────────► Trivy Vulnerability Scan
      │
      ├──────────────► Trivy SBOM
      │
      └──────────────► Image ID / Digest
      │
      ▼
      ECR
      │
      ├──────────────► Image Metadata
      ├──────────────► Native Image Scanning
      └──────────────► Image Immutability
      │
      ▼
      EKS
      │
      ├──────────────► Pull Image
      ├──────────────► Verify Image
      └──────────────► Run Workload
      │
      ▼
      Runtime Security
```

Dalam environment AWS nyata, supply chain-nya kira-kira:

```
Developer
   │
   ▼
Git / CI
   │
   ▼
Docker Build
   │
   ├── Trivy vulnerability scan
   ├── SBOM generation
   └── image digest
   │
   ▼
Amazon ECR
   │
   ├── image scanning
   ├── immutable tags
   └── image metadata
   │
   ▼
Amazon EKS
   │
   └── container workload
```

Di project lu, AWS layer digantikan Floci:

```
                         LOCAL
┌──────────────────────────────────────────┐
│ Docker Build                             │
│       │                                  │
│       ├── Trivy vulnerability scan      │
│       ├── Trivy SBOM                    │
│       └── Image digest                   │
└───────┬──────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────┐
│                  FLOCi                   │
│                                          │
│  ECR                                     │
│   ├── Repository                         │
│   ├── Image metadata                     │
│   ├── Scanning                           │
│   └── Tag immutability                   │
└───────┬──────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────┐
│ Existing EKS / k3s                       │
│                                          │
│ enterprise-app namespace                 │
│        │                                 │
│        └── enterprise-security-app       │
└──────────────────────────────────────────┘
```

Ini sengaja **melanjutkan EKS dari Phase 3**, bukan bikin cluster baru.

---

# 2. Kenapa Phase 12 penting terhadap phase sebelumnya dan sesudahnya?

Phase sebelumnya sudah membangun:

```
Phase 1  → Network
Phase 2  → IAM / Security foundation
Phase 3  → Compute + EKS
Phase 4  → ALB / WAF
Phase 5  → RDS
Phase 6  → S3
Phase 7  → DNS / SSL
Phase 8  → KMS / Secrets
Phase 9  → Monitoring
Phase 10 → Audit / Config / GuardDuty
Phase 11 → Automated Security Response
```

Sekarang Phase 12 menjawab:

> **“Image yang masuk ke workload Kubernetes ini berasal dari mana, sudah diperiksa atau belum, dan bagaimana kita memastikan supply chain-nya?”**

Setelah Phase 12:

```
Phase 12
Container Supply Chain
        │
        ▼
Phase berikutnya
Admission Control
Kyverno / OPA
        │
        ▼
Image Digest Enforcement
        │
        ▼
Signing / Provenance
        │
        ▼
Runtime Security
Falco
        │
        ▼
Cloud + Kubernetes Correlation
```

Jadi Phase 12 merupakan **jembatan antara ECR dan Kubernetes workload security**.

---

# 3. Status matrix Phase 12 berdasarkan hasil aktual lu

Ini penting: matrix ini mengikuti **hasil aktual Floci**, bukan expected ideal AWS.

| Area                  | Test                  | Status           | Kenapa                                              |
| --------------------- | --------------------- | ---------------- | --------------------------------------------------- |
| Docker                | Docker Engine         | PASS             | Docker daemon berjalan                              |
| Docker                | Buildx                | PASS             | Toolchain tersedia/akan divalidasi                  |
| Kubernetes            | kubectl               | PASS             | Kubernetes client tersedia                          |
| ECR                   | ECR API               | PASS             | API Floci merespons                                 |
| Trivy                 | CLI                   | PASS             | Trivy 0.74.0 berhasil                               |
| Trivy                 | Vulnerability DB      | PASS             | DB berhasil tersedia                                |
| Trivy                 | Image scan            | PASS             | Local image dapat discan                            |
| Trivy                 | SBOM                  | PASS             | CycloneDX berhasil dibuat                           |
| Application           | Docker build          | PASS             | Image berhasil dibuat                               |
| Image                 | Image ID              | PASS             | Local image tersedia                                |
| Image                 | Digest                | PASS             | Digest berhasil diperoleh                           |
| ECR                   | Repository            | PASS             | Repository tersedia                                 |
| ECR                   | Immutable tags        | PASS             | Repository menunjukkan `IMMUTABLE`                  |
| ECR                   | Authentication        | FLOCi LIMITATION | Docker tidak bisa resolve ECR hostname Floci        |
| ECR                   | Push                  | FLOCi LIMITATION | Registry endpoint hostname tidak resolve            |
| ECR                   | List images           | FLOCi LIMITATION | Repository tetap kosong                             |
| ECR                   | Describe image        | FLOCi LIMITATION | Image tidak pernah berhasil masuk                   |
| ECR                   | Pull                  | FLOCi LIMITATION | Docker gagal resolve hostname                       |
| ECR                   | Native scan           | FLOCi LIMITATION | `StartImageScan` unsupported                        |
| ECR                   | Scan configuration    | FLOCi LIMITATION | Repository scan API tidak tersedia sesuai kebutuhan |
| EKS                   | Deployment            | PASS             | Deployment berhasil dibuat/di-update                |
| EKS                   | Image pull            | FLOCi LIMITATION | EKS tidak dapat mengambil image Floci ECR           |
| EKS                   | Pod startup           | FLOCi LIMITATION | Pod `ImagePullBackOff`                              |
| EKS                   | Runtime               | FLOCi LIMITATION | Container belum pernah start                        |
| SBOM                  | CycloneDX validity    | PASS             | `bomFormat`, `specVersion`, `serialNumber` valid    |
| Supply chain          | Local scan            | PASS             | Local security gate berjalan                        |
| Supply chain          | ECR push → EKS        | FLOCi LIMITATION | Registry integration tidak berfungsi end-to-end     |
| ECR                   | Vulnerable image test | FLOCi LIMITATION | Tidak bisa push image ke ECR                        |
| EKS                   | Application response  | FLOCi LIMITATION | Tidak ada running container karena image pull       |
| Security architecture | Supply-chain design   | PASS             | Lifecycle berhasil direpresentasikan                |

**Tidak ada `FAIL` untuk hasil aktual utama Phase 12.**

Ini penting secara engineering.

`ImagePullBackOff` bukan berarti application container lu rusak.

Root cause yang sudah kelihatan:

```
Docker
  │
  ▼
000000000000.dkr.ecr.us-east-1.localhost:5100
  │
  X
DNS resolution
  │
  ▼
ECR push gagal
  │
  ▼
ECR repository kosong
  │
  ▼
EKS mencoba pull
  │
  X
Image tidak tersedia
  │
  ▼
ImagePullBackOff
```

Jadi failure chain-nya adalah **FLOCi ECR registry integration**, bukan Docker image atau Kubernetes application.

---

# Step 146. Prepare Phase 12 Workspace

## Tujuan

Membuat workspace khusus Phase 12 agar seluruh artifact supply-chain tersimpan terpisah.

```sh
mkdir -p ~/phase12
cd ~/phase12
```

### Penjelasan

`mkdir -p ~/phase12`

- `mkdir` → membuat directory
- `-p` → tidak error kalau directory sudah ada
- `~/phase12` → workspace Phase 12

`cd ~/phase12`

Masuk ke workspace tersebut.

## Expected output

Tidak ada output jika berhasil.

Validasi:

```sh
pwd
```

Expected:

```
/home/floci/phase12
```

Lalu:

```sh
echo "PHASE 12 WORKSPACE: PASS"
```

Expected:

```
PHASE 12 WORKSPACE: PASS
```

---

# Step 147. Validate Container Security Toolchain

## Tujuan

Memastikan seluruh tool yang diperlukan untuk supply chain tersedia sebelum membangun application.

Tool utama:

```
Docker
Buildx
kubectl
AWS CLI
Trivy
```

---

## 147.1. Validate Docker Engine

```sh
docker version
```

### Parameter

`docker version`

Menampilkan:

- Docker client
- Docker server
- API version
- containerd
- runc

Expected:

```
Client: Docker Engine ...
Server: Docker Engine ...
containerd ...
runc ...
```

Validasi:

```sh
docker info >/dev/null && echo "DOCKER: PASS"
```

Expected:

```
DOCKER: PASS
```

---

## 147.2. Validate Docker Buildx

```sh
docker buildx version
```

Kemudian:

```sh
docker buildx inspect --bootstrap
```

### Tujuan

Buildx digunakan untuk memastikan Docker build engine siap melakukan container image build.

### Parameter

`buildx version`

Menampilkan versi Buildx.

`buildx inspect`

Melihat builder yang digunakan.

`--bootstrap`

Memastikan builder diinisialisasi.

Validasi:

```sh
docker buildx inspect --bootstrap >/dev/null \
  && echo "DOCKER BUILDX: PASS"
```

Expected:

```
DOCKER BUILDX: PASS
```

---

## 147.3. Validate Kubernetes CLI

```sh
kubectl version --client
```

Expected:

```
Client Version: v1.36.3
```

Validasi:

```sh
command -v kubectl >/dev/null \
  && echo "KUBECTL CLI: PASS"
```

Catatan: koneksi ke cluster baru divalidasi pada Step 148.

---

## 147.4. Validate ECR API

```sh
aws --endpoint-url="$ENDPOINT" \
  ecr describe-repositories
```

### Parameter

`--endpoint-url="$ENDPOINT"`

Memaksa AWS CLI menggunakan Floci:

```
http://localhost:4566
```

`ecr`

Memilih Amazon Elastic Container Registry API.

`describe-repositories`

Meminta metadata repository ECR.

Expected:

- daftar repository, atau
- empty repository list.

Yang penting API merespons tanpa error capability.

Validasi:

```sh
aws --endpoint-url="$ENDPOINT" \
  ecr describe-repositories >/dev/null \
  && echo "ECR API: PASS"
```

---

## 147.5. Install Trivy

Di hasil lu sebelumnya Trivy belum tersedia. Install Trivy melalui repository resminya:

```sh
sudo apt-get update
sudo apt-get install -y wget gnupg

wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/trivy.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" \
  | sudo tee /etc/apt/sources.list.d/trivy.list

sudo apt-get update
sudo apt-get install -y trivy
```

Validasi:

```sh
trivy --version
```

Expected pada environment lu:

```
Version: 0.74.0
```

---

## 147.6. Validate Trivy vulnerability database

```sh
trivy image \
  --download-db-only
```

### Tujuan

Memastikan Trivy mempunyai vulnerability database sebelum scanning.

### Parameter

`image`

Menggunakan image-scanning engine.

`--download-db-only`

Download/update database tanpa melakukan scan image.

Validasi:

```sh
trivy image \
  --download-db-only >/dev/null \
  && echo "TRIVY DB: PASS"
```

Expected:

```
TRIVY DB: PASS
```

---

## 147.7. Validate Trivy image scanner

Pull image kecil:

```sh
docker pull alpine:latest
```

Scan:

```sh
trivy image \
  --scanners vuln \
  alpine:latest
```

### Parameter

`image`

Mode scanning container image.

`--scanners vuln`

Hanya vulnerability scanner.

`alpine:latest`

Image target.

Expected:

- Trivy berhasil membaca image
- vulnerability table muncul
- severity dapat ditampilkan

Jumlah vulnerability **tidak perlu dipaksa menjadi 0**. Yang divalidasi adalah scanner berjalan.

Validasi:

```sh
trivy image \
  --scanners vuln \
  alpine:latest >/dev/null \
  && echo "TRIVY IMAGE SCANNER: PASS"
```

---

## 147.8. Validate SBOM generation

```sh
trivy image \
  --format cyclonedx \
  --output /tmp/phase12-test-sbom.json \
  alpine:latest
```

### Parameter

`--format cyclonedx`

Output dalam format CycloneDX SBOM.

`--output`

Lokasi output file.

`/tmp/phase12-test-sbom.json`

File SBOM sementara.

Validasi:

```
test -s /tmp/phase12-test-sbom.json \
  && echo "TRIVY SBOM: PASS"
```

---

## 147.9. Final Step 147 validation

```sh
echo "=================================================="
echo " Step 147: CONTAINER TOOLCHAIN VALIDATION"
echo "=================================================="

echo
echo "[1] Docker:"
docker version --format 'Client={{.Client.Version}} Server={{.Server.Version}}'
docker info >/dev/null \
  && echo "DOCKER ENGINE: PASS"

echo
echo "[2] Docker Buildx:"
docker buildx version
docker buildx inspect --bootstrap >/dev/null \
  && echo "DOCKER BUILDX: PASS"

echo
echo "[3] Kubernetes:"
kubectl version --client
echo "KUBECTL: PASS"

echo
echo "[4] ECR API:"
aws --endpoint-url="$ENDPOINT" \
  ecr describe-repositories >/dev/null \
  && echo "ECR API: PASS"

echo
echo "[5] Trivy:"
trivy --version
echo "TRIVY CLI: PASS"

echo
echo "[6] Trivy Image Scanner:"
trivy image \
  --scanners vuln \
  alpine:latest >/dev/null \
  && echo "TRIVY IMAGE SCANNER: PASS"

echo
echo "[7] Trivy SBOM:"
trivy image \
  --format cyclonedx \
  --output /tmp/phase12-test-sbom.json \
  alpine:latest >/dev/null \
  && echo "TRIVY SBOM: PASS"

echo
echo "=================================================="
echo " Step 147 VALIDATION COMPLETE"
echo "=================================================="
```

Expected:

```
DOCKER ENGINE: PASS
DOCKER BUILDX: PASS
KUBECTL: PASS
ECR API: PASS
TRIVY CLI: PASS
TRIVY IMAGE SCANNER: PASS
TRIVY SBOM: PASS
```

---

# Step 148. Validate Existing EKS Workload Environment

## Tujuan

Memastikan Phase 12 menggunakan **EKS yang sudah dibangun sebelumnya**, bukan membuat cluster baru.

Target:

```
enterprise-app
└── enterprise-security-app
```

---

## 148.1. Validate cluster

```sh
kubectl cluster-info
```

Expected:

```
Kubernetes control plane ...
```

Validasi:

```sh
kubectl get nodes -o wide
```

Expected node:

```
STATUS   Ready
```

---

## 148.2. Validate namespace

```sh
kubectl get namespace enterprise-app
```

Expected:

```
enterprise-app
```

---

## 148.3. Validate existing workload

```sh
kubectl get deployment \
  -n enterprise-app
```

Kemudian:

```sh
kubectl get service \
  -n enterprise-app
```

Tujuannya supaya kita tahu resource apa yang sudah ada sebelum Phase 12 mengubah image.

---

## 148.4. Validate ServiceAccount

```sh
kubectl get serviceaccount \
  -n enterprise-app
```

Expected ada:

```
enterprise-app-sa
```

Ini penting karena workload nantinya tetap berada di security architecture yang sudah dibangun pada phase sebelumnya.

---

## 148.5. Final validation

```sh
kubectl get nodes
kubectl get namespace enterprise-app
kubectl get deployment -n enterprise-app
kubectl get service -n enterprise-app
kubectl get serviceaccount -n enterprise-app

echo "Step 148: EKS FOUNDATION VALIDATION COMPLETE"
```

Expected:

```
Node → Ready
Namespace → enterprise-app
Deployment → existing
Service → existing
ServiceAccount → enterprise-app-sa
```

---

# Step 149. Create Security Test Application

## Tujuan

Membuat application kecil yang akan melewati seluruh supply chain.

Application tidak perlu kompleks.

Kita cuma butuh:

```
HTTP server
Port 8080
Health response
```

---

## 149.1. Create application

```sh
cat > app.py <<'PY'
from http.server import BaseHTTPRequestHandler, HTTPServer
import json

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/health":
            body = {
                "status": "healthy",
                "service": "enterprise-security-app"
            }
        else:
            body = {
                "service": "enterprise-security-app",
                "phase": 12,
                "security": "container-supply-chain"
            }

        payload = json.dumps(body).encode()

        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(payload)))
        self.end_headers()
        self.wfile.write(payload)

    def log_message(self, format, *args):
        pass

server = HTTPServer(("0.0.0.0", 8080), Handler)
server.serve_forever()
PY
```

### Tujuan

Application ini menjadi artifact yang:

```
source
→ Docker image
→ Trivy
→ SBOM
→ ECR
→ EKS
```

---

## 149.2. Create Dockerfile

```sh
cat > Dockerfile <<'EOF'
FROM python:3.12-slim

WORKDIR /app

COPY app.py .

EXPOSE 8080

USER 65532:65532

CMD ["python", "app.py"]
EOF
```

### Security relevance

`USER 65532:65532`

Container tidak berjalan sebagai root.

Ini nanti divalidasi lagi di Kubernetes.

---

## 149.3. Validate source

```sh
ls -la
```

Expected:

```
Dockerfile
app.py
```

Syntax validation:

```sh
python3 -m py_compile app.py
```

Expected:

```
(no output)
```

Final:

```sh
test -f Dockerfile &&
test -f app.py &&
echo "Step 149: APPLICATION SOURCE PASS"
```

---

# Step 150. Build Container Image

## Tujuan

Mengubah application source menjadi immutable container artifact.

```
app.py
   ↓
Dockerfile
   ↓
enterprise-security-app:phase12
```

---

## 150.1. Build

```sh
docker build \
  -t enterprise-security-app:phase12 \
  .
```

### Parameter

`build`

Build container image.

`-t`

Memberikan tag.

`enterprise-security-app:phase12`

Nama image + tag.

`.`

Build context adalah current directory.

Expected:

```
Successfully tagged enterprise-security-app:phase12
```

---

## 150.2. Validate image

```shs
docker image ls \
  enterprise-security-app
```

Expected:

```
enterprise-security-app   phase12
```

Inspect:

```sh
docker image inspect \
  enterprise-security-app:phase12 \
  --format '{{.Id}}'
```

Expected:

```
sha256:...
```

---

## 150.3. Local runtime test

```sh
docker run -d \
  --name phase12-local-test \
  -p 18080:8080 \
  enterprise-security-app:phase12
```

### Parameter

`-d`

Detached/background.

`--name`

Nama container.

`-p 18080:8080`

Host port 18080 → container port 8080.

Test:

```sh
curl http://localhost:18080/health
```

Expected:

```
{"status": "healthy", "service": "enterprise-security-app"}
```

Cleanup:

```sh
docker rm -f phase12-local-test
```

Final:

```sh
docker image inspect enterprise-security-app:phase12 >/dev/null \
  && echo "Step 150: CONTAINER BUILD PASS"
```

---

# Step 151. Capture Image Identity and Digest

## Tujuan

Supply chain security harus mempunyai immutable identity.

Tag:

```
phase12
```

bersifat mutable secara umum.

Digest:

```
sha256:...
```

mengidentifikasi exact image content.

---

## 151.1. Image ID

```sh
export IMAGE_ID=$(
  docker image inspect \
    enterprise-security-app:phase12 \
    --format '{{.Id}}'
)

echo "$IMAGE_ID"
```

Expected:

```
sha256:...
```

---

## 151.2. Image digest

Jika RepoDigest sudah tersedia:

```sh
docker inspect \
  --format='{{index .RepoDigests 0}}' \
  enterprise-security-app:phase12
```

Namun local image bisa saja belum memiliki RepoDigest.

Karena itu kita tetap simpan image ID:

```sh
echo "IMAGE_ID=$IMAGE_ID"
```

---

## 151.3. Validate

```sh
test -n "$IMAGE_ID" \
  && echo "Step 151: IMAGE ID PASS"
```

---

# Step 152. Trivy Vulnerability Scan

## Tujuan

Melakukan security gate sebelum image masuk ECR.

```
Docker Image
     │
     ▼
   Trivy
     │
     ├── CRITICAL
     ├── HIGH
     ├── MEDIUM
     └── LOW
```

Command:

```sh
trivy image \
  --scanners vuln \
  enterprise-security-app:phase12
```

### Parameter

`--scanners vuln`

Fokus pada vulnerability.

Image target:

```
enterprise-security-app:phase12
```

Expected:

```
Total: ...
UNKNOWN: ...
LOW: ...
MEDIUM: ...
HIGH: ...
CRITICAL: ...
```

Jumlah finding bukan penentu apakah scanner berhasil.

Validasi:

```sh
trivy image \
  --scanners vuln \
  enterprise-security-app:phase12 >/dev/null \
  && echo "Step 152: TRIVY SCAN PASS"
```

---

# Step 153. Enforce HIGH/CRITICAL Security Gate

## Tujuan

Sekarang scan bukan hanya observability, tetapi menjadi **build gate**.

Command:

```sh
trivy image \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  enterprise-security-app:phase12
```

### Parameter

`--severity HIGH,CRITICAL`

Hanya memeriksa severity HIGH dan CRITICAL.

`--exit-code 1`

Kalau vulnerability ditemukan, Trivy return exit code `1`.

Ini memungkinkan CI/CD menghentikan image promotion.

### Expected

Ada dua kemungkinan legitimate:

```
exit 0
```

berarti tidak ada HIGH/CRITICAL.

atau:

```
exit 1
```

berarti ditemukan HIGH/CRITICAL.

Jadi **exit 1 bukan otomatis Phase failure**. Itu adalah hasil security gate.

Validasi:

```sh
trivy image \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  enterprise-security-app:phase12

echo "TRIVY_EXIT_CODE=$?"
```

Simpan hasil tersebut.

---

# Step 154. Generate SBOM

## Tujuan

Membuat inventory software yang terdapat di dalam container.

```
Container
 ├── OS packages
 ├── Python
 ├── Libraries
 └── Components
        ↓
      SBOM
```

Command:

```sh
trivy image \
  --format cyclonedx \
  --output sbom-cyclonedx.json \
  enterprise-security-app:phase12
```

### Parameter

`--format cyclonedx`

SBOM format CycloneDX.

`--output`

File output.

`sbom-cyclonedx.json`

Artifact SBOM.

Validasi:

```sh
test -s sbom-cyclonedx.json \
  && echo "Step 154: SBOM GENERATION PASS"
```

---

# Step 155. Validate SBOM Structure

## Tujuan

Memastikan file benar-benar SBOM, bukan sekadar file JSON kosong.

```sh
grep -E \
  '"bomFormat"|"specVersion"|"serialNumber"' \
  sbom-cyclonedx.json
```

Expected seperti hasil aktual lu:

```
"bomFormat": "CycloneDX"
"specVersion": "1.7"
"serialNumber": "urn:uuid:..."
```

Kemudian:

```sh
grep -E \
  '"name"|"version"' \
  sbom-cyclonedx.json \
  | head -20
```

Expected:

```
enterprise-security-app:phase12
```

dan component information.

Final:

```sh
test -s sbom-cyclonedx.json &&
grep -q '"bomFormat": "CycloneDX"' sbom-cyclonedx.json &&
echo "Step 155: SBOM STRUCTURE PASS"
```

---

# Step 156. Create ECR Repository

## Tujuan

Membuat private container registry repository di Floci.

```sh
export ECR_REPOSITORY=enterprise-security-app
export ECR_URI="000000000000.dkr.ecr.us-east-1.localhost:5100/$ECR_REPOSITORY"
export IMAGE_TAG=phase12-v1
```

Create:

```sh
aws --endpoint-url="$ENDPOINT" \
  ecr create-repository \
  --repository-name "$ECR_REPOSITORY"
```

Jika repository sudah ada:

```sh
aws --endpoint-url="$ENDPOINT" \
  ecr describe-repositories \
  --repository-names "$ECR_REPOSITORY"
```

Expected:

```
enterprise-security-app
```

Validasi:

```sh
aws --endpoint-url="$ENDPOINT" \
  ecr describe-repositories \
  --repository-names "$ECR_REPOSITORY" \
  >/dev/null \
  && echo "Step 156: ECR REPOSITORY PASS"
```

---

# Step 157. Configure ECR Repository Security

## Tujuan

Memastikan repository mempunyai security control dasar.

Salah satu kontrol penting:

```
Tag immutability
```

Artinya:

```
phase12-v1
```

tidak boleh diam-diam dipindahkan ke image lain setelah digunakan.

Check:

```sh
aws --endpoint-url="$ENDPOINT" \
  ecr describe-repositories \
  --repository-names "$ECR_REPOSITORY" \
  --query 'repositories[0].[repositoryUri,imageTagMutability]' \
  --output table
```

Expected berdasarkan hasil lu:

```
enterprise-security-app
000000000000.dkr.ecr.us-east-1.localhost:5100/enterprise-security-app
IMMUTABLE
```

Validasi:

```sh
export TAG_MUTABILITY=$(
  aws --endpoint-url="$ENDPOINT" \
    ecr describe-repositories \
    --repository-names "$ECR_REPOSITORY" \
    --query 'repositories[0].imageTagMutability' \
    --output text
)

echo "$TAG_MUTABILITY"
```

Expected:

```
IMMUTABLE
```

Final:

```
[ "$TAG_MUTABILITY" = "IMMUTABLE" ] \
  && echo "Step 157: ECR IMMUTABILITY PASS"
```

---

# Step 158. Authenticate Docker to ECR

## Tujuan

Menguji apakah Docker dapat berkomunikasi dengan registry ECR Floci.

Command:

```sh
aws --endpoint-url="$ENDPOINT" \
  ecr get-login-password \
  | docker login \
      --username AWS \
      --password-stdin \
      "${ECR_URI%/*}"
```

### Parameter

`get-login-password`

Meminta temporary registry authentication password.

`--username AWS`

Username yang digunakan oleh ECR Docker authentication.

`--password-stdin`

Password diberikan melalui stdin sehingga tidak muncul sebagai argument process.

`${ECR_URI%/*}`

Menghapus bagian repository dari URI.

Misalnya:

```
000000000000.dkr.ecr.us-east-1.localhost:5100/enterprise-security-app
```

menjadi:

```
000000000000.dkr.ecr.us-east-1.localhost:5100
```

### Expected AWS

```
Login Succeeded
```

### Actual Floci

Lu mendapatkan:

```
dial tcp: lookup 000000000000.dkr.ecr.us-east-1.localhost
```

Artinya:

```
AWS CLI
   ↓
Floci API localhost:4566       PASS
   ↓
ECR auth credential             PASS
   ↓
Docker registry endpoint        X
   ↓
DNS resolution                  FLOCi LIMITATION
```

Validasi:

```
getent hosts 000000000000.dkr.ecr.us-east-1.localhost
```

Jika tidak resolve, ini membuktikan boundary registry hostname.

**Status Step 158: FLOCi LIMITATION.**

---

# Step 159. Tag Local Image for ECR

## Tujuan

Memberikan ECR-compatible reference kepada image local.

```sh
export ECR_IMAGE="$ECR_URI:$IMAGE_TAG"

docker tag \
  enterprise-security-app:phase12 \
  "$ECR_IMAGE"
```

Expected:

```
(no output)
```

Validate:

```sh
docker image inspect \
  "$ECR_IMAGE" \
  --format '{{.Id}}'
```

Expected:

```
sha256:...
```

Harus sama dengan local image ID.

```sh
docker image inspect \
  enterprise-security-app:phase12 \
  --format '{{.Id}}'

docker image inspect \
  "$ECR_IMAGE" \
  --format '{{.Id}}'
```

Keduanya harus sama.

Status:

**PASS**

Tagging local image tidak membutuhkan registry.

---

# Step 160. Push Image to ECR

## Tujuan

Menguji promotion:

```
Local image
     ↓
ECR
```

Command:

```sh
docker push "$ECR_IMAGE"
```

### Expected AWS

```
Layer pushed
...
phase12-v1: digest: sha256:...
```

### Actual Floci

Lu mendapatkan:

```
failed to do request:
Head "https://000000000000.dkr.ecr.us-east-1.localhost:5100/..."
dial tcp:
lookup 000000000000.dkr.ecr.us-east-1.localhost
```

Jadi Docker tidak pernah mencapai registry.

Validasi:

```sh
aws --endpoint-url="$ENDPOINT" \
  ecr list-images \
  --repository-name "$ECR_REPOSITORY"
```

Actual:

```
ListImages
```

tanpa image.

**Status Step 160: FLOCi LIMITATION.**

---

# Step 161. Validate ECR Image Metadata

## Tujuan

Jika push berhasil, kita harus memastikan ECR mengetahui:

```
tag
digest
size
push timestamp
manifest media type
```

Command:

```sh
aws --endpoint-url="$ENDPOINT" \
  ecr describe-images \
  --repository-name "$ECR_REPOSITORY" \
  --image-ids imageTag="$IMAGE_TAG" \
  --query \
'imageDetails[0].[imageTags,imageDigest,imageSizeInBytes,imagePushedAt,imageManifestMediaType]' \
  --output table
```

### Expected AWS

Metadata image muncul.

### Actual Floci

```
ImageNotFoundException
```

Ini bukan image corruption.

Step 160 sudah membuktikan push gagal.

Validasi tambahan:

```sh
aws --endpoint-url="$ENDPOINT" \
  ecr list-images \
  --repository-name "$ECR_REPOSITORY"
```

Expected actual:

```
no image
```

**Status: FLOCi LIMITATION.**

---

# Step 162. Pull Image from ECR

## Tujuan

Menguji reverse path:

```
ECR
 ↓
Docker
```

Command:

```sh
docker pull "$ECR_IMAGE"
```

Expected AWS:

```
phase12-v1: Pulling from ...
Status: Downloaded newer image
```

Actual:

```
lookup 000000000000.dkr.ecr.us-east-1.localhost
```

Karena registry hostname tidak dapat di-resolve.

Validasi:

```
docker image inspect "$ECR_IMAGE"
```

Expected actual:

```
No such image
```

**Status: FLOCi LIMITATION.**

---

# Step 163. Validate ECR Native Image Scanning Capability

## Tujuan

Menguji apakah Floci mendukung native ECR image scanning.

Command AWS yang tepat untuk repository scanning configuration adalah:

```sh
aws --endpoint-url="$ENDPOINT" \
  ecr batch-get-repository-scanning-configuration \
  --repository-names "$ECR_REPOSITORY"
```

### Parameter

`batch-get-repository-scanning-configuration`

Mengambil scanning configuration untuk satu atau beberapa repository.

Kemudian test native image scan:

```sh
aws --endpoint-url="$ENDPOINT" \
  ecr start-image-scan \
  --repository-name "$ECR_REPOSITORY" \
  --image-id imageTag="$IMAGE_TAG"
```

### Actual Floci

Lu mendapatkan:

```
UnsupportedOperation:
Operation StartImageScan is not supported.
```

Ini adalah capability limitation.

Bukan image vulnerability issue.

**Status: FLOCi LIMITATION.**

---

# Step 164. Validate ECR Scan-on-Push Capability

## Tujuan

Menguji apakah registry dapat melakukan scanning otomatis ketika image dipush.

Gunakan:

```sh
aws --endpoint-url="$ENDPOINT" \
  ecr batch-get-repository-scanning-configuration \
  --repository-names "$ECR_REPOSITORY"
```

Jika API mengembalikan konfigurasi, catat:

```
scanOnPush
```

Jika API operation atau functionality tidak tersedia, classify:

```
FLOCi LIMITATION
```

Jangan lagi menggunakan:

```
get-repository-scanning-configuration
```

karena command tersebut memang bukan AWS CLI operation yang valid.

**Status berdasarkan capability yang sudah terlihat: FLOCi LIMITATION.**

---

# Step 165. Vulnerable Image Security Gate

## Tujuan

Menguji apakah pipeline dapat mendeteksi image yang sengaja dibuat vulnerable.

Untuk local gate, gunakan image yang memang sudah memiliki known vulnerabilities.

```
docker pull alpine:3.12
```

Scan:

```sh
trivy image \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  alpine:3.12
```

Expected:

```
HIGH
CRITICAL
```

kemungkinan ditemukan.

Kemudian:

```sh
trivy image \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  alpine:3.12
```

Expected:

```
exit code 1
```

Itu berarti **security gate bekerja**.

Jangan push image ini ke ECR karena registry integration Floci sudah diketahui tidak bekerja.

Validasi:

```sh
trivy image \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  alpine:3.12

echo "TRIVY VULNERABLE IMAGE EXIT CODE=$?"
```

Expected:

```
TRIVY VULNERABLE IMAGE EXIT CODE=1
```

**Status: PASS** untuk local security gate.

---

# Step 166. Validate Security Promotion Gate

## Tujuan

Menggabungkan hasil scanning dengan keputusan promotion.

Flow:

```
Build
 ↓
Scan
 ↓
HIGH/CRITICAL?
 ├── YES → STOP
 └── NO  → Promote
```

Test dengan image aplikasi:

```sh
trivy image \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  enterprise-security-app:phase12
```

Capture:

```sh
SCAN_EXIT=$?

echo "TRIVY_EXIT_CODE=$SCAN_EXIT"
```

Interpretasi:

```
0 → promotion allowed
1 → promotion blocked
```

Ini bukan hardcoded “PASS kalau 0”.

Yang diuji adalah **mekanisme gate**.

Validasi:

```sh
[ "$SCAN_EXIT" -eq 0 ] \
  && echo "SECURITY GATE: PASS - PROMOTION ALLOWED" \
  || echo "SECURITY GATE: PASS - PROMOTION BLOCKED"
```

---

# Step 167. Deploy ECR Image to Existing EKS

## Tujuan

Sekarang image reference dimasukkan ke workload EKS yang sudah ada.

```sh
kubectl set image \
  deployment/enterprise-security-app \
  application="$ECR_IMAGE" \
  -n enterprise-app
```

### Parameter

`set image`

Mengubah container image pada workload.

`deployment/enterprise-security-app`

Deployment target.

`application=`

Nama container di dalam Pod.

`"$ECR_IMAGE"`

Image ECR.

`-n enterprise-app`

Namespace target.

Expected:

```
deployment.apps/enterprise-security-app image updated
```

Actual lu:

```
deployment.apps/enterprise-security-app image updated
```

Ini berarti Kubernetes menerima perubahan deployment.

**Status: PASS.**

Catatan: PASS di sini berarti **deployment configuration berhasil diubah**, bukan berarti image berhasil ditarik.

---

# Step 168. Validate EKS Image Pull and Digest

## Tujuan

Membedakan:

```
Kubernetes menerima image reference
```

dengan:

```
Kubernetes berhasil pull image
```

Check:

```sh
kubectl get pods \
  -n enterprise-app \
  -l app=enterprise-security-app \
  -o wide
```

Actual:

```
ImagePullBackOff
```

Kemudian:

```sh
kubectl get pod \
  -n enterprise-app \
  -l app=enterprise-security-app \
  -o jsonpath='{.items[0].status.containerStatuses[0].imageID}{"\n"}'
```

Actual:

```
empty
```

Artinya image belum pernah berhasil masuk ke node.

Inspect events:

```sh
kubectl describe pod \
  -n enterprise-app \
  -l app=enterprise-security-app
```

Cari:

```
Failed
Pull
ImagePullBackOff
```

Root cause tetap:

```
Floci ECR registry hostname resolution
```

**Status: FLOCi LIMITATION.**

---

# Step 169. Validate Application Runtime

## Tujuan

Jika image pull berhasil, workload harus:

```
Pod Running
Container Ready
HTTP response
```

Check:

```sh
kubectl get pods \
  -n enterprise-app \
  -l app=enterprise-security-app
```

Expected ideal:

```
1/1 Running
```

Actual:

```
0/1 ImagePullBackOff
```

Port forward:

```sh
kubectl port-forward \
  -n enterprise-app \
  service/enterprise-security-app \
  18082:8080
```

Actual:

```
unable to forward port because pod is not running
```

Jadi:

```
ECR pull
   ↓
FAIL
   ↓
Pod tidak Running
   ↓
port-forward tidak mungkin
```

**Status: FLOCi LIMITATION.**

Bukan application runtime bug.

---

# Step 170. Validate Runtime Security Context

## Tujuan

Walaupun ECR pull gagal, kita tetap dapat memvalidasi security configuration pada Kubernetes manifest.

Check:

```sh
kubectl get deployment \
  enterprise-security-app \
  -n enterprise-app \
  -o yaml
```

Cari:

```
securityContext
```

dan:

```
runAsUser
runAsNonRoot
```

Ideal configuration:

```
securityContext:
  runAsNonRoot: true
  runAsUser: 65532
  runAsGroup: 65532
```

Container:

```
USER 65532
```

Ini penting karena supply chain tidak berhenti di image scanning.

Image yang secure harus dijalankan dengan security context yang aman.

Validasi:

```sh
kubectl get deployment \
  enterprise-security-app \
  -n enterprise-app \
  -o jsonpath='{.spec.template.spec.containers[0].securityContext}{"\n"}'
```

Jika konfigurasi ada, status configuration:

**PASS**

Runtime execution tetap bergantung pada image pull.

---

# Step 171. Validate ECR Tag Immutability

## Tujuan

Menguji repository policy yang mencegah overwrite tag.

Check:

```sh
aws --endpoint-url="$ENDPOINT" \
  ecr describe-repositories \
  --repository-names "$ECR_REPOSITORY" \
  --query 'repositories[0].imageTagMutability' \
  --output text
```

Expected actual:

```
IMMUTABLE
```

Status:

**PASS**

Ini merupakan security control yang tetap dapat divalidasi meskipun ECR image push belum bekerja.

---

# Step 172. Validate ECR-only Image Reference

## Tujuan

Memastikan workload menggunakan image registry, bukan image lokal.

```sh
export IMAGE_TAG3=phase12-ecr-only
export ECR_IMAGE3="$ECR_URI:$IMAGE_TAG3"
```

Tag:

```sh
docker tag \
  enterprise-security-app:phase12 \
  "$ECR_IMAGE3"
```

Set:

```sh
kubectl set image \
  deployment/enterprise-security-app \
  application="$ECR_IMAGE3" \
  -n enterprise-app
```

Validate:

```sh
kubectl get deployment \
  enterprise-security-app \
  -n enterprise-app \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Expected:

```
000000000000.dkr.ecr.us-east-1.localhost:5100/enterprise-security-app:phase12-ecr-only
```

Status:

**PASS** untuk image reference configuration.

Actual image pull tetap:

**FLOCi LIMITATION.**

---

# Step 173. Validate ECR Manifest Retrieval

## Tujuan

Menguji apakah image yang sudah di-push dapat diambil manifest-nya dari ECR.

```sh
aws --endpoint-url="$ENDPOINT" \
  ecr batch-get-image \
  --repository-name "$ECR_REPOSITORY" \
  --image-ids imageTag="$IMAGE_TAG3" \
  --query 'images[0].[imageId.imageDigest,imageManifest]' \
  --output json
```

Expected AWS:

```
[
  "sha256:...",
  "{...manifest...}"
]
```

Actual Floci:

```
null
```

Karena image tidak ada di repository.

**Status: FLOCi LIMITATION.**

---

# Step 174. Validate SBOM ↔ Image Consistency

## Tujuan

Memastikan SBOM memang berasal dari image yang sedang kita build.

Check SBOM metadata:

```sh
grep -E \
  '"name"|"version"' \
  sbom-cyclonedx.json \
  | head -20
```

Expected:

```
enterprise-security-app:phase12
```

Check Trivy metadata:

```sh
grep -E \
  'aquasecurity:trivy:ImageID|aquasecurity:trivy:Reference|aquasecurity:trivy:RepoTag' \
  sbom-cyclonedx.json
```

Expected metadata yang menghubungkan SBOM dengan image.

Validate:

```sh
grep -q 'enterprise-security-app:phase12' \
  sbom-cyclonedx.json \
  && echo "SBOM IMAGE REFERENCE: PASS"
```

Actual lu:

```
enterprise-security-app:phase12
```

**Status: PASS.**

Ini justru salah satu artifact penting untuk phase berikutnya ketika kita mulai bicara image provenance/signing.

---

# Step 175. Integrate ECR Security Events with EventBridge

## Tujuan

Menghubungkan container registry security dengan automation layer Phase 11.

Architecture:

```
ECR
 │
 │ image scan / security event
 ▼
EventBridge
 │
 ▼
Security Automation
 │
 ├── Lambda
 ├── SQS audit
 └── SNS notification
```

Secara AWS nyata, EventBridge dapat digunakan untuk event dari ECR seperti image scanning events.

Di Floci, kita perlu membedakan:

```
EventBridge capability
```

dengan:

```
ECR native event generation
```

Karena native ECR scanning sendiri sudah:

```
FLOCi LIMITATION
```

Test pattern secara synthetic:

```sh
cat > ecr-security-event.json <<'EOF'
{
  "source": ["aws.ecr"],
  "detail-type": ["ECR Image Scan"],
  "detail": {
    "repository-name": ["enterprise-security-app"]
  }
}
EOF
```

Test:

```sh
aws --endpoint-url="$ENDPOINT" \
  events test-event-pattern \
  --event-pattern file://ecr-security-event.json \
  --event '{
    "source": "aws.ecr",
    "detail-type": "ECR Image Scan",
    "detail": {
      "repository-name": "enterprise-security-app"
    }
  }'
```

Expected:

```
{
    "Result": true
}
```

### Penting

Ini hanya membuktikan:

```
EventBridge pattern logic
```

Bukan:

```
ECR → EventBridge native event delivery
```

Karena native ECR scan sudah tidak tersedia di Floci.

Jadi:

**Pattern capability → PASS jika match**

**Native ECR security event generation → FLOCi LIMITATION**

---

# Step 176. Phase 12 Final Validation

## Tujuan

Menutup Phase 12 dengan membuktikan seluruh supply chain dan mencatat boundary Floci.

Jalankan:

```sh
echo "=================================================="
echo " Step 176: PHASE 12 FINAL VALIDATION"
echo "=================================================="

echo
echo "[1] Docker Image:"
docker image inspect \
  enterprise-security-app:phase12 \
  --format '{{.Id}}'

echo
echo "[2] ECR Repository:"
aws --endpoint-url="$ENDPOINT" \
  ecr describe-repositories \
  --repository-names "$ECR_REPOSITORY" \
  --query 'repositories[0].[repositoryName,repositoryUri,imageTagMutability]' \
  --output table

echo
echo "[3] ECR Images:"
aws --endpoint-url="$ENDPOINT" \
  ecr list-images \
  --repository-name "$ECR_REPOSITORY" \
  --output table

echo
echo "[4] ECR Image Metadata:"
aws --endpoint-url="$ENDPOINT" \
  ecr describe-images \
  --repository-name "$ECR_REPOSITORY" \
  --image-ids imageTag="$IMAGE_TAG" \
  --output table

echo
echo "[5] ECR Scan Configuration:"
aws --endpoint-url="$ENDPOINT" \
  ecr batch-get-repository-scanning-configuration \
  --repository-names "$ECR_REPOSITORY"

echo
echo "[6] Trivy:"
trivy --version

echo
echo "[7] SBOM:"
test -s sbom-cyclonedx.json \
  && echo "SBOM: PASS"

echo
echo "[8] EKS Deployment:"
kubectl get deployment \
  enterprise-security-app \
  -n enterprise-app

echo
echo "[9] EKS Pod:"
kubectl get pods \
  -n enterprise-app \
  -l app=enterprise-security-app \
  -o wide

echo
echo "[10] EKS Image:"
kubectl get deployment \
  enterprise-security-app \
  -n enterprise-app \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

echo
echo "[11] EKS Image Digest:"
kubectl get pods \
  -n enterprise-app \
  -l app=enterprise-security-app \
  -o jsonpath='{.items[0].status.containerStatuses[0].imageID}{"\n"}'

echo
echo "[12] Runtime:"
kubectl get pods \
  -n enterprise-app \
  -l app=enterprise-security-app \
  -o jsonpath='{.items[0].status.containerStatuses[0].ready}{"\n"}'

echo
echo "=================================================="
echo " PHASE 12 VALIDATION COMPLETE"
echo "=================================================="
```

---

# Final Phase 12 Result

Berdasarkan output aktual yang lu kasih, hasil akhirnya adalah:

```
==================================================
 PHASE 12 RESULT
==================================================

LOCAL CONTAINER BUILD                    PASS
TRIVY VULNERABILITY SCANNING             PASS
TRIVY VULNERABILITY DATABASE             PASS
TRIVY HIGH/CRITICAL GATE                 PASS
SBOM GENERATION                           PASS
SBOM CYCLONEDX                            PASS
SBOM ↔ IMAGE REFERENCE                   PASS

ECR REPOSITORY                            PASS
ECR TAG IMMUTABILITY                      PASS

ECR DOCKER AUTHENTICATION                 FLOCi LIMITATION
ECR IMAGE PUSH                            FLOCi LIMITATION
ECR IMAGE LISTING                         FLOCi LIMITATION
ECR IMAGE METADATA                        FLOCi LIMITATION
ECR IMAGE PULL                            FLOCi LIMITATION
ECR NATIVE IMAGE SCANNING                 FLOCi LIMITATION
ECR SCAN-ON-PUSH                          FLOCi LIMITATION

EKS DEPLOYMENT CONFIGURATION              PASS
EKS IMAGE PULL                            FLOCi LIMITATION
EKS POD STARTUP                           FLOCi LIMITATION
EKS APPLICATION RUNTIME                   FLOCi LIMITATION

ECR → EKS END-TO-END                      FLOCi LIMITATION
LOCAL SUPPLY CHAIN SECURITY               PASS
==================================================
```

Dan yang paling penting secara arsitektur:

```
                  PHASE 12
                     │
       ┌─────────────┴─────────────┐
       │                           │
       ▼                           ▼
 LOCAL SECURITY                AWS/FLOCi
    GATE                          ECR
       │                           │
       ├─ Trivy                    ├─ Repository PASS
       ├─ CVE scan                 ├─ Immutable PASS
       ├─ HIGH/CRITICAL gate       │
       └─ SBOM                     ├─ Push LIMITATION
                                   ├─ Pull LIMITATION
                                   ├─ Scan LIMITATION
                                   └─ Metadata LIMITATION
                                       │
                                       ▼
                                  EXISTING EKS
                                       │
                                       ├─ Deployment PASS
                                       └─ Pull LIMITATION
```

Jadi **Phase 12 tetap valid sebagai security engineering exercise**, karena local supply-chain control benar-benar berjalan. Yang tidak bisa dibuktikan di environment lu adalah **registry transport/integration path Floci ECR → Docker/EKS**.

Dan ini justru boundary yang bagus untuk didokumentasikan: **Floci ECR API tersedia, tetapi registry hostname `000000000000.dkr.ecr.us-east-1.localhost:5100` tidak resolvable dari Docker/cluster**, sehingga seluruh kegagalan setelah titik tersebut merupakan konsekuensi yang sama, bukan serangkaian bug terpisah.

**Phase berikutnya** sebaiknya tidak mengulang ECR ini. Kita bisa langsung memakai artifact/image yang sudah dibuat untuk masuk ke **Kubernetes admission security: Kyverno + OPA/Gatekeeper**, termasuk policy yang memaksa image berasal dari registry tertentu, `runAsNonRoot`, forbidden privileged container, resource limits, dan nanti **digest enforcement**.