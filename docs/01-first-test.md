# บทที่ 1 — ยิง load test ครั้งแรก

## 1.1 Script แรก

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

```bash
mkdir -p results
k6 run --summary-export=results/first-test.json scripts/01-first-test.js
```

> เพิ่ม `results/` ลงใน `.gitignore`

## ✅ Checkpoint

- [ ] รัน script แรกสำเร็จ
- [ ] กรอกตาราง 1.3 ครบและตอบคำถาม 3 ข้อได้
- [ ] อธิบายได้ว่าทำไมต้องดู p95 และ `http_req_waiting` บอกอะไร

➡️ [บทที่ 2 — Pattern ของ k6](02-k6-patterns.md)
