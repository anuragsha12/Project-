# Campus Lost & Found Portal

A full-stack lost and found portal for college campuses with **CLIP-based image recognition**. Uploaded photos are encoded into 512-dimensional vectors and matched against existing reports using cosine similarity via pgvector.

![React](https://img.shields.io/badge/React-18-61DAFB?logo=react)
![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688?logo=fastapi)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169E1?logo=postgresql)
![CLIP](https://img.shields.io/badge/CLIP-ViT--B%2F32-orange)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker)

## System Architecture

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  React SPA  │────▶│   FastAPI    │────▶│  PostgreSQL +   │
│  (Vite)     │     │   Backend    │     │  pgvector       │
└─────────────┘     └──────┬───────┘     └─────────────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         ┌─────────┐ ┌──────────┐ ┌──────────┐
         │ Cloudinary│ │ SendGrid │ │ CLIP ML  │
         │ (images) │ │ (email)  │ │ (vectors)│
         └─────────┘ └──────────┘ └──────────┘
```

**Flow:** User uploads item photo → image stored on Cloudinary → CLIP encodes to vector → stored in pgvector → matching engine finds similar opposite-type items → email alerts sent on match.

## Quick Start

### Prerequisites
- Docker & Docker Compose
- (Optional) Google OAuth credentials, Cloudinary account, SendGrid API key

### 1. Clone and configure

```bash
cd campus-lost-found
cp .env .env.local   # edit with your API keys
```

### 2. Start all services

```bash
docker-compose up --build
```

| Service  | URL                      |
|----------|--------------------------|
| Frontend | http://localhost:5173    |
| Backend  | http://localhost:8000    |
| API Docs | http://localhost:8000/docs |

### 3. Seed sample data

```bash
docker-compose exec backend python -m app.utils.seed_data
```

### Default Dev Accounts

| Email              | Role    | Password (reference) |
|--------------------|---------|----------------------|
| student@campus.edu | student | Test@1234            |
| staff@campus.edu   | staff   | Test@1234            |
| admin@campus.edu   | admin   | Test@1234            |

> Auth uses Google OAuth in production. For dev, sign in via Google with a matching `@campus.edu` email, or configure OAuth credentials in `.env`.

## Environment Variables

| Variable | Description |
|----------|-------------|
| `DATABASE_URL` | PostgreSQL connection string (asyncpg) |
| `GOOGLE_CLIENT_ID` | Google OAuth client ID |
| `GOOGLE_CLIENT_SECRET` | Google OAuth client secret |
| `GOOGLE_REDIRECT_URI` | OAuth callback URL |
| `SECRET_KEY` | JWT signing secret |
| `ALGORITHM` | JWT algorithm (HS256) |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | JWT expiry (default: 60) |
| `CLOUDINARY_CLOUD_NAME` | Cloudinary cloud name |
| `CLOUDINARY_API_KEY` | Cloudinary API key |
| `CLOUDINARY_API_SECRET` | Cloudinary API secret |
| `SENDGRID_API_KEY` | SendGrid API key for emails |
| `FROM_EMAIL` | Sender email address |
| `CLIP_MODEL` | HuggingFace CLIP model ID |
| `MATCH_THRESHOLD` | Minimum combined match score (0.65) |
| `ITEM_EXPIRY_DAYS` | Days before auto-archive (30) |
| `SKIP_CLIP` | Skip CLIP loading in dev (true) |
| `FRONTEND_URL` | Frontend URL for email links |

## API Endpoints

### Auth (`/auth`)
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/auth/google` | Redirect to Google OAuth |
| GET | `/auth/google/callback` | OAuth callback, returns JWT |
| POST | `/auth/guest-otp` | Send OTP to guest email |
| POST | `/auth/verify-otp` | Verify OTP, return guest token |
| GET | `/auth/me` | Current user from JWT |

### Items (`/items`)
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/items/` | Create item (multipart + image) |
| POST | `/items/auto-tags` | Get CLIP auto-tags from image |
| GET | `/items/` | Paginated list with filters |
| GET | `/items/my` | Current user's items |
| GET | `/items/{id}` | Item detail with matches |
| PATCH | `/items/{id}/status` | Update item status |
| DELETE | `/items/{id}` | Soft delete (archive) |

### Search (`/search`)
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/search/` | Text-to-image CLIP search |
| POST | `/search/by-image` | Image similarity search |

### Matches (`/matches`)
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/matches/{item_id}` | List matches for item |
| PATCH | `/matches/{match_id}` | Accept/reject match |

### Claims (`/claims`)
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/claims/` | Submit claim on found item |
| PATCH | `/claims/{id}` | Approve/reject (staff/admin) |
| POST | `/claims/{id}/handover` | Initiate OTP handover |
| POST | `/claims/{id}/verify-otp` | Verify handover OTP |

### Admin (`/admin`)
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/admin/dashboard` | Dashboard statistics |
| GET | `/admin/items` | All items with filters |
| PATCH | `/admin/items/{id}` | Force status change |
| GET | `/admin/claims` | Pending claims queue |
| POST | `/admin/expire` | Run expiry job |
| GET | `/admin/audit-logs` | Paginated audit log |

## Tech Stack

- **Frontend:** React 18, Vite, Tailwind CSS, React Router, Axios
- **Backend:** FastAPI, SQLAlchemy (async), Alembic, Pydantic v2
- **Database:** PostgreSQL 15 + pgvector (IVFFlat cosine index)
- **ML:** CLIP `openai/clip-vit-base-patch32` via HuggingFace Transformers
- **Storage:** Cloudinary (all image uploads)
- **Auth:** Google OAuth 2.0 + JWT with role-based access
- **Email:** SendGrid (match alerts, OTP, expiry warnings)
- **Infra:** Docker Compose with hot reload

## Project Structure

```
campus-lost-found/
├── backend/          # FastAPI + CLIP + pgvector
├── frontend/         # React SPA
├── docker-compose.yml
├── .env
└── README.md
```

## Development Notes

- CLIP model loads once at startup via `@app.on_event("startup")` and is stored in `app.state.clip`
- Set `SKIP_CLIP=true` in Docker dev to avoid downloading the ~600MB model
- Seed data uses random 512-dim unit vectors instead of real CLIP encoding
- Match alerts run as FastAPI `BackgroundTasks` — never blocking item submission
- pgvector IVFFlat index is created on startup (requires data for optimal performance)

## License

MIT
