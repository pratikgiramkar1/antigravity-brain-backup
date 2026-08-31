# PTM Scheduler - Completed Implementation & Architecture

## 1. Overview
The Parent-Teacher Meeting (PTM) Scheduler allows teachers to publish time slots and parents to book them. The primary design challenge is handling concurrent booking requests without double-booking or race conditions.

---

## 2. PostgreSQL Schema & Constraints

```sql
CREATE TABLE students (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE
);

CREATE TABLE teachers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE
);

CREATE TABLE parents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE
);

CREATE TABLE meeting_slots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    teacher_id UUID NOT NULL REFERENCES teachers(id),
    date DATE NOT NULL,
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    status VARCHAR(50) DEFAULT 'AVAILABLE' -- AVAILABLE, BOOKED, CANCELLED
);

CREATE TABLE bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slot_id UUID NOT NULL UNIQUE REFERENCES meeting_slots(id), -- Enforces 1 booking per slot
    parent_id UUID NOT NULL REFERENCES parents(id),
    student_id UUID NOT NULL REFERENCES students(id),
    teacher_id UUID NOT NULL REFERENCES teachers(id),         -- Denormalized for composite UNIQUE
    booking_date DATE NOT NULL,                              -- Denormalized for composite UNIQUE
    UNIQUE(parent_id, teacher_id, booking_date)              -- 1 booking per parent per teacher per day
);
```

---

## 3. Concurrency Strategy
* **Database Constraint Protection:** `bookings.slot_id` has a `UNIQUE` index.
* **Race Condition Flow:**
  1. Two parents hit `POST /bookings` for the same `slot_id` at the exact same millisecond.
  2. Both pass the initial `SELECT status = 'AVAILABLE'` check.
  3. Both execute `INSERT INTO bookings...`.
  4. PostgreSQL allows one transaction to succeed and throws Error Code `23505` (Unique Violation) for the second transaction.
  5. The controller catches `23505` and returns HTTP `409 Conflict`.

---

## 4. API Endpoints

1. **`POST /slots`**
   * Creates a meeting slot.
   * Body: `{ teacher_id, date, start_time, end_time }`
   * Response: `201 Created`

2. **`GET /slots?teacher_id={id}&date={date}`**
   * Fetches available/booked slots for a teacher on a specific day.
   * Response: `200 OK` with array of slots.

3. **`POST /bookings`**
   * Books a slot for a parent/student.
   * Body: `{ slot_id, parent_id, student_id }`
   * Response: `201 Created` (or `409 Conflict` on double-booking).

---

## 5. Directory & Code Structure
Project location: `/Users/pratik.giramkar/skills/toddle/ptm-scheduler`
* `config/db.js`: PostgreSQL connection pool setup (`pg`).
* `controllers/slotController.js`: Logic for creating and fetching slots.
* `controllers/bookingController.js`: Logic for booking slots and catching `23505` unique violations.
* `routes/slotRoutes.js` & `routes/bookingRoutes.js`: Express router definitions.
* `index.js`: Express app entry point.
* `seed.js`: Database initialization and test data seeding script.
