# Job Board API and DB Design

## System Overview
A job board platform (think LinkedIn Jobs or Indeed) where employers post job listings and candidates search, filter, and apply. The system manages the full application lifecycle from submission through hiring decision, with role-based access for employers and candidates.

## 1. Requirements

### Functional Requirements
- Employers post job listings with requirements, salary range, and location
- Candidates search jobs by keyword, location, salary, job type, and experience level
- Candidates submit applications with resume and cover letter
- Employers review applications and move them through hiring stages
- Candidates track application status across all their applications
- Saved jobs and job alerts for candidates
- Company profiles with employer branding

### Non-Functional Requirements
- Availability: 99.9%
- Search latency: <300ms for job search with filters
- Read-heavy: 100:1 search-to-apply ratio
- Scalability: 10M job listings, 50M candidates, 1M applications/day
- Privacy: Candidates control visibility of their profiles to employers

## 2. Scale Estimation

```
Active job listings     = 10M
Candidates              = 50M
Applications/day        = 1M
Job searches/day        = 100M
Searches/sec (avg)      = 100M / 86400 ≈ 1,160/sec
Peak searches/sec       ≈ 5,000/sec

Job listing size        = 5KB
Application size        = 50KB (includes resume reference)
Application storage/yr  = 1M × 365 × 50KB ≈ 18TB/year
```

## 3. API Design

### Key Endpoints

#### Post a Job
```
POST /api/v1/jobs
Authorization: Bearer <token>  (employer role)

Request:
{
  "title": "Senior Backend Engineer",
  "description": "We are looking for...",
  "requirements": ["5+ years Go", "PostgreSQL", "Kubernetes"],
  "job_type": "FULL_TIME",
  "location": "San Francisco, CA",
  "is_remote": true,
  "salary_min": 150000,
  "salary_max": 200000,
  "currency": "USD",
  "experience_level": "SENIOR",
  "expires_at": "2025-08-01"
}

Response 201:
{
  "job_id": "job_abc",
  "status": "ACTIVE",
  "url": "https://jobs.example.com/jobs/job_abc"
}
```

#### Search Jobs
```
GET /api/v1/jobs/search?q=backend+engineer&location=San+Francisco&remote=true&salary_min=120000&experience=SENIOR&page=1

Response 200:
{
  "jobs": [
    {
      "job_id": "job_abc",
      "title": "Senior Backend Engineer",
      "company": { "id": "co_xyz", "name": "Acme Corp", "logo_url": "..." },
      "location": "San Francisco, CA",
      "is_remote": true,
      "salary_range": "$150K–$200K",
      "job_type": "FULL_TIME",
      "posted_at": "2025-06-01T00:00:00Z",
      "application_count": 47
    }
  ],
  "total": 312,
  "page": 1
}
```

#### Apply to Job
```
POST /api/v1/jobs/{jobId}/apply
Authorization: Bearer <token>  (candidate role)

Request:
{
  "resume_id": "res_abc",
  "cover_letter": "I am excited to apply...",
  "answers": [
    { "question_id": "q_001", "answer": "5 years" }
  ]
}

Response 201:
{
  "application_id": "app_xyz",
  "job_id": "job_abc",
  "status": "SUBMITTED",
  "submitted_at": "2025-06-01T10:00:00Z"
}

Errors:
- 409: Already applied to this job
```

#### Update Application Status (Employer)
```
PATCH /api/v1/applications/{applicationId}/status
Authorization: Bearer <token>  (employer role)

Request:
{
  "status": "INTERVIEW_SCHEDULED",
  "notes": "Phone screen scheduled for June 5th"
}

Response 200: { "application_id": "app_xyz", "status": "INTERVIEW_SCHEDULED" }
```

#### Get Candidate's Applications
```
GET /api/v1/applications?status=active

Response 200:
{
  "applications": [
    {
      "application_id": "app_xyz",
      "job": { "title": "Senior Backend Engineer", "company": "Acme Corp" },
      "status": "INTERVIEW_SCHEDULED",
      "submitted_at": "2025-06-01T10:00:00Z",
      "last_updated_at": "2025-06-03T14:00:00Z"
    }
  ]
}
```

### Endpoint Summary

| Method | Path | Description |
|--------|------|-------------|
| POST | /api/v1/jobs | Post a job listing |
| GET | /api/v1/jobs/search | Search jobs |
| GET | /api/v1/jobs/{jobId} | Get job details |
| PATCH | /api/v1/jobs/{jobId} | Update job listing |
| DELETE | /api/v1/jobs/{jobId} | Close/delete job |
| POST | /api/v1/jobs/{jobId}/apply | Apply to job |
| GET | /api/v1/applications | List candidate's applications |
| GET | /api/v1/jobs/{jobId}/applications | List job's applications (employer) |
| PATCH | /api/v1/applications/{id}/status | Update application status |
| POST | /api/v1/jobs/{jobId}/save | Save job for later |

