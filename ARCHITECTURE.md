# MailOSX v0.1 architecture

MailOSX is a local-first mail system: local state updates immediately; network work is durable, incremental, and idempotent.

```text
Browser / mobile client
  IndexedDB (messages, bodies, FTS index, operation outbox, transfer parts)
       | immediate read / optimistic write
       +-- Sync API --> modular monolith --> Postgres / object storage
                              |                  |
                              +-- SMTP outbound   +-- malware scan
                              +-- SMTP inbound --> message processor
```

## Client

Use TypeScript + React, a service worker, IndexedDB (Dexie), and Web Workers. The UI reads folder summaries and already-synced messages from IndexedDB first; sync uses a monotonic mailbox cursor and merges server changes in the background. A durable outbox stores user operations with an idempotency key, retry count, and next-at timestamp.

Transfers use tus-style, content-addressed chunks. Persist `(uploadId, chunk indexes, hash, byte count)` locally. Upload missing chunks with exponential backoff and jitter; ask the server which chunks it owns after restart or network switching. Downloads use ranged requests, a bitmap of verified ranges, and an object hash. Metadata and text body jobs always outrank HTML, images, and attachments.

## Modular backend

Start as one deployable TypeScript service (Fastify/NestJS) backed by PostgreSQL, Redis, and S3-compatible object storage (MinIO in development). Separate workers only for SMTP delivery, inbound ingestion, search indexing, attachment scanning, and async queues.

Core tables: `users`, `sessions`, `mailboxes`, `messages`, `message_recipients`, `message_versions`, `attachments`, `upload_sessions`, `upload_chunks`, `client_operations`, `sync_changes`, `transfer_jobs`, `audit_events`, and `abuse_events`.

```text
Compose → IndexedDB outbox → POST /messages (idempotency key)
  → accepted message state → outbound queue → SMTP MX delivery
  → per-recipient delivery state → sync change → client merge

SMTP inbound → SPF/DKIM/DMARC + reputation + malware/link scan
  → object storage + normalized metadata/body → inbox/spam decision
  → search index + sync change → client gets metadata before assets
```

## API outline

- `POST /auth/register`, `POST /auth/login`, `POST /auth/recovery/*`, `POST /auth/logout`
- `GET /sync?cursor=` returns mailbox changes, message metadata, and a new cursor.
- `GET /messages/:id?representation=text|html` returns the requested priority tier.
- `POST /messages` accepts a message and `Idempotency-Key`; duplicate keys return the original acceptance result.
- `POST /uploads`, `HEAD /uploads/:id`, `PATCH /uploads/:id` provide resumable chunks; completion verifies SHA-256.
- `GET /attachments/:id` supports `Range` and immutable chunk hashes.
- `GET /search?q=` combines local results with optional server continuation.

## Identity, safety, and operations

Use Argon2id passwords, opaque rotating session tokens in secure HttpOnly cookies, email recovery with rate limits, MFA-ready TOTP fields, encrypted secrets, TLS everywhere, DKIM signing, SPF/DMARC, and per-account/device/IP velocity limits. Apply CAPTCHA and outbound restrictions only after suspicious signals. Quarantine malicious attachments; rewrite/check risky links; preserve audit records.

Send Guard runs entirely on device: placeholder/template regexes, attachment-language detection, empty subject detection, and recipient/greeting heuristics. It returns warnings and offsets to highlight, never blocks sending.

Use Postgres full-text search for server fallback and IndexedDB/worker indexing for local instant search. Back up Postgres with PITR plus daily encrypted snapshots; object storage uses versioning, lifecycle policy, checksum audits, and cross-region replication. Monitor sync lag, queue depth, SMTP outcomes, resume success, duplicate-send rate, retransmitted bytes, text-readable time, and UI blocking time.

## Test gates

Run Playwright UI tests with offline/online transitions, integration tests for idempotency and cursor conflict merges, and transfer tests at 1/5/20/100 KB/s, 500 ms latency, 2%/10% loss, random 5–30 s disconnects, restarts, and Wi-Fi/mobile switches. Assert no duplicate send and byte-perfect resumed attachments.
