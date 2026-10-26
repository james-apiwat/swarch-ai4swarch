> Agent: Claude Code (VS Code extension) · Model: Claude Opus 5.5 (`claude-opus-5-5`) · Run: a (zero-shot, context: none)

# ADR-0002: Publish an "OrderPlaced" event to a message broker for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** Not evidenced in the provided files (ดู Open questions)

## Context and problem

หลังจากลูกค้าสั่งซื้อสำเร็จ Online Boutique ต้องทำงานต่อจากการสั่งซื้อ (post-order actions) เพิ่มขึ้น ตาม requirement:

| # | Post-order action | ผู้รับผิดชอบ / ระบบ | หมายเหตุ |
|---|---|---|---|
| 1 | ส่งอีเมลยืนยันคำสั่งซื้อ | MailFast (external email provider ใหม่) | มาแทน logic แบบ mock ใน `emailservice` |
| 2 | ให้คะแนนสะสม (loyalty points) | `loyaltyservice` (ต้องสร้างใหม่) | |
| 3 | บันทึก analytics event "order placed" | ทีม marketing | |
| 4+ | SMS, fraud scoring, การอัปเดต recommendations | ยังไม่ระบุ | marketing คาดว่าจะเพิ่มในปีหน้า |

ปัญหาที่ต้องตัดสินใจ: **จะเชื่อมขั้นตอนสั่งซื้อ (order placement) เข้ากับ post-order actions เหล่านี้อย่างไร** เพื่อให้
- การสั่งซื้อไม่ช้าลงหรือล้มเหลวเพราะ MailFast (ระบบภายนอก) หรือ `loyaltyservice` (ระบบใหม่) มีปัญหา
- เพิ่ม action ใหม่ในปีหน้าได้โดยไม่ต้องแก้ service ที่ทำการสั่งซื้อทุกครั้ง

ข้อจำกัดของ ADR นี้: เขียนจาก requirement เพียงอย่างเดียว (`context: none`) จึงไม่มีหลักฐานจากโค้ดว่าปัจจุบัน service ใดเป็นผู้ทำการสั่งซื้อ หรือเรียก `emailservice` ด้วยวิธีใด **[Assumption]** ในเอกสารนี้เรียกผู้ทำการสั่งซื้อว่า "order-placing service" (มักเป็น `checkoutservice` ใน Online Boutique แต่ Not evidenced in the provided files) และไม่มีตัวเลข เช่น ปริมาณคำสั่งซื้อ, latency target หรือ SLA ของ MailFast (ดู Open questions)

## Decision drivers

เรียงจากสำคัญที่สุด:

1. **Availability ของการสั่งซื้อ** — การสั่งซื้อต้องสำเร็จได้แม้ MailFast, `loyaltyservice` หรือ analytics ล่มหรือช้า
2. **Modifiability / extensibility** — เพิ่ม post-order action ใหม่ (SMS, fraud scoring, recommendations) ได้โดยแก้ order-placing service น้อยที่สุด
3. **Performance (latency ของการสั่งซื้อ)** — ไม่เพิ่ม latency ที่ลูกค้ารู้สึกได้ตามจำนวน action
4. **Reliability ของ action** — ต้องไม่ทำอีเมลหรือคะแนนสะสมหาย และไม่ให้คะแนนซ้ำ
5. **Operability และ cost** — จำนวน infrastructure ใหม่ที่ต้องดูแล
6. **Integrability กับระบบภายนอก** — แยก dependency กับ MailFast ไว้ที่จุดเดียว เพื่อเปลี่ยน provider ได้ในอนาคต

## Considered options

