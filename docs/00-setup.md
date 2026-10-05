# บทที่ 0 — เตรียมเครื่อง

## 0.1 ติดตั้งเครื่องมือ

```bash
brew install k6          # load generator
brew install --cask docker   # หรือใช้ Docker Desktop / OrbStack ที่มีอยู่แล้ว
```

ตรวจสอบ:

```bash
k6 version          # ควรได้ k6 v1.x หรือ v2.x
docker version
docker compose version
```

> **Docker Desktop → Settings → Resources**: ให้ CPU อย่างน้อย 4 cores, RAM 6 GB
> เพราะบทที่ 5 จะรัน app 6 ตัว + Prometheus + Grafana พร้อมกัน

## 0.2 Target สำหรับฝึกยิง: QuickPizza

ห้ามฝึกยิงเว็บของคนอื่นหรือ environment ที่ใช้ร่วมกัน ให้ใช้ **QuickPizza** — แอปตัวอย่างที่ Grafana ทำไว้ให้ฝึก k6 โดยเฉพาะ

```bash
docker run --rm -p 3333:3333 ghcr.io/grafana/quickpizza-local:latest
```

เปิด http://localhost:3333 จะเห็นหน้าสุ่มพิซซ่า

API ที่จะใช้ฝึก (ลองยิงด้วย curl ให้ครบก่อนไปบทต่อไป):

| Method | Path | ต้อง token? | ใช้ทำอะไร |
| --- | --- | --- | --- |
| GET | `/` | ไม่ | หน้าแรก |
| POST | `/api/users` | ไม่ | สมัคร `{"username","password"}` → 201 |
| POST | `/api/users/token/login` | ไม่ | login → `{"token": "..."}` |
| POST | `/api/pizza` | ใช่ | ขอพิซซ่าแนะนำ → `{"pizza": {"id": ...}}` |
| POST | `/api/ratings` | ใช่ | ให้คะแนน `{"pizza_id", "stars"}` → 201 |
| GET | `/api/ratings` | ใช่ | ดูคะแนนของตัวเอง |
| GET | `/metrics` | ไม่ | Prometheus metrics ฝั่ง server |

header ที่ต้องใช้กับ API ที่ต้อง token: `Authorization: token <token>`

### โจทย์ 0

ใช้ `curl` ทำ flow นี้ให้สำเร็จ: สมัคร → login → เอา token ไปขอพิซซ่า → ให้คะแนนพิซซ่าที่ได้

<details>
<summary>เฉลย</summary>

```bash
curl -X POST localhost:3333/api/users -d '{"username":"alice1","password":"secret12"}'
TOKEN=$(curl -s -X POST localhost:3333/api/users/token/login \
  -d '{"username":"alice1","password":"secret12"}' | jq -r .token)

curl -X POST localhost:3333/api/pizza -H "Authorization: token $TOKEN" \
  -d '{"maxCaloriesPerSlice":1000,"mustBeVegetarian":false,"excludedIngredients":[],"excludedTools":[],"maxNumberOfToppings":5,"minNumberOfToppings":2}'

curl -X POST localhost:3333/api/ratings -H "Authorization: token $TOKEN" \
  -d '{"pizza_id":1,"stars":5}'
```

สังเกต: request ถัดไปต้องใช้ค่าที่ได้จาก response ก่อนหน้า (token, pizza id) — นี่คือ **correlation** ซึ่งเป็นหัวใจของการต่อ flow check-in ในบทที่ 4
</details>

## ✅ Checkpoint

- [ ] `k6 version` ทำงาน
- [ ] เปิด http://localhost:3333 ได้
- [ ] ทำโจทย์ 0 ผ่าน

➡️ [บทที่ 1 — ยิง load test ครั้งแรก](01-first-test.md)
