# รายงานอ้างอิงโครงงาน FoodFlow (ฉบับละเอียดพร้อมโค้ด)

เอกสารนี้จัดทำเพื่อใช้เป็นตัวอย่างสำหรับเขียนรายงานโครงงาน โดยอ้างอิงจากโค้ดจริงในโปรเจกต์ `FoodFlow_FullStack_Code` ของคุณ

---

## บทที่ 1 บทนำ

### 1.1 วัตถุประสงค์ของโครงงาน
1. พัฒนาระบบสั่งอาหารผ่านเว็บให้ลูกค้าสามารถสั่งอาหารจากโต๊ะได้ด้วย QR Code
2. พัฒนาระบบหลังบ้านให้ผู้จัดการร้านบริหารเมนู ออเดอร์ โต๊ะ และสมาชิกได้
3. ลดความผิดพลาดในการรับออเดอร์ด้วยการบันทึกข้อมูลแบบดิจิทัลและติดตามสถานะอัตโนมัติ
4. สร้างรายงานยอดขายและเมนูขายดีเพื่อสนับสนุนการตัดสินใจของผู้บริหารร้าน
5. วางโครงสร้างระบบให้รองรับการขยายฟีเจอร์ในอนาคต เช่น โปรโมชัน/สมาชิก/การวิเคราะห์ข้อมูล

### 1.2 ขอบเขตของโครงงาน
1. รองรับการจัดการเมนูอาหาร (เพิ่ม/แก้ไข/ลบ/ดูรายการ)
2. รองรับการจัดการสมาชิกและเข้าสู่ระบบของสมาชิก
3. รองรับการสร้างออเดอร์และรายละเอียดออเดอร์หลายรายการต่อ 1 บิล
4. รองรับสถานะออเดอร์และสถานะการชำระเงิน
5. รองรับการสร้างและตรวจสอบ QR Token ต่อโต๊ะ
6. รองรับแดชบอร์ดผู้จัดการและรายงาน 2 รูปแบบ (ยอดขายตามเวลา, เมนูขายดี)
7. ใช้งานผ่านเว็บแอป (Frontend + API + SQLite)

### 1.3 ประโยชน์ที่คาดว่าจะได้รับ
1. ร้านอาหารรับออเดอร์ได้รวดเร็วและเป็นระบบมากขึ้น
2. ลดความผิดพลาดจากการจดออเดอร์ด้วยมือ
3. ผู้จัดการติดตามสถานะการขายและการชำระเงินได้แบบใกล้เคียงเรียลไทม์
4. ได้ข้อมูลเชิงสถิติเพื่อวางแผนการตลาดและการจัดการเมนู
5. เป็นต้นแบบระบบ Full Stack ที่นำไปต่อยอดใช้งานจริงได้

### 1.4 เครื่องมือที่ใช้ในโครงงาน
1. Backend: `Node.js`, `Express.js`
2. Frontend: `EJS`, `HTML/CSS/JavaScript`
3. Database: `SQLite3`
4. Security/Infrastructure: `helmet`, `csurf`, `express-session`, `express-rate-limit`
5. Testing: `node --test`, `supertest`
6. Utility: `dotenv`, `multer`

---

## บทที่ 2 ระบบสำหรับจัดการเมนูอาหาร การสั่งอาหาร และข้อมูลลูกค้า

### 2.1 ภาพรวมสถาปัตยกรรมระบบ
ระบบประกอบด้วย 3 ชั้นหลัก
1. Frontend Server (`app.js`) สำหรับ render หน้าเว็บและ proxy ไปยัง API
2. API Server (`server/createApiApp.js`) สำหรับจัดการข้อมูลธุรกิจทั้งหมด
3. SQLite Database (`database/database.sqlite`) สำหรับเก็บข้อมูลถาวร

ตัวอย่างโค้ดสรุปการประกอบระบบ API:

```js
const models = createModels(dbClient);
const menusController = createMenusController({ menuModel: models.menuModel });
const membershipsController = createMembershipsController({
  membershipModel: models.membershipModel,
  passwordService,
});
const ordersController = createOrdersController({
  orderModel: models.orderModel,
  orderDetailModel: models.orderDetailModel,
  diningTableModel: models.diningTableModel,
  membershipModel: models.membershipModel,
  menuModel: models.menuModel,
  onOrderEvent: publishOrderEvent,
  subscribeOrderEvents,
});
```

### 2.2 ฟิลด์ในระบบฐานข้อมูล

โค้ดโครงสร้างตาราง (ย่อ) จาก `database/schema.js`:

```sql
CREATE TABLE IF NOT EXISTS Menu (
  menu_id INTEGER PRIMARY KEY AUTOINCREMENT,
  menu_name TEXT NOT NULL,
  category_name TEXT,
  price REAL NOT NULL,
  stock INTEGER,
  tags TEXT,
  status TEXT DEFAULT 'available',
  image_path TEXT
);

CREATE TABLE IF NOT EXISTS Membership (
  membership_id INTEGER PRIMARY KEY AUTOINCREMENT,
  member_name TEXT,
  member_lastname TEXT,
  phone TEXT,
  email TEXT NOT NULL UNIQUE,
  password_hash TEXT,
  salt TEXT,
  tier TEXT DEFAULT 'basic',
  points INTEGER DEFAULT 0,
  created_at TEXT,
  active INTEGER DEFAULT 1
);

CREATE TABLE IF NOT EXISTS Orders (
  order_id INTEGER PRIMARY KEY AUTOINCREMENT,
  table_id INTEGER,
  membership_id INTEGER,
  order_date TEXT,
  total_price REAL DEFAULT 0,
  order_status TEXT DEFAULT 'pending',
  payment_status TEXT DEFAULT 'unpaid',
  paid_at TEXT,
  receipt_no TEXT,
  service_charge REAL DEFAULT 0,
  tax REAL DEFAULT 0
);
```

