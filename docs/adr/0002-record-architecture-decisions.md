# ADR-002: Event-Driven Architecture for Asynchronous Notifications

---

**Status:** Accepted  
**Date:** 13 October 2025  
**Last Updated:** 10 november 2025
**Author:** Adam James Brown  

---

## Context and Problem Statement

The CMS needs to send notifications to users whenever their complaint status changes - things like when a complaint is created, assigned to someone, or resolved. The system serves multiple organisations and each might have different notification preferences. 

The tricky bit is making sure these notifications get sent reliably without slowing down the main complaint processing. If someone submits a complaint, I want that to be fast - not sitting there waiting for an email to be sent before confirming the submission.

**The big question:** Should I send notifications synchronously (wait for them to send) or asynchronously (fire and forget)?

---

## Decision Drivers

### What Needs to Happen (Functional Requirements)
- Users need to be notified when complaint status changes (R001, R005)
- Need to support both email and SMS
- Notifications must be reliable - can't just lose them if something goes wrong
- Creating/updating complaints can't be slowed down by notification sending

### Performance & Quality Requirements
- **NFR2 (Performance):** Need < 2s response time - if I'm calling SendGrid synchronously, that's adding 500ms-2s of latency every time
- **NFR3 (Reliability):** 99.5% uptime - if SendGrid is down, I can't let that crash my complaint service
- **NFR6 (Maintainability):** Adding new notification channels (like WhatsApp later) should be straightforward

### Technical Challenges
- External services (SendGrid for email, Twilio for SMS) can be slow or fail
- Network issues to external APIs need to be handled gracefully
- Multiple parts of my system need to know about the same events (Notification Service obviously, but also Reporting Service for analytics)

---

## Options I Considered

### Option 1: Synchronous Notification Calls

**What it is:** When someone updates a complaint, the Complaint Service directly calls the Notification Service via HTTP and waits for it to finish.

**Pros:**
- Really simple to implement - just a direct HTTP call
- Get immediate feedback if the notification failed
- Easy to debug - just follow the call stack
- No extra infrastructure needed

**Cons:**
- **Tight coupling** - My complaint service now depends on the notification service being up
- **Performance hit** - Every complaint update now takes 200-500ms longer
- **Single point of failure** - If notifications are down, complaint updates start failing too
- **Can't handle spikes** - If 1000 complaints get resolved at once, this would be really slow
- **No retry logic** - If it fails, I have to build all the retry stuff myself

**My thoughts:** This violates my performance requirements (NFR2) and would make the system less reliable. It also goes against the whole point of microservices - services should be independent. If the notification service is having issues, why should that stop people from updating complaints?

---

### Option 2: Event-Driven Architecture with Message Queue ✓ **(My Choice)**

**What it is:** When a complaint status changes, the Complaint Service publishes an event to RabbitMQ. The Notification Service listens to these events and sends notifications when it sees them.

**Pros:**
- **Loose coupling** - Services talk through events, not direct calls
- **Non-blocking** - Complaint updates finish in <100ms regardless of how long notifications take
- **Fault tolerant** - If the Notification Service crashes, messages sit in the queue and get processed when it comes back up
- **Scalable** - Can run multiple Notification Service instances to process the queue faster
- **Built-in retry** - RabbitMQ handles redelivery if something fails
- **Extensible** - Can add new services (like Analytics) that listen to the same events without touching the Complaint Service
- **Pub/Sub pattern** - One event can trigger multiple things (send notification AND update analytics)

**Cons:**
- Need to run RabbitMQ (more infrastructure to manage)
- Eventual consistency - notification might arrive a couple seconds after the status change
- Debugging is trickier - need to trace events across services
- Have to think about message ordering
- Need to monitor the message queue health

**Why I chose this:** This is the proper microservices way to do it. Yes, it's more complex, but the benefits are worth it. The performance and reliability gains align with my NFRs, and it's how companies like Netflix and Uber actually do this in production. Plus, it gives me something interesting to talk about in my assessment - just doing synchronous HTTP calls wouldn't demonstrate much understanding.

---

### Option 3: Database Polling

**What it is:** The Notification Service checks the database every few seconds looking for new complaint status changes.

**Pros:**
- No message queue needed
- Simple to implement
- Guaranteed delivery (database is the source of truth)

**Cons:**
- **Inefficient** - Constantly hitting the database wastes resources
- **Slow** - Notifications delayed by whatever the polling interval is (5-30 seconds usually)
- **Doesn't scale** - Polling adds tons of unnecessary database load
- **Tight coupling** - Notification Service needs to understand the Complaint Service's database schema

**My thoughts:** . The latency would be terrible and it wouldn't scale at all. Not a good solution for "real-time" notifications.

---

## My Decision

**I'm going with Event-Driven Architecture using RabbitMQ.**

### Why This Makes Sense

