# บทที่ 3 — Monitor ด้วย Grafana

> ### 🎯 บทนี้ทำไปเพื่ออะไร
>
> - **เป้าหมาย:** เห็นผลแบบ realtime ตามเวลา และเห็นทั้งฝั่งผู้ใช้ (k6) กับฝั่ง server (CPU/memory) บนกราฟเดียวกัน
> - **ถ้าข้ามบทนี้:** จะรู้แค่ผลรวมตอนจบ แต่ไม่รู้ว่า "เริ่มพังที่กี่ TPS" และ "พังเพราะอะไร" — ซึ่งคือ 2 คำถามหลักของการหา capacity
> - **ผลที่ได้ไปใช้ต่อ:** dashboard "Load Test — Capacity" ที่ใช้อ่าน Max TPS และคอขวดในบทที่ 5

summary ตอนจบบอกแค่ภาพรวม แต่ไม่บอกว่า **"เริ่มพังตอนไหน ที่โหลดเท่าไร"** — ต้องดูเป็นกราฟตามเวลา

```
┌──────┐ remote write  ┌────────────┐  query  ┌─────────┐
│  k6  │ ────────────▶ │ Prometheus │ ◀────── │ Grafana │
└──────┘               └────────────┘         └─────────┘
                              ▲ scrape /metrics
                       ┌──────┴─────┐
                       │    App     │  (ฝั่ง server: CPU, latency, request count)
                       └────────────┘
```

ต้องเห็น **ทั้งสองฝั่ง**: ฝั่ง k6 (ผู้ใช้เห็นอะไร) และฝั่ง app (server ทำงานหนักแค่ไหน)

---

## 3.1 เขียน docker-compose

> **ทำไปเพื่อ:** ได้ระบบ monitor ที่ทุกคนในทีมรันได้เหมือนกันด้วยคำสั่งเดียว — cAdvisor ใส่ไว้เพื่อดู CPU/memory ของ app ซึ่งเป็นตัวบอกคอขวด

สร้าง `docker-compose.yml`

```yaml
services:
  quickpizza:
    image: ghcr.io/grafana/quickpizza-local:latest
    ports: ["3333:3333"]

  prometheus:
    image: prom/prometheus:v3.5.0
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --web.enable-remote-write-receiver   # ให้ k6 push metric เข้ามาได้
      - --enable-feature=native-histograms
    volumes:
      - ./infra/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports: ["9090:9090"]

  grafana:
    image: grafana/grafana:12.1.0
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    volumes:
      - ./infra/grafana/provisioning:/etc/grafana/provisioning:ro
    ports: ["3000:3000"]

  cadvisor:   # CPU / memory ของแต่ละ container — สำคัญมากในบทที่ 5
    image: gcr.io/cadvisor/cadvisor:v0.52.1
    privileged: true
    # Docker Desktop (Mac): cAdvisor อ่านข้อมูล container จาก Docker ไม่ได้
    # จึงใช้โหมด raw cgroup แทน → metric จะมี label `id` (container id) แต่ไม่มี `name`
    command: ["--raw_cgroup_prefix_whitelist=/docker/", "--docker_only=false", "--housekeeping_interval=5s"]
    volumes:
      - /sys:/sys:ro
```

> - บน Linux ใช้ config มาตรฐานของ cAdvisor ได้ (mount `/var/run`, `/var/lib/docker`, `/rootfs`) จะได้ label `name` มาด้วย
> - ระหว่างยิงเปิด `docker stats` ใน terminal อีกอันไว้ด้วยเสมอ — เป็นตัวยืนยันค่าที่เร็วที่สุด

## 3.2 ตั้งค่า Prometheus

> **ทำไปเพื่อ:** ให้ Prometheus ดึง metric ฝั่ง server เข้ามาอยู่ที่เดียวกับ metric ของ k6 จะได้เอามาวางเทียบบนแกนเวลาเดียวกัน

`infra/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: app
    static_configs:
      - targets: ["quickpizza:3333"]   # /metrics ของ app
  - job_name: cadvisor
    static_configs:
      - targets: ["cadvisor:8080"]
```

