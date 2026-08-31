# System Design: Live Classroom Polling Feature

**Problem Statement:** A Teacher can create a multiple-choice poll during a live class. Students in that class can submit their single-choice vote. The Teacher needs to see the results update in real-time.

---

## 1. Scope (S)
*Always ask clarifying questions to narrow the problem into an MVP.*

**Clarifying Questions Asked:**
1. *Can a student change their vote?* -> **No, votes are locked.** (Avoids complex UPDATE logic).
2. *Is it multi-select?* -> **No, single choice.** (Simplifies database relationships).
3. *Do we need to track who voted for what in the UI?* -> **No, UI just shows aggregate counts, but we must prevent double-voting.**
4. *Can a poll be reopened?* -> **No, once closed it remains closed.**

---

## 2. Data & Schema (D)
*Designing for 3rd Normal Form (3NF) to ensure data integrity and prevent anomalies.*

Assumptions: `teachers`, `students`, and `classes` tables already exist. We are designing the feature-specific tables.

### Tables

**1. `Poll` Table**
*The core entity representing the poll event.*
* `id` (UUID, Primary Key)
* `teacher_id` (UUID, Foreign Key)
* `classroom_id` (UUID, Foreign Key)
* `question` (TEXT)
* `is_open` (BOOLEAN, default: true)

**2. `Option` Table**
*Represents the choices for a poll. Foreign key goes on this child table to satisfy 1NF (No arrays in the Poll table).*
* `id` (UUID, Primary Key)
* `poll_id` (UUID, Foreign Key -> references `Poll(id)`)
* `text` (TEXT)

**3. `Vote` Table**
*Records the action of a student voting. We denormalize slightly by including `poll_id` so we can use a composite unique constraint to guarantee no double-voting at the database layer.*
* `id` (UUID, Primary Key)
* `poll_id` (UUID, Foreign Key -> references `Poll(id)`)
* `option_id` (UUID, Foreign Key -> references `Option(id)`)
* `student_id` (UUID, Foreign Key -> references `Student(id)`)
* **`UNIQUE(poll_id, student_id)`** -> *CRITICAL: This database constraint strictly prevents a student from voting twice in the same poll.*

---

## 3. API (A)
*RESTful endpoints representing the business actions.*

* **Create Poll:** `POST /polls`
* **Cast Vote:** `POST /polls/:id/votes`
* **Close Poll:** `PATCH /polls/:id` (payload: `{ is_open: false }`)
* **Get Results:** `GET /polls/:id/results`

**Example Request Payload for `POST /polls`:**
*(Note how the API contract uses an array for ease of use by the frontend, but the backend will break this apart and insert into the `Option` table).*
```json
{
    "teacher_id": "uuid",
    "classroom_id": "uuid",
    "question": "Which is the tallest Mountain Range?",
    "options": [
        "Andes",
        "Himalayas",
        "Rocky Mountains",
        "Atlas Mountains"
    ]
}
```

---

## 4. Implementation (I)
*Addressing the "Real-time" requirement.*

While standard HTTP REST is used for creating polls and submitting votes, REST is unidirectional (Client -> Server) and cannot push updates to the Teacher. 

To achieve real-time dashboard updates for the Teacher:
* **WebSockets (e.g., Socket.io):** Establish a persistent connection when the Teacher opens the dashboard. When a `POST /polls/:id/votes` request successfully saves a vote, the Node.js server emits an event (e.g., `NEW_VOTE_COUNT`) with the updated aggregate data directly to the Teacher's WebSocket connection.
