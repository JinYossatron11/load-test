# บทที่ 1 — ยิง load test ครั้งแรก

> ### 🎯 บทนี้ทำไปเพื่ออะไร
>
> - **เป้าหมาย:** เข้าใจว่า k6 สร้างโหลดยังไง (VU, iteration, think time) และอ่านตัวเลขผลลัพธ์ออก
> - **ถ้าข้ามบทนี้:** จะเห็นตัวเลขเยอะๆ แต่ไม่รู้ว่าตัวไหนสำคัญ เช่น ดู avg แทน p95 แล้วสรุปว่า "เร็ว" ทั้งที่ผู้ใช้ 5% รอหลายวินาที
> - **ผลที่ได้ไปใช้ต่อ:** ความเข้าใจ VU ↔ throughput ↔ latency ที่ใช้ตลอดทั้ง roadmap และเป็นพื้นฐานของสูตร Little's Law ในบทที่ 6

## 1.0 VU กับ TPS คืออะไร (อ่านก่อนเริ่ม)

> **ทำไปเพื่อ:** VU กับ TPS เป็น 2 ตัวเลขที่ใช้ตลอดทั้ง roadmap — ถ้าไม่เข้าใจว่ามันคืออะไรและได้มาจากไหน จะตั้งค่า test ผิด (เช่น ตั้ง VU = จำนวนผู้ใช้ทั้งหมดในระบบ) และอ่านผลผิด

### VU (Virtual User) คืออะไร

**1 VU = ผู้ใช้จำลอง 1 คน** ที่ k6 สร้างขึ้นมา ทำงานตาม `default function` ซ้ำไปเรื่อยๆ จนหมดเวลา
VU แต่ละตัว **ทำทีละอย่างตามลำดับ** เหมือนคนจริง: ส่ง request → **รอ** จนได้ response → พัก (think time) → ทำขั้นต่อไป → จบรอบแล้วเริ่มรอบใหม่

```
VU 1 คน (request ใช้ 0.2s, sleep 1s)

เวลา(s)  0    0.2         1.2  1.4         2.4  2.6         3.6
         |req |   sleep    |req |   sleep    |req |   sleep    |
         └────── รอบที่ 1 ──────┘└────── รอบที่ 2 ──────┘└────── รอบที่ 3 ──────┘
                1.2s / รอบ  →  VU 1 ตัวทำได้ 1 ÷ 1.2 ≈ 0.83 รอบต่อวินาที
```

สิ่งที่ต้องเข้าใจ:
- **1 VU ≠ 1 request ต่อวินาที** — VU 1 ตัวยิงได้เร็วแค่ไหนขึ้นกับว่าแต่ละรอบใช้เวลานานเท่าไร
- **ถ้า app ช้าลง VU ก็ยิงช้าลง** เพราะต้องรอ response ก่อน (ตัวอย่าง: request ช้าลงเป็น 2s → 1 รอบใช้ 3s → เหลือ 0.33 รอบ/วินาที)
- **VU ≠ จำนวนผู้ใช้ทั้งหมดในระบบ** — ผู้ใช้ 10,000 คนต่อวัน ไม่ได้แปลว่าต้องตั้ง 10,000 VU เพราะเขาไม่ได้ใช้งานพร้อมกัน VU คือ **ผู้ใช้ที่กำลังทำรายการอยู่ในขณะเดียวกัน (concurrent)**

### ค่า VU ได้มาจากไหน

มี 3 ทาง ขึ้นกับว่ารู้ข้อมูลอะไร:

**ทาง 1 — รู้จำนวนผู้ใช้พร้อมกัน** (จาก analytics เช่น Google Analytics realtime, จำนวน session ที่ active)
→ ตั้ง VU เท่านั้นได้เลย (`vus: 200`) พร้อมใส่ think time ให้เหมือนคนจริง

ถ้าไม่มีตัวเลขพร้อมกันตรงๆ คำนวณได้จาก:

```
ผู้ใช้พร้อมกัน = จำนวนคนที่เข้ามาต่อวินาที × เวลาที่ 1 คนใช้ทำรายการจนจบ (วินาที)
```

ตัวอย่าง: ช่วง peak มีคนเริ่ม check-in นาทีละ 300 คน (= 5 คน/วินาที) แต่ละคนใช้เวลาทั้ง flow 60 วินาที
→ ผู้ใช้พร้อมกัน = 5 × 60 = **300 คน** → ใช้ 300 VU