> k6 ไม่ต้องอยู่ใน scrape_configs เพราะ k6 **push** เข้ามาเองผ่าน `/api/v1/write`

## 3.3 ให้ Grafana รู้จัก Prometheus อัตโนมัติ (provisioning)

> **ทำไปเพื่อ:** ไม่ต้องตั้งค่าผ่านหน้าเว็บทุกครั้งที่ลบ container — config อยู่ใน repo ทำซ้ำได้

`infra/grafana/provisioning/datasources/prometheus.yml`

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    uid: prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090   # ชื่อ service ใน compose ไม่ใช่ localhost
    isDefault: true
```

```bash
docker compose up -d
```

- Prometheus: http://localhost:9090 → Status → Targets ต้องเป็น UP
- Grafana: http://localhost:3000 → Connections → Data sources ต้องเห็น Prometheus

## 3.4 ส่ง metric จาก k6

> **ทำไปเพื่อ:** ส่งผลจาก k6 เข้า Prometheus ระหว่างยิง — `testid` ทำให้แยกและเทียบผลแต่ละรอบได้ (เช่น 1 task กับ 6 tasks)

```bash
export K6_PROMETHEUS_RW_SERVER_URL=http://localhost:9090/api/v1/write
export K6_PROMETHEUS_RW_TREND_STATS="p(95),p(99),avg,max"

k6 run -o experimental-prometheus-rw --tag testid=pizza-$(date +%H%M) scripts/patterns/pizza-flow.js
```

- `K6_PROMETHEUS_RW_TREND_STATS` — ถ้าไม่ตั้ง จะไม่มี p95 ใน Prometheus
- `--tag testid=...` — ติดป้ายให้ทุก metric ไว้แยกผลแต่ละรอบใน Grafana **ทำทุกครั้ง**

> แนะนำใส่ export 2 บรรทัดไว้ใน `.env` (และ gitignore) แล้ว `set -a; source .env; set +a`

ชื่อ metric ใน Prometheus จะขึ้นต้นด้วย `k6_` เช่น

| k6 | Prometheus |
| --- | --- |
| `http_reqs` | `k6_http_reqs_total` |
| `iterations` | `k6_iterations_total` |
| `http_req_duration` p95 | `k6_http_req_duration_p95` (หน่วยเป็น **วินาที**) |
| `http_req_failed` | `k6_http_req_failed_rate` |
| `checks` | `k6_checks_rate` |
| `vus` | `k6_vus` |
| custom `txn_success` | `k6_txn_success_total` |
| custom `txn_duration` | `k6_txn_duration_p95` |

## 3.5 Dashboard สำเร็จรูป

> **ทำไปเพื่อ:** ได้ภาพรวมเร็วโดยไม่ต้องสร้างเอง และใช้เป็นตัวอย่างว่า dashboard ที่ดีดูอะไรบ้าง

Grafana → Dashboards → New → **Import** → ใส่ ID **19665** (k6 Prometheus) → เลือก datasource Prometheus
รัน k6 อีกรอบแล้วเลือก testid ด้านบน

## 3.6 สร้าง dashboard เอง (สำคัญ — ต้องทำ)

> **ทำไปเพื่อ:** dashboard สำเร็จรูปไม่มี TPS ของ check-in และไม่มี CPU ของ app — สองค่านี้คือหัวใจของการตอบว่า "รับได้กี่ TPS และติดที่อะไร"

dashboard สำเร็จรูปไม่มีกราฟ TPS ของ business และไม่มีฝั่ง server ให้สร้าง dashboard ชื่อ **"Load Test — Capacity"** เพิ่มตัวแปร `testid` (Dashboard settings → Variables → Query: `label_values(k6_vus, testid)`) แล้วสร้าง panel ตามนี้:

| Panel | PromQL | Unit |
| --- | --- | --- |
| VUs | `sum(k6_vus{testid=~"$testid"})` | short |
| **TPS (สำเร็จ)** | `sum(rate(k6_txn_success_total{testid=~"$testid"}[30s]))` | ops/s |
| RPS | `sum(rate(k6_http_reqs_total{testid=~"$testid"}[30s]))` | req/s |
| p95 latency | `max(k6_http_req_duration_p95{testid=~"$testid"})` | s |
| p95 แยก step | `max by (name) (k6_http_req_duration_p95{testid=~"$testid"})` | s |
| Error rate | `avg(k6_http_req_failed_rate{testid=~"$testid"})` | percent (0-1) |
| RPS แยก status | `sum by (status) (rate(k6_http_reqs_total{testid=~"$testid"}[30s]))` | req/s |
| Dropped iterations | `sum(rate(k6_dropped_iterations_total{testid=~"$testid"}[30s]))` | ops/s |
| **App CPU** (% ของ limit) | `sum by (id) (rate(container_cpu_usage_seconds_total{id=~"/docker/($app_ids).*"}[30s])) / 0.25` | percent (0-1) |
| **App Memory** | `sum by (id) (container_memory_working_set_bytes{id=~"/docker/($app_ids).*"})` | bytes |
| App request rate (ฝั่ง server) | `sum(rate(quickpizza_server_http_requests_total[30s]))` | req/s |

> - `$app_ids` เป็นตัวแปรแบบ **Custom/Textbox** ใส่ container id ของ app (12 ตัวแรก คั่นด้วย `|`) หาได้จาก
>   `docker compose ps -q app | cut -c1-12 | paste -sd'|' -` — ต้องอัปเดตทุกครั้งที่ container ถูกสร้างใหม่
>   (บน Linux ใช้ `name=~".*app.*"` แทนได้เลย)
> - panel CPU หารด้วย `0.25` เพราะ limit ต่อ task คือ 0.25 vCPU (บทที่ 5) → 1.0 = 100% ของ limit
> - panel App request rate ใช้ชื่อ metric ของ QuickPizza ถ้าเป็น app จริงให้เปลี่ยนเป็น metric ที่ app ของคุณ expose
> - **วาง TPS, p95, Error rate, CPU ไว้แถวเดียวกัน** แกนเวลาตรงกัน จะเห็นทันทีว่า "TPS แบนตอน CPU ชน 100% แล้ว p95 พุ่ง"

Export dashboard เป็น JSON (Share → Export → Save to file) เก็บไว้ที่ `infra/grafana/dashboards/` แล้วเพิ่ม provider ให้โหลดอัตโนมัติ:

`infra/grafana/provisioning/dashboards/dashboards.yml`

```yaml
apiVersion: 1
providers:
  - name: load-test
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

