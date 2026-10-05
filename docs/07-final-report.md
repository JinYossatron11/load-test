# บทที่ 7 — สรุปผล (Capacity Report)

คัดลอก template ด้านล่างไปเป็น `results/REPORT-<yyyy-mm-dd>.md` แล้วกรอกด้วยผลจริง
หลักการ: **ทุกตัวเลขต้องอ้างถึง testid** ที่เปิดดูใน Grafana ได้ และมี screenshot แนบ

---

```markdown
# Check-in Capacity Report — <วันที่>

## 1. คำตอบ (อ่านแค่ส่วนนี้ก็พอ)

- สเปก production ปัจจุบัน: 0.25 vCPU / 1 GB × 6 tasks
- รับ check-in ได้อย่างปลอดภัย: **__ TPS** (ระหว่าง deploy / เหลือ 5 tasks: **__ TPS**)
- เพดานก่อนผิด SLO: **__ TPS**
- Peak ที่ต้องรองรับ (รวม growth): **__ TPS**
- Headroom: **__ เท่า** → ✅ ไหว / ⚠️ ไหวแบบเสี่ยง / ❌ ไม่ไหว
- ข้อเสนอ: <เช่น คง 6 tasks / เพิ่มเป็น N tasks / เพิ่ม CPU เป็น 512 / แก้คอขวดที่ ...>

## 2. นิยามและ SLO

- 1 transaction = check-in สำเร็จครบ flow (<step แรก> → <step สุดท้าย>), __ requests ต่อ transaction
- SLO: p95 ต่อ request < __ ms, error rate < 1%, check-in success > 99%

## 3. สภาพแวดล้อมทดสอบ

| | Production | Local test |
| --- | --- | --- |
| CPU / task | 0.25 vCPU (Fargate) | 0.25 CPU (Docker, <รุ่นเครื่อง>) |
| Memory / task | 1 GB | 1 GB |
| จำนวน task | 6 | 1 และ 6 |
| Load balancer | ALB | nginx |
| DB | <สเปก> | <สเปก> |
| ข้อมูลใน DB | <จำนวนแถวหลัก> | <จำนวนแถวหลัก> |
| External dependency | จริง | mock (delay __ ms) |
| Calibration factor | – | __ |

## 4. ผลการทดสอบ

| Test | testid | Target TPS | TPS สำเร็จ | p95 (ms) | Error % | CPU/task | Mem/task | ผ่าน? |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Baseline 1 task | | | | | | | | |
| Breakpoint 1 task | | | | | | | | |
| Confirm safe 1 task | | | | | | | | |
| Breakpoint 6 tasks | | | | | | | | |
| Confirm 6 tasks | | | | | | | | |
| Spike | | | | | | | | |
| Soak | | | | | | | | |

<screenshot dashboard ของ breakpoint 1 task และ 6 tasks — ชี้จุดที่เริ่มพัง>

## 5. การคำนวณ

<ตารางจากโจทย์บทที่ 6 พร้อมแสดงการแทนค่าในสูตร>

## 6. คอขวดที่พบ

| ลำดับ | คอขวด | หลักฐาน (panel / ค่า) | ผลกระทบ | แนวทางแก้ |
| --- | --- | --- | --- | --- |
| 1 | <เช่น CPU ของ app ชน 100% ที่ __ TPS> | | | |
| 2 | <เช่น step 06_confirm p95 พุ่งก่อน step อื่น> | | | |

## 7. ข้อจำกัดของการทดสอบ

- CPU ของเครื่อง local ต่างจาก Fargate → ใช้ calibration factor __ (ความแม่นยำ ±__%)
- DB / network อยู่บนเครื่องเดียวกัน latency ต่ำกว่าจริง
- <อื่นๆ>

## 8. ขั้นตอนถัดไป

- [ ] ยืนยันผลด้วยการรัน breakpoint บน staging สเปกเดียวกับ prod
- [ ] ตั้ง autoscaling / alarm ที่ __ TPS หรือ CPU __%
- [ ] รันซ้ำหลังแก้คอขวด / ก่อน release ใหญ่
```

---

## ✅ Checkpoint สุดท้าย

- [ ] รายงานตอบได้ใน 1 ประโยค: "6 tasks รับได้ __ TPS, peak ต้องการ __ TPS → ไหว/ไม่ไหว"
- [ ] ทุกตัวเลขอ้าง testid และมี screenshot
- [ ] แสดงการคำนวณครบ (คนอื่นคิดตามแล้วได้ผลเดียวกัน)
- [ ] ระบุคอขวดและข้อจำกัดของการทดสอบ
- [ ] คนอื่นในทีม clone repo แล้วรันซ้ำได้ด้วยคำสั่งใน README

🎉 จบ roadmap
