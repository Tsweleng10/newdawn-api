# 🌅 NewDawn API

> **REST API for the NewDawn township job opportunity platform.**

Node.js + Express.js REST API backed by PostgreSQL. Handles user authentication with bcrypt-encrypted passwords and JWT sessions, job listings, and offers between job posters and workers.

**Live API:** https://newdawn-api-production.up.railway.app/

---

## 📖 Table of Contents

- [Purpose](#-purpose)
- [Tech Stack](#-tech-stack)
- [Architecture](#-architecture)
- [Database Schema](#-database-schema)
- [Getting Started](#-getting-started)
- [Environment Variables](#-environment-variables)
- [API Reference](#-api-reference)
- [Security](#-security)
- [Deployment](#-deployment)
- [Testing with Postman](#-testing-with-postman)
- [Project Structure](#-project-structure)
- [Related Repository](#-related-repository)
- [Team](#-team)
- [References](#-references)

---

## 🎯 Purpose

This repository contains the backend REST API for the NewDawn Android application. It acts as the middle layer between the mobile app and the PostgreSQL database — the Android app never talks to the database directly.

**Responsibilities:**
- Validate and process all incoming requests from the Android client
- Hash and verify user passwords (bcrypt)
- Issue and verify JWT authentication tokens
- Enforce ownership rules (only job posters can view their own offers, only the poster can accept an offer, etc.)
- Perform all PostgreSQL reads and writes
- Return JSON responses with proper HTTP status codes

---

## 🛠 Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 20 |
| Framework | Express.js |
| Database | PostgreSQL (hosted on Neon) |
| Auth | JWT (jsonwebtoken) |
| Password Hashing | bcrypt (10 rounds) |
| Middleware | cors, express.json, custom auth middleware |
| Env Management | dotenv |
| Hosting | Railway |

---

## 🏗 Architecture

```text
┌───────────────────────────────────────────────────┐
│                  Android Client                   │
└──────────────────────┬────────────────────────────┘
                       │ HTTPS + JWT
                       ▼
┌───────────────────────────────────────────────────┐
│                Express.js Server                  │
│                                                   │
│  server.js                                        │
│   ├── cors()                                      │
│   ├── express.json()                              │
│   ├── request logger                              │
│   └── routes                                      │
│        ├── /api/auth   → routes/auth.js           │
│        ├── /api/jobs   → routes/jobs.js           │
│        └── /api/offers → routes/offers.js         │
│                                                   │
│  middleware/auth.js → verifies JWT for            │
│                       protected routes            │
└──────────────────────┬────────────────────────────┘
                       │ pg (connection pool)
                       ▼
┌───────────────────────────────────────────────────┐
│                PostgreSQL (Neon)                  │
│             users | jobs | offers                 │
└───────────────────────────────────────────────────┘
```

---

## 🗄 Database Schema

The schema is created automatically on server startup from `schema.sql`.

### `users`

| Column | Type | Notes |
|---|---|---|
| id | SERIAL PRIMARY KEY | Auto-increment |
| full_name | VARCHAR(100) | Required |
| email | VARCHAR(150) | Required, UNIQUE |
| password_hash | VARCHAR(255) | bcrypt hash (`$2b$10$...`) |
| location | VARCHAR(100) | Optional |
| language | VARCHAR(10) | Default `'en'` |
| notifications_enabled | BOOLEAN | Default `TRUE` |
| created_at | TIMESTAMP | Default `CURRENT_TIMESTAMP` |

### `jobs`

| Column | Type | Notes |
|---|---|---|
| id | SERIAL PRIMARY KEY | Auto-increment |
| poster_id | INTEGER | FK → users(id), ON DELETE CASCADE |
| title | VARCHAR(150) | Required |
| category | VARCHAR(80) | Required |
| description | TEXT | Required |
| location | VARCHAR(150) | Required |
| budget | NUMERIC(10,2) | Required |
| status | VARCHAR(20) | Default `'OPEN'` (OPEN/ACTIVE/COMPLETED) |
| created_at | TIMESTAMP | Default `CURRENT_TIMESTAMP` |

### `offers`

| Column | Type | Notes |
|---|---|---|
| id | SERIAL PRIMARY KEY | Auto-increment |
| job_id | INTEGER | FK → jobs(id), ON DELETE CASCADE |
| worker_id | INTEGER | FK → users(id), ON DELETE CASCADE |
| proposed_price | NUMERIC(10,2) | Required |
| message | TEXT | Optional |
| availability | VARCHAR(100) | Optional |
| status | VARCHAR(20) | Default `'PENDING'` |
| created_at | TIMESTAMP | Default `CURRENT_TIMESTAMP` |

**Constraint:** `UNIQUE (job_id, worker_id)` — prevents the same worker from submitting two offers on the same job.

---

## 🚀 Getting Started

### Prerequisites

- Node.js 20 or later
- A PostgreSQL database (Neon free tier works great)
- Git

### Installation

```bash
git clone https://github.com/Tsweleng10/newdawn-api.git
cd newdawn-api
npm install
```

### Configure Environment

Create a `.env` file at the project root:

```text
DATABASE_URL=postgresql://user:password@host:5432/database
JWT_SECRET=your_long_random_secret_here
PORT=3000
```

- `DATABASE_URL` — full PostgreSQL connection string
- `JWT_SECRET` — any long random string used to sign tokens
- `PORT` — local port (Railway overrides this in production)

### Run Locally

```bash
npm run dev
```

Expected output:

```text
✅ Database schema ready
🚀 Server on port 3000
```

Then open http://localhost:3000/ — you should see:

```json
{ "status": "NewDawn API running" }
```

### Run in Production

```bash
npm start
```

---

## 🔐 Environment Variables

| Name | Required | Description |
|---|---|---|
| `DATABASE_URL` | ✅ | PostgreSQL connection string |
| `JWT_SECRET` | ✅ | Secret used to sign JWT tokens |
| `PORT` | Optional | Port to listen on (Railway sets this automatically) |
| `NODE_VERSION` | Optional | Node version (Railway only — set to `20`) |

> ⚠️ **Never commit `.env` to Git.** It contains your database password. The `.gitignore` in this repo already excludes it.

---

## 📡 API Reference

**Base URL (production):** `https://newdawn-api-production.up.railway.app/`

All authenticated endpoints require this header:

```text
Authorization: Bearer <jwt_token>
```

### Health Check

| Method | Endpoint | Description |
|---|---|---|
| GET | `/` | Returns server status |

### Authentication

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/api/auth/register` | ❌ | Register a new user, returns user + JWT |
| POST | `/api/auth/login` | ❌ | Login with email/password, returns user + JWT |
| GET | `/api/auth/me` | ✅ | Get the currently authenticated user |
| PUT | `/api/auth/settings` | ✅ | Update language and notification preference |
| PUT | `/api/auth/change-password` | ✅ | Change password (requires old password) |

### Jobs

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/api/jobs` | ✅ | List all OPEN jobs. Supports `?search=`, `?category=`, `?location=` |
| GET | `/api/jobs/my` | ✅ | List jobs posted by the current user |
| GET | `/api/jobs/:id` | ✅ | Get details of a single job |
| POST | `/api/jobs` | ✅ | Create a new job listing |
| PUT | `/api/jobs/:id/status` | ✅ | Update job status (poster only) |

### Offers

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/api/offers/job/:jobId` | ✅ | Submit an offer on a job |
| GET | `/api/offers/job/:jobId` | ✅ | View all offers for a job (poster only) |
| GET | `/api/offers/my` | ✅ | View offers submitted by the current user |
| PUT | `/api/offers/:id/accept` | ✅ | Accept an offer (poster only) |

### Example: Register

**Request**

```http
POST /api/auth/register
Content-Type: application/json

{
  "full_name": "Sarah Lekoane",
  "email": "sarah@example.com",
  "password": "test1234",
  "location": "Soweto"
}
```

**Response `201 Created`**

```json
{
  "user": {
    "id": 1,
    "full_name": "Sarah Lekoane",
    "email": "sarah@example.com",
    "location": "Soweto"
  },
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

### Example: Login

**Request**

```http
POST /api/auth/login
Content-Type: application/json

{
  "email": "sarah@example.com",
  "password": "test1234"
}
```

**Response `200 OK`**

```json
{
  "user": {
    "id": 1,
    "full_name": "Sarah Lekoane",
    "email": "sarah@example.com",
    "location": "Soweto"
  },
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

### Example: Create a Job

**Request**

```http
POST /api/jobs
Authorization: Bearer <jwt_token>
Content-Type: application/json

{
  "title": "Gardener Needed",
  "category": "Gardening",
  "description": "I need someone to cut grass and clean my garden.",
  "location": "Soweto",
  "budget": 300
}
```

**Response `201 Created`**

```json
{
  "id": 1,
  "poster_id": 1,
  "title": "Gardener Needed",
  "category": "Gardening",
  "description": "I need someone to cut grass and clean my garden.",
  "location": "Soweto",
  "budget": "300.00",
  "status": "OPEN",
  "created_at": "2026-09-24T10:15:00.000Z"
}
```

### Example: Submit an Offer

**Request**

```http
POST /api/offers/job/1
Authorization: Bearer <jwt_token>
Content-Type: application/json

{
  "proposed_price": 280,
  "message": "I have gardening experience and am available Saturday.",
  "availability": "Saturday"
}
```

**Response `201 Created`**

```json
{
  "id": 1,
  "job_id": 1,
  "worker_id": 2,
  "proposed_price": "280.00",
  "message": "I have gardening experience and am available Saturday.",
  "availability": "Saturday",
  "status": "PENDING",
  "created_at": "2026-09-24T11:20:00.000Z"
}
```

### HTTP Status Codes Used

| Code | Meaning |
|---|---|
| 200 | Success |
| 201 | Resource created |
| 400 | Bad request (missing/invalid fields) |
| 401 | Unauthorized (missing or invalid token, wrong password) |
| 403 | Forbidden (e.g. accepting an offer that isn't yours) |
| 404 | Resource not found |
| 409 | Conflict (duplicate email, duplicate offer) |
| 500 | Server error |

---

## 🔒 Security

### Password Encryption

Passwords are **never stored in plain text**. When a user registers, the password is hashed with **bcrypt** using 10 salt rounds before being inserted into the database.

```javascript
const password_hash = await bcrypt.hash(password, 10);
```

The resulting hash looks like this:

```text
$2b$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
```

This means: even if the database is compromised, the original passwords cannot be recovered.

### JWT Authentication

On successful login or registration, the server issues a JWT signed with `JWT_SECRET`. The token payload contains `{ id, email }` and expires after **7 days**.

Every protected route runs through `middleware/auth.js`, which:

1. Reads the `Authorization` header
2. Extracts the token (`Bearer <token>`)
3. Verifies it using `JWT_SECRET`
4. Attaches `req.user = { id, email }` to the request
5. Rejects the request with `401` if the token is missing or invalid

### SQL Injection Protection

All database queries use **parameterized queries** (`$1`, `$2`, ...) — never string concatenation. This makes SQL injection impossible.

```javascript
// ✅ Safe
await pool.query('SELECT * FROM users WHERE email = $1', [email]);

// ❌ Never do this
await pool.query(`SELECT * FROM users WHERE email = '${email}'`);
```

### CORS

CORS is enabled globally so the Android app can communicate with the API from any origin.

---

## ☁️ Deployment

The API is deployed on **Railway** and connected to a **Neon** PostgreSQL database.

### Environment Variables on Railway

| Key | Value |
|---|---|
| `DATABASE_URL` | Neon connection string |
| `JWT_SECRET` | Long random string |
| `NODE_VERSION` | `20` |

### Start Command

```text
node server.js
```

### Build Command

```text
npm install
```

### Deployment Steps (for reference)

1. Push code to GitHub
2. In Railway, create a **New Project → Deploy from GitHub repo**
3. Select this repository
4. Add the environment variables above
5. Railway builds and deploys automatically on every push to `master`

The schema is applied automatically on first boot, so no manual database setup is needed.

---

## 🧪 Testing with Postman

The easiest way to test the API is with [Postman](https://www.postman.com/).

### 1. Register a user

- **Method:** `POST`
- **URL:** `https://newdawn-api-production.up.railway.app/api/auth/register`
- **Body:** raw JSON

```json
{
  "full_name": "Test User",
  "email": "test@newdawn.com",
  "password": "test1234",
  "location": "Soweto"
}
```

Send → save the `token` from the response.

### 2. Get your profile

- **Method:** `GET`
- **URL:** `https://newdawn-api-production.up.railway.app/api/auth/me`
- **Headers:** `Authorization: Bearer <paste token>`

### 3. Create a job

- **Method:** `POST`
- **URL:** `https://newdawn-api-production.up.railway.app/api/jobs`
- **Headers:** `Authorization: Bearer <token>`
- **Body:** raw JSON

```json
{
  "title": "Gardener Needed",
  "category": "Gardening",
  "description": "Cut grass and clean my garden.",
  "location": "Soweto",
  "budget": 300
}
```

### 4. List jobs

- **Method:** `GET`
- **URL:** `https://newdawn-api-production.up.railway.app/api/jobs`
- **Headers:** `Authorization: Bearer <token>`

### Verify password hashing in the database

Open the Neon SQL Editor and run:

```sql
SELECT id, full_name, email, password_hash FROM users ORDER BY id DESC LIMIT 5;
```

You should see `password_hash` values that start with `$2b$10$...` — proof that bcrypt is working.

---

## 📁 Project Structure

```text
newdawn-api/
├── server.js              # Entry point — sets up Express, middleware, and routes
├── db.js                  # PostgreSQL connection pool
├── schema.sql             # Database schema (auto-applied on startup)
├── package.json
├── .env                   # Local secrets (gitignored)
├── .gitignore
├── middleware/
│   └── auth.js            # JWT verification middleware
└── routes/
    ├── auth.js            # /api/auth endpoints
    ├── jobs.js            # /api/jobs endpoints
    └── offers.js          # /api/offers endpoints
```

---

## 🔗 Related Repository

**Android client:** [github.com/Tsweleng10/newdawn-android](https://github.com/Tsweleng10/newdawn-android)

The Android app uses Retrofit to call this API over HTTPS, stores the JWT in DataStore, and attaches the token to every authenticated request via the `Authorization` header.

---

## 👥 Team

| Name | Student Number | Role |
|---|---|---|
| Joshua Tsweleng | st10451745 | Lead — API Development, Integration |
| Fortune Lemekwana | st10450241 | Person 3 — Jobs & Home |
| Phuti Magwai | st10452585 | Person 2 — Authentication & Profile |

---

## 📚 References

- Node.js Foundation. (2024). *Node.js Documentation*. https://nodejs.org/docs
- OpenJS Foundation. (2024). *Express.js Documentation*. https://expressjs.com
- PostgreSQL Global Development Group. (2024). *PostgreSQL Documentation*. https://www.postgresql.org/docs
- Neon. (2024). *Serverless PostgreSQL*. https://neon.tech/docs
- Railway. (2024). *Deploy Node.js Apps*. https://docs.railway.app
- Auth0. (2024). *Introduction to JSON Web Tokens*. https://jwt.io/introduction
- Provos, N., & Mazières, D. (1999). *A Future-Adaptable Password Scheme*. USENIX.

---

## 📄 License

Created for academic purposes as part of the **OPSC6312** module. Not licensed for commercial use.

---

*Built with ❤️ in South Africa*
