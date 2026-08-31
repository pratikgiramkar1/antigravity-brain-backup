# System Design: Google Drive-style File Manager

**Problem Statement:** Build a Google Drive-style File Manager for Toddle Classrooms. Teachers can upload files and create folders. Folders can be nested inside other folders infinitely. Students can view files if they belong to the class.

---

## 1. Scope (S)
*Clarifying questions to establish the MVP boundaries.*

**Questions Asked:**
1. *Can teachers upload bulk files at once?* -> **No, one file at a time for the MVP.**
2. *What permissions do students have?* -> **Read-only. Only teachers can create/upload/delete.**
3. *How deep can folders go?* -> **Infinitely deep.**

---

## 2. Data & Schema (D)
*Designing for hierarchical data (Adjacency Lists) and ensuring data integrity.*

Assumptions: `teachers`, `students`, and `classes` tables already exist.

### Tables

**1. `Folder` Table**
*Uses a self-referencing foreign key to handle infinite nesting.*
* `id` (UUID, Primary Key)
* `name` (TEXT)
* `teacher_id` (UUID, Foreign Key)
* `classroom_id` (UUID, Foreign Key)
* **`parent_folder_id`** (UUID, Foreign Key -> references `Folder(id)`, nullable for root folders)
* `is_active` (BOOLEAN, default true -> used for soft deletes)
* `created_at` (TIMESTAMP)
* `updated_at` (TIMESTAMP)
* **`UNIQUE(parent_folder_id, name)`**: Prevents two folders with the exact same name from existing *inside the same parent folder*.
* **Index on `parent_folder_id`**: For lightning-fast $O(\log n)$ retrieval of sub-folders.

**2. `File` Table**
*Points to its parent folder. Stores metadata about the actual file residing in an object store (e.g., S3).*
* `id` (UUID, Primary Key)
* `name` (TEXT)
* `folder_id` (UUID, Foreign Key -> references `Folder(id)`)
* `file_url` (TEXT) -> *The S3 object key or CDN URL where the actual file lives.*
* `size_bytes` (INTEGER) -> *Bonus points: Allows frontend to show file size without downloading.*
* `is_active` (BOOLEAN, default true -> used for soft deletes)
* `created_at` (TIMESTAMP)
* `updated_at` (TIMESTAMP)
* **`UNIQUE(folder_id, name)`**: Prevents duplicate file names within the same folder.
* **Index on `folder_id`**: For lightning-fast retrieval of all files in a specific folder.

---

## Key Interview Concepts Demonstrated:
1. **The Adjacency List Pattern:** By having `parent_folder_id` point to its own table, we can model hierarchies infinitely deep without creating separate tables for "SubFolder1", "SubFolder2", etc.
2. **Composite Unique Constraints:** Using `UNIQUE(parent_folder_id, name)` instead of just `UNIQUE(name)` ensures that name uniqueness is scoped correctly.
3. **Soft Deletes:** Using `is_active` instead of actually executing SQL `DELETE` commands, preserving data for recovery or auditing.
4. **B-Tree Indexing:** Explicitly calling out the need for an index on Foreign Keys (`parent_folder_id` / `folder_id`) to optimize read queries when opening a folder.