#### 2.2.1 ตาราง `Menu`
1. `menu_id` รหัสเมนู (Primary Key)
2. `menu_name` ชื่อเมนู
3. `category_name` หมวดหมู่เมนู
4. `price` ราคาเมนู
5. `stock` จำนวนคงเหลือ
6. `tags` แท็กเมนู
7. `status` สถานะพร้อมขาย
8. `image_path` path รูปภาพเมนู

#### 2.2.2 ตาราง `Membership`
1. `membership_id` รหัสสมาชิก
2. `member_name` ชื่อสมาชิก
3. `member_lastname` นามสกุลสมาชิก
4. `phone` เบอร์โทร
5. `email` อีเมล (Unique)
6. `password_hash` รหัสผ่านแบบ hash
7. `salt` ค่า salt สำหรับ hash
8. `tier` ระดับสมาชิก
9. `points` คะแนนสะสม
10. `created_at` วันที่สมัคร
11. `active` สถานะการใช้งานสมาชิก

#### 2.2.3 ตาราง `Orders`
1. `order_id` รหัสออเดอร์
2. `table_id` อ้างอิงโต๊ะ
3. `membership_id` อ้างอิงสมาชิก
4. `order_date` วันเวลาสั่ง
5. `total_price` ยอดรวมก่อนค่าใช้จ่ายเพิ่ม
6. `order_status` สถานะออเดอร์
7. `payment_status` สถานะการชำระเงิน
8. `paid_at` วันเวลาชำระเงิน
9. `receipt_no` เลขใบเสร็จ
10. `service_charge` ค่าบริการ
11. `tax` ภาษี

#### 2.2.4 ตาราง `OrderDetail`
1. `order_detail_id` รหัสรายการย่อย
2. `order_id` อ้างอิงออเดอร์หลัก
3. `menu_id` อ้างอิงเมนู
4. `quantity` จำนวน
5. `sub_total` ยอดรวมของรายการย่อย

#### 2.2.5 ตาราง `DiningTable`
1. `table_id` รหัสโต๊ะ
2. `table_name` ชื่อโต๊ะ
3. `status` สถานะโต๊ะ (`open`, `occupied`, `billing`, `closed`)
4. `created_at` วันที่สร้างข้อมูลโต๊ะ

#### 2.2.6 ตาราง `TableQR`
1. `token` token QR (Primary Key)
2. `table_id` อ้างอิงโต๊ะ
3. `created_at` วันที่สร้าง token
4. `expires_at` วันที่หมดอายุ (ถ้ามี)
5. `used` สถานะการใช้ token

---

### 2.3 ฟังก์ชันพื้นฐานการทำงานของระบบ

### 2.3.1 ฟังก์ชันเปิดระบบ Frontend + Security
หน้าที่หลัก:
1. ตั้งค่า view engine, static files, body parser
2. เปิด session เพื่อรองรับ login ผู้ดูแล
3. เปิด CSRF protection สำหรับ route ที่ไม่ใช่ API
4. proxy endpoint บางส่วนไป API server

โค้ดอ้างอิง:

```js
web.use(
  session({
    secret: SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
    cookie: {
      secure: COOKIE_SECURE,
      httpOnly: true,
      sameSite: COOKIE_SAME_SITE,
      maxAge: COOKIE_MAX_AGE_MS,
    },
  }),
);

const csrfProtection = csrf();
web.use((req, res, next) => {
  if (req.path.startsWith("/api")) return next();
  return csrfProtection(req, res, next);
});
```

### 2.3.2 ฟังก์ชันจัดการเมนูอาหาร (CRUD)
การเพิ่มเมนูใน `menusController.create`:

```js
if (!(payload.menu_name || "").trim()) {
  return res.status(400).json({ error: "menu_name is required" });
}
const row = await menuModel.create({
  menu_name: (payload.menu_name || "").trim(),
  category_name: (payload.category_name || "").trim() || null,
  price: Number(payload.price) || 0,
  status: (payload.status || "available").trim().toLowerCase(),
  image_path: (payload.image_path || "").trim() || null,
});
```

สรุปการทำงาน:
1. ตรวจสอบว่ามีชื่อเมนู
2. normalize ค่าที่รับเข้ามา
3. บันทึกลงฐานข้อมูล
4. ส่งผลลัพธ์กลับแบบ JSON

### 2.3.3 ฟังก์ชันสมัครสมาชิก
ฟังก์ชัน `membershipsController.create`:

```js
if (!member_name || !email || !password) {
  return res.status(400).json({ error: "member_name, email, and password are required" });
}

const existing = await membershipModel.findByEmail(email);
if (existing) return res.status(409).json({ error: "email already registered" });

const { salt, passwordHash } = passwordService.hashPassword(password);
```

