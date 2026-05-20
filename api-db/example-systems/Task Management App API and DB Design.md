# Task Management App API and DB Design

## System Overview
A collaborative task management application (think Asana or Linear) where users create projects, organize tasks in hierarchies, assign work to team members, and track progress in real time. Concurrent edits to the same task by multiple users require careful conflict handling.

## 1. Requirements

### Functional Requirements
- Create and manage projects with members
- Create tasks with title, description, assignee, due date, and priority
- Organize tasks hierarchically (subtasks under parent tasks)
- Move tasks through status stages (To Do → In Progress → Done)
- Comment on tasks
- Real-time updates when collaborators edit tasks
- Search tasks by title, assignee, status, or due date

### Non-Functional Requirements
- Availability: 99.9% — task data must always be accessible
- Consistency: Optimistic locking to handle concurrent edits without blocking
- Latency: <200ms for task reads; <500ms for writes
- Real-time: WebSocket push for collaborators within the same project
- Scalability: 10M users, 100M tasks, 1M active projects

## 2. Scale Estimation

```
DAU                   = 2M active users
Tasks created/day     = 5M
Task updates/day      = 50M (10× creates)
Requests/sec (avg)    = 50M / 86400 ≈ 580/sec
Peak requests/sec     ≈ 3,000/sec

Storage:
  Tasks               = 100M × 2KB = 200GB
  Comments            = 500M × 500B = 250GB
  Activity log        = 1B events × 200B = 200GB
  Total               ≈ 650GB
```

## 3. API Design

### Key Endpoints

#### Create Task
```
POST /api/v1/projects/{projectId}/tasks
Authorization: Bearer <token>

Request:
{
  "title": "Implement login page",
  "description": "Build the OAuth2 login flow",
  "assignee_id": "user_456",
  "parent_task_id": null,
  "priority": "HIGH",
  "due_date": "2025-07-15",
  "labels": ["frontend", "auth"]
}

Response 201:
{
  "id": "task_abc",
  "title": "Implement login page",
  "status": "TODO",
  "priority": "HIGH",
  "assignee": { "id": "user_456", "name": "Alice" },
  "parent_task_id": null,
  "subtask_count": 0,
  "version": 1,
  "created_at": "2025-06-01T10:00:00Z"
}
```

#### Update Task (with optimistic locking)
```
PATCH /api/v1/tasks/{taskId}
Authorization: Bearer <token>

Request:
{
  "title": "Implement OAuth2 login page",
  "status": "IN_PROGRESS",
  "version": 3          // must match current version in DB
}

Response 200:
{
  "id": "task_abc",
  "title": "Implement OAuth2 login page",
  "status": "IN_PROGRESS",
  "version": 4,
  "updated_at": "2025-06-01T11:00:00Z"
}

Errors:
- 409: Version conflict — task was modified by another user (include current version)
```

#### Get Task with Subtasks
```
GET /api/v1/tasks/{taskId}?include_subtasks=true

Response 200:
{
  "id": "task_abc",
  "title": "Implement login page",
  "status": "IN_PROGRESS",
  "version": 4,
  "subtasks": [
    { "id": "task_def", "title": "Design mockup", "status": "DONE" },
    { "id": "task_ghi", "title": "Write unit tests", "status": "TODO" }
  ]
}
```

#### List Project Tasks
```
GET /api/v1/projects/{projectId}/tasks?status=IN_PROGRESS&assignee=user_456&page=1

Response 200:
{
  "tasks": [...],
  "total": 87,
  "page": 1,
  "page_size": 20
}
```

#### Add Comment
```
POST /api/v1/tasks/{taskId}/comments
Authorization: Bearer <token>

Request: { "content": "Blocked on design review" }
Response 201: { "id": "cmt_xyz", "content": "...", "author": {...}, "created_at": "..." }
```

### Endpoint Summary

| Method | Path | Description |
|--------|------|-------------|
| POST | /api/v1/projects | Create project |
| GET | /api/v1/projects/{projectId}/tasks | List tasks in project |
| POST | /api/v1/projects/{projectId}/tasks | Create task |
| GET | /api/v1/tasks/{taskId} | Get task details |
| PATCH | /api/v1/tasks/{taskId} | Update task (optimistic lock) |
| DELETE | /api/v1/tasks/{taskId} | Delete task |
| POST | /api/v1/tasks/{taskId}/comments | Add comment |
| GET | /api/v1/tasks/{taskId}/comments | List comments |
| GET | /api/v1/tasks/search?q=... | Search tasks |

## 4. Database Schema

```sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    full_name   VARCHAR(255) NOT NULL,
    avatar_url  VARCHAR(500),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE projects (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    description TEXT,
    owner_id    BIGINT NOT NULL REFERENCES users(id),
    status      VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',  -- ACTIVE, ARCHIVED
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE project_members (
    project_id  BIGINT NOT NULL REFERENCES projects(id),
    user_id     BIGINT NOT NULL REFERENCES users(id),
    role        VARCHAR(30) NOT NULL DEFAULT 'MEMBER',  -- OWNER, ADMIN, MEMBER, VIEWER
    joined_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (project_id, user_id)
);

-- Self-referential hierarchy for subtasks
CREATE TABLE tasks (
    id              BIGSERIAL PRIMARY KEY,
    project_id      BIGINT NOT NULL REFERENCES projects(id),
    parent_task_id  BIGINT REFERENCES tasks(id),        -- NULL = top-level task
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'TODO',
                    -- TODO, IN_PROGRESS, IN_REVIEW, DONE, CANCELLED
    priority        VARCHAR(20) NOT NULL DEFAULT 'MEDIUM',
                    -- LOW, MEDIUM, HIGH, URGENT
    assignee_id     BIGINT REFERENCES users(id),
    created_by      BIGINT NOT NULL REFERENCES users(id),
    due_date        DATE,
    completed_at    TIMESTAMPTZ,
    version         INT NOT NULL DEFAULT 1,             -- optimistic locking
    position        FLOAT NOT NULL DEFAULT 0,           -- ordering within status column
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tasks_project_id ON tasks (project_id);
CREATE INDEX idx_tasks_parent_task_id ON tasks (parent_task_id);
CREATE INDEX idx_tasks_assignee_id ON tasks (assignee_id);
CREATE INDEX idx_tasks_status ON tasks (project_id, status);
CREATE INDEX idx_tasks_due_date ON tasks (due_date) WHERE due_date IS NOT NULL;
-- Full-text search
CREATE INDEX idx_tasks_title_fts ON tasks USING GIN (to_tsvector('english', title));

CREATE TABLE labels (
    id          BIGSERIAL PRIMARY KEY,
    project_id  BIGINT NOT NULL REFERENCES projects(id),
    name        VARCHAR(100) NOT NULL,
    color       CHAR(7) NOT NULL DEFAULT '#6366f1',
    UNIQUE (project_id, name)
);

CREATE TABLE task_labels (
    task_id     BIGINT NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    label_id    BIGINT NOT NULL REFERENCES labels(id) ON DELETE CASCADE,
    PRIMARY KEY (task_id, label_id)
);

CREATE TABLE comments (
    id          BIGSERIAL PRIMARY KEY,
    task_id     BIGINT NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    author_id   BIGINT NOT NULL REFERENCES users(id),
    content     TEXT NOT NULL,
    edited_at   TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_comments_task_id ON comments (task_id, created_at);

-- Audit trail for all task changes
CREATE TABLE task_activity (
    id          BIGSERIAL PRIMARY KEY,
    task_id     BIGINT NOT NULL REFERENCES tasks(id),
    actor_id    BIGINT NOT NULL REFERENCES users(id),
    action      VARCHAR(50) NOT NULL,   -- CREATED, STATUS_CHANGED, ASSIGNED, COMMENTED
    old_value   JSONB,
    new_value   JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_task_activity_task_id ON task_activity (task_id, created_at DESC);
```

### Schema Summary

| Table | Purpose |
|-------|---------|
| projects | Project containers |
| project_members | Project membership and roles |
| tasks | Tasks with self-referential hierarchy |
| labels | Project-scoped labels/tags |
| task_labels | Many-to-many task-label association |
| comments | Task comments |
| task_activity | Immutable audit log of all changes |

## 5. Key Design Decisions

### 5.1 Optimistic Locking for Concurrent Edits

When two users edit the same task simultaneously, last-write-wins would silently discard one user's changes. Optimistic locking detects the conflict and lets the client resolve it.

```sql
UPDATE tasks
SET title = $1, status = $2, version = version + 1, updated_at = NOW()
WHERE id = $3 AND version = $4;
-- If 0 rows affected → version mismatch → return 409 with current state
```

The client sends the `version` it last read. If another user updated the task in the meantime, the version won't match and the update is rejected with a 409. The client fetches the latest state and can merge or retry.

### 5.2 Hierarchical Tasks with Self-Reference

`parent_task_id` creates a tree structure. Depth is intentionally limited to 3 levels (task → subtask → sub-subtask) to keep queries simple. For arbitrary-depth trees, a closure table or `ltree` extension would be needed.

Fetching a task with all subtasks:
```sql
SELECT * FROM tasks WHERE parent_task_id = $1 ORDER BY position;
```

Deleting a parent cascades to subtasks via application logic (or `ON DELETE CASCADE` on the FK).

### 5.3 Real-Time Updates via WebSocket

When a task is updated, the server publishes an event to a Redis pub/sub channel keyed by `project:{projectId}`. All WebSocket connections subscribed to that project receive the update. This avoids polling and keeps all collaborators in sync.

```
PUBLISH project:proj_123 '{"type":"TASK_UPDATED","taskId":"task_abc","version":4}'
```

### 5.4 Task Ordering with Float Positions

Tasks within a status column (Kanban board) use a `position` float for ordering. Moving a task between positions sets its position to the midpoint of its neighbors. This avoids rewriting all positions on every reorder. When positions converge (gap < 0.001), a background job renormalizes the entire column.

### 5.5 Full-Text Search

PostgreSQL GIN index on `to_tsvector('english', title)` enables fast full-text search without Elasticsearch for moderate scale. For larger deployments, sync to Elasticsearch via CDC.

## 6. Failure Scenarios

### Concurrent Edit Conflict (Version Mismatch)
- **Impact**: User's update is rejected; they see a 409 with the current task state
- **Recovery**: Client shows a diff view; user can merge changes and resubmit with the new version
- **Prevention**: Optimistic locking is the prevention — it surfaces conflicts rather than silently losing data

### WebSocket Connection Drop
- **Impact**: User misses real-time updates while disconnected
- **Recovery**: On reconnect, client fetches the latest task state via REST; WebSocket resumes from current state
- **Prevention**: Client tracks last-seen `updated_at`; reconnect logic fetches delta since that timestamp

### Subtask Orphan on Parent Delete
- **Impact**: Subtasks lose their parent reference, becoming invisible in the UI
- **Recovery**: Application logic moves subtasks to top-level before deleting parent, or cascades deletion
- **Prevention**: Enforce deletion policy in application layer; never rely solely on DB cascade for business-critical trees

### Database Overload from Deep Subtask Queries
- **Impact**: Recursive CTE queries for deep hierarchies can be slow
- **Recovery**: Limit recursion depth; add query timeout
- **Prevention**: Enforce max depth of 3 in application layer; use `ltree` extension if deeper hierarchies are needed