1. **Option A — Synchronous orchestration:** order-placing service เรียก MailFast adapter, `loyaltyservice` และ analytics ทีละตัวโดยตรง (เช่น gRPC/HTTPS) ภายใน request การสั่งซื้อ
2. **Option B — Event-driven publish/subscribe:** order-placing service publish event `OrderPlaced` ไปยัง message broker หนึ่งครั้ง แต่ละ action เป็น consumer อิสระ (`emailservice` ที่เรียก MailFast, `loyaltyservice`, analytics consumer)
3. **Option C — Dedicated post-order orchestrator / workflow service:** order-placing service ส่งคำสั่งแบบ asynchronous ไปให้ service ใหม่ตัวเดียว (เช่น `postorderservice`) ซึ่งเรียก action แต่ละตัวตามลำดับ พร้อม retry

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A — Synchronous orchestration | • ง่ายที่สุด ไม่มี infrastructure ใหม่<br>• debug ง่าย เห็น flow ทั้งหมดในที่เดียว<br>• รู้ผลทันทีว่า action สำเร็จหรือไม่ | • การสั่งซื้อล้มหรือช้าเมื่อ MailFast / `loyaltyservice` มีปัญหา (availability ลดลงเป็นผลคูณของทุก dependency)<br>• latency เพิ่มตามจำนวน action<br>• ทุก action ใหม่ในปีหน้าต้องแก้และ deploy order-placing service ใหม่ (coupling สูง)<br>• ต้องเขียน timeout/retry/partial failure เองในทุก call |
| B — Event-driven pub/sub | • order-placing service ไม่ต้องรู้จัก consumer — เพิ่ม SMS / fraud scoring / recommendations ได้โดยเพิ่ม subscriber<br>• MailFast หรือ `loyaltyservice` ล่มไม่กระทบการสั่งซื้อ; event รอใน queue แล้ว retry ได้<br>• latency ของการสั่งซื้อคงที่ (publish ครั้งเดียว)<br>• แต่ละ consumer scale และ deploy อิสระ | • ต้องติดตั้งและดูแล message broker ใหม่ (operability, cost)<br>• eventual consistency: อีเมล/คะแนนมาถึงช้ากว่าการสั่งซื้อเล็กน้อย<br>• delivery แบบ at-least-once → consumer ต้อง idempotent (กันให้คะแนนซ้ำ/อีเมลซ้ำ)<br>• ความเสี่ยง "dual write": คำสั่งซื้อสำเร็จแต่ publish event ไม่สำเร็จ<br>• tracing ข้าม service ยากขึ้น ต้องมี correlation ID |
| C — Post-order orchestrator | • การสั่งซื้อไม่ผูกกับแต่ละ action โดยตรง<br>• logic ลำดับขั้น, retry, compensation อยู่ที่เดียว เห็นสถานะของแต่ละ order ชัด<br>• เหมาะถ้า action ในอนาคต (เช่น fraud scoring) ต้องทำตามลำดับหรือมีเงื่อนไข | • orchestrator เป็น single point of failure และคอขวดใหม่<br>• ทุก action ใหม่ยังต้องแก้ orchestrator (coupling ย้ายที่ ไม่ได้หายไป)<br>• ยังต้องมีช่องทาง asynchronous ระหว่าง order-placing service กับ orchestrator อยู่ดี จึงมีความซับซ้อนของ B บางส่วนด้วย<br>• ต้องสร้างและดูแล service เพิ่มอีกตัว |

## Decision

เลือก **Option B — Event-driven publish/subscribe**

- order-placing service publish event `OrderPlaced` (เช่น order ID, user/email, รายการสินค้า, ยอดรวม, เวลา) ไปยัง message broker **หลังจาก** การสั่งซื้อสำเร็จ แล้วตอบลูกค้าทันที
- consumer แต่ละตัว subscribe แยกกัน:
  - `emailservice` → เปลี่ยน logic mock เป็นเรียก MailFast (MailFast adapter อยู่ใน `emailservice` เท่านั้น)
  - `loyaltyservice` (ใหม่) → คำนวณและบันทึกคะแนนสะสม
  - analytics consumer → บันทึก event "order placed" ให้ทีม marketing
