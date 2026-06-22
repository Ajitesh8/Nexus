# Nexus — Role-Based Event Operations Platform

> A full-stack web platform for managing physical events end-to-end — from team registration and payment verification to secure QR-based check-in and real-time attendance tracking. Built for any event that involves people showing up at a place.

---

## Table of Contents

- [What is Nexus?](#what-is-nexus)
- [Who is it for?](#who-is-it-for)
- [Use Cases](#use-cases)
- [How it Works — User Perspective](#how-it-works--user-perspective)
- [System Architecture](#system-architecture)
- [Tech Stack](#tech-stack)
- [Database Design](#database-design)
- [The QR Token System](#the-qr-token-system)
- [Role-Based Access Control](#role-based-access-control)
- [Real-Time Volunteer Dashboard](#real-time-volunteer-dashboard)
- [API Endpoints](#api-endpoints)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Security Considerations](#security-considerations)
- [Future Roadmap](#future-roadmap)

---

## What is Nexus?

Nexus is a web application that replaces the spreadsheets, paper lists, and WhatsApp chaos that typically runs event operations. It gives organizers a single platform to handle the entire lifecycle of a physical event:

- Collect registrations with multi-step forms
- Verify payment proof via file upload
- Issue unique, time-limited QR codes to approved participants
- Let volunteers scan those QR codes for instant check-in
- Track meal and resource distribution with the same QR infrastructure
- Watch a live dashboard update in real time as people arrive

Everything is role-aware. Organizers see everything. Candidates see only their own registration and QR code. Volunteers see only the scanning interface and live stats. Nobody can access what they shouldn't.

---

## Who is it for?

Any team that runs a physical event and needs to manage attendance, access control, or resource distribution — without the overhead of enterprise event software.

---

## Use Cases

Nexus is domain-agnostic. The same QR check-in and role-based dashboard infrastructure works across completely different contexts:

### Hackathons and Coding Competitions
The original use case. Participants register in teams, select tracks (e.g. Web Dev, AI/ML, Hardware), upload payment proof, and receive QR codes. Volunteers scan them at the venue entrance and at meal counters. The system prevents duplicate entries and double meal claims.

### College Fests and Cultural Events
Multi-day events with hundreds of participants across different sub-events. Each sub-event can track its own attendance. Stall coordinators use the volunteer dashboard to manage crowd flow. Organizers see a consolidated view.

### Technical Conferences and Workshops
Attendees register for sessions, receive QR codes, and check in at individual session entrances. This gives organizers precise headcount per session, not just overall attendance. Useful for capacity-limited rooms.

### Marathons and Sports Events
Participants register with their bib number, receive a QR code. Checkpoint volunteers scan codes as runners pass through. The system logs timestamps at each checkpoint, creating a partial timing record.

### Corporate Offsites and Town Halls
Employees register, check in at the venue, and collect their kit or meal. Ideal when HR wants a digital attendance record without managing a dedicated enterprise system.

### NGO Field Operations and Distribution Drives
Beneficiaries register in advance. Field volunteers scan QR codes to log attendance and mark when someone has received a food packet, medicine kit, or voucher. Prevents double-collection in high-volume distribution scenarios.

### University Examinations
Students register for an exam slot. Invigilators scan QR codes at entry. The system logs exact entry times and prevents someone from entering a room they are not registered for.

---

## How it Works — User Perspective

There are three roles in Nexus. Each role sees a completely different interface.

### The Organizer

The organizer sets up the event — defines the registration form fields, tracks, team size limits, and payment details. They review submitted registrations and approve or reject them based on payment proof. Once approved, the system automatically enables the candidate's QR code. The organizer also manages the volunteer roster, assigning users the volunteer role so they can access scanning dashboards.

### The Candidate (Participant)

A candidate visits the registration page and completes a multi-step form:

1. Team details — name, track selection, team size
2. Member details — name, email, and role for each member
3. Payment — uploads a screenshot or PDF of payment confirmation

After submission, the registration sits in `pending` state while the organizer reviews it. Once approved, the candidate's dashboard shows a button to reveal their QR code. The QR code is not static — it is generated fresh on demand and expires in 30 seconds. This prevents sharing, screenshotting, and reuse.

### The Volunteer

A volunteer opens the scanning dashboard on any phone or tablet — no app installation required, it runs entirely in the browser. They point the camera at a participant's QR code. Within a second, the screen flashes:

- **Green** — valid check-in, participant's name and team displayed
- **Yellow** — already checked in, shows the timestamp of the first scan
- **Red** — QR expired, invalid token, or registration not approved

The same flow applies at meal distribution counters. Each resource (lunch, snacks, kit bag) has its own scanning station. The system checks whether that participant has already collected that specific resource and blocks duplicates.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Browser Client                       │
│   Candidate Dashboard  │  Volunteer Scanner  │  Organizer   │
└──────────────┬──────────────────┬────────────────┬──────────┘
               │                  │                │
               ▼                  ▼                ▼
┌─────────────────────────────────────────────────────────────┐
│                  Next.js Application Layer                  │
│                                                             │
│   App Router (file-based routing per role)                  │
│   Middleware (JWT validation + role-based redirect)         │
│   API Routes (/api/register, /api/generate-qr, /api/scan)  │
└──────────────────────────────┬──────────────────────────────┘
                               │
               ┌───────────────┼───────────────┐
               ▼               ▼               ▼
        ┌────────────┐  ┌───────────┐  ┌──────────────┐
        │ Supabase   │  │ Supabase  │  │  Supabase    │
        │ PostgreSQL │  │   Auth    │  │   Storage    │
        │ (data)     │  │  (JWT)    │  │  (uploads)   │
        └────────────┘  └───────────┘  └──────────────┘
               │
               ▼
        ┌────────────┐
        │  Supabase  │
        │  Realtime  │
        │ (WebSocket │
        │  push)     │
        └────────────┘
```

The architecture is deliberately simple. Next.js handles both the frontend and the API layer in one deployment. Supabase provides the entire backend infrastructure — authentication, database, file storage, and real-time event streaming — so there is no separate backend server to maintain.

---

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Framework | Next.js 15 (App Router) | Unified frontend + API in one project, file-based routing, server components |
| UI Library | React 19 | Component model, hooks for state management |
| Language | TypeScript | Type safety across data models — critical when dealing with registrations, tokens, and roles |
| Styling | Tailwind CSS 4 | Utility-first, consistent design without context switching to CSS files |
| Backend & Auth | Supabase | Managed PostgreSQL + Auth + Storage + Realtime in one service |
| Animations | Framer Motion | Scan result transitions (green/red flash) — improves volunteer UX under pressure |
| QR Generation | react-qr-code | Client-side QR image rendering from a token string |
| QR Scanning | html5-qrcode | Browser-based camera access and QR decoding — no app install required |

---

## Database Design

The database is a PostgreSQL instance managed by Supabase. Here is the schema and the reasoning behind each table.

### `profiles`
Extends Supabase's built-in auth users table. Stores the role assigned to each user.

```sql
id          uuid  (references auth.users)
role        text  -- 'organizer', 'candidate', or 'volunteer'
created_at  timestamp
```

Every user who signs up gets a profile row. The role field is set by the organizer and read by middleware on every page load.

### `registrations`
One row per team registration.

```sql
id              uuid
submitted_by    uuid  (references profiles)
team_name       text
track           text
payment_proof   text  (Supabase Storage path)
status          text  -- 'pending', 'approved', 'rejected'
created_at      timestamp
```

The `status` field drives what the candidate sees on their dashboard. Only `approved` registrations can generate QR tokens.

### `team_members`
One row per person in a team. A registration has multiple members.

```sql
id               uuid
registration_id  uuid  (references registrations)
name             text
email            text
role             text  -- 'leader' or 'member'
```

### `qr_tokens`
The heart of the check-in system. One row per QR code generation request.

```sql
id               uuid
registration_id  uuid  (references registrations)
token            text  (unique, random string)
expires_at       timestamp  -- 30 seconds from creation
used             boolean
created_at       timestamp
```

Tokens are single-use and time-limited. The `used` flag is set atomically when a volunteer scans — preventing race conditions where two volunteers scan the same code at the same moment.

### `check_ins`
Audit log of every successful venue entry.

```sql
id               uuid
registration_id  uuid
scanned_by       uuid  (volunteer's profile id)
scanned_at       timestamp
```

Querying `WHERE registration_id = X` before allowing entry tells us immediately if someone has already checked in.

### `resource_claims`
Tracks distribution of meals, kits, or any other physical resource.

```sql
id               uuid
registration_id  uuid
resource_type    text  -- 'lunch', 'snacks', 'kit', etc.
claimed_at       timestamp
claimed_by       uuid  (volunteer)
```

The combination of `registration_id + resource_type` is unique, enforced at the database level. Even if the application layer has a bug, the database will reject a duplicate claim.

---

## The QR Token System

This is the most technically interesting part of Nexus, and the question most interviewers will focus on.

### Why not just use the registration ID?

The simplest approach would be to encode the registration ID directly into a QR code and show it permanently on the candidate's dashboard. This is insecure for three reasons:

1. A candidate could screenshot the QR and share it with someone who didn't register
2. A bad actor could guess sequential registration IDs and generate fake QR codes
3. Once someone checks in, their static QR is still valid and could be used again

### The time-limited token approach

When a candidate clicks "Show my QR code", the following happens:

**Step 1 — Token generation (`POST /api/generate-qr`)**
```
1. Verify the candidate's JWT and confirm their registration is approved
2. Generate a cryptographically random token string (UUID v4)
3. Insert a row into qr_tokens with:
   - The token string
   - The registration ID
   - expires_at = NOW() + 30 seconds
   - used = false
4. Encode the token into a QR image and return it to the browser
```

**Step 2 — QR display**

The candidate's browser renders the QR image. A countdown timer shows how many seconds remain. When it hits zero, the QR disappears and the candidate must click again to generate a fresh one.

**Step 3 — Scanning (`POST /api/scan`)**
```
1. Volunteer's camera reads the token string from the QR
2. API looks up the token in qr_tokens
3. Checks:
   - Does the token exist? → if not, invalid
   - Is expires_at in the past? → if yes, expired
   - Is used = true? → if yes, already scanned
4. If all checks pass:
   - Set used = true on the token (atomic update)
   - Insert a row into check_ins
   - Return success with candidate's name and team
5. Return the appropriate response (success / expired / already used / invalid)
```

### Handling the race condition

What if two volunteers scan the same QR code at the exact same millisecond? Without protection, both scans could read `used = false` simultaneously and both succeed, creating two check-in records.

The fix is a database-level atomic update:

```sql
UPDATE qr_tokens
SET used = true
WHERE token = $1
  AND used = false
  AND expires_at > NOW()
RETURNING id, registration_id;
```

If this query returns a row, the scan succeeded and this volunteer "won." If it returns nothing, the token was already used or expired. Only one concurrent transaction can win this race — the database handles the locking automatically.

---

## Role-Based Access Control

Next.js middleware runs before every page request. It sits in `middleware.ts` at the project root.

```
Request arrives for /volunteer/dashboard
        ↓
Middleware reads the JWT from the cookie
        ↓
Decodes the JWT to get user_id
        ↓
Queries profiles table for user's role
        ↓
Role is 'candidate'
        ↓
Redirect to /candidate/dashboard
```

The middleware enforces this on every request, server-side. Even if someone manually types `/organizer/registrations` in the URL bar, the middleware intercepts the request before the page renders and redirects them away. There is no client-side-only guard that can be bypassed by disabling JavaScript.

The route protection matrix:

| Route | Organizer | Volunteer | Candidate |
|---|---|---|---|
| `/organizer/*` | ✓ | ✗ | ✗ |
| `/volunteer/*` | ✓ | ✓ | ✗ |
| `/candidate/*` | ✗ | ✗ | ✓ |
| `/registration` | ✓ | ✓ | ✓ |

---

## Real-Time Volunteer Dashboard

The volunteer dashboard shows a live counter of check-ins and a feed of recent scans. This updates across all volunteer devices simultaneously without anyone refreshing the page.

This uses **Supabase Realtime**, which is built on PostgreSQL's logical replication (LISTEN/NOTIFY). Here is how it works:

1. When the volunteer dashboard loads, the browser opens a WebSocket connection to Supabase
2. It subscribes to INSERT events on the `check_ins` table
3. When any volunteer anywhere scans a QR and a new row is inserted, PostgreSQL fires a change notification
4. Supabase's Realtime server picks up the notification and pushes it over WebSocket to all subscribed clients
5. Each client's JavaScript receives the event and updates the counter and feed without a page reload

From the frontend code perspective:

```typescript
const channel = supabase
  .channel('check-ins-feed')
  .on(
    'postgres_changes',
    { event: 'INSERT', schema: 'public', table: 'check_ins' },
    (payload) => {
      setCheckinCount(prev => prev + 1);
      setRecentScans(prev => [payload.new, ...prev].slice(0, 10));
    }
  )
  .subscribe();
```

This pattern means one organizer on a laptop and twenty volunteers on phones are all looking at the same live number, with no polling, no manual refresh, and no delay.

---

## API Endpoints

All endpoints are Next.js Route Handlers located in `app/api/`.

| Method | Path | Auth Required | Description |
|---|---|---|---|
| `POST` | `/api/register` | Candidate JWT | Submit a new team registration with member details |
| `GET` | `/api/registration/status` | Candidate JWT | Get the current status of the caller's registration |
| `POST` | `/api/generate-qr` | Candidate JWT | Generate a 30-second QR token for an approved registration |
| `POST` | `/api/scan` | Volunteer JWT | Validate a scanned token and record check-in |
| `POST` | `/api/claim-resource` | Volunteer JWT | Mark a resource (lunch, snacks) as claimed for a registration |
| `GET` | `/api/registrations` | Organizer JWT | List all registrations with filters for status and track |
| `PATCH` | `/api/registrations/:id` | Organizer JWT | Approve or reject a registration |
| `GET` | `/api/stats` | Organizer JWT | Aggregate stats — total registered, checked in, meals claimed |

---

## Project Structure

```
nexus/
├── app/
│   ├── api/
│   │   ├── generate-qr/        # QR token generation endpoint
│   │   ├── register/           # Team registration endpoint
│   │   ├── scan/               # QR scan validation endpoint
│   │   ├── claim-resource/     # Meal/resource claim endpoint
│   │   └── registrations/      # Organizer registration management
│   ├── candidate/
│   │   └── dashboard/          # Candidate's QR code and status view
│   ├── volunteer/
│   │   └── dashboard/          # Live scanning interface + stats
│   ├── organizer/
│   │   └── registrations/      # Approval queue and analytics
│   ├── registration/           # Public multi-step registration form
│   └── page.tsx                # Landing page
├── components/
│   ├── auth/                   # Login, signup, session management
│   ├── registration/           # Multi-step form components
│   ├── volunteer/              # QR scanner + result display
│   └── ui/                     # Shared primitives (buttons, cards, badges)
├── lib/
│   ├── supabase/               # Supabase client initialisation (server + browser)
│   ├── auth.ts                 # JWT helpers and role extraction
│   └── tokens.ts               # QR token generation and validation logic
├── middleware.ts                # Route-level role enforcement
└── types/
    └── index.ts                # Shared TypeScript types for all data models
```

---

## Getting Started

### Prerequisites

- Node.js 18 or later
- A Supabase project (free tier works)

### Installation

```bash
git clone https://github.com/yourusername/nexus.git
cd nexus
npm install
```

### Database Setup

Run the following SQL in your Supabase SQL editor to create the schema:

```sql
-- Profiles (extends auth.users)
create table profiles (
  id uuid references auth.users on delete cascade primary key,
  role text not null default 'candidate',
  created_at timestamp default now()
);

-- Registrations
create table registrations (
  id uuid primary key default gen_random_uuid(),
  submitted_by uuid references profiles,
  team_name text not null,
  track text not null,
  payment_proof text,
  status text default 'pending',
  created_at timestamp default now()
);

-- Team members
create table team_members (
  id uuid primary key default gen_random_uuid(),
  registration_id uuid references registrations on delete cascade,
  name text not null,
  email text not null,
  role text default 'member'
);

-- QR tokens
create table qr_tokens (
  id uuid primary key default gen_random_uuid(),
  registration_id uuid references registrations,
  token text unique not null,
  expires_at timestamp not null,
  used boolean default false,
  created_at timestamp default now()
);

-- Check-ins
create table check_ins (
  id uuid primary key default gen_random_uuid(),
  registration_id uuid references registrations,
  scanned_by uuid references profiles,
  scanned_at timestamp default now()
);

-- Resource claims
create table resource_claims (
  id uuid primary key default gen_random_uuid(),
  registration_id uuid references registrations,
  resource_type text not null,
  claimed_at timestamp default now(),
  claimed_by uuid references profiles,
  unique(registration_id, resource_type)
);
```

Enable Realtime for the `check_ins` table in your Supabase dashboard under Database → Replication.

### Run

```bash
cp .env.local.example .env.local
# Fill in your Supabase credentials (see Environment Variables below)
npm run dev
```

Open `http://localhost:3000`.

---

## Environment Variables

```env
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
```

`NEXT_PUBLIC_` variables are safe to expose to the browser — they are the public Supabase URL and anon key, which are designed to be public. Row-level security policies on the database enforce access control, not the key itself.

`SUPABASE_SERVICE_ROLE_KEY` is secret and used only in server-side API routes where admin-level database access is needed (e.g. approving registrations). It is never sent to the browser.

---

## Security Considerations

**Time-limited QR tokens** — tokens expire in 30 seconds and are single-use. A screenshot is worthless after 30 seconds. Sharing a code is impossible in practice.

**Atomic token consumption** — the database UPDATE that marks a token as used is atomic. Concurrent scans of the same token cannot both succeed.

**Server-side role enforcement** — role checks run in Next.js middleware on the server before any page content is sent. There is no client-side-only guard.

**Service role key isolation** — the Supabase service role key (which bypasses row-level security) is only used in server API routes. It is never exposed to the browser.

**Payment proof privacy** — uploaded payment screenshots are stored in a private Supabase Storage bucket. Only the organizer can access them via signed URLs generated server-side.


## License

MIT
