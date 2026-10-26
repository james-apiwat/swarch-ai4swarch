Agent: Claude Code — model: Claude Opus 5.5 (`claude-opus-5-5`)

# ADR-0002: Publish an "OrderPlaced" event to RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** ทีมพัฒนา Online Boutique (4 คน) — รายชื่อผู้ตัดสินใจจริง: ดู Open questions

## Context and problem

ข้อกำหนดใหม่ R-12 (`docs/scenario.md`) ต้องการให้หลังจากสั่งซื้อสำเร็จ ระบบต้อง (1) ส่งอีเมลยืนยันผ่าน **MailFast** ซึ่งเป็น external provider แทน logic จำลองของ `emailservice` (2) ให้ loyalty points ผ่าน `loyaltyservice` ที่จะสร้างใหม่ และ (3) บันทึก analytics event "order placed" ให้ทีม marketing และปีหน้าจะมี post-order action เพิ่มอีก (SMS, fraud scoring, recommendations update)

**สภาพปัจจุบัน (จากโค้ด):**

- `PlaceOrder` ทำงานแบบ synchronous ตามลำดับ: `chargeCard` → `shipOrder` → `emptyUserCart` → `sendOrderConfirmation` (`system/online-boutique/src/checkoutservice/main.go` บรรทัด 252–277)
- ถ้าส่งอีเมลล้มเหลว จะแค่ `log.Warnf` แล้วคำสั่งซื้อยังสำเร็จ (`main.go` บรรทัด 273–277)
- `sendOrderConfirmation` เรียก `EmailService.SendOrderConfirmation` ผ่าน gRPC ด้วย `ctx` เดียวกับ request โดยไม่มี timeout แยก (`main.go` บรรทัด 379–384) ดังนั้น latency ของอีเมลรวมอยู่ใน latency ของ `PlaceOrder` โดยตรง
- `checkoutservice` เชื่อมต่อ dependency ทุกตัวผ่าน env var + gRPC (`mustMapEnv` / `mustConnGRPC`, `main.go` บรรทัด 110–123) การเพิ่มปลายทางใหม่ทุกครั้งจึงต้องแก้ `checkoutservice`
- `orderID` สร้างด้วย `uuid.NewUUID()` (`main.go` บรรทัด 233) ใช้เป็น idempotency key ได้
- `checkoutservice` ไม่มี data store ของตัวเอง — Not evidenced in the provided files ว่ามีที่เก็บ order แบบถาวร

**ตัวเลขที่เกี่ยวข้อง (`docs/scenario.md`):**

| หัวข้อ | ค่า | ผลต่อการตัดสินใจ |
|---|---|---|
| p95 ของ `PlaceOrder` | ≤ 500 ms (ปัจจุบันประมาณ 300 ms) | เหลือ budget ประมาณ 200 ms |
| MailFast | p95 1.5 s, p99 8 s, fail ประมาณ 0.5 %, rate limit 50 emails/s | เรียกแบบ synchronous ไม่ได้ — p95 เพียงตัวเดียวก็เกิน budget แล้ว |
| Peak | 200 orders/s (11.11) | มากกว่า rate limit ของ MailFast 4 เท่า → ต้องมี buffer; backlog สะสมประมาณ 150 emails/s ระหว่าง peak |
| Loyalty | ภายใน 1 นาที, exactly once | ต้องมี retry + การกันซ้ำ (dedup) |
| Availability | order ต้องไม่ล้มเพราะ email/loyalty/analytics ล่ม | ต้องแยก post-order action ออกจาก critical path |
| ทีม | 4 คน, เก่ง Go/gRPC, 2 คนเคยใช้ RabbitMQ | |
| Operations | Kubernetes, ชอบ open-source self-hosted | |

**ปัญหา:** จะเชื่อม `checkoutservice` กับ post-order action ทั้งสามอย่าง (และที่จะเพิ่มในอนาคต) อย่างไร โดยไม่ทำให้ latency และ availability ของ checkout แย่ลง

## Decision drivers

1. **Availability:** คำสั่งซื้อต้องไม่ล้มเมื่อ MailFast, `loyaltyservice` หรือ analytics ล่ม
2. **Performance:** p95 ของ `PlaceOrder` ≤ 500 ms ที่ 200 orders/s
3. **Reliability / correctness:** loyalty points ต้องได้ภายใน 1 นาที และ exactly once; อีเมลต้องไม่หาย แม้ MailFast fail 0.5 %
4. **Rate-limit handling:** ต้องรองรับ 200 orders/s ขณะที่ MailFast รับได้ 50 emails/s
5. **Modifiability:** เพิ่ม post-order action ใหม่ (SMS, fraud scoring, recommendations) โดยไม่ต้องแก้ `checkoutservice` ทุกครั้ง
6. **Team fit และ cost:** ทีม 4 คน, ใช้ open-source self-hosted บน Kubernetes

