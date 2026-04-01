## 1. What is Encryption at Rest?

**Encryption at Rest** protects data stored on disk (databases, file systems, backups) by encrypting it. If an attacker gains physical access to the storage or steals a backup, the data is unreadable without the encryption key.

```
Without encryption:
  Attacker steals hard drive → reads database files → sees plaintext data

With encryption:
  Attacker steals hard drive → reads encrypted files → sees gibberish
  Data is useless without the encryption key
```

**Contrast with Encryption in Transit:** Encryption in transit (TLS/HTTPS) protects data moving over the network. Encryption at rest protects data sitting on disk.

---

## 2. Why Encrypt at Rest?

### Compliance

Many regulations require encryption at rest:

```
GDPR:     Protect personal data with appropriate security measures
HIPAA:    Encrypt electronic protected health information (ePHI)
PCI DSS:  Encrypt cardholder data stored on disk
SOC 2:    Demonstrate data protection controls
```

---

### Protect against physical theft

```
Scenarios:
  - Stolen laptop or hard drive
  - Decommissioned disk not properly wiped
  - Backup tape lost in transit
  - Cloud provider employee with physical access
```

Encryption at rest ensures data is protected even if the physical storage is compromised.

---

### Defense in depth

Even if an attacker bypasses application-level security (SQL injection, compromised credentials), encrypted data on disk provides an additional layer of protection.

---

## 3. Encryption Algorithms

### AES (Advanced Encryption Standard)

The industry standard for symmetric encryption.

```
AES-128:  128-bit key, fast, secure for most use cases
AES-256:  256-bit key, slower, required by some compliance standards

Block size: 128 bits (16 bytes)
Modes:      CBC, GCM, CTR
```

**AES-256-GCM** is the most common choice for encryption at rest. GCM (Galois/Counter Mode) provides both encryption and authentication (detects tampering).

---

### RSA (Asymmetric Encryption)

Used for encrypting the data encryption keys (DEKs), not the data itself. Slower than AES.

```
RSA-2048:  2048-bit key, secure for key encryption
RSA-4096:  4096-bit key, more secure but slower
```

---

## 4. Key Management

The hardest part of encryption at rest is managing the keys. If you lose the key, the data is permanently unreadable. If an attacker gets the key, encryption is useless.

### Key Hierarchy

Most systems use a two-tier key hierarchy:

```
┌─────────────────────────────────────┐
│  Master Key (KEK)                   │  ← Stored in KMS, rarely changes
│  Key Encryption Key                 │
└──────────────┬──────────────────────┘
               │ Encrypts
               ▼
┌─────────────────────────────────────┐
│  Data Encryption Keys (DEKs)        │  ← Stored with data, rotated frequently
│  (one per file, table, or object)   │
└──────────────┬──────────────────────┘
               │ Encrypts
               ▼
┌─────────────────────────────────────┐
│  Data (files, database rows)        │
└─────────────────────────────────────┘
```

**Why two tiers?**

- Master key is stored securely in a Key Management Service (KMS) and rarely accessed
- Data encryption keys (DEKs) are generated per file/table and stored alongside the encrypted data
- To decrypt data: fetch master key from KMS → decrypt DEK → decrypt data
- Key rotation: rotate DEKs frequently without touching the master key

---

### Envelope Encryption

The pattern of encrypting data with a DEK, then encrypting the DEK with a master key.

```
1. Generate random DEK (256-bit AES key)
2. Encrypt data with DEK
3. Encrypt DEK with master key (from KMS)
4. Store encrypted data + encrypted DEK together
5. Discard plaintext DEK from memory

To decrypt:
1. Fetch encrypted DEK from storage
2. Decrypt DEK using master key (from KMS)
3. Decrypt data using DEK
4. Discard plaintext DEK from memory
```

**Benefits:**

- Master key never leaves KMS
- DEKs can be rotated without re-encrypting all data
- Each file/table has its own DEK (limits blast radius if a key is compromised)

---

## 5. Key Management Services (KMS)

A KMS is a secure service for storing and managing encryption keys.

### AWS KMS