## 4. Database Schema

```sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    full_name   VARCHAR(255) NOT NULL,
    role        VARCHAR(20) NOT NULL,   -- CANDIDATE, EMPLOYER, ADMIN
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE companies (
    id              BIGSERIAL PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    website         VARCHAR(500),
    logo_url        VARCHAR(500),
    industry        VARCHAR(100),
    size_range      VARCHAR(50),   -- "1-10", "11-50", "51-200", "201-1000", "1000+"
    headquarters    VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE company_members (
    company_id  BIGINT NOT NULL REFERENCES companies(id),
    user_id     BIGINT NOT NULL REFERENCES users(id),
    role        VARCHAR(20) NOT NULL DEFAULT 'RECRUITER',  -- ADMIN, RECRUITER
    PRIMARY KEY (company_id, user_id)
);

CREATE TABLE jobs (
    id                  BIGSERIAL PRIMARY KEY,
    company_id          BIGINT NOT NULL REFERENCES companies(id),
    posted_by           BIGINT NOT NULL REFERENCES users(id),
    title               VARCHAR(255) NOT NULL,
    description         TEXT NOT NULL,
    requirements        TEXT[],
    job_type            VARCHAR(30) NOT NULL,
                        -- FULL_TIME, PART_TIME, CONTRACT, INTERNSHIP, FREELANCE
    experience_level    VARCHAR(20),
                        -- ENTRY, MID, SENIOR, LEAD, EXECUTIVE
    location            VARCHAR(255),
    is_remote           BOOLEAN NOT NULL DEFAULT FALSE,
    salary_min          INT,
    salary_max          INT,
    currency            CHAR(3) DEFAULT 'USD',
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
                        -- DRAFT, ACTIVE, PAUSED, CLOSED, EXPIRED
    application_count   INT NOT NULL DEFAULT 0,
    view_count          INT NOT NULL DEFAULT 0,
    expires_at          DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_jobs_company_id ON jobs (company_id);
CREATE INDEX idx_jobs_status ON jobs (status, created_at DESC);
CREATE INDEX idx_jobs_location ON jobs (location) WHERE status = 'ACTIVE';
CREATE INDEX idx_jobs_salary ON jobs (salary_min, salary_max) WHERE status = 'ACTIVE';
CREATE INDEX idx_jobs_remote ON jobs (is_remote) WHERE status = 'ACTIVE';
-- Full-text search on title + description
CREATE INDEX idx_jobs_fts ON jobs
    USING GIN (to_tsvector('english', title || ' ' || description));

CREATE TABLE job_questions (
    id          BIGSERIAL PRIMARY KEY,
    job_id      BIGINT NOT NULL REFERENCES jobs(id) ON DELETE CASCADE,
    question    TEXT NOT NULL,
    is_required BOOLEAN NOT NULL DEFAULT FALSE,
    position    INT NOT NULL DEFAULT 0
);

CREATE TABLE candidate_profiles (
    user_id         BIGINT PRIMARY KEY REFERENCES users(id),
    headline        VARCHAR(255),
    summary         TEXT,
    location        VARCHAR(255),
    years_experience INT,
    is_open_to_work BOOLEAN NOT NULL DEFAULT TRUE,
    profile_visibility VARCHAR(20) NOT NULL DEFAULT 'PUBLIC'
                    -- PUBLIC, EMPLOYERS_ONLY, PRIVATE
);

CREATE TABLE resumes (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL REFERENCES users(id),
    filename    VARCHAR(255) NOT NULL,
    url         VARCHAR(500) NOT NULL,
    is_default  BOOLEAN NOT NULL DEFAULT FALSE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE applications (
    id              BIGSERIAL PRIMARY KEY,
    job_id          BIGINT NOT NULL REFERENCES jobs(id),
    candidate_id    BIGINT NOT NULL REFERENCES users(id),
    resume_id       BIGINT REFERENCES resumes(id),
    cover_letter    TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'SUBMITTED',
                    -- SUBMITTED, UNDER_REVIEW, INTERVIEW_SCHEDULED,
                    -- OFFER_EXTENDED, HIRED, REJECTED, WITHDRAWN
    rejection_reason VARCHAR(100),
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (job_id, candidate_id)   -- one application per job per candidate
);

CREATE INDEX idx_applications_job_id ON applications (job_id, submitted_at DESC);
CREATE INDEX idx_applications_candidate_id ON applications (candidate_id);
CREATE INDEX idx_applications_status ON applications (status);

CREATE TABLE application_answers (
    id              BIGSERIAL PRIMARY KEY,
    application_id  BIGINT NOT NULL REFERENCES applications(id),
    question_id     BIGINT NOT NULL REFERENCES job_questions(id),
    answer          TEXT NOT NULL
);

-- Application status history for audit trail
CREATE TABLE application_status_history (
    id              BIGSERIAL PRIMARY KEY,
    application_id  BIGINT NOT NULL REFERENCES applications(id),
    old_status      VARCHAR(30),
    new_status      VARCHAR(30) NOT NULL,
    changed_by      BIGINT REFERENCES users(id),
    notes           TEXT,
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_app_history_application_id ON application_status_history (application_id);

CREATE TABLE saved_jobs (
    user_id     BIGINT NOT NULL REFERENCES users(id),
    job_id      BIGINT NOT NULL REFERENCES jobs(id),
    saved_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, job_id)
);
```

