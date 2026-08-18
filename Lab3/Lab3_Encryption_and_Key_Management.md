# Lab 3: Data Protection — Encryption & Key Management
**Course:** IKB42603 Cloud Computing Security Essentials  
**Institution:** Universiti Kuala Lumpur — Malaysian Institute of Information Technology (UniKL MIIT)  
**Lecturer:** Prof. Dr. Shahrulniza Musa  
**Student:** Afiq Danial (`afiqdanial@afiq`)  
**Scope:** Weeks 5–6 (Session A: Encryption Fundamentals | Session B: Key Management, Envelope Encryption & Erasure)  

---

## 1. Executive Summary & Learning Outcomes

This report documents the step-by-step execution, output analysis, security principles, and deliverable answers for **Lab 3: Data Protection — Encryption & Key Management**.

### Lab Learning Outcomes
By completing this lab, the following core cryptographic and cloud security capabilities are demonstrated:
1. **Symmetric & Asymmetric Cryptography:** Encrypting and decrypting data at rest using AES-256-CBC and RSA-2048 key pairs via OpenSSL.
2. **Protection of Data in Transit:** Securing communication channels using Transport Layer Security (TLS/HTTPS) with self-signed X.509 certificates served via NGINX containers.
3. **Key Management Service (KMS) & Envelope Encryption:** Generating Customer Master Keys (CMKs) and Data Encryption Keys (DEKs) using AWS KMS / LocalStack to separate master key management from bulk data payload encryption.
4. **Per-Tenant Isolation & Cryptographic Erasure:** Provisioning distinct master keys per tenant and performing provable data destruction by disabling/scheduling deletion of KMS master keys.
5. **Data Integrity & Tamper-Evidence:** Generating cryptographic hash fingerprints (SHA-256) and constructing sequential, tamper-evident hash chains for immutable audit logging.

### Course & Assessment Mapping
| Item | Mapping Details |
| :--- | :--- |
| **Course Learning Outcome** | **CLO2** — Construct secure cloud operations that safeguard data integrity (VBE3) |
| **Lecture Topics** | **Week 4** (Data Protection) & **Week 9** (Key Management patterns) |
| **Value / Skill Clusters** | **VBE3** (Integrity) · **SC8** (Integrated Problem-Solving) |
| **Assessment** | Lab report (outputs + short answers) — contributes to Lab Assignment |

---

## 2. Session A (Week 5) — Encryption Fundamentals

---

### Task 1 — Symmetric Encryption (Data at Rest)

#### Objective
Create a sensitive data record and encrypt it using symmetric cipher **AES-256-CBC** with key derivation function PBKDF2. Verify that the encrypted file is unreadable and that decryption restores the original content exactly.

#### Step-by-Step Execution Commands
```bash
# 1. Create a sample sensitive record
echo 'Patient: Ahmad, Diagnosis: confidential' > record.txt

# 2. Encrypt with AES-256-CBC (prompted for passphrase/key)
openssl enc -aes-256-cbc -pbkdf2 -salt -in record.txt -out record.enc

# 3. Prove it is unreadable ciphertext
cat record.enc

# 4. Decrypt ciphertext back to plaintext
openssl enc -d -aes-256-cbc -pbkdf2 -in record.enc -out record.dec.txt

# 5. Verify integrity and exact match
diff record.txt record.dec.txt && echo 'MATCH: decryption successful'
```

#### Terminal Execution & Evidence
![Task 1 — Symmetric Encryption](Task%201%20—%20Symmetric%20Encryption.png)

```text
(afiqdanial@afiq)-[~/Lab3]
$ diff record.txt record.dec.txt && echo 'MATCH: decryption successful'
MATCH: decryption successful
```

#### Security Analysis & Key Takeaways
- **Symmetric Encryption:** AES-256 uses the same secret key for both encryption and decryption. It offers high computational speed and strong security for bulk data encryption.
- **Key Distribution Problem:** Because sender and receiver must both possess the secret key prior to communication, distributing symmetric keys securely across distributed cloud environments without interception is a fundamental challenge.

---

### Task 2 — Asymmetric Encryption & Digital Signatures

#### Objective
Generate a 2048-bit RSA key pair. Demonstrate asymmetric encryption (encrypting with public key, decrypting with private key) and digital signatures (signing digest with private key, verifying signature with public key).

