# บทที่ 2 — Pattern ของ k6

> ### 🎯 บทนี้ทำไปเพื่ออะไร
>
> - **เป้าหมาย:** เขียน script ที่ "เชื่อถือได้" — ตรวจผลถูก วัดแยกทีละ step ปล่อยโหลดได้ตามรูปแบบที่ต้องการ และใช้ซ้ำได้
> - **ถ้าข้ามบทนี้:** script จะยิงได้แต่ผลไม่น่าเชื่อ: นับ request ที่ error เป็นความสำเร็จ, ไม่รู้ว่า step ไหนช้า, ใช้ executor ผิดแบบจนได้ TPS สูงเกินจริง
> - **ผลที่ได้ไปใช้ต่อ:** ทุก pattern ในบทนี้ถูกใช้ตรงๆ ใน `scripts/checkin/` บทที่ 4–5

บทนี้ฝึกกับ QuickPizza ทั้งหมด เก็บไฟล์ไว้ใน `scripts/patterns/`

---

## 2.1 Lifecycle ของ script

> **ทำไปเพื่อ:** รู้ว่าโค้ดส่วนไหนถูกวัดผล (default) และส่วนไหนไม่ถูกวัด (init/setup/teardown) — การเตรียมข้อมูลต้องไม่ปนเข้ามาในตัวเลข TPS

```js
// 1) init — รัน 1 ครั้งต่อ VU: import, อ่านไฟล์, ประกาศ metric (ยิง HTTP ไม่ได้)
import http from 'k6/http';

export const options = { /* ... */ };

// 2) setup — รัน 1 ครั้งก่อนเริ่มทั้ง test: เตรียมข้อมูล, login admin
export function setup() {
  return { startedAt: Date.now() }; // ค่าที่ return ส่งให้ default/teardown
}

// 3) default (VU code) — รันวนซ้ำ = ส่วนที่วัดผล
export default function (data) { }

// 4) teardown — รัน 1 ครั้งตอนจบ: ลบข้อมูลที่สร้างระหว่าง test
export function teardown(data) { }
```

---

## 2.2 Checks — ตรวจว่า response ถูก

> **ทำไปเพื่อ:** ยืนยันว่า app ทำงาน "ถูก" ไม่ใช่แค่ "ตอบกลับมา" — app ที่ตอบเร็วแต่ตอบ error ไม่ได้แปลว่ารับโหลดไหว

ไม่ใช่แค่ status 200 — ต้องเช็ค **เนื้อหา** ด้วย เพราะบาง app ตอบ 200 พร้อม error message

```js
import { check } from 'k6';

const res = http.post(`${BASE_URL}/api/users/token/login`, JSON.stringify({ username, password }));
check(res, {
  'login: status 200': (r) => r.status === 200,
  'login: has token': (r) => r.json('token') !== undefined,
});
```

> check ที่ fail **ไม่ทำให้ test หยุด** และไม่ทำให้ exit code ผิด — ต้องใช้คู่กับ thresholds

---

## 2.3 Thresholds — เกณฑ์ผ่าน/ไม่ผ่าน (SLO)

> **ทำไปเพื่อ:** แปลง "ช้าแค่ไหนถึงรับไม่ได้" ให้เป็นตัวเลขชัดเจน — Max TPS ในบทที่ 5 นิยามจาก threshold เหล่านี้ และให้ k6 ตัดสินผ่าน/ไม่ผ่านให้อัตโนมัติ

```js
export const options = {
  thresholds: {
    http_req_failed: ['rate<0.01'],                    // error < 1%
    http_req_duration: ['p(95)<800', 'p(99)<1500'],    // ms
    checks: ['rate>0.99'],
    // threshold เฉพาะ request ที่ tag name ตรงกัน
    'http_req_duration{name:login}': ['p(95)<500'],
    // หยุด test ทันทีถ้าพังหนัก (ประหยัดเวลาตอนหา breakpoint)
    'http_req_failed{scenario:default}': [{ threshold: 'rate<0.1', abortOnFail: true, delayAbortEval: '30s' }],
  },
};
```

ถ้า threshold ไม่ผ่าน → k6 exit code ≠ 0 → ใช้ใน CI ได้

---

## 2.4 Groups และ Tags — แยกผลทีละ step

