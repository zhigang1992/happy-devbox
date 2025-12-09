# E2E Encrypted Image Upload Design

## Overview

This document describes the design for adding image upload support to Happy, allowing users to paste images in the app/web client that get forwarded to Claude Code for multimodal understanding.

**Key requirement:** Maintain the existing zero-knowledge encryption architecture where the server never sees plaintext data.

## Current Architecture Summary

### Encryption Model

Happy uses a zero-knowledge E2E encryption model:

```
App encrypts → Server stores encrypted blob → CLI decrypts
```

**Key hierarchy:**
- Master secret (in app memory / CLI's `~/.happy/access.key`)
- Per-session `dataEncryptionKey` (AES-256-GCM, 32 bytes)
- Server only stores encrypted blobs + metadata (IDs, timestamps)

**Encryption algorithm:** AES-256-GCM with 12-byte nonce
- Format: `[version: 1 byte] [nonce: 12 bytes] [ciphertext: N bytes] [auth tag: 16 bytes]`

### Current Message Flow

```
App → encrypt(message) → Server (stores blob) → CLI → decrypt(message) → Claude Code
```

Messages are JSON structures encrypted with the session's `dataEncryptionKey`:
```typescript
{
  role: 'user',
  content: { type: 'text', text: '...' },
  meta: { sentFrom: 'web', permissionMode: '...', ... }
}
```

### Key Files

| Purpose | File Path |
|---------|-----------|
| App encryption core | `happy/sources/sync/encryption/encryption.ts` |
| Session encryption | `happy/sources/sync/encryption/sessionEncryption.ts` |
| CLI encryption | `happy-cli/src/api/encryption.ts` |
| CLI message handling | `happy-cli/src/api/apiSession.ts` |
| Message types (CLI) | `happy-cli/src/api/types.ts` |
| Message types (App) | `happy/sources/sync/typesRaw.ts` |
| Server session routes | `happy-server/sources/app/api/routes/sessionRoutes.ts` |
| Existing file upload | `happy-server/sources/storage/uploadImage.ts` |

---

## Proposed Solution: Option B - Encrypted Blob Storage

### Why Blob Storage?

1. **Maintains zero-knowledge** - Server stores encrypted blobs, never sees images
2. **Uses existing encryption** - Same session keys (AES-256-GCM), no new crypto needed
3. **Scalable** - Images don't bloat WebSocket messages
4. **Clean separation** - Blob upload is independent of message flow

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              APP / WEB CLIENT                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. User pastes image                                                       │
│  2. Generate blobId (UUID)                                                  │
│  3. Encrypt image with session's encryptor (AES-256-GCM)                   │
│  4. Upload encrypted blob to server: POST /v1/sessions/:sid/blobs           │
│  5. Send message with image_ref content type                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                 SERVER                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  Blob Upload Endpoint (NEW):                                                │
│  - POST /v1/sessions/:sid/blobs                                             │
│  - Validates auth + session ownership                                       │
│  - Stores encrypted blob to Minio: blobs/{accountId}/{sessionId}/{blobId}  │
│  - Returns blobId + download URL                                            │
│                                                                             │
│  Blob Download Endpoint (NEW):                                              │
│  - GET /v1/sessions/:sid/blobs/:blobId                                      │
│  - Validates auth (CLI or app)                                              │
│  - Streams encrypted blob from Minio                                        │
│  - Server NEVER decrypts (zero-knowledge preserved)                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                   CLI                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. Receive message with content.type === 'image_ref'                       │
│  2. Download encrypted blob: GET /v1/sessions/:sid/blobs/:blobId            │
│  3. Decrypt with session's encryptionKey (same as messages)                 │
│  4. Convert to Claude API format:                                           │
│     { type: 'image', source: { type: 'base64', media_type, data } }        │
│  5. Pass to Claude Code SDK via stream-json input                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLAUDE CODE SDK                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  Receives message with image content block in Anthropic API format          │
│  Processes image alongside text for multimodal understanding                │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Structures

### 1. New Message Content Type (App Schema)

**File:** `happy/sources/sync/typesRaw.ts`

```typescript
// Add new image_ref content schema
const rawImageRefContentSchema = z.object({
  type: z.literal('image_ref'),
  blobId: z.string(),           // UUID of the blob
  mimeType: z.enum(['image/jpeg', 'image/png', 'image/gif', 'image/webp']),
  width: z.number().optional(),
  height: z.number().optional(),
  thumbhash: z.string().optional(), // For placeholder while loading
});

// Update user content to support both text and image_ref
const rawUserContentSchema = z.discriminatedUnion('type', [
  z.object({ type: z.literal('text'), text: z.string() }),
  rawImageRefContentSchema,
]);

// User message can now have array of content
z.object({
  role: z.literal('user'),
  content: z.union([
    z.object({ type: z.literal('text'), text: z.string() }),  // Legacy single text
    z.array(rawUserContentSchema)                              // New: array of text/images
  ]),
  meta: MessageMetaSchema.optional()
})
```

### 2. CLI Types Update

**File:** `happy-cli/src/api/types.ts`

```typescript
export const ImageRefContentSchema = z.object({
  type: z.literal('image_ref'),
  blobId: z.string(),
  mimeType: z.enum(['image/jpeg', 'image/png', 'image/gif', 'image/webp']),
  width: z.number().optional(),
  height: z.number().optional(),
});

export const UserMessageSchema = z.object({
  role: z.literal('user'),
  content: z.union([
    z.object({ type: z.literal('text'), text: z.string() }),
    z.array(z.discriminatedUnion('type', [
      z.object({ type: z.literal('text'), text: z.string() }),
      ImageRefContentSchema,
    ]))
  ]),
  localKey: z.string().optional(),
  meta: MessageMetaSchema.optional()
});
```

### 3. Database Schema (Server)

**File:** `happy-server/prisma/schema.prisma`

```prisma
model SessionBlob {
    id          String   @id @default(uuid())
    sessionId   String
    session     Session  @relation(fields: [sessionId], references: [id], onDelete: Cascade)
    accountId   String
    account     Account  @relation(fields: [accountId], references: [id])
    mimeType    String
    size        Int      // Original size before encryption
    s3Key       String   // Path in Minio: blobs/{accountId}/{sessionId}/{id}
    createdAt   DateTime @default(now())

    @@index([sessionId])
    @@index([accountId])
}
```

---

## API Endpoints

### 1. Upload Encrypted Blob

**Endpoint:** `POST /v1/sessions/:sid/blobs`

**File:** `happy-server/sources/app/api/routes/blobRoutes.ts` (new file)

```typescript
import { FastifyInstance } from 'fastify';
import { randomUUID } from 'crypto';
import { db } from '../../db';
import { s3client, S3_BUCKET } from '../../../storage/files';

export function blobRoutes(app: FastifyInstance) {

  // Upload encrypted blob
  app.post<{
    Params: { sid: string }
  }>('/v1/sessions/:sid/blobs', {
    preHandler: app.authenticate,
    config: {
      rawBody: true, // Need raw binary body
    },
  }, async (request, reply) => {
    const { sid } = request.params;
    const userId = request.user.id;

    // Verify session ownership
    const session = await db.session.findFirst({
      where: { id: sid, accountId: userId }
    });
    if (!session) {
      return reply.status(404).send({ error: 'Session not found' });
    }

    const blobId = randomUUID();
    const mimeType = request.headers['x-blob-mimetype'] as string;
    const originalSize = parseInt(request.headers['x-blob-size'] as string);

    if (!mimeType || !['image/jpeg', 'image/png', 'image/gif', 'image/webp'].includes(mimeType)) {
      return reply.status(400).send({ error: 'Invalid or missing X-Blob-MimeType header' });
    }

    // Get raw encrypted body
    const encryptedBody = request.body as Buffer;

    if (encryptedBody.length > 20 * 1024 * 1024) { // 20MB limit
      return reply.status(413).send({ error: 'Blob too large (max 20MB)' });
    }

    // Upload to Minio (encrypted, server doesn't touch content)
    const s3Key = `blobs/${userId}/${sid}/${blobId}`;
    await s3client.putObject(S3_BUCKET, s3Key, encryptedBody, encryptedBody.length, {
      'Content-Type': 'application/octet-stream',
      'x-amz-meta-original-mimetype': mimeType,
      'x-amz-meta-original-size': originalSize.toString(),
    });

    // Save metadata to database
    await db.sessionBlob.create({
      data: {
        id: blobId,
        sessionId: sid,
        accountId: userId,
        mimeType,
        size: originalSize,
        s3Key,
      }
    });

    return { blobId, size: encryptedBody.length };
  });

  // Download encrypted blob
  app.get<{
    Params: { sid: string; blobId: string }
  }>('/v1/sessions/:sid/blobs/:blobId', {
    preHandler: app.authenticate,
  }, async (request, reply) => {
    const { sid, blobId } = request.params;
    const userId = request.user.id;

    // Verify ownership
    const blob = await db.sessionBlob.findFirst({
      where: { id: blobId, sessionId: sid, accountId: userId }
    });
    if (!blob) {
      return reply.status(404).send({ error: 'Blob not found' });
    }

    // Stream from Minio
    const stream = await s3client.getObject(S3_BUCKET, blob.s3Key);

    reply.header('Content-Type', 'application/octet-stream');
    reply.header('X-Blob-MimeType', blob.mimeType);
    reply.header('X-Blob-Size', blob.size.toString());

    return reply.send(stream);
  });
}
```

---

## Client-Side Implementation

### App: Encrypt and Upload Image

**File:** `happy/sources/sync/sync.ts` (add method)

```typescript
async sendMessageWithImages(
  sessionId: string,
  text: string,
  images: Array<{ data: Uint8Array, mimeType: string, width?: number, height?: number }>
) {
  const encryption = this.encryption.getSessionEncryption(sessionId);
  if (!encryption) {
    console.error(`Session ${sessionId} not found`);
    return;
  }

  const session = storage.getState().sessions[sessionId];
  if (!session) {
    console.error(`Session ${sessionId} not found in storage`);
    return;
  }

  // Upload each image as encrypted blob
  const imageRefs: ImageRefContent[] = [];
  for (const image of images) {
    // Encrypt image data using session encryption (same as messages)
    const encryptedImage = await encryption.encryptRaw(image.data);
    const encryptedBuffer = decodeBase64(encryptedImage, 'base64');

    // Generate thumbhash for placeholder (optional)
    // const thumbhash = await generateThumbhash(image.data);

    // Upload to server
    const response = await fetch(`${getServerUrl()}/v1/sessions/${sessionId}/blobs`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.credentials.token}`,
        'Content-Type': 'application/octet-stream',
        'X-Blob-MimeType': image.mimeType,
        'X-Blob-Size': image.data.length.toString(),
      },
      body: encryptedBuffer,
    });

    if (!response.ok) {
      throw new Error(`Failed to upload blob: ${response.status}`);
    }

    const { blobId } = await response.json();

    imageRefs.push({
      type: 'image_ref',
      blobId,
      mimeType: image.mimeType as any,
      width: image.width,
      height: image.height,
      // thumbhash,
    });
  }

  // Build content array
  const content: UserContent[] = [];
  if (text) {
    content.push({ type: 'text', text });
  }
  content.push(...imageRefs);

  // Create and encrypt message (same flow as sendMessage)
  const localId = randomUUID();
  const rawRecord: RawRecord = {
    role: 'user',
    content,  // Array of text + image_refs
    meta: {
      sentFrom: Platform.OS === 'web' ? 'web' : Platform.OS,
      permissionMode: session.permissionMode || 'default',
      // ... other meta fields
    }
  };

  const encryptedRawRecord = await encryption.encryptRawRecord(rawRecord);

  // Send via socket (same as regular messages)
  apiSocket.send('message', {
    sid: sessionId,
    message: encryptedRawRecord,
    localId,
  });
}
```

---

## CLI-Side Implementation

### Download and Decrypt Images

**File:** `happy-cli/src/api/apiSession.ts` (modify message handling)

```typescript
// Add to ApiSessionClient class

