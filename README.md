# event-management-nosql

> A MongoDB-based NoSQL backend for an Event Management System, built as part of the Database Programming (SECP3623) course at Universiti Teknologi Malaysia.

---

## Table of Contents

- [Overview](#overview)
- [Team Members](#team-members)
- [Data Model](#data-model)
  - [Collections](#collections)
  - [Embedding vs Referencing](#embedding-vs-referencing)
  - [Data Types](#data-types)
- [CRUD Operations](#crud-operations)
- [Advanced MongoDB Operations](#advanced-mongodb-operations)
  - [Indexing](#indexing)
  - [Aggregation Pipelines](#aggregation-pipelines)
  - [Sorting](#sorting)
- [Security & Challenges](#security--challenges)
- [Getting Started](#getting-started)
- [Project Structure](#project-structure)

---

## Overview

This project implements a NoSQL document database for managing events, participants, sessions, and payments. It was built using MongoDB and demonstrates the full lifecycle of an event management backend — from data modelling and CRUD operations to indexing, aggregation pipelines, and security analysis.

**Course:** Database Programming (SECP3623) — Section 02, Semester 01 2025/2026  
**Lecturer:** Dr. Shamini A/P Raja Kumaran  
**Presentation:** [YouTube Video](https://youtu.be/_RTkglE2Tuc)

---

## Team Members

| Name | Matric No. | Role |
|---|---|---|
| Dheshieghan A/L Saravana Moorthy | A23CS0072 | NoSQL Data Model Design & Collection Structure |
| Muhammad Adam bin Razali | A23CS0116 | CRUD Operations & Projections |
| Pravinraj A/L Sivabathi | A23CS0171 | Indexing, Aggregation Pipelines & Sorting |
| Muhammad Afiq Danish bin Mohd Hazni | A23CS0118 | Security Analysis & Report |

---

## Data Model

### Collections

The database consists of four collections that together model the full event management workflow:

**`users`** — Stores all system users (attendees and organizers). Separating user data avoids duplication across registrations and ensures a single point of update for profile changes.

```json
{
  "_id": "user_01",
  "name": "Muhammad Adam",
  "email": "adam@gmail.com",
  "role": "organizer",
  "city": "Johor Bahru",
  "phone": "012-3456789"
}
```

**`events`** — Stores core event information including title, location, date, organizer reference, and embedded ticket types. Ticket types are embedded because they are tightly coupled with the event and always retrieved together.

```json
{
  "_id": "evt_01",
  "title": "Tech Innovation Conference 2026",
  "description": "A conference focusing on emerging technology trends.",
  "location": "Kuala Lumpur Convention Centre",
  "startDate": "2026-03-10T09:00:00Z",
  "endDate": "2026-03-10T18:00:00Z",
  "organizerId": "user_02",
  "ticketTypes": [
    { "type": "VIP", "price": 500, "qty": 50 },
    { "type": "General", "price": 150, "qty": 200 }
  ],
  "isPublished": true,
  "createdAt": "2026-01-01T10:00:00Z"
}
```

**`registrations`** — Acts as the linking collection between users and events (many-to-many). Embeds payment and ticket details to preserve historical accuracy even if event prices change later.

```json
{
  "_id": "reg_01",
  "registrationCode": "EVT-2026-0045",
  "userId": "user_01",
  "eventId": "evt_10",
  "ticket": { "type": "VIP", "price": 500 },
  "payment": { "method": "FPX", "amount": 500, "date": "2026-02-15T06:40:00Z" },
  "status": "CONFIRMED",
  "registeredAt": "2026-02-15T06:40:00Z"
}
```

**`sessions`** — Stores individual sessions or agenda items within an event. Each session references its parent event and embeds speaker details for efficient retrieval without additional lookups.

```json
{
  "_id": "ses_01",
  "eventId": "evt_10",
  "title": "AI in Everyday Applications",
  "speaker": { "name": "Dr. Lim Wei", "bio": "AI researcher at UTM" },
  "startTime": "2026-03-10T10:00:00Z",
  "endTime": "2026-03-10T11:30:00Z",
  "room": "Hall A"
}
```

### Embedding vs Referencing

| Strategy | Applied To | Reason |
|---|---|---|
| **Embedding** | Ticket types inside events | Always accessed together; improves read performance |
| **Embedding** | Payment details inside registrations | Preserves historical accuracy; avoids stale data |
| **Embedding** | Speaker info inside sessions | Frequently read together; avoids extra lookups |
| **Referencing** | `userId` and `eventId` in registrations | Shared across documents; changes independently |
| **Referencing** | `organizerId` in events | User profiles are managed separately |

### Data Types

| Type | Usage |
|---|---|
| `ObjectId` | `_id`, `userId`, `eventId` |
| `String` | names, email, role, status |
| `Number` | ticket price, quota, payment amount |
| `Boolean` | `isPublished`, notification preferences |
| `Date` | event dates, payment date, registration time |
| `Object` | embedded documents (payment, speaker, preferences) |
| `Array` | ticket types list, session categories |

---

## CRUD Operations

### Insert

`insertMany()` was used to populate each collection with 10 records. Ticket types are embedded directly in event documents, and payment details are embedded in registrations while cross-referencing users and events via `userId` and `eventId`.

```js
db.users.insertMany([
  { _id: "user_01", name: "Muhammad Adam", email: "adam@gmail.com", role: "organizer", city: "Johor Bahru", phone: "012-3456789" },
  { _id: "user_02", name: "Afiq", email: "afiq@gmail.com", role: "organizer", city: "Kuala Lumpur", phone: "019-8765432" },
  // ... 8 more
])
```

### Query (Read)

The `find()` method is used with filter conditions and projections.

```js
// Filter events by location
db.events.find({ location: "Dataran Merdeka" })

// Filter registrations with PENDING status and payment under RM 100
db.registrations.find({
  $and: [
    { status: "PENDING" },
    { "payment.amount": { $lte: 100 } }
  ]
})

// Query with projection — show only relevant fields, hide _id
db.events.find(
  {
    location: "Kuala Lumpur",
    "ticketTypes": { $elemMatch: { type: "VIP", price: { $lte: 100 } } }
  },
  { title: 1, location: 1, ticketTypes: 1, _id: 0 }
)
```

### Update

Update operations use specific operators for targeted modifications:

```js
// $set — update a user's phone number
db.users.updateOne(
  { _id: "user_01" },
  { $set: { phone: "011-9999888" } }
)

// $push — append a new category to a session
db.sessions.updateOne(
  { _id: "ses_01" },
  { $push: { cat: "Technology" } }
)

// $inc — decrement ticket quantity atomically on booking
db.events.updateOne(
  { _id: "evt_01", "tickets.type": "VIP" },
  { $inc: { "tickets.$.qty": -1 } }
)

// updateMany — bulk status update
db.registrations.updateMany(
  { status: "PENDING" },
  { $set: { status: "PAID" } }
)
```

### Delete

```js
// deleteOne — remove a specific session
db.sessions.deleteOne({ _id: "ses_10" })

// deleteMany — clean up cancelled registrations
db.registrations.deleteMany({ status: "CANCELLED" })
```

---

## Advanced MongoDB Operations

### Indexing

Indexes are created to avoid full collection scans as data grows.

**Single-field index on `date` (events collection):**
```js
db.events.createIndex({ date: 1 })
// Returns: date_1
```
Improves performance for queries that filter or sort events chronologically. Without this, MongoDB must inspect every document to evaluate its date.

**Compound index on `event_id` + `status` (registrations collection):**
```js
db.registrations.createIndex({ event_id: 1, status: 1 })
// Returns: event_id_1_status_1
```
Optimizes multi-condition queries such as retrieving all PAID registrations for a specific event — a common administrative operation.

### Aggregation Pipelines

**Total registrations per event:**
```js
db.registrations.aggregate([
  {
    $group: {
      _id: "$event_id",
      totalRegistrations: { $sum: 1 }
    }
  }
])
```
Sample output: `evt_01: 2`, `evt_04: 2`, `evt_02: 2`

**Total revenue per event (PAID only):**
```js
db.registrations.aggregate([
  { $match: { status: "PAID" } },
  {
    $group: {
      _id: "$event_id",
      totalRevenue: { $sum: "$payment.amount" }
    }
  }
])
```
Sample output: `evt_01: 650`, `evt_10: 100`, `evt_04: 80`

**Registration status summary:**
```js
db.registrations.aggregate([
  {
    $group: {
      _id: "$status",
      count: { $sum: 1 }
    }
  }
])
```
Sample output: `PAID: 7`, `PENDING: 2`, `CANCELLED: 1`

### Sorting

```js
// Events sorted by date ascending (upcoming first)
db.events.find().sort({ date: 1 })

// Events sorted by date descending (most recent first)
db.events.find().sort({ date: -1 })

// Registrations sorted by payment amount descending (highest first)
db.registrations.find().sort({ "payment.amount": -1 })
```

---

## Security & Challenges

### Access Control

MongoDB's role-based access control (RBAC) should be configured with three roles:

| Role | Permissions |
|---|---|
| `admin` | Full access to all collections and system settings |
| `organizer` | Create and update events; read registrations for their events |
| `user` | Read events; create and read their own registrations |

### NoSQL Injection

Attackers can exploit query operators like `$or` or `$ne` to bypass authentication filters. Mitigation strategies:
- Validate and sanitize all user inputs before they reach the database
- Avoid passing raw request objects directly into queries
- Enforce fixed query structures in application logic

### Consistency Trade-offs

MongoDB in distributed environments (replica sets) uses an eventual consistency model. This can cause temporary inconsistencies such as a ticket appearing available after it has already been booked. Critical operations like ticket registration should use atomic updates (`$inc`) or multi-document transactions to prevent overbooking.

---

## Getting Started

**Prerequisites:** MongoDB 6.0+ and mongosh

1. Clone the repository:
   ```bash
   git clone https://github.com/<your-username>/event-management-nosql.git
   cd event-management-nosql
   ```

2. Start MongoDB and open mongosh:
   ```bash
   mongosh
   ```

3. Create the database and seed data:
   ```js
   use event_management
   load("seed/users.js")
   load("seed/events.js")
   load("seed/registrations.js")
   load("seed/sessions.js")
   ```

4. Apply indexes:
   ```js
   load("indexes.js")
   ```

5. Run aggregation demos:
   ```js
   load("aggregations.js")
   ```

---

## Project Structure

```
event-management-nosql/
├── seed/
│   ├── users.js             # insertMany for users collection
│   ├── events.js            # insertMany for events collection
│   ├── registrations.js     # insertMany for registrations collection
│   └── sessions.js          # insertMany for sessions collection
├── crud/
│   ├── insert.js            # Insert operation examples
│   ├── query.js             # find() with filters and projections
│   ├── update.js            # $set, $push, $inc operations
│   └── delete.js            # deleteOne and deleteMany examples
├── indexes.js               # Single-field and compound index creation
├── aggregations.js          # Revenue, registration count, status summary pipelines
├── sorting.js               # Sorting queries by date and payment amount
└── README.md
```