```
- Managed service for creating and controlling encryption keys
- Keys never leave KMS in plaintext
- Integrates with S3, RDS, EBS, etc.
- Supports key rotation, access policies, audit logging
```

**Example: Encrypt data with AWS KMS**

```python
import boto3

kms = boto3.client('kms')

# Generate a data encryption key
response = kms.generate_data_key(
    KeyId='arn:aws:kms:us-east-1:123456789012:key/abc-123',
    KeySpec='AES_256'
)

plaintext_key = response['Plaintext']       # Use this to encrypt data
encrypted_key = response['CiphertextBlob']  # Store this with the data

# Encrypt data
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
cipher = Cipher(algorithms.AES(plaintext_key), modes.GCM(iv))
encryptor = cipher.encryptor()
ciphertext = encryptor.update(data) + encryptor.finalize()

# Store: ciphertext + encrypted_key + iv + tag
```

**To decrypt:**

```python
# Decrypt the data encryption key
response = kms.decrypt(CiphertextBlob=encrypted_key)
plaintext_key = response['Plaintext']

# Decrypt data
cipher = Cipher(algorithms.AES(plaintext_key), modes.GCM(iv, tag))
decryptor = cipher.decryptor()
plaintext = decryptor.update(ciphertext) + decryptor.finalize()
```

---

### Google Cloud KMS

Similar to AWS KMS. Supports key rotation, IAM policies, audit logging.

---

### HashiCorp Vault

Open-source, self-hosted KMS. Supports dynamic secrets, key rotation, audit logging.

---

### Azure Key Vault

Microsoft's managed KMS. Integrates with Azure services.

---

## 6. Database Encryption at Rest

### Transparent Data Encryption (TDE)

Encrypts the entire database at the storage layer. Application code doesn't change.

```
Supported by:
  - MySQL (InnoDB)
  - PostgreSQL
  - SQL Server
  - Oracle

How it works:
  - Database engine encrypts pages before writing to disk
  - Decrypts pages when reading from disk
  - Transparent to the application
```

**Pros:**

- No application changes
- Protects against physical theft of disk

**Cons:**

- Doesn't protect against SQL injection or compromised credentials (attacker queries the database normally)
- Key management is critical — if the key is on the same server, it's not much protection

---

### Column-Level Encryption

Encrypt specific columns (e.g., credit card numbers, SSNs) in the application before storing in the database.

```sql
-- Application encrypts before insert
INSERT INTO users (id, email, ssn_encrypted)
VALUES (1, 'user@example.com', 'AES_ENCRYPTED_BLOB');

-- Application decrypts after select
SELECT id, email, ssn_encrypted FROM users WHERE id = 1;
-- Application decrypts ssn_encrypted → "123-45-6789"
```

**Pros:**

- Fine-grained control (only sensitive columns encrypted)
- Protects against SQL injection (attacker sees encrypted data)
- Can use different keys per column or per user

**Cons:**

- Application must handle encryption/decryption
- Can't query encrypted columns (e.g., `WHERE ssn = '123-45-6789'` won't work)
- More complex

---

### Field-Level Encryption (FLE)

MongoDB's approach: encrypt specific fields in documents. Similar to column-level encryption.

---

## 7. File System Encryption

### Full Disk Encryption (FDE)

Encrypts the entire disk. Operating system decrypts on boot.

```
Examples:
  - BitLocker (Windows)
  - FileVault (macOS)
  - LUKS (Linux)
```

**Pros:**

- Protects against physical theft
- Transparent to applications

**Cons:**

- Doesn't protect against attacks while the system is running (key is in memory)
- If the OS is compromised, encryption is bypassed

---

### File-Level Encryption

Encrypt individual files or directories.

```
Examples:
  - eCryptfs (Linux)
  - EncFS (Linux, macOS)
  - EFS (Windows)
```

**Pros:**

- Fine-grained control
- Can use different keys per file

**Cons:**

- More complex
- Performance overhead

---

## 8. Object Storage Encryption

### S3 Server-Side Encryption (SSE)

AWS S3 encrypts objects before storing them.

