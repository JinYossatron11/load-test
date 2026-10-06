# บทที่ 5 — Capacity test ด้วยสเปก production

> ### 🎯 บทนี้ทำไปเพื่ออะไร
>
> - **เป้าหมาย:** ได้ตัวเลขจริงว่า app ที่สเปกเท่า production รับ check-in ได้กี่ TPS ต่อ task และทั้ง 6 tasks และติดคอขวดที่อะไร
> - **ถ้าข้ามบทนี้:** ถ้ายิงบนเครื่องที่ไม่จำกัด CPU app จะได้ CPU ทั้งเครื่อง ตัวเลขจะสูงกว่า production หลายเท่า และตอบคำถาม "prod ไหวไหม" ไม่ได้
> - **ผลที่ได้ไปใช้ต่อ:** Max TPS / Safe TPS / scaling efficiency ที่เป็น input ของสูตรในบทที่ 6

```
แผนการทดสอบ
5.1 จำกัดทรัพยากรเท่า prod  →  5.2 Baseline 1 task  →  5.3 Breakpoint 1 task
→  5.4 ยืนยันที่ Max TPS  →  5.5 Scale เป็น 6 tasks  →  5.6 Spike + Soak
```

---

## 5.1 จำกัด CPU / RAM ให้เท่า production

> **ทำไปเพื่อ:** จำลอง 1 ECS task บนเครื่องตัวเอง (0.25 vCPU, 1 GB) เพื่อให้ app เจอข้อจำกัดเดียวกับใน production — รวมถึงการโดน OOM kill เมื่อเกิน 1 GB

Production (ECS): `cpu: 256` = **0.25 vCPU**, `memory: 1024` = **1 GB**, จำนวน **6 tasks**

แก้ service `app` ใน `docker-compose.yml`:

```yaml
  app:
    image: <checkin-app-image>
    cpus: "0.25"        # = 256 CPU units
    mem_limit: 1g       # = 1024 MB, เกินแล้วโดน OOM kill เหมือน ECS
    memswap_limit: 1g   # ห้ามใช้ swap (ECS Fargate ไม่มี swap)
    restart: unless-stopped  # ตายแล้วเริ่มใหม่ เหมือน ECS แทน task ที่ตาย (ดูบท 3.8)
    environment:
      # ใช้ env / runtime config ชุดเดียวกับ production ทุกตัว
      # สำคัญ: runtime ต้องรู้ว่าตัวเองมี CPU น้อย
      # - Java:  JAVA_TOOL_OPTIONS=-XX:ActiveProcessorCount=1 -XX:MaxRAMPercentage=75
      # - Node:  NODE_OPTIONS=--max-old-space-size=768
      # - Go:    GOMAXPROCS=1
      # - Python (gunicorn/uvicorn): จำนวน worker เท่า prod
      # - DB connection pool size เท่า prod
```

ตรวจว่าจำกัดได้จริง:

```bash
docker compose up -d app
docker stats   # คอลัมน์ CPU % ของ app ต้องไม่เกิน ~25%, MEM LIMIT = 1GiB
```

### ⚠️ ข้อควรระวังเรื่องความต่างของ CPU (ต้องอ่าน)

> **ทำไปเพื่อ:** รู้ว่าผลจาก local "ดีเกินจริง" ตรงไหนบ้าง จะได้ปรับตัวเลขก่อนเอาไปตอบเรื่อง production

