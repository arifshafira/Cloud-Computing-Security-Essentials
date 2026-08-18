# Lab 3: Encryption and Key Management

**Course:** IKB42603 Cloud Computing  
**Lab:** Lab 3 — Data Protection: Encryption and Key Management  
**Student:** Muhammad Arif Shafira Bin Shahrin Amri  

## Objectives

This lab demonstrates protection of cloud data at rest and in transit, key management through LocalStack KMS, envelope encryption, cryptographic erasure, and integrity verification. The work follows the supplied *IKB42603_Lab3_Encryption_and_Key_Management* guide.

## Environment

- Kali Linux terminal with OpenSSL and AWS CLI.
- Docker/Nginx for the TLS demonstration.
- LocalStack KMS endpoint: `http://localhost:4566`.
- Test data: `record.txt`, containing a confidential patient record.

## Session A — Encryption Fundamentals

### Task 1 — Symmetric encryption (data at rest)

1. Created `record.txt` with a confidential patient record.
2. Encrypted it with AES-256-CBC, PBKDF2 key derivation, and a salt:

   ```bash
   openssl enc -aes-256-cbc -pbkdf2 -salt -in record.txt -out record.enc
   ```

3. Decrypted `record.enc` into `record.dec.txt` using the same passphrase.
4. Compared the original and decrypted files. The command returned `MATCH: decryption successful`, confirming correct recovery of the original plaintext.

   ![alt text](<Task 1 - 1.png>)

**Result:** AES-256 protected the file at rest; it could only be recovered with the shared secret/passphrase.

### Task 2 — RSA encryption and digital signature

1. Generated a 2048-bit RSA private key and its public key.
2. Used the public key to encrypt the record and the private key to decrypt it.
3. Signed `record.txt` with the private key using SHA-256.
4. Verified `record.sig` with the public key. The output was `Verified OK`.

   ![alt text](<Task 2.png>)

**Result:** The successful signature verification proves that the signed file has not changed since signing and that the corresponding private-key holder created the signature.

### Task 3 — Encryption in transit (TLS)

1. Generated a self-signed certificate for `localhost`.
2. Started an Nginx container that exposes HTTPS on port 8443 and serves `record.txt`.
3. Retrieved the file with `curl -k https://localhost:8443/record.txt`; `-k` was used only because the certificate is self-signed.

![Task 3 — HTTPS service and TLS retrieval](<Session A/Task 3/Task 3.png>)

**Result:** The patient record was received through HTTPS/TLS. Unlike HTTP, the application data is encrypted while travelling across the network.

## Session B — Key Management, Envelope Encryption and Erasure

### Task 4 — Create and use a KMS master key

1. Set `EP=--endpoint-url=http://localhost:4566` to target LocalStack.
2. Created KMS keys for the two tenants.
3. Used the tenant-A KMS key to encrypt the small plaintext value `hello`.

![Task 4 — KMS encrypt operation](<Session B/Task 4/Task 4.png>)

The later key listing confirms these KMS KeyIds:

| Tenant | KMS KeyId |
|---|---|
| A | `8dadbc04-63cf-4a9b-8aff-9e53e8e9a19b` |
| B | `517efe2e-9839-45ff-89fa-1c012405e7af` |

### Task 5 — Envelope encryption

1. Requested an AES-256 data key from KMS. KMS returned both its plaintext form and the KMS-wrapped form.
2. Saved the plaintext key as `datakey.b64`, decoded it to `datakey.bin`, and saved the wrapped key as `datakey.enc`.
3. Encrypted `record.txt` locally as `record.env.enc` with AES-256-CBC and `datakey.bin`.
4. Deleted `datakey.bin`, `datakey.b64`, and the temporary `keys.txt`; only `datakey.enc`, the wrapped data key, remained.

![Task 5.1 — Generate KMS data key](<Session B/Task 5/Task 5 - 1.png>)

![Task 5.2 — Separate and decode the data key](<Session B/Task 5/Task 5 - 2.png>)

![Task 5.3 — Encrypt the record locally](<Session B/Task 5/Task 5 - 3.png>)

![Task 5.4 — Remove plaintext data-key material](<Session B/Task 5/Task 5 - 4.png>)

**Result:** The large record is encrypted locally with a data key, while KMS protects only the much smaller wrapped data key/master-key relationship.

### Task 6 — Per-tenant keys and cryptographic erasure

1. Created a separate master key for tenant B.
2. Scheduled deletion of tenant A's key with the seven-day minimum deletion window.
3. Disabled tenant A's key immediately to simulate erasure.
4. Tried to decrypt tenant A's wrapped data key. KMS returned `KMSInvalidStateException` and reported that the key was pending deletion.