private async downloadBlob(blobId: string): Promise<Uint8Array> {
  const response = await fetch(
    `${configuration.serverUrl}/v1/sessions/${this.sessionId}/blobs/${blobId}`,
    {
      headers: { 'Authorization': `Bearer ${this.token}` }
    }
  );

  if (!response.ok) {
    throw new Error(`Failed to download blob ${blobId}: ${response.status}`);
  }

  return new Uint8Array(await response.arrayBuffer());
}

private decryptBlob(encryptedBlob: Uint8Array): Uint8Array {
  // Use same decryption as messages - decryptWithDataKey returns parsed JSON
  // For binary blobs, we need a variant that returns raw bytes
  return decryptBlobWithDataKey(encryptedBlob, this.encryptionKey);
}

async processUserMessage(message: UserMessage): Promise<ClaudeMessage> {
  // Handle legacy text-only format
  if (typeof message.content === 'object' && 'type' in message.content && message.content.type === 'text') {
    return {
      role: 'user',
      content: message.content.text
    };
  }

  // Handle new array format with images
  if (Array.isArray(message.content)) {
    const processedContent: ClaudeContentBlock[] = [];

    for (const block of message.content) {
      if (block.type === 'text') {
        processedContent.push({ type: 'text', text: block.text });
      } else if (block.type === 'image_ref') {
        // Download encrypted blob
        const encryptedBlob = await this.downloadBlob(block.blobId);

        // Decrypt using session key
        const decryptedImage = this.decryptBlob(encryptedBlob);

        // Convert to Claude API format
        processedContent.push({
          type: 'image',
          source: {
            type: 'base64',
            media_type: block.mimeType,
            data: encodeBase64(decryptedImage, 'base64'),
          }
        });
      }
    }

    return {
      role: 'user',
      content: processedContent
    };
  }

  throw new Error('Unknown message content format');
}
```

### Add Binary Blob Decryption

**File:** `happy-cli/src/api/encryption.ts` (add function)

```typescript
/**
 * Decrypt binary blob using AES-256-GCM with the data encryption key
 * Unlike decryptWithDataKey, this returns raw bytes instead of parsed JSON
 * @param bundle - The encrypted data bundle
 * @param dataKey - The 32-byte AES-256 key
 * @returns The decrypted binary data or null if decryption fails
 */
