---
name: business-worksheet
description: |
  ใช้ skill นี้เมื่อ user ต้องการสร้าง JSON payload สำหรับ Business Worksheet (กระดาษทำการงบการเงิน)
  โดยรับ HTML ตัวอย่างงบการเงินและผังบัญชี CSV แล้วประกอบเป็น payload พร้อม POST ไปยัง API

  Trigger เมื่อ user:
  - ส่ง HTML งบการเงิน + ผังบัญชี CSV แล้วขอสร้าง worksheet / template / payload
  - พูดถึง "กระดาษทำการ", "business worksheet", "worksheet template"
  - ต้องการ variables[], tabs[], หรือ JSON payload สำหรับ business worksheet API
---

# Business Worksheet — Payload Builder

รับ HTML งบการเงิน + ผังบัญชี CSV แล้วสร้าง JSON payload สำหรับ `POST /business-worksheet/template`

---

## ขั้นตอนการทำงาน

### Step 1 — อ่านและทำความเข้าใจ Input

**HTML:**
- ตรวจสอบว่ามี tag `<tab>...</tab>` หรือไม่ เพื่อแบ่ง tab (ดูรายละเอียด Step 1.1)
- ระบุ cell/placeholder ทุกจุดที่ต้องการแสดงตัวเลข
- สังเกต label ข้างๆ เพื่ออนุมานว่า cell นั้นคือบัญชีอะไร
- หา pattern ที่อาจเป็น Formula (เช่น "รายได้รวม", "กำไรสุทธิ", "รวมสินทรัพย์")
- สังเกตว่ามีคอลัมน์ **งบการเงินรวม** หรือไม่ — ถ้ามีต้องสร้าง `_total` formula variable ทุกรายการ

**CSV ผังบัญชี:**
- ระบุ columns ที่มี: รหัสบัญชี, ชื่อบัญชี, `is_credit`
- Map ชื่อบัญชีจาก HTML → รหัสบัญชีใน CSV
- บันทึก `is_credit` ของแต่ละบัญชีไว้ใช้คำนวณ consolidated formula

---

### Step 1.1 — แยก Tab จาก HTML

#### กรณีมี `<tab>` tag

User ระบุ tab โดยครอบเนื้อหาแต่ละ tab ด้วย tag `<tab name="ชื่อ tab">`:

```html
<tab name="งบกำไรขาดทุน">
  <table>
    <tr><td>รายได้จากการขาย</td>...</tr>
  </table>
</tab>

<tab name="งบดุล">
  <table>
    <tr><td>เงินสดและเงินฝากธนาคาร</td>...</tr>
  </table>
</tab>
```

- แต่ละ `<tab>` → 1 entry ใน `tabs[]`
- `tab_name` ดึงจาก attribute `name`
- `sort_order` เรียงตามลำดับที่ปรากฏใน HTML (1, 2, 3...)
- `template_html` คือ HTML ข้างใน `<tab>` (ไม่รวม tag `<tab>` เอง)

#### กรณีไม่มี `<tab>` tag

ถือว่ามี **1 tab** — ใช้ HTML ทั้งหมดเป็น `template_html` และตั้ง `tab_name` ตามชื่อ template หรือถาม user

#### ตัวอย่าง tabs[] output

```json
"tabs": [
  {
    "tab_name": "งบกำไรขาดทุน",
    "sort_order": 1,
    "template_html": "<table>...</table>"
  },
  {
    "tab_name": "งบดุล",
    "sort_order": 2,
    "template_html": "<table>...</table>"
  }
]
```

#### กฎสำคัญ — Variables ใช้ร่วมกันทุก Tab

`variables[]` เป็น **ระดับ template** — ตัวแปรทุกตัวจากทุก tab อยู่ใน array เดียวกัน ไม่แยกตาม tab

- **ใช้ซ้ำข้าม tab ได้** — หาก tab อื่นต้องการแสดงตัวแปรเดิม (เช่น `%net_profit_b0001%` ที่นิยามใน tab งบกำไรขาดทุน) ให้ใช้ชื่อเดิมใน `template_html` ของ tab นั้นได้เลย **ไม่ต้องนิยามซ้ำใน `variables[]`**
- **ห้ามนิยาม variable ชื่อเดิมซ้ำ** — `variables[]` ต้องมีชื่อ variable ไม่ซ้ำกันเลย หากต้องการค่าเดิมในต่าง tab ให้อ้างชื่อเดิมใน `template_html` เท่านั้น

---

### Step 2 — ถามข้อมูลที่จำเป็นก่อน generate

