# Project Requirements Document (PRD)
## Academic Lifecycle & Event Management System (AEMS)

## 1. Overview
AEMS is a college-specific event management platform that replaces generic event
aggregation with a role-aware, semester-scoped, approval-gated workflow. It
connects Student Coordinators, Faculty In-Charges, and Students around the full
lifecycle of an academic or co-curricular event: proposal → approval →
publication → RSVP/ticketing → door check-in → attendance record.

## 2. Goals
- G1: Give every event an accountable Faculty owner before it becomes visible.
- G2: Target events precisely by department + semester so students only see
  what's relevant to them.
- G3: Replace manual attendance sheets with a signed QR pass + live scanner.
- G4: Produce an auditable attendance trail usable for credits/certificates.
- G5: Ship an MVP a small team can build and demo within one semester.

## 3. Non-Goals (v1)
- Payment/paid ticketing.
- Native mobile apps (web-only, mobile-responsive).
- Multi-college / multi-tenant support (single institution per deployment).
- Automated timetable/course integration.

## 4. User Roles
| Role | Description |
|---|---|
| **Student (Attendee)** | Institutional-email user. Browses events filtered to their department/semester, RSVPs, holds QR passes, views attendance history. |
| **Student Coordinator** | A student granted elevated permissions for a specific club/department. Drafts events, manages logistics, runs the door scanner. |
| **Faculty In-Charge / Professor** | Approves/rejects event proposals, signs off on budget notes, is the accountable owner of an event. |
| **HOD (Head of Department)** | Departmental oversight; can view all department events/analytics and override/escalate approvals. |
| **Admin** | Manages departments, semesters, role assignments, and system configuration. |

## 5. Functional Requirements

### 5.1 Authentication & Identity
- FR-1: Users must sign up/log in using an institutional email domain (e.g. `@college.edu`).
- FR-2: Each user has exactly one primary role at a time, with department and
  (for students) current semester attached to their profile.
- FR-3: Admin can promote/demote a Student to Student Coordinator for a scope
  (club or department), and assign Faculty/HOD roles.

### 5.2 Event Lifecycle
- FR-4: A Student Coordinator can create an event draft with: title,
  description, category (Guest Lecture / Departmental Workshop / Club
  Activity / Campus Fest), date/time, venue, capacity, target department(s),
  target semester(s), and a designated Faculty In-Charge.
- FR-5: On submission, the event enters `PENDING` status and the assigned
  Faculty In-Charge is notified (email + dashboard).
- FR-6: Faculty In-Charge can Approve, Request Changes, or Reject. Approval
  moves the event to `APPROVED` → auto-published as `LIVE` at a configured
  time (or immediately).
- FR-7: Faculty approval can be granted via a one-click secure link (OTP/token)
  without requiring a full dashboard session, or via the dashboard.
- FR-8: HOD can view all departmental events regardless of status and can
  escalate/override a stalled approval.

### 5.3 Discovery & Targeting
- FR-9: Students only see `LIVE` events whose target department/semester
  matches their profile, plus any event explicitly marked "Open to All."
- FR-10: Students can filter/search by category, date range, and department.
- FR-11: Events display remaining capacity and RSVP deadline (if any).

### 5.4 RSVP & Ticketing
- FR-12: A student can RSVP to a `LIVE` event they are eligible for, subject
  to capacity.
- FR-13: On RSVP, the system generates a unique ticket containing a signed
  QR payload (JWT: studentId, eventId, ticketId, issuedAt).
- FR-14: The QR pass is emailed to the student and viewable in-app.
- FR-15: A student cannot hold two active tickets for the same event.

### 5.5 Check-In / Attendance
- FR-16: Student Coordinators (assigned to the event) can open a web-based
  scanner using the device camera.
- FR-17: Scanning a QR sends the payload to `POST /api/tickets/verify`,
  which validates the signature, checks ticket state, and marks it
  `ATTENDED` with a timestamp — idempotently (a second scan is rejected as
  "already checked in").
- FR-18: Every scan is written to an Attendance Log with scanning user id
  and timestamp for audit purposes.
- FR-19: Faculty/HOD can export attendance as CSV/PDF for credit or
  certificate processing.

### 5.6 Notifications
- FR-20: System sends email notifications for: proposal submitted, event
  approved/rejected, RSVP confirmation + QR pass, event reminder (T-24h),
  post-event certificate (optional).

## 6. Non-Functional Requirements
- NFR-1 (Security): QR payloads must be signed (JWT, HS256/RS256) and
  verified server-side on every scan; tickets cannot be forged or replayed.
- NFR-2 (Performance): Scanner verify endpoint must respond in <500ms under
  normal load to support door-line throughput.
- NFR-3 (Availability): Core RSVP/scan flows should degrade gracefully
  offline-first is out of scope for v1, but scanner should retry on
  connectivity loss.
- NFR-4 (Auditability): Attendance logs are append-only; no destructive
  edits, only status transitions.
- NFR-5 (Privacy): Student personal data limited to what's required;
  department/semester used only for targeting and analytics.
- NFR-6 (Accessibility): Public event listing pages must be responsive and
  usable on low-end Android devices (majority of campus traffic).
- NFR-7 (RBAC integrity): Every API route must enforce role- and scope-based
  authorization server-side, not just hide UI elements client-side.

## 7. Success Metrics
- % of events with a Faculty In-Charge assigned before going live: 100%.
- Average approval turnaround time: < 24 hours.
- Check-in scan success rate: > 98% (valid tickets scanned without manual
  fallback).
- Reduction in manual attendance-sheet usage across pilot department: > 80%.

## 8. Open Questions
- Do certificates require a signed PDF with a verifiable hash?
- Should HODs be able to force-approve without Faculty sign-off in
  emergencies?
- Is cross-department co-hosting (multiple Faculty In-Charges) needed in v1?