และ mount `./infra/grafana/dashboards:/var/lib/grafana/dashboards:ro` ใน service grafana

## 3.7 อ่านกราฟให้เป็น

> **ทำไปเพื่อ:** แปลงกราฟเป็นข้อสรุป: ระบบยังสบาย / ถึงเพดานแล้ว / คอขวดอยู่ที่ CPU หรือที่อื่น — ทักษะที่ใช้ตรงๆ ในบทที่ 5

รูปแบบที่ต้องจำได้:

| สิ่งที่เห็น | แปลว่า |
| --- | --- |
| VUs ↑, TPS ↑ ตามสัดส่วน, p95 คงที่ | ยังสบาย |
| VUs ↑, **TPS แบน**, **p95 ↑** | **ถึงเพดานแล้ว (saturation)** — request เริ่มต่อคิว |
| TPS แบน + CPU ~100% | คอขวดคือ CPU ของ app |
| TPS แบน + CPU ยังต่ำ | คอขวดอยู่ที่อื่น: DB, connection pool, downstream API, lock |
| Memory ไต่ขึ้นเรื่อยๆ ไม่ลง | memory leak (ยืนยันด้วย soak test) — ที่ 1 GB จะโดน OOM kill |
| Error 5xx / timeout เริ่มโผล่ | เลยจุดแตกแล้ว |
| `dropped_iterations` > 0 | k6 ยิงไม่ถึงเป้า — VU ไม่พอ หรือ server ช้าจน VU ไม่ว่าง |
| p95 แยก step: step เดียวพุ่ง | step นั้นคือคอขวด → ไปดู query / downstream ของ step นั้น |

### โจทย์ 3

> **ทำไปเพื่อ:** ซ้อมหา "จุดที่เริ่มพัง" จากกราฟจริงบน QuickPizza ก่อนทำกับ app จริง

