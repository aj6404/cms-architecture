
# ADR-001: Adoption of Microservices Architecture with Event-Driven Patterns

---

**Status:** Accepted  
**Date:** 29 September 2025  
**Last Updated:** 10 november 2025  
**Author:** Adam James Brown  

---

## Context and Problem Statement

I'm designing a Complaint Management System (CMS) that needs to serve multiple large organisations (like banks and telecoms) in a multi-tenant environment. The system could potentially handle millions of users, so I need to make some important architectural decisions early on.

**The key requirements are:**
- Multiple access channels (web, mobile, phone)
- Real-time status updates and notifications
- Complex workflows with different user roles (consumers, agents, support staff, managers)
- Performance monitoring and analytics
- Future extensibility (e.g., chatbot integration later)
- 24/7 availability with 99.5% uptime

**The main question:** What architectural style is going to work best for these requirements while still being realistic for a 2-month proof-of-concept?

---

## Decision Drivers

### What the System Needs to Do (Functional Requirements)
- Keep each organisation's data completely isolated (multi-tenancy)
- Send notifications asynchronously (email and SMS)
- Provide real-time status updates to customers
- Generate reports and analytics for managers
- Enforce role-based access control across multiple organisations

### Performance & Quality Requirements (Non-Functional)
- **Scalability:** Need to support 20M+ users (I'm basing this on Barclays' customer numbers from their 2024 investor update)
- **Performance:** Response times under 2 seconds for 95% of requests
- **Reliability:** 99.5% uptime (excluding planned maintenance)
- **Security:** Everything encrypted, strict tenant isolation, role-based access
- **Accessibility:** WCAG 2.1 AA compliance for the UI

### My Constraints
- I'm working solo with a 2-month timeline
- Need to demonstrate a clear "golden thread" from architecture diagrams to actual implementation
- Have to use contemporary patterns (not just basic stuff from lectures)
- The POC needs to implement at least 2 functional requirements plus 1 non-functional requirement

---

## Options I Considered

### Option 1: Monolithic Architecture
**What it is:** Everything in one application with a layered architecture.

