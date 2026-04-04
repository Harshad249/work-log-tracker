# WorkLog Tracker — Delta Metrology
**Internal Employee Work Log & Billing Report System**

---

## What's Included

```
worklog-tracker/
├── index.html           ← Standalone frontend (works offline, no build step)
├── backend/
│   ├── main.py          ← FastAPI backend (production-ready)
│   └── requirements.txt ← Python dependencies
└── README.md
```

---

## Quick Start (Frontend Only — No Backend Needed)

The `index.html` file is a **fully self-contained app** using localStorage.
Just open it in any browser — no server, no install.

```bash
# Option 1: Double-click index.html in File Explorer
# Option 2: Serve with Python
python -m http.server 8080
# Open http://localhost:8080
```

### Demo Logins

| Role     | Email                          | Password  |
|----------|-------------------------------|-----------|
| Admin    | admin@deltametrology.in        | admin123  |
| Employee | rahul@deltametrology.in        | emp123    |
| Employee | priya@deltametrology.in        | emp123    |

---

## Production Backend Setup (FastAPI + PostgreSQL)

### 1. Prerequisites
- Python 3.11+
- PostgreSQL 15+ (or use SQLite for small teams)
- Node.js (optional, only if you migrate frontend to React build)

### 2. Install & Run Backend