1. รันโจทย์ 2 แบบ stress (ramping-arrival-rate จาก 5 → 100 iterations/s ใน 5 นาที) โดยเปิด dashboard ค้างไว้
2. Screenshot dashboard เก็บไว้ใน `results/`
3. ตอบ: TPS เริ่มแบนที่เท่าไร? ตอนนั้น p95 เท่าไร? step ไหนช้าที่สุด?

> QuickPizza บนเครื่องที่ไม่จำกัด CPU อาจไม่แตกเลย — ลองใส่ `cpus: "0.25"` และ `mem_limit: 1g` ให้ service quickpizza (เหมือนบทที่ 5.1) แล้วรันใหม่
> จากการทดลอง: ที่ 0.25 CPU โจทย์ 2 แค่ 10 iterations/s ก็ทำให้ p95 พุ่งไปหลายสิบวินาที, error ~20% และมี `dropped_iterations` จำนวนมาก
> เป็นการซ้อมบทที่ 5 ทั้งบทกับ QuickPizza ก่อนไปทำกับ app จริง — **แนะนำมาก**

## 3.8 กรณี app ตาย — รู้ได้ยังไง และดูใน Grafana ยังไง

> **ทำไปเพื่อ:** ตอนหาเพดาน (บทที่ 5) app **จะ** ตายแน่นอน — ต้องแยกให้ออกว่าตายแบบไหน ตายตอนกี่ TPS และฟื้นเองได้ไหม เพราะคำตอบต่างกันมาก: "ช้าลง" ยังรับได้ แต่ "ตายแล้ว restart วนไปเรื่อยๆ" คือระบบล่มใน production

### app "ตาย" มีหลายแบบ

| แบบ | เกิดอะไรขึ้น | สาเหตุที่พบบ่อย |
| --- | --- | --- |
| **1. Crash** | process จบการทำงาน container หยุด | unhandled exception, panic |
| **2. OOM kill** | memory เกิน limit (1 GB) → ถูก kill ทันทีแล้ว restart | memory leak, โหลดเยอะจน object ค้างใน memory, cache ไม่จำกัดขนาด |
| **3. ค้าง (hang)** | container ยัง running แต่ **ไม่ตอบ** | thread/worker เต็ม, รอ DB connection, GC ทำงานไม่หยุด, deadlock |
| **4. ป่วย** | ยังตอบอยู่แต่ตอบ error 5xx | DB/downstream ล้ม, timeout ภายใน |
| **5. ตายบาง task** (6 tasks) | บาง task ตาย ที่เหลือรับต่อ | เหมือนข้อ 1–3 แต่ load balancer ตอบ 502/504 แทน |

แบบ 3 อันตรายที่สุด เพราะดูจาก `docker ps` จะเห็นว่า "running" ปกติ — ต้องดูจาก metric เท่านั้น

### เตรียมก่อนยิง

**1. ใส่ restart policy ให้ app** — ECS จะสร้าง task ใหม่แทนตัวที่ตาย local ต้องจำลองแบบเดียวกัน ไม่อย่างนั้นตายแล้วตายเลย

```yaml
  app:
    restart: unless-stopped
```

**2. ลด timeout ของ k6** — ค่า default คือ 60 วินาที ถ้า app ค้าง VU จะรอนานมากกว่าจะรู้ ให้ตั้งเท่ากับ timeout ของ ALB / client จริง

```js
http.get(url, { timeout: '10s' });
```

**3. เพิ่ม panel ใน dashboard "Load Test — Capacity"** (ต่อจากบท 3.6)