สรุปการทำงาน:
1. validate ฟิลด์บังคับ
2. ตรวจสอบ email ซ้ำ
3. hash password ด้วย PBKDF2
4. บันทึกข้อมูลสมาชิกพร้อม `created_at`

### 2.3.4 ฟังก์ชันล็อกอินสมาชิก
ฟังก์ชัน `authController.login`:

```js
const member = await membershipModel.findByEmail(email);
if (!member) return res.status(401).json({ error: "invalid credentials" });

const isValid = passwordService.verifyPassword(password, member.salt || "", member.password_hash);
if (!isValid) return res.status(401).json({ error: "invalid credentials" });
```

สรุปการทำงาน:
1. ค้นหาสมาชิกจาก email
2. verify password hash
3. ส่งข้อมูลโปรไฟล์ขั้นต่ำกลับเมื่อ login สำเร็จ

### 2.3.5 ฟังก์ชันสร้างออเดอร์
ฟังก์ชัน `ordersController.create`:

```js
const normalizedDetails = Array.isArray(details)
  ? details.filter((item) => Number.isFinite(Number(item.menu_id)))
  : [];

const validDetails = [];
for (const item of normalizedDetails) {
  const menuId = Number(item.menu_id);
  const qty = Number.isFinite(Number(item.quantity)) && Number(item.quantity) > 0 ? Number(item.quantity) : 1;
  const menuRow = await menuModel.findById(menuId);
  if (!menuRow) continue;
  validDetails.push({ menu: menuRow, qty });
}

if (!validDetails.length) {
  return res.status(400).json({ error: "No valid menu items in order" });
}
```

และคำนวณยอดรวม:

```js
let total = 0;
for (const detail of validDetails) {
  const subTotal = (detail.menu.price || 0) * detail.qty;
  total += subTotal;
  await orderDetailModel.create({
    order_id: orderId,
    menu_id: detail.menu.menu_id,
    quantity: detail.qty,
    sub_total: subTotal,
  });
}
await orderModel.updateTotalPrice(orderId, total);
```

สรุปการทำงาน:
1. รับรายการเมนูในบิล
2. ตรวจสอบเมนูจริงในฐานข้อมูล
3. บันทึก Orders ก่อน แล้วบันทึก OrderDetail ทีละรายการ
4. คำนวณ `total_price` อัตโนมัติ
5. ส่ง event ให้หน้าจอที่ subscribe อยู่

### 2.3.6 ฟังก์ชันแก้ไขออเดอร์
หลักการใน `ordersController.update`:
1. ตรวจสอบ `order_id`
2. อัปเดตข้อมูลหัวบิล
3. ถ้ามี details ใหม่ ให้ลบรายละเอียดเดิมแล้วบันทึกใหม่
4. คำนวณ total ใหม่
5. sync สถานะโต๊ะ (โต๊ะเดิม/โต๊ะใหม่)

โค้ดอ้างอิงส่วนล้างรายละเอียดเดิม:

```js
await clearOrderDetails(id);
let nextTotal = 0;
for (const detail of validDetails) {
  const subTotal = (detail.menu.price || 0) * detail.qty;
  nextTotal += subTotal;
  await orderDetailModel.create({
    order_id: id,
    menu_id: detail.menu.menu_id,
    quantity: detail.qty,
    sub_total: subTotal,
  });
}
await orderModel.updateTotalPrice(id, nextTotal);
```

### 2.3.7 ฟังก์ชันจ่ายเงินและออกใบเสร็จ
ฟังก์ชัน `ordersController.markPaid`:

```js
const nowIso = new Date().toISOString();
const dateCode = nowIso.slice(0, 10).replace(/-/g, "");
const receiptNo = existing.receipt_no || `R-${dateCode}-${String(id).padStart(6, "0")}`;
const paid = await orderModel.markPaid(id, {
  paid_at: nowIso,
  receipt_no: receiptNo,
  service_charge: Number(req.body?.service_charge) || 0,
  tax: Number(req.body?.tax) || 0,
});
```

ฟังก์ชัน `getReceipt` ใช้สร้างข้อมูลสรุป:
1. ดึง Orders + OrderDetail
2. คำนวณ `subtotal + service_charge + tax = grand_total`
3. ส่งข้อมูลสำหรับแสดงใบเสร็จ

### 2.3.8 ฟังก์ชันคิวครัว + การอัปเดตแบบเรียลไทม์
คิวครัว (`listKitchenQueue`) จะดึงเฉพาะออเดอร์ที่ยังไม่ปิดงาน:

```js
WHERE lower(coalesce(order_status, 'pending')) IN (?, ?, ?, ?, ?, ?)
  AND lower(coalesce(payment_status, 'unpaid')) != 'paid'
ORDER BY datetime(order_date) ASC, order_id ASC
```

stream แบบ SSE (`ordersController.stream`):

```js
res.setHeader("Content-Type", "text/event-stream");
res.write(`event: connected\ndata: ${JSON.stringify({ ok: true, ts: Date.now() })}\n\n`);
```