```bash
cd backend

# Create virtual environment
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Set environment variables
export SECRET_KEY="your-super-secret-key-change-this"
export DATABASE_URL="postgresql://user:password@localhost:5432/worklog"
# For SQLite (small teams): export DATABASE_URL="sqlite:///./worklog.db"

# Run the API server
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

API will be live at: http://localhost:8000  
Swagger docs at:    http://localhost:8000/docs

### 3. Connect Frontend to Backend

In `index.html`, replace the `DB` object calls with fetch() calls to your API.
The API URL goes in a single constant at the top of the script:

```javascript
const API_BASE = "https://your-api-server.com";
```

---

## API Reference

All endpoints require `Authorization: Bearer <token>` except `/auth/*`.

### Authentication

#### POST /auth/register
Create a new account.
```json
{
  "name": "Rahul Deshmukh",
  "email": "rahul@deltametrology.in",
  "password": "secure123",
  "role": "employee"
}
```
Returns: `{ access_token, token_type, user }`

#### POST /auth/login
```
Content-Type: application/x-www-form-urlencoded
username=rahul@deltametrology.in&password=secure123
```
Returns: `{ access_token, token_type, user }`

#### GET /auth/me
Returns current user profile.

---

### Work Logs

#### POST /logs — Create a log entry
```json
{
  "date": "2026-04-04",
  "project": "CMM Inspection",
  "task": "Dimensional inspection of brake components for Tata Motors",
  "hours": 8.0,
  "remarks": "Rush job, completed on time"
}
```

#### GET /logs — List logs
Query params (admin only for filtering across users):
- `user_id=<uuid>`
- `project=<name>`
- `date_from=YYYY-MM-DD`
- `date_to=YYYY-MM-DD`

#### PUT /logs/{id} — Update a log
```json
{ "hours": 7.5, "remarks": "Revised after review" }
```
Employees: only same-day entries. Admins: any entry.

#### DELETE /logs/{id} — Delete a log

---

### Reports (Admin only)

#### GET /reports/monthly/xlsx?year=2026&month=4
Downloads Excel file: `WorkLog_2026_04.xlsx`

**Columns:**
- Employee Name | Date | Project | Task Description | Hours Worked | Remarks

#### GET /reports/monthly/csv?year=2026&month=4
Downloads CSV file.

#### GET /reports/summary?year=2026&month=4
Returns JSON summary:
```json
{
  "total_entries": 62,
  "total_hours": 496.5,
  "by_employee": [
    { "name": "Rahul Deshmukh", "hours": 168.0 }
  ],
  "by_project": [
    { "project": "CMM Inspection", "hours": 124.5 }
  ]
}
```

---

### User Management (Admin only)

#### GET /users — List all employees
#### POST /users — Add employee
#### DELETE /users/{id} — Remove employee

---

## Deployment Guide

### Option A: Render.com (Recommended, Free Tier Available)

1. Push code to GitHub
2. Create a new **Web Service** on Render
3. Set:
   - Build command: `pip install -r backend/requirements.txt`
   - Start command: `uvicorn backend.main:app --host 0.0.0.0 --port $PORT`
4. Add environment variables:
   - `SECRET_KEY` → generate with `python -c "import secrets; print(secrets.token_hex(32))"`
   - `DATABASE_URL` → Render provides a PostgreSQL URL
5. Deploy frontend `index.html` on **Render Static Site** or **Vercel**

### Option B: AWS EC2

```bash
# On Ubuntu 22.04 EC2
sudo apt update && sudo apt install python3.11 python3-pip nginx certbot -y

# Clone your repo, install deps, set env vars
# Run with gunicorn + nginx reverse proxy

pip install gunicorn
gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app --bind 0.0.0.0:8000

# Nginx config at /etc/nginx/sites-available/worklog:
# server {
#   listen 80;
#   server_name api.yourdomain.com;
#   location / { proxy_pass http://127.0.0.1:8000; }
# }
```

### Option C: Docker

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY backend/requirements.txt .
RUN pip install -r requirements.txt
COPY backend/ .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
docker build -t worklog-api .
docker run -p 8000:8000 \
  -e SECRET_KEY=your-secret \
  -e DATABASE_URL=postgresql://... \
  worklog-api
```

---

## Security Checklist for Production

- [ ] Change `SECRET_KEY` to a strong random value
- [ ] Set `CORS allow_origins` to your exact frontend domain only
- [ ] Use PostgreSQL (not SQLite) for production
- [ ] Enable HTTPS (SSL certificate via Let's Encrypt or Render's auto-SSL)
- [ ] Set `ACCESS_TOKEN_EXPIRE_MINUTES` to 480 (8 hours) or less
- [ ] Do NOT store passwords in plaintext — bcrypt is already used ✓
- [ ] Add rate limiting (slowapi library) on `/auth/login`

---

## Role Permissions Summary

| Action                     | Employee | Admin |
|---------------------------|----------|-------|
| Log own work today         | ✓        | ✓     |
| Edit own log (same day)    | ✓        | ✓     |
| View own history           | ✓        | ✓     |
| View all employee logs     | ✗        | ✓     |
| Edit any log               | ✗        | ✓     |
| Delete any log             | ✗        | ✓     |
| Export reports (XLSX/CSV)  | ✗        | ✓     |
| Manage employees           | ✗        | ✓     |

---

## Sample Test Data (Seed Script)

```python
# Run once to populate test data
import requests

BASE = "http://localhost:8000"

# Create admin
r = requests.post(f"{BASE}/auth/register", json={
    "name": "Harshad Patil", "email": "admin@deltametrology.in",
    "password": "admin123", "role": "admin"
})
token = r.json()["access_token"]
headers = {"Authorization": f"Bearer {token}"}

# Create employees
for emp in [
    {"name": "Rahul Deshmukh", "email": "rahul@deltametrology.in", "password": "emp123"},
    {"name": "Priya Kulkarni", "email": "priya@deltametrology.in", "password": "emp123"},
]:
    requests.post(f"{BASE}/users", json={**emp, "role": "employee"}, headers=headers)

print("Seed complete!")
```

---

## Environment Variables

| Variable                      | Default             | Description                        |
|------------------------------|---------------------|------------------------------------|
| `SECRET_KEY`                 | (insecure default)  | JWT signing key — **change this!** |
| `DATABASE_URL`               | sqlite:///worklog.db| PostgreSQL or SQLite URL           |
| `ACCESS_TOKEN_EXPIRE_MINUTES`| 480                 | Token validity in minutes          |

---

*Built for Delta Metrology — www.deltametrology.in*
