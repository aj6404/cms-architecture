# ADR-005: Technology Stack Selection - Python FastAPI

---

**Status:**  Validated  
**Date:** 10 October 2025  
**Last Updated:** 24 November 2025  
**Author:** Adam James Brown  

---

## Context and Problem Statement

Now that I've decided on microservices (ADR-001), I need to pick the actual programming language and framework for building the backend services. This is important because it affects how fast I can develop, how well it performs, and whether the code will be maintainable.

I'm working solo with about 3 months to build a working proof-of-concept, so I need something that lets me move quickly without sacrificing quality.

**The main question:** What technology stack will let me build high-performance microservices fast enough for my POC timeline while still being good enough for production?

---

## Decision Drivers

### What I Need

- **Timeline:** About 3 months total, working alone
- **Performance:** Need to handle a "large user base" (case study requirement) with good response times
- **Maintainability:** Code needs to be clear and well-structured for academic assessment
- **Learning curve:** Can't spend weeks learning a completely new language
- **Async support:** Need proper async/await for the event-driven stuff (ADR-002)
- **Type safety:** Want to catch errors before runtime
- **Multi-tenancy:** Data isolation between companies (case study requirement)

---

## Options I Considered

### Option 1: Node.js + Express

**Pros:**
- Same language for frontend and backend
- Really good async support
- Massive ecosystem
- Fast to develop REST APIs

**Cons:**
- Type safety is weak even with TypeScript
- Can get messy with nested callbacks
- Not as good for data-heavy operations

**My thoughts:** Node.js is solid but Python feels better for database work and complex business logic.

---

### Option 2: Java Spring Boot

**Pros:**
- Proper enterprise-grade framework
- Really strong typing with compile-time checking
- Mature ecosystem with solutions for everything
- Spring Cloud is built for microservices

**Cons:**
- Development is slower than Python
- Steep learning curve
- Way overkill for a solo POC

**My thoughts:** Spring Boot is what big companies use, but it's designed for large teams. For a solo POC, it would slow me down massively.

---

### Option 3: Python FastAPI ✓ **(My Choice)**

**Pros:**
- **Fast development:** Python's syntax is concise
- **Async built-in:** Built on Starlette, handles async operations really well
- **Type safety:** Pydantic models give me runtime validation AND type hints
- **Auto documentation:** Swagger/OpenAPI docs generated automatically - massive time saver
- **Performance:** Fast
- **Readable code:** Perfect for academic assessment
- **Great ecosystem:** SQLAlchemy for database, Pika for RabbitMQ, pytest for testing

**Cons:**
- Some errors only show up at runtime

**Why I chose this:** FastAPI works brilliantly for my POC. It's fast to develop with, performs well, and produces clean readable code. The automatic API documentation is brilliant - saves hours of work.

---

### Option 4: Go (Golang)

**Pros:**
- Excellent performance
- Built-in concurrency with goroutines
- Compiles to a single binary

**Cons:**
- Steeper learning curve
- Would slow down POC development

**My thoughts:** Go is brilliant for production systems but would slow me down for rapid POC development.

---

## My Decision

**I'm going with Python 3.11+ and FastAPI 0.104+.**

### Why This Makes Sense

1. **Development Speed:** Python's concise syntax means I can build features really quickly. FastAPI's automatic validation and documentation saves me hours of work.

2. **Performance:** FastAPI benchmarks show it can handle 20,000+ requests per second. More than enough for the case study's "large user base" requirement.

3. **Type Safety:** Pydantic models give me runtime validation. Invalid data gets caught automatically. Plus type hints help my IDE catch mistakes.

4. **Async Support:** FastAPI has first-class async/await support, essential for my event-driven architecture. Can handle database queries, RabbitMQ messages, and external APIs all asynchronously.

5. **Maintainability:** Python code is really readable. For academic assessment, this is important - markers can understand what I'm doing easily.

6. **Free Documentation:** FastAPI automatically generates interactive API documentation. Just browse to `/docs` and get a full Swagger UI. Brilliant for demonstrations.