## Considered options

1. **Synchronous gRPC ทั้งหมด** — ขยาย `PlaceOrder` ให้เรียก MailFast (ผ่าน `emailservice`), `loyaltyservice` และ analytics ตามลำดับ เหมือนแบบปัจจุบัน
2. **Fire-and-forget ภายใน process** — `checkoutservice` สั่งงานทั้งสามใน goroutine แล้วตอบกลับทันที (ไม่มี broker)
3. **Event-driven ด้วย RabbitMQ (self-hosted)** — `checkoutservice` publish event `OrderPlaced` หนึ่งครั้งไปยัง exchange แบบ fanout/topic; แต่ละ consumer (`emailservice`, `loyaltyservice`, analytics) มี queue ของตัวเอง
4. **Event-driven ด้วย Apache Kafka (self-hosted)** — แนวคิดเดียวกับข้อ 3 แต่ใช้ Kafka topic + consumer group

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous gRPC | ง่าย ทีมถนัด; ไม่มี component ใหม่; ผลลัพธ์รู้ทันที | MailFast p95 1.5 s / p99 8 s ทำให้เกิน 500 ms แน่นอน; 200 orders/s ชน rate limit 50/s; ทุก action เพิ่ม latency และจุดล้ม; ทุก action ใหม่ต้องแก้ `checkoutservice` |
| 2. Goroutine fire-and-forget | latency ต่ำ; ไม่มี infrastructure ใหม่ | งานหายถ้า pod restart/crash (ไม่มี durability) → loyalty ไม่ได้ "exactly once"; retry และ rate limiting ต้องเขียนเองใน `checkoutservice`; ยังผูก `checkoutservice` กับทุก action (modifiability ต่ำ) |
| 3. RabbitMQ | ตัด post-order ออกจาก critical path (publish ครั้งเดียว); queue เป็น buffer รับ 200/s แล้วปล่อยให้ email consumer ส่งที่ ≤ 50/s; มี ack/retry/dead-letter queue; เพิ่ม consumer ใหม่ได้โดยไม่แก้ publisher; open-source, รันบน Kubernetes ได้; 2 ใน 4 คนเคยใช้ | ต้อง operate broker เพิ่ม (cluster, monitoring); delivery เป็น at-least-once → ต้องทำ idempotency เอง; publish อาจล้มถ้า broker ล่ม; debug แบบ async ยากกว่า |
| 4. Kafka | throughput สูงมาก; เก็บ event ไว้ replay ได้ (เหมาะกับ analytics/recommendations ในอนาคต) | operate ยากกว่าสำหรับทีม 4 คน; ไม่มีใครในทีมเคยใช้ (ตาม scenario); 200 orders/s ไม่ต้องการ throughput ระดับนี้; rate limiting ต่อ consumer ต้องจัดการเอง |

## Decision

เลือก **Option 3: publish event `OrderPlaced` ไปยัง RabbitMQ** ที่ self-host บน Kubernetes

- ใน `PlaceOrder` แทนที่การเรียก `sendOrderConfirmation` (`main.go` บรรทัด 273–277) ด้วยการ publish `OrderPlaced` หนึ่งครั้ง (มี `order_id`, email, items, total) หลังจาก `shipOrder` สำเร็จ ใช้ publisher confirms และ timeout สั้นที่แยกจาก request `ctx`
- ถ้า publish ล้มเหลว: **ไม่ทำให้ order ล้ม** (driver 1) — log และ retry แบบจำกัดเวลา (ดู Risks)
- Consumer แต่ละตัวมี queue ของตัวเอง (bind กับ exchange เดียวกัน):
  - `emailservice` → เรียก MailFast โดยจำกัดอัตราไม่เกิน 50 emails/s, retry แบบ backoff, ส่งข้อความที่ล้มซ้ำไป dead-letter queue
  - `loyaltyservice` → ให้คะแนนแบบ idempotent: บันทึก `order_id` ที่ประมวลผลแล้วใน transaction เดียวกับการเพิ่มคะแนน เพื่อให้ at-least-once delivery กลายเป็น "effectively exactly once"
  - analytics consumer → บันทึก event "order placed"
- action ใหม่ในอนาคต (SMS, fraud scoring, recommendations) = เพิ่ม queue + consumer ใหม่ โดยไม่แก้ `checkoutservice`

**เหตุผล:** เป็นตัวเลือกเดียวที่ตอบ driver 1–5 ได้ครบ; Option 1 ละเมิด latency และ availability ตั้งแต่ต้น; Option 2 ไม่ durable จึงรับประกัน loyalty ไม่ได้; Option 4 ตอบโจทย์ได้แต่ต้นทุนการ operate และการเรียนรู้สูงกว่าความต้องการ (200 orders/s) สำหรับทีม 4 คน ในขณะที่ RabbitMQ มีประสบการณ์ในทีมอยู่แล้ว

## Consequences