export function decryptBlobWithDataKey(bundle: Uint8Array, dataKey: Uint8Array): Uint8Array | null {
  if (bundle.length < 1) {
    return null;
  }
  if (bundle[0] !== 0) { // Only version 0
    return null;
  }
  if (bundle.length < 12 + 16 + 1) { // Minimum: version + nonce + auth tag
    return null;
  }

  const nonce = bundle.slice(1, 13);
  const authTag = bundle.slice(bundle.length - 16);
  const ciphertext = bundle.slice(13, bundle.length - 16);

  try {
    const decipher = createDecipheriv('aes-256-gcm', dataKey, nonce);
    decipher.setAuthTag(authTag);

    const decrypted = Buffer.concat([
      decipher.update(ciphertext),
      decipher.final()
    ]);

    return new Uint8Array(decrypted);
  } catch (error) {
    return null;
  }
}
```

---

## Implementation Plan

### Phase 1: Server Infrastructure (4-6 hours)
1. Add `SessionBlob` model to Prisma schema
2. Run migration
3. Create `blobRoutes.ts` with upload/download endpoints
4. Register routes in `api.ts`
5. Test with curl/Postman

### Phase 2: CLI Support (8-12 hours)
1. Add `ImageRefContentSchema` to types
2. Add `decryptBlobWithDataKey` function
3. Add `downloadBlob` method to `ApiSessionClient`
4. Modify message processing to handle `image_ref` content
5. Format images for Claude Code SDK
6. Test with mock encrypted blobs

### Phase 3: App Support (14-20 hours)
1. Update `typesRaw.ts` with new schemas
2. Add `sendMessageWithImages` method to `sync.ts`
3. Add binary blob encryption to `sessionEncryption.ts`
4. Create image picker/paste UI component
5. Integrate with chat input
6. Add thumbhash generation for placeholders
7. Handle loading states in message display

### Phase 4: Testing & Polish (8-12 hours)
1. E2E encryption round-trip tests
2. Large image handling
3. Error handling and retry logic
4. Progress indicators for upload
5. Cleanup of orphaned blobs

---

## Effort Estimate

| Component | Files to Change | New Files | Estimated Hours |
|-----------|-----------------|-----------|-----------------|
| Server: Blob routes | `api.ts` (register) | `blobRoutes.ts` | 4-6h |
| Server: Prisma schema | `schema.prisma` | - | 1h |
| App: Message types | `typesRaw.ts`, `storageTypes.ts` | - | 2h |
| App: Upload flow | `sync.ts`, `sessionEncryption.ts` | `blobUpload.ts` | 6-8h |
| App: UI for image paste | Various components | - | 8-12h |
| CLI: Types | `types.ts` | - | 1h |
| CLI: Download/decrypt | `apiSession.ts`, `encryption.ts` | - | 4-6h |
| CLI: Claude SDK format | `runClaude.ts`, `loop.ts` | - | 4-6h |
| Testing | - | Test files | 8-12h |

**Total: ~40-55 hours (1-1.5 weeks)**

---

## Security Considerations

1. **Zero-knowledge preserved** - Server never sees plaintext images
2. **Same encryption as messages** - No new key management needed
3. **Blob tied to session** - Can't access blobs from other sessions
4. **Size limits** - 20MB max per blob, can add rate limiting
5. **Cleanup** - Blobs cascade delete when session is deleted
6. **Auth required** - Both upload and download require valid session auth

---

## Future Enhancements

1. **Image compression** - Resize large images client-side before encryption
2. **Multiple images per message** - Already supported in schema
3. **Image thumbnails** - Generate encrypted thumbnails for faster loading
4. **Blob deduplication** - Hash-based dedup for repeated images
5. **Streaming decryption** - For very large images
6. **Image from URL** - Download and encrypt images from URLs

---

## References

- Existing encryption: `happy-cli/src/api/encryption.ts`
- Session encryption: `happy/sources/sync/encryption/sessionEncryption.ts`
- Minio setup: `happy-server/sources/storage/files.ts`
- Message flow: `happy/sources/sync/sync.ts:sendMessage()`
- CLI message handling: `happy-cli/src/api/apiSession.ts`