| Panel | PromQL | ใช้ดู |
| --- | --- | --- |
| **App up** | `up{job="app"}` | 1 = Prometheus ดึง `/metrics` ได้, **0 = app ตายหรือค้าง** |
| **Restarts** | `changes(process_start_time_seconds{job="app"}[1m])` | > 0 = process เริ่มใหม่ (crash / OOM) |
| **Restarts (container)** | `changes(container_start_time_seconds{id=~"/docker/($app_ids).*"}[1m])` | ใช้แทนได้ถ้า app ไม่มี `process_start_time_seconds` |
| **Memory เทียบ limit** | `container_memory_working_set_bytes{id=~"/docker/($app_ids).*"} / (1024*1024*1024)` | unit: percent (0-1) — ใกล้ 1.0 = ใกล้โดน OOM |
| **Errors แยกสาเหตุ** | `sum by (status, error_code, error) (rate(k6_http_reqs_total{testid=~"$testid", expected_response="false"}[30s]))` | แยกว่า error มาจากอะไร (ตารางด้านล่าง) |

> - `process_start_time_seconds` มีใน Prometheus client มาตรฐานของ Go / Java / Node / Python ถ้า app ไม่มี ให้ใช้ของ cAdvisor
> - `container_oom_events_total` ของ cAdvisor **นับ OOM ไม่ได้** บน Docker Desktop (Mac) (ทดลองแล้วได้ 0 ตลอด) ให้ใช้ `docker events` แทน (ด้านล่าง)

**4. ใส่ annotation ให้เห็นเส้นตอน restart บนทุก panel** — Dashboard settings → Annotations → New
- Data source: Prometheus
- Query: `changes(process_start_time_seconds{job="app"}[1m]) > 0`
- จะได้เส้นแนวตั้งบนทุกกราฟตรงเวลาที่ app restart → เห็นทันทีว่า TPS / p95 ตอนนั้นเป็นยังไง

### อ่าน error จาก k6

เวลา request ไม่ได้ response กลับมา k6 จะให้ `status = 0` และใส่ `error_code` กับ `error` (ข้อความ) มาด้วย:

| ที่เห็น | แปลว่า | น่าจะเป็นแบบไหน |
| --- | --- | --- |
| `status=0`, `error` มี `connection refused` | ไม่มีใครรอรับที่ port นั้น | **ตายแล้ว** (1, 2) หรือกำลัง start |
| `status=0`, `error` มี `connection reset by peer` | ต่อได้แล้วโดนตัดกลางทาง | กำลังตาย / โดน kill ระหว่างตอบ |
| `status=0`, `error` มี `request timeout` | ต่อได้แต่ไม่ตอบภายใน timeout | **ค้าง** (3) หรือช้ามาก |
| `status=5xx` | app ตอบแต่ตอบ error | **ป่วย** (4) |
| `status=502` / `504` (มี nginx/ALB หน้า app) | load balancer ต่อไป task ไม่ได้ / task ไม่ตอบ | **ตายบาง task** (5) |