สรุปการทำงาน:
1. หน้าแอดมินเปิด stream ค้างไว้
2. เมื่อมี event เช่น `order_created`, `order_updated`, `order_paid`, `table_status_changed`
3. client รับ event แล้วรีเฟรชข้อมูลเฉพาะส่วน

### 2.3.9 ฟังก์ชันซิงก์สถานะโต๊ะอัตโนมัติ
`syncTableLifecycle(tableId)` ประเมินจากจำนวนบิลค้าง:
1. ถ้าไม่มีบิลค้าง -> `open`
2. ถ้ามีบิลค้างและมีรายการพร้อมคิดเงิน -> `billing`
3. อื่นๆ -> `occupied`

โค้ดอ้างอิง:

```js
const snapshot = await orderModel.getWorkflowSnapshotByTable(safeTableId);
const unpaidOpen = Number(snapshot?.unpaid_open_count) || 0;
const billingCandidates = Number(snapshot?.billing_candidate_count) || 0;
const nextStatus = unpaidOpen <= 0 ? "open" : billingCandidates > 0 ? "billing" : "occupied";
const updatedTable = await diningTableModel.updateStatus(safeTableId, nextStatus);
```

### 2.3.10 ฟังก์ชันจัดการ QR โต๊ะ
ฟังก์ชัน `tableQRController.generate`:

```js
const token = `${Date.now()}-${Math.random().toString(36).slice(2, 10)}`;
await tableQRModel.removeByTableId(tableId);
const row = await tableQRModel.create({ token, table_id: tableId, created_at, expires_at });

const qrData = `${req.protocol}://${req.get('host')}/t/${encodeURIComponent(token)}`;
const qrImageUrl = `https://api.qrserver.com/v1/create-qr-code/?size=300x300&data=${encodeURIComponent(qrData)}`;
```

สรุปการทำงาน:
1. สร้าง token ใหม่
2. ลบ token เก่าของโต๊ะนั้น
3. บันทึก token ใหม่
4. ส่ง URL สำหรับสร้าง QR image และ URL ปลายทางให้ลูกค้า

### 2.3.11 ฟังก์ชันรายงานยอดขายและเมนูขายดี
ใน `reportController.js` มีรายงานหลัก 2 ชุด

1. `renderReport1` รายงานยอดขายตามวัน/เดือน
2. `renderReport2` รายงานเมนูขายดี + breakdown รายวัน

ตัวอย่าง query model สำหรับยอดขายตามเวลา:

```js
SELECT
  date(o.order_date) AS period,
  COUNT(o.order_id) AS total_orders,
  ROUND(IFNULL(SUM(o.total_price), 0), 2) AS total_sales
FROM Orders o
WHERE ...
GROUP BY date(o.order_date)
ORDER BY period DESC
```

ตัวอย่าง query model สำหรับเมนูขายดี:

```js
SELECT
  m.menu_id,
  COALESCE(m.menu_name, 'Unknown Menu') AS menu_name,
  IFNULL(SUM(od.quantity), 0) AS total_quantity,
  ROUND(IFNULL(SUM(od.sub_total), 0), 2) AS total_sales
FROM OrderDetail od
INNER JOIN Orders o ON o.order_id = od.order_id
LEFT JOIN Menu m ON m.menu_id = od.menu_id
WHERE ...
GROUP BY m.menu_id, m.menu_name, m.category_name
ORDER BY total_quantity DESC, total_sales DESC
LIMIT ?
```

---

### 2.4 ตัวอย่าง Endpoint สำคัญของระบบ

#### เมนู
1. `GET /menus`
2. `GET /menus/:id`
3. `POST /menus`
4. `PUT /menus/:id`
5. `DELETE /menus/:id`

#### สมาชิก
1. `POST /memberships`
2. `POST /login`
3. `GET /memberships`
4. `PUT /memberships/:id`

#### ออเดอร์
1. `GET /orders`
2. `GET /orders/:id`
3. `POST /orders`
4. `PUT /orders/:id`
5. `POST /orders/:id/pay`
6. `GET /orders/:id/receipt`
7. `GET /orders/kitchen/queue`
8. `GET /orders/stream`

#### QR โต๊ะ
1. `POST /api/tableqr/generate`
2. `GET /api/tableqr/table/:tableId`
3. `GET /api/tableqr/token/:token`

---

## บทที่ 3 ลำดับการทำงานของระบบ (Workflow)

### 3.1 Workflow ฝั่งลูกค้า
1. ลูกค้าสแกน QR -> เข้าลิงก์ `/t/:token`
2. ระบบ redirect ไปหน้า `/?tableKey=...`
3. หน้าเว็บเรียก API เพื่อโหลดเมนู
4. ลูกค้าเลือกเมนูและส่งคำสั่งซื้อ (`POST /orders`)
5. ระบบบันทึก Orders + OrderDetail และอัปเดตยอดรวม

### 3.2 Workflow ฝั่งพนักงาน/ผู้จัดการ
1. แอดมินล็อกอินเข้า dashboard
2. ระบบเปิด stream รับ event order แบบเรียลไทม์
3. เห็นคิวครัวจาก `GET /orders/kitchen/queue`
4. เปลี่ยนสถานะออเดอร์และปิดบิล
5. ชำระเงินผ่าน `POST /orders/:id/pay` และออกใบเสร็จ

### 3.3 Workflow รายงาน
1. ผู้จัดการเลือกรายงานและเงื่อนไขกรอง (วันเริ่ม/วันสิ้นสุด/สถานะ)
2. ระบบ query ข้อมูลจากตาราง Orders และ OrderDetail
3. ระบบคำนวณ summary เช่น ยอดรวม ค่าเฉลี่ย เทรนด์
4. แสดงผลในหน้า `report1` และ `report2`

---

## บทที่ 4 ข้อเสนอสำหรับใส่ในรายงานฉบับส่งจริง

1. เพิ่ม ER Diagram จากตาราง `Menu`, `Orders`, `OrderDetail`, `Membership`, `DiningTable`, `TableQR`
2. เพิ่ม sequence diagram สำหรับ "สแกน QR -> สั่งอาหาร -> ชำระเงิน"
3. เพิ่มตารางทดสอบ (Test Case) เช่น
   - สมัครสมาชิกด้วย email ซ้ำต้องได้ `409`
   - สั่งอาหารด้วย menu_id ไม่ถูกต้องต้องได้ `400`
   - ปิดบิลต้องสร้าง `receipt_no` อัตโนมัติ
4. เพิ่มภาพหน้าจอระบบจริงประกอบคำอธิบาย

---

## ภาคผนวก A: ไฟล์โค้ดสำคัญที่ใช้อ้างอิง
1. `app.js`
2. `server/createApiApp.js`
3. `controllers/api/menusController.js`
4. `controllers/api/membershipsController.js`
5. `controllers/api/authController.js`
6. `controllers/api/ordersController.js`
7. `controllers/api/orderDetailsController.js`
8. `controllers/api/tableQRController.js`
9. `controllers/reportController.js`
10. `models/reportModel.js`
11. `database/schema.js`

---

## ภาคผนวก B: โค้ดเสริมสำหรับอธิบายเรื่อง Security (รหัสผ่าน)

```js
const crypto = require("crypto");

