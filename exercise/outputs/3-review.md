> Agent: Claude Code — model: Claude Opus 5.5 (`claude-opus-5-5`) · Task 3 · language: Thai · reviewed: `docs/adr/0002-adr.md`

# ATAM Review: ADR-0002 — Publish "OrderPlaced" to RabbitMQ

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QA1 Performance | ลูกค้า | สั่งซื้อ **200 orders/s** | ช่วง 11.11, MailFast ช้าถึง p99 **8 s** | `PlaceOrder` คืนผลหลัง publish (พร้อม publisher confirm) โดยไม่รอ MailFast | p95 ของ `PlaceOrder` ≤ **500 ms**; publish + confirm ใช้งบไม่เกินประมาณ **200 ms** |
| QA2 Availability + correctness | RabbitMQ (node ล่ม หรือ cluster ไม่ว่าง) | publish `OrderPlaced` ล้มเหลว/timeout | peak 200 orders/s | order ยังสำเร็จ **และ** event ไม่หาย ส่งถึง `loyaltyservice` ภายหลัง | order ล้มเหลวเพราะ broker = **0**; แต้มหาย = **0**; แต้มซ้ำ = **0**; ให้แต้มภายใน **60 s** |
| QA3 Rate limit | `emailservice` | ข้อความเข้า 200/s + retry ของ **0.5 %** ที่ล้มเหลว | peak, `emailservice` หลาย replica | queue พักงาน, ส่งออกรวมทุก replica (รวม retry) ไม่เกินขีด | เรียก MailFast ≤ **50/s** ตลอดเวลา; อีเมลหาย = 0 (ที่ค้างต้องเห็นใน DLQ) |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | Decision ข้อ 2: ถ้า publish ล้มเหลว checkout คืน order สำเร็จแล้วแค่ log — event หายถาวร ส่วน outbox ถูกเลื่อนไปเป็น Open question | ลูกค้าไม่ได้แต้มเลย ซึ่งก็ไม่ใช่ "exactly once" เหมือนกัน ADR จึงยังไม่ผ่าน driver 3 ตามที่อ้างใน *เหตุผล* |
| Risk | อ้างว่ามี "rate control ในตัว" แต่ RabbitMQ มีแค่ prefetch (คุมจำนวน message ค้างต่อ consumer) ไม่ได้คุมอัตราต่อวินาที | ถ้ามีหลาย replica จะเกิน 50/s ได้ เพราะแต่ละ pod นับเอง; retry ก็กินโควตาเดียวกัน ต้องมี limiter รวมทุก replica |
| Risk | ส่ง event ฉบับเต็ม (มี email, รายการสินค้า) ไปที่ topic exchange เดียว แต่ mitigation บอกว่า "ให้ analytics ได้แค่ field ที่จำเป็น" | การ fan-out ส่ง message เดียวกันให้ทุก queue ข้อความที่ analytics ได้จึงมี PII เต็ม — mitigation นี้ทำไม่ได้ถ้าไม่มี event แยกหรือขั้นแปลงข้อมูล |
| Sensitivity point | **publish timeout** ของ checkout เมื่อใช้ publisher confirms กับ quorum queue 3 nodes (ต้อง replicate ให้ majority ก่อนจึง confirm) | ตั้ง timeout ใกล้ 200 ms จะกินงบ p95 จนหมดตอน broker ช้า; ตั้งสั้นเกินไปจะเพิ่มจำนวน event ที่หาย (ย้อนไป Risk แรก) |
| Sensitivity point | throughput ของ `loyaltyservice` (transaction ที่ dedup + ให้แต้ม) เทียบกับ 200/s | ถ้าทำได้ต่ำกว่า 200/s ต่อเนื่อง backlog จะโตจนเกิน 60 s; การ retry แบบ backoff ใน loyalty ก็ทำให้เกิน 60 s ได้เหมือนกัน |
| Sensitivity point | ระยะเวลา peak T (ADR คำนวณ backlog 150×T, ระบาย 3×T ได้ถูกต้อง) | ใช้กำหนดขนาด disk ของ queue และความช้าของอีเมล แต่ T ยังไม่รู้ (Open question 1) |
| Trade-off point | ไม่ block order เมื่อ broker ล่ม (availability ↑) แลกกับ event หาย (correctness ของ loyalty ↓) | เป็นแกนของ ADR นี้ ต้องเลือกให้ชัดว่ารับได้แค่ไหน ไม่ใช่ปล่อยไว้ใน Open questions |
| Trade-off point | publisher confirms + quorum queue (durability ↑) แลกกับ latency ของ checkout (performance ↓) และภาระดูแล stateful cluster ของทีม 4 คน (operability ↓) | ยังไม่มีตัวเลข confirm latency (ADR ยอมรับว่าเป็น [Inferred]) |
| Non-risk | ตัด Option 1 ด้วยตัวเลข: MailFast p95 1.5 s > งบ 500 ms และ 200/s > 50/s | เหตุผลอิงตัวเลขจาก scenario โดยตรง ไม่ต้องถกเถียงเพิ่ม |
| Non-risk | แยก durable queue ต่อ consumer และ dedup ด้วย `order_id` ใน transaction เดียวกับการให้แต้ม | backlog ของอีเมลไม่ถ่วง loyalty; at-least-once delivery + idempotent consumer = แต้มไม่ซ้ำ (เฉพาะ event ที่ publish สำเร็จ) |
| Non-risk | เลือก RabbitMQ แทน Kafka ที่ 200 msg/s | ปริมาณนี้ไม่ต้องใช้ partition/replay ของ Kafka และทีมมีคนเคยใช้ RabbitMQ 2 คน |

