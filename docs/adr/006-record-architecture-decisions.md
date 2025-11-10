# ADR-006: Authentication Strategy - JWT with RBAC

---

**Status:** Accepted  
**Date:** 12 October 2025  
**Last Updated:** 10 november 2025  

---

## Context and Problem Statement

The CMS has five distinct user roles with different permissions, accessed via web and mobile. With microservices architecture (ADR-001), each service needs to know who the user is and what they're allowed to do 

**The big question:** What authentication mechanism gives me secure, stateless authentication that works with microservices?

---

## Decision Drivers

- **NFR4 (Security):** Secure authentication with encrypted credentials
- **Microservices:** Auth must work across services without shared state
- **RBAC:** Must carry role information for authorization
- **Multi-tenant:** Must include tenant_id to prevent cross-tenant access
- **Performance:** Can't add significant latency
- **Mobile:** Has to work with React Native

---

## Options I Considered

### Option 1: Session-Based Authentication (Cookies)

**What it is:** Traditional server-side sessions stored in Redis, session ID in cookie.

**Pros:**
- Familiar pattern
- Server controls session invalidation instantly

**Cons:**
- **Stateful:** Requires shared Redis across all services
- **Scalability bottleneck:** Session storage becomes the bottleneck
- **Microservices anti-pattern:** All services depend on centralized store
- **Mobile:** Cookie management is painful on mobile

**Why I rejected this:** Defeats the point of stateless microservices. Every service would need to call Redis for every request.

---

### Option 2: JWT (JSON Web Tokens) ✓ **(My Choice)**

**What it is:** Stateless tokens containing user claims, signed by server.

**Pros:**
- **Stateless:** No server-side session storage
- **Microservices-friendly:** Each service validates independently
- **Contains claims:** User ID, role, tenant_id embedded
- **Mobile-friendly:** Easy to store and send in Authorization header
- **Scalability:** No session storage bottleneck

**Cons:**
- Can't immediately revoke before expiry (mitigated with short expiry + refresh tokens)
- Token size larger (~200 bytes vs 16 bytes)
- Must protect secret key

**Why I chose this:** Best fit for stateless microservices with mobile support

---

### Option 3: API Keys

**What it is:** Static keys assigned to each user.

**Pros:**
- Simple implementation
- Good for service-to-service auth

**Cons:**
- No expiration mechanism
- Doesn't carry user context (role, tenant)
- Not suitable for human authentication

---

## Decision Outcome

**My Choice:** JWT (JSON Web Tokens) with Refresh Token Pattern

### Why This Makes Sense

**1. Stateless Performance**
No session storage required. Services scale horizontally without shared state. Token validation is local (<1ms, no database/network call).

**2. Microservices Architecture**
Each service validates JWT independently using shared secret. No network call to auth service.

**3. Role Information for RBAC**
JWT payload contains everything needed:

```json
{
  "sub": "user-uuid-123",
  "tenant_id": "natwest-001",
  "role": "HELP_DESK_AGENT",
  "email": "agent@natwest.com",
  "exp": 1633024800,
  "iat": 1633021200
}
```

**4. Multi-Tenant Security**
The `tenant_id` claim is critical. Every query filters by this to ensure users only see their organisation's data.

**5. Mobile Support**
Apps store JWT in SecureStore, send in `Authorization: Bearer {token}` header. Simple.

---

## How It Works

**Token Types:**
- **Access Token:** 15 minutes, used for API requests
- **Refresh Token:** 7 days, used to get new access tokens

**Login Flow:**
1. User submits credentials
2. User Service validates and generates both tokens
3. Refresh token stored in database
4. Return both to client

**API Request:**
1. Client sends `Authorization: Bearer {access_token}`
2. API Gateway validates JWT signature
3. Extracts tenant_id and role
4. Forwards to service if valid
5. Client uses refresh token if expired

**JWT Implementation:**

```python
# Signing (User Service)
payload = {
    "sub": user.id,
    "tenant_id": user.tenant_id,
    "role": user.role,
    "email": user.email,
    "exp": datetime.utcnow() + timedelta(minutes=15),
    "iat": datetime.utcnow()
}
token = jwt.encode(payload, SECRET_KEY, algorithm="HS256")

# Validation (All Services)
try:
    payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    tenant_id = payload["tenant_id"]
    role = payload["role"]
except jwt.ExpiredSignatureError:
    raise Unauthorized("Token expired")
```

**RBAC Example:**

```python
@router.put("/complaints/{id}/assign")
@require_role(["HELP_DESK_AGENT", "MANAGER"])
async def assign_complaint(id: str, current_user: User = Depends(get_current_user)):
    # Only agents and managers can assign
    ...
```
---

## Security Considerations

**Token Storage:**
- Web: localStorage 
- Mobile: SecureStore/Keychain (encrypted)
- Never in cookies (avoids CSRF)

**Token Expiry:**
- Access Token: 15 min (limits damage if stolen)
- Refresh Token: 7 days (balance security and UX)

**Secret Key:**
- Dev: Environment variable
- Prod: AWS Secrets Manager 
- Rotation: Every 90 days with grace period

**HTTPS Only:**
All tokens over TLS 1.2+ (NFR4). HSTS headers enforce HTTPS.

**Token Revocation:**
- Refresh tokens: Stored in database, can be deleted
- Access tokens: Redis blacklist for compromised tokens
- Logout: Deletes refresh token, blacklists access token
- Worst case: 15-minute window if token stolen

---

## Consequences

**Positive **
- Fast authentication (no database lookup per request)
- Works seamlessly with mobile
- Each service validates independently
- Easy rate limiting per user

**Negative **
- Can't immediately revoke (15-min window)
- Must protect secret key carefully
- Client handles token refresh

**Mitigations:**
- Short 15-min expiry limits revocation window
- Key rotation every 90 days
- Monitor for suspicious patterns
- Rate limit refresh endpoint

---

## Compliance

- OWASP Top 10: A01:2021 Broken Access Control
- GDPR Article 32: Technical security measures

---

## Sources I Used

- Jones, M., Bradley, J., Sakimura, N. (2015). *JSON Web Token (JWT) RFC 7519*. IETF. https://tools.ietf.org/html/rfc7519
- OWASP. (2021). *JSON Web Token Cheat Sheet*. https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html
- Auth0. (2024). *JWT Handbook*. https://auth0.com/resources/ebooks/jwt-handbook
- Madden, N. (2020). *API Security in Action*. Manning Publications.
- Parecki, A. (2020). *OAuth 2.0 Simplified*. https://oauth2simplified.com/

---

## Related Decisions

- **ADR-001:** Microservices Architecture (requires stateless auth)
- **ADR-004:** Multi-Tenant Strategy (tenant_id in JWT)
- **ADR-005:** Technology Stack (Python libraries: PyJWT, python-jose)
