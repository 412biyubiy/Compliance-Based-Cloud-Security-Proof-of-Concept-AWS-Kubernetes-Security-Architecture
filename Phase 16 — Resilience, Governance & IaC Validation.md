### 1. Phase ini ngapain?

Phase 16 adalah **enterprise finishing layer**.

Kalau Phase 1–15 menjawab:

> “Apakah kita berhasil membangun infrastructure dan security architecture?”

maka Phase 16 menjawab:

> **“Apakah infrastructure tersebut bisa dikelola, dibackup, dipulihkan, di-reconcile, dan dibangun ulang secara reproducible tanpa kehilangan security posture?”**

Ada tiga domain utama:

### Resilience

Menguji:

```
Resource
   ↓
Backup Vault
   ↓
Backup Plan
   ↓
Backup Selection
   ↓
Backup Job
   ↓
Recovery Point
   ↓
Restore Job
```

Jadi bukan cuma memastikan backup configuration ada, tetapi lifecycle backup sampai restore benar-benar dicoba.

### Governance

Menguji:

```
AWS Resources
      ↓
Resource Inventory
      ↓
Tags
      ↓
Tag Filtering
      ↓
Ownership / Environment / Security Classification
```

Contohnya:

```
Environment=production
Owner=platform
Application=enterprise-app
DataClassification=confidential
SecurityTier=tier-1
```

Resource Explorer seharusnya menjadi centralized discovery layer. Namun di environment lu:

```
aws resource-explorer-2 search
→ UnknownOperationException
→ Unknown operation: POST /Search
```

Jadi capability tersebut harus dicatat sebagai **FLOCi LIMITATION**, bukan dianggap project failure.

### IaC / Reproducibility

Menguji:

```
Terraform
   ↓
init
   ↓
validate
   ↓
plan
   ↓
apply
   ↓
state
   ↓
update
   ↓
drift detection
   ↓
import
   ↓
destroy
   ↓
recreate
```

Tujuannya memastikan infrastructure tidak bergantung pada konfigurasi manual yang sudah dibuat di Phase 1–15.

---

# 2. Arsitektur Phase 16

Secara keseluruhan:

```
                         ENTERPRISE PLATFORM
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
       Governance Plane   Resilience Plane    IaC Plane
              │                 │                 │
       Resource Inventory   AWS Backup         Terraform
              │                 │                 │
        ┌─────┴─────┐      ┌────┴─────┐      ┌───┴────────┐
        │           │      │          │      │            │
      Tags    Resource     Vault     Plan   State      Lifecycle
        │       Explorer     │          │      │            │
        │           │        │     Selection  │         Destroy
        │           │        │          │      │            │
        └───────────┘        └────┬─────┘      │         Recreate
                                  │            │
                              Backup Job       │
                                  │            │
                           Recovery Point      │
                                  │            │
                             Restore Job       │
                                  │            │
                                  └──────┬─────┘
                                         │
                                         ▼
                              Enterprise Validation
                                         │
              ┌──────────────────────────┼──────────────────────────┐
              │                          │                          │
              ▼                          ▼                          ▼
       AWS Security               Kubernetes Security       Workload Identity
       Phase 1–12                 Phase 13–14                Phase 15
              │                          │                          │
              └──────────────────────────┼──────────────────────────┘
                                         ▼
                                  Final E2E Validation
```

Jadi Phase 16 **bukan subsystem yang berdiri sendiri**.

Dia menjadi layer yang menguji apakah seluruh hasil Phase 1–15 tetap valid setelah lifecycle operations.

---

# 3. Hubungan dengan Phase sebelumnya

```
Phase 1–7
Networking / Compute / Edge / Database / Storage / Domain
        │
        ▼
Terraform + Inventory + Backup
        │
Phase 8
KMS / Secrets
        │
        ▼
IaC security + state governance
        │
Phase 9–11
Monitoring / Audit / Threat Detection / Response
        │
        ▼
Regression validation
        │
Phase 12
Container Supply Chain
        │
        ▼
ECR regression
        │
Phase 13
EKS + RBAC + IRSA + NetworkPolicy
        │
        ▼
Kubernetes regression
        │
Phase 14
Kyverno + Gatekeeper + Falco
        │
        ▼
Admission/runtime regression
        │
Phase 15
SPIFFE/SPIRE + workload identity
        │
        ▼
SPIRE regression
        │
        ▼
PHASE 16
Resilience + Governance + IaC
        │
        ▼
Enterprise closure
```

Ini penting karena Phase 16 bukan sekadar:

> “coba Terraform.”

Tetapi:

> **“setelah seluruh security architecture dibangun, apakah platform tersebut lifecycle-safe?”**

---

# 4. Final Capability Matrix

Ini tabel yang sudah disesuaikan **dengan hasil aktual Phase 16 lu**.

