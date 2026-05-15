# Case Study: GitHub Gist
**Covered:** Day 2 | **Topic:** Pastebin at developer scale

---

## What They Got Right

### Metadata + blob separation
- Content stored in Git objects (essentially blob storage), not in DB
- Metadata (gist_id, owner, visibility, created_at) in MySQL — tiny, fast, cacheable
- Same pattern as the Pastebin design

### CDN for static content
- Heavy CDN usage — paste content served from edge
- Origin (S3/Git objects) never hit on repeat views of same content
- Latency improvement: ~200ms → ~10ms for cached content

---

## Unique Challenges (beyond basic Pastebin)

### Syntax Highlighting
- Parsing and highlighting 10MB of code on every request is CPU-intensive
- Solution: **render once, cache the highlighted HTML output**
- Never re-render unless content changes
- The rendered HTML is stored separately from raw content

### Forks and Revisions
- Each edit creates a new version (full history preserved)
- Naive approach: store each version separately → massive storage cost
- Git solution: **content-addressable storage**
  - If two versions share identical lines/blocks → they reference the same blob
  - Only deltas (differences) stored as new objects
  - Massive storage savings at scale

---

## The Lesson
The metadata/blob separation pattern you designed is production-proven across every major system handling user-generated content:
- GitHub Gist → Git objects + MySQL
- Notion → block metadata in DB + files in S3
- Google Docs → document metadata in Spanner + content in GCS
- Pastebin → paste metadata in DB + text in S3