```
SSE-S3:   S3 manages keys (AES-256)
          Simplest option, no key management required

SSE-KMS:  S3 uses AWS KMS for key management
          More control, audit logging, key rotation

SSE-C:    Customer provides encryption key with each request
          Full control, but customer must manage keys
```

**Example: Upload with SSE-KMS**

```python
s3.put_object(
    Bucket='my-bucket',
    Key='file.txt',
    Body=data,
    ServerSideEncryption='aws:kms',
    SSEKMSKeyId='arn:aws:kms:us-east-1:123456789012:key/abc-123'
)
```

---

### Client-Side Encryption

Encrypt data in the application before uploading to S3.

```python
# Encrypt data before upload
encrypted_data = encrypt(data, key)
s3.put_object(Bucket='my-bucket', Key='file.txt', Body=encrypted_data)

# Decrypt after download
encrypted_data = s3.get_object(Bucket='my-bucket', Key='file.txt')['Body'].read()
data = decrypt(encrypted_data, key)
```

**Pros:**

- Full control over encryption
- Data never exists in plaintext on S3

**Cons:**

- Application must handle encryption/decryption
- More complex

---

## 9. Key Rotation

Periodically changing encryption keys to limit the impact of a compromised key.

### Master Key Rotation

Rotate the master key (KEK) in KMS. DEKs are re-encrypted with the new master key.

```
1. Generate new master key in KMS
2. For each DEK:
   - Decrypt DEK with old master key
   - Encrypt DEK with new master key
   - Store re-encrypted DEK
3. Retire old master key
```

**AWS KMS automatic rotation:** Rotates master keys annually. Old keys are retained for decryption.

---

### Data Encryption Key Rotation

Rotate DEKs by re-encrypting data with new keys.

```
1. Generate new DEK
2. Decrypt data with old DEK
3. Encrypt data with new DEK
4. Store re-encrypted data + new encrypted DEK
5. Delete old DEK
```

**Challenge:** Re-encrypting large datasets is expensive. Often done incrementally (re-encrypt on write).

---

## 10. Performance Considerations

Encryption adds CPU overhead. Modern CPUs have AES-NI (AES hardware acceleration), making AES encryption very fast.

```
Typical overhead:
  AES-256-GCM with AES-NI:  ~1-5% CPU overhead
  Without AES-NI:           ~10-20% CPU overhead
```

**Best practices:**

- Use AES-NI-enabled CPUs
- Encrypt at the block/page level (not per-row) to amortize overhead
- Cache decrypted data in memory (but protect memory from dumps)

---

## 11. Who Holds the Keys?

### Cloud Provider Manages Keys (SSE-S3, RDS TDE)

```
Pros:
  ✅ Simple, no key management required
  ✅ Automatic key rotation

Cons:
  ❌ Cloud provider has access to keys (and thus data)
  ❌ Less control
```

**Use when:** Compliance requires encryption at rest, but you trust the cloud provider.

---

### Customer Manages Keys in KMS (SSE-KMS)

```
Pros:
  ✅ More control (access policies, audit logging)
  ✅ Cloud provider can't access data without your permission

Cons:
  ❌ More complex
  ❌ You must manage KMS access policies
```

**Use when:** Compliance requires customer-controlled keys.

---

### Customer Manages Keys Outside Cloud (SSE-C, BYOK)

```
Pros:
  ✅ Full control
  ✅ Cloud provider never sees keys

Cons:
  ❌ Complex key management
  ❌ You must provide keys with every request
  ❌ No automatic rotation
```

**Use when:** Regulatory requirements prohibit cloud provider access to keys.

---

## 12. Common Interview Questions + Answers

### Q: What's the difference between encryption at rest and encryption in transit?

> "Encryption in transit protects data moving over the network using TLS/HTTPS. Encryption at rest protects data stored on disk. Both are necessary — TLS prevents eavesdropping during transmission, and encryption at rest prevents data theft if physical storage is compromised. They use different keys and are independent layers of security."

---

### Q: How does envelope encryption work?

