# Image Upload Feature - Detailed Task Breakdown

## Overview

This document breaks down the implementation of E2E encrypted image upload into granular, actionable tasks. Each task is designed to be independently testable.

**Reference:** See `docs/IMAGE_UPLOAD_DESIGN.md` for the full design document.

---

## Phase 1: Server Infrastructure

### 1.1 Database Schema (Prisma)

**File:** `happy-server/prisma/schema.prisma`

- [ ] **Task 1.1.1:** Add `SessionBlob` model to Prisma schema
  ```prisma
  model SessionBlob {
      id          String   @id @default(uuid())
      sessionId   String
      session     Session  @relation(fields: [sessionId], references: [id], onDelete: Cascade)
      accountId   String
      account     Account  @relation(fields: [accountId], references: [id])
      mimeType    String
      size        Int
      s3Key       String
      createdAt   DateTime @default(now())

      @@index([sessionId])
      @@index([accountId])
  }
  ```

- [ ] **Task 1.1.2:** Add `blobs` relation to `Session` model
  ```prisma
  blobs SessionBlob[]
  ```

- [ ] **Task 1.1.3:** Add `sessionBlobs` relation to `Account` model
  ```prisma
  sessionBlobs SessionBlob[]
  ```

- [ ] **Task 1.1.4:** Run `npx prisma migrate dev --name add_session_blobs`

- [ ] **Task 1.1.5:** Generate Prisma client: `npx prisma generate`

### 1.2 Blob Routes

**File:** `happy-server/sources/app/api/routes/blobRoutes.ts` (new file)

- [ ] **Task 1.2.1:** Create `blobRoutes.ts` file with route registration function

- [ ] **Task 1.2.2:** Implement `POST /v1/sessions/:sid/blobs` endpoint
  - Validate auth via `app.authenticate`
  - Verify session ownership (accountId matches)
  - Accept raw binary body with headers:
    - `X-Blob-MimeType`: image/jpeg, image/png, image/gif, image/webp
    - `X-Blob-Size`: original size before encryption
  - Validate size limit (20MB max)
  - Generate UUID for blobId
  - Upload to Minio: `blobs/{accountId}/{sessionId}/{blobId}`
  - Create `SessionBlob` record in database
  - Return `{ blobId, size }`

- [ ] **Task 1.2.3:** Implement `GET /v1/sessions/:sid/blobs/:blobId` endpoint
  - Validate auth via `app.authenticate`
  - Verify blob ownership (accountId + sessionId match)
  - Stream encrypted blob from Minio
  - Set response headers:
    - `Content-Type: application/octet-stream`
    - `X-Blob-MimeType`: original mime type
    - `X-Blob-Size`: original size

- [ ] **Task 1.2.4:** Implement `DELETE /v1/sessions/:sid/blobs/:blobId` endpoint (optional)
  - For cleanup of orphaned blobs
  - Delete from Minio and database

### 1.3 Route Registration

**File:** `happy-server/sources/app/api/api.ts`

- [ ] **Task 1.3.1:** Import `blobRoutes` function
- [ ] **Task 1.3.2:** Register blob routes: `blobRoutes(app)`

### 1.4 Fastify Raw Body Configuration

**File:** `happy-server/sources/app/api/api.ts` or relevant config

- [ ] **Task 1.4.1:** Ensure Fastify is configured to handle raw binary bodies
  - May need `@fastify/multipart` or custom body parser
  - Configure content-type handling for `application/octet-stream`

### 1.5 Server Testing

- [ ] **Task 1.5.1:** Write unit tests for blob upload endpoint
- [ ] **Task 1.5.2:** Write unit tests for blob download endpoint
- [ ] **Task 1.5.3:** Manual test with curl:
  ```bash
  # Upload
  curl -X POST http://localhost:3000/v1/sessions/{sid}/blobs \
    -H "Authorization: Bearer {token}" \
    -H "X-Blob-MimeType: image/png" \
    -H "X-Blob-Size: 1234" \
    -H "Content-Type: application/octet-stream" \
    --data-binary @test-encrypted.bin

  # Download
  curl http://localhost:3000/v1/sessions/{sid}/blobs/{blobId} \
    -H "Authorization: Bearer {token}" \
    -o downloaded.bin
  ```

---

## Phase 2: CLI Support

### 2.1 Type Definitions

**File:** `happy-cli/src/api/types.ts`

