# CLAUDE.md — ผัวพารวย / ຜົວພາລວຍ (phuaparuay)

คู่มือ convention สำหรับ Claude Code เมื่อทำงานกับโปรเจกต์นี้ อ่านไฟล์นี้ก่อนแก้โค้ดเสมอ

---

## 1. โปรเจกต์คืออะไร

แอปบันทึกรายรับ-รายจ่าย + จัดการค่าคอมมิชชั่นตัวแทน (income-expense tracker with agent commissions) สำหรับธุรกิจในลาว/ไทย
- **ภาษา UI:** ลาว (`lo`) + ไทย (`th`) — สลับได้ในแอป (ค่าเริ่มต้นตาม user preference)
- **สกุลเงิน:** ₭ (กีบลาว, Lao Kip)
- **โครงสร้าง:** ไฟล์เดียว `index.html` (~805KB, ~11,400 บรรทัด) — HTML + CSS + JS รวมในไฟล์เดียว **ไม่มี build step**
- **Deploy:** GitHub Pages (public repo `mhdgroup01/phuaparuay`) → live ที่ https://mhdgroup01.github.io/phuaparuay/
- **Backend:** Supabase (region Singapore `ap-southeast-1`)

### กลุ่มผู้ใช้ 2 ระบบ
1. **ตัวแทนหลัก (main agents)** — มี workflow อนุมัติ: user สร้างตัวแทน → admin อนุมัติ → user แก้/เพิ่มบิล → "ส่งอัพเดต" → admin เห็น
2. **ตัวแทนย่อย (sub-agents)** — ระบบคู่ขนาน ใช้ user-id อย่างเดียว **ไม่มี** workflow admin (มิเรอร์ของ main แต่ตัดส่วนอนุมัติออก)

โค้ดของ 2 ระบบนี้เกือบเหมือนกันเป๊ะ — main ใช้ prefix `data-row-*`, sub ใช้ `data-sub-row-*` **แก้ฟีเจอร์ต้องแก้ทั้งคู่เสมอ**

---

## 2. กฎเหล็ก (ทำทุกครั้งก่อน commit)

### 2.1 Bump version
`APP_VERSION` มี **จุดเดียว** (ต่างจาก Paruay ที่มี 4 จุด):
```bash
grep -n "APP_VERSION =" index.html    # ~บรรทัด 1896
```
เปลี่ยนเลข เช่น `1.99.56` → `1.99.57` ทุกครั้งที่ปล่อยของ (แสดงบน header ให้ผู้ใช้เห็นว่าอัพเดทแล้ว)

### 2.2 ตรวจ JS syntax ก่อน commit (บังคับ)
```bash
sed -n '/<script>/,/<\/script>/p' index.html | tail -n +2 | head -n -1 > /tmp/app.js && node --check /tmp/app.js && echo "JS OK"
```
**ห้าม commit ถ้า syntax ไม่ผ่าน** — ไฟล์เดียวพังทั้งแอป

### 2.3 Deploy
```bash
git add index.html
git commit -m "v1.99.57: <สรุปสั้นๆ>"
git push
```
GitHub Pages อัพเดทอัตโนมัติใน ~1 นาที **ไม่ต้องใช้ present_files** (นั่นคือ workflow เก่าตอนอยู่บน claude.ai)

---

## 3. เทคนิคการแก้ไฟล์ใหญ่ (single-file 11k บรรทัด)

### 3.1 หาตำแหน่งด้วย grep เสมอ ก่อน edit
บรรทัดขยับตลอดเมื่อแก้ — **อย่าเชื่อเลขบรรทัดเก่า** grep หา anchor สดทุกครั้ง:
```bash
grep -n "function renderAgents" index.html
grep -n "data-sub-save-bill" index.html
```

### 3.2 str_replace ต้อง unique
ถ้า string ซ้ำ (main + sub มักซ้ำ) ให้ใส่ context รอบๆ ให้ unique หรือใช้ Python heredoc + assert:
```bash
python3 << 'EOF'
with open('index.html','r',encoding='utf-8') as f: c=f.read()
old = "..."
assert c.count(old) == 1, f"found {c.count(old)}"   # กันแก้ผิดจุด
c = c.replace(old, "...", 1)
with open('index.html','w',encoding='utf-8') as f: f.write(c)
EOF
```
Pattern `rep()` assertion นี้ใช้ตลอดในโปรเจกต์นี้ (เหมือน Paruay) — กันแก้พลาดเมื่อ string ปรากฏหลายที่

### 3.3 แก้ทั้ง main + sub พร้อมกัน
เกือบทุกฟีเจอร์มี 2 ก๊อปปี้ ตรวจว่าแก้ครบ:
```bash
grep -c "data-row-calc" index.html      # main
grep -c "data-sub-row-calc" index.html  # sub
```

---

## 4. โครงสร้างโค้ด (แผนที่)

### ค่าคงที่ + config (ต้นไฟล์ ~1890-1900)
- `APP_VERSION` ~1896
- `SUPABASE_URL` ~1899 = `https://yukfctbxjkgyymkjzknl.supabase.co`
- `SUPABASE_ANON_KEY` ~1900
- `ADMIN_FN_URL` ~10641 = `${SUPABASE_URL}/functions/v1/admin-users`

