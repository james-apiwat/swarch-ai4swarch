> Agent: Claude Code (Anthropic) · Model: Claude Opus 5.5 (`claude-opus-5-5`) · Run: a (zero-shot, context: none)

# ADR-0002: Publish an "OrderPlaced" event for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-31
- **Deciders:** ยังไม่ระบุ (ดู Open questions)

## Context and problem

หลังจากลูกค้าสั่งซื้อสำเร็จ Online Boutique ต้องทำงานต่อเนื่อง (post-order actions) 3 อย่าง ตาม requirement:

| # | งาน | ผู้รับผิดชอบ | สถานะ |
|---|---|---|---|
| 1 | ส่งอีเมลยืนยันคำสั่งซื้อผ่าน **MailFast** (external email provider) แทน logic จำลองใน `emailservice` | MailFast (ภายนอก) | ใหม่ |
| 2 | ให้ **loyalty points** | `loyaltyservice` | ต้องสร้างใหม่ |
| 3 | บันทึก analytics event **"order placed"** ให้ทีม marketing | ยังไม่ระบุระบบปลายทาง | ใหม่ |

นอกจากนี้ marketing คาดว่า **ปีหน้าจะมี post-order actions เพิ่ม** (SMS, fraud scoring, recommendations update)

ปัญหาที่ต้องตัดสินใจ: **service ที่ place order ควรแจ้ง/สั่งงาน post-order actions เหล่านี้อย่างไร** เพื่อให้เพิ่มงานใหม่ได้ง่าย และไม่ให้ความล้มเหลวของระบบภายนอก (เช่น MailFast) กระทบการสั่งซื้อ

ข้อจำกัด: run นี้เขียนจาก requirement อย่างเดียว ไม่ได้อ่านโค้ดหรือเอกสารของระบบ ดังนั้นรายละเอียดของ service ที่ place order ในปัจจุบัน วิธีสื่อสารระหว่าง service และตัวเลขด้าน load/latency คือ **Not evidenced in the provided files**

## Decision drivers

1. **Modifiability / extensibility:** เพิ่ม post-order action ใหม่ (SMS, fraud scoring, recommendations) ได้โดยแก้ service ที่ place order ให้น้อยที่สุด (มาจาก requirement โดยตรง)
2. **Availability ของการสั่งซื้อ:** การสั่งซื้อไม่ควรล้มเหลวหรือช้าลงเพราะ MailFast, `loyaltyservice` หรือ analytics ล่ม **[Assumption]** ว่างานทั้งสามไม่จำเป็นต้องเสร็จก่อนตอบลูกค้า
3. **Reliability:** event ต้องไม่หาย ลูกค้าต้องได้อีเมลและ points ครบ และ points ต้องไม่ถูกให้ซ้ำ
4. **Performance (latency ของการสั่งซื้อ):** ไม่เพิ่ม latency จากการเรียกระบบภายนอก (ยังไม่มีตัวเลข target ดู Open questions)
5. **Operability / cost:** จำนวน infrastructure ใหม่ที่ทีมต้องดูแล

## Considered options