> `error_code` เป็นตัวเลขของ k6 (เช่น 1050, 1212) ความหมายดูได้จาก [k6 error codes](https://grafana.com/docs/k6/latest/javascript-api/error-codes/) — ใน panel ให้ใส่ label `error` ด้วยจะอ่านเป็นข้อความได้เลย

### ยืนยันจากฝั่ง Docker

```bash
# app restart ไปกี่ครั้งแล้ว
docker inspect <container> --format 'RestartCount={{.RestartCount}}'

# ดู event การตาย/OOM ย้อนหลัง (เปิดค้างไว้ระหว่างยิงได้โดยตัด --until ออก)
docker events --since 30m --until 0s --filter container=<container> --filter event=die --filter event=oom

# log ก่อนตาย (มักมี stack trace / out of memory)
docker compose logs --tail 100 app
```

> `docker inspect ... .State.OOMKilled` จะเป็น `false` หลัง restart แล้ว (ค่านี้ดูแค่รอบล่าสุด) — เชื่อ `docker events` กับ `RestartCount` มากกว่า

### ตัวอย่างจริง: ลองทำให้ QuickPizza ตาย

ทดลองบน MacBook — QuickPizza `cpus: "0.25"`, `restart: unless-stopped`, ยิง `constant-arrival-rate`, timeout 10s

**ทดลอง 1 — หยุด app กลาง test** (`docker compose stop app` แล้ว 25 วินาทีต่อมา `start`) ที่ 5 req/s

| เวลา | App up | k6 เห็น |
| --- | --- | --- |
| ก่อนหยุด | 1 | `status=200` ~5 req/s |
| ระหว่างหยุด | **0** | `status=0`, `connection refused` ~5 req/s |
| หลัง start | 1 | กลับมา `status=200` ทันที |
| ผลรวม | | `http_req_failed` 28.6%, checks ผ่าน 71% |

**ทดลอง 2 — memory ไม่พอ** (`mem_limit: 40m`) ยิง `/api/pizza` ที่ 60 req/s 1 นาที

| ช่วง | สิ่งที่เห็นใน Grafana |
| --- | --- |
| 10 วินาทีแรก | Restarts ขึ้นเป็น 1, 2, 4 ภายในไม่กี่วินาที (`docker events` เห็น `oom` → `die` → `start`) error ผสม: `connection refused`, `500`, `401` |
| หลังจากนั้น | container **running** ปกติ แต่ **App up = 0** ตลอด, memory ค้างที่ ~34/40 MB, k6 ได้ `request timeout` ~50 req/s → **ค้าง** |
| ผลรวม | `http_req_failed` 89%, `RestartCount=4` |

บทเรียนจากการทดลอง 2: app ตายแล้ว restart (แบบ 2) แล้วต่อด้วยค้าง (แบบ 3) — ถ้าดูแค่ `docker ps` จะคิดว่าปกติ

### Timeline การตาย — ตายตอนไหน ฟื้นตอนไหน ล่มนานเท่าไร

> **ทำไปเพื่อ:** "app ตาย" อย่างเดียวยังไม่พอจะตัดสินใจ ต้องตอบได้ว่า **ตายตอนกี่โมง ที่โหลดเท่าไร มีสัญญาณเตือนก่อนไหม ฟื้นตอนไหน และล่มรวมกี่วินาที** — ตัวเลขเหล่านี้ใช้ตั้ง alarm / autoscaling และใส่ในรายงานบทที่ 7

#### ขั้นที่ 1 — บันทึก `docker events` ระหว่างยิง (เวลาแม่นระดับวินาที)

เปิด terminal อีกอันก่อนเริ่ม k6:

```bash
docker events \
  --filter container=<ชื่อ container ของ app> \
  --filter event=oom --filter event=die --filter event=start \
  | tee results/events-<testid>.log
```

กด `Ctrl+C` เมื่อ test จบ ถ้าไฟล์ว่าง แต่ Grafana บอกว่า app ล่ม = app **ค้าง** (ไม่ได้ตาย Docker จึงไม่มี event)

#### ขั้นที่ 2 — เพิ่ม panel timeline ใน Grafana

| Panel | ชนิด | PromQL | ตั้งค่า |
| --- | --- | --- | --- |
| **สถานะ app** | State timeline | `up{job="app"}` | Value mappings: `1` → `UP` (เขียว), `0` → `DOWN` (แดง) — จะเห็นแถบสีตามเวลา ชี้เมาส์เห็นเวลาเริ่ม/จบ |
| **เวลาที่ล่มครั้งแรก** | Stat | `min_over_time(timestamp(up{job="app"} == 0)[$__range:5s]) * 1000` | Unit: Date & time → Datetime ISO |
| **ล่มรวม (วินาที)** | Stat | `count_over_time((up{job="app"} == 0)[$__range:5s]) * 5` | Unit: seconds, No value = 0 |
| **จำนวนครั้งที่ restart** | Stat | `changes(process_start_time_seconds{job="app"}[$__range])` | No value = 0 |
| **TPS ที่ยิง vs TPS สำเร็จ** | Time series | A: `sum(rate(k6_iterations_total{testid=~"$testid"}[30s]))`<br>B: `sum(rate(k6_txn_success_total{testid=~"$testid"}[30s]))`<br>(app จริงใช้ `k6_checkin_success_total`) | 2 เส้นบนกราฟเดียว — เส้นแยกกัน = เริ่มรับไม่ไหว |

> - `* 5` และ `:5s` มาจาก `scrape_interval: 5s` ถ้าตั้งค่าอื่นให้เปลี่ยนตาม
> - Stat ทั้ง 3 ตัวคิดจากช่วงเวลาที่เลือกบน dashboard — ตั้ง time range ให้ครอบเฉพาะ test รอบนั้น
> - เมื่อใช้ 6 tasks ให้ใส่ `by (instance)` จะได้ timeline แยกทีละ task

#### ขั้นที่ 3 — เขียน timeline

ไล่กราฟจากซ้ายไปขวา แล้วจดทุกจุดที่มีการเปลี่ยนแปลงลงตาราง:

| เวลา | เหตุการณ์ | TPS ที่ยิง / สำเร็จ | ดูจาก |
| --- | --- | --- | --- |
| | เริ่ม test | | k6 |
| | **สัญญาณเตือนแรก** — TPS สำเร็จเริ่มต่ำกว่า TPS ที่ยิง / p95 เริ่มพุ่ง / memory ไต่ขึ้น | | panel TPS, p95, Memory |
| | **ล่มครั้งแรก** — App up = 0 / error เริ่มขึ้น / `die` หรือ `oom` | | panel สถานะ app, Errors, `docker events` |
| | restart (ถ้ามี) | | `docker events` `start`, panel Restarts |
| | **ฟื้น** — App up = 1 และ error กลับเป็น 0 | | panel สถานะ app, Errors |
| | จบ test | | k6 |

แล้วคำนวณ:

| ค่า | วิธีคิด | ใช้ทำอะไร |
| --- | --- | --- |
| **TPS ตอนล่ม** | TPS ที่ยิง ณ เวลาล่มครั้งแรก | Max TPS ต้องต่ำกว่าค่านี้ |
| **เวลาเตือนล่วงหน้า** | เวลาล่ม − เวลาสัญญาณเตือนแรก | ถ้าสั้นมาก alarm / autoscaling จะตอบสนองไม่ทัน |
| **เวลาล่มแต่ละครั้ง** | เวลาฟื้น − เวลาล่ม | ผู้ใช้ใช้งานไม่ได้กี่วินาที |
| **ล่มรวม** | ผลรวมทุกครั้ง (หรือจาก panel "ล่มรวม") | ใส่รายงาน |
| **ฟื้นเองได้ไหม** | หลังโหลดลดลงแล้ว App up กลับเป็น 1 ไหม | ถ้าไม่ฟื้น = ต้องมีคน/ระบบ restart ให้ → ใน prod ต้องพึ่ง health check ของ ECS |

#### ตัวอย่างจริง — QuickPizza ค้างแล้วไม่ฟื้น

ทดลองบน MacBook: QuickPizza `cpus: "0.25"`, `mem_limit: 40m`, `restart: unless-stopped`
ยิง `ramping-arrival-rate`: 5 req/s 30 วินาที → ไต่ไป 60 req/s ใน 30 วินาที → ค้าง 30 วินาที → ลดกลับ 5 req/s 1 นาที, timeout 10s

| เวลา | เหตุการณ์ | ยิง / สำเร็จ (req/s) | ดูจาก |
| --- | --- | --- | --- |
| 08:15:54 | เริ่ม test | 5 / 5 | k6 |
| 08:16:24 | เริ่มไต่โหลด | 5 / 5 | k6 |
| 08:16:34 | ยังตามทัน | 9.3 / 9.3 | panel TPS |
| **08:16:39** | **สัญญาณเตือนแรก** — สำเร็จต่ำกว่าที่ยิง | 17.3 / 12.3 | panel TPS (2 เส้นแยกกัน) |
| **08:16:44** | **ล่ม** — App up = 0, error เริ่มขึ้น | ~17 / 1.3 | panel สถานะ app, Errors |
| 08:16:54 | error ทั้งหมดเป็น `request timeout` = **ค้าง** | 19 / 0 | panel Errors |
| 08:17:39 | โหลดสูงสุด ทุก request timeout | 57 / 0 | panel TPS |
| 08:17:54 | ลดโหลดกลับเหลือ 5 req/s | 5 / 0 | panel TPS |
| 08:18:34 | จบ test — **ยังไม่ฟื้น** container ยัง `Up`, `curl` ยัง timeout | 5 / 0 | panel สถานะ app, `docker ps` |

`docker events`: **ว่าง** (ไม่มี `die`/`oom`) · Restarts: 0

| ค่า | ผล |
| --- | --- |
| TPS ตอนล่ม | ~17 req/s |
| เวลาเตือนล่วงหน้า | ~5 วินาที (08:16:39 → 08:16:44) |
| ล่มรวม | ≥ 115 วินาที (panel "ล่มรวม") — นับถึงจบ test เท่านั้น |
| ฟื้นเองได้ไหม | **ไม่** — แม้โหลดลดเหลือ 5 req/s แล้ว |

สิ่งที่ได้จาก timeline นี้:
- app ไม่ได้ "ตาย" แต่ **ค้าง** → `restart: unless-stopped` ช่วยไม่ได้ เพราะ Docker ไม่รู้ว่า app มีปัญหา (ใน prod ต้องมี health check ที่ทำให้ ECS เปลี่ยน task ใหม่)
- สัญญาณเตือนมาก่อนล่มแค่ ~5 วินาที → การตั้ง alarm จาก CPU/latency เพียงอย่างเดียวจะไม่ทัน ต้องจำกัดโหลดไม่ให้ถึงจุดนี้ (Max TPS ต้องต่ำกว่า ~17 พอสมควร)
- เทียบกับ "ทดลอง 2" ด้านบน: ตั้ง memory 40 MB เท่ากัน แต่ครั้งนั้น (ยิงคงที่ 60 req/s ทันที) โดน OOM restart 4 ครั้งก่อนค้าง → **การตายไม่ได้เกิดเหมือนเดิมทุกรอบ** ต้องทดสอบซ้ำอย่างน้อย 2 รอบ

### ระบุลงผลการทดสอบ

เมื่อ app ตายระหว่างทดสอบ ให้จดลงตารางผล (บทที่ 5) ทุกครั้ง:
- **ตายที่ TPS เท่าไร** (อ่านจาก panel TPS ตรงเส้น annotation)
- **ตายแบบไหน** (1–5) และหลักฐาน (panel ไหน, `docker events`, log)
- **ฟื้นเองได้ไหม ใช้เวลากี่วินาที** (จากจุดที่ App up = 0 จนกลับมา 1 และ error กลับเป็น 0)
- **timeline** ตามตารางด้านบน แนบไฟล์ `results/events-<testid>.log` และ screenshot panel สถานะ app
- **ตายซ้ำไหม** — ถ้า restart แล้วตายอีกเรื่อยๆ (crash loop) = ระดับโหลดนั้นทำให้ระบบล่ม ไม่ใช่แค่ช้า

> **Max TPS ต้องอยู่ต่ำกว่าจุดที่ app เริ่มตาย/restart เสมอ** แม้ p95 ณ จุดนั้นยังผ่าน SLO ก็ตาม

### โจทย์ 3.8

ทำซ้ำการทดลองทั้ง 2 แบบด้านบนกับ QuickPizza ของตัวเอง แล้ว screenshot dashboard ที่เห็น App up, Restarts, Memory, Errors แยกสาเหตุ และ annotation พร้อมกัน
แล้วเขียน timeline ของตัวเอง 1 ตาราง พร้อมค่า TPS ตอนล่ม, เวลาเตือนล่วงหน้า, ล่มรวม และฟื้นเองได้ไหม

## ✅ Checkpoint

- [ ] `docker compose up -d` แล้ว Prometheus target UP ครบ
- [ ] เห็น metric k6 ใน Grafana แยกตาม testid
- [ ] มี dashboard "Load Test — Capacity" ของตัวเอง (export JSON ไว้ใน repo)
- [ ] อธิบายรูปแบบ saturation จากกราฟได้
- [ ] dashboard มี panel App up, Restarts, Memory เทียบ limit, Errors แยกสาเหตุ และ annotation ตอน restart
- [ ] แยกได้ว่า app ตายแบบไหน (crash / OOM / ค้าง / ป่วย) จาก Grafana และ `docker events`
- [ ] เขียน timeline การตายได้: ตายกี่โมง ที่ TPS เท่าไร เตือนล่วงหน้ากี่วินาที ฟื้นตอนไหน ล่มรวมกี่วินาที

➡️ [บทที่ 4 — Check-in flow](04-checkin-flow.md)
