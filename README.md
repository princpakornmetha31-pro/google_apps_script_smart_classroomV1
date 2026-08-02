# Smart Classroom V8 Final — คู่มือใช้งาน

## ไฟล์ที่ต้องมีใน Apps Script

- `Code.gs` — ระบบหลังบ้าน
- `Index.html` — หน้าครู เช็กชื่อ Dashboard กราฟ และ PDF
- `Student.html` — นักเรียนกรอก ID + PIN เพื่อดูคะแนนและเปอร์เซ็นต์เข้าเรียน
- `QR.html` — สร้าง QR Code สำหรับ Student Portal

`Style.html` เดิมไม่จำเป็นต่อไฟล์ชุดนี้ หาก `Index.html` ของคุณเก็บ CSS ไว้ภายในไฟล์ครบแล้ว

## โครงสร้างชีต Students

| คอลัมน์ | ข้อมูล |
|---|---|
| A | ID |
| B | No |
| C | Firstname |
| D | Nickname |
| E | Present |
| F | Absent |
| G | AttendancePercent |
| H | Examscore |
| I | Result |
| J | PIN |

## โครงสร้างชีต Attendance

- A = ID
- B = Nickname
- C เป็นต้นไป = วันที่ เช่น `01-Jul`
- ค่าเช็กชื่อเป็น `TRUE` หรือ `FALSE`

## ติดตั้งไฟล์

1. สำรองไฟล์เดิมก่อน
2. เปิด `Code.gs` ใน Apps Script แล้วแทนที่ด้วยไฟล์ชุดนี้
3. เปิด `Index.html` แล้วแทนที่ด้วยไฟล์ชุดนี้
4. สร้างหรือแทนที่ `Student.html`
5. สร้างหรือแทนที่ `QR.html`
6. กด `Ctrl + S` ทุกไฟล์

## Deploy

1. กด **Deploy**
2. เลือก **Manage deployments**
3. กดรูปดินสอ
4. เลือก **New version**
5. Execute as: **Me**
6. Who has access: **Anyone**
7. กด **Deploy**
8. ใช้ URL ที่ลงท้าย `/exec`

## URL ที่ใช้

- หน้าครู: `WEB_APP_URL/exec`
- Student Portal: `WEB_APP_URL/exec?page=Student`
- หน้า QR: `WEB_APP_URL/exec?page=QR`

## Auto Refresh Dashboard

หน้า `Index.html` จะอัปเดตตัวเลข Dashboard จาก Google Sheets ทุก 30 วินาที โดยไม่รีโหลดทั้งหน้า

ระบบจะไม่แก้ checkbox ที่ครูกำลังติ๊ก แต่จะอัปเดตเฉพาะ:

- นักเรียนทั้งหมด
- มาเรียน
- ขาดเรียน
- กราฟรายวัน
- เวลาอัปเดตล่าสุด

เมื่อครูกลับมายังแท็บเว็บ ระบบจะอัปเดตทันทีอีกครั้ง

## การใช้งานประจำวัน

1. เปิดหน้าครู
2. เลือกวันที่
3. ติ๊กมาเรียน
4. กดบันทึก
5. Dashboard และเวลาอัปเดตจะเปลี่ยนทันที
6. หลังจากนั้น Dashboard ตรวจข้อมูลใหม่ทุก 30 วินาที

## เพิ่มนักเรียน

เพิ่มข้อมูลในชีต `Students` แถวใหม่ โดยต้องกรอกอย่างน้อย:

- ID
- No
- Firstname
- Nickname
- PIN

หลังเพิ่มนักเรียน หากต้องการให้รายชื่อเข้า Attendance ให้สำรองข้อมูลเดิมก่อน แล้วสร้างปฏิทินใหม่ หรือเพิ่ม ID/Nickname ลง Attendance ด้วยตนเอง

## เปลี่ยน PIN

แก้ค่าในชีต `Students` คอลัมน์ J แล้วกด Enter นักเรียนใช้ PIN ใหม่ได้ทันทีหลังเปิดหน้าใหม่

## เริ่มภาคเรียนใหม่

1. สำรอง Spreadsheet เดิม
2. ตั้ง `StartDate` และ `EndDate` ในชีต Settings
3. ตรวจวันหยุดในชีต Holiday
4. กดเมนู `🛠️ ระบบเช็คชื่อ`
5. กด `สร้างปฏิทินเช็คชื่อ`

คำเตือน: การสร้างปฏิทินจะล้างชีต Attendance เดิม

## หมายเหตุด้านความปลอดภัย

- อย่าแจก URL หน้าครูให้นักเรียน
- แจกเฉพาะ QR หรือ URL `?page=Student`
- PIN เป็นข้อมูลลับรายบุคคล
- Student Portal ส่งกลับเฉพาะชื่อ คะแนน และเปอร์เซ็นต์เข้าเรียนของ ID ที่ Login สำเร็จ