0.25 vCPU บน Mac (Apple Silicon) **เร็วกว่า** 0.25 vCPU บน AWS Fargate มาก เพราะ 1 core ของ Mac แรงกว่า 1 vCPU ของ Fargate (ซึ่งเป็นแค่ 1 hyperthread)
→ TPS ที่วัดได้บน local จะ **สูงเกินจริง** ต้องปรับด้วย **calibration factor** (วิธีคิดอยู่ใน [บทที่ 6.5](06-tps-calculation.md#65-calibration--แปลงผล-local-เป็น-production))

ความต่างอื่นที่ทำให้ local ดีเกินจริง: DB อยู่เครื่องเดียวกัน (network latency ~0), ข้อมูลใน DB น้อย, ไม่มี TLS, ไม่มี ALB — จดไว้ในรายงานเป็น "ข้อจำกัดของการทดสอบ"

### เครื่องที่ยิง (k6) ต้องไม่เป็นคอขวด

> **ทำไปเพื่อ:** ถ้า k6 ยิงไม่ทันเอง ตัวเลขที่ได้จะเป็นเพดานของเครื่องเรา ไม่ใช่ของ app

ดู CPU ของ process k6 ระหว่างยิง (Activity Monitor / `top`) ถ้า > 80% ผลใช้ไม่ได้ ลดด้วย `--discard-response-bodies` (ถ้าไม่ต้องอ่าน body) หรือลด think-time-less VUs

---

## 5.2 Baseline — 1 task, โหลดต่ำ

> **ทำไปเพื่อ:** ได้ "ความเร็วดีที่สุด" ของ app ไว้เป็นจุดอ้างอิง และได้ CPU ต่อ 1 TPS ไว้ประมาณเพดานคร่าวๆ ก่อนยิงจริง

```bash
k6 run -o experimental-prometheus-rw --tag testid=baseline-1task \
  -e TPS=1 -e DURATION=5m scripts/checkin/load.js
```

จดค่า:
- p95 ของแต่ละ step ตอนไม่มีโหลด (ค่านี้คือ "ความเร็วดีที่สุด" ของ app)
- CPU ต่อ 1 TPS (เช่น 1 TPS ใช้ CPU 8% ของ limit) → ใช้ประมาณ max TPS คร่าวๆ: `100% / 8% ≈ 12 TPS`
- Memory ตอน idle และตอนมีโหลด

---

## 5.3 Breakpoint — หาเพดานของ 1 task

> **ทำไปเพื่อ:** หา Max TPS ต่อ task — จุดสุดท้ายที่ยังผ่าน SLO — และระบุว่าอะไรหมดก่อน (CPU, memory, DB) เพื่อรู้ว่าต้องแก้หรือ scale ที่ไหน

`scripts/checkin/breakpoint.js`

```js
import { checkinFlow } from './flow.js';

export const options = {
  scenarios: {
    breakpoint: {
      executor: 'ramping-arrival-rate',
      startRate: 1, timeUnit: '1s',
      preAllocatedVUs: 100, maxVUs: 2000,
      stages: [
        // ไต่ช้าๆ ให้ app มีเวลาเข้าสู่สภาวะคงที่ที่แต่ละระดับ
        { duration: __ENV.RAMP || '20m', target: Number(__ENV.MAX_TPS || 50) },
      ],
    },
  },
  thresholds: {
    // หยุดเองเมื่อพังชัดเจน ไม่ต้องรอจนจบ
    http_req_failed: [{ threshold: 'rate<0.05', abortOnFail: true, delayAbortEval: '1m' }],
    http_req_duration: [{ threshold: 'p(95)<5000', abortOnFail: true, delayAbortEval: '1m' }],
  },
};

export default checkinFlow;
```

> **เรื่อง `abortOnFail`:** threshold 2 บรรทัดนี้สั่งให้ k6 **หยุดยิงทันที** เมื่อ error สะสม > 5% หรือ p95 > 5 วินาที (`delayAbortEval: '1m'` = เริ่มเช็คหลัง test เริ่มไปแล้ว 1 นาที กันไม่ให้หยุดเพราะช่วงเริ่มต้นที่ยังไม่นิ่ง)
> - ข้อดี: ไม่เสียเวลายิงต่อหลังพังแล้ว และไม่ทำให้ DB พังตามจนต้อง reset นาน
> - ข้อเสีย: **จะไม่เห็นว่า app ฟื้นเองได้ไหม** หลังโหลดลด
> - ถ้าต้องการดูการฟื้นตัวด้วย ให้ลบ `abortOnFail` ออก แล้วเพิ่ม stage ลดโหลดตอนท้าย แบบ `scripts/death/03-ramp-recover.js` ใน [บท 3.8](03-grafana-monitoring.md#38-กรณี-app-ตาย--รู้ได้ยังไง-และดูใน-grafana-ยังไง) หรือดูจาก spike test ใน 5.6

ตั้ง `MAX_TPS` ประมาณ 2–3 เท่าของค่าที่ประมาณจาก baseline

```bash
k6 run -o experimental-prometheus-rw --tag testid=breakpoint-1task \
  -e MAX_TPS=40 -e RAMP=20m scripts/checkin/breakpoint.js
```

### อ่านผลใน Grafana

> ถ้าระหว่างไต่โหลด app ตาย / restart / ค้าง ให้ดูวิธีแยกแยะใน [บทที่ 3.8](03-grafana-monitoring.md#38-กรณี-app-ตาย--รู้ได้ยังไง-และดูใน-grafana-ยังไง) — จุดที่เริ่มตายถือเป็นจุดแตกด้วย

หาเวลา **t** แรกที่เกิดเหตุการณ์ใดเหตุการณ์หนึ่ง:
1. p95 เกิน SLO (เช่น 800ms)
2. error rate > 1%
3. TPS สำเร็จ (`checkin_success`) ไม่ตามเป้าที่ยิง (เส้นแยกออกจากกัน / `dropped_iterations` เริ่มขึ้น)

**Max TPS ต่อ task** = TPS สำเร็จจริงที่เวลา t (อ่านจาก panel TPS ไม่ใช่ค่าเป้า)
พร้อมจดว่า **คอขวดคืออะไร** (CPU ชน 100%? memory? DB? connection pool?)

> ทำซ้ำอย่างน้อย 2 รอบ (reset DB ทุกครั้ง) ถ้าผลต่างกัน > 10% ให้หาสาเหตุก่อน

---

## 5.4 ยืนยันที่ Max TPS — ยืนได้นานไหม

> **ทำไปเพื่อ:** breakpoint ไต่ขึ้นเรื่อยๆ ไม่ได้พิสูจน์ว่าระดับนั้นยืนได้นาน (queue สะสม, GC, connection pool) — ต้องยิงคงที่ยืนยันก่อนเอาตัวเลขไปใช้

breakpoint ไต่ขึ้นเรื่อยๆ ไม่ได้พิสูจน์ว่า app ยืนที่ระดับนั้นได้นาน ให้ยิงคงที่:

```bash
# ที่ Max TPS
k6 run -o experimental-prometheus-rw --tag testid=confirm-max-1task \
  -e TPS=<max> -e DURATION=15m scripts/checkin/load.js

# ที่ 80% ของ Max (ค่าที่จะเสนอเป็น Safe TPS)
k6 run -o experimental-prometheus-rw --tag testid=confirm-safe-1task \
  -e TPS=<max*0.8> -e DURATION=15m scripts/checkin/load.js
```

ผ่านเมื่อ thresholds ใน `load.js` ผ่านครบตลอด 15 นาที ถ้าไม่ผ่าน → ลด Max TPS ลงทีละ 10% แล้วทดสอบใหม่

---

## 5.5 Scale เป็น 6 tasks

> **ทำไปเพื่อ:** 6 tasks ไม่ได้รับได้ 6 เท่าเสมอ เพราะใช้ DB/cache ร่วมกัน — ต้องวัดจริงเพื่อหา scaling efficiency และดูว่าคอขวดย้ายไปอยู่ที่ไหน

production มี load balancer (ALB) กระจายไป 6 tasks — local ใช้ nginx แทน

`docker-compose.yml`

```yaml
  app:
    # ... เหมือน 5.1 แต่ลบ ports ออก (6 ตัว map port เดียวกันไม่ได้)
    deploy:
      replicas: 6

  lb:
    image: nginx:1.29
    volumes:
      - ./infra/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    ports: ["8000:80"]
    depends_on: [app]
```

`infra/nginx/nginx.conf`

```nginx
events { worker_connections 10240; }
http {
  upstream app {
    server app:8000;   # Docker DNS คืน IP ของทั้ง 6 replicas → round robin
    keepalive 64;
  }
  server {
    listen 80;
    location / {
      proxy_pass http://app;
      proxy_http_version 1.1;
      proxy_set_header Connection "";
      proxy_set_header Host $host;
    }
  }
}
```

ตรวจว่ากระจายจริง: ระหว่างยิง `docker stats` ต้องเห็น app ทั้ง 6 ตัวมี CPU ใกล้ๆ กัน

**ให้ Prometheus เห็น task ทุกตัวแยกกัน** — ถ้าใช้ `targets: ["app:8000"]` แบบเดิม Prometheus จะดึงแค่ตัวเดียว ให้เปลี่ยนเป็น DNS discovery ใน `infra/prometheus/prometheus.yml`:

```yaml
  - job_name: app
    dns_sd_configs:
      - names: ["app"]   # Docker DNS คืน IP ของทั้ง 6 replicas
        type: A
        port: 8000
```

จากนั้น panel **App up** จะมี 6 เส้น (แยกตาม `instance`) — ถ้าเส้นใดเป็น 0 คือ task นั้นตาย/ค้าง ส่วน k6 จะเห็นเป็น `status=502/504` จาก nginx (แบบ 5 ใน [บท 3.8](03-grafana-monitoring.md#38-กรณี-app-ตาย--รู้ได้ยังไง-และดูใน-grafana-ยังไง))
นับจำนวน task ที่ยังรอดได้ด้วย `sum(up{job="app"})` — ใช้คู่กับกราฟ TPS ดูว่าเหลือ 5 tasks แล้ว TPS ตกไปเท่าไร (ตรงกับกรณี N-1 ในบทที่ 6.4)

> - Docker Desktop ต้องมี CPU มากพอ: app 6 × 0.25 = 1.5 cores + DB + nginx + Prometheus + **k6** — แนะนำให้ Docker ≥ 6 cores
> - DB ตอนนี้ถูก 6 tasks ใช้ร่วมกัน — ถ้า prod DB มีสเปกจำกัด ให้จำกัด `cpus`/`mem_limit` ของ DB ให้ใกล้เคียง prod ด้วย ไม่อย่างนั้นจะไม่เห็นคอขวดที่ DB

รัน breakpoint อีกครั้ง โดยตั้ง `MAX_TPS` ≈ 6 × Max TPS ต่อ task × 1.5

```bash
k6 run -o experimental-prometheus-rw --tag testid=breakpoint-6task \
  -e MAX_TPS=<...> -e RAMP=20m scripts/checkin/breakpoint.js
```

แล้วยืนยันด้วย 5.4 อีกรอบที่ระดับ 6 tasks

คำนวณ **scaling efficiency** (สูตรในบทที่ 6.4) — ถ้าต่ำกว่า ~80% แปลว่ามีคอขวดที่ใช้ร่วมกัน (DB, cache, lock) ไม่ใช่ที่ app

---

## 5.6 Spike และ Soak (ที่ 6 tasks)

> **ทำไปเพื่อ:** ตอบความเสี่ยงที่ breakpoint ไม่ครอบคลุม: คนเข้าพร้อมกันฉับพลันแล้วระบบล้มไหม/ฟื้นไหม และรันนานๆ แล้ว memory รั่วจนโดน kill ไหม

**Spike** — เช่น ช่วงเปิดให้ check-in หรือ batch notification ทำให้คนเข้าพร้อมกัน

```js
// scripts/checkin/spike.js
scenarios: {
  spike: {
    executor: 'ramping-arrival-rate',
    startRate: <ปกติ>, timeUnit: '1s', preAllocatedVUs: 200, maxVUs: 3000,
    stages: [
      { duration: '2m',  target: <ปกติ> },
      { duration: '30s', target: <peak × 2> },
      { duration: '3m',  target: <peak × 2> },
      { duration: '30s', target: <ปกติ> },
      { duration: '3m',  target: <ปกติ> },   // ฟื้นตัวไหม
    ],
  },
}
```

ดู: error ระหว่าง spike, p95 กลับมาปกติภายในกี่วินาทีหลัง spike, มี task ตาย / restart / ค้างไหม (panel App up, Restarts, Memory และ `docker events` — วิธีอ่านอยู่ใน [บท 3.8](03-grafana-monitoring.md#38-กรณี-app-ตาย--รู้ได้ยังไง-และดูใน-grafana-ยังไง))

ถ้ามี task ตายระหว่าง spike ให้ดูต่อว่า:
- task ที่ตายตายเพราะอะไร (OOM? ค้าง?) — memory พุ่งชน 1 GB ตอน spike หรือเปล่า
- task ที่เหลือรับโหลดต่อไหวไหม หรือ **ตายต่อกันเป็นทอดๆ** (cascading failure: ตาย 1 ตัว → โหลดไปลงที่เหลือ → ตายตาม)
- หลัง spike จบ ทุก task กลับมา `up = 1` และ error กลับเป็น 0 ภายในกี่วินาที

**Soak** — ยิงที่ Safe TPS นาน 1–2 ชั่วโมง ดู memory ของแต่ละ task ว่าไต่ขึ้นเรื่อยๆ ไหม (1 GB ถ้ารั่วจะโดน kill)

---

## ตารางผลที่ต้องได้จากบทนี้

> **ทำไปเพื่อ:** รวมหลักฐานทุก test ไว้ที่เดียว ใช้เป็น input ของบทที่ 6 และแนบในรายงาน

| Test | testid | Target TPS | TPS สำเร็จ | p95 (ms) | Error % | CPU/task | Mem/task | ตาย/restart (กี่ครั้ง, แบบไหน, ที่กี่ TPS) | คอขวด | ผ่าน? |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Baseline 1 task | | 1 | | | | | | – | | |
| Breakpoint 1 task | | ramp | **Max =** | | | | | | | |
| Confirm max 1 task | | | | | | | | | | |
| Confirm safe 1 task | | | | | | | | | | |
| Breakpoint 6 tasks | | ramp | **Max =** | | | | | | | |
| Confirm 6 tasks | | | | | | | | | | |
| Spike 6 tasks | | | | | | | | | | |
| Soak 6 tasks | | | | | | | | | | |

## ✅ Checkpoint

- [ ] `docker stats` ยืนยันว่า app ถูกจำกัดที่ 0.25 CPU / 1 GB
- [ ] ได้ Max TPS ต่อ task พร้อมระบุคอขวด (ทำซ้ำ 2 รอบ ผลใกล้กัน)
- [ ] ยืนยัน Safe TPS ต่อ task 15 นาทีผ่าน
- [ ] ได้ Max TPS ของ 6 tasks และ scaling efficiency
- [ ] spike / soak ไม่มี OOM หรือ restart
- [ ] ตารางผลกรอกครบ

➡️ [บทที่ 6 — การคำนวณ TPS](06-tps-calculation.md)
