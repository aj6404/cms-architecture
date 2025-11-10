# ADR-005: Technology Stack Selection - Python FastAPI

---

**Status:** Accepted  
**Date:** 10 October 2025  
**Last Updated:** 10  november 2025  
**Author:** Adam James Brown  

---

## Context and Problem Statement

Now that I've decided on microservices (ADR-001), I need to pick the actual programming language and framework for building the backend services. This is a pretty important decision because it affects how fast I can develop, how well it performs, and whether the code will be maintainable.

I'm working solo with about 2 months to build a working proof-of-concept, so I need something that lets me move quickly without sacrificing quality.

**The main question:** What technology stack will let me build high-performance microservices fast enough for my POC timeline while still being good enough for production?

---

## Decision Drivers

### What I Need
- **Timeline:** I've got about 3 months total, and I'm working alone
- **NFR2 (Performance):** Need to support 500 concurrent users with <2s response time
- **NFR6 (Maintainability):** Code needs to be clear and well-structured for academic assessment
- **Learning curve:** I can't spend weeks learning a completely new language
- **Async support:** Need proper async/await for the event-driven stuff (ADR-002)
- **Type safety:** Want to catch errors before runtime
- **Community support:** Need good docs and help when I get stuck

---

## Options I Considered

### Option 1: Node.js + Express

**What it is:** JavaScript (or TypeScript) with the Express framework.

**Pros:**
- Same language for frontend and backend (React + Node.js)
- Really good async support - JavaScript is built for async
- Massive ecosystem (npm has everything)
- Fast to develop REST APIs
- Tons of tutorials and examples

**Cons:**
- Type safety is weak even with TypeScript
- Can still get "callback hell" if you're not careful
- Not as good for data-heavy operations
- TypeScript compilation adds extra steps
- Can still get runtime type errors

**My thoughts:** Node.js is solid and I've used it before. The single language thing is tempting. But for data-intensive stuff with databases and complex business logic, Python feels like a better fit.

---

### Option 2: Java Spring Boot

**What it is:** Enterprise Java with the Spring Boot framework.

**Pros:**
- Proper enterprise-grade framework
- Really strong typing with compile-time checking
- Excellent for big teams
- Mature ecosystem with solutions for everything
- Great IDE support
- Spring Cloud is built for microservices

**Cons:**
- SO MUCH boilerplate code
- Development is slower than Python
- Steeper learning curve
- Compilation takes forever
- Uses loads of memory
- Way overkill for a 2-month POC

**My thoughts:** Spring Boot is what big companies use, but it's designed for teams of 20+ developers. For a solo POC, it would slow me down massively. I'd spend more time fighting with configuration than building features.

---

### Option 3: Python FastAPI ✓ **(My Choice)**

**What it is:** Modern Python framework specifically designed for building APIs with automatic documentation.

**Pros:**
- **Fast development:** Python's syntax is concise - maybe 3-5x less code than Java
- **Async built-in:** Built on top of Starlette, handles async operations really well
- **Type safety:** Pydantic models give me runtime validation AND type hints
- **Auto documentation:** Swagger/OpenAPI docs generated automatically - massive time saver
- **Performance:** Surprisingly fast - comparable to Node.js according to benchmarks
- **Readable code:** Perfect for academic assessment - shows understanding without masses of boilerplate
- **Great ecosystem:** SQLAlchemy for database, Celery for background tasks, pytest for testing
- **Easy to learn:** Python is one of the easiest languages to pick up

