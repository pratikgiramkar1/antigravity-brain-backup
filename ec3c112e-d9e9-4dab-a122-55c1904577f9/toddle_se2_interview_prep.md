# Toddle SE-2 Backend Live Coding Preparation Guide

Based on an analysis of Toddle's interview process, candidate reports, and industry expectations for an SE-2 Backend role (specifically transitioning to a Node.js stack), here is your focused preparation plan.

> [!NOTE]
> **Context:** Given your 3 years at Juspay (Rust/Haskell, heavy DB/Distributed Systems), interviewers will likely assume your fundamental architecture and backend knowledge is very strong. **Their primary goal in this round is to see if you can write idiomatic, production-ready Node.js code and design practical features.**

---

## 1. What to Expect on Monday
*(Based on Candidate Reports & Toddle's Process)*

* **Format:** A 60–90 minute pair-programming session with 2–3 engineers.
* **The Task:** You will likely be given a realistic problem statement related to an EdTech product (e.g., managing classrooms, student assignments, real-time collaboration).
* **The Environment:** You may be asked to share your screen and use your own IDE to spin up a quick Express.js/Node.js app, or use an online platform like CoderPad.
* **The Flow:**
  1. **Requirement Gathering (5-10 mins):** Discussing the API requirements.
  2. **Schema Design (15 mins):** Designing the SQL/Postgres schema to support the feature.
  3. **Implementation (30-40 mins):** Writing the actual REST/GraphQL endpoints, connecting to the mock DB, adding validation, and writing the business logic.
  4. **Deep Dive (15 mins):** They will ask you to optimize queries, explain how you'd handle concurrency/race conditions, and discuss Node.js specifics (e.g., "What happens to the event loop if this API does heavy JSON parsing?").

---

## 2. Topic Priority

* **HIGH:** Node.js (Async/await, Event Loop, Express.js structure), Database Schema Design (Postgres, Normalization, complex JOINs), API Design (REST/GraphQL, validation, error handling).
* **MEDIUM:** Core JS Concepts (Closures, Promises, `this` context), Authentication flows (JWT, role-based access), Practical Data Structures (Hash Maps, Tree traversal for JSON).
* **LOW:** Hard LeetCode (DP, Graphs), Advanced System Design (Kafka, Microservices - they know you know this from Juspay, they want to see you *code*).

---

## 3. Top 15 Node.js / Machine Coding Problems to Practice
*(Inferred & Generic Industry expectations for Node.js live coding)*

**API & Express Specific:**
1. **Rate Limiter:** Write an Express middleware to rate-limit requests per IP (using in-memory JS Maps, no Redis).
2. **Role-based Auth:** Implement a JWT authorization middleware that checks if a user is a 'Teacher' or 'Student' before allowing access to a route.
3. **Pagination & Filtering:** Build a REST endpoint that takes query params (`?page=2&limit=10&status=active`) and returns correctly structured paginated JSON.
4. **Error Handling Wrapper:** Create a global error handling middleware in Express that catches unhandled async rejections and formats standard API error responses.

**Async & Concurrency in JS:**
5. **Batch Processing:** Write a function that processes an array of 10,000 IDs asynchronously by making an API call, but only processes exactly `N` requests concurrently to avoid memory spikes.
6. **Retry Logic:** Implement an async function wrapper that retries a failing Promise up to 3 times with exponential backoff (good way to show Juspay resiliency skills in JS).
7. **Race Condition Prevention:** Simulate an endpoint where two users try to book the same "slot" concurrently. Show how you handle this at the DB level (Transactions/Locks) in Node.
8. **Callback to Promise:** Convert a legacy nested callback function into a clean `async/await` structure.

**Data Manipulation (Very likely):**
9. **JSON to Tree:** Given a flat array of database rows (e.g., comments with `parent_id`), write a function to construct a nested JSON tree.
10. **CSV Parser:** Write an endpoint that accepts a simulated CSV upload, parses it streamingly (or in chunks), and prepares bulk DB inserts.
11. **Event Emitter:** Use Node's native `EventEmitter` to build a simple pub-sub system for a mock chat room.
12. **Deep Merge/Compare:** Write a function to deeply compare two JSON objects and return the diff (often used in audit logs or versioning).

---

## 4. Top 10 Database & Schema Design Problems
*(Confirmed Toddle focuses heavily on relational databases and ACID)*

Be prepared to write the **CREATE TABLE** statements and the **SQL queries**.

1. **Google Classroom Clone:** Design a schema for Teachers, Students, Classes, and Assignments. Write a query to fetch a student's pending assignments.
2. **Attendance System:** Design a schema to track daily attendance. Write a query to find students with < 75% attendance for a specific month.
3. **RBAC (Role Based Access Control):** Design tables for Users, Roles, and Permissions. Write a `JOIN` to verify if user X has permission Y.
4. **File Management (Google Drive style):** Schema for files, folders, and shared access. How do you model the nested folder hierarchy (Adjacency list vs Closure table)?
5. **Chat Application:** Schema for users, group chats, messages, and read receipts.
6. **Handling Money/Wallets:** (Play to your Juspay strengths) Design a ledger table that guarantees no double spending using ACID transactions and constraints.
7. **N+1 Problem Mitigation:** Given a GraphQL query asking for a Class and all its Students, write the optimized SQL query (using `IN` or `JOIN`) rather than looping.
8. **Indexing Strategy:** You have a table with millions of submissions. The query `WHERE status = 'graded' AND updated_at > X` is slow. How do you index it? (Composite index order matters).

---

## 5. Important Node.js Concepts to Revise
*(They will ask these to ensure you aren't just treating Node like Rust)*

* **The Event Loop:** Understand phases (Timers, Pending, Poll, Check). Know the exact difference between `process.nextTick()`, `setImmediate()`, and `setTimeout(..., 0)`.
* **Single Thread Limitations:** How does Node handle 10,000 concurrent I/O requests vs. 1 heavy CPU task (like parsing a massive JSON or image processing)? (Answer: Libuv thread pool vs blocking the main thread).
* **Memory Management:** Briefly review how V8 garbage collection works and what causes memory leaks in Node (e.g., global arrays, un-cleared intervals, heavy closures).
* **Module System:** CommonJS (`require`) vs ES Modules (`import`).
* **Scoping & `this`:** Understand lexical scoping, closures, and how arrow functions inherit `this` from the parent scope (a common bug source for new JS devs).

---

## 6. DSA Topics Worth Preparing
*(Candidate reports indicate easy-medium DSA, primarily for data manipulation)*

Don't grind hard LeetCode. Focus on:
* **Hash Maps / Sets:** Fast lookups, counting frequencies, deduplicating DB results.
* **Arrays:** Two-pointer techniques, `map`, `reduce`, `filter`, `sort`.
* **Trees:** Basic traversal (DFS/BFS) for working with hierarchical data (like org charts or comment threads).

---

## 7. What Engineers Are Actually Evaluating

1. **JS/Node Fluency:** Can you write clean JS without fighting the syntax? Do you understand Promises natively?
2. **Pragmatism:** Do you try to over-engineer a simple CRUD endpoint, or do you build something simple that works and then discuss scaling?
3. **Database Rigor:** Since you come from Juspay, they will expect your DB schemas to be flawless (normalized, proper constraints, foreign keys).
4. **Communication:** Toddle highly values proactive communication. **Think out loud.** Tell them *why* you are choosing a `Map` over an `Object`, or why you are adding an index.
5. **Handling Feedback:** If they point out a bug, how quickly do you adapt?

---

## 8. Focused Preparation Plan for the Remaining Days

* **Day 1: Muscle Memory (Node/Express)**
  * Initialize an empty Node project. Install `express`, `cors`, `pg` (or `sqlite3` for mock DB).
  * Build a fully functioning CRUD API for "Classrooms" with basic validation and a global error handler. Get fast at writing this boilerplate.
* **Day 2: Async Mastery & Middlewares**
  * Implement the Rate Limiter and JWT Auth middlewares from the list above.
  * Write the "Concurrency limiter / Batch processor" script to practice Promises.
* **Day 3: Schema & SQL**
  * Take a piece of paper and design the schemas for the Chat App and the Google Classroom clone.
  * Write out the raw SQL for the 3 most complex queries for those schemas.
* **Day 4: The "Juspay Translation"**
  * Review your Juspay projects. Figure out how you would explain those distributed systems concepts (idempotency, retries) in the context of a Node.js server.
  * Do one timed mock session: Give yourself 45 minutes to build an API endpoint that uploads a file, parses it, and saves it to a DB.
