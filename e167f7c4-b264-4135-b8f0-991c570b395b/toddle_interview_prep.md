# Toddle Interview Preparation: Director of Engineering (Miftah Jafary)

This document is designed to prepare you for a highly technical, context-aware discussion with the Director of Engineering at Toddle. It connects Toddle's specific product and engineering challenges directly to your backend and distributed systems experience at Juspay/Breeze.

---

## 1. Understand Toddle (Product & Business)

### What exactly is Toddle?
Toddle is an **AI-first, unified teaching and learning ecosystem** for K-12 schools. Instead of schools using five different apps (one for grading, one for curriculum planning, one for parent communication), Toddle consolidates everything into one operating system. 

### Who are its users?
- **Primary:** Teachers (creating lesson plans, grading).
- **Secondary:** School Administrators (managing curriculums, generating reports).
- **Tertiary:** Students (submitting work to portfolios) and Parents (viewing progress/communication).

### What problems does it solve?
- **Teacher Burnout/Admin Overload:** Teachers spend hours on administrative tasks. Toddle uses AI to generate rubrics, lesson plans, and reports, saving massive amounts of time.
- **Fragmented Data:** When grading is in one app and report cards are in another, data gets lost. Toddle centralizes the student data lifecycle.

### What are its main modules?
1. Curriculum Planning (AI-assisted lesson design).
2. Assessments & Gradebook (AI-assisted grading, rubrics).
3. Student Portfolios (Digital lockers for student work).
4. Progress Reports & Transcripts (Evidence-based report cards).
5. Family Communication (Messaging hub).

### Business Model & Scale
- **Who pays?** Schools and school districts pay on a B2B Enterprise SaaS recurring subscription model. 
- **Scale:** Over 2,000 schools in 100+ countries. Heavily focused on International Baccalaureate (IB), Cambridge, and premium K-12 schools.
- **Funding:** Backed by Tier-1 investors (Peak XV / Sequoia), having raised ~$36M, currently targeting a ~$100M valuation round.

### Major Integrations (Critical for Backend)
Toddle is deeply integrated into school ecosystems. It integrates with:
- **SIS (Student Information Systems):** PowerSchool, Veracross, Blackbaud.
- **SSO/Productivity:** Google Workspace, Microsoft Teams.
- **EdTech Tools:** Turnitin, Edpuzzle.

---

## 2. Understand the Engineering (Tech Stack)

*(Note: Toddle does not maintain a public engineering blog. The following is sourced from their job descriptions, engineering team profiles, and industry standard SaaS architecture).*

- **Backend:** **Node.js** is heavily utilized (confirmed via multiple engineering job postings).
- **Architecture:** Cloud-native, API-driven. Given the distinct modules (Planning vs. Grading vs. Portfolios), they likely operate a modular monolith or microservices architecture.
- **Database:** Relational databases (likely PostgreSQL/MySQL) for structured tenant/school data, supplemented by document stores (MongoDB) or blob storage (S3) for unstructured portfolio media.
- **Integrations:** Heavy reliance on external API consumption (webhooks, REST, OAuth) to sync with SIS platforms like PowerSchool.
- **AI Infrastructure:** They embed LLMs directly into workflows (AI lesson planners, AI grading). This requires async task queues to handle LLM latency without blocking the UI.
- **Frontend/Mobile:** React/React Native (standard for modern EdTech).

---

## 3. Think Like an Engineering Director

Based on their product, here are the technical problems Miftah and his team are actively fighting.

| The Challenge | Status | Why it's a problem |
| :--- | :--- | :--- |
| **Strict Multi-Tenancy & Data Isolation** | *Publicly Indicated* | 2,000 schools mean 2,000 tenants. A bug that leaks School A's student grades to School B is an extinction-level event (FERPA/GDPR violations). |
| **Complex Role-Based Access Control (RBAC)** | *Confirmed Toddle Challenge* | A teacher can view grades for Class A, but not Class B. A parent can view Student X, but not Student Y. Permissions are deeply hierarchical and nested. |
| **External API Reliability (SIS Syncing)** | *Inference based on Integrations* | Schools sync their rosters via PowerSchool. If the PowerSchool API goes down or rate-limits Toddle, Toddle must queue the syncs, retry them gracefully, and guarantee data consistency. |
| **Handling AI Latency & Costs** | *Inference based on AI Features* | Generating a lesson plan via LLM takes 5-15 seconds. This cannot be synchronous. They need robust background workers, async queues, and webhook callbacks to the UI. |
| **High Concurrency during "School Hours"** | *Inference based on EdTech patterns* | EdTech traffic is extremely spiky. 9:00 AM Monday sees massive load; 2:00 AM Sunday sees zero. The infrastructure must auto-scale rapidly. |