**ทาง 2 — รู้ TPS ที่ต้องการ** (นี่คือกรณีของ roadmap นี้)

```
VU ที่ต้องใช้ = TPS เป้าหมาย × เวลาของ 1 รอบ (วินาที)
```

ตัวอย่าง: อยากยิง 10 TPS, 1 รอบ check-in ใช้ 6 วินาที (request ทุกตัว + think time) → 10 × 6 = **60 VU**
(สูตรนี้ชื่อ Little's Law — รายละเอียดเพิ่มเติมในบทที่ 6.3)

**ทาง 3 — ให้ k6 จัดการเอง** (open model / arrival-rate — บทที่ 2.6)
→ เรากำหนด **TPS** (`rate: 10`) แทนจำนวน VU แล้ว k6 หยิบ VU จาก pool (`preAllocatedVUs`, `maxVUs`) มาใช้เท่าที่จำเป็น
VU ในกรณีนี้เป็นแค่ "คนงาน" ที่ช่วยยิงให้ได้ตามอัตรา **ถ้าจำนวน VU ที่ใช้งานในกราฟพุ่งขึ้น = app ช้าลง** (แต่ละรอบนานขึ้นจึงต้องใช้คนงานมากขึ้น)

### TPS (Transactions Per Second) คืออะไร

**TPS = จำนวน transaction ที่สำเร็จ ÷ จำนวนวินาที**

**transaction** = งาน 1 ชิ้นที่มีความหมายต่อ business — ใน roadmap นี้คือ **check-in สำเร็จ 1 ครั้งตั้งแต่ step แรกจนจบ** (ซึ่งอาจมี 5–10 request ข้างใน)
ต่างจาก **RPS** ที่นับทุก HTTP request

```
1 transaction (check-in 1 ครั้ง)
├── POST /login          ┐
├── GET  /booking        │
├── POST /passenger      ├─ 5 requests
├── POST /checkin        │
└── GET  /boarding-pass  ┘

10 TPS  =  check-in สำเร็จ 10 ครั้งต่อวินาที  =  50 RPS
```

### ค่า TPS ได้มาจากไหน

TPS **ไม่ได้ตั้งเอาเอง แต่ได้จากการนับ** — k6 นับจำนวน transaction ที่สำเร็จ แล้วหารด้วยเวลา

```
ยิง 10 นาที (600 วินาที) มี check-in สำเร็จ 5,400 ครั้ง
TPS = 5,400 ÷ 600 = 9 TPS
```

ใน k6 นับจาก (รายละเอียดบทที่ 2.5 และ 6.2):
- `iterations` — จำนวนรอบที่รันจบ (นับรวมรอบที่ fail ด้วย ใช้ดูคร่าวๆ)
- custom counter `checkin_success` — นับเฉพาะรอบที่สำเร็จทุก step **← ใช้ค่านี้เป็น TPS จริง**

คำว่า "TPS" ใน roadmap นี้มี 4 แบบ อย่าสับสน:

| ชื่อ | ความหมาย | ได้มาจาก |
| --- | --- | --- |
| **Target TPS** | อัตราที่เราสั่งให้ k6 ยิง | เราตั้งเอง (`rate: 10`) |
| **Achieved TPS** | ทำสำเร็จได้จริง | k6 นับ `checkin_success` ÷ เวลา — ถ้าน้อยกว่า Target = app เริ่มรับไม่ไหว |
| **Max TPS (capacity)** | สูงสุดที่ app รับได้โดยยังผ่าน SLO | จาก breakpoint test (บทที่ 5) |
| **Peak TPS (demand)** | สูงสุดที่ production ต้องรับจริง | จาก log/APM ของ production (บทที่ 6.6) |

คำถาม "ไหวไหม" ของ roadmap นี้คือการเทียบ **Max TPS** กับ **Peak TPS**

### VU กับ TPS สัมพันธ์กันยังไง

```
TPS = จำนวน VU ÷ เวลาของ 1 รอบ (วินาที)
```

| VU | request ใช้ | think time | 1 รอบใช้ | TPS |
| --- | --- | --- | --- | --- |
| 10 | 0.2 s | 1 s | 1.2 s | 10 ÷ 1.2 ≈ **8.3** |
| 10 | 0.2 s | ไม่มี | 0.2 s | 10 ÷ 0.2 = **50** |
| 10 | 2.0 s (app ช้าลง) | 1 s | 3.0 s | 10 ÷ 3 ≈ **3.3** |
| 100 | 0.2 s | 1 s | 1.2 s | 100 ÷ 1.2 ≈ **83** (ถ้า app ยังเร็วเท่าเดิม) |

ข้อสังเกตจากตาราง:
- VU เท่ากัน แต่ TPS ต่างกันได้มาก ขึ้นกับ think time และความเร็วของ app → **การบอกว่า "ทดสอบ 100 VU" อย่างเดียวไม่มีความหมาย** ต้องบอก TPS ด้วย
- แถวที่ 3: app ช้าลง TPS ก็ตกเอง ทั้งที่ VU เท่าเดิม — นี่คือเหตุผลที่การ **หา capacity ต้องกำหนด TPS** (arrival-rate) ไม่ใช่กำหนด VU
- แถวที่ 4: เพิ่ม VU แล้ว TPS เพิ่มตามได้ **ก็ต่อเมื่อ app ยังไม่เต็ม** — เมื่อ app เต็ม เพิ่ม VU แค่ไหน TPS ก็ไม่ขึ้น แต่ response time จะพุ่งแทน (จะได้เห็นจริงในข้อ 1.3)

### ดูค่าเหล่านี้ได้ที่ไหน

| ค่า | ใน summary ตอนจบ | ใน Grafana (บทที่ 3) |
| --- | --- | --- |
| VU | `vus`, `vus_max` | `k6_vus` |
| รอบที่รันจบ/วินาที | `iterations ... /s` | `rate(k6_iterations_total[1m])` |
| TPS สำเร็จ | `checkin_success ... /s` | `rate(k6_checkin_success_total[1m])` |
| RPS | `http_reqs ... /s` | `rate(k6_http_reqs_total[1m])` |
| เวลาของ 1 รอบ | `iteration_duration` | `k6_iteration_duration_avg` |

---

## 1.1 Script แรก

> **ทำไปเพื่อ:** เห็นโครงขั้นต่ำของ k6 script: `options` = จะยิงแรงแค่ไหน, `default function` = ผู้ใช้ 1 คนทำอะไร

สร้างไฟล์ `scripts/01-first-test.js`

```js
import http from 'k6/http';
import { sleep } from 'k6';

// options = การตั้งค่าโหลด
export const options = {
  vus: 5,          // ผู้ใช้จำลองพร้อมกัน 5 คน
  duration: '30s', // ยิงนาน 30 วินาที
};

// default function = สิ่งที่ VU แต่ละตัวทำ "วนซ้ำ" จนหมดเวลา
export default function () {
  http.get('http://localhost:3333/');
  sleep(1); // think time: ผู้ใช้จริงไม่ได้กดรัวๆ
}
```

รัน:

```bash
k6 run scripts/01-first-test.js
```

## 1.2 อ่าน summary ตอนจบ

> **ทำไปเพื่อ:** รู้ว่าต้องดู metric ไหนก่อน และแยกได้ว่าช้าเพราะ server คิดนาน (`waiting`) หรือเพราะ network/connection — เพื่อชี้ปัญหาให้ถูกฝั่ง

ตัวที่ต้องดูเป็นอันดับแรก:

| Metric | อ่านว่า | ดูอะไร |
| --- | --- | --- |
| `http_reqs` | จำนวน request ทั้งหมด และ **/s** | นี่คือ RPS |
| `http_req_duration` | เวลาตั้งแต่ส่ง request จนได้ response ครบ | ดู **p(95)** ไม่ใช่ avg |
| `http_req_failed` | % request ที่ status ≥ 400 หรือ error | ควรเป็น 0% |
| `iterations` | จำนวนรอบที่ default function รันจบ และ **/s** | ถ้า 1 iteration = 1 transaction นี่คือ TPS |
| `iteration_duration` | เวลาของ 1 รอบ (รวม sleep) | ใช้คำนวณ VUs ในบทที่ 6 |
| `vus` / `vus_max` | จำนวน VU | |

ทำไมดู p95 ไม่ดู avg? — avg ซ่อน request ที่ช้า: ถ้า 90 request ใช้ 100ms และ 10 request ใช้ 3s, avg = 390ms ดูโอเค แต่ผู้ใช้ 10% รอ 3 วินาที

`http_req_duration` แยกย่อยได้เป็น:
`blocked` (รอ connection ว่าง) → `connecting` → `tls_handshaking` → `sending` → **`waiting` (TTFB = เวลาที่ server คิด)** → `receiving`
ถ้า `waiting` สูง = server ช้า, ถ้า `blocked`/`connecting` สูง = ปัญหา connection / ฝั่ง client

## 1.3 ทดลองเปลี่ยนค่าแล้วสังเกต

> **ทำไปเพื่อ:** เห็นกับตาว่า VU, think time และความเร็ว server สัมพันธ์กันยังไง และเห็นอาการ "เพิ่มโหลดแต่ throughput ไม่เพิ่ม" ซึ่งคือสิ่งที่เราจะตามหาตอนวัด capacity

ทำทีละข้อ แล้วจดผลลงตาราง:

| การทดลอง | RPS | p95 | iterations/s |
| --- | --- | --- | --- |
| `vus: 5`, `sleep(1)` | | | |
| `vus: 20`, `sleep(1)` | | | |
| `vus: 20`, ลบ `sleep` ออก | | | |
| `vus: 100`, ลบ `sleep` ออก | | | |

คำถามให้ตอบ:
1. ทำไม 5 VUs + `sleep(1)` ได้ประมาณ 5 iterations/s?
2. ตอนไม่มี sleep ทำไม RPS พุ่งขึ้นมาก? ผลแบบนี้สะท้อนผู้ใช้จริงไหม?
3. ตอนเพิ่มจาก 20 → 100 VUs (ไม่มี sleep) RPS เพิ่ม 5 เท่าไหม? p95 เป็นยังไง?

<details>
<summary>เฉลย</summary>

1. VU 1 ตัวใช้เวลา ~1s ต่อรอบ (request ไม่กี่ ms + sleep 1s) → 1 VU ≈ 1 iteration/s → 5 VUs ≈ 5/s
   นี่คือ **Little's Law**: `throughput = VUs / iteration_duration` (รายละเอียดบทที่ 6)
2. ไม่มี sleep → VU ยิงทันทีที่ได้ response → throughput ถูกจำกัดแค่ความเร็ว server ไม่สะท้อนผู้ใช้จริง แต่มีประโยชน์ตอนอยากหา "เพดาน" ของ server
3. ไม่เพิ่มตามสัดส่วน — เมื่อ server เริ่มเต็ม RPS จะแบน แต่ p95 พุ่งขึ้น (request ไปต่อคิว) นี่คือสัญญาณของ **saturation** ที่เราจะตามหาในบทที่ 5
</details>

## 1.4 ใช้ environment variable แทน hard-code URL

> **ทำไปเพื่อ:** ใช้ script เดียวยิงได้หลาย environment (local / 1 task / 6 tasks / staging) และปรับโหลดได้โดยไม่ต้องแก้โค้ด

```js
const BASE_URL = __ENV.BASE_URL || 'http://localhost:3333';
```

```bash
k6 run -e BASE_URL=http://localhost:3333 scripts/01-first-test.js
```

และ override options จาก command line ได้ (สะดวกตอนทดลอง):

```bash
k6 run --vus 10 --duration 1m scripts/01-first-test.js
```

## 1.5 เก็บผลเป็นไฟล์

> **ทำไปเพื่อ:** เก็บหลักฐานผลแต่ละรอบไว้เทียบกันและแนบในรายงานบทที่ 7

```bash
mkdir -p results
k6 run --summary-export=results/first-test.json scripts/01-first-test.js
```

> เพิ่ม `results/` ลงใน `.gitignore`

## ✅ Checkpoint

- [ ] อธิบายได้ว่า VU คืออะไร ไม่ใช่จำนวนผู้ใช้ทั้งหมด และคำนวณ VU จาก TPS เป้าหมายได้
- [ ] อธิบายได้ว่า TPS ได้มาจากการนับ และต่างจาก RPS ยังไง
- [ ] รัน script แรกสำเร็จ
- [ ] กรอกตาราง 1.3 ครบและตอบคำถาม 3 ข้อได้
- [ ] อธิบายได้ว่าทำไมต้องดู p95 และ `http_req_waiting` บอกอะไร

➡️ [บทที่ 2 — Pattern ของ k6](02-k6-patterns.md)