1. **Performance (NFR2):** Complaint updates finish in <100ms. Users don't sit there waiting for emails to send - they get immediate confirmation and the notification happens in the background.

2. **Reliability (NFR3):** RabbitMQ stores messages durably. If my Notification Service crashes while processing, the messages don't get lost - they'll be reprocessed when the service comes back up.

3. **Scalability (NFR5):** During peak times (like when support staff resolve a bunch of complaints at once), I can scale up the Notification Service instances to process the queue faster. The queue absorbs the burst.

4. **Maintainability (NFR6):** If I want to add a new feature that reacts to complaint events (like updating analytics or triggering a chatbot), I just create a new subscriber. No changes needed to the Complaint Service.

5. **Industry Standard:** This is how it's actually done in production systems. Companies like Netflix, Uber, and Amazon use event-driven architectures for exactly this kind of thing.

### How I'm Implementing It

**Events I'm publishing:**
- `ComplaintCreated` - When a user submits a new complaint
- `ComplaintStatusChanged` - When status changes (NEW → ASSIGNED → IN_PROGRESS → RESOLVED → CLOSED)
- `ComplaintAssigned` - When a complaint gets assigned to a support person

**Message format (JSON):**
```json
{
  "event_id": "uuid-here",
  "event_type": "ComplaintStatusChanged",
  "timestamp": "2025-10-13T14:30:00Z",
  "aggregate_id": "complaint-uuid",
  "tenant_id": "tenant-001",
  "payload": {
    "old_status": "ASSIGNED",
    "new_status": "IN_PROGRESS",
    "changed_by": "user-uuid"
  }
}
```

**RabbitMQ setup:**
- Using a topic exchange so I can route messages flexibly
- Durable queues (messages persist to disk so they survive restarts)
- Dead Letter Queue - if a message fails 3 times, it goes to a separate queue for me to investigate manually
- Manual acknowledgment - so I can be sure each message gets processed at least once

---

## What I'm Actually Building for POC

For the proof-of-concept, I'm simplifying the Notification Service:
- **Mocked:** Instead of actually calling SendGrid/Twilio, it just logs the notifications to console
- **Why:** Don't need real email/SMS for demonstration, and this avoids needing API keys
- **Still demonstrates:** The event-driven pattern, message queuing, and asynchronous processing

The RabbitMQ setup is real though - events are actually published and consumed through the queue.

---

## Consequences

### What I Gain
- Complaint Service stays fast and responsive
- System handles notification service failures gracefully
- Easy to add new event subscribers (Analytics, Audit logging, future chatbot)
- Can scale notification processing horizontally
- Get a clear audit trail of all domain events for free

### What I'm Dealing With
- RabbitMQ is now critical infrastructure (needs monitoring and backups)
- Eventual consistency - users see notifications 1-3 seconds after the status change (acceptable though)
- Debugging requires tracing events across services (using correlation IDs to help with this)
- More complex deployment (need to ensure RabbitMQ is running)

### How I'm Managing It
- Using Docker Compose to run RabbitMQ alongside other services (keeps it manageable for POC)
- Adding correlation IDs to trace events through the system
- Documenting the message schemas clearly
- Setting an SLA: notifications should arrive within 5 seconds under normal conditions

---

## Trade-offs I'm Making

**Complexity vs Performance:** Yes, adding a message queue makes things more complex. But the performance benefits are significant - users aren't waiting for external API calls to complete. And the complexity is manageable with Docker Compose.

**Immediate vs Eventual Consistency:** With synchronous calls, you know immediately if the notification sent. With events, there's a 1-3 second delay. But this is totally acceptable for notifications - users don't notice a 2-second delay in receiving an email.

**Learning Curve vs Industry Practice:** I had to learn RabbitMQ and event-driven patterns, which took time. But this is how it's actually done in industry, so the learning is valuable beyond just this assignment.

---

## How This Aligns with Standards

This approach follows:
- **Reactive Manifesto:** Message-driven architecture for building resilient systems
- **Twelve-Factor App:** Concurrency through stateless processes
- **Domain-Driven Design:** Treating domain events as first-class citizens in the architecture

---

## Sources I Used

- Fowler, M. (2017). *What do you mean by "Event-Driven"?* Retrieved from https://martinfowler.com/articles/201701-event-driven.html
- Richardson, C. (2018). *Microservices Patterns*, Chapter 3: Interprocess communication. Manning Publications.
- Stopford, B. (2018). *Designing Event-Driven Systems*. O'Reilly Media.
- RabbitMQ Documentation. (2024). *Reliability Guide*. Retrieved from https://www.rabbitmq.com/reliability.html

---

## Related Decisions

These other ADRs connect to this one:
- **ADR-001:** Microservices Architecture (establishes the foundation)
- **ADR-003:** CQRS Pattern for Reporting Service (also uses these events)
- **ADR-005:** Technology Stack Selection (explains why RabbitMQ over Kafka)