> "Envelope encryption uses a two-tier key hierarchy. You generate a random data encryption key (DEK) to encrypt the data. Then you encrypt the DEK with a master key stored in a KMS. You store the encrypted data and the encrypted DEK together. To decrypt, you fetch the encrypted DEK, decrypt it using the master key from KMS, then decrypt the data with the DEK. This way, the master key never leaves the KMS, and you can rotate DEKs without re-encrypting all data."

---

### Q: Why use a KMS instead of storing keys in the application?

> "A KMS provides secure key storage with hardware security modules (HSMs), access control policies, audit logging, and automatic key rotation. If you store keys in the application, they're vulnerable to code leaks, compromised servers, and insider threats. A KMS centralizes key management and ensures keys are never exposed in plaintext outside the KMS."

---

### Q: How do you rotate encryption keys?

> "For master keys, use automatic rotation in the KMS — the KMS generates a new key version and re-encrypts data encryption keys with the new master key. For data encryption keys, you can re-encrypt data with new keys, but this is expensive for large datasets. A common approach is incremental rotation: re-encrypt data with new keys on write, so over time all data is re-encrypted. Old keys are retained for decrypting old data until it's all migrated."

---

## 13. Interview Tricks & Pitfalls

### ✅ Trick 1: Mention envelope encryption

When discussing encryption at rest, always mention envelope encryption and the two-tier key hierarchy. It shows you understand real-world key management.

---

### ✅ Trick 2: Distinguish TDE vs column-level encryption

TDE is transparent and protects against physical theft. Column-level encryption protects against SQL injection and compromised credentials. Know when to use each.

---

### ✅ Trick 3: Talk about key rotation

Encryption at rest is incomplete without key rotation. Mention automatic rotation for master keys and incremental rotation for data keys.

---

### ❌ Pitfall 1: Storing keys with the data

Never store encryption keys on the same server as the encrypted data. If an attacker gets the disk, they get both. Use a separate KMS.

---

### ❌ Pitfall 2: Forgetting about key loss

If you lose the encryption key, the data is permanently unreadable. Always have a backup and recovery plan for keys.

---

### ❌ Pitfall 3: Assuming encryption solves all security problems

Encryption at rest doesn't protect against SQL injection, compromised credentials, or application-level attacks. It's one layer of defense in depth.

---

## 14. Quick Reference

```
Encryption at Rest: Protects data stored on disk

Why:
  - Compliance (GDPR, HIPAA, PCI DSS)
  - Protect against physical theft
  - Defense in depth

Algorithms:
  AES-256-GCM:  Industry standard, fast with AES-NI
  RSA-2048:     For encrypting keys, not data

Key Hierarchy:
  Master Key (KEK) → stored in KMS, rarely changes
  Data Encryption Keys (DEKs) → one per file/table, rotated frequently

Envelope Encryption:
  1. Encrypt data with DEK
  2. Encrypt DEK with master key
  3. Store encrypted data + encrypted DEK
  4. To decrypt: decrypt DEK with master key, then decrypt data

KMS (Key Management Service):
  AWS KMS, Google Cloud KMS, Azure Key Vault, HashiCorp Vault
  - Secure key storage (HSM)
  - Access policies, audit logging
  - Automatic key rotation

Database Encryption:
  TDE (Transparent Data Encryption):  Encrypts entire database, transparent to app
  Column-Level Encryption:            Encrypt specific columns in app code

File System Encryption:
  Full Disk Encryption (FDE):  BitLocker, FileVault, LUKS
  File-Level Encryption:       eCryptfs, EncFS

Object Storage Encryption:
  SSE-S3:   S3 manages keys (simplest)
  SSE-KMS:  S3 uses KMS (more control)
  SSE-C:    Customer provides keys (full control)
  Client-Side: Encrypt before upload (full control)

Key Rotation:
  Master key:  Automatic rotation in KMS (annual)
  DEKs:        Re-encrypt data with new keys (incremental)

Performance:
  AES-256-GCM with AES-NI:  ~1-5% overhead
  Use hardware acceleration, encrypt at block level

Who Holds Keys:
  Cloud provider:  Simple, less control
  Customer in KMS: More control, audit logging
  Customer outside cloud: Full control, complex
```
