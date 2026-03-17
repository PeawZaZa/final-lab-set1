# INDIVIDUAL_REPORT_675432100401.md

## ข้อมูลผู้จัดทำ
- ชื่อ-นามสกุล: นายพนาวุฒน์ อภิปสันติ
- รหัสนักศึกษา: 67543210040-1
- กลุ่ม: ENGSE207 Software Architecture — Final Lab Set 1

## ขอบเขตงานที่รับผิดชอบ
- Task Service (`task-service/`)
- Log Service (`log-service/`)
- Frontend (`frontend/index.html`, `frontend/logs.html`)
- Docker Compose Integration (`docker-compose.yml`, `.env.example`)

## สิ่งที่ได้ดำเนินการด้วยตนเอง

### Task Service
เขียน CRUD routes ทั้งหมดสำหรับ Tasks โดยมี logic ว่า admin เห็น task ของทุกคน แต่ member เห็นเฉพาะ task ของตัวเอง ใช้ `authMiddleware.js` ตรวจสอบ JWT ก่อนทุก request และส่ง logEvent ไปที่ Log Service เมื่อมีการ create/delete task รวมถึงเมื่อ JWT invalid

เพิ่ม ownership check ใน PUT และ DELETE ว่าเฉพาะเจ้าของ task หรือ admin เท่านั้นที่แก้ไข/ลบได้ ถ้าไม่ใช่จะ return 403 Forbidden

### Log Service
เขียน service ใหม่ทั้งหมด รับ log จาก services อื่นผ่าน `POST /api/logs/internal` โดยไม่ต้องมี JWT เพราะเรียกภายใน Docker network เท่านั้น ส่วน `GET /api/logs/` และ `GET /api/logs/stats` ต้องมี JWT และ role=admin เท่านั้นถึงจะเข้าได้

### Frontend
ปรับ `index.html` โดยลบ Register tab, ลบ Users nav, เปลี่ยน `apiFetch()` → `fetch()` ธรรมดา, เปลี่ยน `user.name` → `user.username`, เปลี่ยน endpoint `/api/users/me` → `/api/auth/me`

สร้าง `logs.html` ใหม่ทั้งหมด มี filter by service/level, client-side search, auto-refresh ทุก 5 วินาที โดยดึงข้อมูลจาก Log Service DB แทนการใช้ localStorage

### Docker Compose
ตั้งค่า service dependencies ให้ถูกต้อง เช่น auth-service ต้อง wait postgres healthy ก่อน และ task-service ต้อง wait ทั้ง postgres และ auth-service ตั้งค่า environment variables และ network `taskboard-net` ให้ทุก service คุยกันได้

## ปัญหาที่พบและวิธีการแก้ไข

**ปัญหา 1: Task Service แสดง username ไม่ได้**
ตอนแรก query แค่ `SELECT * FROM tasks` เลยไม่มี field `username` แก้โดยทำ JOIN กับ table users เพื่อดึง `username` มาด้วย

**ปัญหา 2: Log Dashboard ไม่ขึ้น log ทันที**
เพราะ auto-refresh ทำงานเฉพาะหลัง login สำเร็จ แก้โดยเรียก `loadLogs()` ทันทีหลัง login และเรียก `startAutoRefresh()` เพื่อ set interval ทุก 5 วินาที

**ปัญหา 3: docker compose up แล้ว services ไม่ start เพราะ postgres ยังไม่พร้อม**
แก้โดยใส่ retry loop ใน `start()` function ของแต่ละ service ให้ลอง connect DB ซ้ำทุก 3 วินาที สูงสุด 10 ครั้ง

## สิ่งที่ได้เรียนรู้จากงานนี้
- **Microservices Architecture**: เข้าใจการแบ่ง service ตาม business domain และการที่ services คุยกันผ่าน HTTP REST ภายใน network
- **JWT Middleware**: เข้าใจการ verify JWT ใน middleware และการส่งต่อ `req.user` ให้ route handler ใช้งาน
- **Lightweight Logging**: เข้าใจการออกแบบ centralized logging ที่ services ส่ง log มารวมที่ Log Service แทนการเขียน log แยกกัน
- **Docker Compose Networking**: เข้าใจ service discovery ใน Docker — services เรียกกันด้วย service name แทน IP เช่น `http://log-service:3003`
- **Docker Health Check + depends_on**: เข้าใจการใช้ `condition: service_healthy` เพื่อให้ services start ตามลำดับที่ถูกต้อง

## แนวทางการพัฒนาต่อไปใน Set 2
- Deploy ขึ้น Cloud (AWS/GCP/Azure) และเปลี่ยนจาก localhost เป็น domain จริง
- เพิ่ม pagination บน Log Dashboard เพราะปัจจุบัน limit แค่ 200 รายการ
- เพิ่ม real-time log streaming ด้วย WebSocket แทน polling ทุก 5 วินาที
- แยก Frontend ออกเป็น proper SPA framework เช่น React/Vue แทน vanilla HTML
- เพิ่ม unit test สำหรับ Task Service และ Log Service