function hashPassword(password, salt = crypto.randomBytes(16).toString("hex")) {
  const passwordHash = crypto
    .pbkdf2Sync(password, salt, 100000, 64, "sha512")
    .toString("hex");
  return { salt, passwordHash };
}

function verifyPassword(password, salt, expectedHash) {
  const { passwordHash } = hashPassword(password, salt);
  return passwordHash === expectedHash;
}
```

จุดสำคัญ:
1. ไม่เก็บรหัสผ่านแบบ plain text
2. ใช้ salt ต่อผู้ใช้แต่ละคน
3. ตรวจสอบความถูกต้องด้วยการ hash ซ้ำแล้วเทียบค่า

---

## บทที่ 5 คู่มือการใช้งานเว็บไซต์ฝั่งแอดมิน (Admin) ครบทุกหน้า

> เหมาะสำหรับผู้ใช้งานจริงของร้าน (ผู้จัดการ/แอดมิน) เพื่อใช้งานระบบได้ครบตั้งแต่ล็อกอินจนถึงรายงาน

### 5.1 หน้าเข้าสู่ระบบแอดมิน (`/admin-login`)
1. เปิดหน้า `http://localhost:5000/admin-login`
2. กรอก `Email Address` และ `Password`
3. กดปุ่ม `Sign in`
4. หากข้อมูลถูกต้อง ระบบจะพาไปหน้า `Dashboard` (`/dashboard`)
5. หากข้อมูลไม่ถูกต้อง จะแสดงข้อความ `Invalid credentials`

หมายเหตุการใช้งาน:
1. ค่าเริ่มต้นจากไฟล์ `.env.example` คือ `ADMIN_USER=admin@foodflow.com` และ `ADMIN_PASS=adminpass`
2. ระบบจำกัดการลองล็อกอินผิดซ้ำ (ค่าเริ่มต้น 10 ครั้งต่อ 15 นาที)

### 5.2 หน้า Dashboard (โครงสร้างหลัก)
หลังล็อกอินสำเร็จ จะอยู่หน้าเดียวที่มีหลายแท็บทางซ้าย:
1. `Menus` - จัดการเมนูอาหาร
2. `Customers` - จัดการข้อมูลลูกค้า/สมาชิก
3. `Orders` - จัดการออเดอร์และการชำระเงิน
4. `Table QR` - จัดการโต๊ะและ QR
5. `Reports` - ดูรายงานธุรกิจ
6. `Settings` - ตั้งค่าหน้าจอ
7. `Logout` - ออกจากระบบ

### 5.3 หน้า Menus (Menu Management)
#### งานที่ทำได้
1. เพิ่มเมนูใหม่
2. ดูรายละเอียดเมนู
3. แก้ไขเมนู
4. ลบเมนู
5. ค้นหาเมนูตามชื่อหรือหมวด
6. รีเฟรชข้อมูลตาราง

#### ขั้นตอนเพิ่มเมนู
1. กด `+ Add New`
2. กรอกข้อมูล:
   - `Menu Name` (จำเป็น)
   - `Category` (จำเป็น)
   - `Price` (จำเป็น)
   - `Status` (`available` / `unavailable`)
   - `Menu Image` (ไม่บังคับ)
3. กด `Save`