- ใช้ **transactional outbox** หรือกลไกเทียบเท่าใน order-placing service เพื่อแก้ปัญหา dual write **[Inferred]** — จำเป็นหรือไม่ขึ้นกับว่า order ถูกเก็บถาวรหรือเปล่า ซึ่ง Not evidenced in the provided files
- consumer ทุกตัวต้อง **idempotent** โดยใช้ order ID เป็น key และมี dead-letter queue สำหรับ event ที่ล้มเหลวซ้ำ ๆ

เหตุผล: B ตอบ driver 1–3 (availability, modifiability, latency) ได้ดีที่สุด ซึ่งเป็น driver ที่ requirement ชี้ชัด — มี external dependency ใหม่ (MailFast), service ใหม่ที่ยังไม่ mature (`loyaltyservice`), และ action เพิ่มอีกอย่างน้อย 3 ตัวในปีหน้า A ทำให้ทุก action ใหม่เพิ่มทั้ง latency และจุดล้มเหลวให้การสั่งซื้อ ส่วน C ย้าย coupling ไปที่ orchestrator แต่ไม่ได้กำจัดมัน B แลกด้วยต้นทุน operability ของ broker และ eventual consistency ซึ่งยอมรับได้สำหรับอีเมล, คะแนนสะสม และ analytics **[Inferred]** — ต้องยืนยันกับ stakeholder (ดู Open questions)

ถ้าในอนาคต fraud scoring ต้อง **บล็อก** การสั่งซื้อ (ต้องตรวจก่อนยืนยัน order) สิ่งนั้นไม่ใช่ post-order action และควรเป็น synchronous call แยกต่างหาก พร้อม ADR ใหม่

## Consequences

- **Positive:** การสั่งซื้อสำเร็จได้แม้ MailFast, `loyaltyservice` หรือ analytics ล่ม; event ค้างใน broker แล้วประมวลผลภายหลัง (quality attribute: availability)
- **Positive:** เพิ่ม SMS, fraud scoring (แบบไม่บล็อก) และ recommendations update ได้โดยเพิ่ม subscriber ใหม่ ไม่ต้องแก้ order-placing service (quality attribute: modifiability / extensibility)
- **Positive:** latency ของการสั่งซื้อไม่เพิ่มตามจำนวน post-order action (quality attribute: performance)
- **Positive:** dependency กับ MailFast อยู่ใน `emailservice` ที่เดียว; เปลี่ยน provider ได้โดยไม่กระทบ service อื่น (quality attribute: modifiability, integrability)
- **Positive:** consumer แต่ละตัว scale ได้อิสระ เช่น analytics ที่มี volume สูง (quality attribute: scalability)
- **Negative:** ต้องติดตั้ง, monitor และดูแล message broker เพิ่ม (quality attribute: operability, cost)
- **Negative:** อีเมลยืนยันและคะแนนสะสมเป็น eventual consistency — ลูกค้าอาจเห็นคะแนนช้ากว่าการสั่งซื้อ (quality attribute: consistency / usability)
- **Negative:** debug flow ข้าม service ยากขึ้น ต้องมี distributed tracing / correlation ID (quality attribute: testability, observability)
- **Negative:** schema ของ `OrderPlaced` กลายเป็น contract ร่วมของหลายทีม; การเปลี่ยน schema ต้อง backward compatible (quality attribute: modifiability)
- **Risks and mitigations:**
  - *Event หายระหว่าง commit order กับ publish (dual write)* → transactional outbox หรือ publish พร้อม retry และ reconciliation job
  - *ให้คะแนนซ้ำหรือส่งอีเมลซ้ำ (at-least-once delivery)* → idempotency key = order ID ใน `loyaltyservice` และ `emailservice`; ใช้ idempotency key ของ MailFast ถ้ามี (Not evidenced in the provided files)
  - *MailFast ล่มนาน หรือ rate limit* → retry with exponential backoff, circuit breaker, dead-letter queue และ alert
  - *Broker ล่ม* → ใช้ broker แบบ replicated/managed; outbox ทำให้ event รอใน order-placing service จนกว่า broker กลับมา
  - *ข้อมูลส่วนบุคคล (อีเมลลูกค้า) อยู่ใน event ที่ทีม marketing ใช้* → ส่งเฉพาะ field ที่จำเป็นให้ analytics หรือแยก topic; ตรวจกับนโยบาย privacy (ดู Open questions)
  - *Schema drift* → version field ใน event และ schema registry หรือ protobuf contract ร่วม