|No|Capability|Actual Result|Kenapa|
|---|---|---|---|
|1|Phase 1–15 baseline|**PASS**|Phase 16 dimulai dari platform yang sudah healthy|
|2|AWS resource inventory|**PASS**|Resource AWS dapat ditemukan melalui service inventory|
|3|Resource Explorer|**FLOCi LIMITATION**|`resource-explorer-2 search` mengembalikan `UnknownOperationException: Unknown operation: POST /Search`|
|4|Tag creation|**PASS**|Resource dapat diberikan governance metadata|
|5|Tag update|**PASS**|Metadata resource dapat diperbarui|
|6|Tag filtering|**PASS**|Resource dapat difilter berdasarkan tag|
|7|Tag key/value discovery|**PASS**|Tag keys dan values dapat ditemukan untuk governance audit|
|8|Missing-tag detection|**PASS**|Resource tanpa metadata dapat diidentifikasi|
|9|Backup vault|**PASS**|Recovery infrastructure berhasil dibuat|
|10|Backup plan|**PASS**|Policy-based backup berhasil dikonfigurasi|
|11|Backup selection|**PASS**|Resource targeting berhasil dikonfigurasi|
|12|Backup job|**PASS**|Backup lifecycle berhasil dijalankan|
|13|Recovery point|**PASS**|Recovery point berhasil dibuat|
|14|Recovery point inspection|**PASS**|Recovery point dapat diinspeksi|
|15|Restore job|**PASS**|`StartRestoreJob` berhasil didukung dan dijalankan oleh Floci|
|16|Backup deletion protection|**PASS**|Vault/recovery-point lifecycle protection berjalan|
|17|Backup tagging|**PASS**|Backup resource dapat diberi governance metadata|
|18|Terraform provider|**PASS**|AWS provider berhasil digunakan terhadap Floci|
|19|Terraform init|**PASS**|Provider dan Terraform environment berhasil diinisialisasi|
|20|Terraform validate|**PASS**|Konfigurasi Terraform valid|
|21|Terraform plan|**PASS**|Desired state berhasil dihitung|
|22|Terraform apply|**PASS**|Resource berhasil diprovision melalui IaC|
|23|Terraform read/state|**PASS**|Terraform state dapat membaca dan merepresentasikan resource|
|24|Terraform update|**PASS**|Perubahan desired state berhasil diterapkan|
|25|Terraform drift detection|**PASS**|Perubahan di luar Terraform dapat dideteksi melalui plan|
|26|Terraform import|**PASS/FLOCi LIMITATION**|Import berhasil diuji; capability dapat bergantung pada resource/provider compatibility|
|27|Terraform destroy|**PASS**|Resource berhasil dihancurkan melalui Terraform|
|28|Terraform recreate|**PASS/FLOCi LIMITATION**|Recreate berhasil diuji; compatibility bergantung pada resource yang digunakan|
|29|Security regression|**PASS**|Lifecycle IaC tidak merusak security controls|
|30|Kubernetes regression|**PASS**|EKS workload/security layer tetap healthy|
|31|SPIRE regression|**PASS**|Phase 15 workload identity layer tetap healthy|
|32|Final E2E infrastructure validation|**PASS/FLOCi LIMITATION**|Enterprise lifecycle berhasil divalidasi, dengan limitation yang berasal dari capability Floci tertentu seperti Resource Explorer|

---

## 5. Dua hasil yang paling penting

### Resource Explorer

Ini bukan:

```
FAIL
```

karena command lu bukan salah dan infrastructure project bukan broken.

Hasil aktual:

```
aws resource-explorer-2 search ...

UnknownOperationException:
Unknown operation: POST /Search
```

Artinya:

```
AWS Resource Explorer API
        │
        ▼
Floci Resource Explorer implementation
        │
        ├── capability/resource support
        │
        └── Search API ❌
```

Jadi:

**Classification: `FLOCi LIMITATION`**

Dan jangan tulis:

> Resource inventory gagal.

Karena inventory AWS sendiri **PASS**.

Yang gagal adalah **centralized Resource Explorer Search API**.

---

### Restore Job

Ini justru kebalikannya.

Expected awal kita:

```
Restore Job → FLOCi LIMITATION
```

Tetapi hasil aktual:

```
StartRestoreJob
      ↓
SUCCESS
```

Maka final classification harus:

**`PASS`**

Ini memperkuat bahwa Phase 16 benar-benar melakukan capability validation, bukan sekadar menerima limitation berdasarkan dokumentasi awal.

---

# 6. Makna akhir Phase 16

Setelah Phase 16, lifecycle enterprise lu secara konseptual menjadi:

```
                BUILD
                  │
                  ▼
          Infrastructure
                  │
                  ▼
             SECURITY
                  │
                  ▼
        Kubernetes Workload
                  │
                  ▼
       SPIFFE/SPIRE Identity
                  │
                  ▼
             GOVERNANCE
                  │
             ┌────┴────┐
             ▼         ▼
           Tags     Inventory
             │
             ▼
           BACKUP
             │
             ▼
       Recovery Point
             │
             ▼
           RESTORE
             │
             ▼
         TERRAFORM
             │
       ┌─────┼─────┐
       ▼     ▼     ▼
     Plan  Drift  State
       │
       ▼
     Destroy
       │
       ▼
     Recreate
       │
       ▼
 SECURITY REGRESSION
       │
       ├── AWS Security
       ├── EKS
       ├── Kyverno
       ├── Gatekeeper
       ├── Falco
       └── SPIRE
              │
              ▼
       ENTERPRISE VALIDATION
```

Jadi **Phase 16 adalah layer yang membuktikan platform bukan cuma “berhasil dibangun”, tetapi juga punya lifecycle management**.

Hasil akhirnya bukan 32 capability semuanya harus PASS. Yang lebih penting adalah:

```
PASS
  = capability benar-benar terbukti bekerja

FLOCi LIMITATION
  = capability sudah diuji tetapi emulator memang membatasi/berbeda

FAIL
  = capability seharusnya bisa bekerja tetapi implementation/test kita bermasalah
```

Dan dari hasil aktual yang lu kasih, **Resource Explorer adalah satu-satunya limitation yang jelas di matrix utama**, sementara **Restore Job harus dinaikkan menjadi PASS**.

# Step 269. Prepare Phase 16 Workspace

## Tujuan

Membuat workspace khusus Phase 16 untuk seluruh artifact:

```
~/phase16/
├── backup/
├── inventory/
├── terraform/
├── reports/
└── manifests/
```

## 269.1 Create workspace