หมายเหตุรูปภาพ:
1. ระบบรับเฉพาะไฟล์ภาพ (`image/*`)
2. ขนาดไฟล์ไม่เกิน 5MB
3. ระบบบันทึกไฟล์อัตโนมัติที่ `public/assets/images`

#### การจัดการจากตารางเมนู
1. `View` - เปิดฟอร์มแบบอ่านอย่างเดียว
2. `Edit` - แก้ไขข้อมูลเมนูแล้วกด `Save`
3. `Delete` - ลบเมนู (มีหน้าต่างยืนยันก่อนลบ)

### 5.4 หน้า Customers (Customer Management)
#### ส่วนข้อมูลบนหน้า
1. สรุปจำนวน `Total Customers`, `Active`, `Inactive`
2. ตารางรายชื่อลูกค้า
3. ฟอร์มเพิ่ม/แก้ไขลูกค้า

#### เพิ่มลูกค้าใหม่
1. กด `+ Add New`
2. กรอกข้อมูลหลัก:
   - `First Name` (จำเป็น)
   - `Email` (จำเป็น)
   - `Password` (จำเป็นเฉพาะตอนเพิ่มใหม่)
3. กรอกข้อมูลเสริมตามต้องการ: `Last Name`, `Phone`, `Tier`, `Points`, `Status`
4. กด `Save`

#### แก้ไขลูกค้า
1. กด `Edit` หรือ `View` ที่แถวลูกค้า
2. ปรับข้อมูลที่ต้องการ
3. ช่องรหัสผ่านสามารถเว้นว่างได้ (ระบบจะใช้รหัสเดิม)
4. กด `Save`

#### ลบลูกค้า
1. กด `Delete`
2. ยืนยันการลบในกล่องยืนยัน

### 5.5 หน้า Orders (Order Management)
#### งานที่ทำได้
1. สร้างออเดอร์ใหม่
2. ดู/แก้ไขออเดอร์
3. ลบออเดอร์
4. ปิดชำระเงิน (`Mark Paid`)
5. ดูข้อมูลใบเสร็จ (`Receipt`)
6. ดูคิวครัว (`Kitchen Queue`)

#### สร้างออเดอร์ใหม่
1. กด `+ Create Order`
2. เลือกลูกค้า (`Walk-in customer` ได้)
3. เลือก `Table ID` (หรือไม่เลือกได้)
4. เลือกวันที่ `Order Date`
5. เลือก `Order Status` และ `Payment Status`
6. เพิ่มรายการเมนูในส่วน `Selected Items`
7. กด `Save`

ข้อสำคัญ:
1. ต้องมีอย่างน้อย 1 เมนูในออเดอร์ มิฉะนั้นระบบจะไม่ให้บันทึก
2. จำนวน (`Quantity`) ต้องมากกว่า 0

#### จัดการออเดอร์จากตาราง
1. `View` - ดูข้อมูลออเดอร์และรายการเมนู
2. `Edit` - แก้ไขข้อมูลออเดอร์และรายการเมนู
3. `Mark Paid` - เปลี่ยนสถานะชำระเงินเป็น paid และสร้างเลขใบเสร็จอัตโนมัติ
4. `Receipt` - แสดงสรุปใบเสร็จ (เลขใบเสร็จ, ยอดรวม, service charge, tax)
5. `Delete` - ลบออเดอร์

#### Kitchen Queue
1. แสดงเฉพาะออเดอร์ที่ยัง Active และยังไม่ชำระเงิน
2. หน้า Orders รีเฟรชข้อมูลแบบอัตโนมัติทุก 15 วินาที และรองรับอัปเดตแบบเรียลไทม์

### 5.6 หน้า Table QR (QR Management)
#### เพิ่มโต๊ะ
1. กรอก `New Table Name` (เว้นว่างได้)
2. กด `Add Table`
3. ถ้าเว้นว่าง ระบบจะตั้งชื่ออัตโนมัติ เช่น `Table 12`

#### สร้าง QR ให้โต๊ะ
1. เลือกโต๊ะจาก `Select Table`
2. กด `Generate QR`
3. ระบบจะแสดงรูป QR และลิงก์สำหรับลูกค้า

หมายเหตุ:
1. 1 โต๊ะจะมี QR token ที่ใช้งานล่าสุดได้ 1 ตัว (สร้างใหม่จะทับตัวเก่า)

#### ลบโต๊ะ
1. เลือกโต๊ะ
2. กด `Drop Selected Table`
3. ยืนยันการลบ

ข้อควรระวัง:
1. ถ้าโต๊ะมีออเดอร์ค้างชำระ ระบบจะไม่อนุญาตให้ลบโต๊ะ

### 5.7 หน้า Reports (Business Reports)
หน้า Reports ใน Dashboard จะแสดงรายงาน 2 ตัวผ่าน iframe

#### Report 1: Sales by Time (`/reports/report1`)
ใช้วิเคราะห์ยอดขายตามช่วงเวลา
1. เลือก `Group By` (Day/Month)
2. กำหนด `Start Date` และ `End Date`
3. เลือก `Order Status`
4. กด `Generate`
5. ดูสรุป: Total Orders, Total Sales, Average Order Value และตารางรายช่วงเวลา

