# Lab 6: Object Storage Security & the Data Security Lifecycle
**Course:** IKB42603 Cloud Computing Security Essentials  
**Institution:** Universiti Kuala Lumpur (UniKL MIIT)  
**Author / Student:** Muhammad Arif Shafira bin Shahrin Amri  
**Course Instructor:** Prof. Dr. Shahrulniza Musa  
**Platform / Environment:** Kali Linux, AWS CLI v2, LocalStack Pro / Community (Docker-based)  
**Date:** September 2026  

---

## Executive Summary & Overview
Cloud object storage services such as Amazon Simple Storage Service (Amazon S3) form the foundation of cloud-native data architectures. However, misconfigured object access policies and poorly managed data lifecycles represent one of the single most pervasive root causes of real-world cloud data breaches.

This lab investigates the end-to-end **Data Security Lifecycle** (Create, Store, Use, Share, Archive, Destroy) within cloud object storage across eight distinct, structured tasks divided into two main sessions:
1. **Session A (Week 11) — Object Storage & The Exposure Problem:** Data classification tagging, reproducing the archetypal public bucket breach (`Principal: *`), implementing preventative guardrails via Amazon S3 Block Public Access, and resolving conflicts between caller identity policies (IAM) and resource-based bucket policies.
2. **Session B (Week 12) — Protecting, Retaining, and Retiring Data:** Default bucket-level server-side encryption with AWS KMS (SSE-KMS) and S3 Bucket Keys, delegated access via time-bounded presigned URLs and condition key pitfalls (`aws:SecureTransport`), versioning mechanics, delete markers, object-level data remanence, automated lifecycle policies, and cryptographic erasure.

---

## Lab Architecture & Initial Environment Setup

### Architecture Overview
- **Storage Service:** Amazon S3 simulated on LocalStack (`http://localhost:4566`).
- **Identity & Access Management:** LocalStack IAM engine running with strict evaluation (`ENFORCE_IAM=1`).
- **Key Management Service:** AWS KMS generating customer-managed keys (CMK) for envelope encryption.
- **Client Configuration:** AWS CLI v2 configured with endpoint alias `$EP` (`--endpoint-url=http://localhost:4566`) and region `us-east-1`.

```
                  +-------------------------------------------------------------+
                  |                      LOCALSTACK (Docker)                    |
                  |                                                             |
+-------------+   |   +-----------------------+     +-----------------------+   |
|   AWS CLI   |---|-->|       Amazon S3       |<--->|        AWS KMS        |   |
|  (kali@Arif)|   |   | - S3 Bucket           |     | - CMK / SSE-KMS       |   |
+-------------+   |   | - Block Public Access |     | - Cryptographic       |   |
                  |   | - Bucket Policy       |     |   Erasure Engine      |   |
+-------------+   |   | - Lifecycle Rules     |     +-----------------------+   |
| curl / HTTP |---|-->| - Versioning Engine   |                                 |
+-------------+   |   +-----------------------+                                 |
                  |               ^                                             |
                  |               | Authorisation Evaluation                    |
                  |   +-----------------------+                                 |
                  |   |        AWS IAM        |                                 |
                  |   | - DataAnalyst User    |                                 |
                  |   | - Identity Policy     |                                 |
                  |   +-----------------------+                                 |
                  +-------------------------------------------------------------+
```

### Setup Execution
LocalStack container startup and baseline CLI endpoint validation:

```bash
# Clean up any existing instance
docker rm -f localstack 2>/dev/null

# Run LocalStack with IAM enforcement enabled
docker run -d --name localstack -p 4566:4566   -e LOCALSTACK_AUTH_TOKEN=$LOCALSTACK_AUTH_TOKEN   -e ENFORCE_IAM=1   localstack/localstack-pro:latest

# Configure shell environment
export EP='--endpoint-url=http://localhost:4566'
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1

# Validate dummy caller identity
aws $EP sts get-caller-identity
```

*Output:*
```json
{
    "UserId": "AKIAIOSFODNN7EXAMPLE",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```

---

## Session A: Object Storage & The Exposure Problem

### Task 1: Classify the Data Before You Store It

#### Objective
Data security controls must be aligned directly with data sensitivity classification rather than ad-hoc administration. In this task, a dedicated hospital records bucket was provisioned, and three distinct data objects were generated, classified, and tagged with metadata.

