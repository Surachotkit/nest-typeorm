# nest-typeorm

โปรเจคตัวอย่างสำหรับเรียนรู้ [NestJS](https://nestjs.com) ร่วมกับ [TypeORM](https://typeorm.io) และ PostgreSQL
มี REST API จัดการผู้ใช้ (CRUD) เป็นตัวอย่าง

## เทคโนโลยีที่ใช้

- NestJS 12 (ESM)
- TypeORM + `@nestjs/typeorm`
- PostgreSQL (ไดรเวอร์ `pg`)
- `class-validator` / `class-transformer` สำหรับตรวจข้อมูลที่รับเข้ามา
- `@nestjs/config` สำหรับอ่านค่าจาก `.env`
- Vitest (ทดสอบ), oxlint (lint)

## สิ่งที่ต้องมี

- Node.js 20 ขึ้นไป
- PostgreSQL (ติดตั้งในเครื่อง หรือใช้ Docker ตาม `docker-compose.yml`)

## เริ่มต้นใช้งาน

### 1. ติดตั้งแพ็กเกจ

```bash
npm install
```


### 2. ตั้งค่า database

สร้างไฟล์ `.env` ที่รากโปรเจค แล้วแก้ให้ตรงกับ Postgres ของเครื่องคุณ

```env
PORT=3000
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=postgres
DB_PASSWORD=postgres
DB_NAME=nest_typeorm
```

**แบบ A: ใช้ Docker** (ค่าข้างบนตรงกับ `docker-compose.yml` อยู่แล้ว)

```bash
docker compose up -d
```

**แบบ B: ใช้ Postgres ในเครื่อง**

สร้าง database ก่อน แล้วแก้ `DB_USERNAME` / `DB_PASSWORD` ให้ตรงกับเครื่อง

```bash
createdb nest_typeorm
```

> Postgres ที่ติดตั้งด้วย Homebrew มักสร้าง role ตามชื่อผู้ใช้ในเครื่อง (ไม่ใช่ `postgres`)
> และเชื่อมต่อจาก localhost โดยไม่ต้องใช้รหัสผ่าน ดูรายชื่อ role ได้ด้วย
> `psql -h localhost -d postgres -c '\du'`

> ถ้า Docker กับ Postgres ในเครื่องใช้พอร์ต 5432 พร้อมกัน จะชนกัน ให้เปลี่ยนพอร์ตฝั่งเครื่องใน
> `docker-compose.yml` (เช่น `5433:5432`) แล้วแก้ `DB_PORT`

### 3. รันแอป

```bash
npm run start:dev
```

แอปเปิดที่ http://localhost:3000 ตารางจะถูกสร้างให้อัตโนมัติ (ดูหัวข้อ [ข้อควรระวัง](#ข้อควรระวัง))

## API

| Method | Path | คำอธิบาย |
|---|---|---|
| `POST` | `/users` | สร้างผู้ใช้ |
| `GET` | `/users` | ดึงผู้ใช้ทั้งหมด |
| `GET` | `/users/:id` | ดึงผู้ใช้ตาม id |
| `PATCH` | `/users/:id` | แก้ไขผู้ใช้ (ส่งเฉพาะฟิลด์ที่ต้องการแก้) |
| `DELETE` | `/users/:id` | ลบผู้ใช้ |

ตัวอย่าง

```bash
# สร้าง
curl -X POST localhost:3000/users \
  -H 'content-type: application/json' \
  -d '{"email":"a@b.com","name":"John doe"}'

# ดึงทั้งหมด
curl localhost:3000/users

# แก้ไข
curl -X PATCH localhost:3000/users/1 \
  -H 'content-type: application/json' \
  -d '{"name":"New Name"}'

# ลบ
curl -X DELETE localhost:3000/users/1
```

ข้อมูลที่ไม่ถูกต้อง (เช่น อีเมลผิดรูปแบบ) จะได้ `400` และไม่พบผู้ใช้จะได้ `404`
อีเมลต้องไม่ซ้ำกัน

## โครงสร้างโปรเจค

```
src/
├── main.ts                  # จุดเริ่มต้น เปิด ValidationPipe
├── app.module.ts            # โมดูลหลัก ตั้งค่า config และการเชื่อมต่อ database
└── users/
    ├── user.entity.ts       # Entity: ตัวแทนตาราง users
    ├── dto/                 # รูปแบบข้อมูลที่รับเข้ามา + กฎตรวจสอบ
    ├── users.controller.ts  # กำหนดเส้นทาง URL
    ├── users.service.ts     # ตรรกะ + คุยกับ database ผ่าน Repository
    └── users.module.ts      # ประกอบส่วนต่าง ๆ ของฟีเจอร์ users
```

ลำดับการทำงานของ request

```
Request → Controller → Service → Repository (TypeORM) → PostgreSQL
              ↑ DTO + ValidationPipe ตรวจข้อมูลก่อนเข้า Controller
```

## ข้อควรระวัง

- `synchronize: true` ใน `src/app.module.ts` ทำให้ TypeORM สร้างและแก้ตารางตาม entity อัตโนมัติ
  สะดวกตอนพัฒนา แต่แก้หรือเปลี่ยนชื่อ property อาจทำให้คอลัมน์และข้อมูลถูกลบ
  **อย่าเปิดกับข้อมูลจริง** ให้ใช้ [migration](https://typeorm.io/migrations) แทนใน production
- ไฟล์ `.env` อยู่ใน `.gitignore` อย่า commit รหัสผ่านจริงขึ้น repository

## เพิ่ม entity ใหม่

1. สร้างคลาสด้วย `@Entity()` และกำหนดคอลัมน์ด้วย `@Column()`
2. ลงทะเบียนในโมดูลด้วย `TypeOrmModule.forFeature([MyEntity])`
3. ฉีด `@InjectRepository(MyEntity)` ใน service แล้วเรียก `find`, `save`, `remove` ฯลฯ
4. เพิ่มโมดูลใหม่ใน `imports` ของ `AppModule`