> **ทำไปเพื่อ:** เมื่อ flow ช้า ต้องรู้ว่า **step ไหน** ช้า — ไม่อย่างนั้นได้แค่รู้ว่า "ระบบช้า" แต่ไม่รู้ว่าต้องแก้ตรงไหน

```js
import { group } from 'k6';

group('01_login', () => {
  http.post(url, body, { tags: { name: 'login' } });
});
```

- **group** → ได้ metric `group_duration` ของแต่ละขั้น (เวลารวมของ step)
- **tag `name`** → ใช้ตั้งชื่อ request ให้คงที่ สำคัญมากเมื่อ URL มี id เช่น `/api/booking/123`, `/api/booking/456` ถ้าไม่ตั้ง `name` Grafana จะแตกเป็นหลายพันเส้น

```js
http.get(`${BASE_URL}/api/booking/${id}`, { tags: { name: 'GET /api/booking/:id' } });
```

---

## 2.5 Custom metrics

> **ทำไปเพื่อ:** k6 นับ "request" ให้อัตโนมัติ แต่ไม่รู้ว่า "check-in สำเร็จ 1 ครั้ง" คืออะไร — ต้องนับเองเพื่อให้ได้ TPS ในความหมายของ business

| ชนิด | ใช้กับ | ตัวอย่าง |
| --- | --- | --- |
| `Counter` | นับสะสม | จำนวน check-in สำเร็จ (ใช้คิด TPS) |
| `Rate` | สัดส่วน true/false | % check-in สำเร็จ |
| `Trend` | ค่าที่มีการกระจาย (p95) | เวลาทั้ง flow ตั้งแต่ต้นจนจบ |
| `Gauge` | ค่าล่าสุด | ขนาด queue |

```js
import { Counter, Rate, Trend } from 'k6/metrics';

const txnSuccess = new Counter('txn_success');
const txnSuccessRate = new Rate('txn_success_rate');
const txnDuration = new Trend('txn_duration', true); // true = หน่วยเวลา

export default function () {
  const start = Date.now();
  const ok = doFlow();       // return true ถ้าทุก step ผ่าน
  txnSuccessRate.add(ok);
  if (ok) {
    txnSuccess.add(1);
    txnDuration.add(Date.now() - start);
  }
}
```

> `txn_success` คือตัวที่จะใช้คำนวณ **TPS จริง** ในบทที่ 6 — `iterations` นับรวมรอบที่ fail ด้วย

---

## 2.6 Scenarios และ Executors — รูปแบบการปล่อยโหลด

> **ทำไปเพื่อ:** เลือกรูปแบบโหลดให้ตรงกับคำถาม — การหา capacity ต้องยิงที่อัตราคงที่ (open model) ไม่อย่างนั้นเมื่อ server ช้า k6 จะยิงช้าลงตามและได้ผลดีเกินจริง

มี 2 แนวคิดหลัก:

| แบบ | Executor | ควบคุม | ใช้เมื่อ |
| --- | --- | --- | --- |
| **Closed model** | `constant-vus`, `ramping-vus` | จำนวน VU | จำลอง "ผู้ใช้ N คนพร้อมกัน" |
| **Open model** | `constant-arrival-rate`, `ramping-arrival-rate` | จำนวน iteration ที่ **เริ่ม** ต่อวินาที | ต้องการยิงที่ TPS เป้าหมาย / หา capacity |

ข้อต่างสำคัญ: ใน closed model ถ้า server ช้าลง VU ก็ยิงช้าลงเอง → โหลดลดลงโดยอัตโนมัติ ทำให้ผลดูดีเกินจริง (coordinated omission)
ใน open model k6 จะเริ่ม iteration ใหม่ตามอัตราที่กำหนดเสมอ เหมือนผู้ใช้จริงที่ไม่สนว่า server ช้า
**→ การหา TPS ใช้ open model (arrival-rate) เสมอ**

```js
export const options = {
  scenarios: {
    checkin_load: {
      executor: 'constant-arrival-rate',
      rate: 20,              // 20 iterations
      timeUnit: '1s',        // ต่อวินาที = 20 TPS เป้าหมาย
      duration: '5m',
      preAllocatedVUs: 50,   // VU ที่เตรียมไว้
      maxVUs: 200,           // ถ้าไม่พอ k6 เพิ่มได้ถึงเท่านี้
    },
  },
};
```

