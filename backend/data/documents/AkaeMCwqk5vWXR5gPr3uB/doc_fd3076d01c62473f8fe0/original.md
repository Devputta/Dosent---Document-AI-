# design.md
## System & UX Design — AEMS

## 1. Database Schema (Prisma)

```prisma
enum Role {
  STUDENT
  COORDINATOR
  FACULTY
  HOD
  ADMIN
}

enum EventStatus {
  DRAFT
  PENDING
  APPROVED
  LIVE
  REJECTED
  CANCELLED
  COMPLETED
}

enum EventCategory {
  GUEST_LECTURE
  DEPARTMENTAL_WORKSHOP
  CLUB_ACTIVITY
  CAMPUS_FEST
}

enum TicketStatus {
  ACTIVE
  CANCELLED
  ATTENDED
}

model Department {
  id        String   @id @default(uuid())
  name      String   @unique
  code      String   @unique   // e.g. "CSE"
  users     User[]
  events    Event[]  @relation("EventDepartments")
}

model User {
  id           String   @id @default(uuid())
  email        String   @unique
  name         String
  role         Role
  departmentId String?
  department   Department? @relation(fields: [departmentId], references: [id])
  semester     Int?          // relevant for STUDENT role
  createdAt    DateTime @default(now())

  eventsInCharge     Event[]  @relation("FacultyInCharge")
  coordinatorScopes  CoordinatorAssignment[]
  tickets            Ticket[]
  scannedLogs        AttendanceLog[] @relation("Scanner")
}

model CoordinatorAssignment {
  id           String   @id @default(uuid())
  userId       String
  user         User     @relation(fields: [userId], references: [id])
  departmentId String?  // scope: department OR club
  clubName     String?
  createdAt    DateTime @default(now())
}

model Event {
  id                 String   @id @default(uuid())
  title              String
  description        String
  category           EventCategory
  venue              String
  startsAt           DateTime
  endsAt             DateTime?
  capacity           Int?
  rsvpDeadline       DateTime?

  status             EventStatus @default(DRAFT)
  openToAll          Boolean  @default(false)
  targetSemesters    Int[]
  targetDepartments  Department[] @relation("EventDepartments")

  createdById        String
  createdBy          User     @relation("CreatedEvents", fields: [createdById], references: [id])
  facultyInChargeId  String
  facultyInCharge    User     @relation("FacultyInCharge", fields: [facultyInChargeId], references: [id])

  rejectionReason    String?
  approvedAt         DateTime?

  tickets            Ticket[]

  createdAt          DateTime @default(now())
  updatedAt           DateTime @updatedAt
}

model Ticket {
  id           String   @id @default(uuid())
  eventId      String
  event        Event    @relation(fields: [eventId], references: [id])
  studentId    String
  student      User     @relation(fields: [studentId], references: [id])
  qrHash       String   @unique   // hash of signed JWT payload
  status       TicketStatus @default(ACTIVE)
  attendedAt   DateTime?
  createdAt    DateTime @default(now())

  attendanceLogs AttendanceLog[]

  @@unique([eventId, studentId])   // R16: one active ticket per student per event
}

model AttendanceLog {
  id            String   @id @default(uuid())
  ticketId      String
  ticket        Ticket   @relation(fields: [ticketId], references: [id])
  scannedByUserId String
  scannedBy     User     @relation("Scanner", fields: [scannedByUserId], references: [id])
  result        String   // SUCCESS | DUPLICATE | INVALID
  timestamp     DateTime @default(now())
}
```

## 2. API Design (REST)

| Method | Route | Role | Purpose |
|---|---|---|---|
| POST | `/api/auth/signup` | Public (domain-restricted) | Create account |
| POST | `/api/auth/login` | Public | Session login |
| GET | `/api/events` | Student+ | List events, auto-filtered by dept/semester |
| POST | `/api/events` | Coordinator | Create DRAFT event |
| PATCH | `/api/events/:id/submit` | Coordinator (owner) | DRAFT → PENDING |
| PATCH | `/api/events/:id/approve` | Faculty (assigned) | PENDING → APPROVED |
| PATCH | `/api/events/:id/reject` | Faculty (assigned) | PENDING → REJECTED |
| GET | `/api/events/approve/:token` | Faculty (signed link) | One-click approval landing |
| POST | `/api/events/:id/rsvp` | Student | Create ticket + QR |
| DELETE | `/api/events/:id/rsvp` | Student | Cancel own RSVP |
| POST | `/api/tickets/verify` | Coordinator/Faculty (assigned) | Scan → mark attended |
| GET | `/api/events/:id/attendance` | Faculty/HOD | Export attendance CSV |
| GET | `/api/admin/departments` | Admin | Manage reference data |
| POST | `/api/admin/roles` | Admin/HOD | Grant coordinator/faculty roles |

`/api/tickets/verify` request/response contract:
```json
// Request
{ "qrPayload": "<jwt-string>", "eventId": "uuid" }

// Response (success)
{ "result": "SUCCESS", "student": "Jane D.", "ticketId": "uuid" }

// Response (duplicate)
{ "result": "DUPLICATE", "attendedAt": "2026-09-09T10:03:00Z" }

// Response (invalid)
{ "result": "INVALID", "reason": "signature_mismatch" }
```

## 3. UX / Screen Map

**Student**
- Home feed (filtered events, search/category chips)
- Event detail (RSVP button, capacity bar, targeting badge e.g. "Sem 6 CSE")
- My Passes (QR list, past attendance history)

**Student Coordinator**
- My Events (drafts, pending, live) with status chips
- Event editor (title, category, targeting picker, Faculty In-Charge select)
- Scanner (camera view, live success/duplicate/invalid feed, running count)

**Faculty**
- Pending Approvals queue (approve/reject/comment)
- My Events (events they're in charge of, attendance stats)
- One-click email approval landing page (token-based, no full login needed)

**HOD**
- Department-wide event board (all statuses)
- Analytics: RSVP vs attendance by category/coordinator
- Escalation queue (stalled approvals)

**Admin**
- Departments/Semesters config
- Role assignment console

## 4. Visual/UI Direction
- Clean, information-dense list views (students scan feeds quickly between
  classes) — card-based event listing with a colored left-border by category.
- Status chips use a consistent color language: PENDING (amber), APPROVED
  (blue), LIVE (green), REJECTED/CANCELLED (red/gray).
- Scanner screen is full-bleed camera view with a large success/fail toast
  and haptic-style color flash (green/red) for fast door-line usability.
- Mobile-first: majority of student and coordinator usage will be on phones.

## 5. Security & Edge Case Design
- QR JWTs are short-lived is not appropriate here (pass must remain valid
  until the event); instead, validity is controlled by `Ticket.status`, not
  token expiry — expiry is set generously (event end + buffer) purely as a
  defense-in-depth backstop.
- Scanner endpoint checks `event.status == LIVE` at scan time — no
  check-ins accepted for CANCELLED events even if a stale QR is presented.
- Duplicate scans return HTTP 409 with the original `attendedAt`, not a
  generic error, so coordinators can explain to the student.