#### Step-by-Step Execution Commands
```bash
# 1. Generate a 2048-bit RSA private key
openssl genrsa -out private.pem 2048

# 2. Extract corresponding public key
openssl rsa -in private.pem -pubout -out public.pem

# 3. Encrypt with PUBLIC key, decrypt with PRIVATE key
openssl pkeyutl -encrypt -pubin -inkey public.pem -in record.txt -out record.rsa
openssl pkeyutl -decrypt -inkey private.pem -in record.rsa -out record.rsa.txt

# 4. Sign data digest with PRIVATE key; verify signature with PUBLIC key
openssl dgst -sha256 -sign private.pem -out record.sig record.txt
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```

#### Terminal Execution & Evidence
![Task 2 — Asymmetric Encryption & Digital Signatures](Task%202%20—%20Asymmetric%20Encryption%20%26%20Digital%20Signatures.png)

```text
(afiqdanial@afiq)-[~/Lab3]
$ openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
Verified OK
```

#### Security Analysis & Key Takeaways
- **Role Reversal:** Encryption uses the **Public Key** (anyone can encrypt data intended for the recipient), whereas Signing uses the **Private Key** (only the key owner can create a valid signature).
- **Authentication & Non-Repudiation:** Digital signatures provide non-repudiation and proof of origin because only the holder of the private key could have produced the signature, while anyone holding the public key can verify it.

---

### Task 3 — Encryption in Transit (TLS)

#### Objective
Generate a self-signed X.509 certificate and serve the sensitive file over HTTPS using an NGINX container. Verify that traffic in transit is encrypted using TLS.

#### Step-by-Step Execution Commands
```bash
# 1. Generate self-signed certificate and private key
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 7 -nodes -subj '/CN=localhost'

# 2. Serve HTTPS over port 8443 using NGINX container
docker run --rm -d --name tls -p 8443:443 \
  -v $(pwd)/cert.pem:/etc/nginx/cert.pem \
  -v $(pwd)/key.pem:/etc/nginx/key.pem \
  -v $(pwd)/record.txt:/usr/share/nginx/html/record.txt nginx

# 3. Connect over TLS (-k ignores self-signed certificate trust warning)
curl -k https://localhost:8443/record.txt
```

#### Terminal Execution & Evidence
![Task 3 — Encryption in Transit](Task%203%20—%20Encryption%20in%20Transit.png)

```text
(afiqdanial@afiq)-[~/Lab3]
$ curl -k https://localhost:8443/record.txt
Patient: Ahmad, Diagnosis: confidential
```

#### Security Analysis & Key Takeaways
- **Protection Against Eavesdropping:** Plain HTTP transmits payload data as unencrypted cleartext across network routers, making it vulnerable to packet sniffing (man-in-the-middle).
- **TLS Channel Security:** TLS encrypts session traffic dynamically using hybrid cryptography (asymmetric key exchange followed by symmetric session encryption), rendering wiretapped data unreadable.

---

## 3. Session B (Week 6) — Key Management, Envelope Encryption & Erasure

---

### Task 4 — Create and Use a KMS Master Key

#### Objective
Initialize AWS KMS (via LocalStack) to manage Customer Master Keys (CMKs) and encrypt small secrets directly using centralized KMS API endpoints.

#### Step-by-Step Execution Commands
```bash
# 1. Define LocalStack KMS Endpoint variable
EP='--endpoint-url=http://localhost:4566'

# 2. Create Customer Master Key (CMK) for Tenant A and capture KeyId
aws $EP kms create-key --description 'CCSE tenant-A master key'

# 3. Export KeyId into environment variable
KEY_A="c8beb313-b70d-4e4f-bf22-62647adcafb7"

# 4. Encrypt a small secret payload directly with KMS Master Key
aws $EP kms encrypt --key-id $KEY_A --plaintext "$(echo -n 'hello' | base64)" \
  --query CiphertextBlob --output text
```

#### Terminal Execution & Evidence
![Task 4 — Create and Use a KMS Master Key](Task%204%20—%20Create%20and%20Use%20a%20KMS%20Master%20Key.png)

```text
(afiqdanial@afiq)-[~/Lab3]
$ aws $EP kms encrypt --key-id $KEY_A --plaintext "$(echo -n 'hello' | base64)" \
  --query CiphertextBlob --output text
YzhiZWIzMTMtYjcwZC00ZTRmLWJmMjItNjI2NDdhZGNhZmI38eLrXKAvGKxImGBu5sB2APK9rAFa8BkCqpjtZD7UKzCA+CTYNRKoHsMe0XGUMU4T
```

---

### Task 5 — Envelope Encryption

#### Objective
Implement **Envelope Encryption** to secure bulk datasets. Generate a plaintext Data Encryption Key (DEK) and encrypted DEK wrapper from KMS. Encrypt payload data locally with the DEK, then purge the plaintext DEK from memory and disk while storing only the KMS-wrapped DEK.