ถ้า VU ไม่พอจะยิงตาม rate ไม่ทัน → k6 รายงาน metric **`dropped_iterations`** ถ้าค่านี้ > 0 แปลว่ายิงไม่ถึงเป้า (เพิ่ม `maxVUs` หรือ server ช้าจน VU ไม่ว่าง)

สามารถรันหลาย scenario พร้อมกันได้ เช่น 80% check-in + 20% ดูสถานะ (`exec: 'functionName'` เพื่อเลือก function)

---

## 2.7 ตัวอย่าง: ยิงพร้อมกัน vs ทยอยยิง

> **ทำไปเพื่อ:** ผู้ใช้จริงเข้ามาได้ 2 แบบ คือ **แห่เข้ามาพร้อมกัน** (เปิดให้ check-in, ส่ง notification, หมดเวลาโปรโมชัน) กับ **ทยอยเพิ่มขึ้นเรื่อยๆ** (ช่วงเช้าที่คนค่อยๆ เข้า) — app ตัวเดียวกันรับ 2 แบบนี้ได้ไม่เท่ากัน ต้องทดสอบทั้งสองแบบ

ตัวอย่างทั้งหมดใช้ request เดียวกัน ให้สร้าง module กลางไว้ก่อน — `scripts/patterns/pizza.js`

```js
import http from 'k6/http';
import { check } from 'k6';

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3333';

export function getPizza() {
  const res = http.post(`${BASE_URL}/api/pizza`, JSON.stringify({
    maxCaloriesPerSlice: 1000, mustBeVegetarian: false, excludedIngredients: [],
    excludedTools: [], maxNumberOfToppings: 5, minNumberOfToppings: 2,
  }), {
    headers: { Authorization: 'token abcdef0123456789' }, // token ตัวอย่างของ QuickPizza
    tags: { name: 'get_pizza' },
  });
  check(res, { 'pizza 200': (r) => r.status === 200 });
}
```

### แบบ A — ยิงพร้อมกันทีเดียว (burst)

`scripts/patterns/burst-once.js` — 100 คนกดพร้อมกันในวินาทีเดียว

```js
import { getPizza } from './pizza.js';

export const options = {
  scenarios: {
    burst: {
      executor: 'per-vu-iterations',
      vus: 100,        // ผู้ใช้ 100 คน
      iterations: 1,   // คนละ 1 ครั้ง → 100 request ออกพร้อมกันทันที
      maxDuration: '30s',
    },
  },
};

export default getPizza;
```

ใช้จำลอง: วินาทีที่เปิดให้ check-in หรือส่ง push notification แล้วคนกดเข้ามาพร้อมกัน

### แบบ B — ยิงพร้อมกันเป็นระลอก (waves)

`scripts/patterns/burst-waves.js` — 50 คนยิงพร้อมกันทุกๆ 5 วินาที

```js
import { sleep } from 'k6';
import { getPizza } from './pizza.js';

const WAVE_EVERY = 5; // ยิงพร้อมกันทุกๆ 5 วินาที

export const options = {
  scenarios: {
    waves: { executor: 'constant-vus', vus: 50, duration: '30s' },
  },
};

export default function () {
  // รอจนถึงวินาทีที่หาร 5 ลงตัวของนาฬิกา → ทุก VU ตื่นพร้อมกัน
  const now = Date.now() / 1000;
  sleep(WAVE_EVERY - (now % WAVE_EVERY));
  getPizza();
}
```

ใช้จำลอง: ระบบที่มี client polling พร้อมกัน, cron/batch ที่ยิงเข้ามาตรงเวลา, หลายเครื่อง kiosk retry พร้อมกัน

### แบบ C — กระโดดขึ้นฉับพลันแล้วค้างไว้ (spike)

`scripts/patterns/spike.js` — จาก 5 → 150 request/s ภายใน 5 วินาที

```js
import { getPizza } from './pizza.js';

export const options = {
  scenarios: {
    spike: {
      executor: 'ramping-arrival-rate',
      startRate: 5, timeUnit: '1s',
      preAllocatedVUs: 50, maxVUs: 300,
      stages: [
        { duration: '20s', target: 5 },    // ปกติ 5 req/s
        { duration: '5s',  target: 150 },  // กระโดดเป็น 150 req/s ใน 5 วินาที
        { duration: '20s', target: 150 },  // ค้างไว้
        { duration: '5s',  target: 5 },    // ลดกลับ
        { duration: '20s', target: 5 },    // ดูว่าฟื้นตัวไหม
      ],
    },
  },
};

export default getPizza;
```

