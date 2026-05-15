# Metadata + Blob Storage Pattern
**Introduced:** Day 2

---

## The Problem
Storing large content (files, images, videos, large text) directly in a relational DB destroys performance.
- DB spends time moving huge chunks of data instead of serving queries
- Index performance degrades
- Backup/restore becomes painful
- Cost: DB storage is 10–100× more expensive than object storage

---

## The Pattern
```
DB (Postgres):    small metadata row + pointer    (~200 bytes, fast, cheap)
Blob store (S3):  actual content                  (up to GBs, cheap, scalable)
```

## Example — Pastebin
```
Postgres row:
  paste_id: "xK9mP2"
  title: "My code snippet"
  s3_key: "pastes/xK9mP2.txt"
  expires_at: 2026-06-11

S3 object:
  key: "pastes/xK9mP2.txt"
  content: <10MB of text>
```

---

## Read Flow
```
GET /pastes/xK9mP2
  → Redis cache (metadata hit?) → s3_key
  → S3 fetch using s3_key → content
  → Return to user
```
DB only touched on cache miss. S3 always serves content directly (or via CDN).

---

## Used By
- GitHub Gist — Git blob objects + MySQL metadata
- Notion — block metadata in DB, file content in S3
- Google Docs — document metadata in Spanner, content in GCS
- Pastebin — paste metadata in DB, text content in S3
- Any system with user-uploaded files, images, or videos

---

## Rule
If a field can exceed ~1KB, don't store it in the DB. Store a pointer instead.