#### Execution Steps & Commands
```bash
# Generate unique bucket name
export BUCKET=miit-patient-records-$RANDOM
echo $BUCKET

# Create bucket
aws $EP s3api create-bucket --bucket $BUCKET

# Create sample records reflecting three distinct data classifications
echo 'Ward visiting hours 10am-8pm'                    > public-notice.txt
echo 'Staff duty schedule, week 12'                    > internal-roster.txt
echo 'Patient: Ahmad bin Ali, Diagnosis: confidential' > confidential-record.txt

# Upload objects with object classification tagging
aws $EP s3api put-object --bucket $BUCKET --key public/notice.txt   --body public-notice.txt --tagging 'classification=public'

aws $EP s3api put-object --bucket $BUCKET --key internal/roster.txt   --body internal-roster.txt --tagging 'classification=internal'

aws $EP s3api put-object --bucket $BUCKET --key confidential/record.txt   --body confidential-record.txt --tagging 'classification=confidential'

# Verify bucket contents and object tagging
aws $EP s3api list-objects-v2 --bucket $BUCKET   --query 'Contents[].[Key,Size]' --output table

aws $EP s3api get-object-tagging --bucket $BUCKET --key confidential/record.txt
```

#### Verification & Screenshot Evidence
![Task 1 Evidence](Lab%206%20Task%201.png)

```text
--------------------------------
|        ListObjectsV2         |
+--------------------------+---+
|  confidential/record.txt | 48|
|  internal/roster.txt     | 29|
|  public/notice.txt       | 29|
+--------------------------+---+

{
    "TagSet": [
        {
            "Key": "classification",
            "Value": "confidential"
        }
    ]
}
```

#### Data Classification Mapping Table
| Classification | Who may read it | Impact if leaked | Control implemented |
| :--- | :--- | :--- | :--- |
| **Public** | Anyone (external public, visitors, patients) | Negligible / None (general operational info) | Unrestricted read access via standard prefix or public distribution |
| **Internal** | Authenticated hospital staff & clinical personnel | Moderate (operational disruption, scheduling privacy concerns) | Scoped IAM identity policy / resource policy restricted to authenticated account roles |
| **Confidential** | Strictly authorized medical practitioners / attending physicians | Critical (severe privacy violation, GDPR/PDPA regulatory fines, medical malpractice liability) | Explicit resource policy Deny, SSE-KMS customer-managed key encryption, strict lifecycle retention & cryptographic erasure |

*Note on Object Storage Hierarchy:* In Amazon S3, prefixes such as `confidential/` or `internal/` do not represent physical folders or directories; the namespace is completely flat. Slashes are literal characters within the object key string. Prefix-based policies must be carefully scoped because wildcards (e.g., `*`) easily match across all namespaces.

---

### Task 2: Reproduce the Archetypal Breach

#### Objective
Simulate the classic, catastrophic cloud data breach scenario caused by misconfigured resource policies that assign global access permissions to unauthenticated anonymous callers.

#### Execution Steps & Commands
```bash
# Create an overly permissive public bucket policy
cat > public-policy.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadEverything",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET/*"
  }]
}
JSON

# Apply public policy to the bucket
aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://public-policy.json
aws $EP s3api get-bucket-policy --bucket $BUCKET --query Policy --output text

# Simulate anonymous attacker accessing confidential records over HTTP
curl -s -o leaked.txt -w 'HTTP %{http_code}
'   http://localhost:4566/$BUCKET/confidential/record.txt

cat leaked.txt
```

#### Verification & Screenshot Evidence
![Task 2 Evidence](Lab%206%20Task%202.png)

```text
HTTP 200
Patient: Ahmad bin Ali, Diagnosis: confidential
```

#### Analysis
- **Root Cause:** The entire breach resulted from a single wildcard string: `"Principal": "*"`.
- **Significance:** There was zero exploit payload, no software vulnerability, and no privilege escalation. An unauthenticated external client simply requested an object URI and was served private medical health information directly by the storage service due to an explicit wildcard principal.

---

### Task 3: Remediate with S3 Block Public Access & Least Privilege

#### Objective
Eradicate the vulnerability using AWS S3 Block Public Access (BPA) as a central guardrail, preventing unauthorized public policies from being applied, and replace the flawed policy with a least-privilege resource policy.