1. **A: Synchronous orchestration:** service ที่ place order เรียก MailFast (ผ่าน `emailservice`), `loyaltyservice` และ analytics ตามลำดับ ภายใน request เดียวกัน
2. **B: Event-driven publish/subscribe:** service ที่ place order publish event `OrderPlaced` ไปยัง message broker หนึ่งครั้ง แล้ว consumer แต่ละตัว (`emailservice` → MailFast, `loyaltyservice`, analytics) subscribe แยกกันเอง
3. **C: Workflow orchestrator:** service ที่ place order เริ่ม workflow "post-order" ใน workflow engine ซึ่งเรียกแต่ละขั้นตอนแบบ async พร้อม retry และเก็บสถานะ

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A: Synchronous orchestration | เรียบง่าย ไม่มี infrastructure ใหม่ · debug/trace ง่าย · รู้ผลทันทีว่าสำเร็จหรือไม่ | latency ของการสั่งซื้อ = ผลรวมของทุก call · MailFast ล่ม = สั่งซื้อล้มหรือต้องเขียน error handling เอง · เพิ่ม action ใหม่ = ต้องแก้และ deploy service ที่ place order ทุกครั้ง (coupling สูง) |
| B: Event-driven pub/sub | ผู้ publish ไม่ต้องรู้จัก consumer, เพิ่ม SMS/fraud/recommendations = เพิ่ม subscriber ใหม่ · แยกความล้มเหลว: consumer ล่มไม่กระทบการสั่งซื้อ · consumer retry และ scale ได้อิสระ · analytics consume event เดียวกันได้โดยตรง | ต้องดูแล message broker ใหม่ · eventual consistency (อีเมล/points มาช้ากว่าการสั่งซื้อ) · ต้องจัดการ duplicate (idempotency) และ dead-letter · ภาพรวม flow อ่านยากขึ้น ต้องมี tracing |
| C: Workflow orchestrator | เห็นสถานะทุกขั้นที่เดียว · retry/timeout/compensation มีให้ในตัว · เหมาะถ้า fraud scoring ต้องมาก่อนขั้นอื่น (มีลำดับ) | infrastructure หนักที่สุดและทีมต้องเรียนรู้ engine ใหม่ · workflow definition ยังรวมศูนย์ เพิ่ม action ใหม่ = แก้ workflow (coupling ปานกลาง) · overkill สำหรับงาน fire-and-forget ที่ไม่ขึ้นต่อกัน |

## Decision

เลือก **Option B: Event-driven publish/subscribe**

- service ที่ place order publish event `OrderPlaced` **หนึ่งครั้ง** หลังการสั่งซื้อสำเร็จ (เช่น order id, ลูกค้า, อีเมล, รายการสินค้า, ยอดรวม) แล้วตอบลูกค้าทันทีโดยไม่รอ post-order actions
- `emailservice` เปลี่ยนจาก mock เป็น adapter ที่ consume `OrderPlaced` แล้วเรียก MailFast
- `loyaltyservice` (ใหม่) consume `OrderPlaced` แล้วให้ points แบบ idempotent (ใช้ order id เป็น key)
- analytics consumer consume `OrderPlaced` แล้วบันทึก event "order placed" ให้ marketing
- action ในอนาคต (SMS, fraud scoring, recommendations) = เพิ่ม subscriber ใหม่ โดยไม่แก้ผู้ publish

เหตุผล: driver อันดับ 1 (modifiability) และ 2 (availability) เป็นสิ่งที่ B ตอบได้ดีที่สุด เพราะผู้ publish ไม่ผูกกับ consumer และความล้มเหลวของ MailFast ถูกแยกออกจากการสั่งซื้อ ส่วน A ไม่ตอบทั้งสองข้อ และ C ให้การควบคุมที่ยังไม่จำเป็นสำหรับงานที่ไม่ขึ้นต่อกัน แลกกับความซับซ้อนที่สูงกว่า

ยังไม่เลือกเทคโนโลยี broker (เช่น RabbitMQ, Kafka, Google Cloud Pub/Sub) เพราะขึ้นกับข้อมูลที่ยังไม่มี (ดู Open questions)

## Consequences