### Project Structure
```
services/
├── user-service/
│   ├── app/
│   │   ├── api/          # FastAPI endpoints
│   │   ├── models/       # Database models
│   │   ├── services/     # Business logic
│   │   └── main.py       
│   └── Dockerfile
├── complaint-service/
│   └── app/...
└── notification-service/
    └── app/...
```

### Key Libraries
```
fastapi==0.104.1          # Web framework
uvicorn==0.24.0           # ASGI server
pydantic==2.4.2           # Data validation
sqlalchemy==2.0.23        # ORM for PostgreSQL
bcrypt==4.0.1             # Password hashing
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

**What's brilliant:**
- Type hints for IDE autocomplete
- Pydantic automatically validates the request
- Auto-generated docs show this at `/docs`
- Async/await means no blocking
- Dependency injection keeps code testable

---

## Validation Results

### What Actually Happened 

#### Development Speed
- Built User Service, Complaint Service, Notification Service in 5 weeks
- Python code is more concise than the amount of equivalent Java code

#### Performance
- Average response times: 100-200ms
- Health checks: <10ms
- Handles concurrent requests easily
- Memory usage: ~50MB per service

#### Type Safety
- Pydantic caught enum case mismatches (uppercase vs lowercase)
- Caught missing dependencies (email-validator)
- Caught UUID serialization issues before runtime
- Saved hours of debugging

#### Multi-Tenancy
- SQLAlchemy + PostgreSQL schemas work perfectly
- Data isolation validated (tenant_001 vs tenant_002)
- Case study requirement met

#### Documentation
- Interactive API docs at `/docs` for both services
- Zero manual documentation needed
- Brilliant for testing and demonstrations

### Problems I Hit (And Fixed)

1. **Bcrypt version conflict** - Fixed by pinning versions (30 mins)
2. **Enum case mismatch** - Changed Python enums to lowercase (10 mins)
3. **UUID serialization** - Manual string conversion needed (20 mins)
4. **Missing email-validator** - Added to requirements (5 mins)

**Total debugging time:** About 1 hour

Python's error messages were clear and easy to Google solutions.

---

## Consequences

### What I Gain 

- Really fast development - features built in hours not days
- Clean, readable code perfect for academic assessment
- Automatic API documentation for demonstrations
- Type checking catches errors early
- Easy to write tests with pytest
- Low memory usage (good for Docker)
- Excellent async performance for events
  
### What I'm Dealing With 

- Python's dynamic nature means some errors only show up at runtime
- Smaller talent pool for enterprise hiring compared to Java

### How I'm Managing It

- Pydantic catches data validation errors at the API boundary
- Writing comprehensive tests (targeting 80%+ coverage)
- All my services are I/O-bound (database, message queues), not CPU-bound
- Adding type hints and docstrings everywhere

---

## Trade-offs

**Python vs Java:** Java's stronger typing catches more errors at compile-time, but Python's faster development is more valuable for a POC. Can always rewrite critical services later if needed.

**Single Language:** Could use Node.js for Notification Service or Go for high-performance parts. But keeping everything in Python reduces cognitive overhead for solo development.

**Framework Choice:** FastAPI is simpler than Spring Boot but more structured than Flask. Good balance - provides structure without being overwhelming.

---

## Sources

- Ramírez, S. (2023). *FastAPI Documentation*. Retrieved from https://fastapi.tiangolo.com/
- TechEmpower. (2024). *Web Framework Benchmarks Round 22*. Retrieved from https://www.techempower.com/benchmarks/
- Percival, H., & Gregory, B. (2020). *Architecture Patterns with Python*. O'Reilly Media.

---

## Related Decisions

- **ADR-001:** Microservices Architecture (needs lightweight framework)
- **ADR-002:** Event-Driven Architecture (requires async/await support)
- **ADR-003:** CQRS Pattern (SQLAlchemy handles read/write separation)
- **ADR-004:** Multi-Tenant Strategy (PostgreSQL search_path implemented successfully)
