# INDIVIDUAL_REPORT_675432100377.md

## 1. ข้อมูลผู้จัดทำ

| | |
|---|---|
| **ชื่อ-นามสกุล** | นายปวริศ คูณศรี |
| **รหัสนักศึกษา** | 67543210037-7 |
| **วิชา** | ENGSE207 Software Architecture |
| **งาน** | Final Lab Set 1 — Microservices + HTTPS + Basic Logging |

---

## 2. ส่วนที่รับผิดชอบ

- Auth Service (`auth-service/`)
- HTTPS + Nginx API Gateway (`nginx/`)
- Database Schema + Seed Users (`db/init.sql`)
- Script สร้าง Self-Signed Certificate (`scripts/gen-certs.sh`)

---

## 3. สิ่งที่ลงมือพัฒนาด้วยตนเอง

### Auth Service
เขียน `POST /api/auth/login` โดยใช้ `bcryptjs` เปรียบเทียบรหัสผ่านกับ hash ที่เก็บใน DB พร้อมป้องกัน Timing Attack ด้วย Dummy Hash เพื่อไม่ให้ response time ต่างกันระหว่าง user ที่มีและไม่มีในระบบ หลัง login สำเร็จจะออก JWT ด้วย `jsonwebtoken` โดยใส่ `sub`, `email`, `role`, `username` ลงใน payload และส่ง logEvent ไปที่ Log Service แบบ fire-and-forget

เขียน `GET /api/auth/verify` และ `GET /api/auth/me` สำหรับตรวจสอบ token และดึงข้อมูล user จาก DB

### Nginx Config
ตั้งค่า TLS ด้วย Self-Signed Certificate บน port 443 และ redirect HTTP :80 → HTTPS :443 ด้วย `return 301` ตั้งค่า Rate Limiting 2 zones คือ `login_limit` (5 req/min) สำหรับ login และ `api_limit` (30 req/min) สำหรับ endpoint อื่น บล็อก `/api/logs/internal` จากภายนอกด้วย `return 403` เพื่อให้เฉพาะ services ภายใน Docker network เท่านั้นที่เรียกได้

### Database Schema
ออกแบบ 3 tables ได้แก่ `users`, `tasks`, `logs` พร้อม index บน logs table และ Seed Users 3 คน (alice/bob/admin) โดย hash รหัสผ่านด้วย bcrypt saltRounds=10

---

## 4. ปัญหาที่พบและวิธีแก้ไข

**ปัญหา 1: รัน docker compose บน Windows แล้วขึ้น env variable not set**
ไม่ได้สร้างไฟล์ `.env` ก่อนรัน แก้โดยรัน `Copy-Item .env.example .env` ก่อน `docker compose up`

**ปัญหา 2: Browser แสดง "Not Secure" / NET::ERR_CERT_AUTHORITY_INVALID**
เป็นเรื่องปกติของ Self-Signed Certificate เพราะ browser ไม่รู้จัก CA ที่ออก cert แก้โดยกด Advanced → Proceed to localhost สำหรับ development

**ปัญหา 3: Timing Attack บน Login endpoint**
หากตรวจสอบ user ก่อนแล้วค่อย compare password — response time จะเร็วกว่ากรณีที่ user ไม่มีในระบบ ทำให้ attacker ดักดูได้ว่า email นั้นมีหรือไม่ แก้โดยใช้ Dummy Hash เพื่อให้ bcrypt.compare ทำงานทุกครั้งไม่ว่า user จะมีหรือไม่มี

---

## 5. สิ่งที่ได้เรียนรู้

- **TLS/HTTPS**: เข้าใจความแตกต่างระหว่าง Self-Signed Certificate กับ CA-Signed Certificate และเหตุผลที่ browser แสดง warning
- **JWT**: เข้าใจโครงสร้าง 3 ส่วน (Header.Payload.Signature) และทำไม Payload ถึงถูกอ่านได้โดยใครก็ตาม แต่แก้ไขไม่ได้เพราะมี Signature
- **Nginx Rate Limiting**: เข้าใจการใช้ `limit_req_zone` และ `limit_req` เพื่อป้องกัน Brute Force บน login endpoint
- **bcrypt Timing Attack**: เข้าใจ concept ของ Timing Attack และวิธีป้องกันด้วย Dummy Hash
- **Docker Networking**: services คุยกันผ่าน service name ภายใน Docker network ได้โดยตรง เช่น `http://log-service:3003`

---

## 6. แนวทางที่จะพัฒนาต่อใน Set 2

- เปลี่ยนจาก Self-Signed Certificate เป็น Let's Encrypt certificate เมื่อ deploy ขึ้น Cloud เพื่อกำจัด browser warning
- แยก Database เป็น Database per Service แทนที่จะใช้ shared DB เพื่อให้แต่ละ service เป็นอิสระจากกันอย่างแท้จริง
- เพิ่ม Refresh Token เพื่อให้ JWT ที่หมดอายุ renew ได้โดยไม่ต้อง login ใหม่
- เพิ่ม HTTPS environment variable สำหรับ production แทนที่จะ hardcode ใน Nginx config