#### Step-by-Step Execution Commands
```bash
# 5.1 Request a Data Encryption Key (DEK) from KMS (returns Plaintext + CiphertextBlob)
aws $EP kms generate-data-key --key-id $KEY_A --key-spec AES_256 \
  --query '[Plaintext,CiphertextBlob]' --output text

# Save Column 1 as datakey.b64 (plaintext) and Column 2 as datakey.enc (wrapped DEK)

# 5.2 Encrypt file locally with plaintext DEK
base64 -d datakey.b64 > datakey.bin
openssl enc -aes-256-cbc -pbkdf2 -in record.txt -out record.env.enc -pass file:./datakey.bin

# 5.3 Securely delete/destroy plaintext DEK from local system
rm datakey.bin datakey.b64
echo 'Only the KMS-wrapped data key (datakey.enc) remains.'
```

#### Terminal Execution & Evidence
![Task 5 — Envelope Encryption](Task%205%20—%20Envelope%20Encryption.png)

```text
(afiqdanial@afiq)-[~/Lab3]
$ rm datakey.bin datakey.b64

(afiqdanial@afiq)-[~/Lab3]
$ echo 'Only the KMS-wrapped data key (datakey.enc) remains.'
Only the KMS-wrapped data key (datakey.enc) remains.
```

---

### Task 6 — Per-Tenant Keys & Cryptographic Erasure

#### Objective
Demonstrate per-tenant key separation and perform **Cryptographic Erasure** by disabling/deleting Tenant A's KMS Master Key. Verify that without the master key, unwrapping the DEK fails, rendering the underlying ciphertext permanently unrecoverable.

#### Step-by-Step Execution Commands
```bash
# 1. Create separate KMS Master Key for Tenant B
aws $EP kms create-key --description 'CCSE tenant-B master key'
KEY_B="e62c2f7b-1b70-4d25-b39d-8173ebf6aa82"

# 2. Schedule deletion and disable Tenant A's master key
aws $EP kms schedule-key-deletion --key-id $KEY_A --pending-window-in-days 7
aws $EP kms disable-key --key-id $KEY_A

# 3. Attempt to unwrap Tenant A's wrapped data key (datakey.enc) via KMS Decrypt
aws $EP kms decrypt --ciphertext-blob fileb://datakey.enc 2>&1 | head -3
```

#### Terminal Execution & Evidence
![Task 6 — Per-Tenant Keys & Cryptographic Erasure](Task%206%20—%20Per-Tenant%20Keys%20%26%20Cryptographic%20Erasure.png)

```text
(afiqdanial@afiq)-[~/Lab3]
$ aws $EP kms disable-key --key-id $KEY_A
aws: [ERROR]: An error occurred (KMSInvalidStateException) when calling the DisableKey operation: arn:aws:kms:us-east-1:000000000000:key/c8beb313-b70d-4e4f-bf22-62647adcafb7 is pending deletion.

(afiqdanial@afiq)-[~/Lab3]
$ aws $EP kms decrypt --ciphertext-blob fileb://datakey.enc 2>&1 | head -3
aws: [ERROR]: An error occurred (NotFoundException) when calling the Decrypt operation: Invalid keyId 'YzhiZWIzMTMtYjcwZC00ZTRmLWJmMjItNjI2'
```

#### Security Analysis & Key Takeaways
- **Cryptographic Erasure (Crypto-Shredding):** In cloud environments where storage spans multi-tenant clusters, physical block overwriting (such as DoD 5220.22-M) is impractical or impossible to verify. Destroying the KMS Master Key renders ciphertext un-decryptable mathematical noise, achieving instant, provable deletion.

---

### Task 7 — Integrity & Tamper-Evidence

#### Objective
Use SHA-256 hashing to detect file tampering and implement a sequential **Hash Chain** where every log record includes the hash of the preceding entry, establishing a tamper-evident audit record.

#### Step-by-Step Execution Commands
```bash
# 1. Compute SHA-256 fingerprint of original file
sha256sum record.txt

# 2. Tamper with a copy of the file and compare hashes
cp record.txt tampered.txt
echo 'x' >> tampered.txt
sha256sum record.txt tampered.txt

# 3. Construct a sequential hash chain for audit log entries
PREV=0
for line in 'login ok' 'file read' 'export data'; do
  PREV=$(echo -n "$PREV$line" | sha256sum | cut -d' ' -f1);
  echo "$line | $PREV";
done
```

#### Terminal Execution & Evidence
![Task 7 — Integrity & Tamper-Evidence](Task%207%20—%20Integrity%20%26%20Tamper-Evidence.png)

