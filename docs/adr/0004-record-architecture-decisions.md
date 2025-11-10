# ADR-004: Multi-Tenant Data Isolation Strategy

---

**Status:** Accepted  
**Date:** 8 October 2025  
**Last Updated:** 10 november 2025  
**Author:** Adam James Brown  

---

## Context and Problem Statement

The CMS needs to serve multiple organisations - think NatWest, HSBC, Vodafone, O2 - and their data needs to be completely isolated. The case study is really clear on this: "Natwest helpdesk data should not be visible to Barclays and vice versa." Each organisation could have 100,000+ complaints, so this isn't a small-scale problem.

The challenge is figuring out how to structure the database so that:
1. There's absolutely no chance of data leaking between organisations
2. It performs well
3. It doesn't cost a fortune to run

**The main question:** How do I architect the database to guarantee complete tenant isolation while keeping it affordable and fast?

---

## Decision Drivers

### What I Need
- **NFR4 (Security):** Absolute data isolation - no possibility of one organisation seeing another's data
- **NFR5 (Scalability):** Can onboard new organisations without changing code
- **Compliance:** GDPR, PCI-DSS (for banking clients), SOC 2 - these aren't optional for banks
- **Cost:** This needs to be viable as a SaaS model - can't spend thousands per customer
- **Performance:** Queries still need to be fast despite the multi-tenant setup
- Some organisations might have data residency requirements (data must stay in UK vs EU)

---

## Options I Considered

### Option 1: Separate Database Per Tenant

**What it is:** Every organisation gets their own PostgreSQL database instance.

**Pros:**
- **Maximum isolation:** Complete database separation - literally impossible for data to leak
- **Performance:** No overhead from filtering by tenant
- **Compliance:** Easy to meet data residency rules (just deploy databases in the right region)
- **Disaster recovery:** Can restore NatWest's data without affecting HSBC
- **Simple queries:** Don't need to filter by tenant_id in every query

**Cons:**
- **Stupidly expensive:** Each PostgreSQL instance costs £50-200/month. With 100 tenants, that's £5,000-20,000/month!
- **Management nightmare:** Backing up, monitoring, and updating 100+ databases
- **Schema hell:** Every time I change the schema, I have to apply it to 100 databases
- **Wasteful:** A small organisation with 50 complaints doesn't need a whole database
- **Doesn't scale:** Can't realistically manage 1000+ databases

**My thoughts:** This would be great for huge enterprise clients who pay loads of money, but it's completely impractical for a SaaS model. The costs and operational complexity are just too high.

---

### Option 2: Shared Database with tenant_id Column

**What it is:** One database, every table has a `tenant_id` column, and I filter everything by tenant.

**Pros:**
- **Cheap:** Just one database for everyone
- **Simple infrastructure:** Easy to manage and backup
- **Efficient:** Shared resources across all tenants
- **Easy schema changes:** One migration applies to everyone

**Cons:**
- **Massive security risk:** One bug in my WHERE clause and suddenly NatWest can see HSBC's data
- **Performance issues:** Indexes have to include tenant_id, making them bigger
- **No physical separation:** All data is in the same tables
- **Query complexity:** Every single query needs `WHERE tenant_id = ?` - miss it once and you've leaked data
- **Compliance nightmare:** Auditors for banks won't be happy with this

**My thoughts:** This is what cheap SaaS apps use, but for banking and telecom? No way. The security risk is too high. One mistake in my code and I've exposed confidential complaint data across organisations. Not acceptable.

---

### Option 3: PostgreSQL Schema-Based Multi-Tenancy ✓ **(My Choice)**

**What it is:** One database, but each organisation gets their own PostgreSQL schema (like `tenant_001`, `tenant_002`). Each schema has a complete set of tables.

