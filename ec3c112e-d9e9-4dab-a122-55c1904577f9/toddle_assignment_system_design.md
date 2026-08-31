# The 4-Point "Scope" Cheat Sheet
*Use this mental checklist whenever an interviewer gives you a vague problem statement to instantly generate high-quality clarifying questions.*

1. **The Multiplicity Rule (One vs. Many):**
   * *Always ask:* Can X do this multiple times, or just once?
2. **The "Oops" Rule (Updates & Deletes):**
   * *Always ask:* What happens if a user makes a mistake? Can they undo or edit it?
3. **The Race Condition Rule (Concurrency):**
   * *Always ask:* What happens if two people try to do the exact same thing at the exact same millisecond?
4. **The "Out of Scope" Rule (Cutting the fat / YAGNI):**
   * *Always ask:* Do I need to worry about [complex edge case/penalties/notifications], or can we assume a simplified version?

---

# System Design: Assignment Submission System

**Problem Statement:** Toddle wants to build a feature where Teachers can create a homework assignment with a deadline. Students can upload a document as their submission for that assignment before the deadline.

---

## 1. Scope (S)
*Using the cheat sheet to define the MVP.*

**Questions Asked:**
1. *(Multiplicity)*: Can a student upload MULTIPLE documents per assignment, or just a single file? -> **Single file for the MVP.**
2. *(Oops)*: Are Students allowed to update the submission if they upload a wrong one? -> **No, locked once submitted.**
3. *(Concurrency/Edge)*: What if they exactly upload at the deadline? -> **Accepted if exact (`<=`).**
4. *(Out of scope)*: Can Students submit after the deadline and do we need to penalize them for it? -> **No late submissions allowed. The API should reject them. No penalty logic needed.**

---

## 2. Data & Schema (D)
*Designing for extensibility (separating Submission and File) but implementing for the MVP.*

### Tables

**1. `Assignment` Table**
* `id` (UUID, Primary Key)
* `teacher_id` (UUID, Foreign Key)
* `class_id` (UUID, Foreign Key)
* `due_date` (TIMESTAMP) -> *Crucial for enforcing the business logic.*
* `created_at` (TIMESTAMP)
* `updated_at` (TIMESTAMP)

**2. `Submission` Table**
* `id` (UUID, Primary Key)
* `assignment_id` (UUID, Foreign Key -> references `Assignment(id)`)
* `student_id` (UUID, Foreign Key -> references `Student(id)`)
* `created_at` (TIMESTAMP)
* `updated_at` (TIMESTAMP)
* **`UNIQUE(assignment_id, student_id)`** -> *Guarantees at the database level that a student can only submit once per assignment, regardless of API race conditions.*

**3. `File` Table**
*Separated from Submission to future-proof for multiple file uploads, even though the MVP only requires one.*
* `id` (UUID, Primary Key)
* `submission_id` (UUID, Foreign Key -> references `Submission(id)`)
* `file_name` (TEXT)
* `file_url` (TEXT) -> *S3 bucket key.*
* `file_size` (INTEGER)
* `mime_type` (TEXT)
* `created_at` (TIMESTAMP)
* `updated_at` (TIMESTAMP)
* **`UNIQUE(submission_id, file_name)`** -> *Prevents uploading the same file twice to a submission, without strictly restricting the submission to a 1-to-1 relationship, leaving room for future scale.*