### i18n (แปลภาษา)
- Object `T = { lo: {...}, th: {...} }` — keys เยอะมาก (~300)
- ใช้ผ่าน `tt('key')` — มี fallback `tt('key') || 'ค่าสำรอง'`
- **เพิ่ม key ต้องใส่ทั้ง lo และ th** ไม่งั้นภาษาลาวจะโชว์ไทย
- ตรวจ key ที่ขาด:
```bash
python3 -c "
import re; c=open('index.html').read()
used=set(re.findall(r\"tt\('([a-zA-Z_0-9]+)'\)\",c))
miss=[k for k in sorted(used) if not re.search(rf'(?<![-\w]){k}\s*:\s*[\'\"]',c)]
print('missing:', miss or 'none')
"
```

### State globals (~1880)
- `transactions[]` — บิลตัวแทนหลัก (keyed by `agentId`)
- `agents[]`, `agentGroups[]`, `groupFilter`, `_fullscreenAgentId`
- `subTransactions[]` (keyed by `subAgentId`), `subAgents[]`
- `currentUser`, `viewingAsUser` (admin view-as mode)
- test accounts: `mahudone`, `mhdpro07` (mhdpro07 = admin หลัก)

### STORE_KEYS (localStorage) ~2390
`tx`, `agents`, `agentGroups`, `subAgents`, `subTx`

### ฟังก์ชันหลัก (grep หาเลขบรรทัดสด)
- `loadState`, `saveTx/saveAgents/saveAgentGroups/saveSubAgents/saveSubTx`
- `pushCloudData` — เซฟขึ้น cloud (มี token refresh + retry 401 + pending-save queue)
- `loadCloudData` — โหลดจาก cloud (กัน 401 ทับ local)
- `refreshSessionIfNeeded(force)` — **singleton** (กัน refresh_token rotation ชนกัน)
- `callAdminFn(action, args)` — เรียก edge function (refresh + retry 401)
- `renderAgents` / `renderSubAgents` — วาดการ์ดตัวแทน
- `renderBillListByDate` / `renderSubBillListByDate` — วาดรายการบิลแยกวัน
- `buildBillRow` / `buildSubBillRow` + `addBillRow` / `addSubBillRow` — ฟอร์มเพิ่มบิล
- `showSaveConfirmation` / `showSubSaveConfirmation` — modal ยืนยันก่อนบันทึก
- `showConfirm(title, text, onConfirm, dangerous=true, opts={})` — popup ยืนยันกลาง
- `openExportModal(agentId, isSub)` + `doExportExcel/doExportPdf/doExportPng`
- `buildExportSeqMaps()` / `exportSeqLabel(t, maps)` — เลข #N ใน export

---

## 5. ระบบสำคัญ + ข้อควรระวัง

### 5.1 Token refresh (session) — **จุดเปราะบาง**
Supabase access token อายุ ~1 ชม. ต้อง refresh ด้วย refresh_token
- **refresh_token ใช้ครั้งเดียว (rotation)** → ถ้าเรียก refresh พร้อมกันหลายที่จะชนกัน → หลุด login
- แก้ด้วย `_refreshInFlight` singleton — เรียกพร้อมกันกี่ครั้งใช้ผลเดียว **ห้ามทำลาย pattern นี้**
- `pushCloudData` มี pending-save retry queue (`_pendingCloudSave`) — save fail แล้วลองใหม่ทุก 20 วิ
- keep-alive refresh ทุก 5 นาที + refresh ตอน tab กลับมา visible
- warm-up: refresh token ตอนเปิด confirm-save modal (ระหว่างผู้ใช้อ่าน token จะ fresh พอดี)

### 5.2 Realtime (v1.99.53+)
- Supabase Realtime (WebSocket) subscribe row ตัวเอง filter `id=eq.{userId}`
- **ต้องเปิดฝั่ง Supabase ก่อน:** `alter publication supabase_realtime add table user_data;`
- Smart merge: เก็บค่าฟอร์มที่กรอกค้าง (`captureOpenFormState`) → merge → คืนค่า (`restoreOpenFormState`) — ไม่รบกวนที่กรอกอยู่
- กัน echo loop: เทียบ `realtimeDataSignature` + timestamp 8 วิ
- Admin view-as → subscribe row ของ user ที่ดูอยู่
- จุดสถานะเขียวกระพริบข้างวันที่บน header = เชื่อมต่ออยู่

### 5.3 โหมดคำนวณในช่องจำนวนเงิน (calc + K mode)
- **calc mode** (ปุ่ม 🧮): พิมพ์ `1+2=` → `1+2=3` (เห็นนิพจน์ค้างไว้)
- **K mode** (ปุ่ม K): พิมพ์ `1` → เก็บเป็น `1000` (×1000)
- ทั้งคู่ติ๊กเป็นค่าเริ่มต้น
- helper: `evalExpression`, `handleCalcAmountInput`, `resolveCalcAmount`, `readBillAmount`
- **อ่านค่า amount ต้องใช้ `readBillAmount(inp)` เสมอ** (ไม่ใช่ `parseAmount`) — จัดการ K + calc + comma
- พิมพ์ต่อหลัง `=`: ตัวเลข→เริ่มใหม่, operator→ต่อยอด (เหมือนเครื่องคิดเลข)
- มี test harness — extract 6 ฟังก์ชันแล้ว node run (ดู v1.99.55 log)