### Schema Summary

| Table | Purpose |
|-------|---------|
| companies | Employer company profiles |
| jobs | Job listings with search indexes |
| applications | Candidate applications (one per job per candidate) |
| application_status_history | Audit trail of status changes |
| candidate_profiles | Candidate profile and visibility settings |
| resumes | Resume file references |
| saved_jobs | Candidate bookmarks |

## 5. Key Design Decisions

### 5.1 One Application Per Job Per Candidate

The `UNIQUE (job_id, candidate_id)` constraint prevents duplicate applications at the database level. The API returns 409 if a candidate tries to apply twice:

```sql
INSERT INTO applications (job_id, candidate_id, resume_id, cover_letter)
VALUES ($1, $2, $3, $4)
ON CONFLICT (job_id, candidate_id) DO NOTHING
RETURNING id;
-- NULL returned = already applied
```

### 5.2 Full-Text Search vs. Elasticsearch

PostgreSQL GIN index on `to_tsvector('english', title || ' ' || description)` handles moderate search load. For production scale (100M searches/day), sync to Elasticsearch via CDC for:
- Faceted filtering (salary range, job type, experience level)
- Geo-distance search for location
- Relevance ranking with custom scoring

The PostgreSQL schema remains the source of truth; Elasticsearch is the search index.

### 5.3 Application Status Machine

```
SUBMITTED → UNDER_REVIEW → INTERVIEW_SCHEDULED → OFFER_EXTENDED → HIRED
                         ↓                      ↓
                      REJECTED               REJECTED
SUBMITTED | UNDER_REVIEW | INTERVIEW_SCHEDULED → WITHDRAWN (by candidate)
```

All transitions are logged in `application_status_history`. Employers can only move forward; candidates can only withdraw.

### 5.4 Denormalized Application Count

`jobs.application_count` is denormalized for fast display in search results. Updated atomically when applications are submitted or withdrawn:
```sql
UPDATE jobs SET application_count = application_count + 1 WHERE id = $1;
```

### 5.5 Candidate Profile Visibility

`profile_visibility` controls whether employers can see a candidate's profile when they apply:
- `PUBLIC`: Anyone can view
- `EMPLOYERS_ONLY`: Only companies with active job postings can view
- `PRIVATE`: Only visible to employers the candidate has applied to

Enforced at the API layer — queries filter based on the requesting user's relationship to the candidate.

## 6. Failure Scenarios

### Duplicate Application on Retry
- **Impact**: Candidate applies twice to the same job
- **Recovery**: `UNIQUE (job_id, candidate_id)` constraint prevents DB duplicates; API returns 409 with existing application details
- **Prevention**: Client-side deduplication; server-side unique constraint as final guard

### Job Expires with Active Applications
- **Impact**: Job expires but applications are still in progress
- **Recovery**: Applications remain in their current status; employers can still process them; job status changes to `EXPIRED` but applications are unaffected
- **Prevention**: Notify employers 7 days before expiry; auto-extend if applications are in late stages

### Resume File Deleted
- **Impact**: Application references a resume that no longer exists in object storage
- **Recovery**: Soft-delete resumes; mark as `is_deleted` but keep the record; applications retain the reference
- **Prevention**: Never hard-delete resumes that are referenced by applications; check for references before deletion

### Search Index Out of Sync
- **Impact**: Job appears in search after it's closed, or doesn't appear after posting
- **Recovery**: CDC ensures Elasticsearch is updated within seconds; manual re-index for specific jobs
- **Prevention**: Invalidate search cache on job status change; TTL on cached search results
