# ADR-005: Technology Stack Selection - Python FastAPI

---

**Status:** Validated  
**Date:** 10 October 2025  
**Last Updated:** 18 December 2025  
**Author:** Adam James Brown  

---

## Context and Problem Statement

After deciding on microservices (ADR-001), I need to pick the actual programming language and framework for building the backend services. I'm working solo with about 2-3 months to build a working proof-of-concept, so I need something that lets me move quickly without sacrificing quality.

**The main question:** What technology stack will let me build high-performance microservices fast enough for my POC timeline while still being production-quality?

---

## Decision Drivers

- **Timeline:** 2-3 months total, working alone
- **Performance:** Need good response times for high user load
- **Async support:** Essential for event-driven architecture (ADR-002)
- **Type safety:** Want to catch errors before runtime
- **Learning curve:** Can't spend weeks learning something completely new
- **Code readability:** Important for academic assessment

---

## Options I Considered

### Option 1: Node.js + Express

**Pros:** Great async support, massive ecosystem, fast development  
**Cons:** Weak type safety even with TypeScript, not ideal for data-heavy operations  
**My thoughts:** Solid choice but Python feels better for database work

### Option 2: Java Spring Boot

**Pros:** Enterprise-grade, strong typing, mature ecosystem  
**Cons:** Slower development, steep learning curve, overkill for solo POC  
**My thoughts:** Designed for large teams, would slow me down massively

### Option 3: Python FastAPI ✓ **(My Choice)**

**Pros:** Fast development, built-in async, runtime validation with Pydantic, auto-generated docs, readable code  
**Cons:** Some errors only at runtime (not compile-time)  
**Why I chose this:** Perfect balance of speed, performance, and readability for POC development

### Option 4: Go (Golang)

**Pros:** Excellent performance, built-in concurrency  
**Cons:** Steeper learning curve, would slow POC development  
**My thoughts:** Great for production but not for rapid prototyping

---

## My Decision

**Python 3.11+ and FastAPI 0.104+**

### Why This Makes Sense

1. **Development Speed:** Python's concise syntax + FastAPI's automatic validation saves hours
2. **Performance:** Benchmarks show 20,000+ requests/second - more than enough
3. **Type Safety:** Pydantic models give runtime validation, type hints catch IDE mistakes
4. **Async Support:** First-class async/await for event-driven architecture
5. **Free Documentation:** Auto-generated interactive API docs at `/docs` - brilliant for demos

### Key Libraries
```python
fastapi==0.104.1          # Web framework
uvicorn==0.24.0           # ASGI server
pydantic==2.4.2           # Data validation
sqlalchemy==2.0.23        # ORM for PostgreSQL
pyjwt==2.8.0              # JWT tokens
pika==1.3.2               # RabbitMQ client
```

### Code Example
```python
@router.post("/complaints", status_code=201)
async def create_complaint(
    request: ComplaintCreate,
    current_user: dict = Depends(get_current_user)
) -> ComplaintResponse:
    complaint = await service.create_complaint(request, current_user)
    return ComplaintResponse.from_entity(complaint)
```

**What's good:**
- Type hints for IDE autocomplete
- Pydantic automatically validates the request
- Auto-generated docs show this at `/docs`
- Async/await means no blocking
- Dependency injection keeps code testable

---

## Validation Results (What Actually Happened)

### Development Speed
Built User Service, Complaint Service, and Notification Service in about 5 weeks. Python code is way more concise than equivalent Java would have been.

### Performance
- Average response times: 100-200ms
- Health checks: under 10ms
- Handles concurrent requests easily
- Memory usage: ~50MB per service

### Type Safety
Pydantic caught several bugs before they became runtime problems:
- Enum case mismatches (uppercase vs lowercase)
- Missing dependencies (email-validator)
- Invalid data formats

Saved hours of debugging.

### Multi-Tenancy
SQLAlchemy + PostgreSQL schemas work perfectly. Data isolation validated between tenant_001 and tenant_002 - case study requirement met.

### Documentation
Interactive API docs at `/docs` for all services. Zero manual documentation needed. Absolutely brilliant for testing and demonstrations.

---

## Problems I Hit (And Fixed)

1. **Bcrypt version conflict** - Pinned versions (30 mins)
2. **Enum case mismatch** - Changed Python enums to lowercase (10 mins)
3. **UUID serialization** - Had to manually convert UUIDs to strings (20 mins)
4. **Missing email-validator** - Added to requirements (5 mins)

**Total debugging time:** About 1 hour

Python's error messages were clear and easy to Google solutions for.

---

## Consequences

**What I Gain:**
- Really fast development - features built in hours not days
- Clean, readable code perfect for academic assessment
- Automatic API documentation
- Type checking catches errors early
- Easy testing with pytest
- Good async performance for events

**What I'm Dealing With:**
- Dynamic typing means some errors only show at runtime
- Smaller enterprise talent pool compared to Java

**How I'm Managing It:**
- Pydantic catches validation errors at API boundary
- Comprehensive type hints throughout
- All services are I/O-bound (database, queues) not CPU-bound, so Python's performance is fine

---

## Reflection

Looking back, FastAPI was definitely the right choice. I built three working microservices in 5 weeks, which would have taken way longer with Spring Boot. The automatic API documentation saved me hours - just point assessors to `/docs` and they can see and test every endpoint interactively.

The type hints with Pydantic caught a bunch of bugs early. The UUID serialization issue was annoying, but once I understood it, the fix was straightforward. Python's clear error messages made debugging much easier than I expected.

If I had to do it again, I wouldn't change this decision. FastAPI strikes the perfect balance between "quick to develop" and "production quality." The code is readable enough for academic assessment while being performant enough for real use.

The only thing I'd do differently is add more comprehensive type hints from the start - I added them incrementally, but having them from day one would have caught a couple more issues earlier.

---

## Related Decisions

- **ADR-001:** Microservices Architecture (needs lightweight framework)
- **ADR-002:** Event-Driven Architecture (requires async/await)
- **ADR-003:** CQRS Pattern (SQLAlchemy handles separation)
- **ADR-004:** Multi-Tenant Strategy (PostgreSQL search_path works perfectly)