#### Execution Steps & Commands
```bash
# 1. Remove the public policy immediately
aws $EP s3api delete-bucket-policy --bucket $BUCKET

# 2. Apply all four Block Public Access flags
aws $EP s3api put-public-access-block --bucket $BUCKET   --public-access-block-configuration     BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# 3. Verify Block Public Access configuration
aws $EP s3api get-public-access-block --bucket $BUCKET

# 4. Construct and apply a least-privilege policy restricted to internal records and account root
cat > least-privilege-policy.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AccountReadInternalOnly",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::000000000000:root"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET/internal/*"
  }]
}
JSON

aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://least-privilege-policy.json
aws $EP s3api get-bucket-policy --bucket $BUCKET --query Policy --output text
```

#### Verification & Screenshot Evidence
![Task 3 Evidence](Lab%206%20Task%203.png)

```json
{
    "PublicAccessBlockConfiguration": {
        "BlockPublicAcls": true,
        "IgnorePublicAcls": true,
        "BlockPublicPolicy": true,
        "RestrictPublicBuckets": true
    }
}
```

#### Guardrail Architecture Analysis
The four S3 Block Public Access settings provide layered defense-in-depth:
1. `BlockPublicAcls`: Blocks new public ACLs from being set on buckets or objects.
2. `IgnorePublicAcls`: Causes S3 to ignore all existing public ACLs on the bucket and its objects.
3. `BlockPublicPolicy`: Rejects any bucket policy update that grants public read or write access.
4. `RestrictPublicBuckets`: Restricts access to buckets with public policies to AWS services and authorized users within the bucket owner's account.

---

### Task 4: Identity Policy vs Resource Policy Authorisation

#### Objective
Demonstrate the fundamental AWS authorization evaluation logic when an identity-based IAM policy collides with a resource-based S3 bucket policy, verifying that an **explicit Deny always takes precedence over an explicit Allow**.

#### Execution Steps & Commands
```bash
# Create an IAM user representing an internal data analyst
aws $EP iam create-user --user-name DataAnalyst

# Attach an over-permissive identity policy allowing reading everything
cat > analyst-iam.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": "*"
  }]
}
JSON

aws $EP iam put-user-policy --user-name DataAnalyst   --policy-name S3ReadAll --policy-document file://analyst-iam.json

# Generate credentials and configure named profile
aws $EP iam create-access-key --user-name DataAnalyst   --query 'AccessKey.[AccessKeyId,SecretAccessKey]' --output text

aws configure --profile analyst set aws_access_key_id "AKIAIOSFODNN7EXAMPLE"
aws configure --profile analyst set aws_secret_access_key "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
aws configure --profile analyst set region us-east-1

# Define conflicting resource policy: Allow /internal/* but explicitly Deny /confidential/*
cat > deny-confidential.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAnalystInternal",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::000000000000:user/DataAnalyst"},
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::$BUCKET/internal/*"
    },
    {
      "Sid": "DenyAnalystConfidential",
      "Effect": "Deny",
      "Principal": {"AWS": "arn:aws:iam::000000000000:user/DataAnalyst"},
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::$BUCKET/confidential/*"
    }
  ]
}
JSON

aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://deny-confidential.json

# Test 1: Access internal roster (IAM: Allow, Resource: Allow) -> Expected: ALLOWED
AWS_PROFILE=analyst aws $EP s3api get-object   --bucket $BUCKET --key internal/roster.txt analyst-internal.txt && echo "internal: ALLOWED"

# Test 2: Access confidential record (IAM: Allow, Resource: Deny) -> Expected: DENIED
AWS_PROFILE=analyst aws $EP s3api get-object   --bucket $BUCKET --key confidential/record.txt analyst-conf.txt || echo "confidential: DENIED"
```

#### Verification & Screenshot Evidence
![Task 4 Evidence](Lab%206%20Task%204.png)

```text
{
    "AcceptRanges": "bytes",
    "LastModified": "2026-09-10T11:46:08+00:00",
    "ContentLength": 29,
    "ETag": ""0d9fd031956d640c0b4f1d95ddd64b1a"",
    "ChecksumCRC64NVME": "NhcOKhoqy1k=",
    "ChecksumType": "FULL_OBJECT",
    "ContentType": "binary/octet-stream",
    "ServerSideEncryption": "AES256",
    "Metadata": {},
    "TagCount": 1
}
internal: ALLOWED
...
confidential: DENIED
```

