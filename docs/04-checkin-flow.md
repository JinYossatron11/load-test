# บทที่ 4 — ต่อ transaction flow check-in ทั้ง flow (local)

> ### 🎯 บทนี้ทำไปเพื่ออะไร
>
> - **เป้าหมาย:** ได้ script ที่จำลองผู้ใช้ check-in ครบทุก step เหมือนของจริง และรันซ้ำได้บนเครื่องตัวเอง
> - **ถ้าข้ามบทนี้:** ถ้ายิงแค่ endpoint เดียว (เช่น หน้าแรก) ตัวเลขจะไม่บอกอะไรเกี่ยวกับ check-in เลย — แต่ละ step ใช้ทรัพยากรไม่เท่ากัน และ step ที่เขียน DB มักเป็นคอขวด
> - **ผลที่ได้ไปใช้ต่อ:** `scripts/checkin/flow.js` ที่บทที่ 5 ใช้วัด capacity และนิยาม "1 transaction" ที่ใช้คิด TPS ในบทที่ 6

ตั้งแต่บทนี้เปลี่ยน target จาก QuickPizza เป็น **app check-in จริง** ที่รันบนเครื่อง

---

## 4.1 รัน app check-in บน local

> **ทำไปเพื่อ:** มีสภาพแวดล้อมที่ยิงได้เต็มที่โดยไม่กระทบใคร — mock ระบบภายนอกเพื่อไม่ยิงของจริง แต่ต้องใส่ delay เท่าของจริง ไม่อย่างนั้นผลจะดีเกินจริง

เพิ่ม service ของ app (และ dependency ทั้งหมด เช่น DB, Redis) เข้าไปใน `docker-compose.yml`

```yaml
  app:
    image: <checkin-app-image>      # หรือ build: ../path/to/checkin-app
    environment:
      - DB_HOST=db
      # ... config อื่นๆ ให้เหมือน production มากที่สุด
    ports: ["8000:8000"]
    depends_on: [db]

  db:
    image: postgres:16             # ใช้ engine + version เดียวกับ production
    environment:
      - POSTGRES_PASSWORD=postgres
```

> ยังไม่ต้องจำกัด CPU/RAM ในบทนี้ — บทนี้เป้าหมายคือ **flow ต้องถูก** ก่อน

