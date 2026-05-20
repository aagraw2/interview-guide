# Library Management API and DB Design

## System Overview
A library management system where members borrow physical books, librarians manage the catalog and inventory, and the system enforces borrowing rules (loan limits, due dates, reservations). The interesting challenge is managing a reservation queue when all copies of a book are checked out.

## 1. Requirements

### Functional Requirements
- Members can search the catalog and view book availability
- Members can borrow books (up to 5 at a time, 21-day loan period)
- Members can reserve a book when all copies are checked out
- Automatic fine calculation for overdue books ($0.25/day)
- Librarians can add/update books and manage inventory
- Renewal of loans (up to 2 renewals, unless reserved by another member)
- Email notifications for due dates, reservations available, and fines

### Non-Functional Requirements
- Availability: 99.9%
- Consistency: Strong — a book copy can only be checked out to one member at a time
- Latency: <200ms for catalog search; <500ms for checkout
- Scalability: 100K members, 500K books, 1M loans/year

## 2. Scale Estimation

```
Members                 = 100K
Books (titles)          = 200K
Book copies             = 500K (avg 2.5 copies per title)
Active loans            = 50K (10% of copies checked out)
Loans/day               = 5K
Reservations/day        = 1K

Loan record size        = ~500B
Annual loan storage     = 5K × 365 × 500B ≈ 900MB/year (small scale)
```

## 3. API Design

### Key Endpoints

#### Search Catalog
```
GET /api/v1/catalog/search?q=harry+potter&author=rowling&available=true

Response 200:
{
  "books": [
    {
      "isbn": "978-0439708180",
      "title": "Harry Potter and the Sorcerer's Stone",
      "author": "J.K. Rowling",
      "year": 1997,
      "total_copies": 3,
      "available_copies": 1,
      "next_available_date": null
    }
  ],
  "total": 1
}
```

#### Checkout Book
```
POST /api/v1/loans
Authorization: Bearer <token>  (librarian role)

Request:
{
  "member_id": "mem_abc",
  "copy_id": "copy_xyz"
}

Response 201:
{
  "loan_id": "loan_123",
  "member_id": "mem_abc",
  "book_title": "Harry Potter and the Sorcerer's Stone",
  "copy_barcode": "LIB-001-042",
  "due_date": "2025-06-22",
  "renewals_remaining": 2
}

Errors:
- 409: Member has reached maximum loan limit (5)
- 409: Copy is not available
- 409: Member has unpaid fines above threshold
```

#### Return Book
```
POST /api/v1/loans/{loanId}/return
Authorization: Bearer <token>  (librarian role)

Response 200:
{
  "loan_id": "loan_123",
  "returned_at": "2025-06-20T14:00:00Z",
  "days_overdue": 0,
  "fine_amount": 0.00
}
```

#### Renew Loan
```
POST /api/v1/loans/{loanId}/renew
Authorization: Bearer <token>

Response 200:
{
  "loan_id": "loan_123",
  "new_due_date": "2025-07-13",
  "renewals_remaining": 1
}

Errors:
- 409: No renewals remaining
- 409: Book is reserved by another member
```

#### Reserve Book
```
POST /api/v1/reservations
Authorization: Bearer <token>

Request: { "isbn": "978-0439708180" }

Response 201:
{
  "reservation_id": "res_abc",
  "isbn": "978-0439708180",
  "queue_position": 3,
  "estimated_available_date": "2025-07-01"
}
```

#### Get Member Account
```
GET /api/v1/members/{memberId}/account

Response 200:
{
  "member_id": "mem_abc",
  "active_loans": 3,
  "loans_remaining": 2,
  "overdue_loans": 0,
  "outstanding_fines": 0.00,
  "active_reservations": 1
}
```

### Endpoint Summary

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/catalog/search | Search book catalog |
| GET | /api/v1/catalog/{isbn} | Get book details and availability |
| POST | /api/v1/loans | Checkout a book copy |
| POST | /api/v1/loans/{loanId}/return | Return a book |
| POST | /api/v1/loans/{loanId}/renew | Renew a loan |
| GET | /api/v1/members/{memberId}/loans | Member's loan history |
| POST | /api/v1/reservations | Reserve a book |
| DELETE | /api/v1/reservations/{id} | Cancel reservation |
| GET | /api/v1/members/{memberId}/account | Account summary with fines |