- **Positive:** latency ของ `PlaceOrder` ไม่ขึ้นกับ MailFast อีกต่อไป เหลือเพียงต้นทุนการ publish (quality attribute: performance)
- **Positive:** การล่มของ MailFast, `loyaltyservice` หรือ analytics ไม่กระทบ checkout; ข้อความรออยู่ใน queue จนกว่า consumer กลับมา (quality attribute: availability)
- **Positive:** queue ดูดซับ peak 200 orders/s ขณะที่ `emailservice` ส่งที่ ≤ 50/s (quality attribute: scalability / rate-limit compliance)
- **Positive:** เพิ่ม post-order action ใหม่ได้โดยเพิ่ม consumer อย่างเดียว (quality attribute: modifiability)
- **Negative:** อีเมลยืนยันอาจล่าช้าระหว่าง peak — ถ้า peak 200 orders/s ต่อเนื่อง backlog โตประมาณ 150 emails/s (เช่น peak 10 นาทีจะค้างประมาณ 90,000 อีเมล ใช้เวลาระบายประมาณ 30 นาทีที่ 50/s) (quality attribute: timeliness / user experience)
- **Negative:** ต้อง operate RabbitMQ cluster เพิ่มบน Kubernetes (quality attribute: operability / cost)
- **Negative:** ระบบเป็น eventually consistent และ trace ข้ามขอบเขต async ยากกว่า (quality attribute: testability / observability)
- **Risks and mitigations:**
  - *Broker ล่มขณะ publish → event หาย (loyalty ไม่ได้)* — retry แบบจำกัดเวลาใน `checkoutservice`; ถ้าต้องการรับประกันเต็มที่ ให้ใช้ transactional outbox ซึ่งต้องมี order store (ปัจจุบัน Not evidenced in the provided files) — ดู Open questions
  - *ส่งซ้ำ (at-least-once) → ให้คะแนนซ้ำ / อีเมลซ้ำ* — dedup ด้วย `order_id` ใน `loyaltyservice` (บังคับ) และใน `emailservice` (แนะนำ)
  - *Loyalty เกิน 1 นาที* — queue ของ loyalty แยกจาก email จึงไม่ติด backlog ของ MailFast; ตั้ง alert เมื่อ queue age > 30 s
  - *Broker เป็น single point of failure ของ post-order* — รัน RabbitMQ แบบ cluster (เช่น quorum queues) [Inferred: จำนวน node ที่เหมาะสมยังไม่ได้กำหนด]
  - *MailFast fail 0.5 %* — retry ด้วย backoff + dead-letter queue + alert

## Validation

- Load test ด้วย Locust `loadgenerator` ที่ 200 orders/s: p95 ของ `PlaceOrder` ต้อง ≤ 500 ms
- Fault injection: ปิด MailFast (หรือ stub), `loyaltyservice` และ analytics consumer ระหว่าง test → อัตรา order สำเร็จต้องไม่ลดลง
- วัด p99 ของเวลาจาก order ถึงการให้ loyalty points ต้อง < 1 นาทีที่ peak; ส่ง event ซ้ำโดยเจตนาแล้วตรวจว่าคะแนนไม่ซ้ำ
- Monitor อัตราการเรียก MailFast ไม่เกิน 50/s, ความยาว queue และจำนวนข้อความใน dead-letter queue
- ทบทวน ADR นี้อีกครั้งหลัง 11.11 sale ครั้งแรก หรือก่อนเพิ่ม action ใหม่ปีหน้า

## Open questions

- อีเมลยืนยันยอมให้ล่าช้าได้นานเท่าไร? (ไม่ได้ระบุ — สำคัญต่อ backlog ระหว่าง peak; อาจต้องขอเพิ่ม rate limit จาก MailFast)
- ระยะเวลาของ peak 11.11 นานเท่าไร? (ใช้ประมาณขนาด backlog และ queue storage)
- การสูญเสีย event ระหว่าง publish ยอมรับได้แค่ไหน? ถ้ายอมไม่ได้เลย ต้องมี order store สำหรับ transactional outbox — ปัจจุบัน `checkoutservice` ไม่มี data store (Not evidenced in the provided files)
- ทีมใด operate RabbitMQ และมีงบ/ทรัพยากร cluster เท่าไร?
- `loyaltyservice` จะใช้ data store อะไร (สำหรับคะแนนและตาราง `order_id` ที่ประมวลผลแล้ว)?
- analytics event ส่งไปเก็บที่ไหน (ระบบ analytics ของทีม marketing คืออะไร) และต้องการ schema แบบใด?
- MailFast รองรับ idempotency key หรือไม่ (ป้องกันอีเมลซ้ำเมื่อ retry)?
- latency ของการ publish พร้อม publisher confirms บน cluster จริงเท่าไร? (ยังไม่ได้วัด)
- ใครคือ deciders ที่ต้องอนุมัติ ADR นี้?