**Dependency ภายนอก** (payment, ระบบของ partner, SMS/email): ห้ามยิงของจริง ให้ทำ mock แทน
- mock ต้องตอบ **ช้าเท่าของจริง** (ดู latency จาก production APM) ไม่อย่างนั้นผลจะดีเกินจริง
- ใช้ [WireMock](https://wiremock.org/) หรือ `mockserver` ตั้ง fixed delay ได้

## 4.2 Map flow ให้ครบก่อนเขียน script

> **ทำไปเพื่อ:** script ต้องยิง request ให้เหมือน browser/app จริงทุกตัว — request ที่ตกหล่นคือโหลดที่หายไป ทำให้ TPS ที่วัดได้สูงเกินจริง

เปิด browser DevTools → Network (หรือ export HAR) แล้วทำ check-in เองด้วยมือ 1 รอบ จดทุก request ที่เกิดขึ้น **รวม request ที่ frontend ยิงเบื้องหลัง** (config, polling, โหลดข้อมูลประกอบ)

กรอกตารางนี้ (เก็บเป็น `docs/checkin-flow-map.md`):

| # | Step | Method + Path | Input มาจากไหน | Output ที่ step ถัดไปต้องใช้ | Check ที่ต้องผ่าน | Think time จริง |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Login / ยืนยันตัวตน | | test data | token / session cookie | | |
| 2 | ค้นหา booking | | test data (booking ref, นามสกุล) | booking id, passenger ids | | |
| 3 | ดูรายละเอียด / เลือกผู้ใช้ที่จะ check-in | | step 2 | | | |
| 4 | กรอก/ยืนยันข้อมูลเพิ่มเติม | | | | | |
| 5 | (ถ้ามี) เลือกที่นั่ง / ตัวเลือกเสริม | | | | | |
| 6 | **Confirm check-in** | | | check-in id | | |
| 7 | ได้เอกสารยืนยัน / boarding pass | | step 6 | | | |

> ตารางนี้เป็นตัวอย่าง — แก้ step ให้ตรงกับ flow จริงของ app

คำถามที่ต้องตอบให้ได้ก่อนเขียน script:

1. **Correlation**: ค่าไหนได้จาก response แล้วต้องส่งต่อ? (token, id, CSRF token, cookie)
2. **Idempotency**: check-in ซ้ำ booking เดิมได้ไหม? ถ้าไม่ได้ → ต้องเตรียม booking ใหม่ทุก iteration
3. **Transaction นับตรงไหน**: 1 transaction = flow สำเร็จจนถึง step ไหน? (แนะนำ: ได้ check-in id / boarding pass)
4. **Request ไหน "แพง"**: step ไหนเขียน DB, เรียก downstream, generate PDF
5. **สัดส่วน traffic จริง**: ผู้ใช้ 100 คนที่เข้ามา มีกี่คนทำจนจบ? กี่คนแค่ค้นหาแล้วออก?

## 4.3 เตรียม test data

> **ทำไปเพื่อ:** check-in ส่วนใหญ่ทำซ้ำไม่ได้ ถ้าข้อมูลหมดกลาง test ผลจะเต็มไปด้วย error ที่ไม่ได้มาจากโหลด และปริมาณข้อมูลใน DB ต้องใกล้ prod ไม่อย่างนั้น query จะเร็วเกินจริง

ตัวอย่าง: breakpoint test 20 นาทีที่ 100 TPS ต้องใช้ booking ถึง ~60,000 รายการ

ทางเลือก:
- **seed script** สร้าง booking N รายการลง DB ก่อน test แล้ว export เป็น `data/bookings.json` (ไม่ commit ไฟล์นี้)
- หรือสร้าง booking ใน iteration เอง (ถ้ามี API สร้าง) แต่ **ต้องแยก metric** ไม่ให้นับรวมกับ check-in — ใช้ tag หรือ scenario แยก
- **reset DB ระหว่างรอบ** (`docker compose down -v && up`) เพื่อให้แต่ละรอบเริ่มจากสภาพเดียวกัน
- ปริมาณข้อมูลใน DB ควรใกล้ production — query ที่เร็วบน table 100 แถว อาจช้ามากบน table 10 ล้านแถว

## 4.4 เขียน `scripts/checkin/flow.js`

> **ทำไปเพื่อ:** รวม pattern จากบทที่ 2 เข้ากับ flow จริง — ได้ทั้งผลแยกทีละ step (หาคอขวด) และ counter `checkin_success` (คิด TPS)

โครงที่แนะนำ (เติม TODO ด้วยข้อมูลจากตาราง 4.2):

```js
import http from 'k6/http';
import { check, group, sleep, fail } from 'k6';
import { Counter, Rate, Trend } from 'k6/metrics';
import { SharedArray } from 'k6/data';
import exec from 'k6/execution';

export const BASE_URL = __ENV.BASE_URL || 'http://localhost:8000';
const THINK = __ENV.THINK !== '0'; // ปิด think time ได้ตอนหาเพดาน: -e THINK=0

const bookings = new SharedArray('bookings', () => JSON.parse(open('../../data/bookings.json')));

// metric หลักสำหรับคำนวณ TPS (บทที่ 6)
export const checkinSuccess = new Counter('checkin_success');
export const checkinSuccessRate = new Rate('checkin_success_rate');
export const checkinDuration = new Trend('checkin_duration', true);

const think = (min, max) => THINK && sleep(min + Math.random() * (max - min));

export function checkinFlow() {
  // booking ไม่ซ้ำกันทั้ง test
  const idx = exec.scenario.iterationInTest;
  if (idx >= bookings.length) fail(`test data หมด (ใช้ไป ${idx})`);
  const booking = bookings[idx];

  const start = Date.now();
  const ctx = {}; // เก็บค่า correlation ระหว่าง step

  const ok =
    step('01_login', () => {
      const res = http.post(`${BASE_URL}/TODO/login`, JSON.stringify({ /* TODO */ }), {
        headers: { 'Content-Type': 'application/json' },
        tags: { name: '01_login' },
      });
      ctx.token = res.json('TODO.token');
      return check(res, {
        '01_login status 200': (r) => r.status === 200,
        '01_login has token': () => !!ctx.token,
      });
    }) &&
    (think(1, 3), step('02_find_booking', () => {
      // TODO: ใช้ booking.ref และ ctx.token
      return true;
    })) &&
    // ... step 03–06 ตามตาราง 4.2
    (think(1, 2), step('07_boarding_pass', () => {
      // TODO
      return true;
    }));

  checkinSuccessRate.add(ok);
  if (ok) {
    checkinSuccess.add(1);
    checkinDuration.add(Date.now() - start);
  }
}

function step(name, fn) {
  let result = false;
  group(name, () => { result = fn(); });
  return result;
}
```

หลักการสำคัญ:
- **หยุด flow ทันทีเมื่อ step fail** (ใช้ `&&`) ไม่ให้ step ถัดไปยิงด้วยค่า `undefined`
- **tag `name` ทุก request** ให้เป็นชื่อคงที่ ห้ามมี id อยู่ในชื่อ
- **check เนื้อหา** ไม่ใช่แค่ status
- **think time** ใช้ค่าจริงจาก analytics ถ้าไม่มี ประมาณจากการทดลองทำเอง
- ถ้า app ใช้ cookie session: k6 จัดการ cookie ให้อัตโนมัติต่อ VU (cookie jar) — แต่ VU เดิมจะพก cookie ข้าม iteration ถ้าไม่อยากได้ ให้ `http.cookieJar().clear(BASE_URL)` ต้น iteration

## 4.5 Smoke test — flow ต้องผ่าน 100%

> **ทำไปเพื่อ:** ยืนยันว่า script ถูกก่อนเพิ่มโหลด — ถ้า 1 คนยังทำไม่ผ่าน error ตอนยิงหนักจะแยกไม่ออกว่ามาจาก script หรือจาก app รับไม่ไหว

`scripts/checkin/smoke.js`

```js
import { checkinFlow } from './flow.js';

export const options = {
  vus: 1,
  iterations: 5,
  thresholds: {
    checks: ['rate==1.0'],
    checkin_success_rate: ['rate==1.0'],
  },
};

export default checkinFlow;
```

```bash
k6 run --http-debug=full -e BASE_URL=http://localhost:8000 scripts/checkin/smoke.js   # ดู request/response เต็มตอน debug
k6 run -e BASE_URL=http://localhost:8000 scripts/checkin/smoke.js
```

แล้วตรวจใน DB ด้วยว่า **check-in ถูกบันทึกจริง 5 รายการ** — check ใน k6 ผ่านไม่ได้แปลว่าข้อมูลถูก

## 4.6 Load test เบื้องต้น

> **ทำไปเพื่อ:** ลองยิงที่ TPS ต่ำๆ คงที่ เพื่อเช็คว่าทั้งระบบ (script → app → Grafana) ทำงานครบ ก่อนเริ่มทดสอบจริงในบทที่ 5

`scripts/checkin/load.js` — ยิงที่ TPS คงที่ ดู Grafana

```js
import { checkinFlow } from './flow.js';

export const options = {
  scenarios: {
    checkin: {
      executor: 'constant-arrival-rate',
      rate: Number(__ENV.TPS || 5), timeUnit: '1s',
      duration: __ENV.DURATION || '10m',
      preAllocatedVUs: 50, maxVUs: 500,
    },
  },
  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<800'],          // TODO: ใช้ SLO จริงของทีม
    checkin_success_rate: ['rate>0.99'],
    checkin_duration: ['p(95)<15000'],          // ทั้ง flow รวม think time
    dropped_iterations: ['count==0'],
  },
};

export default checkinFlow;
```

```bash
k6 run -o experimental-prometheus-rw --tag testid=checkin-load-$(date +%H%M) \
  -e TPS=5 -e DURATION=5m scripts/checkin/load.js
```

## ✅ Checkpoint

- [ ] app + dependency ทั้งหมดรันด้วย docker compose ได้, external dependency เป็น mock ที่มี delay จริง
- [ ] มี `docs/checkin-flow-map.md` ครบทุก step
- [ ] มี seed script + `data/bookings.json` และวิธี reset DB
- [ ] smoke test ผ่าน 100% และยืนยันใน DB แล้ว
- [ ] load test 5 TPS 5 นาทีผ่าน thresholds และเห็นใน Grafana แยกทุก step

➡️ [บทที่ 5 — Capacity test สเปก production](05-capacity-test.md)
