# บทที่ 2 — Pattern ของ k6

บทนี้ฝึกกับ QuickPizza ทั้งหมด เก็บไฟล์ไว้ใน `scripts/patterns/`

---

## 2.1 Lifecycle ของ script

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

## 2.7 Test types — แต่ละแบบตอบคำถามต่างกัน

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

## 2.8 Test data — ผู้ใช้ไม่ซ้ำกัน

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

## 2.9 แยก module — flow เดียวใช้กับทุก test type

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

➡️ [บทที่ 3 — Monitor ด้วย Grafana](03-grafana-monitoring.md)