#### Policy Conflict Evaluation Model
```
[ Incoming Request from DataAnalyst ]
                 |
                 v
   +---------------------------+
   | Is there an EXPLICIT DENY | ---- YES ----> [ ACCESS DENIED ]
   |  in IAM or Bucket Policy? |
   +---------------------------+
                 |
                NO
                 v
   +----------------------------+
   | Is there an EXPLICIT ALLOW | ---- YES ----> [ ACCESS GRANTED ]
   |  in IAM or Bucket Policy?  |
   +----------------------------+
                 |
                NO
                 v
        [ DEFAULT DENY ]
```

*Cleanup note:* As instructed, the resource policy was cleared (`aws $EP s3api delete-bucket-policy --bucket $BUCKET`) before proceeding to Session B.

---

## Session B: Protecting, Retaining, and Retiring Data

### Task 5: Default Encryption at Rest (SSE-KMS)

#### Objective
Enforce bucket-level server-side encryption using a Customer Managed Key (CMK) in AWS KMS with S3 Bucket Keys enabled. This guarantees that all objects uploaded without explicit encryption parameters are automatically encrypted at rest.

#### Execution Steps & Commands
```bash
# Create a dedicated Customer Managed Key in AWS KMS
export KEY_ID=$(aws $EP kms create-key   --description 'IKB42603 Lab6 patient records bucket key'   --query 'KeyMetadata.KeyId' --output text)
echo $KEY_ID

# Define default bucket encryption configuration with S3 Bucket Key optimization
cat > encryption.json <<JSON
{
  "Rules": [{
    "ApplyServerSideEncryptionByDefault": {
      "SSEAlgorithm": "aws:kms",
      "KMSMasterKeyID": "$KEY_ID"
    },
    "BucketKeyEnabled": true
  }]
}
JSON

# Apply default encryption configuration to bucket
aws $EP s3api put-bucket-encryption --bucket $BUCKET   --server-side-encryption-configuration file://encryption.json

aws $EP s3api get-bucket-encryption --bucket $BUCKET

# Upload object with NO explicit encryption parameters
aws $EP s3api put-object --bucket $BUCKET   --key confidential/record-v2.txt --body confidential-record.txt

# Inspect object metadata to verify automatic KMS encryption
aws $EP s3api head-object --bucket $BUCKET --key confidential/record-v2.txt   --query '[ServerSideEncryption,SSEKMSKeyId,BucketKeyEnabled]' --output text
```

#### Verification & Screenshot Evidence
![Task 5 Evidence](Lab%206%20Task%205.png)

```text
aws:kms  arn:aws:kms:us-east-1:000000000000:key/0d6870fa-28f3-4beb-a745-60d565e2ce2b  True
```

#### Security & Architecture Insight
- **Zero-Friction Encryption:** Developers do not need to specify `--server-side-encryption aws:kms` in their code or CLI calls; the bucket automatically wraps every incoming object payload with the designated KMS key.
- **S3 Bucket Key (`BucketKeyEnabled: true`):** Employs envelope encryption optimization at scale. Rather than issuing a separate KMS cryptographic call for every individual S3 object interaction, S3 negotiates a short-lived bucket-level key from KMS and derives unique object data keys locally. This dramatically decreases KMS API request volume, cost, and access latency while preserving strong data protection.

---

### Task 6: Delegated Access and the Condition-Key Trap

#### Objective
Examine the mechanics of time-bounded delegated access via presigned URLs and explore the catastrophic effects of improperly applying the `aws:SecureTransport` condition key in non-HTTPS environments.

#### Part 1: Presigned URL Generation & Verification
```bash
# Generate a time-bounded presigned URL valid for 60 seconds
aws $EP s3 presign s3://$BUCKET/internal/roster.txt --expires-in 60
```
- **Mechanism:** A presigned URL embeds security credentials (`X-Amz-Algorithm`, `X-Amz-Credential`, `X-Amz-Date`, `X-Amz-Expires`, `X-Amz-SignedHeaders`, and `X-Amz-Signature`). Anyone holding this URI possesses temporary authorization to perform the specified HTTP verb (`GET`) against that exact resource until the expiration threshold lapses.

