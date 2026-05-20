# Appointment Scheduling API and DB Design

## Problem Statement

Design the API and database schema for an appointment scheduling system where users book slots with providers (e.g., doctors, consultants, salons).

**Core requirements:**
- Define provider availability windows
- Search and book open time slots
- Prevent double-booking
- Support cancel/reschedule with policy checks

**Scale:** 100K providers, 20M bookings/month, bursty daytime traffic

---

## API Design

### Core Endpoints
```http
POST /api/v1/providers/{providerId}/availability
GET /api/v1/providers/{providerId}/slots?date=2026-05-21
POST /api/v1/appointments
PATCH /api/v1/appointments/{appointmentId}/reschedule
POST /api/v1/appointments/{appointmentId}/cancel
GET /api/v1/users/{userId}/appointments?status=UPCOMING
```

### Example: Book Appointment
```json
POST /api/v1/appointments
{
  "providerId": "pro_101",
  "userId": "usr_202",
  "slotStart": "2026-05-21T10:30:00Z",
  "slotEnd": "2026-05-21T11:00:00Z",
  "idempotencyKey": "1dd95b08-..."
}
```

```json
201 Created
{
  "appointmentId": "apt_303",
  "status": "CONFIRMED"
}
```

### Design Decisions
- Booking API must be transactional and idempotent
- Slot reads can be cached briefly; booking writes always hit primary DB
- Optional waitlist endpoint for full schedules

---

## Database Schema

```sql
CREATE TABLE provider_availability (
    id                  BIGSERIAL PRIMARY KEY,
    provider_id         VARCHAR(64) NOT NULL,
    start_time          TIMESTAMPTZ NOT NULL,
    end_time            TIMESTAMPTZ NOT NULL,
    recurrence_rule     VARCHAR(200), -- optional RRULE
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_availability_provider_time ON provider_availability (provider_id, start_time, end_time);

CREATE TABLE appointments (
    id                  VARCHAR(64) PRIMARY KEY,
    provider_id         VARCHAR(64) NOT NULL,
    user_id             VARCHAR(64) NOT NULL,
    slot_start          TIMESTAMPTZ NOT NULL,
    slot_end            TIMESTAMPTZ NOT NULL,
    status              VARCHAR(24) NOT NULL, -- CONFIRMED, CANCELED, COMPLETED
    cancel_reason       VARCHAR(200),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Prevent overlapping confirmed appointments for same provider and slot
CREATE UNIQUE INDEX uq_provider_slot
ON appointments (provider_id, slot_start, slot_end)
WHERE status = 'CONFIRMED';

CREATE INDEX idx_appointments_user_time ON appointments (user_id, slot_start DESC);
CREATE INDEX idx_appointments_provider_time ON appointments (provider_id, slot_start);

CREATE TABLE booking_idempotency (
    idempotency_key     VARCHAR(128) PRIMARY KEY,
    appointment_id      VARCHAR(64) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Trade-offs and Considerations

- Strict DB constraints are safest against race conditions.
- Time-zone handling should normalize to UTC and render in local time.
- Reschedule can be modeled as update-in-place or cancel+rebook (better audit trail).