## Validation

- **Fault-injection test:** ปิด MailFast (หรือ mock ให้ timeout) และปิด `loyaltyservice` แล้วยืนยันว่าการสั่งซื้อยังสำเร็จ 100% และเมื่อ service กลับมา event ที่ค้างถูกประมวลผลครบ
- **Load test:** เปรียบเทียบ latency ของการสั่งซื้อก่อนและหลังเปลี่ยน; เป้าหมายตัวเลขยังไม่กำหนด (ดู Open questions)
- **Idempotency test:** ส่ง `OrderPlaced` ซ้ำ ID เดิมหลายครั้ง แล้วยืนยันว่าได้คะแนนครั้งเดียวและอีเมลฉบับเดียว
- **Metrics ใน production:** consumer lag, จำนวน event ใน dead-letter queue, อัตราส่งอีเมล MailFast สำเร็จ, เวลาจากสั่งซื้อจนถึงส่งอีเมล/ให้คะแนน (end-to-end delay)
- **Extensibility check:** เมื่อเพิ่ม action แรกในปีหน้า (เช่น SMS) ยืนยันว่าไม่ต้องแก้หรือ deploy order-placing service
- **Review date:** ทบทวน ADR นี้ก่อนเริ่มทำ action ของปีหน้า หรือเมื่อ fraud scoring ต้องเป็นแบบบล็อกการสั่งซื้อ

## Open questions

ข้อมูลต่อไปนี้ไม่ได้ให้มาใน requirement และไม่ได้คิดขึ้นเอง:

1. service ใดเป็นผู้ทำการสั่งซื้อ และปัจจุบันเรียก `emailservice` อย่างไร (synchronous? protocol อะไร?) — Not evidenced in the provided files
2. ปริมาณคำสั่งซื้อ (orders/second ทั้งปกติและช่วง peak) และ latency target ของการสั่งซื้อ
3. ยอมรับ delay ได้เท่าไรสำหรับอีเมลยืนยันและคะแนนสะสม (วินาที? นาที?)
4. SLA, rate limit, API (REST? SMTP?), รองรับ idempotency key หรือไม่, และค่าใช้จ่ายของ MailFast
5. order ถูกบันทึกลง database ถาวรหรือไม่ (มีผลต่อการใช้ transactional outbox)
6. มี message broker ในระบบอยู่แล้วหรือไม่ และทีมมีความชำนาญเทคโนโลยีใด (เช่น RabbitMQ, Kafka, Google Pub/Sub); งบประมาณสำหรับ infrastructure ใหม่
7. ทีม marketing ต้องการ analytics event ไปที่ระบบใด (มี analytics platform อยู่แล้ว หรือต้องสร้าง?) และต้องการ field อะไรบ้าง
8. กฎการคำนวณคะแนนสะสม และ `loyaltyservice` ต้องเก็บข้อมูลที่ไหน
9. fraud scoring ในปีหน้าต้องบล็อกการสั่งซื้อหรือทำหลังการสั่งซื้อ
10. ข้อกำหนดด้าน privacy / ข้อมูลส่วนบุคคล (เช่น PDPA) สำหรับการส่งอีเมลลูกค้าไปยัง MailFast และ analytics
11. ใครเป็น deciders ของ ADR นี้
