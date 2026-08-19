---

## 1. เครื่องมือที่ใช้

- Python
- Pandas
- SQLite

---

## 2. วิธีติดตั้ง

ก่อนรันโปรแกรมควรติดตั้ง Python ให้เรียบร้อยก่อน

จากนั้น Clone Repository

```bash
git clone https://github.com/67160337-lab/readytogo.git
cd readytogo
```

ติดตั้ง Library ที่ใช้ในโปรเจกต์

```bash
pip install pandas openpyxl
```

จากนั้นนำไฟล์ Dataset

```text
Python_Data_Pipeline_Lab_Dataset.xlsx
```

มาไว้ในโฟลเดอร์เดียวกับ `pipeline.py`

โครงสร้างไฟล์ควรเป็นประมาณนี้

```text
readytogo/
│
├── pipeline.py
├── Python_Data_Pipeline_Lab_Dataset.xlsx
├── retail_dw.db
├── quarantine.csv
└── pipeline_run_log.csv
```

---

## 3. วิธีรัน

เปิด Terminal ในโฟลเดอร์โปรเจกต์ แล้วใช้คำสั่ง

```bash
python pipeline.py
```

โปรแกรมจะอ่านข้อมูลจาก Excel ทั้ง 3 Batch ได้แก่

```text
orders_batch_1
orders_batch_2
orders_batch_3
```

จากนั้นโปรแกรมจะตรวจสอบข้อมูลก่อนนำเข้า Data Warehouse

ถ้าข้อมูลถูกต้อง → นำเข้า `fact_sales`

ถ้าข้อมูลไม่ถูกต้อง → นำไปเก็บใน `quarantine`

เมื่อทำงานเสร็จจะมีไฟล์ที่เกี่ยวข้องกับผลลัพธ์ เช่น

```text
retail_dw.db
quarantine.csv
pipeline_run_log.csv
```

---

## 4. การทำงานของ Pipeline

การทำงานของโปรแกรมสามารถสรุปง่าย ๆ ได้ดังนี้

```text
Excel
  │
  ▼
Extract
  │
  ▼
ตรวจสอบข้อมูล
  │
  ├── ข้อมูลถูกต้อง ──► Transform ──► fact_sales
  │
  └── ข้อมูลผิดพลาด ───────────────► quarantine
                                     
                     ▼
               pipeline_run_log
```

### Extract

โปรแกรมอ่านข้อมูลจาก Excel โดยใช้ Pandas และอ่านข้อมูลการขายทีละ Batch

มีทั้งหมด 3 Batch

- `orders_batch_1`
- `orders_batch_2`
- `orders_batch_3`

### Transform

ก่อนนำข้อมูลเข้า Fact Table โปรแกรมจะตรวจสอบข้อมูล เช่น

- Customer ID มีอยู่ใน Customer Dimension หรือไม่
- Product ID มีอยู่ใน Product Dimension หรือไม่
- วันที่สามารถอ่านได้หรือไม่
- Quantity ต้องมากกว่า 0
- Unit Price ต้องมากกว่า 0
- Discount ต้องอยู่ระหว่าง 0 ถึง 100

นอกจากนี้ยังคำนวณ

```text
gross_amount = quantity × unit_price

net_amount = gross_amount × (1 - discount / 100)
```

ถ้าข้อมูลไม่ผ่านการตรวจสอบ จะถูกส่งไปยัง `quarantine` พร้อมระบุ `reason_code` ว่าข้อมูลผิดเพราะอะไร

### Load

ข้อมูลที่ผ่านการตรวจสอบจะถูกนำไปเก็บใน `fact_sales`

และมีการสร้างข้อมูลวันที่ใน `dim_date` รวมถึงเชื่อมข้อมูล Customer และ Product ด้วย Key ของแต่ละ Dimension

---

# 5. Star Schema

โปรเจกต์นี้ใช้ **Star Schema** โดยมี `fact_sales` เป็นตารางหลัก และมี Dimension 3 ตาราง

```text
                    dim_customer
                         │
                         │
                         ▼
dim_product ───────► fact_sales ◄─────── dim_date
```

### Fact Table

#### `fact_sales`

เป็นตารางที่เก็บข้อมูลการขาย เช่น

- Order ID
- วันที่ขาย
- ลูกค้า
- สินค้า
- จำนวนสินค้า
- ราคาต่อหน่วย
- ส่วนลด
- ยอดขายก่อนหักส่วนลด
- ยอดขายสุทธิ
- วิธีชำระเงิน
- ช่องทางการขาย

`order_id` ถูกใช้เป็น Primary Key เพื่อป้องกัน Order เดิมถูกเพิ่มเข้าไปซ้ำ

---

### Dimension Table

#### `dim_customer`

เก็บข้อมูลลูกค้า

```text
customer_key
customer_id
customer_name
province
segment
```

#### `dim_product`

เก็บข้อมูลสินค้า

```text
product_key
product_id
product_name
category
```

#### `dim_date`

เก็บข้อมูลวันที่ เพื่อใช้วิเคราะห์ข้อมูลตามช่วงเวลา

```text
date_key
full_date
day
month
quarter
year
```

---

## 6. Quarantine

ข้อมูลที่ไม่ผ่านการตรวจสอบจะไม่ถูกนำเข้า `fact_sales`

แต่จะถูกเก็บไว้ในตาราง `quarantine`

ตัวอย่างเหตุผลที่ข้อมูลถูก Reject ได้แก่

```text
INVALID_CUSTOMER_FK
INVALID_PRODUCT_FK
INVALID_ORDER_DATE
INVALID_METRICS
```