## 4. Database Schema

```sql
CREATE TABLE members (
    id              BIGSERIAL PRIMARY KEY,
    library_card_number VARCHAR(20) NOT NULL UNIQUE,
    email           VARCHAR(255) NOT NULL UNIQUE,
    full_name       VARCHAR(255) NOT NULL,
    phone           VARCHAR(30),
    status          VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
                    -- ACTIVE, SUSPENDED, EXPIRED
    max_loans       INT NOT NULL DEFAULT 5,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Book titles (bibliographic record)
CREATE TABLE books (
    isbn            VARCHAR(20) PRIMARY KEY,
    title           VARCHAR(500) NOT NULL,
    author          VARCHAR(255) NOT NULL,
    publisher       VARCHAR(255),
    published_year  INT,
    genre           VARCHAR(100),
    description     TEXT,
    cover_url       VARCHAR(500),
    total_copies    INT NOT NULL DEFAULT 0,
    available_copies INT NOT NULL DEFAULT 0  -- denormalized
);

CREATE INDEX idx_books_author ON books (author);
CREATE INDEX idx_books_genre ON books (genre);
CREATE INDEX idx_books_fts ON books
    USING GIN (to_tsvector('english', title || ' ' || author));

-- Physical copies of books
CREATE TABLE book_copies (
    id          BIGSERIAL PRIMARY KEY,
    isbn        VARCHAR(20) NOT NULL REFERENCES books(isbn),
    barcode     VARCHAR(50) NOT NULL UNIQUE,
    condition   VARCHAR(20) NOT NULL DEFAULT 'GOOD',
                -- NEW, GOOD, FAIR, POOR, WITHDRAWN
    status      VARCHAR(20) NOT NULL DEFAULT 'AVAILABLE',
                -- AVAILABLE, CHECKED_OUT, RESERVED, LOST, WITHDRAWN
    location    VARCHAR(100),   -- shelf location, e.g., "A-12-3"
    acquired_at DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE INDEX idx_book_copies_isbn ON book_copies (isbn);
CREATE INDEX idx_book_copies_status ON book_copies (isbn, status);

CREATE TABLE loans (
    id              BIGSERIAL PRIMARY KEY,
    member_id       BIGINT NOT NULL REFERENCES members(id),
    copy_id         BIGINT NOT NULL REFERENCES book_copies(id),
    isbn            VARCHAR(20) NOT NULL REFERENCES books(isbn),
    status          VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
                    -- ACTIVE, RETURNED, OVERDUE, LOST
    checked_out_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    due_date        DATE NOT NULL,
    returned_at     TIMESTAMPTZ,
    renewals_used   INT NOT NULL DEFAULT 0,
    max_renewals    INT NOT NULL DEFAULT 2
);

CREATE INDEX idx_loans_member_id ON loans (member_id);
CREATE INDEX idx_loans_copy_id ON loans (copy_id);
CREATE INDEX idx_loans_due_date ON loans (due_date) WHERE status = 'ACTIVE';
CREATE INDEX idx_loans_status ON loans (status);

-- Reservation queue (FIFO per ISBN)
CREATE TABLE reservations (
    id              BIGSERIAL PRIMARY KEY,
    member_id       BIGINT NOT NULL REFERENCES members(id),
    isbn            VARCHAR(20) NOT NULL REFERENCES books(isbn),
    status          VARCHAR(20) NOT NULL DEFAULT 'WAITING',
                    -- WAITING, READY, FULFILLED, CANCELLED, EXPIRED
    queue_position  INT NOT NULL,
    reserved_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ready_at        TIMESTAMPTZ,    -- when a copy became available for this member
    expires_at      TIMESTAMPTZ,    -- member has 3 days to pick up when READY
    UNIQUE (member_id, isbn, status)  -- one active reservation per member per book
);

CREATE INDEX idx_reservations_isbn ON reservations (isbn, queue_position)
    WHERE status = 'WAITING';
CREATE INDEX idx_reservations_member_id ON reservations (member_id);

-- Fines for overdue books
CREATE TABLE fines (
    id              BIGSERIAL PRIMARY KEY,
    loan_id         BIGINT NOT NULL REFERENCES loans(id),
    member_id       BIGINT NOT NULL REFERENCES members(id),
    amount          DECIMAL(8, 2) NOT NULL,
    days_overdue    INT NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'OUTSTANDING',
                    -- OUTSTANDING, PAID, WAIVED
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    paid_at         TIMESTAMPTZ
);

CREATE INDEX idx_fines_member_id ON fines (member_id, status);
CREATE INDEX idx_fines_loan_id ON fines (loan_id);
```