#### Report 2: Best Selling Menus (`/reports/report2`)
ใช้วิเคราะห์เมนูขายดี
1. กำหนดช่วงวันที่
2. เลือกสถานะออเดอร์
3. กำหนด `Limit` (จำนวนเมนูอันดับบนสุดที่จะแสดง)
4. กด `Generate`
5. ดูสรุป: เมนูขายดีที่สุด, ยอดขายรวม, สัดส่วน Top Menu และตารางรายละเอียด

### 5.8 หน้า Settings
1. มีสวิตช์สำหรับหมวด `Privacy & Security` และ `Appearance`
2. `Dark Mode` ใช้งานได้จริงและจดจำค่าธีมในเบราว์เซอร์
3. สวิตช์อื่นในหน้านี้เป็นการตั้งค่าระดับ UI (แสดงผลหน้าเว็บ) ยังไม่บันทึกลงฐานข้อมูล

### 5.9 ออกจากระบบ (Logout)
1. กดเมนู `Logout` ทางซ้าย
2. ระบบจะแสดงกล่องยืนยัน
3. กด `Yes` เพื่อออกจากระบบ
4. ระบบจะกลับไปหน้า `/admin-login`

### 5.10 ปัญหาที่พบบ่อยและวิธีแก้เบื้องต้น
1. ล็อกอินไม่ได้:
   - ตรวจสอบ Email/Password ในไฟล์ `.env`
   - หากลองผิดหลายครั้ง ให้รอครบช่วงเวลาจำกัดการล็อกอิน
2. บันทึกเมนูไม่ได้:
   - ตรวจสอบว่า `Menu Name`, `Category`, `Price` กรอกครบ
   - ตรวจสอบชนิดไฟล์รูปและขนาดไฟล์
3. สร้างออเดอร์ไม่ได้:
   - ตรวจสอบว่าเพิ่มรายการเมนูอย่างน้อย 1 รายการ
4. ลบโต๊ะไม่ได้:
   - ตรวจสอบว่าโต๊ะนั้นไม่มีออเดอร์ค้างชำระ

---

## บทที่ 6 คู่มือการใช้งานเว็บไซต์ฝั่งลูกค้า (Customer) ครบทุกหน้า

> สำหรับลูกค้าที่สแกน QR ที่โต๊ะ และใช้งานระบบตั้งแต่เข้าเมนูจนยืนยันคำสั่งซื้อ

### 6.1 การเริ่มใช้งานผ่าน QR และการเข้าหน้าเว็บ
1. ลูกค้าสแกน QR ที่โต๊ะ
2. ระบบอาจพาเข้าได้ 2 รูปแบบ:
   - `/member-login/:tableKey` (รูปแบบหลักที่ใช้ในหน้า Table QR ของแอดมิน)
   - `/t/:token` แล้ว redirect ไป `/?tableKey=...` (เส้นทางสำรอง)
3. หาก token/tableKey ถูกต้อง ระบบจะเข้าสู่หน้าลูกค้าของโต๊ะนั้น
4. ถ้า token/tableKey ไม่ถูกต้อง ระบบจะแสดงหน้า `Table Not Found` และให้สแกนใหม่

### 6.2 หน้าเลือกวิธีเข้าใช้งาน (กรณีเข้าจากหน้าแรก)
1. เปิดหน้าแรกของระบบ แล้วกรอก `Table Key`
2. เลือก `Customer Member Login` เพื่อไปหน้าเข้าสู่ระบบสมาชิก
3. หรือเลือก `Continue as Guest` เพื่อเข้าหน้าลูกค้าแบบไม่ล็อกอินสมาชิก

หมายเหตุ:
1. ถ้าไม่กรอก Table Key ระบบจะไม่ให้ไปหน้าถัดไป
2. ถ้าเข้ามาจาก QR ที่ลิงก์ไป `member-login` โดยตรง จะไม่ผ่านขั้นตอนเลือกในหน้าแรก
### 6.3 หน้า Member Login (`/member-login/:tableKey`)
1. กรอก `Email Address` และ `Password`
2. กด `Sign in`
3. หากสำเร็จ ระบบจะบันทึกข้อมูลสมาชิกไว้ในเบราว์เซอร์ (localStorage) และพาไปหน้า Dashboard ของโต๊ะนั้น
4. หากไม่สำเร็จ จะแสดงข้อความผิดพลาด เช่น `Invalid credentials`
5. ถ้าไม่ต้องการล็อกอิน สามารถกด `Continue as Guest` ได้

### 6.4 หน้า Member Register (`/member-register/:tableKey`)
1. กรอกข้อมูลสมัครสมาชิก:
   - First Name (จำเป็น)
   - Last Name
   - Email (จำเป็น)
   - Password (จำเป็น)
   - Confirm Password (จำเป็น)
2. ติ๊กยอมรับเงื่อนไข (`I agree to the terms & privacy`)
3. กด `Create Account`
4. ระบบจะสร้างบัญชี แล้วพากลับไปหน้า Login อัตโนมัติ

เงื่อนไขตรวจสอบก่อนสมัคร:
1. Email ต้องอยู่ในรูปแบบที่ถูกต้อง
2. Password ต้องยาวอย่างน้อย 8 ตัวอักษร
3. Password และ Confirm Password ต้องตรงกัน
4. ต้องติ๊กยอมรับเงื่อนไขก่อนส่งฟอร์ม