**ถามเสมอเมื่อยังไม่ได้รับข้อมูล:**
1. **Branch codes** — รหัสสาขาที่ใช้ใน `branch_code` field (เช่น B0001, B0002...) และ mapping กับชื่อบริษัท
2. **บัญชีที่หา mapping ไม่ได้** — label ใน HTML ที่ไม่มีรหัสบัญชีตรงกันใน CSV
3. **บัญชีที่ใช้ range** — ยืนยันว่า range เช่น `521000:521999` ครอบคลุมถูกต้อง
4. **บัญชีที่ปล่อยว่าง** — รายการที่ยังไม่มีข้อมูล ให้ปล่อย `<td></td>` ว่างไว้ก่อน

**อย่าเดาและ generate ทันทีเมื่อ:**
- ผังบัญชีใน CSV ไม่มีรหัสที่ตรงกับ label ใน HTML
- ไม่รู้ branch code ที่ถูกต้อง

---

### Step 3 — Map HTML → Variables

#### Pattern: Multi-branch + Adjustment + Consolidated

สำหรับตารางที่มีหลายสาขา แต่ละรายการต้องสร้างตัวแปรชุดนี้:

```
%prefix_b0001%  — Account Variable, branch_code: "B0001"
%prefix_b0002%  — Account Variable, branch_code: "B0002"
...
%prefix_adj_d%  — Account Variable, column: "journal_adjustment_debit"  (ไม่มี branch_code)
%prefix_adj_c%  — Account Variable, column: "journal_adjustment_credit" (ไม่มี branch_code)
%prefix_total%  — Formula Variable, คำนวณจาก branch + adj (ดู Step 5)
```

#### เลือก `column` ตามประเภทและส่วนของงบ

column แบ่งเป็น 3 กลุ่ม — เลือกให้ตรงกับ data source และ branch filter ที่ต้องการ

**กลุ่ม Summary** — ดึงจาก `gl_journal_summary_detail` รองรับ `branch_code` filter

| column | คำอธิบาย | ใช้เมื่อ |
|---|---|---|
| `brought_forward_debit` | ยอดยกมา debit | ต้องการยอดยกมาแบบแยก debit |
| `brought_forward_credit` | ยอดยกมา credit | ต้องการยอดยกมาแบบแยก credit |
| `debit` | เดบิตระหว่างงวด | ต้องการยอดเดบิตประจำงวดดิบ |
| `credit` | เครดิตระหว่างงวด | ต้องการยอดเครดิตประจำงวดดิบ |
| `close_account_debit` | ปิดบัญชี debit | รายการปิดบัญชีแบบ debit |
| `close_account_credit` | ปิดบัญชี credit | รายการปิดบัญชีแบบ credit |
| `carried_forward_debit` | ยอดยกไป debit (Summary + Journal รวมกัน) | ยอดยกไปแบบแยก debit |
| `carried_forward_credit` | ยอดยกไป credit (Summary + Journal รวมกัน) | ยอดยกไปแบบแยก credit |
| `net_carried_forward` | `carried_forward_debit` − `carried_forward_credit` | ยอดคงเหลือสุทธิ (งบดุล) |

**กลุ่ม Journal** — ดึงจาก `gl_journal_detail` **ไม่รองรับ** `branch_code` filter

| column | คำอธิบาย | ใช้เมื่อ |
|---|---|---|
| `journal_adjustment_debit` | ยอดปรับปรุง debit จาก journal (รวมทุกสาขา) | คอลัมน์ปรับปรุง debit |
| `journal_adjustment_credit` | ยอดปรับปรุง credit จาก journal (รวมทุกสาขา) | คอลัมน์ปรับปรุง credit |

**กลุ่ม Balance (net)** — คำนวณ net ตาม `is_credit` ของบัญชี: credit-normal = credit − debit, debit-normal = debit − credit

| column | แหล่งข้อมูล | `branch_code` | คำอธิบาย | ใช้เมื่อ |
|---|---|---|---|---|
| `brought_forward_balance` | Summary | รองรับ | ยอดยกมา net | ต้องการยอดยกมาสุทธิ (ต้นงวด) |
| `period_balance` | Summary | รองรับ | ยอดระหว่างงวด net | **งบกำไรขาดทุน** (ตัวเลขประจำงวด) |
| `close_account_balance` | Summary | รองรับ | ยอดปิดบัญชี net | รายการปิดบัญชีสุทธิ |
| `carried_forward_balance` | Summary + Journal | รองรับ | ยอดยกไป net (รวม journal) | **งบดุล / งบกำไรสะสม** (ยอดคงเหลือ) |
| `journal_adjustment_balance` | Journal | ไม่รองรับ | ยอดปรับปรุง net จาก journal | คอลัมน์ปรับปรุงสุทธิ |