### Schema Summary

| Table | Purpose |
|-------|---------|
| members | Library card holders |
| books | Bibliographic catalog (one row per title) |
| book_copies | Physical copies with barcode and status |
| loans | Checkout records with due dates |
| reservations | FIFO queue per book title |
| fines | Overdue fines with payment status |

## 5. Key Design Decisions

### 5.1 Checkout Concurrency — Preventing Double-Checkout

Two librarians could attempt to check out the same copy simultaneously. `SELECT FOR UPDATE` prevents this:

```sql
BEGIN;

-- Lock the copy row
SELECT id, status FROM book_copies WHERE id = $1 FOR UPDATE;

-- Validate: status = AVAILABLE
-- Validate: member active loans < max_loans
-- Validate: member has no blocking fines

UPDATE book_copies SET status = 'CHECKED_OUT' WHERE id = $1;

INSERT INTO loans (member_id, copy_id, isbn, due_date)
VALUES ($2, $1, $3, CURRENT_DATE + INTERVAL '21 days');

UPDATE books SET available_copies = available_copies - 1 WHERE isbn = $3;

COMMIT;
```

### 5.2 Reservation Queue Management

When a book is returned, the system checks for waiting reservations:

```sql
-- Find the next waiting reservation (FIFO by reserved_at)
SELECT id, member_id FROM reservations
WHERE isbn = $1 AND status = 'WAITING'
ORDER BY queue_position ASC
LIMIT 1;

-- If found: mark reservation READY, set expires_at = NOW() + 3 days
-- Notify member that their book is available
-- Mark the returned copy as RESERVED (not AVAILABLE)
```

Queue positions are assigned at reservation time using `MAX(queue_position) + 1` within a transaction.

### 5.3 Fine Calculation

Fines are calculated at return time (not continuously):

```sql
SELECT GREATEST(0, CURRENT_DATE - due_date) AS days_overdue
FROM loans WHERE id = $1;

-- fine_amount = days_overdue × 0.25
```

A background job also runs nightly to flag overdue loans and calculate accruing fines for reporting, but the authoritative fine is calculated at return.

### 5.4 Renewal Blocking by Reservation

A loan cannot be renewed if the book has waiting reservations:

```sql
SELECT COUNT(*) FROM reservations
WHERE isbn = $1 AND status = 'WAITING';
-- If count > 0 → deny renewal
```

This ensures reserved books return to the queue promptly.

### 5.5 Denormalized Available Copies

`books.available_copies` is denormalized for fast catalog display. Updated atomically on checkout and return. A nightly reconciliation job verifies it matches the actual count:

```sql
UPDATE books b SET available_copies = (
    SELECT COUNT(*) FROM book_copies bc
    WHERE bc.isbn = b.isbn AND bc.status = 'AVAILABLE'
);
```

## 6. Failure Scenarios

### Double-Checkout of Same Copy
- **Impact**: Two members assigned the same physical book
- **Recovery**: `SELECT FOR UPDATE` on `book_copies` prevents this; only one transaction succeeds
- **Prevention**: Always lock the copy row before checkout; never use optimistic locking for physical item checkout

### Reservation Queue Corruption
- **Impact**: Queue positions become non-sequential; wrong member notified
- **Recovery**: Recalculate queue positions from `reserved_at` ordering
- **Prevention**: Assign queue positions within a transaction; use `reserved_at` as tiebreaker

### Member Exceeds Loan Limit Due to Race Condition
- **Impact**: Member checks out 6 books when limit is 5
- **Recovery**: Count active loans within the same `SELECT FOR UPDATE` transaction
- **Prevention**: Check `COUNT(*) FROM loans WHERE member_id = $1 AND status = 'ACTIVE'` inside the locked transaction

### Fine Calculation on System Downtime
- **Impact**: System was down; due dates passed without fine calculation
- **Recovery**: Nightly job recalculates all overdue loans; fines are based on actual return date, not system processing date
- **Prevention**: Fine is calculated at return time from the actual due date; downtime doesn't affect correctness