- [ ] **Task 2.1.1:** Add `ImageRefContentSchema` for image references
  ```typescript
  export const ImageRefContentSchema = z.object({
    type: z.literal('image_ref'),
    blobId: z.string(),
    mimeType: z.enum(['image/jpeg', 'image/png', 'image/gif', 'image/webp']),
    width: z.number().optional(),
    height: z.number().optional(),
  });
  export type ImageRefContent = z.infer<typeof ImageRefContentSchema>;
  ```

- [ ] **Task 2.1.2:** Update `UserMessageSchema` to support array content
  ```typescript
  export const UserContentSchema = z.discriminatedUnion('type', [
    z.object({ type: z.literal('text'), text: z.string() }),
    ImageRefContentSchema,
  ]);

  export const UserMessageSchema = z.object({
    role: z.literal('user'),
    content: z.union([
      z.object({ type: z.literal('text'), text: z.string() }), // Legacy
      z.array(UserContentSchema), // New array format
    ]),
    localKey: z.string().optional(),
    meta: MessageMetaSchema.optional()
  });
  ```

- [ ] **Task 2.1.3:** Add `ClaudeImageContentBlock` type for Claude API format
  ```typescript
  export type ClaudeImageContentBlock = {
    type: 'image';
    source: {
      type: 'base64';
      media_type: 'image/jpeg' | 'image/png' | 'image/gif' | 'image/webp';
      data: string;
    };
  };
  ```

### 2.2 Binary Blob Decryption

**File:** `happy-cli/src/api/encryption.ts`

- [ ] **Task 2.2.1:** Add `decryptBlobWithDataKey` function
  ```typescript
  export function decryptBlobWithDataKey(
    bundle: Uint8Array,
    dataKey: Uint8Array
  ): Uint8Array | null {
    // Same logic as decryptWithDataKey but returns raw bytes
    // instead of JSON.parse()
  }
  ```

- [ ] **Task 2.2.2:** Add corresponding `encryptBlobWithDataKey` function (for testing)

- [ ] **Task 2.2.3:** Write unit tests for blob encryption/decryption round-trip

### 2.3 Blob Download

**File:** `happy-cli/src/api/apiSession.ts`

- [ ] **Task 2.3.1:** Add `downloadBlob` method to `ApiSessionClient` class
  ```typescript
  private async downloadBlob(blobId: string): Promise<Uint8Array> {
    const response = await fetch(
      `${this.serverUrl}/v1/sessions/${this.sessionId}/blobs/${blobId}`,
      { headers: { 'Authorization': `Bearer ${this.token}` } }
    );
    if (!response.ok) throw new Error(`Blob download failed: ${response.status}`);
    return new Uint8Array(await response.arrayBuffer());
  }
  ```

- [ ] **Task 2.3.2:** Add `decryptBlob` method
  ```typescript
  private decryptBlob(encryptedBlob: Uint8Array): Uint8Array | null {
    return decryptBlobWithDataKey(encryptedBlob, this.encryptionKey);
  }
  ```

### 2.4 Message Processing

**File:** `happy-cli/src/api/apiSession.ts`

- [ ] **Task 2.4.1:** Create `processUserMessageContent` helper function
  ```typescript
  private async processUserMessageContent(
    content: TextContent | ImageRefContent
  ): Promise<ClaudeTextBlock | ClaudeImageContentBlock>
  ```

- [ ] **Task 2.4.2:** Update message processing to handle `image_ref` content type
  - Check if `content` is array or single object
  - For each `image_ref`:
    1. Download encrypted blob
    2. Decrypt blob
    3. Base64 encode
    4. Return Claude API format

- [ ] **Task 2.4.3:** Add error handling for blob download/decryption failures
  - Log warning but don't crash session
  - Consider placeholder text: "[Image failed to load]"

### 2.5 Claude SDK Integration

**File:** `happy-cli/src/daemon/runClaude.ts` or equivalent

- [ ] **Task 2.5.1:** Verify Claude SDK accepts image content blocks
  - Check `stream-json` input format
  - Ensure multimodal messages are passed correctly

- [ ] **Task 2.5.2:** Test with mock image data end-to-end

### 2.6 CLI Testing

- [ ] **Task 2.6.1:** Unit tests for `ImageRefContentSchema` validation
- [ ] **Task 2.6.2:** Unit tests for blob download/decrypt flow
- [ ] **Task 2.6.3:** Integration test: encrypted blob → decrypted image → Claude format
- [ ] **Task 2.6.4:** Manual test with real server and test image

---

## Phase 3: App/Web Client Support

### 3.1 Type Definitions

