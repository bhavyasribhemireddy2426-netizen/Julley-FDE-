# Real-Time Collaborative Kanban Board

## 1. Project Overview

A production-quality, multi-tenant, real-time collaborative Kanban task management application built for the **JulleyOnline Forward Deployed Engineer Intern Assignment**.

**Live Features:**
- 🏢 Multi-organization & multi-tenant workspace isolation
- ⚡ Real-time collaboration via Socket.IO (live task updates, presence indicators)
- 🔀 Drag-and-drop task management with conflict resolution
- 🔒 JWT authentication + 3-tier RBAC (Org Admin, Tenant Admin, Member)
- 📋 Full task lifecycle: create, assign, prioritize, move, comment, delete
- 📊 Activity history and audit log
- 🛡️ Strict tenant-level data isolation (cross-tenant access blocked at API and WebSocket level)
- 🔄 Optimistic concurrency control (version-based conflict detection)

---

## 2. Business Problem

ABC Technologies has 3 branches (Hyderabad, Bangalore, Chennai). Each branch has its own employees and projects. The application allows employees to collaborate on Kanban boards within their branch (tenant), while strictly preventing cross-tenant data access.

**Key challenge:** When one employee moves a task, others on the same board must see it **instantly** without refreshing — while employees in other branches must **never** receive those events.

---

## 3. Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| Frontend | React 18 + Vite | Kanban UI |
| Styling | Tailwind CSS | Dark-theme design system |
| Drag & Drop | dnd-kit | Task movement |
| Real-Time Client | Socket.IO Client | Live updates |
| HTTP Client | Axios | REST API calls |
| Backend | Node.js + Express.js | API server |
| Real-Time Server | Socket.IO | WebSocket events |
| ORM | Prisma | Database queries |
| Database | PostgreSQL | Persistent storage |
| Auth | JWT + bcrypt | Identity & password security |
| Validation | Zod | Runtime type safety |

---

## 4. System Architecture

```
React Frontend (Vite + Tailwind + dnd-kit)
        │
        ├── REST API (Axios) ──→ Express.js Backend ──→ Prisma ──→ PostgreSQL
        │
        └── WebSocket (Socket.IO) ──→ Socket.IO Server ──→ Tenant Rooms
                                              │
                                    Broadcasts to authorized
                                    users in same tenant room
```

---

## 5. Multi-Tenant Architecture

```
Organization (ABC Technologies)
├── Tenant: Hyderabad  → boards, tasks, members (isolated)
├── Tenant: Bangalore  → boards, tasks, members (isolated)
└── Tenant: Chennai    → boards, tasks, members (isolated)

Organization (XYZ Solutions)
├── Tenant: Mumbai     → boards, tasks, members (isolated)
└── Tenant: Pune       → boards, tasks, members (isolated)
```

**Isolation mechanism:**
1. Every record has `tenant_id`
2. Auth middleware validates JWT and attaches user + memberships
3. Every query filters by verified `tenant_id` (never from request body)
4. Socket.IO rooms named `tenant:{id}` — server verifies membership before join
5. 403 returned for any cross-tenant attempt

---

## 6. Role-Based Access Control

| Role | Permissions |
|---|---|
| `ORG_ADMIN` | Manage organization, all tenants, memberships |
| `TENANT_ADMIN` | Manage boards, columns, tenant members |
| `MEMBER` | View boards, create/edit tasks |

---

## 7. Database Design

8 tables: `organizations`, `tenants`, `users`, `memberships`, `boards`, `columns`, `tasks`, `activity_logs`

Key design decisions:
- UUID primary keys everywhere
- `version` field on tasks for optimistic locking
- `position` float field on tasks for ordering (large gaps = fewer updates)
- `tenant_id` on all business records
- Composite unique constraint on memberships

---

## 8. Concurrency Handling

**Optimistic Locking:**
1. Task carries a `version` integer (starts at 1)
2. Client sends current `version` with every update/move request
3. Backend: `UPDATE ... WHERE id = ? AND version = ?`
4. If version mismatch → `HTTP 409 Conflict`
5. Frontend: rolls back optimistic UI, shows conflict warning, prompts refresh

---

