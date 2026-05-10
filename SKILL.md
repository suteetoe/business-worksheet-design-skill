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
- ระบุ cell/placeholder ทุกจุดที่ต้องการแสดงตัวเลข
- สังเกต label ข้างๆ เพื่ออนุมานว่า cell นั้นคือบัญชีอะไร
- หา pattern ที่อาจเป็น Formula (เช่น "รายได้รวม", "กำไรสุทธิ", "รวมสินทรัพย์")
- สังเกตว่ามีคอลัมน์ **งบการเงินรวม** หรือไม่ — ถ้ามีต้องสร้าง `_total` formula variable ทุกรายการ

**CSV ผังบัญชี:**
- ระบุ columns ที่มี: รหัสบัญชี, ชื่อบัญชี, `is_credit`
- Map ชื่อบัญชีจาก HTML → รหัสบัญชีใน CSV
- บันทึก `is_credit` ของแต่ละบัญชีไว้ใช้คำนวณ consolidated formula

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

| งบ / ประเภทข้อมูล | column |
|---|---|
| งบกำไรขาดทุน (ตัวเลขประจำงวด) | `period_balance` |
| งบดุล / งบกำไรสะสม (ยอดคงเหลือ) | `net_carried_forward` |
| ยอดยกมา (สินค้าคงเหลือต้นงวด, กำไรสะสมต้นงวด) | `brought_forward_debit` หรือ `brought_forward_credit` |
| ยอดยกไป (สินค้าคงเหลือปลายงวด) | `carried_forward_debit` หรือ `carried_forward_credit` |
| รายการปรับปรุง (adj columns) | `journal_adjustment_debit` หรือ `journal_adjustment_credit` |

> **หมายเหตุ:** `net_carried_forward` = carried_forward_debit − carried_forward_credit

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
      "tab_name": "<ชื่อ tab>",
      "sort_order": 1,
      "template_html": "<table>...</table>"
    }
  ]
}
```

**`format` default:** `"0,0.00;(0,0.00);\"-\""` (ทศนิยม 2 ตำแหน่ง, ลบในวงเล็บ, ศูนย์แสดงเป็น "-")

---

### Step 7 — ตรวจสอบก่อนส่ง

- [ ] ทุก `%variable%` ใน `template_html` มีนิยามใน `variables[]`
- [ ] ไม่มี variable ที่นิยามซ้ำกัน
- [ ] `sort_order` ของ Formula Variable > ตัวแปรที่อ้างถึงทุกตัว
- [ ] `adj_d` / `adj_c` ไม่มี `branch_code` field
- [ ] `_total` formula ใช้ is_credit ถูกต้อง (บวก adj_d หรือ adj_c)
- [ ] ไม่มีตัวแปรตัวเดียวกันที่ระบุทั้ง `account` และ `formula`

---

## การจัดการกรณีไม่ชัดเจน

**ถาม user เมื่อ:**
- Map บัญชีจาก HTML ไปยัง CSV ไม่ได้ (ชื่อไม่ตรง หรือหาไม่เจอ)
- ไม่แน่ใจว่า cell ควรเป็น Account Variable หรือ Formula Variable
- ไม่รู้ว่าควรใช้ `column` ไหน
- ไม่แน่ใจ `is_credit` ของบัญชีที่ใช้ range
- ไม่แน่ใจสูตรของ Formula Variable (subtotal / consolidated)
- บัญชีใน HTML ไม่มีใน CSV เลย

**ปล่อยว่างได้เมื่อ:**
- user ระบุชัดว่ารายการนั้นยังไม่มีข้อมูล ให้ `<td></td>` ว่างไว้ก่อน

---

## ตัวอย่าง

### Input HTML (บางส่วน)
```html
<tr><td>เงินสดและเงินฝากธนาคาร</td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
<tr><td>กำไรสุทธิ</td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
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
    { "name": "%cash_bank_b0001%", "account": "111000,113000", "column": "net_carried_forward", "branch_code": "B0001", "sort_order": 1, "format": "0,0.00;(0,0.00);\"-\"" },
    { "name": "%cash_bank_b0002%", "account": "111000,113000", "column": "net_carried_forward", "branch_code": "B0002", "sort_order": 2, "format": "0,0.00;(0,0.00);\"-\"" },
    { "name": "%cash_bank_adj_d%", "account": "111000,113000", "column": "journal_adjustment_debit",  "sort_order": 6, "format": "0,0.00;(0,0.00);\"-\"" },
    { "name": "%cash_bank_adj_c%", "account": "111000,113000", "column": "journal_adjustment_credit", "sort_order": 7, "format": "0,0.00;(0,0.00);\"-\"" },
    { "name": "%cash_bank_total%", "formula": "%cash_bank_b0001% + %cash_bank_b0002% + %cash_bank_adj_d% - %cash_bank_adj_c%", "sort_order": 2001, "format": "0,0.00;(0,0.00);\"-\"" },
    { "name": "%net_profit_b0001%", "formula": "%total_rev_b0001% - %total_cost_b0001% - %sell_admin_exp_b0001%", "sort_order": 1100, "format": "0,0.00;(0,0.00);\"-\"" },
    { "name": "%net_profit_total%",  "formula": "%net_profit_b0001% + %net_profit_b0002%", "sort_order": 2100, "format": "0,0.00;(0,0.00);\"-\"" }
  ],
  "tabs": [
    {
      "tab_name": "กระดาษทำการ",
      "sort_order": 1,
      "template_html": "<tr><td>เงินสดและเงินฝากธนาคาร</td><td>%cash_bank_b0001%</td><td>%cash_bank_b0002%</td><td>%cash_bank_adj_d%</td><td>%cash_bank_adj_c%</td><td>%cash_bank_total%</td></tr>"
    }
  ]
}
```
