# Business Worksheet Skill

Skill สำหรับให้ Claude ประกอบ **JSON payload กระดาษทำการงบการเงิน** โดยอัตโนมัติ จาก HTML ตัวอย่างงบการเงินและผังบัญชี CSV

---

## ภาพรวม

Skill นี้ช่วยให้ Claude สามารถ:
- วิเคราะห์โครงสร้าง HTML งบการเงิน (งบกำไรขาดทุน, งบกำไรสะสม, งบดุล)
- Map รายการบัญชีใน HTML กับรหัสบัญชีใน CSV ผังบัญชี
- สร้าง `variables[]` ทั้ง Account Variable และ Formula Variable อย่างถูกต้อง
- คำนวณ Consolidated formula ตามหลัก `is_credit` ของแต่ละบัญชี
- ประกอบ `template_html` พร้อม placeholder และส่งออกเป็น JSON payload สำหรับ `POST /business-worksheet/template`

---

## วิธีติดตั้ง

นำไฟล์ `SKILL.md` วางไว้ใน skill directory ของระบบ เช่น

```
/mnt/skills/user/business-worksheet/SKILL.md
```

จากนั้น Claude จะ trigger skill นี้โดยอัตโนมัติเมื่อ user พูดถึงกระดาษทำการหรือส่ง HTML + CSV เข้ามา

---

## วิธีใช้งาน

### 1. เตรียม Input

**HTML งบการเงิน** — ไฟล์ `.html` ที่มีโครงสร้างตาราง เช่น

```html
<table>
  <tr>
    <td>รายการ</td>
    <td>บริษัท A</td>
    <td>บริษัท B</td>
    <td>รายการปรับปรุง เดบิต</td>
    <td>รายการปรับปรุง เครดิต</td>
    <td>งบการเงินรวม</td>
  </tr>
  <tr>
    <td>เงินสดและเงินฝากธนาคาร</td>
    <td>1,250,000.00</td>
    <td>850,000.00</td>
    <td></td><td></td><td></td>
  </tr>
</table>
```

**CSV ผังบัญชี** — ไฟล์ `.csv` ที่มีอย่างน้อย 3 columns:

```csv
account_code,name_thai,is_credit
111000,เงินสด,f
113000,เงินฝากธนาคาร,f
410001,รายได้จากการขาย,t
```

| column | คำอธิบาย |
|---|---|
| `account_code` | รหัสบัญชี |
| `name_thai` | ชื่อบัญชีภาษาไทย |
| `is_credit` | `t` = บัญชี credit normal, `f` = บัญชี debit normal |

---

### 2. ส่งคำสั่งให้ Claude

```
ฉันต้องการสร้าง business worksheet payload จากไฟล์นี้
[แนบ HTML + CSV]
```

Claude จะอ่านทั้งสองไฟล์ แล้วถามข้อมูลเพิ่มเติมที่จำเป็นก่อน generate เช่น:

- **Branch codes** — รหัสสาขาที่ใช้ filter เช่น `B0001`, `B0002`
- **บัญชีที่ map ไม่ได้** — label ใน HTML ที่ไม่มีรหัสตรงกันใน CSV
- **บัญชีที่ปล่อยว่าง** — รายการที่ยังไม่มีข้อมูล

---

### 3. ตรวจสอบและรับ Output

Claude จะส่งออก JSON payload พร้อมใช้ POST ไปยัง API:

```json
{
  "name": "กระดาษทำการ",
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
      "name": "%cash_bank_total%",
      "formula": "%cash_bank_b0001% + %cash_bank_b0002% + %cash_bank_adj_d% - %cash_bank_adj_c%",
      "sort_order": 2001,
      "format": "0,0.00;(0,0.00);\"-\""
    }
  ],
  "tabs": [
    {
      "tab_name": "กระดาษทำการ",
      "sort_order": 1,
      "template_html": "<table>...</table>"
    }
  ]
}
```

---

## หลักการสำคัญที่ Skill ใช้

### Column ตามประเภทงบ

| งบ | column |
|---|---|
| งบกำไรขาดทุน (ตัวเลขประจำงวด) | `period_balance` |
| งบดุล / งบกำไรสะสม (ยอดคงเหลือ) | `net_carried_forward` |
| ยอดยกมา (ต้นงวด) | `brought_forward_debit` / `brought_forward_credit` |
| ยอดยกไป (ปลายงวด) | `carried_forward_debit` / `carried_forward_credit` |
| รายการปรับปรุง | `journal_adjustment_debit` / `journal_adjustment_credit` |

### Consolidated Formula ตาม is_credit

```
is_credit = f (สินทรัพย์, ค่าใช้จ่าย):
  total = branch1 + branch2 + ... + adj_debit - adj_credit

is_credit = t (หนี้สิน, ทุน, รายได้):
  total = branch1 + branch2 + ... + adj_credit - adj_debit
```

### sort_order

| ประเภท | ช่วง |
|---|---|
| Account Variables | 1 – 999 |
| Formula Variables ชั้นแรก (per-branch subtotal) | 1000+ |
| Formula Variables consolidated (_total) | 2000+ |

---

## โครงสร้างไฟล์

```
business-worksheet-design-skill/
├── README.md   — คำอธิบายและวิธีใช้งาน (ไฟล์นี้)
└── SKILL.md    — Skill instructions สำหรับ Claude
```

---

## API Endpoint ที่เกี่ยวข้อง

| Method | Path | คำอธิบาย |
|---|---|---|
| `POST` | `/business-worksheet/template` | สร้าง template ใหม่ด้วย payload ที่ได้ |
| `PUT` | `/business-worksheet/template/:id` | แก้ไข template (replace ทั้งหมด) |
| `POST` | `/business-worksheet/render` | Render กระดาษทำการด้วย template ที่สร้าง |
