# ADR-001: Adoption of Microservices Architecture with Event-Driven Patterns

---

**Status:** Accepted  
**Date:** 29 September 2025  
**Last Updated:** 18 December 2025  
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

### Option 2: Microservices Architecture (This is what I chose)

**What it is:** Break the system into separate services that can be deployed independently, each handling a specific business capability.

**Pros:**
- Can scale each service independently (e.g., if notifications are getting hammered, just scale that service)
- If one service fails, the others keep running (fault isolation)
- Can deploy updates to one service without touching the others
- Each service can use different tech if needed (though I'm sticking with Python for consistency)
- Shows understanding of modern industry practices - this is what companies like d Amazon use
- Perfect for demonstrating event-driven patterns and other advanced concepts

**Cons:**
- More complex to set up and manage
- Services talk to each other over the network, which adds latency
- Debugging is harder when the problem spans multiple services
- Takes longer to build initially
- Need to handle distributed transactions carefully

**Why I think this works:** This lets me demonstrate contemporary architecture patterns, which is exactly what the assignment brief asks for. The complexity is manageable with Docker Compose for the POC, and it naturally fits with event-driven communication via message queues. Companies dealing with similar multi-tenant systems at scale all use microservices, so it's grounded in real-world practice.

---

### Option 3: Service-Oriented Architecture (SOA)

**What it is:** Using an Enterprise Service Bus (ESB) to connect coarse-grained services.

**Pros:**
- Well-established pattern with lots of documentation
- Good for complex enterprise integration scenarios
- Supports sophisticated service orchestration

**Cons:**
- The ESB becomes a single point of failure
- Feels overly complex for what I'm trying to achieve
- Not really what modern cloud-native systems use anymore
- Doesn't showcase cutting-edge thinking as well

**My thoughts:** It's more complex than I need and doesn't align with where the modern industry is going.

---

### Option 4: Serverless Architecture

**What it is:** Using Function-as-a-Service like AWS Lambda - just write functions and let the cloud provider handle everything else.

**Pros:**
- Scales automatically without me doing anything
- Pay only for what you use
- No servers to manage
- Great for event-driven workloads

**Cons:**
- Gets expensive quickly with free tier limits
- Cold start latency when functions haven't run recently
- Hard to test and develop locally
- Would need to pay for AWS/Azure to demonstrate it properly

**My thoughts:** While serverless is definitely modern and relevant, it's not practical for a university project that needs to run on my laptop for demonstrations. I'd also be worried about accidentally racking up cloud bills during development.

---

## My Decision

**I'm going with Option 2: Microservices Architecture with Event-Driven Patterns**

### Why This Makes Sense

**1. It demonstrates advanced understanding**  
Microservices represent current best practices for scalable systems. This directly addresses the learning outcomes about contemporary architectures and being able to compare different approaches.

**2. Perfect fit for event-driven patterns**  
Using message queues for asynchronous communication lets me show understanding of event-driven architecture, which is essential for notifications and seems to be important for getting a first-class mark.

**3. Enables CQRS implementation**  
I can separate read and write operations in the reporting service, which demonstrates understanding of performance optimization patterns used in real production systems.

**4. Industry relevance**  
This is what actual companies use. Netflix, Amazon, Uber - they all use microservices for their multi-tenant SaaS platforms. It's not just academic theory.

**5. Independent scaling**  
Each service can scale based on its own demand. The notification service might need more instances during peak hours, while the reporting service might need different resources. This directly addresses the requirement to support 20M+ users.

**6. Shows architectural thinking**  
Even though I'm using Python throughout for simplicity, the architecture allows individual services to be rewritten in different technologies later if needed. This shows I'm thinking beyond just the immediate implementation.

### How I'm Implementing It

**Service Breakdown:**
- **Complaint Service:** Handles all complaint-related operations (the core domain)
- **User Service:** Authentication, authorization, and multi-tenant user management
- **Notification Service:** Consumes events and sends emails/SMS asynchronously
- **Reporting Service:** Separate read model for analytics (CQRS pattern)

**How Services Talk to Each Other:**
- **Synchronous:** REST APIs through an API Gateway for user-facing operations
- **Asynchronous:** RabbitMQ message queue for event-driven workflows (e.g., when a complaint is created, publish an event)

**Data Strategy:**
- Each service owns its own database (database per service pattern)
- CQRS approach: separate write database (transactional) from read database (analytics)
- Multi-tenancy using PostgreSQL schemas (one schema per tenant for strong isolation)

---

## Consequences

### The Good Stuff

- Clear separation of concerns makes the code much more maintainable
- Can deploy and update services independently without breaking everything
- Event-driven patterns make the system more responsive
- Shows production-level architectural thinking (hopefully this helps with marks)
- Goes well beyond what was covered in lectures

### The Challenges

- Takes longer to develop than a simple monolith would
- Need to learn RabbitMQ and event-driven patterns properly
- Local development setup is more complex (need Docker Compose)
- More potential for things to go wrong with network communication between services
- Debugging distributed systems is harder

### How I'm Managing the Challenges

- Using Docker Compose to make running everything locally much simpler
- Implementing proper logging so I can trace requests across services
- Building core services first, adding complexity incrementally
- Using Kong API Gateway to centralize authentication and other cross-cutting concerns
- Writing detailed ADRs (like this one) to document why I made each decision

---

## Standards and Best Practices

This decision aligns with:
- **ISO/IEC 25010:** Software quality model (particularly maintainability and scalability)
- **Twelve-Factor App:** Methodology for building modern cloud-native applications
- **Domain-Driven Design:** Service boundaries follow bounded contexts from DDD
- **RESTful API Guidelines:** Industry-standard API design principles

---

## Related Decisions

These other ADRs build on this foundation:
- **ADR-002:** Event-Driven Architecture for Notifications
- **ADR-003:** CQRS Pattern for Reporting Service
- **ADR-004:** Multi-Tenant Data Isolation Strategy
- **ADR-005:** Technology Stack Selection (Python FastAPI)

---

## Notes

Looking back at this decision after implementing the POC, I think microservices was the right call for demonstrating architectural understanding, even though it definitely took longer than a monolith would have. The event-driven patterns and independent scalability really showcase modern practices. That said, if I were building this for a real startup with limited resources, I'd probably start with a well-structured monolith and migrate to microservices later when the complexity justified it.
