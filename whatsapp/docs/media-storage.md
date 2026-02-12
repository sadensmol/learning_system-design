# Media Storage

Upload, storage, and delivery of encrypted media at WhatsApp scale.

---

## Design Principles

1. **Client-side encryption**: Media encrypted before upload
2. **Direct upload**: Bypass servers, upload directly to storage
3. **CDN delivery**: Edge caching for fast downloads
4. **Efficient thumbnails**: Low-bandwidth previews

---

## Media Flow Overview

```mermaid
sequenceDiagram
    participant Sender
    participant API as Media API
    participant Store as Object Store
    participant CDN
    participant Receiver

    Note over Sender: Encrypt media locally

    Sender->>API: RequestUpload(size, type)
    API-->>Sender: SignedUploadURL + media_id

    Sender->>Store: PUT encrypted blob
    Store-->>Sender: Upload complete

    Sender->>API: ConfirmUpload(media_id, encryption_key_encrypted)
    API-->>Sender: media_url

    Note over Sender: Send message with media_url + key

    Sender->>Receiver: Message(media_url, encrypted_key)

    Receiver->>CDN: GET media_url
    CDN-->>Receiver: Encrypted blob
    Receiver->>Receiver: Decrypt with key
```

---

## Client-Side Encryption

### Process

```mermaid
graph LR
    subgraph "Sender Client"
        Original[Original Media]
        Key[Generate AES-256 Key]
        Encrypt[Encrypt Media]
        EncMedia[Encrypted Blob]
        EncKey[Encrypt Key with<br/>recipient's session]
    end

    Original --> Encrypt
    Key --> Encrypt
    Encrypt --> EncMedia
    Key --> EncKey
```

### Media Encryption Format

```
Encrypted media structure:
├── Header (16 bytes)
│   ├── magic: "WAMEDIA1"
│   └── version: 1
├── IV (16 bytes)
├── Encrypted content (AES-256-CBC)
└── HMAC (32 bytes)

Encryption key bundle (sent in message):
├── media_key (32 bytes) - encrypted with session
├── file_sha256 (32 bytes) - integrity check
├── media_key_timestamp
└── direct_path (URL path)
```

---

## Upload Flow

### Step 1: Request Upload

```
POST /v1/media/upload/request
{
  "file_size": 1048576,
  "file_type": "image/jpeg",
  "file_hash": "sha256:abc123..."
}

Response:
{
  "media_id": "m_xyz789",
  "upload_url": "https://upload.whatsapp.net/...",
  "upload_expires": "2024-01-15T12:00:00Z"
}
```

### Step 2: Direct Upload

```mermaid
graph LR
    Client -->|PUT encrypted blob| ObjectStore[Object Storage]
    ObjectStore -->|Async| Thumbnail[Thumbnail Generator]
```

Upload URL is pre-signed:
- Valid for 5 minutes
- Includes file size limit
- Single-use

### Step 3: Confirm Upload

```
POST /v1/media/upload/confirm
{
  "media_id": "m_xyz789",
  "file_sha256": "abc123...",
  "thumbnail_base64": "..." // Optional client-generated
}

Response:
{
  "media_url": "https://cdn.whatsapp.net/v/t1/...",
  "thumbnail_url": "https://cdn.whatsapp.net/v/t1/.../thumb"
}
```

---

## Thumbnail Handling

### Client-Generated Thumbnails

For E2E encryption, server can't generate thumbnails from encrypted media.

```mermaid
graph LR
    subgraph "Sender"
        Orig[Original Image]
        GenThumb[Generate Thumbnail<br/>64x64, blur]
        EncThumb[Encrypt Thumbnail]
        Upload[Upload both]
    end

    Orig --> GenThumb
    GenThumb --> EncThumb
    Orig --> Upload
    EncThumb --> Upload
```

**Thumbnail specs:**
- Max 64x64 pixels
- Heavily blurred (preview only)
- Encrypted with same key as media
- Inline in message for instant preview

### Thumbnail in Message

```
Message payload:
{
  "type": "image",
  "media_url": "https://cdn.../abc123",
  "thumbnail_base64": "data:image/jpeg;base64,...", // ~2KB
  "media_key": "encrypted_key...",
  "file_sha256": "...",
  "width": 1920,
  "height": 1080
}
```

---

## Storage Architecture

