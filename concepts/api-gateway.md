# API Gateway
**Introduced:** Day 12

---

## What It Is

Single entry point for all client requests. Handles cross-cutting concerns once so microservices don't have to.

## Responsibilities

- **Auth:** validates JWT at gateway, forwards user context header to services
- **Rate limiting:** per-user/tier Redis counter, 429 on breach
- **Routing:** URL path → correct microservice
- **SSL termination:** HTTPS → plain HTTP internally
- **Transformation:** format conversion between external and internal
- **Observability:** centralized traffic logging

## vs Load Balancer

Load balancer: routes TCP/HTTP traffic by IP/port/path.
API Gateway: does all of the above + auth, rate limiting, transforms.

Examples: Kong, AWS API Gateway, Spring Cloud Gateway.

## In Kubernetes

Ingress controller (nginx) = load balancer.
Kong or Spring Cloud Gateway = API gateway, sits behind ingress.