**Pros:**
- Much simpler to build initially
- Easier to debug (it's all in one codebase)
- No network delays between components
- Faster to prototype
- Less infrastructure to manage

**Cons:**
- Everything's tightly coupled - changing one thing affects everything else
- Can't scale individual components independently
- If one part breaks, the whole system could go down
- Hard to use new technologies later
- Deployment gets slower as the system grows

**My thoughts:** This would definitely be quicker to build, but my lecturer pointed out that "a good monolith beats a poor microservices." However, given the multi-tenant requirements and need to scale different parts independently, I don't think a monolith is the right choice. Also, I want to demonstrate understanding of more advanced patterns.

---

### Option 2: Microservices Architecture ✓ **(My Choice)**
**What it is:** Split the system into separate services based on business capabilities (complaints, users, notifications, etc.).

**Pros:**
- Can scale services independently (e.g., notification service separately from complaints)
- If one service fails, others keep working
- Can deploy updates to individual services without touching everything else
- Demonstrates modern industry practices
- Different services could use different technologies in future (though I'm sticking with Python for the POC)

**Cons:**
- More complex to set up and manage (though Docker Compose helps with this)
- Network latency between services (but my projected load is low so this should be fine)
- Distributed transactions are tricky
- Debugging is harder when issues span multiple services
- Takes about 40% longer to develop than a monolith

**Why I chose this:** This aligns best with the requirements. The multi-tenant isolation and independent scaling needs make microservices a good fit. Plus, it lets me demonstrate event-driven patterns and CQRS, which goes beyond what we covered in lectures. The lecturer's guidance about producing "consistent artifacts" matters more than just picking something trendy - and I can justify microservices properly for this use case.

---

### Option 3: Service-Oriented Architecture (SOA)
**What it is:** Uses an Enterprise Service Bus (ESB) to connect coarse-grained services.

**Pros:**
- Well-established pattern with lots of documentation
- Good for enterprise integration scenarios
- Supports complex orchestration

**Cons:**
- The ESB becomes a single point of failure
- Way more complex than I need
- Feels a bit old-fashioned compared to modern approaches

**My thoughts:** This is overkill for my project. A modern API Gateway (Kong) gives me the benefits without the ESB bottleneck.

---

### Option 4: Serverless Architecture
**What it is:** Using AWS Lambda or Azure Functions (Function-as-a-Service).

**Pros:**
- Scales automatically
- Only pay for what you use
- No servers to manage
- Perfect for event-driven stuff

**Cons:**
- Gets locked into a specific cloud provider
- Cold start delays can be annoying
- Hard to develop and test locally
- Would need paid cloud services to demonstrate

**My thoughts:** This would actually work quite well architecturally, but I need to run the POC locally for university submission. Docker Compose gives me similar benefits (containers, orchestration) without needing AWS access.

---

## My Decision

**I'm going with Microservices Architecture with Event-Driven Patterns.**

### Why This Makes Sense

1. **Multi-Tenant Isolation:** Each service can enforce tenant boundaries independently. This works naturally with my schema-based database isolation approach (see ADR-004).

2. **Independent Scaling:** Based on my calculations (360,000 complaints/year from 20M users), the notification service will need to handle way more traffic than the complaint service. With microservices, I can scale them separately.

3. **Demonstrates Advanced Understanding:** This lets me showcase event-driven patterns (using RabbitMQ), CQRS for reporting, and proper service decomposition. That's what I need for a first-class mark.

4. **Real-World Practice:** Companies like Netflix, Uber, and Amazon use this pattern for similar multi-tenant SaaS platforms. I'm basing my design on real industry practices, not just academic theory.

5. **Future Extensibility:** If I want to add a chatbot later, I can just create a new service that subscribes to the events. No need to modify existing services.

### How I'm Implementing It

**Services I'm building:**
- **Complaint Service:** Core functionality - creating and managing complaints
- **User Service:** Authentication (JWT), authorisation (RBAC), user management
- **Notification Service:** Handles sending emails and SMS (mocked for POC - just logs notifications)
- **Reporting Service:** Analytics and dashboards using CQRS pattern (documented but simplified for POC)

**How they communicate:**
- **Synchronously:** REST APIs through Kong API Gateway for user-facing requests
- **Asynchronously:** RabbitMQ message queue for events (e.g., when a complaint is created, publish an event; notification service picks it up)

**Data approach:**
- Each service has its own database (well, own schema in PostgreSQL)
- CQRS pattern: separate write database (normalised) from read database (denormalised for fast queries)
- Multi-tenancy: Using PostgreSQL schemas (tenant_001, tenant_002) for isolation

---

## What I'm Actually Building for POC

The full architecture is designed above, but for the proof-of-concept I'm focusing on:

**Implemented:**
- Complaint Service (create, view, assign complaints)
- User Service (JWT auth, RBAC)
- Basic React UI (complaint form and agent dashboard)
- PostgreSQL with 2 sample tenants
- Kong API Gateway
- Redis for session storage

**Mocked/Simplified:**
- Notification Service (just logs to console instead of calling SendGrid/Twilio)
- Reporting Service (static mock data)
- Event Sourcing (basic StatusHistory table instead of full event store)

This demonstrates the microservices principles while keeping the scope manageable for a 2-month timeline.

---

## Consequences

### What I Gain
- Each service can be developed and deployed independently
- If notifications are slow, it doesn't block complaint submission
- Event-driven patterns make the system more responsive
- Shows production-level architectural thinking
- Exceeds taught material (important for assessment criteria)

### What I'm Dealing With
- Takes longer to develop (about 40% more time than a monolith would)
- Had to learn RabbitMQ and event patterns
- Docker Compose setup is more complex than a single app
- Debugging across services requires proper logging (using structured logs with trace IDs)

### How I'm Managing the Complexity
- Using Docker Compose to run everything locally (much easier than managing services manually)
- Structured logging with trace IDs so I can follow requests across services
- Starting with just Complaint and User services, can add more later
- Kong API Gateway handles auth and rate limiting centrally (don't need to duplicate in every service)
- Writing detailed ADRs like this one to document my decisions

---

## Trade-offs I'm Making

**Complexity vs Scalability:** Yes, microservices are more complex than a monolith. But the multi-tenant requirements and need for independent scaling justify this complexity. If I was building a simple single-tenant system, I'd probably go monolithic.

**Development Time vs Learning:** This is taking longer, but I'm learning contemporary patterns that are actually used in industry. Worth it for both the assessment and my own understanding.

---

## How This Aligns with Standards

This approach follows:
- **ISO/IEC 25010:** Software quality model (covers maintainability, scalability)
- **Twelve-Factor App:** Methodology for building cloud-native apps
- **Domain-Driven Design:** My service boundaries follow bounded contexts
- **RESTful API Guidelines:** Using industry-standard API design

---

## Sources I Used

- Richardson, C. (2018). *Microservices Patterns*. Manning Publications.
- Newman, S. (2021). *Building Microservices* (2nd ed.). O'Reilly Media.
- Fowler, M. (2014). *Microservices*. Retrieved from https://martinfowler.com/articles/microservices.html
- Fowler, M. (2011). *CQRS*. Retrieved from https://martinfowler.com/bliki/CQRS.html
- Vernon, V. (2013). *Implementing Domain-Driven Design*. Addison-Wesley.
- Barclays. (2024). *Investor Update 2024*. Retrieved from https://home.barclays/investor-relations/

---

## Related Decisions

These other ADRs build on this decision:
- **ADR-002:** Event-Driven Architecture for Notifications (explains the RabbitMQ setup)
- **ADR-003:** CQRS Pattern for Reporting Service (why separate read/write models)
- **ADR-004:** Multi-Tenant Data Isolation Strategy (schema-based approach)
- **ADR-005:** Technology Stack Selection (why Python/FastAPI)
- **ADR-006:** Authentication Strategy (JWT with RBAC)