![Task 6 — Decrypt fails after tenant-A key erasure](<Session B/Task 6/Task 6 - 1.png>)

**Result:** Without tenant A's KMS key, `datakey.enc` cannot be unwrapped, so `record.env.enc` is unrecoverable. Tenant B's separate key remains isolated.

### Task 7 — Integrity and tamper evidence

1. Calculated the SHA-256 fingerprint of `record.txt`:

   `c834b974851cfbda897f8d524364992bf677af0c2950a3e1d29f862019c83516`

2. Appended `x` to a copy (`tampered.txt`) and recalculated both hashes. The tampered file hash changed to:

   `9b7c73cf2df7d1097b2b05a92d45cf3a74089c8cda57390ef41facd745d5a6a0`

3. Built a hash chain by hashing each log entry together with the preceding hash.

![Task 7.1 — Original file SHA-256](<Session B/Task 7/Task 7 - 1.png>)

![Task 7.2 — Tampering changes the SHA-256 value](<Session B/Task 7/Task 7 - 2.png>)

![Task 7.3 — Hash-chain log](<Session B/Task 7/Task 7 - 3.png>)

**Result:** A one-character change produces a completely different digest. The chained values make later alteration detectable.

## Verification and cleanup

The final verification listed both KMS keys and again showed `Verified OK` for the RSA signature.

![Verification commands](<Session B/Verify Command.png>)

The TLS container and LocalStack environment were stopped, and the generated key/data files were removed according to the guide.

![Cleanup and teardown](<Session B/Cleanup & Teardown.png>)

## Short-answer questions

### 1. Compare symmetric and asymmetric encryption: speed, key distribution, and typical use.

Symmetric encryption uses one shared secret for encryption and decryption. It is fast and therefore suitable for bulk data, such as files, database fields, and TLS session traffic. Its difficulty is secure key distribution: both parties must obtain and protect the same secret without exposing it.

Asymmetric encryption uses a public/private key pair. It is slower, but the public key may be distributed openly; only the private key must remain secret. It is commonly used for identity, digital signatures, certificate-based authentication, secure key exchange, and protecting small secrets such as symmetric data keys. In practice, systems commonly combine both methods: asymmetric cryptography establishes or wraps a symmetric key, then symmetric cryptography encrypts the data.

### 2. Why is key management described as the weakest link, not the algorithm?

Modern algorithms such as AES-256 and RSA are strong when used correctly. However, encryption offers no protection if an attacker can steal, copy, misuse, or retain the decryption key. Key management determines who can use a key, where it is stored, how it is rotated, audited, backed up, revoked, and destroyed. A KMS reduces this risk by centralising access controls and avoiding routine exposure of master keys to applications or disk.

### 3. Explain envelope encryption and why only the master key needs hardware-grade protection.

Envelope encryption generates a unique data-encryption key (DEK) for the actual data. The DEK encrypts the large file locally because symmetric encryption is efficient. KMS then encrypts (wraps) that DEK with a master key (KEK/CMK); only the wrapped DEK is stored with the ciphertext. To decrypt, KMS unwraps the DEK temporarily, and the application uses it before discarding it.

The master key is small, long-lived, and the root of access to many data keys, so it warrants hardware-grade protection, strict permissions, logging, and rotation. Data keys do not need to be retained in plaintext and can be generated per object or operation.

### 4. How does cryptographic erasure achieve provable deletion where overwriting cannot in the cloud?

Cryptographic erasure destroys or permanently disables the key needed to decrypt ciphertext. Once the tenant-A master key is unavailable, the wrapped DEK cannot be recovered and the encrypted record is computationally useless. This is demonstrable through the failed KMS decrypt operation.

Overwriting is difficult to prove in cloud storage because providers may maintain replicas, snapshots, backups, caches, or remapped physical blocks. Destroying the relevant key renders every encrypted copy inaccessible at once, provided no usable duplicate of that key exists.

### 5. How does a hash chain make a log tamper-evident?

Each record's hash is calculated from its contents plus the previous record's hash. Editing an older record changes its hash and breaks the link to the next record; the mismatch then propagates through every following entry. An attacker would need to recompute all later hashes, and a separately protected final hash, signature, timestamp, or external anchor exposes that rewrite. Thus a hash chain makes unauthorised log changes evident, although it does not by itself prevent a writer with full control from rewriting the entire chain.

## Security checklist

- [x] Data encrypted at rest with AES and decryption verified.
- [x] RSA public/private keys used for encryption and signatures.
- [x] Data retrieved over TLS.
- [x] Envelope encryption used and plaintext data-key files removed.
- [x] Per-tenant KMS keys used and cryptographic erasure demonstrated.
- [x] SHA-256 hashing and a tamper-evident hash chain demonstrated.
