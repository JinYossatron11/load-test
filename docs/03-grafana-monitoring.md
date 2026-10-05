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

## ✅ Checkpoint

- [ ] `docker compose up -d` แล้ว Prometheus target UP ครบ
- [ ] เห็น metric k6 ใน Grafana แยกตาม testid
- [ ] มี dashboard "Load Test — Capacity" ของตัวเอง (export JSON ไว้ใน repo)
- [ ] อธิบายรูปแบบ saturation จากกราฟได้

➡️ [บทที่ 4 — Check-in flow](04-checkin-flow.md)
