# บทที่ 6 — การคำนวณ TPS

บทนี้รวมสูตรทั้งหมดที่ใช้ตอบคำถาม "รับได้กี่ TPS / ไหวไหม" พร้อมตัวอย่างคำนวณ
(ตัวเลขในตัวอย่างเป็นตัวเลขสมมติ — ให้แทนด้วยผลจริงจากบทที่ 5)

---

## 6.1 TPS กับ RPS ต่างกันยังไง

| | นับอะไร | ใครสนใจ |
| --- | --- | --- |
| **RPS** (requests/s) | HTTP request ทุกตัว | engineer, infra |
| **TPS** (transactions/s) | **check-in ที่สำเร็จครบ flow** ต่อวินาที | business, capacity planning |

ความสัมพันธ์:

```
RPS = TPS × (จำนวน request ต่อ 1 transaction)
```

ตัวอย่าง: flow check-in มี 7 request → ที่ 10 TPS server ต้องรับ 70 RPS

> ระวัง: ถ้าคนพูดว่า "ระบบรับได้ 500 TPS" ต้องถามกลับว่า **transaction หมายถึงอะไร** — ใน repo นี้นิยามว่า
> **1 transaction = 1 check-in สำเร็จครบ flow** (นับจาก metric `checkin_success`)

---

## 6.2 วัด TPS จาก k6

### จาก summary ตอนจบ

```
checkin_success........: 5400   9.0/s
```

`9.0/s` = จำนวนสำเร็จ ÷ เวลาทั้ง test **รวมช่วง ramp-up/ramp-down** → ใช้ได้กับ `constant-arrival-rate` เท่านั้น

ถ้ามีช่วง ramp ให้คิดเฉพาะช่วงคงที่ (steady state):

```
TPS = จำนวน checkin_success ในช่วงคงที่ ÷ ระยะเวลาช่วงคงที่ (วินาที)
```

ห้ามใช้ `iterations` เป็น TPS เพราะนับรวมรอบที่ fail ด้วย

### จาก Grafana (PromQL)

```promql
# TPS สำเร็จ แบบ realtime
sum(rate(k6_checkin_success_total{testid="$testid"}[1m]))

# TPS เฉลี่ยตลอดช่วงที่เลือกบน dashboard (ตั้ง time range ให้คลุมแค่ steady state)
sum(increase(k6_checkin_success_total{testid="$testid"}[$__range])) / $__range_s

# TPS ที่ยิงเข้าไป (รวม fail) — ไว้เทียบว่า app ตามทันไหม
sum(rate(k6_iterations_total{testid="$testid"}[1m]))
```

ถ้าเส้น "TPS สำเร็จ" แยกออกจากเส้น "TPS ที่ยิง" = app เริ่มรับไม่ไหว

---

## 6.3 Little's Law — คำนวณจำนวน VUs

```
Concurrency = Throughput × Time
VUs         = TPS × iteration_duration
```

`iteration_duration` = เวลาของ request ทุกตัวใน flow + think time ทั้งหมด

**ตัวอย่าง**: เป้า 30 TPS, flow มี 7 request เฉลี่ยตัวละ 0.3s, think time รวม 8s

```
iteration_duration = 7 × 0.3 + 8 = 10.1 s
VUs ที่ต้องใช้       = 30 × 10.1  ≈ 303 VUs
```

→ ตั้ง `preAllocatedVUs: 350`, `maxVUs: 1000` (เผื่อตอน app ช้าลง iteration ยาวขึ้น ต้องใช้ VU มากขึ้น)

ใช้ Little's Law ฝั่ง server ได้ด้วย — **จำนวน request ที่ค้างอยู่ใน server พร้อมกัน**:

```
in-flight requests = RPS × avg response time
                   = 210 × 0.3 = 63 requests พร้อมกัน
```

ถ้า DB connection pool รวมทุก task มีแค่ 6 × 10 = 60 → เริ่มต่อคิวรอ connection แล้ว

**ใช้ตรวจผล**: ถ้าผลจาก k6 ไม่ตรงกับ Little's Law (เช่น 300 VUs, iteration 10s แต่ได้แค่ 15 TPS) → iteration จริงยาวกว่าที่คิด = app ช้าลงแล้ว

---

## 6.4 Capacity ต่อ task และทั้ง cluster

### ต่อ 1 task