**File:** `happy/sources/sync/typesRaw.ts`

- [ ] **Task 3.1.1:** Add `RawImageRefContentSchema`
  ```typescript
  const rawImageRefContentSchema = z.object({
    type: z.literal('image_ref'),
    blobId: z.string(),
    mimeType: z.enum(['image/jpeg', 'image/png', 'image/gif', 'image/webp']),
    width: z.number().optional(),
    height: z.number().optional(),
    thumbhash: z.string().optional(),
  });
  ```

- [ ] **Task 3.1.2:** Create `RawUserContentSchema` as discriminated union
  ```typescript
  const rawUserContentSchema = z.discriminatedUnion('type', [
    z.object({ type: z.literal('text'), text: z.string() }),
    rawImageRefContentSchema,
  ]);
  ```

- [ ] **Task 3.1.3:** Update user message schema to support content array
  ```typescript
  // Support both legacy single text and new array format
  content: z.union([
    z.object({ type: z.literal('text'), text: z.string() }),
    z.array(rawUserContentSchema),
  ])
  ```

### 3.2 Binary Blob Encryption

**File:** `happy/sources/sync/encryption/sessionEncryption.ts`

- [ ] **Task 3.2.1:** Add `encryptRaw` method for binary data
  ```typescript
  async encryptRaw(data: Uint8Array): Promise<string> {
    // Returns base64-encoded encrypted blob
  }
  ```
  Note: Check if this already exists or if `encryptRawRecord` can be adapted

- [ ] **Task 3.2.2:** Verify encryption format matches CLI's expected format

### 3.3 Blob Upload

**File:** `happy/sources/sync/sync.ts`

- [ ] **Task 3.3.1:** Add `uploadEncryptedBlob` private method
  ```typescript
  private async uploadEncryptedBlob(
    sessionId: string,
    encryptedData: Uint8Array,
    mimeType: string,
    originalSize: number
  ): Promise<string> // Returns blobId
  ```

- [ ] **Task 3.3.2:** Implement `sendMessageWithImages` method
  ```typescript
  async sendMessageWithImages(
    sessionId: string,
    text: string,
    images: Array<{
      data: Uint8Array,
      mimeType: string,
      width?: number,
      height?: number
    }>
  ): Promise<void>
  ```
  Steps:
  1. Get session encryption
  2. For each image: encrypt → upload → get blobId
  3. Build content array with text + image_refs
  4. Create RawRecord
  5. Encrypt and send via socket (same as sendMessage)

- [ ] **Task 3.3.3:** Add error handling and retry logic for upload failures

### 3.4 Storage Types

**File:** `happy/sources/sync/storageTypes.ts` (if exists)

- [ ] **Task 3.4.1:** Update storage types to handle image content in messages
- [ ] **Task 3.4.2:** Ensure message normalization handles new content format

### 3.5 UI Components - Image Input

**Files:** Various UI components (platform-specific)

- [ ] **Task 3.5.1:** Create `ImagePasteHandler` hook/component
  - Listen for paste events
  - Extract image data from clipboard
  - Convert to Uint8Array

- [ ] **Task 3.5.2:** Create `ImagePickerButton` component
  - Open file picker filtered to images
  - Read file as Uint8Array

- [ ] **Task 3.5.3:** Integrate image input with chat input component
  - Add image preview area
  - Support multiple images
  - Remove image button

- [ ] **Task 3.5.4:** Add upload progress indicator
  - Show progress during encryption
  - Show progress during upload

### 3.6 UI Components - Image Display

- [ ] **Task 3.6.1:** Create `MessageImage` component
  - Display image from blob
  - Handle loading state (use thumbhash placeholder if available)
  - Handle error state

- [ ] **Task 3.6.2:** Update message rendering to handle image content
  - Check for array content
  - Render text and images in order

- [ ] **Task 3.6.3:** Add image lightbox/zoom functionality (optional)

### 3.7 Thumbhash Generation (Optional Enhancement)

- [ ] **Task 3.7.1:** Install thumbhash library
- [ ] **Task 3.7.2:** Generate thumbhash before upload
- [ ] **Task 3.7.3:** Include thumbhash in image_ref content

### 3.8 App Testing

- [ ] **Task 3.8.1:** Unit tests for image content schema validation
- [ ] **Task 3.8.2:** Unit tests for blob encryption
- [ ] **Task 3.8.3:** Integration test: paste image → encrypt → upload → message sent
- [ ] **Task 3.8.4:** Manual test on web client
- [ ] **Task 3.8.5:** Manual test on mobile client (if applicable)