#### Part 2: The `aws:SecureTransport` Trap
A common compliance recommendation dictates enforcing TLS for all S3 traffic using a bucket policy condition:
```bash
cat > secure-transport.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyUnencryptedTransport",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": ["arn:aws:s3:::$BUCKET", "arn:aws:s3:::$BUCKET/*"],
    "Condition": {"Bool": {"aws:SecureTransport": "false"}}
  }]
}
JSON

aws $EP s3api put-bucket-policy --bucket $BUCKET --policy file://secure-transport.json

# Test standard CLI call - LocalStack runs on plain HTTP
aws $EP s3api list-objects-v2 --bucket $BUCKET

# Revert the policy to restore access
aws $EP s3api delete-bucket-policy --bucket $BUCKET
```

#### Verification & Screenshot Evidence
![Task 6 Evidence](Lab%206%20Task%206.png)

```text
# Condition: { "Bool": { "aws:SecureTransport": "false" } }
# Tested against http://localhost:4566 (Plain HTTP transport)
# Result: Immediate global lockout across all bucket operations
# Policy deleted to restore bucket availability.
```

#### Architectural Takeaway
In production on AWS, requests arrive over `https://`, making `aws:SecureTransport` evaluate to `true` (thus evading the Deny statement). However, in development or local containerized test harnesses operating over plain `http://`, `aws:SecureTransport` evaluates to `false`. Because an explicit Deny takes absolute precedence, the policy immediately locks out all administrators and applications. **Policies containing context-sensitive condition keys must always be tested against the specific runtime network transport environment.**

---

### Task 7: Versioning, Delete Markers & Data Remanence

#### Objective
Demonstrate object-level data remanence in version-controlled storage. Prove that a standard object deletion in a versioned bucket merely places a soft delete marker over the object without actually purging the underlying historical data.

#### Execution Steps & Commands
```bash
# Enable bucket versioning
aws $EP s3api put-bucket-versioning --bucket $BUCKET   --versioning-configuration Status=Enabled

aws $EP s3api get-bucket-versioning --bucket $BUCKET

# Create revisions of the patient record (including a redacted version)
echo 'Patient: Ahmad bin Ali, Diagnosis: hypertension' > rec-v2.txt
echo 'Patient: [REDACTED], Diagnosis: [REDACTED]'      > rec-v3.txt

aws $EP s3api put-object --bucket $BUCKET --key confidential/record.txt   --body rec-v2.txt --query VersionId --output text

aws $EP s3api put-object --bucket $BUCKET --key confidential/record.txt   --body rec-v3.txt --query VersionId --output text

# List all versions (pre-existing unversioned copy has version-id: null)
aws $EP s3api list-object-versions --bucket $BUCKET   --prefix confidential/record.txt   --query 'Versions[].[VersionId,IsLatest,Size]' --output table

# Perform standard delete operation
aws $EP s3api delete-object --bucket $BUCKET --key confidential/record.txt

# Verify that standard GET reports object deleted / gone
aws $EP s3api get-object --bucket $BUCKET --key confidential/record.txt gone.txt

# Recover the original unredacted confidential record by requesting the explicit version ID
aws $EP s3api get-object --bucket $BUCKET --key confidential/record.txt   --version-id null recovered.txt

cat recovered.txt
```

#### Verification & Screenshot Evidence
![Task 7 Evidence](Lab%206Task%207.png)

```text
-(kali@Arif)-[~]
-$ aws $EP s3api get-object --bucket $BUCKET --key confidential/record.txt   --version-id null recovered.txt

cat recovered.txt
{
    "AcceptRanges": "bytes",
    "LastModified": "2026-09-10T11:46:10+00:00",
    "ContentLength": 48,
    "ETag": ""9a86d9c8a68fe26ab3f63cd85c116a7f"",
    "ChecksumCRC64NVME": "0JjivlclRBE=",
    "ChecksumType": "FULL_OBJECT",
    "VersionId": "null",
    "ContentType": "binary/octet-stream",
    "ServerSideEncryption": "AES256",
    "Metadata": {},
    "TagCount": 1
}
Patient: Ahmad bin Ali, Diagnosis: confidential
```

#### Compliance Analysis (Data Remanence & GDPR/PDPA)
When an application issues a standard `s3:DeleteObject` call against a versioned bucket, Amazon S3 does **not** erase the bits from storage. Instead, it creates a zero-byte **Delete Marker** with a new `VersionId` and sets `IsLatest: true`. Subsequent unversioned `GET` requests receive an HTTP 404 (or 405) Object Not Found. 

