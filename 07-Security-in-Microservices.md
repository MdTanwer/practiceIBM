# Microservices Study Notes — Section 7

## 7. Security in Microservices

---

### 7.1 Authentication and Authorization

---

#### 1. Authentication versus Authorization

| Aspect | Authentication (**AuthN**) | Authorization (**AuthZ**) |
| --- | --- | --- |
| **Core Question** | *"Who are you?"* (Verifying identity) | *"What are you allowed to do?"* (Verifying permissions) |
| **HTTP Error Code** | **`401 Unauthorized`** (missing, invalid, or expired credentials) | **`403 Forbidden`** (valid identity, insufficient roles/scopes/ownership) |
| **Mechanisms** | OpenID Connect (OIDC), Passkeys, Username/Password + MFA, X.509 / mTLS | OAuth 2.0 Scopes, RBAC, ABAC, Open Policy Agent (OPA) |
| **Where Enforced** | Identity Provider (Keycloak) + Edge API Gateway | **Coarse-grained** at Gateway + **Fine-grained** inside each microservice! |

---

#### 2. JWT (JSON Web Token)

A **JSON Web Token (JWT)** (RFC 7519) is a compact, URL-safe, cryptographically **signed** token (**JWS**) that carries identity and authorization claims across stateless microservices:
- **Why microservices love JWTs:** Any downstream microservice can verify a JWT **locally in memory** (using the IdP's cached public key) without calling a central session database or Auth service on every hop!
- **Critical Security Rule — Signed != Encrypted:**
  - Standard JWT (**JWS**) payloads are merely **Base64URL-encoded**, NOT encrypted! Anyone holding the token can decode the payload in 1 second.
  - **Never put secrets, passwords, SSNs, or PII you wouldn't expose inside a JWT** (use **JWE** if payload confidentiality is required).

---

#### 3. JWT Structure

A JWT consists of **three Base64URL-encoded parts separated by dots**: `Header.Payload.Signature`

```text
eyJhbGciOiJSUzI1NiIsImtpZCI6ImtleS0yMDI2LTAxIn0   ◄── 1. HEADER (Algorithm + Key ID)
.
eyJpc3MiOiJodHRwczovL2F1dGguZXhhbXBsZS5jb20iLCJzdWIiOiJ1c2VyLTQyIiwiYXVkIjoib3JkZXItc2VydmljZSIsImV4cCI6MTc5NTQzNzAwMCwicm9sZXMiOlsiQ1VTVE9NRVIiXX0   ◄── 2. PAYLOAD (Claims)
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c       ◄── 3. SIGNATURE (RS256 over Header + "." + Payload)
```

1. **Header:**
   - `alg`: Signing algorithm — always prefer **asymmetric `RS256` or `ES256`** in microservices (only Keycloak holds the private signing key; microservices only need the public key!). Avoid symmetric `HS256` across many services because every service would hold the shared signing secret and could forge tokens!
   - `kid`: Key ID used to look up the right rotating public key in **JWKS**.
2. **Payload (Claims):**
   - **Registered Claims:** `iss` (issuer), `sub` (subject/user ID), `aud` (audience/intended target service), `exp` (expiration epoch), `nbf` (not before), `iat` (issued at), `jti` (unique token ID for revocation).
   - **Custom Claims:** `scope`, `roles`, `tenant_id`.
3. **Signature:**
   - Computed over `base64Url(header) + "." + base64Url(payload)` to detect any tampering.

---

#### 4. Access Tokens

An **Access Token** is a short-lived credential (`5–15 minutes`) passed in HTTP requests (`Authorization: Bearer <token>`) to authorize API calls:
- **Stateless JWT Access Token:** Contains claims; verified offline by Resource Servers.
- **Opaque Access Token:** Random unguessable string; requires calling the Auth Server's `/introspect` endpoint (adds latency & SPOF unless cached).
- **Phantom Token Pattern (Best of Both Worlds):** Public clients (Web/Mobile) receive an **opaque token** (zero internal claims leaked to browsers); the **API Gateway** calls `/introspect` (cached briefly in Redis), exchanges it for an internal **signed JWT**, and passes the JWT downstream!

---

#### 5. Refresh Tokens

A **Refresh Token** is a long-lived credential (hours to days) sent **exclusively** to the Authorization Server's `/token` endpoint to obtain a fresh short-lived Access Token without forcing the user to re-enter credentials.

##### Production Refresh Token Security Rules
- **Refresh Token Rotation + Reuse Detection:** Whenever a client exchanges Refresh Token `R1` for a new Access Token, the Auth Server issues a brand-new `R2` and marks `R1` invalid. If an attacker steals `R1` and replays it (or the user replays `R1` after an attacker used it), the Auth Server detects reuse and **revokes the entire session/token family immediately**!
- **Browser Storage (XSS vs CSRF):** Never store refresh or access tokens in `localStorage` (accessible to any XSS script!). Either store tokens server-side using the **BFF (Backend for Frontend) Session Pattern** with `HttpOnly`, `Secure`, `SameSite=Lax` cookies, or use short-lived in-memory access tokens + rotated `HttpOnly` refresh cookies.

---

#### 6. Token Expiration

Because stateless JWTs cannot be un-signed once issued, keeping access token lifetimes **short (`5–15 minutes`)** is essential to bound the blast radius of a stolen token.

##### Emergency Revocation (When 15 Minutes Is Too Long)
- **Logout / Password Reset:** Always revoke the **Refresh Token** at the Auth Server so no new Access Tokens can be minted.
- **Immediate Revocation for Critical Operations (Banking / Admin):** Check token `jti` against a fast **Redis revocation blocklist** (stored only until `exp` so Redis stays tiny), or check a `token_version` timestamp on sensitive mutations.
- **Clock Skew Allowance:** Servers allow small clock skew (`30–60 seconds` leeway) when validating `exp` and `nbf`.

---

#### 7. Token Validation

Every API Gateway and Resource Server (Spring Security `oauth2ResourceServer` / Quarkus OIDC) **must** perform **all 7 checks** on every incoming JWT:
1. **Enforce Allowed Algorithm:** Reject `alg: "none"` and strictly require expected asymmetric algorithms (`RS256` / `ES256`)—prevents the classic **Algorithm Confusion Attack** (tricking a library into treating an RSA public key as an `HS256` HMAC shared secret!).
2. **Fetch Signing Key via `kid` from JWKS:** Download and cache the IdP's public keys (`/.well-known/jwks.json`), automatically refreshing on unknown `kid` during key rotation.
3. **Verify Cryptographic Signature:** Reject any tampered header or payload.
4. **Validate `exp` & `nbf`:** Reject expired or not-yet-valid tokens.
5. **Validate `iss` (Issuer):** Confirm the exact trusted Authorization Server URL (`https://auth.example.com/realms/prod`).
6. **Validate `aud` (Audience — Often Forgotten!):** Ensure the token was actually minted for *this* service (prevents a token issued for `NotificationService` from being replayed against `PaymentAdminService`).
7. **Validate `scope` / `roles` / `tenant_id`:** Enforce required authorization claims.

---

#### 8. OAuth 2.0

**OAuth 2.0** is an authorization delegation framework defining four roles:
- **Resource Owner** (End User), **Client** (Web/Mobile app or calling service), **Authorization Server** (Keycloak / Auth0), **Resource Server** (your Microservices).

##### Modern OAuth 2.0 Grant Flows
| Grant Flow | When to Use | Key Details |
| --- | --- | --- |
| **Authorization Code + PKCE** | **Web Apps, SPAs, & Mobile Apps** | Client uses cryptographic `code_verifier`/`code_challenge` (PKCE) so intercepted auth codes cannot be redeemed. *(Replaces deprecated Implicit & Password/ROPC flows!)* |
| **Client Credentials** | **Service-to-Service / Background Jobs** | Machine-to-machine auth using `client_id` + `client_secret` (or `private_key_jwt` cert). Zero human user context. |
| **Token Exchange (RFC 8693)** | **Downstream Microservice Delegation** | Microservice exchanges user's incoming JWT for a downscoped downstream JWT (`aud` + `act` claim). |

---

#### 9. OpenID Connect (OIDC)

OAuth 2.0 only handles **authorization** ("grant access to API"), not user identity. **OpenID Connect (OIDC)** adds a standardized **identity layer on top of OAuth 2.0** (`scope=openid profile email`):
- Adds the **`ID Token`** (a JWT containing authenticated user profile claims like `sub`, `email`, `name`, `auth_time`), the `/userinfo` endpoint, and Discovery (`/.well-known/openid-configuration`).
- **Golden Rule:**
  - **`ID Token`** → For the **Client App** to know who logged in (**AuthN**). Never send ID Tokens as `Authorization: Bearer` to backend microservices!
  - **`Access Token`** → Sent by the Client to **Backend Microservices / APIs** (**AuthZ**).

---

#### 10. Role-Based Access Control (RBAC)

**RBAC** grants permissions based on static user roles (`ROLE_CUSTOMER`, `ROLE_SUPPORT`, `ROLE_ADMIN`):
- In Spring Security: `@PreAuthorize("hasRole('ADMIN') or hasAuthority('SCOPE_orders:write')")`
- In Quarkus: `@RolesAllowed({"ADMIN", "ORDER_MANAGER"})`
- **Pros:** Simple to implement, fast to evaluate from JWT `roles` claim, easy compliance auditing.
- **Limitations (Role Explosion):** Cannot naturally express dynamic contextual rules like *"A manager may approve refunds **only up to $5,000** for orders **in their own region** during **business hours**."*

---

#### 11. Attribute-Based Access Control (ABAC)

**ABAC** evaluates fine-grained authorization policies dynamically using attributes of the **Subject** (user role, department, clearance), **Resource** (`order.ownerId`, `order.amount`, `order.region`), **Action** (`read`, `approve`, `refund`), and **Environment** (time, IP geo, mTLS trust):

| Aspect | RBAC | ABAC |
| --- | --- | --- |
| **Decision Basis** | Static user role names | **Dynamic policies over Subject + Resource + Action + Context attributes** |
| **Granularity** | Coarse-to-medium (endpoint/role) | **Fine-grained** (row/field/amount/tenant rules) |
| **Implementation** | JWT `roles` + `@PreAuthorize` | Custom domain guards, Spring Security custom `MethodSecurityExpression`, or **OPA (Open Policy Agent / Rego)** sidecar |

---

#### 12. Keycloak Fundamentals

**Keycloak** (upstream open-source foundation of **Red Hat Build of Keycloak**) is a full-featured **IAM / OAuth 2.0 / OIDC** server widely used in enterprise Java (Spring Boot & Quarkus) microservices:

- **Core Concepts:**
  - **Realms:** Completely isolated security domains (e.g., `internal-employees` realm vs `external-customers` realm, or per-tenant realms), each with its own users, clients, keys, and login themes.
  - **Clients:** Registered applications—**Public Clients** (SPAs/Mobile using PKCE, no client secret) vs **Confidential Clients** (Backend services / Gateways with secrets or mTLS `client_id` auth).
  - **Realm Roles vs Client Roles:** Global roles across the realm vs service-scoped roles (`order-service:refund`).
  - **Protocol Mappers:** Transform user attributes/groups into custom JWT claims (`tenant_id`, `region`, `roles`).
  - **Identity Brokering & User Federation:** Integrates seamlessly with LDAP / Active Directory, SAML 2.0, and social logins (Google/GitHub).
- **Quarkus / Spring Integration:** `quarkus-oidc` auto-discovers Keycloak endpoints (`/.well-known/openid-configuration`), validates JWTs, caches JWKS keys, and supports **Fine-Grained Authorization Services (UMA / RPT)** and **Token Propagation / Exchange** out of the box.

---

### 7.2 Service-to-Service Security

---

#### 1. Service Identity

In Kubernetes/cloud environments, Pod IP addresses are ephemeral and shared across nodes over time—**IP-based firewall rules (`allow 10.244.1.5`) cannot safely prove which microservice is calling**.

**Cryptographic Service Identity** binds a verifiable non-spoofable name to every workload using **X.509 certificates** following the **SPIFFE** standard (`spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>`):
- Example SAN URI inside `order-service` cert: `spiffe://prod.cluster.local/ns/commerce/sa/order-service`
- Tied directly to a Kubernetes `ServiceAccount`, automatically provisioned and rotated by **Istio / Linkerd / SPIRE**!

---

#### 2. Service-to-Service Authentication

How microservices verify each other on East–West internal traffic:
1. **Mutual TLS (mTLS) via Service Mesh:** Proves transport-level workload identity (`OrderService` pod talking to `PaymentService` pod) transparently via Envoy sidecars without changing Java code.
2. **OAuth 2.0 Client Credentials (`grant_type=client_credentials`):** Used for background/daemon jobs (`BillingCronService → PaymentService`) or calling external partner APIs—obtains a service JWT with scopes like `payments:reconcile`.
3. **Combined Defense (mTLS + End-User JWT):** **mTLS** proves *which service* opened the socket, while the **JWT** proves *which end user* triggered the action!

---

#### 3. Mutual TLS (mTLS)

In standard one-way HTTPS, only the client checks the server's certificate. In **Mutual TLS (mTLS)**, **BOTH the client and the server exchange, verify, and authenticate each other's X.509 certificates** during the TLS handshake:

```text
OrderService (Envoy Sidecar)                          PaymentService (Envoy Sidecar)
       │                                                             │
       │─────── 1. ClientHello ─────────────────────────────────────►│
       │◄────── 2. ServerHello + Payment Cert (SPIFFE ID) + ReqCert ─│
       │─────── 3. Order Cert (SPIFFE ID) + CertVerify + Finished ──►│
       │        [BOTH verify cert chain, expiry, & SPIFFE SAN!]      │
       │◄══════ 4. Encrypted mTLS Channel + Identity Policies ══════►│
```

- **Why Service Mesh (Istio / Red Hat OpenShift Service Mesh / Linkerd) is huge here:** Manually managing, distributing, and rotating X.509 keypairs across 50 Java truststores/keystores is error-prone. A Service Mesh injects Envoy sidecars that issue **short-lived (e.g., 24h) mTLS certificates automatically** and rotate them seamlessly in memory.

---

#### 4. Certificate Validation

Every mTLS handshake validates 4 things:
1. **Signature & Trust Chain:** Signed by the cluster's trusted internal Root/Intermediate CA.
2. **Validity Window:** Current timestamp falls between `notBefore` and `notAfter`.
3. **Identity / SAN Check:** The Subject Alternative Name (DNS or SPIFFE URI) matches an authorized caller identity (e.g., `PaymentService` `AuthorizationPolicy` only accepts `spiffe://.../sa/order-service` and `spiffe://.../sa/refund-worker`—blocking `catalog-service` even inside the same cluster!).
4. **Revocation:** Handled primarily via **very short certificate lifetimes (hours/1 day)** rather than fragile CRL/OCSP network lookups (plus OCSP stapling for edge certs).

---

#### 5. Token Propagation

**Token Propagation (Token Relay)** forwards the end-user's incoming `Authorization: Bearer <JWT>` unchanged across downstream service calls (`Gateway → OrderService → PaymentService`):
- **Usefulness:** `PaymentService` gets the original `sub` (`userId`) and `roles` for audit trails and ownership authorization (`order.ownerId == jwt.sub`).
- **Security Risk (Over-Broad Blast Radius / Confused Deputy):** If a user token has broad scopes (`orders:write`, `payments:write`, `admin:audit`) and `OrderService` propagates that exact token to a semi-trusted helper service (`LoggingAnalyticsService`), a vulnerability in `LoggingAnalyticsService` lets attackers replay that token against `PaymentService`!

---

#### 6. Token Exchange

**OAuth 2.0 Token Exchange (RFC 8693)** eliminates the over-broad Token Propagation risk:
- Instead of forwarding the raw user JWT, `OrderService` sends `(subject_token = userJWT, client_credentials = OrderService, audience = "payment-service", scope = "payments:charge")` to **Keycloak's `/token` endpoint**.
- Keycloak returns a **brand-new, short-lived, downscoped JWT** carrying:
  - `sub`: `"user-42"` (preserves end-user identity!)
  - `aud`: `"payment-service"` (cannot be replayed anywhere else!)
  - `scope`: `"payments:charge"` (least privilege!)
  - `act`: `{"sub": "order-service"}` (explicit **actor claim** for audit: *"OrderService acted on behalf of user-42"*).

| Dimension | Token Propagation (Relay) | Token Exchange (RFC 8693) |
| --- | --- | --- |
| **Token Scope & Audience** | Unchanged (broad scopes/audience!) | **Downscoped** (`aud` & `scope` narrowed per hop!) |
| **Performance** | Zero extra auth calls | 1 extra IdP call (mitigate with short in-memory cache!) |
| **Auditability** | Only shows end user | Records **both** user (`sub`) and acting service (`act`) |

---

#### 7. Zero-Trust Architecture

Traditional enterprise security used a **"castle-and-moat"** model: hard perimeter firewall, then plaintext HTTP inside the datacenter. If an attacker breached one pod, they could freely call every database and admin API inside the network.

**Zero-Trust Architecture** assumes **the internal network is already compromised**: *"Never trust, always verify."*
1. **Explicit Cryptographic Identity Everywhere:** Every human and every microservice has a strong identity (`OIDC JWT` + `SPIFFE mTLS`).
2. **Authenticate & Authorize Every Request:** No implicit trust based on IP, subnet, or namespace.
3. **Micro-Segmentation (`Least Privilege`):** Default-deny network policies + explicit L7 allowlists (`OrderService` may `POST /payments`, denied `DELETE /payments`).
4. **Encrypt All Data in Transit & at Rest:** Zero plaintext East–West traffic.

---

#### 8. Secure Internal Communication

- **HTTP / gRPC East–West:** Enforce `STRICT` mTLS (plaintext rejected) via Istio `PeerAuthentication` + L7 `AuthorizationPolicy`.
- **Kafka / RabbitMQ:** Enable TLS encryption + mTLS or SASL/SCRAM auth + **Topic ACLs** (`OrderService` can only `WRITE` to `order-events`; `AnalyticsService` can only `READ`).
- **PostgreSQL / Redis:** Enforce `sslmode=verify-full` / TLS + strong ACLs + dedicated credentials per service.
- **Kubernetes `NetworkPolicy` (Default-Deny L3/L4 Segmentation):** Explicitly allow only required pod-to-pod and pod-to-DB port connections; block `CatalogService` from even opening a TCP socket to `PaymentDB:5432`.

---

#### 9. Secret Management

**Never** store DB passwords, OAuth client secrets, JWT signing keys, or Stripe API keys in Git, Docker image layers (`ENV`), or plaintext config files!
- **Kubernetes `Secret` Trap:** Default K8s `Secrets` are merely **Base64-encoded** (not encrypted!) in `etcd` unless KMS envelope encryption-at-rest is enabled—and anyone with `get secrets` RBAC can read them.
- **Production Secrets Architecture:**
  - Store secrets in a dedicated vault (**HashiCorp Vault**, **AWS Secrets Manager**, **GCP Secret Manager**, **Azure Key Vault**).
  - Inject into pods via **Vault Agent Sidecar / CSI Secrets Store Driver** (mounted into in-memory `tmpfs` volumes, never written to disk) or **External Secrets Operator (ESO)** syncing securely with KMS-encrypted `etcd`.

---

#### 10. Credential Rotation

Static credentials that live for 3 years will eventually leak. Automate **zero-downtime rotation**:
- **JWKS Key Rotation (Overlap Window):** Publish new public key `kid: k2` in `/.well-known/jwks.json` *before* signing tokens with `k2`; keep old `k1` active until all tokens signed by `k1` expire.
- **Database Credential Rotation (Dual-User or Dynamic Secrets):**
  - *Dual-User:* Create `app_user_v2` → update Vault/apps → verify traffic → drop `app_user_v1`.
  - *Dynamic Ephemeral Secrets (HashiCorp Vault DB Engine):* Vault dynamically creates a unique PostgreSQL login with a **1-hour TTL** per pod lease, automatically rotates/renews leases, and revokes the user the instant the pod terminates!

---

#### 11. Least Privilege

Every microservice, database user, and container runs with the **strict minimum permissions needed to function**:
- **Database Users:** App user gets `SELECT, INSERT, UPDATE, DELETE` on *its own schema only*—never `SUPERUSER`, never `DROP TABLE` (keep Flyway/Liquibase DDL migrations under a separate CI migration user!).
- **Kubernetes Pods (`SecurityContext`):** `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities: { drop: ["ALL"] }`, dedicated `ServiceAccount` per service (`automountServiceAccountToken: false` if it doesn't call the K8s API).
- **OAuth Scopes & Cloud IAM (IRSA / Workload Identity):** Grant `OrderService` IAM access only to `s3://orders-invoices/*`, never `*` across all buckets.

---

### 7.3 API Security

---

#### 1. HTTPS

- **Mandatory everywhere:** Reject plain HTTP or immediately redirect `80 → 443` at the edge load balancer/gateway.
- **Modern TLS:** Allow **TLS 1.2 and TLS 1.3 only** (disable SSLv3, TLS 1.0/1.1 and weak CBC/RC4 ciphers).
- **HSTS (`Strict-Transport-Security`):** Tells browsers to automatically upgrade all future requests to HTTPS, blocking SSL-stripping attacks.
- **Automated Certs:** Use **cert-manager** (with Let's Encrypt or internal Vault PKI) in Kubernetes so TLS certificates never silently expire and cause a P1 outage!

---

#### 2. CORS (Cross-Origin Resource Sharing)

Browsers enforce the **Same-Origin Policy**—blocking `https://app.example.com` JavaScript from reading responses from `https://api.example.com` unless the server explicitly returns **CORS** headers (and handles preflight `OPTIONS` requests):
- `Access-Control-Allow-Origin: https://app.example.com`
- `Access-Control-Allow-Methods: GET, POST, PUT, DELETE`
- `Access-Control-Allow-Headers: Authorization, Content-Type, Idempotency-Key`

##### Fatal CORS Anti-Patterns
1. **Never combine `Access-Control-Allow-Origin: *` with `Access-Control-Allow-Credentials: true`** (rejected by browsers and leaks intent).
2. **Never blindly reflect the incoming `Origin` header** (`allowOrigin = request.getHeader("Origin")`)! Any malicious site (`https://evil.com`) could then steal authenticated API responses! Always match against an **exact allowlist** configured centrally at the **API Gateway** (don't duplicate CORS on 20 downstream microservices!).

---

#### 3. CSRF (Cross-Site Request Forgery)

**CSRF** tricks a logged-in user's browser on `evil.com` into silently firing a state-changing request (`<form action="https://bank.com/transfer" method="POST">`) to a trusted site—exploiting the fact that **browsers automatically attach cookies** to cross-site requests!

- **Why pure `Authorization: Bearer <JWT>` APIs are immune:** JavaScript on `evil.com` cannot read or auto-attach custom `Authorization` headers on cross-origin requests.
- **When CSRF protection IS mandatory:** If your **BFF or API Gateway uses session cookies** (`HttpOnly` cookie auth)!
  - **Defenses:** Set **`SameSite=Lax`** (or `Strict`) on all session cookies + require an anti-CSRF token (`X-XSRF-TOKEN` Synchronizer Token / Double-Submit Cookie) + validate `Origin` / `Referer` headers on state-changing requests (`POST`/`PUT`/`PATCH`/`DELETE`).

---

#### 4. SQL Injection

Occurs when untrusted user input is concatenated directly into SQL, JPQL, or NoSQL queries:
```java
// VULNERABLE: Attacker sends name = "x' OR '1'='1"
String sql = "SELECT * FROM users WHERE email = '" + email + "'";
```

##### Defenses in Java Microservices
1. **Always Use Parameterized Queries / Prepared Statements:** Spring Data JPA named parameters (`@Param("email")`), `JdbcTemplate` (`?` placeholders), or Hibernate Criteria API—SQL parameters are never parsed as SQL code!
2. **Watch Out for Dynamic `ORDER BY` & `LIKE`!** Parameterized `?` placeholders **cannot** bind column names in `ORDER BY :sortField`. Always validate sort/filter field names against a **strict Java `Set<String>` allowlist** (`Set.of("createdAt", "totalAmount")`)!
3. **NoSQL Injection (MongoDB):** Validate input types strictly (reject raw JSON operator maps like `{"$ne": ""}` in query inputs).

---

#### 5. Broken Authentication (and BOLA / IDOR)

Covers the top OWASP API Security Top 10 threats (**API1: BOLA** & **API2: Broken Authentication**):
- **Broken Authentication Pitfalls:**
  - Accepting `alg: none` JWTs or missing `iss`/`aud`/`exp` verification.
  - Putting JWTs or API keys inside URL query strings (`GET /orders?token=eyJ...`) → leaks forever in NGINX/Gateway access logs, proxy caches, and `Referer` headers! Always use the `Authorization` header.
  - Unlimited login attempts (no lockout/rate limit/MFA).
- **Broken Object Level Authorization (BOLA / IDOR — OWASP API #1!):**
  - User A (`sub = u-100`) has a valid JWT and calls `GET /api/v1/orders/ord-999` (owned by User B!).
  - Gateway verifies the JWT is valid, **so only `OrderService` can prevent BOLA** by checking ownership in every DB query: `WHERE order_id = :orderId AND user_id = :authenticatedUserId`!

---

#### 6. Rate Limiting

Applied specifically for **security defense** at the API Gateway / WAF:
- **Tighter buckets on Auth & Money endpoints:** e.g., `/auth/login`, `/auth/otp`, `/password-reset` capped at `5 attempts / minute / IP + account` to block brute-force & credential stuffing.
- **Per-Tenant / Per-API-Key Quotas:** Prevents one compromised partner API key from starving cluster capacity (**Denial of Wallet / Resource Exhaustion**).

---

#### 7. API Abuse Prevention

Protects APIs from automated exploitation beyond simple request volume:
- **Mass Assignment Defense:** Never bind incoming JSON directly onto a JPA `@Entity`! Attackers send `{"email": "a@b.com", "role": "ADMIN", "accountBalance": 999999}` and Hibernate persists it. Always bind to **immutable Input DTO `record`s** containing *only* allowed fields.
- **Unbounded Pagination / GraphQL DoS:** Enforce `maxPageSize = 100` (`?size=1000000` causes OOM!) and cap GraphQL query depth & complexity scores.
- **Request Body Size Limits:** Reject oversized payloads early (`413 Payload Too Large`) at the API Gateway.
- **Bot Management & Anomaly Detection:** WAF rules, CAPTCHA/device attestation on sensitive public flows, and alerting on abnormal scraping/enumeration (`404` spikes across sequential IDs → prefer **UUIDv4/ULID** over sequential integer IDs `1, 2, 3` on public APIs!).

---

#### 8. Input Validation

Validate **every** external input strictly using an **Allowlist** approach at controller boundaries:
- Use **Jakarta Bean Validation (`@Valid`)**: `@NotBlank`, `@Size(max = 100)`, `@Positive`, `@DecimalMax`, `@Email`, `@Pattern(regexp = ...)`, strict `Enum` deserialization.
- Reject unsupported media types (`415 Unsupported Media Type`) and enforce strict JSON schema types.
- **File Uploads:** Validate **magic file bytes** (not just `.png` extension!), limit file size, strip metadata, scan with antivirus, rename to random UUIDs, and store in isolated object storage (S3/MinIO) served from a separate non-cookie domain.

---

#### 9. Security Headers

Enforce consistent HTTP security headers at the **API Gateway** (or Spring Security defaults):

| Header | Recommended Value | Protection |
| --- | --- | --- |
| **`Strict-Transport-Security`** | `max-age=31536000; includeSubDomains` | Forces HTTPS; blocks SSL downgrade attacks |
| **`X-Content-Type-Options`** | `nosniff` | Prevents MIME-type sniffing attacks |
| **`X-Frame-Options`** / **CSP** | `DENY` / `frame-ancestors 'none'` | Prevents clickjacking |
| **`Content-Security-Policy`** | `default-src 'none'; frame-ancestors 'none'` *(for JSON APIs)* | Prevents XSS / script injection if API response rendered |
| **`Cache-Control`** | `no-store` *(on sensitive/PII/auth responses)* | Prevents shared proxies/browsers caching private data |
| **`Server` / `X-Powered-By`** | *Strip / Omit* | Hides framework/version fingerprinting |

---

#### 10. Secure Error Handling

1. **Zero Internal Leakage:** Configure `server.error.include-stacktrace=never`, `include-message=never`, and return standardized **RFC 9457 `ProblemDetail`** responses containing only safe error codes, user-friendly messages, and a **`traceId`** (never SQL queries, Hibernate exceptions, file paths, or internal hostnames).
2. **Prevent User Enumeration:** On `/login` and `/forgot-password`, return identical responses (`"Invalid credentials"` / `"If that email exists, a reset link was sent"`) and avoid obvious timing differences between existing vs non-existing users.
3. **Mask Sensitive Data in Server Logs:** Log full diagnostic errors internally, **but mask/scrub** `Authorization` headers, passwords, JWTs, API keys, PAN credit card numbers, and PII before emitting logs to ELK/Loki!

---

#### Quick Interview Revision Checklist (Section 7)
- Can you draw a **JWT (`Header.Payload.Signature`)**, explain why **`Signed != Encrypted`**, and list all **7 validation steps** (especially `alg: none` / algorithm confusion, JWKS `kid`, and **`aud` check**)?
- Can you distinguish **OAuth 2.0 vs OIDC** (`Access Token` vs `ID Token`), and explain **Authorization Code + PKCE**, **Client Credentials**, **Refresh Token Rotation + Reuse Detection**, and **Phantom Token Pattern**?
- Can you contrast **RBAC vs ABAC (OPA)** and explain core **Keycloak** concepts (Realms, Clients, Protocol Mappers, Quarkus OIDC)?
- Can you explain **SPIFFE Service Identity**, how **mTLS** works in a Service Mesh, and why **Token Exchange (RFC 8693)** is safer than plain **Token Propagation**?
- Can you describe **Zero-Trust Architecture**, **Vault Dynamic Secrets**, **Credential Rotation**, and **Least Privilege** across DB + K8s pods?
- Can you defend against **CORS misconfigurations**, **CSRF (why Bearer headers are immune vs BFF cookies)**, **SQL Injection (`ORDER BY` allowlisting)**, **BOLA/IDOR (OWASP API #1)**, **Mass Assignment**, and **insecure error leakage**?
