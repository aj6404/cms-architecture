# ADR-004: Multi-Tenant Data Isolation Strategy

---

**Status:** Accepted  
**Date:** 8 October 2025  
**Last Updated:** 18 December 2025  
**Author:** Adam James Brown  

---

## Context and Problem Statement

The CMS serves multiple organisations (banks, telecoms) and their data must be completely isolated. Each organisation could have 100,000+ complaints. The challenge is guaranteeing complete tenant isolation while keeping it affordable and fast.

**The main question:** How do I architect the database to guarantee complete tenant isolation while keeping it effecient?

---

## Decision Drivers

- **NFR4 (Security):** Absolute data isolation - no possibility of cross-tenant data access
- **NFR5 (Scalability):** Can onboard new organisations without code changes
- **Compliance:** GDPR, PCI-DSS for banking clients
- **Cost:** Needs to be viable as a SaaS model
- **Performance:** Queries must remain fast

---

## Options I Considered

### Option 1: Separate Database Per Tenant

**Pros:** Maximum isolation, simple queries, easy compliance  
**Cons:** in real terms this would be quite expensive and hard to manage   
**My thoughts:** Too complex for SaaS

### Option 2: Shared Database with tenant_id Column

**Pros:**  simple to build 
**Cons:** Massive security risk - one missed WHERE clause leaks data, compliance nightmare  
**My thoughts:** Too risky for banking clients

### Option 3: PostgreSQL Schema-Based Multi-Tenancy ✓ **(My Choice)**

**Pros:** Strong isolation, cost-effective, good performance, compliance-friendly  
**Cons:** Managing multiple schemas, migration complexity  
**Why I chose this:** Best balance of security and performance

---

## My Decision

**PostgreSQL Schema-Based Multi-Tenancy** - each organisation gets their own schema (tenant_001, tenant_002) with a complete set of tables.

### Why This Makes Sense

1. **Security:** PostgreSQL enforces schema boundaries at database level
2. **Performance:** No tenant_id filtering needed, smaller indexes
3. **Compliance:** Separate schemas satisfy banking auditors
5. **Scalable:** Onboarding = creating new schema from template

### Implementation

**Database Setup:**
```sql
-- Create tenant schema
CREATE SCHEMA tenant_001;
CREATE TABLE tenant_001.complaints (LIKE template.complaints INCLUDING ALL);
CREATE TABLE tenant_001.users (LIKE template.users INCLUDING ALL);
```

**Setting schema from JWT:**
```python
def get_tenant_db(request: Request, current_user: dict = Depends(get_current_user), db: Session = Depends(get_db)):
    tenant_id = current_user["tenant_id"]
    db.execute(text(f"SET search_path TO {tenant_id}, public"))
    return db
```

**Security layers:**
1. JWT contains tenant_id
2. PostgreSQL search_path ensures queries only see correct schema
3. Audit logging tracks tenant context

---

## What I Built for POC

Implemented:
- Two tenant schemas (tenant_001 for Barclays, tenant_002 for O2)
- JWT tokens with tenant_id claim
- Database dependency that sets search_path
- Seed data in both schemas
- Validated tenant_001 agent cannot see tenant_002 data

---

## Consequences

**What I Gain:**
- Strong isolation banking clients accept
- Fast queries without filtering
- Easy compliance demonstration

**What I'm Dealing With:**
- Managing multiple schemas
- Migrations run on all schemas
- Must set search_path correctly

**How I'm Managing It:**
- Automated schema creation scripts
- Centralized search_path management
- Logging to track tenant context

---

## Reflection After Implementation

This worked really well. The schema isolation does exactly what I expected - tenant_001 agents can't see tenant_002 data, which I validated through testing.

Setting search_path in the database dependency is reliable. I did spend time debugging why queries failed once, only to realize I forgot to set search_path in one method that's exactly why centralizing it matters.

Testing is much easier with schema isolation. I can create test data in tenant_001 without worrying about tenant_002 pollution. Queries are fast with no tenant_id filtering overhead.

The seed data setup was tedious - manually creating tables in both schemas. In production this needs automation with an onboarding script.

If I had to do it again, I wouldn't change much. Maybe write the schema creation script earlier for quicker testing. But the core decision with PostgreSQL schemas for multi-tenancy was definitely right. It gives strong isolation that's easy to explain and validate.

The critical thing is ensuring search_path gets set on every connection. I added logging showing tenant context per request, which makes it obvious if something's wrong.

---

## Related Decisions

- **ADR-001:** Microservices Architecture
- **ADR-003:** CQRS Pattern (both databases use schema isolation)
- **ADR-005:** Technology Stack Selection (PostgreSQL's schema support)