- **Positive:** เพิ่ม post-order action ใหม่ได้ด้วยการเพิ่ม subscriber โดยไม่แก้หรือ deploy service ที่ place order (quality attribute: modifiability)
- **Positive:** MailFast, `loyaltyservice` หรือ analytics ล่มแล้วการสั่งซื้อยังทำงานได้ งานค้างจะถูกประมวลผลเมื่อกลับมา (quality attribute: availability)
- **Positive:** latency ของการสั่งซื้อไม่ขึ้นกับระบบภายนอก เหลือแค่เวลา publish event (quality attribute: performance)
- **Positive:** consumer แต่ละตัว scale และ deploy ได้อิสระ (quality attribute: scalability, deployability)
- **Negative:** อีเมลยืนยันและ points มาถึงช้ากว่าการสั่งซื้อ (eventual consistency) (quality attribute: consistency / usability)
- **Negative:** มี message broker เพิ่ม ต้อง deploy, monitor และดูแล (quality attribute: cost, operability)
- **Negative:** flow กระจายหลาย service ทำให้ trace และ debug ยากขึ้น (quality attribute: observability / testability)
- **Risks and mitigations:**
  - *Event หายถ้า publish ล้มเหลวหลังบันทึก order* → พิจารณา transactional outbox หรือ retry ฝั่ง publisher
  - *Duplicate delivery ทำให้ให้ points ซ้ำ / ส่งอีเมลซ้ำ* → consumer ต้อง idempotent โดยใช้ order id
  - *MailFast ล่มหรือ rate-limit* → retry with backoff และ dead-letter queue พร้อม alert
  - *Schema ของ `OrderPlaced` เปลี่ยนแล้วทำให้ consumer พัง* → กำหนด schema versioning (backward-compatible)
  - *ข้อมูลส่วนบุคคล (อีเมล) อยู่ใน event ที่หลาย consumer อ่านได้* → จำกัดว่า field ไหนอยู่ใน event และใครอ่านได้

## Validation

- **Load test:** วัด latency ของการสั่งซื้อก่อนและหลังเปลี่ยน ต้องไม่แย่ลง (target ยังไม่มี ดู Open questions)
- **Failure test (chaos):** ปิด MailFast (หรือ mock), `loyaltyservice` และ analytics consumer แล้วยืนยันว่าการสั่งซื้อยังสำเร็จ และงานค้างถูกประมวลผลครบเมื่อเปิดกลับมา
- **Metrics:** consumer lag ต่อ subscriber, จำนวน message ใน dead-letter queue, อัตราสำเร็จของการส่งอีเมลผ่าน MailFast, เวลาตั้งแต่สั่งซื้อจนอีเมลถึง
- **Idempotency test:** ส่ง `OrderPlaced` เดิมซ้ำ แล้วยืนยันว่า points ไม่เพิ่มซ้ำ
- **Modifiability check:** เมื่อเพิ่ม action แรกของปีหน้า (เช่น SMS) ยืนยันว่าไม่ต้องแก้ service ที่ place order
- **Review date:** ยังไม่กำหนด (ดู Open questions)

## Open questions

- service ไหนใน Online Boutique เป็นผู้ place order และปัจจุบันเรียก `emailservice` อย่างไร (protocol, sync/async)? (Not evidenced in the provided files)
- มี message broker ในระบบอยู่แล้วหรือไม่? ทีมมีความรู้กับ broker ตัวไหน? (ใช้เลือกเทคโนโลยี)
- target latency และปริมาณคำสั่งซื้อ (orders/sec, peak) คือเท่าไร?
- อีเมลยืนยันต้องถึงลูกค้าภายในกี่วินาที/นาที? delay จาก eventual consistency รับได้แค่ไหน?
- MailFast มี API แบบไหน, rate limit, SLA, ค่าใช้จ่ายต่ออีเมล และรองรับ idempotency key หรือไม่?
- กฎการให้ loyalty points คืออะไร (สูตร, การยกเลิก/คืนสินค้าต้องหัก points หรือไม่)?
- analytics event ส่งไปที่ระบบไหน (ภายในหรือภายนอก) และ marketing ต้องการ field อะไร?
- fraud scoring ในอนาคตต้อง **block** การสั่งซื้อหรือไม่? ถ้าใช่ จะไม่ใช่ post-order action แบบ async และอาจต้องใช้ Option C หรือ sync call แยก
- ข้อกำหนดด้าน privacy (PDPA/GDPR) สำหรับข้อมูลลูกค้าใน event และการส่งให้ MailFast?
- Deciders และ review date คือใคร/เมื่อไร?
