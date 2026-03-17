# ENGSE207 Software Architecture
# Final Lab Set 1 — Microservices + HTTPS + Basic Logging

> **วิชา:** ENGSE207 Software Architecture  
> **มหาวิทยาลัย:** มหาวิทยาลัยเทคโนโลยีราชมงคลล้านนา  
> **รูปแบบงาน:** งานกลุ่ม กลุ่มละ 2 คน  

---

## รายชื่อสมาชิก

| รหัสนักศึกษา | ชื่อ-นามสกุล |
|---|---|
| 67553210037-7 | นายปวริศ คูณศรี |
| 67543210040-1 | นายพนาวุฒน์ อภิปสันติ |

---

## ภาพรวม

ระบบ **Task Board Microservices** ประกอบด้วย 5 services:
- **Nginx** — API Gateway + TLS Termination + Rate Limiter
- **Auth Service** — Login (Seed Users only) + ออก JWT
- **Task Service** — CRUD Tasks + JWT middleware
- **Log Service** — รับ log ภายใน + API ดึง log สำหรับ admin
- **PostgreSQL** — ฐานข้อมูลร่วม (users, tasks, logs)

---

## Architecture Diagram

```
Browser / Postman
       │
       │ HTTPS :443  (HTTP :80 → redirect HTTPS)
       ▼
┌──────────────────────────────────────────────────────────────┐
│  🛡️ Nginx (API Gateway + TLS Termination + Rate Limiter)     │
│                                                              │
│  /api/auth/*         → auth-service:3001  (ไม่ต้องมี JWT)    │
│  /api/tasks/*        → task-service:3002  [JWT required]     │
│  /api/logs/internal  → BLOCKED (403 จาก Nginx)               │
│  /api/logs/*         → log-service:3003   [JWT + admin only] │
│  /                   → frontend:80        (Static HTML)      │
└──────┬──────────────┬─────────────────┬──────────────────────┘
       │              │                 │
       ▼              ▼                 ▼
┌──────────────┐ ┌───────────────┐ ┌──────────────────┐
│ 🔑 Auth Svc  │ │ 📋 Task Svc   │ │ 📝 Log Service   │
│   :3001      │ │   :3002       │ │   :3003          │
│              │ │               │ │                  │
│ • POST login │ │ • CRUD Tasks  │ │ • POST /internal │
│ • GET verify │ │ • JWT Guard   │ │ • GET  /         │
│ • GET me     │ │ • logEvent()→ │ │ • GET  /stats    │
│ • logEvent() │ │   log-service │ │ • เก็บลง DB       │
└──────┬───────┘ └───────┬───────┘ └──────────────────┘
       │                 │                    │
       └─────────────────┴────────────────────┘
                         │
               ┌─────────────────────┐
               │  🗄️ PostgreSQL      │
               │  (1 shared DB)      │
               │  • users   table    │
               │  • tasks   table    │
               │  • logs    table    │
               └─────────────────────┘
```

---

## โครงสร้าง Repository

```
final-lab-set1/
├── README.md
├── docker-compose.yml
├── .env.example
├── .gitignore
├── nginx/
│   ├── nginx.conf
│   ├── Dockerfile
│   └── certs/          ← สร้างด้วย gen-certs.sh (ไม่ commit)
├── frontend/
│   ├── Dockerfile
│   ├── index.html      ← Task Board UI
│   └── logs.html       ← Log Dashboard (admin only)
├── auth-service/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       ├── index.js
│       ├── routes/auth.js
│       ├── middleware/jwtUtils.js
│       └── db/db.js
├── task-service/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       ├── index.js
│       ├── routes/tasks.js
│       ├── middleware/
│       │   ├── authMiddleware.js
│       │   └── jwtUtils.js
│       └── db/db.js
├── log-service/
│   ├── Dockerfile
│   ├── package.json
│   └── src/index.js
├── db/
│   └── init.sql
├── scripts/
│   └── gen-certs.sh
└── screenshots/
```

---

## Seed Users

| Username | Email | Password | Role |
|---|---|---|---|
| alice | alice@lab.local | alice123 | member |
| bob | bob@lab.local | bob456 | member |
| admin | admin@lab.local | adminpass | admin |

---

## วิธีรันระบบ

