# ระบบคลังวิทยานิพนธ์อิเล็กทรอนิกส์ | คณะวิศวกรรมศาสตร์ มมส

ระบบเว็บแอปที่สร้างด้วย Google Apps Script สำหรับให้นิสิตส่งไฟล์วิทยานิพนธ์ (PDF) เข้าระบบ, แอดมินตรวจ/อนุมัติ, และแสดงผลเป็น Flipbook 3 มิติให้ค้นหา/อ่านออนไลน์ได้

## 🐞 บั๊กที่แก้ไขจากต้นฉบับ

1. **ปุ่ม "ไม่อนุมัติ" กดแล้ว error ทันที** — หน้าบ้านเรียก `updateStatusToRejected()` แต่ฟังก์ชันนี้ไม่เคยถูกสร้างไว้ใน `Code.gs` เลย → เพิ่มฟังก์ชันนี้ให้แล้ว
2. **สคริปต์ทั้งหน้าเว็บพังเงียบๆ (SyntaxError)** — `JavaScript.html` และ `Form.html` ต่างประกาศ `const majorsByDegree` และ `function updateMajors()` ซ้ำกัน เมื่อ Apps Script รวมทุกไฟล์เข้าด้วยกันในหน้าเดียว จะเกิด `Identifier has already been declared` ทำให้ **ปุ่มทั้งหมดในเว็บกดไม่ติด** (เบราว์เซอร์บางตัวอาจไม่ error แต่พฤติกรรมจะเพี้ยน) → ลบส่วนซ้ำออก เหลือประกาศจุดเดียวใน `Form.html`
3. **ตารางรอตรวจสอบไม่โชว์ "สาขาวิชา"** — `getPendingThesesData()` ไม่เคยส่งค่า `major` กลับไป ทั้งที่หน้าบ้านพยายามอ่าน `row.major` → เพิ่มให้แล้ว
4. **การ์ดสถิติ "ไม่อนุมัติ" ค้างที่ 0 ตลอด** — `getAdminStats()` ไม่เคยนับสถานะ "ไม่อนุมัติ" เลย → เพิ่มการนับแล้ว
5. **จุดตั้งค่าซ้ำซ้อน/ไม่สอดคล้องกัน** — เดิม `Code.gs` ใช้ `SpreadsheetApp.getActiveSpreadsheet()` กับ `"Sheet1"` ตรงๆ ในหลายฟังก์ชัน โดยไม่ได้ใช้ `CONFIG` ที่ตั้งไว้ใน `Config.gs` เลย ทำให้แก้ค่าที่ `Config.gs` แล้วไม่มีผลจริง และถ้า deploy เป็นสคริปต์แบบ standalone (แยกจากสเปรดชีต) จะพังทันที → รวมให้ทุกฟังก์ชันอ้างอิงผ่าน `getDatabaseSheet()` และ `CONFIG` จุดเดียว พร้อมย้ายรหัสผ่านแอดมินไปไว้ใน `CONFIG.ADMIN_PASSWORD`

## 📁 โครงสร้างไฟล์

| ไฟล์ | หน้าที่ |
|---|---|
| `Code.gs` | Backend หลัก: routing, บันทึก/อนุมัติ/ปฏิเสธข้อมูล, นับสถิติ |
| `Config.gs` | ค่าคงที่ทั้งหมด (Spreadsheet ID, Folder ID, รหัสผ่านแอดมิน) |
| `Index.html` | หน้าเว็บหลัก (ค้นหา + สลับแท็บ) |
| `Style.html`, `CSS.html` | สไตล์ตกแต่ง / Dark mode |
| `Script.html` | โหลด/กรอง/แสดงผลการ์ดวิทยานิพนธ์ |
| `JavaScript.html` | ระบบแอดมิน (login, อนุมัติ/ไม่อนุมัติ, toast, modal) |
| `Form.html` | ฟอร์มส่งผลงานของนิสิต |
| `Admin.html` | หน้าตาแดชบอร์ดแอดมิน |
| `Flipbook.html` | หน้าอ่านวิทยานิพนธ์แบบพลิกหนังสือ/เลื่อนอ่าน |
| `appsscript.json` | Manifest ของ Apps Script (จำเป็นสำหรับ clasp) |

## ⚙️ ตั้งค่าก่อนใช้งานจริง

แก้ค่าทั้งหมดใน **`Config.gs`**:

```js
const CONFIG = {
  SPREADSHEET_ID: "ใส่ ID ของ Google Sheet ที่จะใช้เก็บข้อมูล",
  SHEET_NAME: "Theses",
  PDF_FOLDER_ID: "ใส่ ID โฟลเดอร์ Google Drive สำหรับเก็บไฟล์ PDF",
  COVER_FOLDER_ID: "ใส่ ID โฟลเดอร์ (สำรอง ยังไม่ได้ใช้งานจริง)",
  ADMIN_PASSWORD: "เปลี่ยนเป็นรหัสผ่านที่คาดเดายาก",
  TELEGRAM_BOT_TOKEN: "YOUR_TELEGRAM_BOT_TOKEN_HERE"
};
```

**Google Sheet** ต้องมีคอลัมน์เรียงตามลำดับนี้ (แถวแรกเป็นหัวตาราง):

| A | B | C | D | E | F | G | H | I | J | K | L | M |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| ID | วันที่ส่ง | ผู้จัดทำ | ระดับ | ชื่อ TH | ชื่อ EN | สาขา | ปี | ที่ปรึกษา | ลิงก์ PDF | สถานะ | เข้าชม | ดาวน์โหลด |

สถานะที่ระบบใช้: `รอตรวจสอบ`, `อนุมัติแล้ว`, `ไม่อนุมัติ`

## 🚀 วิธีอัปขึ้น GitHub

```bash
git init
git add .
git commit -m "Initial commit: ระบบคลังวิทยานิพนธ์อิเล็กทรอนิกส์"
git branch -M main
git remote add origin https://github.com/<username>/<repo-name>.git
git push -u origin main
```

> ไฟล์ `.clasp.json` และ `.clasprc.json` ถูกกันไว้ใน `.gitignore` แล้ว เพราะมี Script ID และโทเค็นบัญชีส่วนตัวของคุณอยู่ **ห้ามอัปขึ้น GitHub เด็ดขาด**

## 🔗 วิธี Deploy กลับเข้า Google Apps Script (ใช้ `clasp`)

1. ติดตั้ง clasp: `npm install -g @google/clasp`
2. ล็อกอิน: `clasp login`
3. สร้างโปรเจกต์ใหม่ (หรือเชื่อมต่อโปรเจกต์เดิม):
   - สร้างใหม่: `clasp create --type webapp --title "ระบบคลังวิทยานิพนธ์"`
   - เชื่อมต่อของเดิม: สร้างไฟล์ `.clasp.json` เอง แล้วใส่ `scriptId` ของโปรเจกต์เดิม
4. อัปโค้ดขึ้น Apps Script: `clasp push`
5. Deploy เป็นเว็บแอป: `clasp deploy` (หรือเปิด Apps Script Editor แล้วกด Deploy > New deployment ด้วยตนเอง)

หากไม่ถนัดใช้ `clasp` สามารถคัดลอกเนื้อหาแต่ละไฟล์ไปวางใน [script.google.com](https://script.google.com) ตรงๆ ได้เช่นกัน (สร้างไฟล์ชื่อเดียวกันทีละไฟล์)
