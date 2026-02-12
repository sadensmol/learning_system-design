# End-to-End Encryption

Signal Protocol implementation for WhatsApp-scale messaging.

---

## Design Principles

1. **Zero server knowledge**: Server cannot read message content
2. **Forward secrecy**: Compromised key doesn't expose past messages
3. **Future secrecy**: Compromised key doesn't expose future messages
4. **Deniability**: Messages can't be cryptographically proven to originate from sender

---

## Signal Protocol Overview

### Key Types

```mermaid
graph TB
    subgraph "Long-term Keys"
        IK[Identity Key<br/>Ed25519 keypair<br/>Permanent per device]
    end

    subgraph "Medium-term Keys"
        SPK[Signed Pre-Key<br/>Curve25519<br/>Rotated monthly]
    end

    subgraph "Ephemeral Keys"
        OPK[One-Time Pre-Keys<br/>Curve25519<br/>Single use, 100 uploaded]
        EK[Ephemeral Key<br/>Generated per message]
    end

    subgraph "Session Keys"
        RK[Root Key]
        CK[Chain Key]
        MK[Message Key]
    end

    IK --> RK
    SPK --> RK
    OPK --> RK
    EK --> RK
    RK --> CK
    CK --> MK
```

---

## Key Exchange (X3DH)

Initial session establishment between Alice and Bob:

```mermaid
sequenceDiagram
    participant Alice
    participant Server
    participant Bob

    Note over Bob: Bob registers keys with server
    Bob->>Server: Upload(IK_B, SPK_B, [OPK_B_1...OPK_B_100])

    Note over Alice: Alice wants to message Bob
    Alice->>Server: Request Bob's keys
    Server-->>Alice: (IK_B, SPK_B, OPK_B_42)
    Server->>Server: Delete OPK_B_42

    Note over Alice: Alice computes shared secret
    Alice->>Alice: Generate ephemeral key EK_A
    Alice->>Alice: DH1 = DH(IK_A, SPK_B)
    Alice->>Alice: DH2 = DH(EK_A, IK_B)
    Alice->>Alice: DH3 = DH(EK_A, SPK_B)
    Alice->>Alice: DH4 = DH(EK_A, OPK_B)
    Alice->>Alice: SK = KDF(DH1 || DH2 || DH3 || DH4)

    Alice->>Server: InitialMessage(IK_A, EK_A, ciphertext)
    Server->>Bob: Deliver message

    Note over Bob: Bob computes same shared secret
    Bob->>Bob: DH1 = DH(SPK_B, IK_A)
    Bob->>Bob: DH2 = DH(IK_B, EK_A)
    Bob->>Bob: DH3 = DH(SPK_B, EK_A)
    Bob->>Bob: DH4 = DH(OPK_B, EK_A)
    Bob->>Bob: SK = KDF(DH1 || DH2 || DH3 || DH4)
    Bob->>Bob: Decrypt message
```

---

## Double Ratchet Algorithm

Provides forward and future secrecy through continuous key evolution.

### Ratchet Types

```mermaid
graph LR
    subgraph "DH Ratchet"
        DHR[New DH key pair<br/>per message exchange]
    end

    subgraph "Symmetric Ratchet"
        SR[Chain key advancement<br/>per message sent/received]
    end

    DHR -->|Resets| SR
    SR -->|Each message| MK[Message Key]
```

### How It Works

```mermaid
sequenceDiagram
    participant Alice
    participant Bob

    Note over Alice,Bob: Session established with Root Key RK

    Alice->>Alice: Generate new DH keypair (Ratchet)
    Alice->>Alice: RK, CK = KDF(RK, DH(Alice_DH, Bob_DH))
    Alice->>Alice: MK, CK = KDF(CK)
    Alice->>Bob: Encrypt(MK, message) + Alice_DH_public

    Bob->>Bob: RK, CK = KDF(RK, DH(Bob_DH, Alice_DH))
    Bob->>Bob: MK, CK = KDF(CK)
    Bob->>Bob: Decrypt(MK, ciphertext)

    Bob->>Bob: Generate new DH keypair (Ratchet)
    Bob->>Alice: Encrypt(MK', reply) + Bob_DH_public

    Note over Alice,Bob: Keys continuously evolve<br/>Old keys deleted after use
```