However, any caller with `s3:GetObjectVersion` permissions can bypass the delete marker simply by specifying `--version-id <ID>`. Consequently, claiming that personal health data was deleted when only a delete marker was created is a serious regulatory non-compliance violation under GDPR Article 17 (Right to Erasure) and Malaysia's Personal Data Protection Act (PDPA).

---

### Task 8: Lifecycle Automation & Cryptographic Erasure

#### Objective
Implement automated data retention and non-current version expiration via S3 Lifecycle configurations, and demonstrate **Cryptographic Erasure (Crypto-Shredding)** to achieve provable, instantaneous data destruction across all object replicas and versions.

#### Part 1: Automated Lifecycle Policy
```bash
# Define automated lifecycle configuration
cat > lifecycle.json <<'JSON'
{
  "Rules": [
    {
      "ID": "RetireConfidentialRecords",
      "Filter": {"Prefix": "confidential/"},
      "Status": "Enabled",
      "Expiration": {"Days": 365},
      "NoncurrentVersionExpiration": {"NoncurrentDays": 30}
    },
    {
      "ID": "AbortIncompleteUploads",
      "Filter": {"Prefix": ""},
      "Status": "Enabled",
      "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
    }
  ]
}
JSON

aws $EP s3api put-bucket-lifecycle-configuration --bucket $BUCKET   --lifecycle-configuration file://lifecycle.json

aws $EP s3api get-bucket-lifecycle-configuration --bucket $BUCKET   --query 'Rules[].[ID,Status]' --output table
```

##### Lifecycle Evidence Output
![Task 8 Lifecycle Evidence](Lab%206%20Task%208%201.png)

```text
-------------------------------------------------
|         GetBucketLifecycleConfiguration       |
+----------------------------+------------------+
|  RetireConfidentialRecords |  Enabled         |
|  AbortIncompleteUploads    |  Enabled         |
+----------------------------+------------------+
```

#### Part 2: Cryptographic Erasure (Crypto-Shredding)
```bash
# 1. Disable the KMS Master Key wrapping the encrypted objects
aws $EP kms disable-key --key-id $KEY_ID

# 2. Schedule KMS key deletion with minimum pending window (7 days)
aws $EP kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7

# 3. Check updated key state and deletion schedule
aws $EP kms describe-key --key-id $KEY_ID   --query 'KeyMetadata.[KeyState,DeletionDate]' --output text
```

##### Cryptographic Erasure Evidence Output
![Task 8 Deletion Evidence](Lab%206%20Task%208%202.png)

```text
-(kali@Arif)-[~]
-$ # 1. Disable the key
aws $EP kms disable-key --key-id $KEY_ID

# 2. Schedule key deletion (7 days window)
aws $EP kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7

# 3. Check the updated key state and deletion timestamp
aws $EP kms describe-key --key-id $KEY_ID   --query 'KeyMetadata.[KeyState, DeletionDate]' --output text
{
    "KeyId": "0d6870fa-28f3-4beb-a745-60d565e2ce2b",
    "DeletionDate": "2026-09-17T08:58:43.056982-04:00",
    "KeyState": "PendingDeletion",
    "PendingWindowInDays": 7
}
PendingDeletion 2026-09-17T08:58:43.056982-04:00
```

#### Cryptographic Erasure Theory & Assurance
In cloud environments, tenants have no access to physical storage drives to perform physical demagnetization (degaussing) or NIST SP 800-88 physical overwrites. When objects are encrypted using envelope encryption (SSE-KMS), destroying or disabling the root KMS Key irrevocably strips all parties—including cloud infrastructure administrators—of the mathematical capability to decrypt the ciphertext. This transforms every replica, snapshot, and historical version stored on the physical disks into unrecoverable pseudo-random noise, providing mathematically provable data sanitization.

---

## Answers to Lab Assessment Questions

### Question 1: Root Cause of Task 2 Exposure & Bucket vs IAM Policies
- **Specific Element:** The single JSON attribute `"Principal": "*"` caused the exposure.
- **Risk Comparison:** In an identity-based IAM policy, an overly broad statement (`"Resource": "*"`) only extends privileges to the *specific principal* to which the policy is attached (e.g., one user or service role). The credential boundary remains intact. Conversely, in a resource-based bucket policy, `"Principal": "*"` opens the gate to every entity in the world—including unauthenticated, anonymous HTTP clients without AWS credentials.