ต่างจากแบบ A ตรงที่ไม่ได้ยิงครั้งเดียวจบ แต่ **ค้างโหลดสูงไว้** — ดูว่า app ยืนได้ไหม และหลังโหลดลดแล้ว p95 กลับมาปกติเร็วแค่ไหน

### แบบ D — ทยอยเพิ่มคน (ramp-up, closed model)

`scripts/patterns/ramp-vus.js` — เพิ่มคนจาก 0 → 50 ใน 1 นาที

```js
import { sleep } from 'k6';
import { getPizza } from './pizza.js';

export const options = {
  scenarios: {
    ramp: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 50 },  // ทยอยเพิ่มคนจาก 0 → 50 ใน 1 นาที
        { duration: '2m', target: 50 },  // ค้างไว้ 50 คน
        { duration: '30s', target: 0 },  // ทยอยออก
      ],
      gracefulRampDown: '10s',
    },
  },
};

export default function () {
  getPizza();
  sleep(1);
}
```

ใช้จำลอง: จำนวนผู้ใช้ที่ค่อยๆ เพิ่มขึ้นตามช่วงเวลา และใช้ warm-up ให้ app (cache, JIT, connection pool) พร้อมก่อนวัดผล

### แบบ E — ทยอยเพิ่มเป็นขั้นบันได (step load, open model)

`scripts/patterns/ramp-steps.js` — เพิ่มทีละ 20 req/s แล้วค้างแต่ละขั้น 1 นาที

```js
import { getPizza } from './pizza.js';

// ขั้นบันได: เพิ่มทีละ STEP req/s แล้วค้างแต่ละขั้นนาน HOLD
const STEP = Number(__ENV.STEP || 20);
const LEVELS = Number(__ENV.LEVELS || 5);
const HOLD = __ENV.HOLD || '1m';

const stages = [];
for (let i = 1; i <= LEVELS; i++) {
  stages.push({ duration: '10s', target: STEP * i }); // ขึ้นขั้น
  stages.push({ duration: HOLD, target: STEP * i });  // ค้าง
}

export const options = {
  scenarios: {
    steps: {
      executor: 'ramping-arrival-rate',
      startRate: 0, timeUnit: '1s',
      preAllocatedVUs: 50, maxVUs: 500,
      stages,
    },
  },
};

export default getPizza;
```

```bash
k6 run -e STEP=20 -e LEVELS=5 -e HOLD=1m scripts/patterns/ramp-steps.js
```

**แบบนี้เหมาะที่สุดสำหรับหา capacity** — แต่ละขั้นค้างไว้นานพอให้ระบบนิ่ง จึงอ่านค่าใน Grafana ได้ชัดว่า "ขั้นไหนยังผ่าน SLO ขั้นไหนเริ่มไม่ผ่าน" (ไต่ขึ้นแบบเส้นตรงยาวๆ อย่างในบทที่ 5.3 ก็ได้ แต่ขั้นบันไดอ่านง่ายกว่า)

### แบบ F — หลายกลุ่มเริ่มต่างเวลา (staggered)

`scripts/patterns/staggered.js` — ผสมทั้งทยอยและพร้อมกันใน test เดียว

```js
import { sleep } from 'k6';
import { getPizza } from './pizza.js';

export const options = {
  scenarios: {
    // กลุ่มแรก: ผู้ใช้ปกติ เริ่มทันที
    normal_users: {
      executor: 'constant-vus', vus: 10, duration: '2m',
      exec: 'browse',
    },
    // กลุ่มที่สอง: เข้ามาทีหลัง 30 วินาที แบบทยอยเพิ่ม
    late_comers: {
      executor: 'ramping-vus', startTime: '30s', startVUs: 0,
      stages: [{ duration: '30s', target: 30 }, { duration: '30s', target: 30 }],
      exec: 'browse',
    },
    // กลุ่มที่สาม: แห่เข้ามาพร้อมกันที่วินาทีที่ 90
    flash_crowd: {
      executor: 'per-vu-iterations', startTime: '90s', vus: 100, iterations: 1,
      exec: 'once',
    },
  },
};

export function browse() { getPizza(); sleep(1); }
export function once() { getPizza(); }
```