| ค่า | สูตร | ตัวอย่าง |
| --- | --- | --- |
| Max TPS/task | วัดจาก breakpoint (บท 5.3) จุดสุดท้ายก่อนผิด SLO | 12 TPS |
| Safe TPS/task | `Max × 0.7` (เผื่อ 30% สำหรับ GC, noisy neighbor, autoscale ช้า, traffic เพี้ยน) | 12 × 0.7 = **8.4 TPS** |

ประมาณ Max คร่าวๆ จาก CPU ได้ด้วย (ถ้าคอขวดคือ CPU):

```
CPU-seconds ต่อ transaction = (CPU ที่ใช้ เป็น core) ÷ TPS
                            = (0.25 × 40%) ÷ 5 TPS = 0.02 cpu-s/txn
Max TPS/task (theoretical)  = 0.25 ÷ 0.02 = 12.5 TPS   (ที่ CPU 100%)
```

### ทั้ง cluster (6 tasks)

```
Scaling efficiency = Max TPS (6 tasks วัดจริง) ÷ (6 × Max TPS/task)
TPS cluster        = TPS/task × จำนวน task × efficiency
```

ตัวอย่าง: วัด 6 tasks ได้ 61 TPS

```
efficiency      = 61 ÷ (6 × 12) = 0.85
Safe TPS 6 task = 8.4 × 6 × 0.85 ≈ 42.8 TPS
```

### เผื่อ task หาย 1 ตัว (N-1)

ระหว่าง deploy แบบ rolling หรือ task crash จะเหลือ 5 tasks:

```
Safe TPS (N-1) = 8.4 × 5 × 0.85 ≈ 35.7 TPS
```

**ใช้ค่า N-1 เป็นตัวเลขหลักในรายงาน** เพราะเป็นสถานการณ์ที่เกิดขึ้นจริงทุกครั้งที่ deploy

---

## 6.5 Calibration — แปลงผล local เป็น production

CPU ของ Mac แรงกว่า vCPU ของ Fargate → ต้องหาตัวคูณปรับ ใช้ **CPU-seconds ต่อ transaction** เป็นตัวเทียบ

**ฝั่ง production** — เลือกช่วง 1 ชั่วโมงที่ traffic สูง จาก CloudWatch / APM:

```
cpu_per_txn_prod = (CPUUtilization เฉลี่ย × 0.25 vCPU × จำนวน task × 3600) ÷ จำนวน check-in ในชั่วโมงนั้น
```

**ฝั่ง local** — จาก baseline (บท 5.2) ที่ TPS ใกล้เคียงกับ prod ต่อ task:

```
cpu_per_txn_local = (CPU ของ app เป็น core) ÷ TPS
```

```
calibration factor = cpu_per_txn_prod ÷ cpu_per_txn_local
Max TPS/task (prod) ≈ Max TPS/task (local) ÷ factor     ← ใช้ได้เมื่อคอขวดคือ CPU
```

**ตัวอย่าง**
- prod: 6 tasks, CPU เฉลี่ย 30%, ชั่วโมงนั้นมี 36,000 check-in
  `cpu_per_txn_prod = 0.30 × 0.25 × 6 × 3600 ÷ 36000 = 0.045 cpu-s`
- local: `cpu_per_txn_local = 0.02 cpu-s`
- `factor = 0.045 ÷ 0.02 = 2.25`
- Max TPS/task (prod) ≈ 12 ÷ 2.25 ≈ **5.3 TPS**

ข้อควรระวัง:
- ถ้า app ใน prod รับ traffic อื่นนอกจาก check-in ด้วย CPU ของ prod จะรวม traffic นั้น → ใช้ CPU-seconds ต่อ **request** แทน หรือแยกเฉพาะ endpoint check-in จาก APM
- ถ้าคอขวดไม่ใช่ CPU (เช่น DB) factor นี้ใช้ไม่ได้ — ต้องเทียบที่ DB แทน
- **วิธีที่แม่นที่สุด**: ทำบทที่ 5.3 ซ้ำบน staging ที่มี task สเปกเดียวกับ prod 1 รอบ แล้วใช้ผลนั้นแทน factor

---

## 6.6 Peak TPS ที่ production ต้องรับ (demand)

**วิธีที่ดีที่สุด** — ดูจากข้อมูลจริง (log / APM / DB) หา **จำนวน check-in สูงสุดใน 1 นาที** ย้อนหลังอย่างน้อย 1–3 เดือน (รวมช่วงเทศกาล/วันที่ traffic สูงสุด)

```
Peak TPS = check-in สูงสุดใน 1 นาที ÷ 60
```

**ถ้ามีแค่ยอดรายวัน**:

```
Peak TPS = ยอดต่อวัน × สัดส่วนของชั่วโมงที่หนาแน่นที่สุด ÷ 3600 × burst factor
```

- สัดส่วนชั่วโมง peak: ดูจาก analytics (มักอยู่ที่ 10–20% ของทั้งวัน)
- burst factor: นาทีที่หนาแน่นที่สุดเทียบกับค่าเฉลี่ยของชั่วโมงนั้น (มัก 1.5–3 ถ้าไม่รู้ใช้ 2)

**ตัวอย่าง**: วันละ 200,000 check-in, ชั่วโมง peak มี 15%, burst factor 2

```
Peak TPS = 200,000 × 0.15 ÷ 3600 × 2 ≈ 16.7 TPS
```

**เผื่อการเติบโต / event** (โปรโมชัน, เทศกาล): `Design TPS = Peak TPS × growth` เช่น × 1.3

```
Design TPS = 16.7 × 1.3 ≈ 21.7 TPS
```

---

## 6.7 ตอบคำถาม "ไหวไหม"

```
Headroom = Safe TPS cluster (N-1, หลัง calibration) ÷ Design TPS
```

| Headroom | คำตอบ |
| --- | --- |
| ≥ 1.5 | ✅ **ไหว** มีที่ว่างพอ |
| 1.0 – 1.5 | ⚠️ **ไหวแบบเสี่ยง** — ช่วง peak + deploy/task crash อาจผิด SLO แนะนำเพิ่ม task หรือตั้ง autoscaling |
| < 1.0 | ❌ **ไม่ไหว** — ต้องเพิ่ม task / เพิ่ม CPU / แก้คอขวด |

**จำนวน task ที่ต้องมี:**

```
tasks ที่ต้องใช้ = ceil( Design TPS ÷ (Safe TPS/task × efficiency) ) + 1   (+1 สำหรับ N-1)
```

### ตัวอย่างเต็ม (ต่อจากด้านบน)

```
Max TPS/task (local)            = 12
calibration factor              = 2.25
Max TPS/task (prod)             = 12 ÷ 2.25           = 5.3
Safe TPS/task (prod)            = 5.3 × 0.7           = 3.7
efficiency                      = 0.85
Safe TPS cluster 6 tasks        = 3.7 × 6 × 0.85      = 18.9
Safe TPS cluster N-1 (5 tasks)  = 3.7 × 5 × 0.85      = 15.7
Design TPS                      = 21.7

Headroom (N-1)                  = 15.7 ÷ 21.7         = 0.72  ❌ ไม่ไหว
tasks ที่ต้องใช้                 = ceil(21.7 ÷ (3.7 × 0.85)) + 1 = ceil(6.9) + 1 = 8 tasks
```

→ คำตอบตัวอย่าง: "สเปกปัจจุบัน 6 tasks รับได้อย่างปลอดภัย ~19 TPS (~16 TPS ระหว่าง deploy) แต่ช่วง peak ต้องการ ~22 TPS → **ไม่พอ** ต้องเพิ่มเป็น 8 tasks หรือเพิ่ม CPU ต่อ task เป็น 512 แล้ววัดใหม่"

> ถ้าคอขวดคือ CPU การเพิ่ม CPU เป็น 512 (0.5 vCPU) มักได้ TPS/task เพิ่มเกือบ 2 เท่า — แต่ **ต้องวัดใหม่** ห้ามคูณเอา เพราะ runtime บางตัว (เช่น JVM GC, Node single-thread) ไม่ได้ใช้ CPU เพิ่มแบบเชิงเส้น

---

## โจทย์ 6

ใช้ผลจริงจากบทที่ 5 กรอกตารางนี้ (คัดลอกไปใส่ในรายงานบทที่ 7):

| ค่า | สูตร | ค่าที่ได้ |
| --- | --- | --- |
| Requests ต่อ transaction | นับจาก flow map | |
| Max TPS/task (local) | บท 5.3 | |
| CPU-s/txn (local) | 6.5 | |
| CPU-s/txn (prod) | 6.5 | |
| Calibration factor | | |
| Max TPS/task (prod) | | |
| Safe TPS/task (prod) | × 0.7 | |
| Scaling efficiency | บท 5.5 | |
| Safe TPS cluster (6) | | |
| Safe TPS cluster (N-1) | | |
| Peak TPS (prod จริง) | 6.6 | |
| Design TPS | × growth | |
| **Headroom** | | |
| **Tasks ที่ต้องใช้** | | |

➡️ [บทที่ 7 — สรุปผล](07-final-report.md)