```bash
mkdir ~/phase16
cd ~/phase16

mkdir -p backup inventory terraform reports manifests
```

### Parameter

- `mkdir` → membuat directory.
- `~` → home directory user.
- `-p` → membuat parent directory bila belum ada.
- `cd` → berpindah ke workspace.

## 269.2 Environment

```bash
export ENDPOINT=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_ACCOUNT_ID=000000000000
export CLUSTER_NAME=enterprise-eks
```

### Parameter

- `ENDPOINT` → endpoint Floci.
- `AWS_DEFAULT_REGION` → region default AWS CLI.
- `AWS_REGION` → region SDK/provider.
- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` → credential lokal.
- `AWS_ACCOUNT_ID` → account ID simulator.
- `CLUSTER_NAME` → cluster EKS Floci.

## 269.3 Validation

```bash
pwd

aws sts get-caller-identity \
  --endpoint-url "$ENDPOINT"

kubectl cluster-info

kubectl get nodes
```

Expected:

```
/root/phase16
```

dan:

```
Account = 000000000000
Region  = us-east-1
Cluster = reachable
Node    = Ready
```

### Validation

|Test|Expected|
|---|---|
|Workspace|PASS|
|Floci endpoint|PASS|
|AWS API|PASS|
|Kubernetes API|PASS|
|Node|PASS|

---

# Step 270. Freeze Phase 15 Baseline

## Tujuan

Sebelum melakukan perubahan apa pun, kita capture kondisi Phase 15.

Ini penting supaya nanti kalau Terraform/backup test membuat perubahan, kita bisa membandingkan sebelum dan sesudah.

## 270.1 SPIRE baseline

```bash
kubectl get pods -n spire -o wide \
  | tee reports/pre-phase16-spire-pods.txt

kubectl get clusterspiffeid -A \
  | tee reports/pre-phase16-spiffeids.txt
```

## 270.2 Identity baseline

```bash
kubectl get pods \
  -n phase15-identity \
  -o wide \
  | tee reports/pre-phase16-identity-pods.txt
```

## 270.3 NetworkPolicy baseline

```bash
kubectl get networkpolicy -A \
  | tee reports/pre-phase16-networkpolicy.txt
```

## 270.4 Admission/runtime baseline

```bash
kubectl get pods -A | grep -Ei 'kyverno|gatekeeper|falco' \
  | tee reports/pre-phase16-security-stack.txt
```

## Validation

```bash
kubectl get pods -n spire
kubectl get pods -n phase15-identity
```

Expected:

```
SPIRE Server     Running
SPIRE Agent      Running
identity-client  Running
identity-server  Running
```

Kalau ada perubahan dari Phase 15, **catat sebagai baseline deviation**, jangan langsung diperbaiki secara blind.

---

# Step 271. AWS Resource Inventory Baseline

## Tujuan

Kita mulai dari pertanyaan governance paling basic:

> "Sebenarnya infrastructure kita sekarang terdiri dari resource apa saja?"

## 271.1 EC2

```bash
aws ec2 describe-instances \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION" \
  > inventory/ec2.json
```

### Parameter

- `describe-instances` → mengambil metadata instance.
- `--endpoint-url` → mengarahkan AWS CLI ke Floci.
- `--region` → region target.
- `>` → menyimpan output JSON.

## 271.2 VPC

```bash
aws ec2 describe-vpcs \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION" \
  > inventory/vpcs.json
```

## 271.3 Subnets

```bash
aws ec2 describe-subnets \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION" \
  > inventory/subnets.json
```

## 271.4 Security Groups

```bash
aws ec2 describe-security-groups \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION" \
  > inventory/security-groups.json
```

## 271.5 S3

```bash
aws s3api list-buckets \
  --endpoint-url "$ENDPOINT" \
  > inventory/s3.json
```

## 271.6 RDS

```bash
aws rds describe-db-instances \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION" \
  > inventory/rds.json
```

## 271.7 IAM

```bash
aws iam list-roles \
  --endpoint-url "$ENDPOINT" \
  > inventory/iam-roles.json
```

## Validation

```bash
ls -lh inventory/
```

Expected:

```
ec2.json
vpcs.json
subnets.json
security-groups.json
s3.json
rds.json
iam-roles.json
```

### Status

**PASS** apabila service inventory dapat dikumpulkan.

Kalau suatu API resource tertentu tidak tersedia, kita test command-nya dan kategorikan hasil aktual.

---

# Step 272. Centralized Resource Inventory dengan Resource Explorer

## Tujuan

Step sebelumnya inventory per service.

Sekarang kita test centralized discovery:

```
Resource Explorer
       │
       ├── EC2
       ├── S3
       ├── IAM/resource
       ├── EKS
       └── other indexed resources
```

Floci menyediakan Resource Explorer 2 untuk search/list resource dari service yang di-enable. [Floci](https://floci.io/floci/services/resource-explorer/?utm_source=chatgpt.com)

## 272.1 Create index

```bash
aws resource-explorer-2 create-index \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

## 272.2 Check index

```bash
aws resource-explorer-2 get-index \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

## 272.3 List supported resource types

```bash
aws resource-explorer-2 list-supported-resource-types \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

## 272.4 Search

```bash
aws resource-explorer-2 search \
  --query-string "*" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION" \
  > inventory/resource-explorer.json
```

### Parameter

- `create-index` → mengaktifkan index.
- `get-index` → melihat status index.
- `list-supported-resource-types` → melihat jenis resource yang dapat di-index.
- `search` → mencari resource.
- `--query-string "*"` → wildcard discovery.
- `>` → menyimpan inventory.

## Validation

```bash
cat inventory/resource-explorer.json
```

Expected:

```
Resources discovered
```

### PASS condition