ใช้จำลอง: วันจริงที่มีผู้ใช้ปกติอยู่แล้ว แล้วมีคลื่นคนแห่เข้ามาซ้อน — ดูว่าคนที่ใช้งานอยู่ก่อน (`normal_users`) ช้าลงไหมตอนที่ `flash_crowd` เข้ามา
(ใน Grafana แยกดูแต่ละกลุ่มด้วย `by (scenario)`)

### เปรียบเทียบ

| แบบ | Executor | รูปกราฟโหลด | ตอบคำถาม |
| --- | --- | --- | --- |
| A ยิงพร้อมกันทีเดียว | `per-vu-iterations` | แท่งเดียวสูงๆ | คนกดพร้อมกัน N คน app ตอบทันไหม error ไหม |
| B ระลอก | `constant-vus` + sleep ตามนาฬิกา | ฟันเลื่อย | รับแรงกระแทกซ้ำๆ ได้ไหม ระหว่างระลอกฟื้นตัวทันไหม |
| C Spike | `ramping-arrival-rate` (ขึ้นชัน) | ขั้นกระโดด | โหลดพุ่งฉับพลันแล้วค้าง ยืนได้ไหม ฟื้นไหม |
| D ทยอยเพิ่มคน | `ramping-vus` | ทางลาด | ผู้ใช้ N คนพร้อมกันเป็นยังไง (closed model) |
| E ขั้นบันได | `ramping-arrival-rate` (ทีละขั้น) | บันได | **TPS สูงสุดที่ยังผ่าน SLO** (ใช้หา capacity) |
| F หลายกลุ่ม | หลาย scenario + `startTime` | ผสม | traffic ผสมแบบวันจริง กลุ่มหนึ่งกระทบอีกกลุ่มไหม |

### โจทย์ 2.7

รันแบบ A และแบบ C (เปิด Grafana ไว้ถ้าทำบทที่ 3 แล้ว) แล้วเทียบ p95

<details>
<summary>ผลที่ได้จากการลองรันบน MacBook (QuickPizza ไม่จำกัด CPU)</summary>

| แบบ | จำนวน request | อัตรา | p95 |
| --- | --- | --- | --- |
| A — 100 คนพร้อมกันทีเดียว | 100 | ~88 req/s | **1.11 s** |
| B — 50 คนพร้อมกันทุก 5 วินาที | 350 | ~10 req/s | 187 ms |
| C — spike ถึง 150 req/s แบบไต่ขึ้นใน 5 วินาที | 3,974 | ~57 req/s | 138 ms |

ข้อสังเกต: แบบ C ยิงถึง 150 req/s แต่ p95 ยังต่ำ ส่วนแบบ A มีแค่ 100 request แต่ p95 สูงกว่าเกือบ 10 เท่า
เพราะ request **มาถึงในเสี้ยววินาทีเดียวกัน** ต้องต่อคิวกัน (connection ใหม่ 100 เส้นพร้อมกัน, worker ไม่พอ)
→ **ตัวเลข TPS เฉลี่ยอย่างเดียวไม่พอ** ต้องทดสอบแบบพร้อมกันด้วย ถ้า check-in มีช่วงที่คนแห่เข้ามาพร้อมกัน
</details>

---

## 2.8 Test types — แต่ละแบบตอบคำถามต่างกัน

> **ทำไปเพื่อ:** รู้ว่าคำถามแต่ละข้อ ("ปกติไหวไหม", "สูงสุดเท่าไร", "คนเข้าพร้อมกันไหวไหม", "รันนานแล้ว memory รั่วไหม") ต้องใช้ test แบบไหน — บทที่ 5 ใช้ครบทุกแบบ

| Type | รูปโหลด | ตอบคำถาม | ระยะเวลา |
| --- | --- | --- | --- |
| **Smoke** | 1–2 VU | script ถูกไหม app ทำงานไหม | 1 นาที |
| **Average load** | ระดับปกติ คงที่ | ช่วงปกติ SLO ผ่านไหม | 10–30 นาที |
| **Stress** | สูงกว่าปกติ (เช่น peak × 1.5) | ช่วง peak เป็นยังไง | 10–30 นาที |
| **Spike** | กระโดดขึ้นทันที | รับคลื่นฉับพลัน (เปิด check-in) ไหว ฟื้นตัวไหม | 5–10 นาที |
| **Soak** | ปกติ แต่ยาวมาก | memory leak, connection leak | 1–8 ชม. |
| **Breakpoint** | เพิ่มเรื่อยๆ จนพัง | **รับได้สูงสุดเท่าไร** | จนกว่าจะพัง |

