# ADR-006: Authentication Strategy - JWT with RBAC

---

**Status:** Accepted  
**Date:** 12 October 2025  
**Last Updated:** 18 December 2025  
**Author:** Adam James Brown  

---

## Context and Problem Statement

The CMS has different user roles (consumers, agents, managers) with different permissions, accessed via web and mobile. With microservices architecture (ADR-001), each service needs to know who the user is and what they're allowed to do without calling a central auth server every time.

**The big question:** What authentication mechanism gives me secure, stateless authentication that works with microservices?

---

## Decision Drivers

- **NFR4 (Security):** Secure authentication with encrypted credentials
- **Microservices:** Auth must work across services without shared state
- **RBAC:** Must carry role information for authorization
- **Multi-tenant:** Must include tenant_id to prevent cross-tenant access
- **Performance:** Can't add significant latency
- **Mobile:** Has to work with mobile apps

---

## Options I Considered

### Option 1: Session-Based Authentication (Cookies)

**Pros:** Familiar pattern, instant session invalidation  
**Cons:** Stateful (requires shared Redis), scalability bottleneck, microservices anti-pattern, painful on mobile  
**My thoughts:** Defeats the point of stateless microservices

### Option 2: JWT (JSON Web Tokens) ✓ **(My Choice)**

**Pros:** Stateless, microservices-friendly, contains user claims (role, tenant_id), mobile-friendly, scales horizontally  
**Cons:** Can't immediately revoke before expiry, larger token size  
**Why I chose this:** Perfect for stateless microservices with mobile support

### Option 3: API Keys

**Pros:** Simple implementation  
**Cons:** No expiration, doesn't carry user context, not suitable for human authentication  
**My thoughts:** Good for service-to-service, not for users

---

## My Decision

**JWT (JSON Web Tokens) with Refresh Token Pattern**

### Why This Makes Sense

**1. Stateless Performance:** No session storage. Services scale horizontally without shared state. Token validation is local (under 1ms, no database call).

**2. Microservices Architecture:** Each service validates JWT independently using shared secret. No network calls to auth service.

**3. Role Information for RBAC:** JWT payload contains everything needed:
```json
{
  "sub": "user-uuid",
  "tenant_id": "tenant_001",
  "role": "agent",
  "email": "agent@barclays.com",
  "exp": 1633024800
}
```

**4. Multi-Tenant Security:** The `tenant_id` claim is critical - every query filters by this to ensure users only see their organisation's data.

---

## How It Works

**Token Types:**
- **Access Token:** 30 minutes, used for API requests
- **Refresh Token:** 7 days, used to get new access tokens

**Login Flow:**
1. User submits credentials
2. User Service validates and generates both tokens
3. Returns both to client

**API Request:**
1. Client sends `Authorization: Bearer {access_token}`
2. Service validates JWT signature
3. Extracts tenant_id and role
4. Forwards request if valid

**Implementation:**
```python
# Signing tokens (User Service)
payload = {
    "sub": user.id,
    "tenant_id": user.tenant_id,
    "role": user.role,
    "email": user.email,
    "exp": datetime.utcnow() + timedelta(minutes=30)
}
token = jwt.encode(payload, SECRET_KEY, algorithm="HS256")

# Validation (All Services)
try:
    payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    tenant_id = payload["tenant_id"]
    role = payload["role"]
except jwt.ExpiredSignatureError:
    raise HTTPException(401, "Token expired")

# RBAC decorator
def require_role(*allowed_roles):
    def role_checker(current_user: dict = Depends(get_current_user)):
        if current_user["role"] not in allowed_roles:
            raise HTTPException(403, "Access denied")
        return current_user
    return role_checker

# Usage
@router.patch("/{id}/assign")
async def assign_complaint(
    current_user: dict = Depends(require_role("agent", "manager"))
):
    # Only agents and managers can assign
    ...
```

---

## Security Considerations

**Token Storage:**
- Web: localStorage
- Mobile: SecureStore (encrypted)

**Token Expiry:**
- Access Token: 30 min (limits damage if stolen)
- Refresh Token: 7 days (balance security and UX)

**Secret Key:**
- Stored in environment variables
- Would use AWS Secrets Manager in production

**HTTPS Only:**
All tokens sent over HTTPS to prevent interception.

**Token Revocation:**
Access tokens can't be revoked before expiry, but 30-minute window is acceptable trade-off for stateless architecture.

---

## What I Actually Built for POC

Implemented:
- JWT generation in User Service using PyJWT
- Access tokens with 30-minute expiry
- Refresh tokens with 7-day expiry
- Token validation middleware in both services
- RBAC decorator for role-based endpoint protection
- tenant_id claim enforcement for multi-tenant isolation

Simplified for POC:
- No refresh token database storage (would add in production)
- No token blacklist for revocation
- Secret key in .env file (would use Secrets Manager in production)

---

## Consequences

**What I Gain:**
- Fast authentication (no database lookup per request)
- Works seamlessly with microservices
- Each service validates independently
- Easy role-based access control

**What I'm Dealing With:**
- Can't immediately revoke access tokens (30-min window)
- Must protect secret key carefully
- Client handles token refresh logic

**Mitigations:**
- Short 30-min expiry limits revocation window
- Environment variable for secret key
- Clear error messages for expired tokens

---

## Reflection After Implementation

JWT authentication worked really well for the POC. The stateless nature means each service can validate tokens independently without hitting the User Service or a database, which keeps response times fast.

The implementation was straightforward using PyJWT. Creating the dependency that extracts and validates the token took about an hour to get right, but once working, adding authentication to new endpoints is literally one line of code: `current_user: dict = Depends(get_current_user)`.

The RBAC decorator pattern works brilliantly. I can protect endpoints with `@require_role("agent", "manager")` and it automatically returns 403 if the user doesn't have the right role. This caught several test cases where I accidentally tried to access agent endpoints as a consumer.

The tenant_id in the JWT is critical - it's how I enforce multi-tenant isolation. Every database query uses this to filter data. I validated this by logging in as tenant_001 and tenant_002 users and confirming they can't see each other's data.

One thing I simplified for the POC is refresh token management. In production, you'd store refresh tokens in the database so you can revoke them, but for POC purposes, just having them work is sufficient. The access token expiry of 30 minutes means users aren't constantly re-authenticating, which is good UX.

The 401 Unauthorized errors when tokens expire are handled nicely in the frontend - it redirects to login automatically. I tested this by manually expiring a token and seeing the frontend handle it gracefully.

If I had to do it again, I wouldn't change much. Maybe I'd implement the refresh token endpoint earlier in development rather than as an afterthought, but the core JWT + RBAC pattern is solid. It's a well-proven approach that works exactly as expected.

---

## Related Decisions

- **ADR-001:** Microservices Architecture (requires stateless auth)
- **ADR-004:** Multi-Tenant Strategy (tenant_id in JWT critical for isolation)
- **ADR-005:** Technology Stack (PyJWT library)