```bash
# 1. Clone repository
git clone https://github.com/PeawZaZa/final-lab-set1.git
cd final-lab-set1

# 2. สร้าง SSL Certificate
chmod +x scripts/gen-certs.sh
./scripts/gen-certs.sh

# 3. สร้างไฟล์ .env
cp .env.example .env

# 4. Build และรัน
docker compose up --build

# 5. เปิด browser
#    https://localhost  (จะมี cert warning — กด Advanced → Proceed)
#    http://localhost   (redirect ไป HTTPS อัตโนมัติ)

# รีเซ็ต DB
docker compose down -v
docker compose up --build
```

---

## HTTPS Flow

1. Browser เรียก `https://localhost:443`
2. Nginx รับ request ผ่าน TLS (Self-Signed Certificate)
3. Nginx ทำ TLS Termination แล้ว forward ไปยัง services ผ่าน HTTP ภายใน Docker network
4. HTTP `:80` redirect ไป HTTPS `:443` ด้วย `301 Redirect`

> Self-signed certificate จะมี browser warning — กด **Advanced → Proceed to localhost** ได้

---

## JWT Flow

1. Client ส่ง `POST /api/auth/login` พร้อม email + password
2. Auth Service ตรวจสอบกับ DB ด้วย bcrypt
3. หากถูกต้อง ออก JWT token (`sub`, `email`, `role`, `username`)
4. Client เก็บ token ใน `localStorage`
5. ทุก request ไปยัง Task Service ต้องแนบ `Authorization: Bearer <token>`
6. Task Service ตรวจ JWT ด้วย `verifyToken()` ก่อนให้ผ่าน

---

## Logging Flow

1. Auth Service และ Task Service เรียก `logEvent()` ส่ง log ไปที่ Log Service
2. Log Service รับผ่าน `POST /api/logs/internal` (ภายใน Docker network เท่านั้น)
3. **Nginx บล็อก `/api/logs/internal` จากภายนอก** → return 403
4. Log เก็บลง PostgreSQL table `logs`
5. Admin ดู log ได้ผ่าน `GET /api/logs/` (ต้องมี JWT + role=admin)

---

## API Endpoints

### Auth Service (:3001)
| Method | Path | Auth | Description |
|---|---|---|---|
| POST | /api/auth/login | ❌ | Login ด้วย email + password |
| GET | /api/auth/verify | Bearer | ตรวจสอบ token |
| GET | /api/auth/me | Bearer | ดูข้อมูลผู้ใช้ปัจจุบัน |
| GET | /api/auth/health | ❌ | Health check |

### Task Service (:3002)
| Method | Path | Auth | Description |
|---|---|---|---|
| GET | /api/tasks/ | Bearer | ดู tasks (admin=ทั้งหมด, member=ของตัวเอง) |
| POST | /api/tasks/ | Bearer | สร้าง task ใหม่ |
| PUT | /api/tasks/:id | Bearer | แก้ไข task |
| DELETE | /api/tasks/:id | Bearer | ลบ task |
| GET | /api/tasks/health | ❌ | Health check |

### Log Service (:3003)
| Method | Path | Auth | Description |
|---|---|---|---|
| POST | /api/logs/internal | ❌ (internal) | รับ log จาก services (Nginx บล็อกจากนอก) |
| GET | /api/logs/ | Bearer (admin) | ดู logs ทั้งหมด |
| GET | /api/logs/stats | Bearer (admin) | สถิติ logs |
| GET | /api/logs/health | ❌ | Health check |

---

## วิธีทดสอบ API

```bash
BASE="https://localhost"

# Login
TOKEN=$(curl -sk -X POST $BASE/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@lab.local","password":"alice123"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

# Create Task
curl -sk -X POST $BASE/api/tasks/ \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"Test Task","priority":"high"}'

# Get Tasks
curl -sk $BASE/api/tasks/ -H "Authorization: Bearer $TOKEN"

# Test 401 (no token)
curl -sk $BASE/api/tasks/

# Test 403 (member accessing logs)
curl -sk $BASE/api/logs/ -H "Authorization: Bearer $TOKEN"
```

---

## Known Limitations

- Self-signed certificate ทำให้ browser แสดง warning — ใช้ได้สำหรับ development เท่านั้น
- ใช้ shared database 1 ตัวสำหรับทุก services (ไม่ใช่ database per service)
- ไม่มี register — ใช้ Seed Users เท่านั้น
- Rate limiting ทำที่ Nginx layer เท่านั้น