ตัวอย่าง stages:

```js
// Stress (closed model)
stages: [
  { duration: '2m', target: 50 },
  { duration: '5m', target: 50 },
  { duration: '2m', target: 100 },
  { duration: '5m', target: 100 },
  { duration: '2m', target: 0 },
]

// Spike
stages: [
  { duration: '30s', target: 10 },
  { duration: '10s', target: 300 }, // กระโดด
  { duration: '2m',  target: 300 },
  { duration: '10s', target: 10 },  // ดูการฟื้นตัว
  { duration: '2m',  target: 10 },
]

// Breakpoint (open model) — ใช้ในบทที่ 5
scenarios: {
  breakpoint: {
    executor: 'ramping-arrival-rate',
    startRate: 1, timeUnit: '1s',
    preAllocatedVUs: 50, maxVUs: 1000,
    stages: [{ duration: '20m', target: 200 }], // ไต่ขึ้นช้าๆ ถึง 200 TPS
  },
}
```

---

## 2.9 Test data — ผู้ใช้ไม่ซ้ำกัน

> **ทำไปเพื่อ:** ระบบจริงไม่ได้มีผู้ใช้คนเดียว — ถ้าใช้ข้อมูลซ้ำ cache จะช่วยจนผลดีเกินจริง และ check-in ซ้ำ booking เดิมจะ error

ใช้ `SharedArray` โหลดไฟล์ครั้งเดียวแชร์ทุก VU (ประหยัด memory)

```js
import { SharedArray } from 'k6/data';
import exec from 'k6/execution';

const users = new SharedArray('users', () => JSON.parse(open('../../data/users.json')));

export default function () {
  // ไม่ซ้ำกันทั้ง test (เหมาะกับ check-in ที่ทำซ้ำไม่ได้)
  const user = users[exec.scenario.iterationInTest % users.length];
}
```

| วิธีเลือก | ใช้เมื่อ |
| --- | --- |
| `users[exec.scenario.iterationInTest]` | ข้อมูลใช้ได้ครั้งเดียว (booking check-in แล้วซ้ำไม่ได้) |
| `users[exec.vu.idInTest - 1]` | 1 VU = 1 user ตลอด test |
| `users[Math.floor(Math.random() * users.length)]` | ข้อมูลใช้ซ้ำได้ |

CSV ใช้ `papaparse` จาก jslib: `import papaparse from 'https://jslib.k6.io/papaparse/5.1.1/index.js';`

---

## 2.10 แยก module — flow เดียวใช้กับทุก test type

> **ทำไปเพื่อ:** เขียน flow ครั้งเดียว ใช้ได้กับ smoke/load/breakpoint/spike — เวลา flow เปลี่ยนแก้ที่เดียว และมั่นใจว่าทุก test วัด flow เดียวกัน

```
scripts/checkin/
├── flow.js        # export function checkinFlow() — ขั้นตอนทั้งหมด
├── smoke.js       # import flow + options แบบ smoke
├── load.js        # import flow + options แบบ load
└── breakpoint.js  # import flow + options แบบ breakpoint
```

```js
// smoke.js
import { checkinFlow } from './flow.js';
export const options = { vus: 1, iterations: 5, thresholds: { checks: ['rate==1'] } };
export default checkinFlow;
```

---

## โจทย์ 2 — QuickPizza flow ครบ

> **ทำไปเพื่อ:** ฝึกรวมทุก pattern ในบทนี้บน app ที่ไม่มีความเสี่ยง — โจทย์นี้คือ "ต้นแบบ" ของ `flow.js` ในบทที่ 4

เขียน `scripts/patterns/pizza-flow.js` ที่:

1. `setup()` — ไม่ต้องทำอะไร หรือสร้าง user ล่วงหน้า
2. แต่ละ iteration: สมัคร user ใหม่ (username ไม่ซ้ำ) → login → ขอพิซซ่า → ให้คะแนนพิซซ่าที่ได้
3. ทุก step อยู่ใน `group` และมี tag `name`
4. มี `check` ทุก step (status + เนื้อหา)
5. custom metric: `Counter txn_success`, `Trend txn_duration`
6. thresholds: error < 1%, p95 < 500ms, checks > 99%
7. ใช้ `constant-arrival-rate` ที่ 10 iterations/s นาน 1 นาที
8. มี `sleep` ระหว่าง step 0.5–1.5 วินาทีแบบสุ่ม (think time)