Resource Explorer berhasil membuat index dan mengembalikan resource.

---

# Step 273. Establish Tagging Standard

## Tujuan

Kita menetapkan metadata standar project:

```
Environment
Owner
Application
DataClassification
SecurityTier
```

Contoh:

```
Environment       = production
Owner             = security-platform
Application       = enterprise-app
DataClassification= confidential
SecurityTier      = tier-1
```

**Catatan:** nilai di atas adalah contoh baseline project, bukan berarti semua resource harus memakai nilai persis tersebut. Resource individual harus menggunakan nilai yang sesuai.

---

# Step 274. Test Resource Tagging API

## Tujuan

Membuktikan centralized tagging.

Floci mendukung:

```text
TagResources
UntagResources
GetResources
GetTagKeys
GetTagValues
```
Ambil salah satu ARN resource nyata terlebih dahulu.

Contoh:

```bash
export TEST_RESOURCE_ARN="$(aws ec2 describe-vpcs \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION" \
  --filters "Name=tag:Name,Values=enterprise-vpc" \
  --query 'Vpcs[0].VpcId' \
  --output text)"
```

Karena output di atas baru VPC ID, kita bentuk ARN:

```bash
export TEST_RESOURCE_ARN="arn:aws:ec2:${AWS_REGION}:${AWS_ACCOUNT_ID}:vpc/${TEST_RESOURCE_ARN}"

echo "$TEST_RESOURCE_ARN"
```

## Apply tags

```bash
aws resourcegroupstaggingapi tag-resources \
  --resource-arn-list "$TEST_RESOURCE_ARN" \
  --tags Environment=production,Owner=security-platform,Application=enterprise-app,DataClassification=confidential,SecurityTier=tier-1 \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

### Parameter

- `resource-arn-list` → resource yang akan diberi tag.
- `--tags` → key/value metadata.
- `Environment` → lifecycle environment.
- `Owner` → accountable team.
- `Application` → aplikasi pemilik.
- `DataClassification` → sensitivitas data.
- `SecurityTier` → criticality/security level.

## Validation

```bash
aws resourcegroupstaggingapi get-resources \
  --resource-arn-list "$TEST_RESOURCE_ARN" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected:

```
Environment
Owner
Application
DataClassification
SecurityTier
```

---

# Step 275. Tag Filtering

## Tujuan

Governance bukan cuma memberi tag.

Kita harus bisa bertanya:

> "Resource mana yang Environment=production?"

## Command

```bash
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=Environment,Values=production \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

## Test Owner

```bash
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=Owner,Values=security-platform \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

## Test multiple filters

```bash
aws resourcegroupstaggingapi get-resources \
  --tag-filters \
    Key=Environment,Values=production \
    Key=SecurityTier,Values=tier-1 \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

### Validation

Expected resource harus memenuhi filter yang diminta.

**PASS** jika centralized filtering bekerja.

---

# Step 276. Tag Key / Value Governance Discovery

## Tujuan

Mengetahui taxonomy tag yang benar-benar digunakan.

## 276.1 Tag keys

```bash
aws resourcegroupstaggingapi get-tag-keys \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

## 276.2 Tag values

```bash
aws resourcegroupstaggingapi get-tag-values \
  --key Environment \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

## Validation

Expected:

```
Environment
Owner
Application
DataClassification
SecurityTier
```

dan values seperti:

```
production
```

### Kenapa penting?

Ini memungkinkan governance audit:

```
Expected taxonomy
       ↓
Actual tag keys
       ↓
Deviation
```

---

# Step 277. Missing Tag Detection

## Tujuan

Sekarang kita melakukan security/governance negative test.

Buat satu resource test tanpa governance tag.

Contoh S3:

```bash
aws s3api create-bucket \
  --bucket phase16-untagged-test \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Jangan beri tag.

## Check

```bash
aws resourcegroupstaggingapi get-resources \
  --resource-type-filters s3 \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

## Expected

Resource:

```
phase16-untagged-test
```

harus dapat ditemukan sebagai resource yang belum memenuhi tagging baseline.

### Validation

Ini bukan failure.

Justru:

```
Untagged resource detected
        ↓
Governance control works
```

Setelah test:

```bash
aws s3api delete-bucket \
  --bucket phase16-untagged-test \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

---

# Step 278. AWS Backup Capability Discovery

## Tujuan

Sebelum membuat backup plan, kita cek apa yang Floci expose.

## 278.1 Supported resource types

```bash
aws backup get-supported-resource-types \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Floci saat ini mendokumentasikan resource type seperti:

```text
S3
RDS
DynamoDB
EFS
EC2
EBS
Aurora
DocumentDB
Neptune
FSx
VirtualMachine
```
## Validation

Simpan output:

```bash
aws backup get-supported-resource-types \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION" \
  > backup/supported-resource-types.json
````

Expected:

```
supported backup resource types returned
```

---

# Step 279. Create Backup Vault

## Tujuan

Vault adalah target penyimpanan logical recovery point.

Architecture:

```
Resource
   │
   ▼
Backup Job
   │
   ▼
phase16-vault
```

## Command

```bash
aws backup create-backup-vault \
  --backup-vault-name phase16-vault \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

### Parameter

- `create-backup-vault` → membuat vault.
- `--backup-vault-name` → nama vault.
- `--endpoint-url` → Floci.
- `--region` → region.

## Validation

```bash
aws backup describe-backup-vault \
  --backup-vault-name phase16-vault \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected:

```
BackupVaultName = phase16-vault
```

---

# Step 280. Create Backup Plan

## Tujuan

Backup harus policy-based, bukan backup manual random.

Buat file:

```bash
cat > backup/backup-plan.json <<'EOF'
{
  "BackupPlanName": "phase16-enterprise-backup",
  "Rules": [
    {
      "RuleName": "daily-local",
      "TargetBackupVaultName": "phase16-vault",
      "ScheduleExpression": "cron(0 1 * * ? *)"
    }
  ]
}
EOF
```

## Create

```bash
aws backup create-backup-plan \
  --backup-plan file://backup/backup-plan.json \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

## Validation

```bash
aws backup list-backup-plans \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected:

```
phase16-enterprise-backup
```

---

# Step 281. Backup Selection

## Tujuan

Menentukan resource yang masuk backup plan.

Kita jangan asal backup semua resource.

Ambil ARN resource test yang sudah ada.

Misalnya S3:

```bash
export BACKUP_BUCKET=phase16-backup-test
```

Create:

```bash
aws s3api create-bucket \
  --bucket "$BACKUP_BUCKET" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Tag:

```bash
aws s3api put-bucket-tagging \
  --bucket "$BACKUP_BUCKET" \
  --tagging '{"TagSet":[{"Key":"Environment","Value":"production"},{"Key":"Owner","Value":"security-platform"},{"Key":"Application","Value":"enterprise-app"},{"Key":"DataClassification","Value":"confidential"},{"Key":"SecurityTier","Value":"tier-1"}]}' \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Set ARN:

```bash
export BACKUP_RESOURCE_ARN="arn:aws:s3:::$BACKUP_BUCKET"
```

## Validation

```bash
aws s3api get-bucket-tagging \
  --bucket "$BACKUP_BUCKET" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected seluruh governance tags muncul.

---

# Step 282. Start On-Demand Backup

## Tujuan

Sekarang kita benar-benar menjalankan backup job.

Pertama ambil plan ID:

```bash
export BACKUP_PLAN_ID="$(
  aws backup list-backup-plans \
    --endpoint-url "$ENDPOINT" \
    --region "$AWS_REGION" \
    --query 'BackupPlansList[?BackupPlanName==`phase16-enterprise-backup`].BackupPlanId' \
    --output text
)"
```

Kemudian:

```bash
export IAM_ROLE_ARN="arn:aws:iam::000000000000:role/AWSBackupDefaultServiceRole"
```

Lalu test:

```bash
aws backup start-backup-job \
  --backup-vault-name phase16-vault \
  --resource-arn "$BACKUP_RESOURCE_ARN" \
  --iam-role-arn "$IAM_ROLE_ARN" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

### Catatan

ARN IAM role mungkin berbeda di environment lu.

Jadi kalau role tersebut tidak ada, **jangan langsung klasifikasikan FLOCi limitation**.

Check:

```bash
aws iam list-roles \
  --endpoint-url "$ENDPOINT" \
  --query 'Roles[].RoleName'
```

Pilih role backup yang benar atau buat dedicated role jika diperlukan.

---

# Step 283. Validate Backup Job Lifecycle

## Tujuan

Kita tidak cukup melihat job ID.

Kita ingin:

```
CREATED
   ↓
RUNNING
   ↓
COMPLETED
```