ทำให้สามารถรู้ได้ว่าข้อมูลแต่ละรายการมีปัญหาจากอะไร

หลังจาก Pipeline ทำงาน ข้อมูลใน `quarantine` จะถูก Export ออกมาเป็น

```text
quarantine.csv
```

---

## 7. Pipeline Run Log

โปรแกรมมี `pipeline_run_log` สำหรับเก็บข้อมูลการทำงานของแต่ละ Batch

เช่น

- Batch ที่กำลังทำงาน
- เวลาเริ่มและจบ
- จำนวนข้อมูลที่อ่าน
- จำนวนข้อมูลที่ผ่าน
- จำนวนข้อมูลที่ Reject
- จำนวนข้อมูลที่ Load
- สถานะของการทำงาน

ผลลัพธ์จะถูกบันทึกไว้ใน

```text
pipeline_run_log.csv
```

ส่วนนี้ช่วยให้สามารถตรวจสอบได้ว่าแต่ละ Batch ทำงานไปกี่รายการ และมีข้อมูลถูก Reject เท่าไร

---

# 8. Acceptance Tests

โปรเจกต์นี้ตรวจสอบการทำงานตามเงื่อนไขที่กำหนดไว้ดังนี้

### 1. Pipeline ต้องรันครบ 3 Batch

โปรแกรมต้องสามารถประมวลผล

```text
orders_batch_1
orders_batch_2
orders_batch_3
```

ได้ครบ โดยไม่จำเป็นต้องแก้ไขข้อมูลต้นฉบับ

### 2. Order ID ใน Fact ต้องไม่ซ้ำ

`order_id` ใน `fact_sales` ต้องไม่ซ้ำกัน

โดยมีการใช้ `order_id` เป็น Primary Key และตรวจสอบ Order ที่มีอยู่แล้วก่อน Insert

### 3. Fact ต้องเชื่อมกับ Dimension ได้

Customer และ Product ที่อยู่ใน Fact ต้องสามารถหา Key ที่ตรงกันใน Dimension ได้

### 4. ค่าตัวเลขต้องไม่ติดลบ

ค่าที่เกี่ยวข้องกับยอดขาย เช่น

```text
quantity
unit_price
net_amount
```

ต้องไม่ติดลบ

### 5. รัน Batch เดิมซ้ำแล้วข้อมูลต้องไม่เพิ่มซ้ำ

ถ้ารัน Batch เดิมอีกครั้ง Order ที่มีอยู่ใน `fact_sales` แล้วจะไม่ถูกเพิ่มเข้าไปอีก

### 6. ข้อมูลที่ Reject ต้องมีเหตุผล

ข้อมูลที่ไม่ผ่านการตรวจสอบต้องมี `reason_code` เพื่อบอกสาเหตุที่ถูก Reject

### 7. Run Log ต้องแสดงจำนวนข้อมูลชัดเจน

Run Log ต้องแสดงจำนวน

```text
read
valid
rejected
loaded
```

เพื่อให้สามารถตรวจสอบได้ว่า Pipeline ทำงานอย่างไรในแต่ละ Batch

---

# 9. Reflection

จากการทำโปรเจกต์นี้ ทำให้เข้าใจการทำงานของ ETL มากขึ้น เพราะได้ลองทำตั้งแต่การอ่านข้อมูลจาก Excel ไปจนถึงการนำข้อมูลเข้า Data Warehouse จริง ๆ

สิ่งที่รู้สึกว่าสำคัญคือ **ข้อมูลที่นำเข้ามาไม่ได้ถูกต้องเสมอไป** ถ้านำข้อมูลเข้า Database โดยไม่ตรวจสอบก่อน ก็อาจทำให้ข้อมูลในระบบผิดและส่งผลต่อการวิเคราะห์ภายหลังได้ ดังนั้นการ Validate ข้อมูลก่อน Load จึงเป็นส่วนที่สำคัญของ Pipeline

อีกเรื่องที่ได้เรียนรู้คือการออกแบบ **Star Schema** ทำให้เห็นความแตกต่างระหว่าง Fact และ Dimension ชัดเจนขึ้น โดย Fact จะเก็บข้อมูลการขาย ส่วน Dimension จะเก็บรายละเอียดของลูกค้า สินค้า และวันที่ ทำให้สามารถนำข้อมูลไปวิเคราะห์ต่อได้ง่ายขึ้น

ปัญหาที่พบระหว่างทำคือเรื่องข้อมูลที่ไม่สมบูรณ์หรือไม่ตรงกับข้อมูลใน Dimension เช่น Customer ID หรือ Product ID ที่ไม่พบในตารางที่เกี่ยวข้อง จึงต้องมีการแยกข้อมูลเหล่านี้ออกไปไว้ใน `quarantine` แทนที่จะนำเข้า Fact โดยตรง

โดยรวมแล้วโปรเจกต์นี้ช่วยให้เข้าใจตั้งแต่การเตรียมข้อมูล การตรวจสอบข้อมูล การทำ ETL การออกแบบ Data Warehouse และการป้องกันข้อมูลซ้ำ ซึ่งสามารถนำแนวคิดเหล่านี้ไปใช้กับงานด้าน Data Engineering ในอนาคตได้

---

# 10. โครงสร้างไฟล์

```text
readytogo/
│
├── pipeline.py
├── Python_Data_Pipeline_Lab_Dataset.xlsx
├── retail_dw.db
├── quarantine.csv
├── pipeline_run_log.csv
└── README.md
```
