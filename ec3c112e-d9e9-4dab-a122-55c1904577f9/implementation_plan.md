# Implementation Plan: Node.js Boilerplate from Scratch

In a live coding machine round, you are often expected to start from an empty folder. 
The goal is to get a server running in less than 5 minutes so you can spend the rest of the time on business logic.

## Goal
Initialize a standard Node.js Express backend from scratch to practice the implementation of the `POST /bookings` endpoint.

## Open Questions
> [!IMPORTANT]
> 1. Do you have PostgreSQL currently running locally on your machine, or should we just mock the `db.query` function for this practice session? (In a real interview, they might provide an online IDE with a DB attached, or ask you to mock it).
> 2. Where would you like me to create this practice project? (e.g., `/Users/pratik.giramkar/Desktop/toddle-practice`)

## Proposed Changes

We will execute the following steps to build muscle memory for Monday:

### 1. Initialize Project
- Run `npm init -y` to create `package.json`.
- Install dependencies: `npm install express`

### 2. Scaffold Architecture
Create a clean, modular structure. Do not put everything in `index.js`.
- `index.js` (Server entry point)
- `/routes`
  - `bookings.js` (Route definitions)
- `/controllers`
  - `bookingController.js` (The business logic and SQL we discussed)

### 3. Implement the Logic
- Fill out the `bookingController.js` with the SQL and try/catch error handling for the Unique Constraint violation.

## Verification Plan
- Run `node index.js`.
- Ping the server to ensure it responds.
- Simulate the `POST /bookings` request to test the mock logic.