**สรุปการเลือก column ที่แนะนำตามงบ:**

| งบ / ประเภทข้อมูล | column แนะนำ |
|---|---|
| งบกำไรขาดทุน (ตัวเลขประจำงวด) | `period_balance` |
| งบดุล / งบกำไรสะสม (ยอดคงเหลือ) | `carried_forward_balance` หรือ `net_carried_forward` |
| ยอดยกมาต้นงวด (net) | `brought_forward_balance` |
| ยอดยกมาต้นงวด (แยก debit/credit) | `brought_forward_debit` / `brought_forward_credit` |
| ยอดยกไปปลายงวด (แยก debit/credit) | `carried_forward_debit` / `carried_forward_credit` |
| รายการปรับปรุง (แยก debit/credit) | `journal_adjustment_debit` / `journal_adjustment_credit` |
| รายการปรับปรุง (net) | `journal_adjustment_balance` |

#### รูปแบบ `account`

| รูปแบบ | ตัวอย่าง | ความหมาย |
|---|---|---|
| Range | `"521000:521999"` | ทุกบัญชีในช่วง |
| Exact | `"410001"` | บัญชีเดียว |
| ผสม | `"111000,113000"` | หลาย exact |
| Suffix sub-account | `"140011-140011-1"` | sub-account เฉพาะ |

---

### Step 4 — sort_order

| ประเภท | ช่วง sort_order แนะนำ |
|---|---|
| Account Variables (branch B0001–B000x) | 1 – 999 (เพิ่มทีละ 1 ต่อ branch) |
| Account Variables (adj_d, adj_c) | ต่อเนื่องจาก branch |
| Formula Variables ชั้นแรก (per-branch subtotal) | 1000+ |
| Formula Variables ชั้นสอง (consolidated _total) | 2000+ |

**กฎเหล็ก:** Formula Variable ต้องมี `sort_order` สูงกว่าทุกตัวแปรที่อ้างถึงเสมอ

---

### Step 5 — Consolidated Formula (_total)

สร้าง `%prefix_total%` สำหรับคอลัมน์งบการเงินรวม โดยดู `is_credit` จาก CSV:

```
is_credit = false (สินทรัพย์, ค่าใช้จ่าย):
  %prefix_total% = branch1 + branch2 + ... + branchN + %prefix_adj_d% - %prefix_adj_c%

is_credit = true (หนี้สิน, ทุน, รายได้):
  %prefix_total% = branch1 + branch2 + ... + branchN + %prefix_adj_c% - %prefix_adj_d%
```

**วิธี resolve `is_credit` จาก CSV เมื่อ account เป็น range:**
- ดู is_credit ของ account code แรกในช่วง (start of range)
- หากทุก account ใน range มี is_credit เดียวกัน → ใช้ค่านั้น
- หากไม่แน่ใจ → ถาม user

---

### Step 6 — ประกอบ Payload

```json
{
  "name": "<ชื่อ template>",
  "is_default": false,
  "variables": [
    {
      "name": "%cash_bank_b0001%",
      "account": "111000,113000",
      "column": "net_carried_forward",
      "branch_code": "B0001",
      "sort_order": 1,
      "format": "0,0.00;(0,0.00);\"-\""
    },
    {
      "name": "%cash_bank_adj_d%",
      "account": "111000,113000",
      "column": "journal_adjustment_debit",
      "sort_order": 6,
      "format": "0,0.00;(0,0.00);\"-\""
    },
    {
      "name": "%cash_bank_adj_c%",
      "account": "111000,113000",
      "column": "journal_adjustment_credit",
      "sort_order": 7,
      "format": "0,0.00;(0,0.00);\"-\""
    },
    {
      "name": "%cash_bank_total%",
      "formula": "%cash_bank_b0001% + %cash_bank_b0002% + %cash_bank_b0003% + %cash_bank_b0004% + %cash_bank_b0005% + %cash_bank_adj_d% - %cash_bank_adj_c%",
      "sort_order": 2001,
      "format": "0,0.00;(0,0.00);\"-\""
    }
  ],
  "tabs": [
    {
      "tab_name": "งบกำไรขาดทุน",
      "sort_order": 1,
      "template_html": "<table>...</table>"
    },
    {
      "tab_name": "งบดุล",
      "sort_order": 2,
      "template_html": "<table>...</table>"
    }
  ]
}
```

**`format` default:** `"0,0.00;(0,0.00);\"-\""` (ทศนิยม 2 ตำแหน่ง, ลบในวงเล็บ, ศูนย์แสดงเป็น "-")