**Pros:**
- **Strong isolation:** Schemas provide proper namespace separation
- **Good security:** PostgreSQL enforces schema boundaries at the database level
- **Cost-effective:** One database, shared infrastructure
- **Better performance:** No tenant_id filtering - queries just use the schema's tables
- **Compliance-friendly:** Physical separation satisfies most auditors
- **Flexible backups:** Can backup just one schema if needed
- **Extra security layer:** Can add PostgreSQL Row-Level Security on top as defense-in-depth

**Cons:**
- **Lots of schemas:** 100 tenants = 100 schemas with duplicate table structures
- **Migration complexity:** Have to apply schema changes to all tenant schemas
- **Connection setup:** Must set the search_path on every database connection
- **Some shared stuff:** Database-level resources (connections, memory) are still shared

**Why I chose this:** This is the sweet spot. It gives me proper isolation that auditors will accept, performs well, and doesn't cost a fortune. It's how a lot of real SaaS companies do multi-tenancy. The migration complexity is manageable with scripts, and the security is way better than shared tables with tenant_id.

---

### Option 4: Hybrid Approach

**What it is:** Big enterprise customers get their own database, smaller customers share schema-based multi-tenancy.

**Pros:**
- Flexibility for different customer tiers
- VIP customers get dedicated resources

**Cons:**
- Way more complex architecture
- Code has to support both patterns
- Operational nightmare

**My thoughts:** This would be a good optimization for the future, but it's too complex for a POC. Start with schema-based for everyone, then if we land a massive client who wants to pay for dedicated infrastructure, we can add that option later.

---

## My Decision

**I'm going with PostgreSQL Schema-Based Multi-Tenancy.**

### Why This Makes Sense

1. **Security (NFR4):** Schemas provide a real security boundary. Even if I have a SQL injection vulnerability (hopefully not!), an attacker can't cross schema boundaries without database admin privileges.

2. **Performance:** Queries run against schema-specific tables without needing to filter by tenant_id everywhere. Indexes are smaller and faster.

3. **Compliance:** When auditors ask "show me how NatWest's data is isolated from HSBC," I can point them to separate schemas. This satisfies banking regulations.

4. **Cost-Effective:** One database instance (£100-200/month) serves all tenants. Can scale it up as we grow rather than paying per tenant.

5. **Scalability (NFR5):** Onboarding a new organisation is just creating a new schema from a template. Can automate this completely.

### How I'm Implementing It

**Database Setup:**
```sql
-- Template schema with all the table definitions
CREATE SCHEMA template;

-- Create all tables in the template
CREATE TABLE template.complaints (...);
CREATE TABLE template.users (...);
-- ... all other tables

-- When NatWest signs up:
CREATE SCHEMA tenant_001;

-- Copy table structure from template
CREATE TABLE tenant_001.complaints (LIKE template.complaints INCLUDING ALL);
CREATE TABLE tenant_001.users (LIKE template.users INCLUDING ALL);
-- ... etc
```

**How I connect it to the right tenant:**
```python
# FastAPI middleware that sets the schema based on JWT
@app.middleware("http")
async def set_tenant_schema(request: Request, call_next):
    # Get tenant_id from the JWT token
    tenant_id = extract_tenant_from_jwt(request)
    schema_name = f"tenant_{tenant_id}"
    
    # Tell PostgreSQL to use this schema
    await db.execute(f"SET search_path TO {schema_name}")
    
    response = await call_next(request)
    return response
```

**Organisation table (in the public schema):**
```sql
-- Public schema for cross-tenant data like organization list
CREATE SCHEMA public;
CREATE TABLE public.organizations (
    id UUID PRIMARY KEY,
    name VARCHAR(255),
    schema_name VARCHAR(50) UNIQUE, -- e.g., 'tenant_001'
    created_at TIMESTAMP
);
```

**Security layers:**
1. **Application level:** JWT token contains tenant_id, I validate it on every request
2. **Database level:** PostgreSQL search_path ensures queries only see the right schema
3. **Row-Level Security:** Added as extra protection in case search_path isn't set properly
4. **Audit logging:** Log any attempts to access cross-schema data