---

## Group Encryption

Groups use Sender Keys for efficiency.

### Sender Key Distribution

```mermaid
sequenceDiagram
    participant Sender
    participant Server
    participant M1 as Member 1
    participant M2 as Member 2
    participant M3 as Member 3

    Note over Sender: Sender generates Sender Key (SK)

    Sender->>M1: Encrypt(SK) with pairwise session
    Sender->>M2: Encrypt(SK) with pairwise session
    Sender->>M3: Encrypt(SK) with pairwise session

    Note over Sender: Now can send to group efficiently

    Sender->>Server: Encrypt(message, SK)
    Server->>M1: Deliver
    Server->>M2: Deliver
    Server->>M3: Deliver

    M1->>M1: Decrypt with SK
    M2->>M2: Decrypt with SK
    M3->>M3: Decrypt with SK
```

### Sender Key Rotation

Rotate sender key when:
- Member leaves group
- Member's device changes
- Periodic rotation (e.g., every 100 messages)

---

## Key Storage

### Client-Side

```
Device Storage (encrypted with device PIN/biometric):
├── identity_keypair        # Never leaves device
├── signed_prekey           # Current + previous
├── one_time_prekeys[]      # Unused pool
├── sessions/
│   ├── {user_id}/
│   │   ├── {device_id}/
│   │   │   ├── root_key
│   │   │   ├── chain_key
│   │   │   └── message_keys[]
│   │   └── ...
│   └── ...
└── sender_keys/
    └── {group_id}/
        └── {sender_id}     # Sender keys from others
```

### Server-Side

```
Server stores only public keys:
├── users/
│   └── {user_id}/
│       └── devices/
│           └── {device_id}/
│               ├── identity_key_public
│               ├── signed_prekey_public
│               ├── signed_prekey_signature
│               └── one_time_prekeys_public[]
```

---

## Multi-Device Support

Each device has independent identity:

```mermaid
graph TB
    subgraph "Alice's Devices"
        A_Phone[Phone<br/>IK_A1]
        A_Desktop[Desktop<br/>IK_A2]
    end

    subgraph "Bob's Devices"
        B_Phone[Phone<br/>IK_B1]
        B_Tablet[Tablet<br/>IK_B2]
    end

    A_Phone -->|Session| B_Phone
    A_Phone -->|Session| B_Tablet
    A_Desktop -->|Session| B_Phone
    A_Desktop -->|Session| B_Tablet
```

**Sending to Bob:**
1. Fetch all of Bob's device keys
2. Encrypt message with each device's session
3. Send all encrypted copies in one message

---

## Security Verification

### Safety Number

```
Concatenate and hash:
  SHA256(Alice_IK_public || Bob_IK_public)

Display as:
  60 digits grouped: 12345 67890 12345...

Or QR code containing both identity keys
```

Users can verify out-of-band to detect MITM.

---

## Threat Model

### Protected Against

| Threat | Protection |
|--------|------------|
| Server reading messages | E2E encryption |
| Past message exposure | Forward secrecy (key ratcheting) |
| Future message exposure | Future secrecy (key ratcheting) |
| Man-in-the-middle | Safety number verification |

### Not Protected Against

| Threat | Limitation |
|--------|------------|
| Compromised device | Keys stored on device |
| Metadata analysis | Server sees who messages whom |
| Sender/recipient identities | Not hidden from server |
| Message timing | Visible to server |

---

## Implementation Notes

### Cryptographic Primitives

| Purpose | Algorithm |
|---------|-----------|
| Identity keys | Ed25519 |
| Key agreement | X25519 (Curve25519) |
| Message encryption | AES-256-GCM |
| Key derivation | HKDF-SHA256 |
| Signatures | Ed25519 |

### Performance Considerations

- Pre-generate one-time keys in batches
- Cache session state
- Batch sender key distribution
- Lazy session establishment (on first message)