---

### Step 7 — ตรวจสอบก่อนส่ง

- [ ] ทุก `%variable%` ใน `template_html` ทุก tab มีนิยามใน `variables[]`
- [ ] ไม่มี variable ที่นิยามซ้ำกันใน `variables[]` (ชื่อต้อง unique ทั้ง template)
- [ ] variable ที่ใช้ซ้ำข้าม tab — อ้างชื่อเดิมใน `template_html` เท่านั้น ไม่เพิ่มใน `variables[]`
- [ ] `sort_order` ของ Formula Variable > ตัวแปรที่อ้างถึงทุกตัว
- [ ] `adj_d` / `adj_c` ไม่มี `branch_code` field
- [ ] `_total` formula ใช้ is_credit ถูกต้อง (บวก adj_d หรือ adj_c)
- [ ] ไม่มีตัวแปรตัวเดียวกันที่ระบุทั้ง `account` และ `formula`
- [ ] `tabs[]` เรียงตาม `sort_order` ตามลำดับที่ปรากฏใน HTML

---

## การจัดการกรณีไม่ชัดเจน

**ถาม user เมื่อ:**
- Map บัญชีจาก HTML ไปยัง CSV ไม่ได้ (ชื่อไม่ตรง หรือหาไม่เจอ)
- ไม่แน่ใจว่า cell ควรเป็น Account Variable หรือ Formula Variable
- ไม่รู้ว่าควรใช้ `column` ไหน
- ไม่แน่ใจ `is_credit` ของบัญชีที่ใช้ range
- ไม่แน่ใจสูตรของ Formula Variable (subtotal / consolidated)
- บัญชีใน HTML ไม่มีใน CSV เลย
- ไม่มี `<tab>` tag และไม่ชัดเจนว่า tab ควรชื่ออะไร

**ปล่อยว่างได้เมื่อ:**
- user ระบุชัดว่ารายการนั้นยังไม่มีข้อมูล ให้ `<td></td>` ว่างไว้ก่อน

---

## ตัวอย่าง

### Input HTML พร้อม tab tags
```html
<tab name="งบกำไรขาดทุน">
  <table>
    <tr><td>รายได้จากการขาย</td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
    <tr><td>กำไรสุทธิ</td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  </table>
</tab>

<tab name="งบดุล">
  <table>
    <tr><td>เงินสดและเงินฝากธนาคาร</td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  </table>
</tab>
```

### Input CSV (บางส่วน)
```
account_code,name_thai,is_credit
111000,เงินสด,f
113000,เงินฝากธนาคาร,f
410001,รายได้จากการขาย,t
```

### Output (บางส่วน)
```json
{
  "name": "กระดาษทำการ",
  "is_default": false,
  "variables": [
    { "name": "%rev_sale_b0001%", "account": "410001", "column": "period_balance", "branch_code": "B0001", "sort_order": 1, "format": "0,0.00;(0,0.00);\"-\"" },
    { "name": "%rev_sale_adj_d%", "account": "410001", "column": "journal_adjustment_debit", "sort_order": 6, "format": "0,0.00;(0,0.00);\"-\"" },
    { "name": "%rev_sale_adj_c%", "account": "410001", "column": "journal_adjustment_credit", "sort_order": 7, "format": "0,0.00;(0,0.00);\"-\"" },
    { "name": "%rev_sale_total%", "formula": "%rev_sale_b0001% + %rev_sale_b0002% + %rev_sale_adj_c% - %rev_sale_adj_d%", "sort_order": 2001, "format": "0,0.00;(0,0.00);\"-\"" },
    { "name": "%cash_bank_b0001%", "account": "111000,113000", "column": "net_carried_forward", "branch_code": "B0001", "sort_order": 50, "format": "0,0.00;(0,0.00);\"-\"" },
    { "name": "%cash_bank_total%", "formula": "%cash_bank_b0001% + %cash_bank_b0002% + %cash_bank_adj_d% - %cash_bank_adj_c%", "sort_order": 2050, "format": "0,0.00;(0,0.00);\"-\"" }
  ],
  "tabs": [
    {
      "tab_name": "งบกำไรขาดทุน",
      "sort_order": 1,
      "template_html": "<table><tr><td>รายได้จากการขาย</td><td>%rev_sale_b0001%</td>...<td>%rev_sale_total%</td></tr></table>"
    },
    {
      "tab_name": "งบดุล",
      "sort_order": 2,
      "template_html": "<table><tr><td>เงินสดและเงินฝากธนาคาร</td><td>%cash_bank_b0001%</td>...<td>%cash_bank_total%</td></tr></table>"
    }
  ]
}
```