**Cons:**
- Smaller in enterprises compared to Java (though it's growing fast)
- Global Interpreter Lock (GIL) can limit CPU parallelism (but I don't care - my services are I/O-bound)
- Ecosystem less mature than Spring Boot

**Why I chose this:** FastAPI hits the sweet spot. It's fast to develop with, performs well, and produces clean readable code. The automatic API documentation is brilliant - I get interactive API docs without writing any documentation code. Perfect for a POC that needs to work well AND be easy to understand.

---

### Option 4: Go (Golang)

**What it is:** Google's compiled language designed for concurrent systems.

**Pros:**
- Excellent performance
- Built-in concurrency with goroutines
- Compiles to a single binary (easy deployment)
- Fast compilation
- Increasingly popular for microservices

**Cons:**
- Less intuitive syntax than Python
- Smaller ecosystem
- More verbose error handling
- Steeper learning curve
- Would slow down POC development

**My thoughts:** Go is brilliant for production systems where performance is critical, but it would slow me down for rapid POC development. Maybe for a future version.

---

## My Decision

**I'm going with Python 3.11+ and FastAPI 0.104+.**

### Why This Makes Sense

1. **Development Speed:** Python's concise syntax means I can build features really quickly. FastAPI's automatic validation and documentation generation saves me hours of work.

2. **Performance (NFR2):** FastAPI benchmarks show it can handle 20,000-30,000 requests per second. I need 500 concurrent users. That's plenty of headroom.

3. **Type Safety:** Pydantic models give me runtime validation. If someone sends invalid data to my API, it gets caught automatically. Plus type hints help my IDE catch mistakes.

4. **Async Support:** FastAPI has first-class async/await support, which is essential for my event-driven architecture. Can handle database queries, RabbitMQ messages, and external APIs all asynchronously.

5. **Maintainability (NFR6):** Python code is really readable. For academic assessment, this is important - markers can understand what I'm doing without wading through boilerplate.

6. **Ecosystem:**
   - **SQLAlchemy:** Mature ORM for PostgreSQL, works great with multi-tenant patterns
   - **Celery:** For async background tasks in Notification Service
   - **Pika:** RabbitMQ client
   - **Pytest:** Best testing framework I've used
   - **Pydantic:** Perfect for validating request/response models

7. **Free Documentation:** FastAPI automatically generates interactive API documentation. I just browse to `/docs` and get a full Swagger UI. Brilliant for demonstration.

### How I'm Setting It Up

**Project structure:**
```
complaint-service/
├── app/
│   ├── api/
│   │   └── controllers/          # FastAPI routers (endpoints)
│   ├── domain/
│   │   └── entities/             # Domain models
│   ├── services/
│   │   └── complaint_service.py  # Business logic
│   ├── repositories/
│   │   └── complaint_repository.py  # Data access
│   └── main.py                   # FastAPI app
├── tests/
├── requirements.txt
└── Dockerfile
```

**Key libraries I'm using:**
```
fastapi==0.104.1          # Web framework
uvicorn==0.24.0           # ASGI server (runs FastAPI)
pydantic==2.4.2           # Data validation
sqlalchemy==2.0.23        # ORM for database
alembic==1.12.1           # Database migrations
celery==5.3.4             # Async background tasks
pika==1.3.2               # RabbitMQ client
redis==5.0.1              # Caching
pytest==7.4.3             # Testing
```

**Example of what FastAPI code looks like:**
```python
from fastapi import APIRouter, Depends
from pydantic import BaseModel

router = APIRouter()

class ComplaintCreateRequest(BaseModel):
    subject: str
    description: str
    category_id: str
    priority: PriorityEnum

@router.post("/complaints", status_code=201)
async def create_complaint(
    request: ComplaintCreateRequest,
    service: ComplaintService = Depends(get_complaint_service),
    tenant_id: str = Depends(get_tenant_from_jwt)
) -> ComplaintResponse:
    complaint = await service.create_complaint(request, tenant_id)
    return ComplaintResponse.from_entity(complaint)
```

**What's brilliant about this:**
- Type hints for IDE autocomplete
- Pydantic automatically validates the request
- Auto-generated API docs show this endpoint at `/docs`
- Async/await means this doesn't block while waiting for database
- Dependency injection keeps code testable

---

## Consequences

### What I Gain
- Really fast development - can build features in hours not days
- Clean, readable code perfect for academic assessment
- Automatic API documentation impresses in demonstrations
- Type checking catches errors early
- Easy to write comprehensive tests
- Low memory usage (good for Docker containers)
- Excellent async performance for event-driven architecture

### What I'm Dealing With
- Python's dynamic nature means some errors only show up at runtime
- GIL limits CPU parallelism (but this doesn't matter for my I/O-bound services)
- Smaller talent pool for enterprise hiring compared to Java
- Python packaging (pip/virtualenv) is less robust than Java's Maven/Gradle

### How I'm Managing It
- Using MyPy for static type checking in my CI pipeline
- Writing lots of unit tests (targeting 80%+ coverage)
- Pydantic catches data validation errors at the API boundary
- All my services are I/O-bound (database, message queues), so GIL doesn't matter
- Adding type hints and docstrings everywhere for maintainability

---

## Performance Validation

**FastAPI benchmarks (TechEmpower):**
- 20,000-30,000 requests/second on a single core
- Comparable to Node.js, much faster than Django or Flask
- Memory efficient: about 50MB per service

**My requirements:**
- 500 concurrent users
- <2s response time
- FastAPI easily exceeds this with loads of room to spare

**My testing plan:**
- Use Locust to simulate 500 concurrent users
- Target: 95% of requests under 500ms (well under my 2s requirement)
- Document results in Task 2

---

## Could I Use Different Tech for Different Services?

**Notification Service:** Could use Node.js (it's great for async), but keeping everything in Python reduces complexity.

**Reporting Service:** Could use specialized analytics tools like Apache Superset, but Python + Pandas is enough for the POC.

**My decision:** Stick with Python/FastAPI for everything. When you're working solo, reducing cognitive overhead is important. Don't want to be context-switching between languages.

---

## Is This Production-Ready?

Yes! FastAPI is used in production by:
- **Microsoft:** Several Azure services
- **Uber:** Internal APIs
- **Netflix:** Parts of their platform
- **Explosion AI:** spaCy API (the NLP library)

**For production deployment:**
- Use Gunicorn with Uvicorn workers
- Docker containerization (already doing this)
- Kubernetes or Docker Swarm for orchestration
- Built-in health check endpoints

---

## Trade-offs I'm Making

**Python vs Java:** Java's stronger typing would catch more errors at compile-time, but Python's faster development is more valuable for a POC. Can always rewrite critical services in Go/Java later if needed.

**Single Language vs Best-of-Breed:** Could use Node.js for Notification Service (great async) and Go for high-performance parts. But the cognitive overhead of switching languages isn't worth it for a solo POC.

**Framework Complexity:** FastAPI is simpler than Spring Boot but more opinionated than Flask. This is a good trade-off - it provides structure without being overwhelming.

---

## Sources I Used

- Ramírez, S. (2023). *FastAPI Documentation*. Retrieved from https://fastapi.tiangolo.com/
- TechEmpower. (2024). *Web Framework Benchmarks Round 22*. Retrieved from https://www.techempower.com/benchmarks/
- Percival, H., & Gregory, B. (2020). *Architecture Patterns with Python*. O'Reilly Media.
- Van Rossum, G., Levkivskyi, I. (2014). *PEP 484 - Type Hints*. Python.org.

---

## Related Decisions

These other ADRs connect to this one:
- **ADR-001:** Microservices Architecture (needs lightweight framework)
- **ADR-002:** Event-Driven Architecture (requires async/await support)
- **ADR-003:** CQRS Pattern (SQLAlchemy handles multiple database connections)
- **ADR-004:** Multi-Tenant Strategy (Python middleware sets PostgreSQL search_path)
