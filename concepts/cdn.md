# CDN — Content Delivery Network
**Introduced:** Day 2 | Deepened: Chapter 3 (Days 11–15)

---

## What It Is
A network of geographically distributed servers that cache content close to users.

```
Without CDN: User in India → origin server in US → ~200ms
With CDN:    User in India → CDN edge in Mumbai  → ~10ms
```

---

## How It Works
```
First request → CDN miss → fetches from origin (S3/app) → caches at edge
Second request (same region) → CDN hit → served from edge instantly
```

CDN = geographic cache at the network level.

---

## What to Cache vs Not Cache

| Cache in CDN | Don't cache |
|---|---|
| Public static content (images, videos, files) | User-specific dynamic data |
| Public paste content | Private pastes |
| JS/CSS/HTML assets | Real-time data |
| Large content objects | Authenticated responses |

---

## When to Add CDN
- Content is large (images, video, files, text blobs)
- Content is public and same for all users
- Users are geographically distributed
- S3/origin latency is a bottleneck at scale

---

## Common CDN Providers
| Provider | Best for |
|---|---|
| Cloudflare | General use, DDoS protection |
| AWS CloudFront | Native S3 integration |
| Akamai | Enterprise |
| Fastly | Developer-friendly |

---

## Pastebin Example
```
Client → CloudFront (CDN) → S3 (origin)
```
Public paste content cached at CDN edge. S3 only hit on first request per region.
Latency: ~50ms (S3 direct) → ~5ms (CDN cached)