## 9. Installation & Setup

### Prerequisites
- Node.js 18+
- PostgreSQL 14+
- Git

### Step 1: Clone & install

```bash
git clone <repo-url>
cd collaborative-kanban
```

### Step 2: Database setup

Create PostgreSQL database:
```sql
CREATE DATABASE collaborative_kanban;
```

### Step 3: Backend setup

```bash
cd backend
cp .env.example .env
# Edit .env — set DATABASE_URL with your PostgreSQL credentials
npm install
npx prisma db push          # Create all tables
node database/seed.js       # Load sample data
npm run dev                 # Start backend on :5000
```

### Step 4: Frontend setup

```bash
cd frontend
npm install
npm run dev                 # Start frontend on :5173
```

---

## 10. Environment Variables

### backend/.env

```
DATABASE_URL="postgresql://USER:PASSWORD@localhost:5432/collaborative_kanban"
JWT_SECRET=your_secret_key_min_32_chars
JWT_EXPIRES_IN=7d
PORT=5000
FRONTEND_URL=http://localhost:5173
```

### frontend/.env

```
VITE_API_URL=http://localhost:5000/api
VITE_SOCKET_URL=http://localhost:5000
```

---

## 11. Test Accounts (after seeding)

All passwords: `password123`

| Email | Role | Tenant |
|---|---|---|
| rahul@abc.com | Org Admin | ABC Technologies |
| priya@abc.com | Tenant Admin | Hyderabad |
| arjun@abc.com | Member | Hyderabad |
| sneha@abc.com | Tenant Admin | Bangalore |
| kiran@xyz.com | Org Admin | XYZ Solutions |
| meera@xyz.com | Tenant Admin | Mumbai |

---

## 12. API Documentation

### Authentication
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | /api/auth/register | No | Register user |
| POST | /api/auth/login | No | Login |
| GET | /api/auth/me | JWT | Get current user |

### Organizations
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | /api/organizations | JWT | List user's orgs |
| POST | /api/organizations | JWT | Create org |
| GET | /api/organizations/:id | JWT | Get org |
| GET | /api/organizations/:id/members | JWT | List members |
| POST | /api/organizations/:id/invite | JWT+Admin | Invite member |

### Tenants
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | /api/tenants/org/:orgId | JWT | List tenants |
| POST | /api/tenants | JWT+OrgAdmin | Create tenant |
| GET | /api/tenants/:id | JWT | Get tenant |
| GET | /api/tenants/:id/members | JWT | List members |

### Boards
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | /api/boards?tenantId= | JWT | List boards |
| POST | /api/boards | JWT+Admin | Create board |
| GET | /api/boards/:id | JWT | Get board with columns+tasks |
| PUT | /api/boards/:id | JWT+Admin | Update board |
| DELETE | /api/boards/:id | JWT+Admin | Delete board |

### Tasks
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | /api/tasks?tenantId= | JWT | List tasks |
| POST | /api/tasks | JWT | Create task |
| GET | /api/tasks/:id | JWT | Get task + activity |
| PUT | /api/tasks/:id | JWT | Update task (version required) |
| PATCH | /api/tasks/:id/move | JWT | Move task (version required) |
| DELETE | /api/tasks/:id | JWT | Delete task |

---

## 13. WebSocket Events

| Event | Direction | Description |
|---|---|---|
| `tenant:join` | Client→Server | Join tenant room (verified server-side) |
| `task:created` | Server→Client | New task broadcast |
| `task:updated` | Server→Client | Task field update |
| `task:moved` | Server→Client | Task column/position change |
| `task:deleted` | Server→Client | Task deletion |
| `presence:update` | Server→Client | Online users list for tenant |

---

## 14. Known Limitations

- No pagination on task lists (suitable for demo scale)
- No email-based invite flow (users must register first)
- AI assistant not implemented (optional per assignment)
- No file attachment support
- Single PostgreSQL instance (would need read replicas at scale)

---

## 15. Future Enhancements

- Python FastAPI AI assistant for natural language board queries
- Real-time notifications (email/push)
- Task comments and @mentions
- Board templates
- Deployment: Vercel (frontend) + Railway (backend) + Supabase (DB)
