# TLS — Transport Layer Security
**Introduced:** Day 11

---

## What It Does

Two things: encrypts traffic, verifies identity (certificate).

## TLS 1.3 Handshake

```
Client → ClientHello    (supported cipher suites)
       ← ServerHello    (chosen cipher + certificate)
Client verifies certificate against trusted CA
Client → Key exchange   (using server's public key)
       ← Finished
Both sides derive same symmetric key → all traffic encrypted
```

TLS 1.2: 2 round trips. TLS 1.3: 1 round trip. TLS 1.3 supports 0-RTT for returning clients.

## TLS Termination in Kubernetes

Terminate at the **ingress controller** (nginx, Istio), not at each microservice.

```
Client → [TLS] → Ingress → [plain HTTP] → Service A
                         → [plain HTTP] → Service B
```

One certificate to manage. One place to rotate. One place to debug.
Terminating at each service = 12 certificates, 12 expiry dates, 12 failure points.

## mTLS (Mutual TLS)

Both client and server present certificates. Used in service meshes (Istio) for zero-trust.
Every service-to-service call authenticated + encrypted without each service managing its own cert.
Sidecar proxies (Envoy) handle it transparently.