### 6.5 หน้า Customer Dashboard (โครงสร้างหลัก)
หลังเข้าระบบ (Member หรือ Guest) จะอยู่หน้าเดียวที่มีแท็บซ้าย:
1. `Menu` - เลือกเมนูอาหาร
2. `Orders` - ตรวจสอบตะกร้าและยืนยันออเดอร์
3. `Settings` - ดู/แก้ไขข้อมูลสมาชิก และตั้งค่าธีม
4. `Logout` - ออกจากระบบลูกค้า

ข้อมูลบนแถบซ้าย:
1. แสดงเลขโต๊ะปัจจุบัน เช่น `Table 1`
2. แสดงสถานะผู้ใช้ (`Guest` หรือชื่อสมาชิก)

### 6.6 หน้า Menu (เลือกเมนูอาหาร)
#### ความสามารถหลัก
1. ดูรายการเมนูทั้งหมด
2. ค้นหาเมนูด้วยชื่อ (Search)
3. กรองตามหมวดหมู่ (Category)
4. เพิ่มเมนูลงตะกร้า
5. ปรับจำนวนในหน้าเมนูได้ทันที (+/-)

#### วิธีสั่งเมนูจากหน้า Menu
1. เลือกเมนูที่ต้องการ
2. กด `Add to order`
3. หากเลือกแล้ว ปุ่มจะเปลี่ยนเป็นตัวปรับจำนวน (+/-)
4. เมนูที่สถานะ `unavailable` จะแสดงเป็น `Out of stock` และกดไม่ได้

### 6.7 หน้า Orders (ตะกร้า + ยืนยันคำสั่งซื้อ)
#### สิ่งที่ทำได้ในหน้า Orders
1. ค้นหารายการในตะกร้า (`Billing Product Search`)
2. เพิ่ม/ลดจำนวนเมนูในตะกร้า
3. ลบรายการเมนูออกจากตะกร้า
4. ใส่รหัสสมาชิกเพื่อรับส่วนลด
5. ดูยอด `Sub Total`, `Membership Discount`, `Total Amount`
6. กด `Complete Order` เพื่อส่งออเดอร์เข้าระบบร้าน

#### การใช้ส่วนลดสมาชิก
1. กรอกรหัสสมาชิกที่ช่อง `Have a Membership Card? Enter Here`
2. ถ้ารหัสถูกต้อง ระบบแสดงสถานะ `Available`
3. ถ้ารหัสไม่ถูกต้อง ระบบแสดงสถานะ `Invalid`
4. ระบบคำนวณส่วนลดให้อัตโนมัติ (ตามค่าที่ระบบกำหนด)

#### ยืนยันคำสั่งซื้อ
1. ตรวจสอบรายการในตะกร้า
2. กด `Complete Order`
3. ระบบส่งคำสั่งซื้อไปที่ API และสร้าง `Order ID`
4. เมื่อสำเร็จ จะแสดงข้อความยืนยันพร้อมเลขออเดอร์

### 6.8 หน้า Settings
#### My Profile
1. ถ้าเป็น Guest: จะแสดงสถานะ Guest
2. ถ้าเป็น Member: จะแสดงข้อมูลสมาชิก เช่น First Name, Last Name, Email, Phone, Points, Tier
3. กด `Edit Profile` เพื่อแก้ไขข้อมูล
4. กรอกข้อมูลใหม่แล้วกด `Save`

เงื่อนไขการแก้ไขโปรไฟล์:
1. First Name ต้องไม่ว่าง
2. ถ้ากรอกรหัสผ่านใหม่ ต้องยาวอย่างน้อย 8 ตัวอักษร
3. หากไม่กรอกรหัสผ่านใหม่ ระบบจะคงรหัสเดิม

#### Appearance
1. ใช้ `Dark Mode` เพื่อสลับธีมสว่าง/มืด
2. ระบบจดจำธีมไว้ในเบราว์เซอร์

### 6.9 ออกจากระบบลูกค้า (Logout)
1. กดเมนู `Logout` ทางซ้าย
2. ระบบแสดงกล่องยืนยัน
3. กด `Yes` เพื่อออกจากระบบ
4. ระบบจะล้างข้อมูลสมาชิกในเบราว์เซอร์ และพาไปหน้า `member-login` ของโต๊ะนั้น

### 6.10 ปัญหาที่พบบ่อยและวิธีแก้เบื้องต้น (ฝั่งลูกค้า)
1. เข้าเว็บไม่ได้หลังสแกน QR:
   - ให้สแกน QR ใหม่
   - ตรวจสอบว่าโต๊ะนั้นยังมี QR ที่ใช้งานได้
2. Login สมาชิกไม่ผ่าน:
   - ตรวจสอบ Email/Password ให้ถูกต้อง
   - ถ้ายังไม่มีบัญชีให้สมัครผ่านหน้า Register
3. กด Complete Order ไม่ได้:
   - ตรวจสอบว่ามีรายการเมนูในตะกร้าอย่างน้อย 1 รายการ
4. โปรไฟล์แก้ไขไม่สำเร็จ:
   - ตรวจสอบว่า First Name ไม่ว่าง
   - ถ้าเปลี่ยนรหัสผ่านใหม่ต้องมีอย่างน้อย 8 ตัวอักษร