---

## 4. Connect Toddle to My Experience

This is where you win the interview. You must map your specific Vayu/Breeze experience to their challenges.

### 1. Reliable Third-Party Integrations (Toddle's SIS vs. Breeze's Magento)
- **Toddle Problem:** Syncing student rosters with legacy SIS APIs (PowerSchool) that fail, timeout, or return 500s.
- **My Experience:** "At Juspay, I built the Magento integration. External platforms are inherently unreliable. I couldn't just write a synchronous API call; I had to build idempotent reconciliation loops. If an order failed to sync, our ProcessTracker safety-net caught it, queued it, and retried it with exponential backoff."

### 2. Async Master-Worker Pipelines (Toddle's AI/Bulk Jobs vs. Breeze's Supermoney Sync)
- **Toddle Problem:** Generating 100 AI report cards at once, or syncing 5,000 students at the start of a semester without crashing the system.
- **My Experience:** "For the Supermoney integration, I designed a distributed Master-Worker pipeline. The Master queried thousands of pending transactions, chunked them into batches of 100, and pushed them to a Redis queue. Concurrent workers popped the batches, handled the 3-way DB joins, and executed the bulk external APIs. I understand how to design for high-throughput async processing without OOMing the servers."

### 3. Crash Recovery & Resilience (Toddle's Data Integrity vs. Breeze's Financial Integrity)
- **Toddle Problem:** If a backend pod crashes while syncing a school's grades, the grades cannot be left in a corrupted half-synced state.
- **My Experience:** "I treat data integrity like financial integrity. In the Supermoney worker, if a pod crashed mid-batch, the batch would be lost. I implemented a two-key Redis pattern: moving the batch to a `processing` key and adding it to a tracking list. If the pod crashed, the next Master run read the tracking list and recovered the orphaned batch. I design for failure."

### 4. Database Optimization (Toddle's Reporting vs. Breeze's Transactions)
- **Toddle Problem:** Generating a school-wide report requires joining students, classes, assignments, and rubrics.
- **My Experience:** "I optimized memory footprint by having the Master process query only IDs, while the localized Workers performed the heavy 3-way Beam JOINs on exactly 100 records at a time. I know how to avoid bringing down the database under load."

---

## 5. Questions I Can Ask the Director

Ask these to show you think like a senior engineer who understands system architecture.

**1. "Toddle integrates with heavy SIS platforms like PowerSchool and Veracross. How do you handle the reliability and state reconciliation when those external APIs inevitably rate-limit you or go down during a massive back-to-school roster sync?"**
- *Why it's good:* Proves you understand the pain of third-party integrations. It opens the door for you to talk about your Magento/ProcessTracker safety nets.

**2. "With the push towards an 'AI-first' LMS, I imagine LLM latency is a challenge. How are you architecting your async pipelines to handle long-running AI tasks without blocking the user experience?"**
- *Why it's good:* Shows you understand that AI isn't just an API call; it's a systems design problem. You can pivot to discussing your Master-Worker Redis queues.

**3. "Given that you serve over 2,000 schools globally, EdTech traffic must be incredibly spiky based on time zones and school hours. How is the backend architected to handle those 8:00 AM login stampedes?"**
- *Why it's good:* Demonstrates you are thinking about infrastructure scaling, database connection pooling, and caching.

**4. "In a multi-tenant environment like this, a data leak between schools would be catastrophic. Are you relying purely on application-level logical isolation (like `tenant_id` in every query), or do you use architectural boundaries like separate database schemas per school?"**
- *Why it's good:* Multi-tenancy is the hardest part of B2B SaaS. This shows deep architectural maturity.

---

## 6. Questions He Might Ask Me (And How to Answer)

**Q: "Tell me about a difficult distributed systems problem you solved."**
- **Answer:** "At Juspay, I had to sync thousands of successful transactions to Supermoney's ledger. A simple cron job would have caused OOM kills and database locks. I designed an asynchronous Master-Worker pipeline. The Master queried only the IDs of pending transactions, chunked them, and pushed them to Redis. N-workers popped the batches concurrently, hydrated the data via a 3-way JOIN, and executed the API calls. To survive pod crashes, I built a Redis tracking list so the Master could recover any orphaned batches the next day."

**Q: "How do you handle external API failures?"**
- **Answer:** "Never trust the external system. When we integrated Magento order creation, if the synchronous API call failed after a payment, we couldn't just drop the order. I utilized a ProcessTracker watchdog. The moment a payment succeeded, an async PT task was spawned. If the synchronous Magento call failed, the PT worker woke up, checked our DB, checked Magento via an idempotent `quoteId` query, and retried the creation with backoff. We built an eventually-consistent safety net around the external API."