---

## Phase 4: Integration Testing & Polish

### 4.1 End-to-End Testing

- [ ] **Task 4.1.1:** E2E test: App paste image → Server store → CLI receive → Claude format
- [ ] **Task 4.1.2:** E2E test: Multiple images in single message
- [ ] **Task 4.1.3:** E2E test: Text + images mixed content
- [ ] **Task 4.1.4:** E2E test: Large image (10MB+) handling
- [ ] **Task 4.1.5:** E2E test: Session deletion cascades to blob cleanup

### 4.2 Error Handling

- [ ] **Task 4.2.1:** Handle network errors during blob upload
- [ ] **Task 4.2.2:** Handle invalid image formats gracefully
- [ ] **Task 4.2.3:** Handle blob not found (deleted session)
- [ ] **Task 4.2.4:** Handle decryption failures (corrupted blob)
- [ ] **Task 4.2.5:** Rate limiting for blob uploads

### 4.3 Performance

- [ ] **Task 4.3.1:** Client-side image compression for large images
- [ ] **Task 4.3.2:** Parallel blob downloads for multiple images
- [ ] **Task 4.3.3:** Blob caching in CLI (avoid re-downloading)

### 4.4 Cleanup & Maintenance

- [ ] **Task 4.4.1:** Orphaned blob cleanup job (blobs without message references)
- [ ] **Task 4.4.2:** Storage usage tracking per account
- [ ] **Task 4.4.3:** Admin endpoint to view blob storage stats

### 4.5 Documentation

- [ ] **Task 4.5.1:** Update API documentation with blob endpoints
- [ ] **Task 4.5.2:** Add image upload section to user documentation
- [ ] **Task 4.5.3:** Update CHANGELOG

---

## Task Summary

| Phase | Tasks | Estimated Effort |
|-------|-------|------------------|
| Phase 1: Server | 15 tasks | 4-6 hours |
| Phase 2: CLI | 14 tasks | 6-8 hours |
| Phase 3: App | 18 tasks | 12-16 hours |
| Phase 4: Testing | 15 tasks | 8-10 hours |
| **Total** | **62 tasks** | **30-40 hours** |

---

## Recommended Implementation Order

1. **Server first** (Phase 1) - Establishes the blob storage infrastructure
2. **CLI second** (Phase 2) - Enables receiving and decrypting images
3. **App types** (Phase 3.1-3.2) - Schema changes before UI work
4. **App upload** (Phase 3.3-3.4) - Blob upload without UI
5. **App UI** (Phase 3.5-3.6) - User-facing components
6. **E2E testing** (Phase 4.1) - Verify full flow works
7. **Polish** (Phase 4.2-4.5) - Error handling, performance, docs

---

## Key Files to Modify

| Component | File | Changes |
|-----------|------|---------|
| Server | `prisma/schema.prisma` | Add SessionBlob model |
| Server | `sources/app/api/routes/blobRoutes.ts` | New file - blob endpoints |
| Server | `sources/app/api/api.ts` | Register blob routes |
| CLI | `src/api/types.ts` | Add ImageRefContent schema |
| CLI | `src/api/encryption.ts` | Add blob decryption function |
| CLI | `src/api/apiSession.ts` | Add blob download, update message processing |
| App | `sources/sync/typesRaw.ts` | Add image content types |
| App | `sources/sync/encryption/sessionEncryption.ts` | Add binary encryption |
| App | `sources/sync/sync.ts` | Add sendMessageWithImages |
| App | UI components | Image input/display components |

---

## Dependencies Between Tasks

```
Phase 1.1 (Schema) → Phase 1.2 (Routes) → Phase 1.3 (Registration)
                                       ↓
Phase 2.1 (Types) → Phase 2.2 (Decrypt) → Phase 2.3 (Download) → Phase 2.4 (Processing)
                                                                           ↓
Phase 3.1 (Types) → Phase 3.2 (Encrypt) → Phase 3.3 (Upload) → Phase 3.5 (UI Input)
                                                              ↓
                                                      Phase 3.6 (UI Display)
                                                              ↓
                                                      Phase 4.1 (E2E Tests)
```

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Encryption mismatch | Test encryption/decryption round-trip early |
| Large file handling | Implement streaming upload/download |
| Memory pressure | Process images in chunks, avoid loading full image in memory |
| Auth issues | Reuse existing session auth, test with multiple accounts |
| Minio configuration | Verify bucket permissions, test with local Minio first |
