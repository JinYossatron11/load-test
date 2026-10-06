# load-test

คู่มือฝึก Load Test ด้วย **k6 + Prometheus + Grafana** แบบลงมือทำเองตั้งแต่ต้นจนจบ
ทุกอย่างรันบนเครื่องตัวเอง (local) ไม่ต้องยิง environment จริง

> repo นี้ไม่มีโค้ดสำเร็จรูปให้ — ทุกไฟล์ (docker-compose, script k6, dashboard) **คุณเขียนเองตามคู่มือ**
> แต่ละบทมีโจทย์ให้ทำ และมีเฉลยซ่อนไว้ใน `<details>` ไว้เปิดดูเมื่อติด

---

## เป้าหมายปลายทาง

เมื่อทำจบทุกบท คุณต้องตอบคำถามนี้ได้ด้วยตัวเลขที่วัดมาจริง:

> **"สเปก production ตอนนี้ (0.25 vCPU / 1 GB RAM × 6 tasks) รับ check-in ได้กี่ TPS?
> และพอสำหรับ peak ที่เกิดขึ้นจริงไหม?"**

ผลลัพธ์สุดท้ายคือรายงาน 1 หน้า ([บทที่ 7](docs/07-final-report.md)) ที่มี:

| ตัวเลข | ความหมาย |
| --- | --- |
| Max TPS ต่อ 1 task | จุดที่ app 1 task ยังผ่าน SLO ได้ (p95, error rate) |
| Safe TPS ต่อ 1 task | Max TPS หลังเผื่อ headroom |
| TPS รวม 6 tasks | วัดจริงจาก 6 replicas + scaling efficiency |
| Peak TPS ที่ต้องรองรับ | คำนวณจาก traffic จริงของ production |
| คำตอบ | ไหว / ไม่ไหว / ไหวแบบเสี่ยง + ต้องเพิ่มกี่ task |

---

## Roadmap

| # | บท | ทำไปเพื่ออะไร | สิ่งที่จะได้ | เสร็จเมื่อ |
| --- | --- | --- | --- | --- |
| 0 | [เตรียมเครื่อง](docs/00-setup.md) | มีสนามซ้อมที่ยิงได้เต็มที่โดยไม่กระทบใคร | k6, Docker, target ฝึกยิง (QuickPizza) | `k6 version` และเปิด QuickPizza ได้ |
| 1 | [ยิง load test ครั้งแรก](docs/01-first-test.md) | เข้าใจ VU / TPS และอ่านตัวเลขผลลัพธ์ออก | VU กับ TPS คืออะไร ได้มาจากไหน, เขียน script แรก, อ่าน end-of-test summary | อธิบายได้ว่า `http_req_duration p(95)` คืออะไร |
| 2 | [Pattern ของ k6](docs/02-k6-patterns.md) | เขียน script ที่ผลเชื่อถือได้ ไม่ดีเกินจริง | checks, thresholds, groups, tags, custom metrics, scenarios/executors, **ยิงพร้อมกัน vs ทยอยยิง**, test types, data | เขียน smoke / load / stress / spike ได้ |
| 3 | [Monitor ด้วย Grafana](docs/03-grafana-monitoring.md) | เห็นว่า "เริ่มพังที่กี่ TPS" และ "พังเพราะอะไร" | docker-compose: Prometheus + Grafana, ส่ง metric จาก k6, สร้าง dashboard, **ดูกรณี app ตาย / restart / ค้าง** | เห็นกราฟ VUs, RPS, p95, error rate แบบ realtime |
| 4 | [Check-in flow ทั้ง flow](docs/04-checkin-flow.md) | วัดโหลดของ check-in จริง ไม่ใช่แค่ endpoint เดียว | ต่อ transaction flow จริง (correlation, test data, think time) | flow check-in ผ่าน smoke test 100% |
| 5 | [Capacity test สเปก production](docs/05-capacity-test.md) | ได้ตัวเลขที่สะท้อนเครื่อง production ไม่ใช่เครื่องเรา | จำกัด CPU/RAM เท่า prod, หาจุดแตก 1 task, ยืนยันด้วย 6 tasks | ได้ Max TPS ต่อ task และ TPS รวม 6 tasks |
| 6 | [การคำนวณ TPS](docs/06-tps-calculation.md) | แปลงผล test เป็นคำตอบ "ไหวไหม / ต้องกี่ task" | TPS vs RPS, Little's Law, คิด VUs, คิด capacity, คิด peak demand | คำนวณ headroom ได้เอง |
| 7 | [สรุปผล / รายงาน](docs/07-final-report.md) | ให้คนอื่นตัดสินใจได้และรันซ้ำได้ | template รายงาน | ตอบได้ว่า "ไหวไหม" พร้อมหลักฐาน |