**Q: "How do you ensure you don't process the same data twice (Idempotency)?"**
- **Answer:** "Two ways. First, database state machines. In our Supermoney sync, we strictly transitioned `supermoneySyncStatus` from PENDING to COMPLETED in the same transaction loop. Second, API idempotency keys. If our worker crashed *after* hitting the external API but *before* updating our DB, we ensured the retry utilized an idempotent identifier (like `partner_order_id`) so the external system would safely reject the duplicate."

**Q: "Why are you interested in Toddle?"**
- **Answer:** "I'm drawn to B2B SaaS platforms with high complexity. Building a unified ecosystem for 2,000 schools means solving hard problems around strict multi-tenancy, complex RBAC permissions, and heavy third-party SIS integrations. It’s exactly the kind of highly-concurrent, data-heavy backend environment where my experience building resilient pipelines at Juspay translates perfectly."

---

## 7. How I Should Talk About Toddle (Mental Models)

### The 30-Second Explanation (If asked: "What do you know about us?")
> "From my understanding, Toddle is a unified, AI-first operating system for K-12 schools. Instead of schools using five disconnected apps for curriculum planning, grading, and parent communication, Toddle brings it all into one multi-tenant SaaS platform. From an engineering side, I know that means deeply integrating with legacy SIS systems like PowerSchool and handling incredibly complex permission models and spiky school-hour traffic."

### Sophisticated Technical Observations to Drop Naturally:
- *"I noticed you integrate heavily with systems like Blackbaud and Veracross. I imagine reconciling data with legacy SOAP/REST endpoints requires some pretty robust async queueing on your backend."*
- *"Since grading and portfolios are so interconnected, I imagine you have to be very careful about circular dependencies in your domain logic when a single action cascades across multiple modules."*
- *"With AI generation for rubrics, you must have had to transition a lot of previously synchronous API calls into asynchronous background workers using Redis or Kafka."*

---

## 8. Final Preparation

### The 10 "Must Know" Facts Before the Call
1. **Core Product:** Unified AI-first LMS for K-12 schools.
2. **Key Modules:** Curriculum, Grading, Portfolios, Reports, Communication.
3. **Scale:** 2,000+ schools in 100+ countries.
4. **Business Model:** B2B Enterprise SaaS.
5. **Funding:** ~$36M raised (Peak XV / Sequoia), targeting ~$100M valuation.
6. **Integrations:** PowerSchool, Google Workspace, Microsoft Teams.
7. **Tech Stack:** Node.js backend.
8. **Data Problem:** Fragmented school data.
9. **Engineering Problem:** Multi-tenancy, RBAC, API reliability, spiky traffic.
10. **Your Value:** You build backend systems that do not drop data when things break.

### The 5 Strongest Technical Connections to Highlight
1. **Breeze's Magento Integration** ➔ **Toddle's SIS Integrations** (Handling external API failures safely).
2. **Breeze's Supermoney Pipeline** ➔ **Toddle's Bulk Report/Grading Generation** (Master-Worker architecture).
3. **Breeze's ProcessTracker** ➔ **Toddle's Async AI Tasks** (Distributed task scheduling).
4. **Breeze's Redis Crash Recovery** ➔ **Toddle's Data Integrity** (In-flight batch recovery).
5. **Breeze's Multi-merchant architecture** ➔ **Toddle's Multi-school architecture** (Data isolation).

### The 10 Areas to Revise Before the Meeting
1. The Supermoney Master-Worker architecture (exactly how batches are tracked in Redis).
2. The Magento `OrderStatusCheckTask` watchdog pattern.
3. How to design an idempotent API.
4. PostgreSQL performance (why query IDs first, then hydrate with 3-way JOINs).
5. OAuth 2.0 Client Credentials flow (how you handled it for Supermoney).
6. Handling race conditions using database locks (e.g., PostgreSQL `SELECT FOR UPDATE`).
7. Dead-letter queues and alerting (how you generated CSVs via Cassava for failed Supermoney transactions).
8. The difference between monolithic and microservice boundaries.
9. Scaling Node.js (Event loop, worker threads) - *just high level since Toddle uses it.*
10. JWTs and Role-Based Access Control.

### The 5 Best Questions to Ask
1. How do you handle the reliability and state reconciliation when SIS external APIs rate-limit you?
2. How are you architecting your async pipelines to handle long-running AI tasks without blocking the UI?
3. How is the backend architected to handle the massive 8:00 AM login traffic spikes?
4. For multi-tenancy, do you use logical isolation (`tenant_id` rows) or physical schema separation?
5. What is the most painful piece of technical debt the backend team is currently trying to pay down?