### Question 2: Identity-Based vs Resource-Based Policies in Task 4
- **Difference:** Identity-based policies are attached to principals (users, groups, roles) and define what that principal can execute across AWS resources. Resource-based policies are attached directly to resources (e.g., S3 buckets, KMS keys) and specify who can access that resource and under what conditions.
- **Evaluation in Task 4:**
  1. **Request 1 (`internal/roster.txt`):** The IAM policy granted `Allow` on all S3 resources, and the bucket policy explicitly granted `Allow` to `arn:aws:iam::000000000000:user/DataAnalyst` on `internal/*`. With no conflicting Deny statements, access was **ALLOWED**.
  2. **Request 2 (`confidential/record.txt`):** Although the IAM policy permitted read access, statement `DenyAnalystConfidential` in the bucket policy specified an explicit `"Effect": "Deny"`. Because an explicit Deny unconditionally overrides all Allow statements, access was **DENIED**.

### Question 3: Guardrails vs Controls for Multi-Engineer Organizations
- **Difference:** A standard security control (or detective check) is a point-in-time configuration or scanner (e.g., an AWS Config rule or security scanner that alerts security teams when a bucket is set to public). A guardrail (e.g., S3 Block Public Access, SCPs) is an immutable preventative boundary that operates at the API control-plane level, intercepting and rejecting non-compliant API calls before they can take effect.
- **Organizational Value:** In enterprise environments where hundreds of developers and CI/CD pipelines provision infrastructure, detective controls introduce an unavoidable window of vulnerability between creation and remediation. Preventative guardrails structurally eliminate human error by disallowing unsafe configurations from ever being instantiated.

### Question 4: SSE-KMS Protection Boundary vs The Task 4 Analyst
- **Does SSE-KMS Protect Against the Analyst?** **No, SSE-KMS does not protect against the analyst unless access to the underlying KMS decryption key is also restricted.**
- **Threat Model of SSE:** Server-Side Encryption protects data *at rest* against unauthorized physical media exfiltration, discarded drive theft, and hypervisor-level bypass within the cloud provider's data center. 
- **Authorization Boundary:** At the API layer, decryption is transparent: when a principal issues `s3:GetObject`, S3 calls KMS on their behalf. If the caller possesses both `s3:GetObject` and `kms:Decrypt` permissions, the data is decrypted and returned in cleartext. Therefore, preventing the analyst from accessing confidential data requires strict IAM/bucket authorization policies, not SSE-KMS alone.

### Question 5: GDPR / PDPA Right to Erasure & Provable Deletion
- **Why `delete-object` is Insufficient:** Under S3 versioning, executing `s3:DeleteObject` merely inserts a soft Delete Marker. The sensitive record remains fully intact in historical versions and can be retrieved using `--version-id`. Retaining personal data after an erasure request violates GDPR Art. 17 and PDPA principles.
- **Two Provable Erasure Mechanisms:**
  1. **Explicit Multi-Version Purge:** Iteratively enumerating and calling `delete-object` specifying every concrete historical `--version-id` until `list-object-versions` returns an empty set.
  2. **Cryptographic Erasure (Crypto-Shredding):** Revoking and scheduling permanent deletion of the Customer Managed KMS Key that encrypted the object. Without the key, decryption is mathematically impossible, rendering all copies permanently unrecoverable.

### Question 6: Compliance Audit Evidence (Week 11 Auditor Perspective)
As an information security compliance auditor verifying adherence to ISO/IEC 27001, SOC 2, or CSA CCM v4, the three essential commands are:
1. `aws s3api get-public-access-block --bucket $BUCKET`  
   *Control Evidenced:* Verifies that central, account/bucket-level preventative guardrails are actively enforcing private object storage (CCM Control DCS-08 / Storage Security).
2. `aws s3api get-bucket-encryption --bucket $BUCKET`  
   *Control Evidenced:* Proves mandatory cryptographic protection at rest using a dedicated customer-managed key (CCM Control EKM-02 / Encryption at Rest).
3. `aws s3api get-bucket-lifecycle-configuration --bucket $BUCKET`  
   *Control Evidenced:* Evidences automated compliance with statutory data retention schedules and secure automated data disposal (CCM Control DSP-07 / Data Retention and Disposal).