Floci mendokumentasikan lifecycle tersebut dan setelah `COMPLETED`, recovery point dibuat di vault. [Floci](https://floci.io/floci/services/backup/?utm_source=chatgpt.com)

Ambil job ID dari output sebelumnya:

```bash
export BACKUP_JOB_ID="$(
  aws backup start-backup-job \
    --backup-vault-name phase16-vault \
    --resource-arn "$BACKUP_RESOURCE_ARN" \
    --iam-role-arn "$IAM_ROLE_ARN" \
    --endpoint-url "$ENDPOINT" \
    --region "$AWS_REGION" \
    --query 'BackupJobId' \
    --output text
)"

# Validasi isi variabelnya
echo "Backup Job ID berhasil disimpan: $BACKUP_JOB_ID"
```

Kemudian:

```bash
aws backup describe-backup-job \
  --backup-job-id "$BACKUP_JOB_ID" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Ulangi:

```bash
sleep 3

aws backup describe-backup-job \
  --backup-job-id "$BACKUP_JOB_ID" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION" \
  --query 'State'
```

Expected:

```
COMPLETED
```

---

# Step 284. Validate Recovery Point

## Tujuan

Membuktikan bahwa backup job menghasilkan recovery artifact.

```bash
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name phase16-vault \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected:

```bash
RecoveryPoints
```

Ambil ARN:

```bash
export RECOVERY_POINT_ARN="$(
  aws backup list-recovery-points-by-backup-vault \
    --backup-vault-name phase16-vault \
    --endpoint-url "$ENDPOINT" \
    --region "$AWS_REGION" \
    --query 'RecoveryPoints[0].RecoveryPointArn' \
    --output text
)"
```

Inspect:

```bash
aws backup describe-recovery-point \
  --backup-vault-name phase16-vault \
  --recovery-point-arn "$RECOVERY_POINT_ARN" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

### Validation

```
Backup Job = COMPLETED
Recovery Point = EXISTS
```

= **PASS**

---

# Step 285. Test Recovery / Restore Capability

```bash
aws backup start-restore-job \
  --recovery-point-arn "$RECOVERY_POINT_ARN" \
  --metadata '{}' \
  --iam-role-arn "$IAM_ROLE_ARN" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

### Expected -> Restore berhasil

```
restore job created
```

→ **PASS**

Lanjutkan describe/list restore job.
# Step 286. Backup Recovery Point Protection Test

## Tujuan

Test lifecycle constraint:

```
Recovery Point exists
       │
       ▼
Delete Vault
```

Coba:

```bash
aws backup delete-backup-vault \
  --backup-vault-name phase16-vault \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected:

```
DeleteBackupVault fails
```

karena vault masih memiliki recovery point. Floci mendokumentasikan constraint tersebut. [Floci](https://floci.io/floci/services/backup/?utm_source=chatgpt.com)

### Ini PASS, bukan failure.

Karena yang sedang kita test adalah:

```
Recovery Point
      ↓
Vault deletion protection
```

---

# Step 287. Backup Resource Tagging

## Tujuan

Governance juga harus berlaku ke backup infrastructure.

Ambil vault ARN dari:

```bash
aws backup describe-backup-vault \
  --backup-vault-name phase16-vault \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Kemudian gunakan ARN yang dikembalikan:

```bash
export VAULT_ARN="arn:aws:backup:us-east-1:000000000000:backup-vault:phase16-vault"
aws backup tag-resource \
  --resource-arn "$VAULT_ARN" \
  --tags Environment=production,Owner=security-platform,Application=enterprise-app,SecurityTier=tier-1 \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Validate:

```bash
aws backup list-tags \
  --resource-arn "$VAULT_ARN" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected:

```
Environment
Owner
Application
SecurityTier
```

---

# Step 288. Terraform Installation & Toolchain Validation

## Tujuan

Sekarang masuk ke IaC.
```bash
# 1. Update package list dan install dependensi yang dibutuhkan
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl

# 2. Unduh dan tambahkan GPG key resmi HashiCorp
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# 3. Tambahkan repository resmi HashiCorp ke daftar sumber apt sistem lu
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# 4. Update ulang dan install terraform
sudo apt-get update && sudo apt-get install -y terraform

# 5. Validasi hasil instalasi
terraform version
which terraform
```



---

# Step 289. Create Terraform Test Project

## Tujuan

Kita **tidak langsung import seluruh enterprise infrastructure**.

Pertama kita buktikan Terraform lifecycle dengan resource isolated.

```bash
mkdir -p ~/phase16/terraform/repro-test
cd ~/phase16/terraform/repro-test
```

Buat provider:

```hcl
cat > provider.tf <<'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
    }
  }
}

provider "aws" {
  region     = "us-east-1"
  access_key = "test"
  secret_key = "test"

  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true

  # WAJIB AKTIFKAN INI UNTUK LOCALSTACK / S3 LOKAL
  s3_use_path_style = true

  endpoints {
    s3 = "http://localhost:4566"
  }
}
EOF
```

Custom service endpoints memang didukung AWS provider untuk AWS-compatible local environments. [Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/custom-service-endpoints?utm_source=chatgpt.com)

---

# Step 290. Terraform Resource Definition

Buat:

```hcl
cat > main.tf <<'EOF'
resource "aws_s3_bucket" "phase16_test" {
  bucket = "phase16-terraform-repro-test"

  tags = {
    Environment        = "test"
    Owner              = "security-platform"
    Application        = "phase16"
    DataClassification = "internal"
    SecurityTier       = "tier-3"
  }
}
EOF
```

### Tujuan

Kita sengaja memakai resource sederhana agar fokusnya:

```
Terraform
   ↓
AWS Provider
   ↓
Floci
   ↓
S3
```

bukan debugging kompleksitas resource.

---

# Step 291. Terraform Init

```bash
terraform init
```

### Parameter

Tidak ada parameter wajib.

Terraform akan:

```
download provider
initialize backend
prepare working directory
```

### Validation

Expected:

```
Terraform has been successfully initialized!
```

---

# Step 292. Terraform Validate

```bash
terraform validate
```

Expected:

```
Success! The configuration is valid.
```

### Arti

Ini hanya membuktikan:

```
HCL syntax
+
provider schema
+
configuration structure
```

**Belum membuktikan Floci API bekerja.**

---

# Step 293. Terraform Plan

```bash
terraform plan -out=phase16.tfplan
```

`-out` menyimpan execution plan ke file sehingga plan yang kita review dapat dipakai untuk apply. Terraform mendokumentasikan `-out` sebagai mekanisme untuk menyimpan plan sebelum apply. [HashiCorp Developer](https://developer.hashicorp.com/terraform/cli/commands/plan?utm_source=chatgpt.com)

Expected:

```
Plan: 1 to add, 0 to change, 0 to destroy.
```

---

# Step 294. Terraform Apply

```bash
terraform apply phase16.tfplan
```

Expected:

```
Apply complete!
Resources: 1 added, 0 changed, 0 destroyed.
```

Validation dari AWS CLI:

```bash
aws s3api head-bucket \
  --bucket phase16-terraform-repro-test \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected:

```
success
```

---

# Step 295. Terraform State Validation

```bash
terraform state list
```

Expected:

```
aws_s3_bucket.phase16_test
```

Inspect:

```bash
terraform state show aws_s3_bucket.phase16_test
```

Expected:

```
id
bucket
tags
```

### Tujuan

Membuktikan:

```
Remote resource
       ↕
Terraform state
```

sudah sinkron.

---

# Step 296. Terraform Update Test

Edit:

```hcl
cat > main.tf <<'EOF'
resource "aws_s3_bucket" "phase16_test" {
  bucket = "phase16-terraform-repro-test"

  tags = {
    Environment        = "production"
    Owner              = "security-platform"
    Application        = "phase16"
    DataClassification = "internal"
    SecurityTier       = "tier-2"
  }
}
EOF
```

Plan:

```bash
terraform plan
```

Expected:

```
1 to change
```

Apply:

```bash
terraform apply -auto-approve
```

Validate:

```bash
aws s3api get-bucket-tagging \
  --bucket phase16-terraform-repro-test \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected tags berubah.

---

# Step 297. Terraform Drift Detection

## Tujuan

Ini salah satu test paling penting.

Kita sengaja mengubah infrastructure **di luar Terraform**.

```bash
aws s3api put-bucket-tagging \
  --bucket phase16-terraform-repro-test \
  --tagging 'TagSet=[
    {Key=Environment,Value=drifted},
    {Key=Owner,Value=unknown}
  ]' \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Sekarang:

```bash
terraform plan
```

Expected:

```
Terraform detects changes
```

Terraform seharusnya menunjukkan bahwa remote state berbeda dari desired configuration.

### Validation

```
Manual change
     ↓
Terraform refresh
     ↓
Drift detected
     ↓
Plan proposes correction
```

= **PASS**

---

# Step 298. Terraform Import Existing Resource

## Tujuan

Ini lebih dekat ke kondisi project sebenarnya.

Infrastructure kita sebagian besar sudah dibuat sebelum Terraform.

Jadi kita test:

```
Existing Floci resource
        ↓
Terraform import
        ↓
Terraform state
```

Terraform memang mendukung import existing resource ke state. Untuk import, resource configuration destination harus tersedia terlebih dahulu. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/import/single-resource?utm_source=chatgpt.com)

Buat:

```bash
mkdir -p ../import-test
cd ../import-test
```

Provider:

```bash
cp ../repro-test/provider.tf .
```

Resource:

```hcl
cat > main.tf <<'EOF'
resource "aws_s3_bucket" "existing" {
  bucket = "phase16-existing-import-test"
}
EOF
```

Create external resource:

```bash
aws s3api create-bucket \
  --bucket phase16-existing-import-test \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Initialize:

```bash
terraform init
```

Import:

```bash
terraform import \
  aws_s3_bucket.existing \
  phase16-existing-import-test
```

Validate:

```bash
terraform state list
```

Expected:

```
aws_s3_bucket.existing
```

---

# Step 299. Terraform Plan After Import

Ini **wajib**.

```bash
terraform plan
```

Expected ideal:

```
No changes.
```

atau hanya perubahan configuration yang memang belum direpresentasikan.

Kalau Terraform ingin:

```
destroy
replace
unexpected modify
```

jangan apply dulu.

Review resource configuration sampai:

```
Existing infrastructure
        ≈
Terraform configuration
        ≈
Terraform state
```

Terraform sendiri menyarankan plan setelah import untuk memastikan konfigurasi sesuai dengan resource yang di-import. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/import/single-resource?utm_source=chatgpt.com)

---

# Step 300. Terraform Destroy / Recreate Test

## Tujuan

Ini pembuktian paling kuat bahwa IaC benar-benar reproducible.

Dari test project:

```bash
cd ~/phase16/terraform/repro-test
```

Destroy:

```bash
terraform destroy -auto-approve
```

Expected:

```
Destroy complete!
Resources: 1 destroyed.
```

Validate:

```bash
aws s3api head-bucket \
  --bucket phase16-terraform-repro-test \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected:

```
Not Found
```

Kemudian recreate:

```bash
terraform apply -auto-approve
```

Expected:

```
Apply complete!
Resources: 1 added.
```

Validate:

```bash
aws s3api head-bucket \
  --bucket phase16-terraform-repro-test \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected:

```
success
```

### Ini membuktikan:

```
Terraform configuration
        │
        ▼
CREATE
        │
        ▼
DESTROY
        │
        ▼
CREATE AGAIN
```

= **reproducibility PASS**

---

# Step 301. Terraform Idempotency

Setelah resource sudah ada:

```bash
terraform plan
```

Expected:

```
No changes.
```

Ini berbeda dengan `validate`.

```
terraform validate
        =
configuration valid

terraform plan
        =
desired state == actual state
```

Terraform plan memang digunakan untuk memeriksa perubahan yang diperlukan tanpa langsung melakukan perubahan. [HashiCorp Developer](https://developer.hashicorp.com/terraform/cli/commands/plan?utm_source=chatgpt.com)

---

# Step 302. Terraform State Backup

## Tujuan

Terraform state sendiri merupakan bagian penting dari IaC control plane.

Check:

```bash
ls -lah
```

Expected:

```bash
terraform.tfstate
terraform.tfstate.backup
```

Backup:

```bash
cp terraform.tfstate \
  ~/phase16/backup/terraform.tfstate.backup
```

Validate:

```bash
ls -lh ~/phase16/backup/terraform.tfstate.backup
```

### Security note

Jangan commit sembarang `terraform.tfstate`.

State dapat berisi data sensitif tergantung resource.

Tambahkan:

```bash
cat > ~/phase16/terraform/.gitignore <<'EOF'
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
*.tfvars
EOF
```

---

# Step 303. Test Terraform Security Boundary

## Tujuan

Karena Phase 8 sudah punya KMS/Secrets Manager dan Phase 13–15 punya identity security, Terraform tidak boleh menjadi jalan bocornya secret.

Test:

```bash
grep -RniE \
  'password|secret|private_key|access_key|secret_key' \
  ~/phase16/terraform \
  --exclude='*.tfstate*' \
  || true
```

Kemudian:

```bash
grep -RniE \
  'AWS_SECRET_ACCESS_KEY|AWS_ACCESS_KEY_ID' \
  ~/phase16/terraform \
  || true
```

### Expected

Tidak ada hardcoded production secret.

Credential test seperti:

```
test
```

boleh karena memang environment emulator.

---

# Step 304. Kubernetes Manifest Reproducibility

Terraform bukan satu-satunya IaC layer.

Kubernetes juga harus reproducible.

Export baseline:

```bash
kubectl get namespace \
  -o yaml > ~/phase16/manifests/namespaces.yaml

kubectl get networkpolicy -A \
  -o yaml > ~/phase16/manifests/networkpolicies.yaml

kubectl get serviceaccount -A \
  -o yaml > ~/phase16/manifests/serviceaccounts.yaml

kubectl get clusterspiffeid -A \
  -o yaml > ~/phase16/manifests/clusterspiffeids.yaml
```

### Validation

```bash
ls -lh ~/phase16/manifests/
```

Expected:

```bash
namespaces.yaml
networkpolicies.yaml
serviceaccounts.yaml
clusterspiffeids.yaml
```

---

# Step 305. Kubernetes Security Regression Test

Setelah semua IaC test:

```bash
kubectl get pods -n spire
kubectl get pods -n phase15-identity
```

Kemudian:

```bash
kubectl get networkpolicy -A
kubectl get clusterspiffeid -A
```

Phase 15 baseline:

```
SPIRE Server
SPIRE Agent
identity workloads
SPIFFE IDs
NetworkPolicy
```

harus tetap tersedia.

Ini penting karena Phase 15 menunjukkan identity chain:

```
Workload
 ↓
SPIRE Attestation
 ↓
SPIFFE ID
 ↓
X.509-SVID
 ↓
Trust Bundle
 ↓
mTLS
```

Phase 15. Workload Identity & …

---

# Step 306. Phase 14 Security Regression

## Kyverno

```bash
kubectl get pods -A | grep -i kyverno
```

## Gatekeeper

```bash
kubectl get pods -A | grep -i gatekeeper
```

## Falco

```bash
kubectl get pods -A | grep -i falco
```

Kemudian ulang minimal satu negative admission test dari Phase 14.

Contoh:

```bash
kubectl apply -f <bad-manifest.yaml>
```

Expected:

```
DENIED
```

Jangan hanya mengecek Pod Kyverno `Running`.

Yang ingin dibuktikan:

```
IaC lifecycle
      ↓
Phase 14 security policy
      ↓
still enforced
```

---

# Step 307. Phase 13 Identity Regression

Check:

```bash
kubectl get serviceaccount -A
```

RBAC:

```bash
kubectl get role,rolebinding,clusterrole,clusterrolebinding -A
```

NetworkPolicy:

```bash
kubectl get networkpolicy -A
```

IRSA/OIDC:

```bash
aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

dan:

```bash
aws eks list-pod-identity-associations \
  --cluster-name "$CLUSTER_NAME" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected:

Phase 13 identity controls masih konsisten dengan baseline.

---

# Step 308. Phase 12 Supply Chain Regression

Kita sudah mengetahui dari Phase 12 bahwa ECR memiliki limitation operasional di environment Floci kita.

Jadi jangan tiba-tiba menyimpulkan ECR failure baru sebagai Phase 16 failure.

Check:

```bash
aws ecr describe-repositories \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Kemudian:

```bash
kubectl get pods -A | grep -Ei 'imagepull|ecr'
```

---

# Step 309. Inventory Governance Report

Sekarang kita gabungkan hasil.

```bash
aws resourcegroupstaggingapi get-tag-keys \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION" \
  > ~/phase16/reports/tag-keys.json

aws resourcegroupstaggingapi get-resources \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION" \
  > ~/phase16/reports/tagged-resources.json
```

Resource Explorer:

```bash
aws resource-explorer-2 search \
  --query-string "*" \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION" \
  > ~/phase16/reports/resource-inventory.json
```

Backup:

```bash
aws backup list-backup-vaults \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION" \
  > ~/phase16/reports/backup-vaults.json

aws backup list-backup-plans \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION" \
  > ~/phase16/reports/backup-plans.json
```

---

# Step 310. Final Backup Validation

```bash
aws backup list-backup-vaults \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"

aws backup list-backup-plans \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"

aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name phase16-vault \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Expected:

```
Vault             EXISTS
Backup Plan       EXISTS
Recovery Point    EXISTS
```

Restore:

```
RESTORE API
   │
   ├── works → PASS
   │
   └── unsupported → FLOCi LIMITATION
```

---

# Step 311. Final Terraform Validation

```bash
cd ~/phase16/terraform/repro-test

terraform fmt -check

terraform validate

terraform plan
```

Expected:

```
terraform fmt      PASS
terraform validate PASS
terraform plan     No changes
```

Kemudian:

```bash
terraform state list
```

Expected resource yang memang dikelola Terraform.

---

# Step 312. Final Resource Inventory

```bash
cd ~/phase16

find inventory -type f -maxdepth 1 -print

find reports -type f -maxdepth 1 -print

find manifests -type f -maxdepth 1 -print
```

Expected:

```
inventory/
reports/
manifests/
backup/
terraform/
```

---

# Step 313. Final EKS Security Validation

```bash
kubectl get nodes

kubectl get pods -n spire

kubectl get pods -n phase15-identity

kubectl get networkpolicy -A

kubectl get clusterspiffeid -A
```

Expected:

```
Nodes                  Ready
SPIRE Server           Running
SPIRE Agent            Running
Identity workloads     Running
NetworkPolicy          Present
ClusterSPIFFEID        Present
```

---

# Step 314. Final Phase 13–15 Regression

Jalankan:

```bash
kubectl get serviceaccount -A

kubectl get role,rolebinding -A

kubectl get networkpolicy -A

kubectl get pods -A | grep -Ei 'kyverno|gatekeeper|falco|spire'
```

Kemudian:

```bash
aws iam list-roles \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"

aws kms list-keys \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"

aws secretsmanager list-secrets \
  --endpoint-url "$ENDPOINT" \
  --region "$AWS_REGION"
```

Ini memastikan Phase 16 tidak menghilangkan:

```
IAM
KMS
Secrets
Kubernetes Identity
Admission
Runtime Security
SPIFFE
```


