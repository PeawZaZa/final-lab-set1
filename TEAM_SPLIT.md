# TEAM_SPLIT.md

## Team Members

| รหัสนักศึกษา | ชื่อ-นามสกุล |
|---|---|
| 67543210037-7 | นายปวริศ คูณศรี |
| 67543210040-1 | นายพนาวุฒน์ อภิปสันติ |

---

## Work Allocation

### Student 1: นายปวริศ คูณศรี (67543210037-7)

- **Auth Service** — `auth-service/` ทั้งหมด
  - `src/routes/auth.js` — POST /login, GET /verify, GET /me, GET /health
  - `src/middleware/jwtUtils.js` — generateToken, verifyToken
  - `src/db/db.js` — PostgreSQL connection pool
  - `src/index.js` — Express app setup + DB retry loop
  - `Dockerfile`, `package.json`
- **HTTPS + Nginx** — `nginx/` ทั้งหมด
  - `nginx.conf` — TLS config, reverse proxy, rate limiting, block /api/logs/internal
  - `Dockerfile`
  - `scripts/gen-certs.sh` — สร้าง self-signed certificate
- **Database** — `db/init.sql`
  - Schema: users, tasks, logs tables
  - Seed Users (bcrypt hash พร้อมใช้)
  - Seed Tasks ตั้งต้น

### Student 2: นายพนาวุฒน์ อภิปสันติ (67543210040-1)

- **Task Service** — `task-service/` ทั้งหมด
  - `src/routes/tasks.js` — GET/, POST/, PUT/:id, DELETE/:id
  - `src/middleware/authMiddleware.js` — JWT verification middleware
  - `src/middleware/jwtUtils.js` — verifyToken (shared logic)
  - `src/db/db.js` — PostgreSQL connection pool
  - `src/index.js` — Express app setup
  - `Dockerfile`, `package.json`
- **Log Service** — `log-service/` ทั้งหมด
  - `src/index.js` — POST /internal, GET /, GET /stats, GET /health
  - `Dockerfile`, `package.json`
- **Frontend** — `frontend/` ทั้งหมด
  - `index.html` — Task Board UI (Login, CRUD Tasks, JWT Inspector)
  - `logs.html` — Log Dashboard (admin only, auto-refresh)
  - `Dockerfile`
- **Docker Compose Integration** — `docker-compose.yml`, `.env.example`, `.gitignore`

---

## Shared Responsibilities

- Architecture diagram และ system design
- End-to-end testing (ทดสอบ Test Cases T1–T11 ร่วมกัน)
- `README.md` และ screenshots
- Code review ระหว่างสมาชิก

---

## Integration Notes

งานของทั้งสองคนเชื่อมต่อกันผ่าน 3 จุดหลัก:

1. **Auth → Task/Log**: ทั้ง Task Service และ Log Service ใช้ `JWT_SECRET` เดียวกัน (ผ่าน environment variable) เพื่อ verify token ที่ Auth Service ออกให้

2. **Task/Auth → Log**: ทั้ง Auth Service และ Task Service เรียก `logEvent()` ส่ง HTTP POST ไปที่ `http://log-service:3003/api/logs/internal` ภายใน Docker network — Nginx บล็อก path นี้จากภายนอก

3. **Nginx → All Services**: Nginx ของนายปวริศทำหน้าที่ route request ไปยัง services ทุกตัวของนายพนาวุฒน์ผ่าน Docker internal network (`taskboard-net`)