<details>
<summary>เฉลย</summary>

```js
import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { Counter, Trend } from 'k6/metrics';
import exec from 'k6/execution';

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3333';
const txnSuccess = new Counter('txn_success');
const txnDuration = new Trend('txn_duration', true);

export const options = {
  scenarios: {
    pizza: {
      executor: 'constant-arrival-rate',
      rate: 10, timeUnit: '1s', duration: '1m',
      // Little's Law: 10 TPS × ~3.2s ต่อรอบ (request + think time) ≈ 32 VUs → เผื่อเป็น 40
      preAllocatedVUs: 40, maxVUs: 100,
    },
  },
  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<500'],
    checks: ['rate>0.99'],
  },
};

const think = () => sleep(0.5 + Math.random());

export default function () {
  const start = Date.now();
  const username = `u_${exec.vu.idInTest}_${exec.scenario.iterationInTest}_${Date.now()}`;
  const password = 'secret12';
  let token, pizzaId;

  const ok = group('01_register', () => {
    const r = http.post(`${BASE_URL}/api/users`, JSON.stringify({ username, password }), { tags: { name: 'register' } });
    return check(r, { 'register 201': (r) => r.status === 201 });
  })
  && (think(), group('02_login', () => {
    const r = http.post(`${BASE_URL}/api/users/token/login`, JSON.stringify({ username, password }), { tags: { name: 'login' } });
    token = r.json('token');
    return check(r, { 'login 200': (r) => r.status === 200, 'login has token': () => !!token });
  }))
  && (think(), group('03_get_pizza', () => {
    const r = http.post(`${BASE_URL}/api/pizza`, JSON.stringify({
      maxCaloriesPerSlice: 1000, mustBeVegetarian: false, excludedIngredients: [],
      excludedTools: [], maxNumberOfToppings: 5, minNumberOfToppings: 2,
    }), { headers: { Authorization: `token ${token}` }, tags: { name: 'get_pizza' } });
    pizzaId = r.json('pizza.id');
    return check(r, { 'pizza 200': (r) => r.status === 200, 'pizza has id': () => !!pizzaId });
  }))
  && (think(), group('04_rate', () => {
    const r = http.post(`${BASE_URL}/api/ratings`, JSON.stringify({ pizza_id: pizzaId, stars: 5 }),
      { headers: { Authorization: `token ${token}` }, tags: { name: 'rate_pizza' } });
    return check(r, { 'rate 201': (r) => r.status === 201 });
  }));

  if (ok) {
    txnSuccess.add(1);
    txnDuration.add(Date.now() - start);
  }
}
```

จุดสังเกต:
- ถ้าตั้ง `preAllocatedVUs: 30` จะเห็น `dropped_iterations` เล็กน้อย เพราะ 1 รอบใช้ ~3.2s (think time 3 ครั้ง) ที่ 10 iterations/s ต้องมี VU ว่างอย่างน้อย ~32 ตัว — สูตรนี้อยู่ในบทที่ 6.3
- ใช้ `&&` ต่อ step เพื่อ **หยุด flow ทันทีเมื่อ step ใด fail** — ไม่อย่างนั้น step ถัดไปจะยิงด้วย token เป็น `undefined` แล้วได้ 401 ปนเข้ามาใน error rate
</details>

## ✅ Checkpoint

- [ ] โจทย์ 2 ผ่าน thresholds ทั้งหมด
- [ ] อธิบายได้ว่าทำไมหา capacity ต้องใช้ arrival-rate ไม่ใช่ ramping-vus
- [ ] รู้ว่า `dropped_iterations` > 0 หมายความว่าอะไร
- [ ] ดัดแปลงโจทย์ 2 เป็น smoke / stress / spike ได้ (เปลี่ยนแค่ options)
- [ ] รันตัวอย่างยิงพร้อมกัน (A) และทยอยยิง (E) แล้วอธิบายได้ว่าทำไม p95 ต่างกัน

➡️ [บทที่ 3 — Monitor ด้วย Grafana](03-grafana-monitoring.md)
