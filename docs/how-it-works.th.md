# MacExplorer ทำงานอย่างไร (ภาษาไทย)

MacExplorer คือ File Manager สำหรับ macOS ที่ให้วิธีใช้งานแบบ Windows Explorer
(address bar, folder tree, ปุ่มลัด `Ctrl+C/X/V`, `F2`, Delete ลง Trash) แต่ทำงาน
แบบแอป macOS แท้ ๆ (Quick Look, Spotlight, ไอคอนระบบ, Light/Dark Mode)

## โครงสร้าง 5 ชั้น

```
UI (SwiftUI + AppKit)
  → ViewModel / State (สถานะของแต่ละ tab)
    → Domain (logic บริสุทธิ์ ไม่มี UI)
      → Services (คุยกับระบบไฟล์จริง)
        → macOS APIs (FileManager, Spotlight, Quick Look)
```

## 1. ชั้น Domain — สมองที่ไม่แตะหน้าจอ

- **NavigationState** เก็บ 3 อย่าง: โฟลเดอร์ปัจจุบัน + ประวัติถอยหลัง + ประวัติไปหน้า
  กด Back/Forward คือการสลับ stack ธรรมดา ส่วนการไปโฟลเดอร์ใหม่จะล้างประวัติ
  ไปหน้าทิ้ง (เหมือนเบราว์เซอร์)
- **PathResolver** แปลงข้อความใน address bar (`~`, `/path`, `file://`) ให้เป็น
  ตำแหน่งกลางก่อนนำทาง — address bar กับ breadcrumb ใช้ฟังก์ชันเดียวกัน
  ไม่มี logic ซ้ำซ้อน
- **SortConfiguration** จัดเรียงไฟล์ (โฟลเดอร์ก่อนไฟล์, 5 คอลัมน์) นอก main thread
  หน้าจอจึงไม่ค้างแม้มีไฟล์ 10,000 ไฟล์
- **FileSystemError** แปลง error ทางเทคนิคเป็นข้อความที่คนอ่านเข้าใจ

## 2. ชั้น Services — คนเดียวที่แตะไฟล์จริง

- หน้าจอ**ห้าม**เรียก `FileManager` โดยตรง ต้องผ่าน `LocalFileSystemService`
  เสมอ กฎเหล็กคือ **ระบบไฟล์คือความจริงหนึ่งเดียว**: หน้าจออัปเดต
  *หลัง*ระบบไฟล์ยืนยันผลแล้วเท่านั้น ไม่มีการ "โชว์ว่าสำเร็จก่อนแล้วค่อยหวังว่าได้"
- **ความปลอดภัยข้อมูล**
  - Copy ผ่านไฟล์ชั่วคราว (staging) แล้วเปลี่ยนชื่อทีเดียว — ถ้าล้มกลางคัน
    ไฟล์ปลายทางไม่พังครึ่ง ๆ กลาง ๆ
  - Replace (เขียนทับ) ต้องสำรองของเดิมก่อน แล้วเอาของเก่าเข้า Trash —
    กู้คืนได้เสมอ ไม่มีการเขียนทับเงียบ
  - ลบ = ย้ายลง Trash ไม่ลบถาวร (ตั้งให้ถามก่อนได้)
  - ย้ายโฟลเดอร์เข้าตัวมันเอง / ย้าย drive root / ย้ายทับตัวเอง
    โดนบล็อก*ก่อน*แตะไฟล์แม้แต่บิตเดียว
  - รู้จัก symlink: ไม่วนลูป ไม่หลงโฟลเดอร์
- **DirectoryMonitor** ใช้ระบบแจ้งเตือนของ macOS (ไม่วนเช็กเอง) + หน่วง 250ms —
  ไฟล์ที่โหลดจากเบราว์เซอร์/Terminal/Finder จะโผล่ในแอปเองอัตโนมัติ
- **SpotlightSearch** ใช้ดัชนีของ macOS ค้นข้ามโฟลเดอร์ย่อย ยกเลิกได้

## 3. ชั้น Presentation — state ใคร state มัน

- **BrowserViewModel (1 ตัวต่อ 1 tab)**: เก็บ navigation, selection, sort, search
  และมี **generation token** — เปิดโฟลเดอร์ใหม่แล้วงานโหลดเก่าจะถูกยกเลิก/ทิ้งผล
  ป้องกัน "หน้าเก่าโผล่ทับหน้าใหม่" ตอนกดย้อนรัว ๆ
- **TabStore**: tab แยก state กัน + จำ session ไว้เปิดใหม่
- **ClipboardService**: sync สถานะ cut กับ clipboard กลางของระบบ —
  copy จาก Finder มา paste ในแอปได้ (และกลับกัน)
- **FileOperationService**: คุม dialog ตอนชื่อไฟล์ชน
  (Replace / Keep Both / Skip / Cancel + ใช้กับทั้งหมด)
- **WorkspaceService**: เปิดไฟล์ด้วยแอป默认 + Quick Look (กด Space)

## 4. ชั้น UI — ใช้ของถูกกับงาน

- SwiftUI ทำโครงหน้าต่าง/toolbar/inspector/settings
- AppKit (`NSTableView`/`NSOutlineView`) ทำตารางไฟล์กับ tree เพราะต้องการ
  inline rename, ลากวาง, คอลัมน์จำค่าได้ — ซึ่งเป็นของที่ตาราง SwiftUI ยังให้ไม่ได้

## 5. คีย์บอร์ด

- Windows mode: `Ctrl+C/X/V/A/L`, `F2`, `Delete`, `Alt+ลูกศร`
- macOS mode: `Cmd+C/X/V/A/L` ทำงานเดียวกัน
- ระบบจะไม่ดักปุ่มขณะกำลังพิมพ์ในช่องข้อความ

## ตัวอย่างโฟลว์: กด Paste

```
กดปุ่ม → ตรวจ collision/validate → ทำบน actor (พื้นหลัง)
      → ระบบไฟล์ยืนยัน → รีเฟรช UI
```

ถ้าชื่อซ้ำจะถามก่อนเสมอ ไม่มีเขียนทับเงียบ ถ้าล้มจะบอกว่าของเดิมอยู่ที่ไหน

## ใหม่ใน v0.2.0

- **Progress + Cancel:** ก๊อปไฟล์ใหญ่แบบ stream ทีละ 1MB พร้อมแถบ progress
  (`Copying A.zip — 12 MB of 48 MB`) และปุ่ม Cancel — ยกเลิกแล้วไฟล์ค้าง
  ถูกลบ ต้นฉบับอยู่ครบ ไม่มี error หลอก
- **Breadcrumb dropdown:** เครื่องหมาย `>` ทุกตัวกดเปิดเมนูโฟลเดอร์ข้างเคียงได้
  มีติ๊กถูกที่ตำแหน่งปัจจุบัน
- **Favorites + Recent:** ปักหมุดโฟลเดอร์ที่ sidebar (ลากมาหย่อนได้) และเมนู
  Go → Recent Locations เก็บ 30 ที่ล่าสุด