### 5.4 ลำดับบิล + เลข #N (v1.99.44-45)
- `uid()` มี monotonic counter กัน id ชนกันใน ms เดียว
- เรียงด้วย secondary key = array index ใน `transactions[]` (ไม่ใช่ id.localeCompare — id เก่าอาจ random)
- income/expense มีเลข #N แยกชุด
- export เรียง + แสดง #N ตรงกับแอป (v1.99.56)

### 5.5 Export swap perspective (สำคัญ — งงง่าย)
ในไฟล์ export **สลับมุมมอง**: income data → คอลัมน์ "ລາຍຈ່າຍ" สีแดง, expense data → คอลัมน์ "ລາຍຮັບ" สีเขียว (เพราะมุมมองตัวแทน) — ดู comment ในโค้ด `doExportExcel` อย่าไปแก้ให้ "ตรง" โดยไม่เข้าใจ
- PDF ภาษาไทยต้องใช้ merged font (DejaVu Lao + Loma Thai) — base64 ใน `PDF_FONT_REGULAR_B64`
- PNG ใช้ canvas ไม่พึ่ง dependency
- Excel/PDF/PNG โหลด lib จาก CDN (jsdelivr) ตอน runtime

### 5.6 Popup / z-index ladder (v1.99.55)
เรียงลำดับ (ห้ามสลับ): header 50 < fab 40 < banner 80 < **fullscreen 90** < modal 100 < confirm 160 < app-loading 190 < toast/auth 200 < realtime-indicator 250
- **fullscreen การ์ดต้องต่ำกว่า modal** ไม่งั้น popup ยืนยันจะถูกซ่อนหลังการ์ด (เคยเป็นบั๊กใหญ่)
- `showConfirm(..., dangerous=false)` = popup ปกติ (ไอคอน ✓ ม่วง), `dangerous=true` = ลบ (ถังขยะแดง + password)

---

## 6. Supabase Backend

### ตาราง `user_data`
- คอลัมน์: `id` (uuid, = auth user id), `username`, `role`, `data` (jsonb เก็บ transactions/agents/ฯลฯ), `updated_at`
- RLS: user เห็นเฉพาะ row ตัวเอง, admin เห็นทุก row (ผ่าน edge function service role)

### Edge Function `admin-users` (ไฟล์ `admin-users-edge-function.ts`)
actions: อนุมัติ/ปฏิเสธตัวแทน, สร้าง/รีเซ็ต user, `update_user_data`, `mirror_agent_to_owner` (admin แก้ → sync กลับ user)

### SQL setup files (ใน repo)
`supabase-setup-v2.sql`, `admin-setup.sql`, `admin-phase2.sql` — รันตอนตั้ง project ใหม่

---

## 7. ข้อจำกัด environment

- แอปเปิดผ่าน hosting URL (ไม่ใช่ `file://`)
- iOS numeric keypad **ไม่มีปุ่ม Enter** → ต้องมีปุ่ม → (next) + form submit fallback ในฟอร์มเพิ่มบิล
- iOS: input font-size ต้อง ≥ 16px กัน zoom-on-focus (ปุ่มบางตัวเล็กกว่าได้ถ้าไม่ใช่ input)
- viewport ตั้ง `maximum-scale=1.0, user-scalable=no`
- ปุ่มเล็ก (26-30px) มี tap area ล่องหน ±8px (`::after`) ให้กดง่าย ~44px

---

## 8. เวิร์กโฟลว์แนะนำสำหรับ session ใหม่

1. `git pull` ก่อนเริ่ม
2. อ่าน request → grep หา anchor ที่เกี่ยว
3. แก้ (main + sub ถ้ามี) ด้วย str_replace หรือ Python assert pattern
4. `node --check` ตรวจ syntax
5. ตรวจ i18n keys ถ้าเพิ่ม tt() ใหม่
6. bump `APP_VERSION`
7. `git add index.html && git commit -m "vX.X.X: ..." && git push`
8. บอกผู้ใช้ให้เทสอะไรบ้าง

---

## 9. ประวัติย่อ (context)

พัฒนามา 56 เวอร์ชัน (ถึง v1.99.56) ฟีเจอร์ล่าสุด: จัดกลุ่มตัวแทน + filter, การ์ดเต็มจอ, โหมด calc/K, ยอดติดต่อวัน, ระบบ realtime, pending-save retry, export เรียงตาม #N

Bug fix ใหญ่ที่เจอ: token rotation ชนกัน (singleton), fullscreen z-index บัง popup, uid ชนกันใน ms เดียว, calc พิมพ์ต่อหลัง =

---

*อัพเดตไฟล์นี้เมื่อเพิ่มระบบใหญ่ หรือเจอ gotcha ใหม่ที่ session ถัดไปควรรู้*
