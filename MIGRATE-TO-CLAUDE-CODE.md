# ย้าย phuaparuay ไป Claude Code — ขั้นตอน

> ทำครั้งเดียว หลังจากนี้แค่เปิด Claude Code ในโฟลเดอร์โปรเจกต์แล้วสั่งงานได้เลย

## ขั้นที่ 1 — Clone repo ลงเครื่อง (ถ้ายังไม่มี)

```bash
cd ~/projects            # หรือที่ไหนก็ได้ที่เก็บโปรเจกต์ (เช่นที่เดียวกับ paruay)
git clone https://github.com/mhdgroup01/phuaparuay.git
cd phuaparuay
```

ถ้า clone ไว้แล้ว แค่ `cd phuaparuay && git pull`

## ขั้นที่ 2 — วางไฟล์ใหม่ 3 ไฟล์ลงในโฟลเดอร์

ดาวน์โหลดจากแชตนี้แล้ววางในราก repo:
- `CLAUDE.md` ← คู่มือให้ Claude Code อ่าน (สำคัญสุด)
- `.gitignore`
- `index.html` ← เวอร์ชันล่าสุด v1.99.56 (ถ้าใน repo เก่ากว่า ให้ทับ)

> เช็คก่อนทับ: `grep "APP_VERSION" index.html` — เอาตัวที่เลขสูงกว่า

## ขั้นที่ 3 — เปิด Claude Code

```bash
cd ~/projects/phuaparuay
claude
```

Claude Code จะอ่าน `CLAUDE.md` อัตโนมัติ → รู้ convention ทั้งหมดทันที (เหมือนที่ทำกับ paruay)

## ขั้นที่ 4 — commit ไฟล์ใหม่

สั่ง Claude Code หรือทำเอง:
```bash
git add CLAUDE.md .gitignore index.html MIGRATE-TO-CLAUDE-CODE.md
git commit -m "chore: add CLAUDE.md + move to Claude Code workflow"
git push
```

## ขั้นที่ 5 — ลองสั่งงานแรก

พิมพ์ใน Claude Code เช่น:
> "เพิ่ม APP_VERSION เป็น 1.99.57 แล้ว push" (ลองให้ครบ loop: แก้ → node --check → commit → push)

ถ้า push ขึ้น GitHub Pages ได้ = migration สำเร็จ ✅

---

## หมายเหตุ

- **ไม่ต้องมี Node project / npm install** — โปรเจกต์นี้ไม่มี build step ใช้ `node --check` แค่ตรวจ syntax
- **Deploy = git push** — GitHub Pages อัพเดทเอง ~1 นาที ไม่ต้อง present_files แล้ว
- Edge function + SQL อยู่ในไฟล์ `.ts` / `.sql` — แก้ผ่าน Claude Code ได้ แต่ต้อง deploy แยกที่ Supabase Dashboard
- ทุกอย่างที่ต้องรู้เรื่องโครงสร้างโค้ด อยู่ใน `CLAUDE.md` แล้ว
