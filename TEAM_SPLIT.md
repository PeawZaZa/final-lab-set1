# TEAM_SPLIT.md

## ข้อมูลกลุ่ม
- รายวิชา: ENGSE207 Software Architecture

## รายชื่อสมาชิก
- 67543210037-7 นายปวริศ คูณศรี
- 67543210040-1 นายพนาวุฒน์ อภิปสันติ

## การแบ่งงานหลัก

### สมาชิกคนที่ 1: นายปวริศ คูณศรี (67543210037-7)
รับผิดชอบงานหลักดังต่อไปนี้
- **Auth Service** (`auth-service/`) — POST /login, GET /verify, GET /me, GET /health, jwtUtils, DB connection pool
- **HTTPS + Nginx** (`nginx/`) — TLS config, reverse proxy, rate limiting, block `/api/logs/internal`
- **Script gen-certs.sh** (`scripts/gen-certs.sh`) — สร้าง self-signed certificate
- **Database Schema** (`db/init.sql`) — ออกแบบ tables users/tasks/logs, Seed Users พร้อม bcrypt hash

### สมาชิกคนที่ 2: นายพนาวุฒน์ อภิปสันติ (67543210040-1)
รับผิดชอบงานหลักดังต่อไปนี้
- **Task Service** (`task-service/`) — CRUD Tasks, JWT middleware, ownership check, logEvent integration
- **Log Service** (`log-service/`) — POST /internal, GET /, GET /stats, GET /health
- **Frontend** (`frontend/`) — Task Board UI (`index.html`), Log Dashboard (`logs.html`) พร้อม auto-refresh
- **Docker Compose Integration** (`docker-compose.yml`, `.env.example`, `.gitignore`) — service dependencies, health check, network config

## งานที่ดำเนินการร่วมกัน
- ออกแบบ architecture diagram และ system design
- ทดสอบระบบแบบ end-to-end (Test Cases T1–T11)
- จัดทำ README.md และ screenshots
- Code review ระหว่างสมาชิก

## เหตุผลในการแบ่งงาน
แบ่งงานตาม service boundary โดยสมาชิกคนที่ 1 รับผิดชอบด้าน security layer (Auth, HTTPS/Nginx, Database) และสมาชิกคนที่ 2 รับผิดชอบด้าน application layer (Task, Log, Frontend, Deployment) เพื่อให้แต่ละคนเข้าใจงานในส่วนของตนอย่างครบถ้วน และสามารถประสานงานกันผ่าน interface ที่ชัดเจน

## สรุปการเชื่อมโยงงานของสมาชิก
งานของทั้งสองคนเชื่อมต่อกันผ่าน 3 จุดหลัก

1. **Auth → Task/Log**: ทั้ง Task Service และ Log Service ใช้ `JWT_SECRET` เดียวกัน (ผ่าน environment variable) เพื่อ verify token ที่ Auth Service ออกให้
2. **Task/Auth → Log**: ทั้ง Auth Service และ Task Service เรียก `logEvent()` ส่ง HTTP POST ไปที่ `http://log-service:3003/api/logs/internal` ภายใน Docker network — Nginx บล็อก path นี้จากภายนอก
3. **Nginx → All Services**: Nginx ของนายปวริศทำหน้าที่ route request ไปยัง services ทุกตัวของนายพนาวุฒน์ผ่าน Docker internal network (`taskboard-net`)