---

## What I'm Actually Building for POC

For the proof-of-concept, I'm implementing:
- Two sample tenant schemas (tenant_001 for "NatWest", tenant_002 for "HSBC")
- The middleware that sets search_path based on JWT
- Seed data in both schemas to demonstrate isolation

This proves the isolation concept without needing to build the full onboarding system.

---

## Consequences

### What I Gain
- Strong data isolation that banking clients will accept
- Fast queries without tenant_id filtering everywhere
- Easy to demonstrate compliance ("here's NatWest's schema, here's HSBC's - they're completely separate")
- Can backup or restore individual tenant data if needed

### What I'm Dealing With
- Managing 100 schemas if I get to 100 customers
- Database migrations have to run on all tenant schemas (could take time)
- Have to be really careful about setting search_path correctly
- At 1000+ schemas, might start hitting PostgreSQL metadata performance limits

### How I'm Managing It
- Writing scripts to automate schema creation and migrations
- Using connection pooling with tenant-specific pools
- Adding monitoring to catch accidental cross-schema queries
- Regular audits to make sure search_path is being set correctly
- Planning to shard across multiple databases if I ever hit 500+ tenants

---

## Onboarding a New Tenant

**When NatWest signs up:**
1. Admin creates the organisation in the CMS
2. System generates schema name: `tenant_001`
3. Automated script creates the schema and copies tables from template
4. Admin creates the initial user accounts in `tenant_001.users`
5. NatWest is live!

**Database Migrations:**
```bash
# Script to apply a migration to all tenant schemas
for schema in $(psql -tc "SELECT schema_name FROM public.organizations"); do
    psql -c "SET search_path TO $schema; ALTER TABLE complaints ADD COLUMN sla_deadline TIMESTAMP;"
done
```

---

## How This Meets Compliance Requirements

This approach satisfies:
- **GDPR Article 32:** "Appropriate technical measures" for data protection ✓
- **PCI-DSS Requirement 3:** Data isolation for sensitive information ✓
- **ISO 27001:** Logical separation of data ✓
- **SOC 2 Type II:** Access control and data segregation ✓

**For auditors:**
- "Show me NatWest's data" → It's in schema tenant_001
- "How do you prevent HSBC from seeing NatWest's data?" → Different schemas, database-level enforcement
- "Can you backup just NatWest?" → Yes, schema-level backup
- "Show me access logs" → Here's every query that touched tenant_001

---

## Trade-offs I'm Making

**Flexibility vs Simplicity:** Schema-based is more complex than shared tables, but the security and compliance benefits are worth it. For banking clients, this isn't negotiable.

**Cost vs Isolation:** It's more expensive than pure shared tables but way cheaper than separate databases. Finding the right balance for SaaS economics.

**Operational Overhead vs Security:** Yes, managing multiple schemas is more work. But accidentally leaking customer data would be catastrophic. The overhead is worth it.

---

## Sources I Used

- Chong, F., Carraro, G., & Wolter, R. (2006). *Multi-Tenant Data Architecture*. Microsoft Architecture Journal. Retrieved from https://learn.microsoft.com/en-us/previous-versions/msp-n-p/aa479086(v=msdn.10)
- PostgreSQL Documentation. (2024). *Schema Usage Patterns*. Retrieved from https://www.postgresql.org/docs/current/ddl-schemas.html
- OWASP. (2021). *Multi-Tenancy Cheat Sheet*. Retrieved from https://cheatsheetseries.owasp.org/

---

## Related Decisions

These other ADRs connect to this one:
- **ADR-001:** Microservices Architecture (establishes the overall design)
- **ADR-003:** CQRS Pattern (both read and write databases use schema-based isolation)
- **ADR-005:** Technology Stack Selection (explains why PostgreSQL - its schema support is key)
- **ADR-006:** Authentication Strategy (JWT contains tenant_id claim that drives schema selection)