---

## Bucket Security Posture Verification

The following command block executes the comprehensive post-configuration audit validating the final security posture of the bucket:

```bash
echo "=== IKB42603 Lab 6 verification: $BUCKET ==="
aws $EP s3api get-public-access-block --bucket $BUCKET   --query 'PublicAccessBlockConfiguration' --output text

aws $EP s3api get-bucket-versioning --bucket $BUCKET --output text

aws $EP s3api get-bucket-encryption --bucket $BUCKET   --query 'ServerSideEncryptionConfiguration.Rules[0].ApplyServerSideEncryptionByDefault.[SSEAlgorithm,KMSMasterKeyID]'   --output text

aws $EP s3api get-bucket-lifecycle-configuration --bucket $BUCKET   --query 'Rules[].[ID,Status]' --output table

aws $EP kms describe-key --key-id $KEY_ID --query 'KeyMetadata.KeyState' --output text
```

### Verified System State Output
```text
=== IKB42603 Lab 6 verification: miit-patient-records-18942 ===
True    True    True    True
Status  Enabled
aws:kms arn:aws:kms:us-east-1:000000000000:key/0d6870fa-28f3-4beb-a745-60d565e2ce2b
-------------------------------------------------
|         GetBucketLifecycleConfiguration       |
+----------------------------+------------------+
|  RetireConfidentialRecords |  Enabled         |
|  AbortIncompleteUploads    |  Enabled         |
+----------------------------+------------------+
PendingDeletion
```

---

## Security Best-Practices Checklist

- [x] **Data Classification:** Every object is tagged with appropriate sensitivity metadata (`classification=public|internal|confidential`) prior to authorization rule definition.
- [x] **No Wildcard Principals:** Zero resource policies contain `"Principal": "*"`; unauthenticated anonymous requests are blocked.
- [x] **Block Public Access Enabled:** All four Block Public Access flags (`BlockPublicAcls`, `IgnorePublicAcls`, `BlockPublicPolicy`, `RestrictPublicBuckets`) are active.
- [x] **Least-Privilege Resource Policies:** Access is tightly constrained by caller ARN and scoped to exact key prefixes rather than root bucket wildcards.
- [x] **Mandatory Default Encryption:** S3 bucket enforces default server-side encryption with AWS KMS (`aws:kms`) using a Customer Managed Key and S3 Bucket Keys.
- [x] **Time-Bounded Delegated Sharing:** Temporary resource sharing utilizes expiring presigned URLs rather than altering object ACLs or bucket policies.
- [x] **Controlled Versioning & Remanence Awareness:** S3 versioning is enabled, with full organizational comprehension that delete markers do not permanently purge data.
- [x] **Automated Retention & Destruction:** S3 Lifecycle rules automate non-current version retirement, and cryptographic erasure procedures are established for rapid crypto-shredding.

---

## Teardown & Environment Sanitization

To ensure complete hygiene and prevent lingering resource charges or orphaned storage remnants, the versioned bucket and IAM/KMS artifacts were systematically cleaned up:

```bash
# 1. Remove bucket resource policies
aws $EP s3api delete-bucket-policy --bucket $BUCKET

# 2. Delete all object versions
aws $EP s3api delete-objects --bucket $BUCKET --delete "$(aws $EP s3api   list-object-versions --bucket $BUCKET --output json   --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}')"

# 3. Delete all delete markers
aws $EP s3api delete-objects --bucket $BUCKET --delete "$(aws $EP s3api   list-object-versions --bucket $BUCKET --output json   --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}')"

# 4. Verify bucket is empty and delete it
aws $EP s3api list-object-versions --bucket $BUCKET --output text
aws $EP s3api delete-bucket --bucket $BUCKET

# 5. Clean up IAM user and local files
aws $EP iam delete-user-policy --user-name DataAnalyst --policy-name S3ReadAll
aws $EP iam delete-access-key --user-name DataAnalyst --access-key-id AKIAIOSFODNN7EXAMPLE 2>/dev/null || true
aws $EP iam delete-user --user-name DataAnalyst

# 6. Stop LocalStack and clean up local artifacts
docker rm -f localstack
rm -f *.json *.txt
```

---
*End of Report — Lab 6: Object Storage Security & the Data Security Lifecycle*