## 3. Missing or weak

- **ตรรกะไม่สม่ำเสมอ:** ตัด Option 2 เพราะ "งานหายเมื่อ pod restart" แต่ Option 3 ที่เลือกก็ยังทำงานหายเมื่อ publish ล้มเหลว (ช่องโหว่แคบกว่า แต่เป็นปัญหาแบบเดียวกัน)
- **ทางเลือกที่ไม่ได้พิจารณา:** (a) ให้ checkout retry ในเครื่อง หรือเก็บ event ลง disk (local spool) ระหว่างรอ broker กลับมา โดยไม่ต้องมี DB; (b) transactional outbox ที่ระบุ datastore ชัดเจน; (c) NATS JetStream (open-source, ดูแลง่ายกว่า) เป็นตัวเลือกที่ 5
- **ข้ออ้างที่อ่อน:** "retry/DLQ มีมาในตัว" — DLQ มีจริง แต่ delayed retry แบบ exponential backoff ต้องใช้ TTL + dead-letter exchange หรือ plugin เพิ่ม ไม่ได้มาฟรี
- **ผลกระทบที่ไม่ได้ระบุ:** schema/versioning ของ `OrderPlaced` เมื่อมี consumer ใหม่; concurrency ของ `emailservice` (50/s × p99 8 s ≈ 400 คำขอค้างพร้อมกัน); **fraud scoring** ปกติต้องทำ *ก่อน* ตัดเงิน จึงอาจไม่เข้ากับ event หลัง order — ข้ออ้าง "action ใหม่ = แค่เพิ่ม consumer" จึงกว้างเกินจริง

## 4. Verdict: **Accept with changes**

1. **ตัดสินเรื่อง event หายตอนนี้เลย ไม่ใช่ใน Open questions:** เลือก outbox (ระบุ datastore) หรือ local spool + retry ให้ QA2 วัดได้ว่า "แต้มหาย = 0"
2. **กำหนดงบและพฤติกรรมของการ publish:** ระบุค่า publish timeout, วัด confirm latency ของ quorum queue ที่ 200/s ก่อน accept และแก้ข้ออ้าง "rate control ในตัว" เป็น limiter รวมทุก replica (นับ retry ด้วย) ที่ ≤ 50/s
3. **แยกข้อมูลต่อ consumer:** ออกแบบ event/payload ให้ analytics ไม่ได้รับ email ของลูกค้า และกำหนดวิธี versioning ของ schema สำหรับ consumer ในอนาคต