> **ทำไมต้องเรียงแบบนี้:** แต่ละบทตัดตัวแปรทีละอย่าง — ฝึกเครื่องมือบน app ที่รู้ว่าทำงานถูก (บท 1–3) → ทำ script check-in ให้ถูกก่อน (บท 4) → ค่อยใส่ข้อจำกัดของ production (บท 5) → แล้วจึงคำนวณ (บท 6)
> ถ้าข้ามขั้น เมื่อเจอ error จะแยกไม่ออกว่ามาจาก script ผิด, ตั้งค่า monitor ผิด หรือ app รับไม่ไหวจริง
>
> แต่ละบทเริ่มด้วยกล่อง **🎯 บทนี้ทำไปเพื่ออะไร** และทุกหัวข้อย่อยมีบรรทัด **ทำไปเพื่อ:** บอกเหตุผลของขั้นตอนนั้น
>
> แนะนำให้ทำตามลำดับ บท 1–3 ฝึกกับ QuickPizza (แอปตัวอย่างของ Grafana) ก่อน
> แล้วบท 4–7 ค่อยเปลี่ยนมาใช้ app check-in จริง

---

## โครงสร้าง repo ที่คุณจะสร้างขึ้นระหว่างทำ

```
load-test/
├── README.md
├── docs/                         # คู่มือ (มีให้แล้ว)
├── docker-compose.yml            # บท 3, 5  — Prometheus, Grafana, app
├── infra/
│   ├── prometheus/prometheus.yml
│   ├── grafana/provisioning/...
│   └── nginx/nginx.conf          # บท 5  — load balancer หน้า 6 tasks
├── scripts/
│   ├── 01-first-test.js          # บท 1
│   ├── patterns/*.js             # บท 2
│   └── checkin/                  # บท 4–5
│       ├── flow.js               # ขั้นตอน check-in (ใช้ซ้ำทุก test type)
│       ├── smoke.js
│       ├── load.js
│       └── breakpoint.js
├── data/                         # test data (ไม่ commit ข้อมูลจริง)
└── results/                      # summary JSON / screenshot (gitignore)
```

## คำศัพท์ที่ใช้บ่อย

| คำ | ความหมาย |
| --- | --- |
| **VU** (Virtual User) | ผู้ใช้จำลอง 1 คนที่ "กำลังทำรายการอยู่พร้อมกัน" วนรัน function `default` ซ้ำไปเรื่อยๆ — ไม่ใช่จำนวนผู้ใช้ทั้งหมด ดู [บท 1.0](docs/01-first-test.md) |
| **Iteration** | การรัน function 1 รอบ = ใน repo นี้คือ check-in 1 ครั้ง |
| **RPS** | HTTP requests ต่อวินาที |
| **TPS** | *business transactions* ต่อวินาที (check-in สำเร็จ / วินาที) ได้จากการนับ ไม่ได้ตั้งเอง — ดู [บท 1.0](docs/01-first-test.md) และ [บท 6](docs/06-tps-calculation.md) |
| **p95** | 95% ของ request เร็วกว่าค่านี้ |
| **SLO** | เกณฑ์ที่ยอมรับได้ เช่น p95 < 800ms และ error < 1% |
| **Task** | 1 container ของ app (ECS task) — prod = 256 CPU units (0.25 vCPU), 1 GB |