```text
PREV=0
for line in 'login ok' 'file read' 'export data'; do
  PREV=$(echo -n "$PREV$line" | sha256sum | cut -d' ' -f1)
  echo "$line | $PREV"
done
login ok | 573f9af26d45d395a1089ef5fec4d50ccddc17c0ea4269c2c91d90929a820053
file read | 6c3adc61ece69412b338e43d761435e95dfbc948253f8f600087b0a4c5ad2d3d
export data | e1470ccfaf43dcab3c17d5710dc9eacbb7ac65c9f522ca98c2c503431b32da68
```

---

## 4. Final Verification Command Output

#### Objective
Run overall lab verification commands to confirm active KMS master keys and validate digital signatures.

#### Verification Commands
```bash
# 1. List active KMS master keys in LocalStack
aws --endpoint-url=http://localhost:4566 kms list-keys

# 2. Verify RSA Digital Signature
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```

#### Verification Terminal Evidence
![Verification Command](Verification%20Command.png)

```text
$ aws --endpoint-url=http://localhost:4566 kms list-keys
{
    "Keys": [
        {
            "KeyId": "c8beb313-b70d-4e4f-bf22-62647adcafb7",
            "KeyArn": "arn:aws:kms:us-east-1:000000000000:key/c8beb313-b70d-4e4f-bf22-62647adcafb7"
        },
        {
            "KeyId": "e62c2f7b-1b70-4d25-b39d-8173ebf6aa82",
            "KeyArn": "arn:aws:kms:us-east-1:000000000000:key/e62c2f7b-1b70-4d25-b39d-8173ebf6aa82"
        }
    ]
}

(afiqdanial@afiq)-[~/Lab3]
$ openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
Verified OK
```

---

## 5. Short-Answer Questions & Conceptual Answers

### Q1. Compare symmetric and asymmetric encryption: speed, key distribution, and typical use.

| Criteria | Symmetric Encryption (e.g., AES-256) | Asymmetric Encryption (e.g., RSA-2048, ECC) |
| :--- | :--- | :--- |
| **Speed & Performance** | Extremely fast; hardware-accelerated (AES-NI). Ideal for bulk data processing. | Relatively slow; involves complex modular exponentiation and large prime arithmetic. |
| **Key Distribution** | **Secret key sharing challenge:** Both parties must share the exact same key securely out-of-band. | **Simplified distribution:** Public keys can be freely published; private keys remain secret. |
| **Typical Use Cases** | Data at rest encryption (S3 buckets, database disk encryption), bulk payload encryption in envelope schemes. | TLS handshakes, digital signatures, SSH key authentication, initial key exchange. |
| **Hybrid Approach** | Cloud systems combine both: asymmetric encryption secures key exchange, symmetric encryption protects bulk data. |

---

### Q2. Why is key management described as the weakest link, not the algorithm?

Modern cryptographic algorithms like **AES-256** and **RSA-2048** are mathematically robust; breaking AES-256 via brute-force attack would require $2^{256}$ operations, exceeding available physical energy in the known universe.

However, real-world breaches almost never break the underlying math. Instead, systems fail due to **Key Management weaknesses**:
1. **Hardcoded Keys:** Storing raw keys in code repositories, config files, or public git commits.
2. **Weak Access Controls:** Over-privileged IAM policies allowing unauthorized users/roles to invoke `kms:Decrypt`.
3. **Insecure Storage:** Storing plaintext keys on unencrypted local disks or shared container volumes.
4. **Lack of Lifecycle Management:** Failure to rotate keys regularly or revoke compromised keys promptly.

---

### Q3. Explain envelope encryption and why only the master key needs hardware-grade protection.

#### Concept of Envelope Encryption
Envelope encryption uses two tiers of keys:
1. **Data Encryption Key (DEK):** A unique symmetric key generated to encrypt a specific dataset or file locally.
2. **Key Encryption Key (KEK / Master Key):** A centralized KMS Master Key used exclusively to encrypt (wrap) and decrypt (unwrap) the DEK.

```
       +--------------------+
       |  Plaintext Data    |
       +---------+----------+
                 |
                 v  [Encrypted with DEK]
       +---------+----------+
       |   Ciphertext Data  |
       +--------------------+

       +--------------------+
       |   Plaintext DEK    |
       +---------+----------+
                 |
                 v  [Wrapped with Master Key inside KMS HSM]
       +---------+----------+
       |    Wrapped DEK     | (Stored alongside Ciphertext)
       +--------------------+
```