```mermaid
graph TB
    subgraph "Upload Path"
        Client --> LB[Load Balancer]
        LB --> UploadAPI[Upload API]
        UploadAPI --> ObjectStore[Object Storage<br/>S3/GCS]
    end

    subgraph "Download Path"
        Client2[Client] --> CDN[CDN Edge<br/>200+ PoPs]
        CDN --> Origin[Origin Shield]
        Origin --> ObjectStore
    end

    subgraph "Storage Tiers"
        ObjectStore --> Hot[Hot Storage<br/>< 30 days]
        ObjectStore --> Cold[Cold Storage<br/>30+ days]
    end
```

### Storage Tiers

| Tier | Age | Storage Class | Access |
|------|-----|--------------|--------|
| Hot | 0-30 days | Standard | Frequent |
| Cold | 30+ days | Infrequent Access | Rare |

### Retention

- Media retained indefinitely (per requirements)
- No server-side deletion (encrypted, server can't identify content)
- Client-side "delete for everyone" sends delete message, clients remove locally

---

## CDN Strategy

### Edge Caching

```mermaid
graph TB
    subgraph "Global CDN"
        User1[User Americas] --> Edge1[Edge Americas]
        User2[User Europe] --> Edge2[Edge Europe]
        User3[User Asia] --> Edge3[Edge Asia]

        Edge1 --> Shield1[Origin Shield Americas]
        Edge2 --> Shield1
        Edge3 --> Shield2[Origin Shield Asia]

        Shield1 --> Origin[Origin Storage]
        Shield2 --> Origin
    end
```

### Cache Rules

| Content Type | Edge TTL | Shield TTL |
|--------------|----------|------------|
| Media files | 7 days | 30 days |
| Thumbnails | 7 days | 30 days |
| Profile photos | 1 day | 7 days |

### Cache Invalidation

Media is immutable (encrypted, unique key per media), so:
- No invalidation needed for media files
- New upload = new URL
- Efficient caching

---

## Download Flow

```mermaid
sequenceDiagram
    participant Client
    participant CDN as CDN Edge
    participant Origin
    participant Store as Object Store

    Client->>CDN: GET /v/t1/abc123
    alt Cache hit
        CDN-->>Client: Encrypted blob (cached)
    else Cache miss
        CDN->>Origin: GET /v/t1/abc123
        alt Origin cache hit
            Origin-->>CDN: Encrypted blob
        else Origin miss
            Origin->>Store: GET object
            Store-->>Origin: Encrypted blob
            Origin-->>CDN: Encrypted blob (cache 30d)
        end
        CDN-->>Client: Encrypted blob (cache 7d)
    end

    Client->>Client: Decrypt with media_key
```

---

## Media Processing

### Upload Processing Pipeline

```mermaid
graph LR
    Upload[Upload Complete] --> Validate[Validate]
    Validate --> Index[Index Metadata]
    Index --> Replicate[Cross-Region Replicate]
```

**Validation:**
- File size matches declared
- Hash matches declared
- Valid encrypted format
- No malware scanning (encrypted, can't scan)

### No Server-Side Transcoding

Unlike unencrypted platforms:
- Server cannot transcode (content encrypted)
- Client responsible for format compatibility
- Client compresses before encryption

---

## Bandwidth Optimization

### Resumable Uploads

```
PUT /upload?uploadType=resumable
X-Upload-Offset: 5242880

If interrupted:
HEAD /upload/{upload_id}
-> X-Upload-Offset: 5242880

Resume:
PUT /upload/{upload_id}
X-Upload-Offset: 5242880
[remaining bytes]
```

### Progressive Download

For video playback:
- Encrypted in chunks
- Each chunk independently decryptable
- Range requests supported
- Start playback before full download

---

## Group Media

### Deduplication Challenge

Same image sent to group - can we deduplicate?

**Problem:** Each recipient has different session key, so media key is encrypted differently.

**Solution:** Store encrypted blob once, multiple key bundles:

```
Media blob: https://cdn.../abc123 (single copy)

Message to group:
{
  "media_url": "https://cdn.../abc123",
  "file_sha256": "same_for_all",
  "media_key_per_recipient": {
    "user_1": "encrypted_key_for_user_1",
    "user_2": "encrypted_key_for_user_2",
    ...
  }
}
```

Actually: Use sender keys for groups, so one encrypted blob, one key distribution.

---

## Monitoring

| Metric | Alert Threshold |
|--------|-----------------|
| Upload success rate | < 99% |
| Upload p99 latency (1MB) | > 5s |
| Download p99 latency (1MB) | > 2s |
| CDN cache hit rate | < 80% |
| Storage growth rate | > 10% week-over-week |
| Cross-region replication lag | > 30 minutes |