#### Protection Strategy
- Transmitting gigabytes or terabytes of raw data to a centralized Hardware Security Module (HSM) for encryption creates massive bandwidth bottlenecks and latency.
- Envelope encryption offloads bulk encryption to local fast AES algorithms using local DEKs.
- Only the compact, high-value **Master Key** (CMK) resides inside the secure HSM boundary. Since DEKs are wrapped before storage, only the Master Key requires hardware-grade HSM protection.

---

### Q4. How does cryptographic erasure achieve provable deletion where overwriting cannot (in the cloud)?

#### Limitations of Overwriting in the Cloud
In cloud architectures, data is fragmented across distributed block storage (SAN/NAS), automated database backups, multi-zone snapshots, and object stores (S3). Overwriting disk blocks (e.g., zero-fill or random pattern overwrite) is non-viable because cloud users have no direct physical access to disk controllers, wear-leveling SSD controllers alter physical write paths, and shadow snapshots persist asynchronously.

#### How Cryptographic Erasure Works
1. All stored data is encrypted at rest using unique DEKs wrapped by a dedicated per-tenant KMS Master Key.
2. To delete data provably, the organization revokes, disables, or destroys the per-tenant KMS Master Key.
3. Without the master key, unwrapping the DEKs is cryptographically impossible.
4. The remaining ciphertext across all distributed storage media and snapshots instantly becomes un-decryptable pseudo-random noise, guaranteeing provable compliance (e.g., GDPR Right to be Forgotten).

---

### Q5. How does a hash chain make a log tamper-evident (link to tamper-proof logs, Week 6)?

A **Hash Chain** constructs an append-only audit record by computing each log entry's hash as a function of both the current record content and the cryptographic hash of the immediate predecessor:

$$\text{Hash}_n = \text{SHA-256}(\text{Hash}_{n-1} \parallel \text{LogEntry}_n)$$

```
+--------------+     +--------------+     +--------------+
| Log Entry 1  |     | Log Entry 2  |     | Log Entry 3  |
| Hash: H1     +---->| PrevHash: H1 +---->| PrevHash: H2 |
+--------------+     | Hash: H2     |     | Hash: H3     |
                     +--------------+     +--------------+
```

#### Tamper Detection Mechanism
- If an attacker modifies or deletes a historical record (e.g., modifying `Log Entry 1`), $H_1$ changes.
- Because $H_2$ depends directly on $H_1$, the computed hash for Entry 2 invalidates, cascading failure down the entire chain.
- This creates an immutable, tamper-evident log structure (foundational to blockchain ledgers and cloud audit trails like AWS CloudTrail log validation).

---

## 6. Security Best-Practices Checklist

| Checklist Item | Status | Verification Detail |
| :--- | :---: | :--- |
| **Data encrypted at rest (AES) and decryption verified** | [x] Completed | AES-256-CBC encryption confirmed via `diff` matching plaintext. |
| **Asymmetric keys used correctly** | [x] Completed | Public key used for encryption; Private key used for signing & decryption. Signature verified (`Verified OK`). |
| **Data protected in transit with TLS** | [x] Completed | NGINX container served HTTPS over port 8443; validated via `curl -k`. |
| **Envelope encryption used; plaintext DEK purged** | [x] Completed | DEK generated via KMS; local `datakey.bin` securely removed from disk. |
| **Per-Tenant keys & cryptographic erasure demonstrated** | [x] Completed | Master Key A disabled/scheduled for deletion; KMS decryption failed (`NotFoundException`). |
| **Integrity verified with hashing / hash chain** | [x] Completed | SHA-256 detected single-byte modification; 3-stage hash chain computed. |

---

## 7. Cleanup & Teardown Guide

To return the environment to a clean state after completing all lab tasks, run the following commands:

```bash
# 1. Stop and remove NGINX TLS container
docker stop tls 2>/dev/null || true

# 2. Clean up temporary cryptographic artifacts
rm -f record.* private.pem public.pem key.pem cert.pem datakey.* tampered.txt

# 3. Stop LocalStack container (if applicable)
docker stop localstack && docker rm localstack
```

---

## 8. Expansion Ideas (Advanced Implementations)

1. **Hardware Security Module (SoftHSM) Integration:** Integrate SoftHSM2 via PKCS#11 interface to offload RSA signing operations directly to a simulated PKCS#11 hardware device.
2. **HashiCorp Vault Transit Engine:** Deploy HashiCorp Vault container and utilize its Transit Secrets Engine to perform encryption-as-a-service and automated key rotation without exposing raw keys to application code.
3. **Mutual TLS (mTLS):** Configure dual-sided certificate verification on NGINX containers requiring client certificates, enforcing strict identity authentication for service-to-service mesh communication.
