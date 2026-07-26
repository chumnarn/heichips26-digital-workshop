
# Exercise 4  
# การสร้างและบูรณาการ Hard Macro ด้วย LibreLane

## 4.1 วัตถุประสงค์ของบทปฏิบัติการ

บทปฏิบัติการนี้สาธิตแนวคิด **Hierarchical Physical Design** โดยเริ่มจากการสร้างวงจรตัวนับขนาด 8 บิตให้เป็น Hard Macro แล้วนำ Hard Macro ดังกล่าวจำนวน 4 ตัวไปประกอบเป็นวงจรตัวนับขนาด 32 บิต

เมื่อจบบทปฏิบัติการ ผู้เรียนจะสามารถ

1. อธิบายความแตกต่างระหว่าง Soft IP และ Hard Macro ได้
2. สร้าง Hard Macro จาก RTL ด้วย LibreLane
3. เลือกวิธีเชื่อมต่อ Power Distribution Network หรือ PDN ระหว่าง Macro และ Top-level
4. จำกัดชั้นโลหะสำหรับการออกแบบแบบลำดับชั้น
5. ส่งออกไฟล์ GDS, LEF, Gate-level Netlist, Liberty และ SPEF ของ Macro
6. ประกาศ Macro ในไฟล์ `config.yaml`
7. กำหนดตำแหน่งและทิศทางของ Macro แต่ละ instance
8. เชื่อมต่อขาไฟเลี้ยงของ Macro เข้ากับ PDN ระดับบน
9. วิเคราะห์ปัญหา PDN pitch และแก้ไขข้อผิดพลาดระหว่างการสร้าง PDN
10. ตรวจสอบผลลัพธ์ของการออกแบบแบบ Hierarchical

Exercise นี้ประกอบด้วยสองขั้นตอนหลัก ได้แก่

- **ขั้นตอนที่ 1:** สร้าง `counter_8bit` เป็น Hard Macro
- **ขั้นตอนที่ 2:** นำ Hard Macro จำนวน 4 ตัวมาประกอบเป็น `counter_32bit`

โครงสร้างดังกล่าวเป็นตัวอย่างพื้นฐานของการออกแบบชิปขนาดใหญ่ ซึ่งแต่ละ subsystem อาจถูกออกแบบ ตรวจสอบ และ sign-off แยกกันก่อนนำมาประกอบในระดับบนสุด

---

## 4.2 แนวคิดพื้นฐานเกี่ยวกับ Hard Macro

### 4.2.1 Soft IP

Soft IP คือโมดูลที่ส่งมอบในรูป RTL เช่น Verilog หรือ SystemVerilog ผู้ใช้สามารถนำ RTL ไปสังเคราะห์และทำ Physical Design ใหม่ได้

ตัวอย่างเช่น

```systemverilog
module counter (
    input  logic       clk_i,
    input  logic       rst_ni,
    output logic [7:0] count_o
);
```

ข้อดีของ Soft IP คือมีความยืดหยุ่นสูง สามารถเปลี่ยนพารามิเตอร์ เทคโนโลยี และข้อกำหนดทางเวลาได้ง่าย

อย่างไรก็ตาม ผลลัพธ์ด้านพื้นที่ ความเร็ว และพลังงานอาจแตกต่างกันไปตาม flow และ PDK ที่ใช้

### 4.2.2 Hard Macro

Hard Macro คือบล็อกที่ผ่านกระบวนการ Physical Design มาแล้ว โดยทั่วไปจะมีข้อมูลหลายชนิดประกอบกัน เช่น

| ไฟล์ | หน้าที่ |
|---|---|
| GDS | รูปร่างทางกายภาพและ geometry สำหรับ fabrication |
| LEF | ขนาด ขอบเขต ตำแหน่งขา และ routing obstruction |
| Liberty `.lib` | timing, power และ logical model |
| Gate-level netlist | แบบจำลองการเชื่อมต่อระดับเซลล์ |
| SPEF | parasitic resistance และ capacitance |
| Verilog model | แบบจำลองสำหรับ simulation หรือ synthesis |

LibreLane ต้องใช้ข้อมูลเหล่านี้ในขั้นตอนต่าง ๆ ของ flow ไม่สามารถใช้เฉพาะไฟล์ GDS เพียงไฟล์เดียวได้

Hard Macro เหมาะสำหรับบล็อกที่ต้องการนำกลับมาใช้ซ้ำ เช่น

- SRAM
- ROM
- Analog IP
- PLL
- ADC/DAC
- CPU core
- DSP accelerator
- I/O cells
- Peripheral subsystem
- บล็อกดิจิทัลที่ทำ sign-off แล้ว

Repository ของ Exercise 4 มีไฟล์หลัก ได้แก่ `counter_32bit.sv`, `top.sv`, `config.yaml` และไดเรกทอรี `counter_8bit` โดย README ระบุชัดเจนว่าเป้าหมายคือสร้าง Macro ขึ้นมาก่อน แล้วนำไปใช้ในงานออกแบบระดับบน 

---

## 4.3 สถาปัตยกรรมของวงจร

วงจรระดับบนเป็นตัวนับขนาด 32 บิตที่ประกอบจากตัวนับ 8 บิตจำนวน 4 ตัว

```text
                    +------------------+
 en_i ------------>| counter_0        |
 clk_i ------------>| count_o[7:0]     |
 rst_ni ----------->| ovf_o ----------+----+
                    +------------------+    |
                                               v
                    +------------------+    |
 clk_i ------------>| counter_1        |<---+
 rst_ni ----------->| count_o[15:8]    |
                    | ovf_o ----------+----+
                    +------------------+    |
                                               v
                    +------------------+    |
 clk_i ------------>| counter_2        |<---+
 rst_ni ----------->| count_o[23:16]   |
                    | ovf_o ----------+----+
                    +------------------+    |
                                               v
                    +------------------+    |
 clk_i ------------>| counter_3        |<---+
 rst_ni ----------->| count_o[31:24]   |
                    | ovf_o            |
                    +------------------+
```

แต่ละ Macro มี clock และ reset ร่วมกัน แต่สัญญาณ enable ถูกต่อแบบ cascade

- `counter_0` นับเมื่อ `en_i = 1`
- `counter_1` นับเมื่อ `counter_0` overflow
- `counter_2` นับเมื่อ `counter_1` overflow
- `counter_3` นับเมื่อ `counter_2` overflow
- overflow ของ `counter_3` ถูกส่งออกเป็น `ovf_o`

RTL ใน repository ประกาศ instance ชื่อ `counter_0` ถึง `counter_3` และแบ่ง output bus ออกเป็นช่วงละ 8 บิตตามโครงสร้างนี้ 

---

# ส่วนที่ 1: การสร้าง Counter 8-bit Hard Macro

## 4.4 เข้าสู่ไดเรกทอรี Exercise

สมมติว่า repository ถูก clone ไว้แล้ว ให้เข้าสู่ไดเรกทอรีหลักของโครงการ

```bash
cd ~/heichips26-digital-workshop
```

ตรวจสอบ branch ปัจจุบัน

```bash
git branch --show-current
```

ผลลัพธ์ควรเป็น

```text
main
```

จากนั้นเข้าสู่ Exercise 4

```bash
cd exercise_4
```

ตรวจสอบไฟล์

```bash
find . -maxdepth 2 -type f | sort
```

โครงสร้างโดยสรุปควรคล้ายกับ

```text
exercise_4/
├── README.md
├── config.yaml
├── counter_32bit.sv
├── top.sv
├── img/
└── counter_8bit/
    ├── config.yaml
    └── counter_8bit.sv
```

เข้าสู่ไดเรกทอรีของ Macro

```bash
cd counter_8bit
```

README ของ Exercise กำหนดให้เริ่มสร้าง Macro จากไดเรกทอรีนี้ก่อน 

---

## 4.5 ตรวจสอบ RTL ของ Counter 8-bit

เปิดไฟล์ RTL

```bash
less counter_8bit.sv
```

หรือใช้ editor

```bash
nano counter_8bit.sv
```

วงจรควรมี interface หลักประมาณนี้

```systemverilog
module counter_8bit (
    input  logic       clk_i,
    input  logic       rst_ni,
    input  logic       en_i,
    output logic [7:0] count_o,
    output logic       ovf_o
);
```

หน้าที่ของสัญญาณแต่ละตัวคือ

| สัญญาณ | ทิศทาง | ความหมาย |
|---|---:|---|
| `clk_i` | Input | สัญญาณนาฬิกา |
| `rst_ni` | Input | Active-low reset |
| `en_i` | Input | อนุญาตให้ตัวนับทำงาน |
| `count_o[7:0]` | Output | ค่าตัวนับ 8 บิต |
| `ovf_o` | Output | pulse หรือสถานะ overflow |

ควรตรวจสอบประเด็นต่อไปนี้ก่อนทำ Physical Design

- ชื่อ top module ต้องตรงกับ `DESIGN_NAME`
- ไม่มี module ซ้ำชื่อ
- reset polarity ตรงตามที่กำหนด
- ไม่มี inferred latch
- ไม่มี unresolved reference
- output ทุกตัวมี driver
- RTL สามารถสังเคราะห์ได้

---

## 4.6 ตรวจสอบไฟล์ Configuration ของ Macro

แสดงไฟล์ configuration

```bash
cat config.yaml
```

ค่าพื้นฐานมักประกอบด้วย

```yaml
DESIGN_NAME: counter_8bit

VERILOG_FILES:
  - dir::counter_8bit.sv

CLOCK_PORT: clk_i
CLOCK_PERIOD: 10
```

ความหมายคือ

- `DESIGN_NAME` กำหนดชื่อโมดูลบนสุด
- `VERILOG_FILES` ระบุ RTL input
- `CLOCK_PORT` ระบุชื่อ clock
- `CLOCK_PERIOD: 10` หมายถึงคาบ 10 ns หรือความถี่เป้าหมาย 100 MHz

คำนวณความถี่ได้จาก

$$f=\frac{1}{T}$$

เมื่อ

$$T=10\text{ ns}$$

จะได้

$$f=100\text{ MHz}$$

---

## 4.7 ทำความเข้าใจชั้นโลหะของ IHP SG13G2

IHP Open PDK มีชั้นโลหะสำหรับ routing ตั้งแต่ Metal1 ถึง Metal5 และมีชั้นโลหะด้านบนคือ TopMetal1 และ TopMetal2 ตามคำอธิบายของ Exercise 

แนวคิดโดยทั่วไปคือ

```text
TopMetal2   : นิยมใช้เป็นแนวตั้งสำหรับ PDN
TopMetal1   : นิยมใช้เป็นแนวนอนสำหรับ PDN
Metal5      : signal routing หรือ intermediate power
...
Metal1      : standard-cell rails และ local routing
```

LibreLane สามารถสร้าง power straps แนวนอนบน TopMetal1 และแนวตั้งบน TopMetal2 ได้ตามค่า default ของ flow ใน Exercise นี้ 

เมื่อ Macro ถูกนำไปวางใน Top-level จะต้องกำหนดวิธีให้ PDN ของ Macro เชื่อมต่อกับ PDN ระดับบน

---

## 4.8 เลือกวิธีการเชื่อมต่อ PDN

Exercise เสนอวิธีหลักสองแบบ

1. Hierarchical Method
2. Ring Method

---

## 4.9 วิธีที่ 1: Hierarchical Method

### 4.9.1 หลักการ

วิธี Hierarchical จะสงวนชั้นโลหะสูงสุดไว้ให้ Top-level

สำหรับ Exercise นี้

- Macro ใช้ routing สูงสุดถึง `TopMetal1`
- Top-level ใช้ `TopMetal2` พาดผ่าน Macro
- Via ถูกสร้างตรงบริเวณที่ TopMetal2 ของ Top-level ซ้อนกับ TopMetal1 ของ Macro
- VPWR ต้องซ้อนกับ VPWR
- VGND ต้องซ้อนกับ VGND

โครงสร้างแนวคิดคือ

```text
Top-level vertical PDN strap
          TopMetal2
              │
              │
             VIA
══════════════╪══════════════  Macro TopMetal1 strap
              │
          Macro core
```

ข้อดีคือ

- ประหยัดพื้นที่
- ไม่ต้องสร้าง power ring รอบ Macro
- เหมาะกับ design hierarchy ที่วางแผนชั้นโลหะอย่างชัดเจน

ข้อจำกัดคือ

- Macro ต้องลด routing layer ลงหนึ่งระดับ
- ต้องควบคุมตำแหน่งและ pitch ของ power straps อย่างระมัดระวัง
- ถ้า strap ไม่ซ้อนกัน Macro อาจไม่ได้รับไฟเลี้ยง

### 4.9.2 ปรับค่า Configuration

เพิ่มหรือแก้ไขค่าใน `counter_8bit/config.yaml`

```yaml
# Use only one PDN strap layer inside the macro
PDN_MULTILAYER: false

# Reserve TopMetal2 for the parent-level design
RT_MAX_LAYER: TopMetal1
```

Exercise ระบุสองตัวแปรนี้เป็นค่าหลักสำหรับวิธี Hierarchical 

### 4.9.3 ความหมายของตัวแปร

#### `PDN_MULTILAYER: false`

กำหนดไม่ให้ PDN ภายใน Macro ใช้ power straps หลายชั้นตามรูปแบบ default

ผลคือ PDN ของ Macro จะถูกจัดให้อยู่บนชั้นที่เหมาะสมกับการเชื่อมต่อจาก Top-level

#### `RT_MAX_LAYER: TopMetal1`

จำกัด signal routing และ routing resource สูงสุดไว้ที่ TopMetal1

TopMetal2 จึงถูกสงวนไว้ให้กับ parent block หรือ Top-level

### 4.9.4 ข้อควรระวัง

ถ้า Macro ยังคงใช้ TopMetal2 อยู่ อาจเกิด

- routing conflict
- blockage
- via generation failure
- Top-level PDN ไม่สามารถพาดข้าม Macro ได้อย่างถูกต้อง
- hierarchical routing strategy ไม่เป็นไปตามที่วางแผนไว้

---

## 4.10 วิธีที่ 2: Ring Method

### 4.10.1 หลักการ

Ring Method สร้างวงแหวนไฟเลี้ยงรอบขอบ Macro โดยใช้ชั้นโลหะระดับบน

```text
+====================================+
||            VPWR ring             ||
||   +--------------------------+   ||
||   |                          |   ||
||   |       Macro core         |   ||
||   |                          |   ||
||   +--------------------------+   ||
||            VGND ring             ||
+====================================+
```

ข้อดีคือ

- Macro สามารถใช้ชั้นโลหะทั้งหมด
- การเชื่อมต่อไฟเลี้ยงมีความชัดเจน
- เหมาะกับ Macro ขนาดใหญ่หรือ Macro ที่ต้องการ interface ด้าน power แบบ ring

ข้อจำกัดคือ

- ใช้พื้นที่มากขึ้น
- ต้องมีระยะรอบ Macro สำหรับ ring
- ต้องตรวจสอบ spacing และ DRC รอบขอบ Macro

### 4.10.2 ปรับค่า Configuration

เพิ่มค่า

```yaml
FP_PDN_CORE_RING: true
```

Exercise ระบุให้เปิด core ring เมื่อต้องการใช้วิธี Ring 

### 4.10.3 การเลือกวิธี

สำหรับบทเรียน สามารถเลือกวิธีใดวิธีหนึ่งได้

แนวทางแนะนำคือ

- ใช้ **Hierarchical Method** เพื่อเรียนรู้การสงวนชั้นโลหะและการเชื่อม strap ระหว่าง hierarchy
- ใช้ **Ring Method** เพื่อเรียนรู้การสร้าง Macro ที่มี power interface รอบบล็อก

ไม่ควรเปิดทั้งสองแนวทางพร้อมกันโดยไม่มีการวางแผน PDN ที่ชัดเจน

---

## 4.11 รัน LibreLane เพื่อสร้าง Macro

ตรวจสอบว่าอยู่ในไดเรกทอรี

```bash
pwd
```

ผลลัพธ์ควรลงท้ายด้วย

```text
exercise_4/counter_8bit
```

ตรวจสอบ LibreLane

```bash
librelane --version
```

ตรวจสอบว่า PDK พร้อมใช้งาน

```bash
ls ~/.ciel
```

จากนั้นรัน flow

```bash
librelane --pdk ihp-sg13g2 config.yaml
```

คำสั่งนี้เป็นคำสั่งที่ Exercise กำหนดสำหรับการ implement Macro 

---

## 4.12 ลำดับการประมวลผลที่ควรเกิดขึ้น

LibreLane จะดำเนินการหลายขั้นตอน เช่น

```text
RTL
 ↓
Synthesis
 ↓
Floorplan
 ↓
PDN generation
 ↓
Placement
 ↓
Clock Tree Synthesis
 ↓
Global Routing
 ↓
Detailed Routing
 ↓
Parasitic Extraction
 ↓
Static Timing Analysis
 ↓
DRC/LVS
 ↓
Final Views
```

ชื่อ step จริงอาจแตกต่างตามรุ่นของ LibreLane และ flow ที่ใช้งาน

ระหว่างการรันควรเฝ้าตรวจสอบข้อความประเภท

```text
ERROR
CRITICAL
WARNING
violation
unconnected
failed
```

ค้นหา error ใน run ล่าสุดได้ด้วย

```bash
grep -RniE "error|critical|failed" runs/*/ | tail -50
```

---

## 4.13 ตรวจสอบ Run Directory

หลัง flow จบ ให้ดูรายการ run

```bash
ls -lt runs
```

กำหนดตัวแปรให้ชี้ไปยัง run ล่าสุด

```bash
LATEST_RUN=$(ls -dt runs/* | head -1)
echo "$LATEST_RUN"
```

ตรวจสอบสถานะโดยสรุป

```bash
find "$LATEST_RUN" -maxdepth 2 -type f | sort | tail -50
```

ตรวจสอบไดเรกทอรี `final`

```bash
find "$LATEST_RUN/final" -maxdepth 3 -type f | sort
```

ผลลัพธ์ควรประกอบด้วยไฟล์สำคัญ เช่น

```text
final/
├── gds/
│   └── counter_8bit.gds
├── lef/
│   └── counter_8bit.lef
├── nl/
│   └── counter_8bit.nl.v
├── lib/
│   ├── nom_typ_1p20V_25C/
│   ├── nom_fast_1p32V_m40C/
│   └── nom_slow_1p08V_125C/
└── spef/
    └── nom/
        └── counter_8bit.nom.spef
```

ชื่อ corner และโครงสร้างย่อยควรตรงกับ PDK และ LibreLane version ที่ติดตั้ง

---

## 4.14 คัดลอก Final Views ของ Macro

Exercise กำหนดให้คัดลอกไดเรกทอรี `final` ของ run ล่าสุดมาไว้ภายใน `counter_8bit` 

ใช้คำสั่ง

```bash
rm -rf final
cp -a "$LATEST_RUN/final" ./final
```

ตรวจสอบ

```bash
find final -maxdepth 3 -type f | sort
```

เหตุผลที่ต้องคัดลอกมาไว้ตำแหน่งนี้ เพราะ configuration ของ Top-level จะอ้างอิงไฟล์ผ่าน path รูปแบบ

```yaml
dir::counter_8bit/final/...
```

ดังนั้นโครงสร้าง directory ต้องตรงตามที่ประกาศ

---

## 4.15 ตรวจสอบไฟล์สำคัญของ Macro

### 4.15.1 ตรวจสอบ GDS

```bash
ls -lh final/gds/counter_8bit.gds
```

ไฟล์ต้องมีขนาดมากกว่า 0 byte

```bash
test -s final/gds/counter_8bit.gds && echo "GDS: OK"
```

### 4.15.2 ตรวจสอบ LEF

```bash
head -50 final/lef/counter_8bit.lef
```

ค้นหาชื่อ Macro

```bash
grep -n "MACRO counter_8bit" final/lef/counter_8bit.lef
```

ค้นหาขาไฟ

```bash
grep -nE "PIN (VPWR|VGND)" final/lef/counter_8bit.lef
```

ค้นหาขาสัญญาณ

```bash
grep -nE "PIN (clk_i|rst_ni|en_i|count_o|ovf_o)" \
    final/lef/counter_8bit.lef
```

### 4.15.3 ตรวจสอบ Gate-level Netlist

```bash
head -40 final/nl/counter_8bit.nl.v
```

ค้นหา module

```bash
grep -n "module counter_8bit" final/nl/counter_8bit.nl.v
```

### 4.15.4 ตรวจสอบ Liberty

```bash
find final/lib -name "*.lib" -type f
```

ค้นหา cell definition

```bash
grep -Rni "cell *(counter_8bit" final/lib
```

ตรวจสอบว่ามี timing corner ที่ Top-level ต้องใช้งาน ได้แก่

- Typical
- Fast
- Slow

### 4.15.5 ตรวจสอบ SPEF

```bash
find final/spef -name "*.spef" -type f
```

ดูส่วนหัว

```bash
head -30 final/spef/nom/counter_8bit.nom.spef
```

ไฟล์ SPEF เป็นข้อมูล parasitic ของ Macro ใช้สำหรับ timing analysis ที่ระดับบน

---

# ส่วนที่ 2: การนำ Macro ไปใช้ใน Counter 32-bit

## 4.16 กลับสู่ไดเรกทอรี Top-level

```bash
cd ..
pwd
```

ผลลัพธ์ควรลงท้ายด้วย

```text
exercise_4
```

ตรวจสอบว่าไฟล์ Macro ถูกคัดลอกแล้ว

```bash
test -s counter_8bit/final/gds/counter_8bit.gds \
    && echo "Macro GDS found"

test -s counter_8bit/final/lef/counter_8bit.lef \
    && echo "Macro LEF found"
```

---

## 4.17 วิเคราะห์ RTL ระดับบน

เปิดไฟล์

```bash
less counter_32bit.sv
```

RTL ประกอบด้วย Macro instances 4 ตัว

```systemverilog
counter_8bit counter_0 (...);
counter_8bit counter_1 (...);
counter_8bit counter_2 (...);
counter_8bit counter_3 (...);
```

ชื่อ instance มีความสำคัญมาก เพราะต้องตรงกับชื่อที่ใช้ใน

- `MACROS.instances`
- `PDN_MACRO_CONNECTIONS`

ตัวอย่างเช่น RTL ใช้ชื่อ

```systemverilog
counter_8bit counter_0 (...);
```

configuration ต้องใช้

```yaml
instances:
  counter_0:
```

และ

```yaml
PDN_MACRO_CONNECTIONS:
  - "counter_0 VPWR VGND VPWR VGND"
```

ถ้าชื่อ instance ไม่ตรงกัน LibreLane จะหา Macro ไม่พบหรือไม่สามารถเชื่อม PDN ได้

---

## 4.18 ทำความเข้าใจ Configuration ระดับบน

ไฟล์เริ่มต้นของ repository กำหนดค่า เช่น

```yaml
DESIGN_NAME: counter_32bit

VERILOG_FILES: dir::counter_32bit.sv

CLOCK_PORT: clk_i
CLOCK_PERIOD: 10

FP_SIZING: absolute
DIE_AREA: [0, 0, 175, 175]

PL_TARGET_DENSITY_PCT: 60
```

นอกจากนี้ยังมีค่าที่ปิดขั้นตอนซึ่งออกแบบมาสำหรับ standard-cell logic เพราะ top-level นี้ทำหน้าที่ประกอบ Macro เป็นหลัก เช่น

```yaml
SYNTH_ELABORATE_ONLY: true
FP_PDN_ENABLE_RAILS: false
RUN_CTS: false

PL_RESIZER_DESIGN_OPTIMIZATIONS: false
PL_RESIZER_TIMING_OPTIMIZATIONS: false
GLB_RESIZER_DESIGN_OPTIMIZATIONS: false
GLB_RESIZER_TIMING_OPTIMIZATIONS: false
PL_RESIZER_BUFFER_INPUT_PORTS: false

RUN_FILL_INSERTION: false
```

ค่าดังกล่าวมีอยู่ใน configuration เริ่มต้นของ Exercise 

### 4.18.1 `SYNTH_ELABORATE_ONLY: true`

สั่งให้ synthesis เน้นการ elaborate โครงสร้าง design hierarchy โดยไม่พยายามสังเคราะห์ logic ภายใน Hard Macro ใหม่

Macro ถูกมองเป็น black box หรือ pre-implemented block ซึ่งมี physical และ timing views แยกต่างหาก

### 4.18.2 `FP_PDN_ENABLE_RAILS: false`

ปิด standard-cell power rails ใน Top-level เนื่องจาก Top-level แทบไม่มี standard cells

อย่างไรก็ตาม PDN straps สำหรับ Macro ยังจำเป็น

### 4.18.3 `RUN_CTS: false`

ไม่สร้าง Clock Tree Synthesis ระดับบน เพราะไม่มี standard-cell clock sinks ที่ต้องสร้าง clock tree แบบปกติ หรือ Exercise ตั้งใจเน้นการเชื่อม Macro

ในการออกแบบจริง การส่ง clock เข้า Macro หลายตัวต้องมีการวิเคราะห์ insertion delay, skew และ top-level clock distribution เพิ่มเติม

### 4.18.4 ปิด Resizer Optimizations

เมื่อ Top-level ไม่มี standard cells สำหรับ buffer หรือ resize การเปิด optimization เหล่านี้อาจไม่จำเป็นหรืออาจทำให้เครื่องมือพยายามเพิ่ม cell ในระดับบน

### 4.18.5 `RUN_FILL_INSERTION: false`

ปิด filler cell insertion เนื่องจากไม่มี standard-cell row ที่ต้องเติมเต็มในระดับบน

---

## 4.19 ประกาศ Macro ใน `config.yaml`

เพิ่มส่วน `MACROS` ลงในไฟล์ `exercise_4/config.yaml`

```yaml
MACROS:
  counter_8bit:
    gds:
      - dir::counter_8bit/final/gds/counter_8bit.gds

    lef:
      - dir::counter_8bit/final/lef/counter_8bit.lef

    nl:
      - dir::counter_8bit/final/nl/counter_8bit.nl.v

    lib:
      nom_typ_1p20V_25C:
        - dir::counter_8bit/final/lib/nom_typ_1p20V_25C/counter_8bit__nom_typ_1p20V_25C.lib

      nom_fast_1p32V_m40C:
        - dir::counter_8bit/final/lib/nom_fast_1p32V_m40C/counter_8bit__nom_fast_1p32V_m40C.lib

      nom_slow_1p08V_125C:
        - dir::counter_8bit/final/lib/nom_slow_1p08V_125C/counter_8bit__nom_slow_1p08V_125C.lib

    spef:
      nom:
        - dir::counter_8bit/final/spef/nom/counter_8bit.nom.spef

    instances:
      counter_0:
        location: [20, 20]
        orientation: N

      counter_1:
        location: [20, 105]
        orientation: N

      counter_2:
        location: [105, 20]
        orientation: N

      counter_3:
        location: [105, 105]
        orientation: N
```

โครงสร้างนี้ตรงกับตัวอย่าง integration ใน Exercise 

---

## 4.20 ความหมายของ Macro Views

### 4.20.1 GDS View

```yaml
gds:
  - dir::counter_8bit/final/gds/counter_8bit.gds
```

ใช้ในขั้นตอน

- GDS merge
- final stream-out
- physical verification
- DRC
- LVS layout side

### 4.20.2 LEF View

```yaml
lef:
  - dir::counter_8bit/final/lef/counter_8bit.lef
```

ใช้ในขั้นตอน

- floorplanning
- macro placement
- pin access
- obstruction handling
- routing
- PDN generation

LEF ไม่ได้บรรจุ geometry ทั้งหมดเหมือน GDS แต่เป็น abstract physical view

### 4.20.3 Netlist View

```yaml
nl:
  - dir::counter_8bit/final/nl/counter_8bit.nl.v
```

ใช้สำหรับ

- logical representation
- LVS
- gate-level hierarchy
- connectivity checking

### 4.20.4 Liberty View

```yaml
lib:
  nom_typ_1p20V_25C:
    - ...
```

ใช้สำหรับ Static Timing Analysis

แต่ละ corner แทนสภาวะ process, voltage และ temperature ต่างกัน เช่น

| Corner | ความหมายโดยทั่วไป |
|---|---|
| Typical | สภาวะ nominal |
| Fast | transistor เร็ว แรงดันสูง อุณหภูมิต่ำ |
| Slow | transistor ช้า แรงดันต่ำ อุณหภูมิสูง |

ชื่อ key ของ corner ต้องตรงกับชื่อ corner ที่ LibreLane และ PDK รู้จัก

### 4.20.5 SPEF View

```yaml
spef:
  nom:
    - ...
```

ใช้แทน parasitic ภายใน Macro โดยไม่ต้องเปิดรายละเอียด routing ภายในให้ Top-level extraction ทำใหม่ทั้งหมด

---

## 4.21 กำหนดตำแหน่ง Macro Instances

Exercise กำหนดตำแหน่งแบบ 2 × 2

```text
y
↑
|
|   counter_1          counter_3
|   [20,105]           [105,105]
|
|   counter_0          counter_2
|   [20,20]            [105,20]
|
+--------------------------------→ x
```

ค่า

```yaml
location: [20, 20]
```

หมายถึงพิกัดมุมอ้างอิงของ Macro ในหน่วยที่ LibreLane ใช้สำหรับ floorplan ซึ่งโดยทั่วไปสัมพันธ์กับไมโครเมตรในฐานข้อมูล physical design

### 4.21.1 ตรวจสอบ Macro overlap

ต้องตรวจสอบว่า

$$x_{\text{next}} \geq x_{\text{current}} + W_{\text{macro}} + S_x$$

และ

$$y_{\text{next}} \geq y_{\text{current}} + H_{\text{macro}} + S_y$$

เมื่อ

- $$W_{\text{macro}}$$ คือความกว้าง Macro
- $$H_{\text{macro}}$$ คือความสูง Macro
- $$S_x, S_y$$ คือระยะเผื่อ routing และ spacing

ขนาด Macro สามารถดูได้จาก LEF

```bash
grep -n "SIZE" counter_8bit/final/lef/counter_8bit.lef | head
```

ตัวอย่างผลลัพธ์

```text
SIZE 60.0 BY 60.0 ;
```

ตำแหน่ง `[20,20]` และ `[105,20]` จะมีช่องว่างแนวนอนประมาณ

$$105-(20+60)=25$$

ซึ่งใช้เป็น routing channel ระหว่าง Macro

---

## 4.22 ทำความเข้าใจ Orientation

ตัวอย่างกำหนด

```yaml
orientation: N
```

`N` หมายถึงวาง Macro ใน orientation ปกติ ไม่มีการหมุนหรือสะท้อน

orientation ที่มักพบ ได้แก่

| Orientation | ความหมาย |
|---|---|
| `N` | ปกติ |
| `S` | หมุน 180° |
| `E` | หมุน 90° |
| `W` | หมุน 270° |
| `FN` | สะท้อนตามแกนหนึ่ง |
| `FS` | สะท้อนและหมุน |
| `FE` | สะท้อนร่วมกับการหมุน 90° |
| `FW` | สะท้อนร่วมกับการหมุน 270° |

การเปลี่ยน orientation มีผลต่อ

- ตำแหน่ง signal pins
- ตำแหน่ง power pins
- การซ้อนของ PDN straps
- routing congestion
- ความยาวสายระหว่าง Macro
- clock routing

สำหรับ Exercise เริ่มต้นควรใช้ `N` ทุก instance เพื่อให้วิเคราะห์ง่าย

---

## 4.23 เชื่อมต่อ Power Pins ของ Macro

เพิ่ม

```yaml
PDN_MACRO_CONNECTIONS:
  - "counter_0 VPWR VGND VPWR VGND"
  - "counter_1 VPWR VGND VPWR VGND"
  - "counter_2 VPWR VGND VPWR VGND"
  - "counter_3 VPWR VGND VPWR VGND"
```

รูปแบบของแต่ละบรรทัดคือ

```text
<instance> <top-level power> <top-level ground> <macro power> <macro ground>
```

ตัวอย่าง

```text
counter_0 VPWR VGND VPWR VGND
```

แปลว่า

- instance ชื่อ `counter_0`
- top-level power net คือ `VPWR`
- top-level ground net คือ `VGND`
- power pin ภายใน Macro คือ `VPWR`
- ground pin ภายใน Macro คือ `VGND`

Exercise กำหนดการเชื่อมต่อให้ Macro ทั้งสี่ตัวด้วยรูปแบบนี้ 

### 4.23.1 จุดที่มักผิดพลาด

ชื่อ power pin ใน LEF อาจแตกต่างจากที่คาดไว้ ควรตรวจสอบก่อน

```bash
grep -nE "PIN (VPWR|VGND)" \
    counter_8bit/final/lef/counter_8bit.lef
```

ชื่อ instance ต้องตรงกับ netlist

```bash
grep -n "counter_8bit counter_" counter_32bit.sv
```

ถ้า RTL ถูก flatten หรือ instance ถูกเปลี่ยนชื่อ การอ้างอิงใน `PDN_MACRO_CONNECTIONS` อาจไม่ตรง

---

## 4.24 ปรับ PDN Pitch

เพิ่มค่า

```yaml
FP_PDN_VPITCH: 20
FP_PDN_HPITCH: 20
```

Exercise เตือนว่า PDN generation อาจล้มเหลวถ้า pitch ของ straps ระดับบนกว้างเกินไป จึงแนะนำให้ลด vertical และ horizontal pitch เป็น 20 

### 4.24.1 ความหมายของ Pitch

Pitch คือระยะจากจุดอ้างอิงของ strap หนึ่งไปยัง strap ถัดไป

```text
strap        strap        strap
  │            │            │
  │<--pitch--->│<--pitch--->│
```

ถ้า pitch มีค่ามาก

- จำนวน strap ลดลง
- routing resource สำหรับ signal เพิ่มขึ้น
- แต่โอกาสที่ strap จะพาดทับ power pin ของ Macro ลดลง

ถ้า pitch มีค่าน้อย

- จำนวน strap เพิ่มขึ้น
- IR drop อาจดีขึ้น
- โอกาสเชื่อม Macro สูงขึ้น
- แต่ใช้พื้นที่ routing มากขึ้น

### 4.24.2 เหตุผลที่ Macro อาจเชื่อมไฟไม่ได้

สำหรับ Hierarchical Method จำเป็นต้องมีการซ้อนกันระหว่าง

```text
Top-level TopMetal2 strap
          +
Macro TopMetal1 power stripe
```

ถ้า Top-level strap spacing กว้างเกินไป อาจไม่มี strap ใดผ่านตำแหน่ง power stripe ของ Macro

ผลที่พบได้ เช่น

```text
Cannot connect macro power pin
No intersection between straps
Unconnected PDN node
PDN generation failed
```

---

## 4.25 ตัวแปร PDN เพิ่มเติม

README กล่าวถึงตัวแปรที่ควรศึกษาเพิ่มเติม ได้แก่

```yaml
FP_PDN_VWIDTH
FP_PDN_HWIDTH

FP_PDN_VSPACING
FP_PDN_HSPACING
```



ความหมายคือ

| ตัวแปร | ความหมาย |
|---|---|
| `FP_PDN_VWIDTH` | ความกว้าง vertical strap |
| `FP_PDN_HWIDTH` | ความกว้าง horizontal strap |
| `FP_PDN_VSPACING` | spacing ระหว่าง vertical power/ground straps |
| `FP_PDN_HSPACING` | spacing ระหว่าง horizontal power/ground straps |
| `FP_PDN_VPITCH` | pitch ของชุด vertical straps |
| `FP_PDN_HPITCH` | pitch ของชุด horizontal straps |

ค่าที่เหมาะสมต้องพิจารณาร่วมกับ

- PDK design rules
- current requirement
- IR drop
- electromigration
- routing congestion
- Macro pin geometry
- die size
- power-domain architecture

---

## 4.26 ตัวอย่าง `config.yaml` ฉบับรวม

ตัวอย่างต่อไปนี้แสดง configuration ระดับบนที่รวม Macro configuration เข้ากับค่าพื้นฐานของ Exercise

```yaml
# ============================================================
# Exercise 4: 32-bit Counter Using Four 8-bit Hard Macros
# ============================================================

DESIGN_NAME: counter_32bit

VERILOG_FILES:
  - dir::counter_32bit.sv

CLOCK_PORT: clk_i
CLOCK_PERIOD: 10

# ------------------------------------------------------------
# Floorplan
# ------------------------------------------------------------

FP_SIZING: absolute
DIE_AREA: [0, 0, 175, 175]

PL_TARGET_DENSITY_PCT: 60

# ------------------------------------------------------------
# Macro-only top-level configuration
# ------------------------------------------------------------

SYNTH_ELABORATE_ONLY: true

FP_PDN_ENABLE_RAILS: false

RUN_CTS: false

PL_RESIZER_DESIGN_OPTIMIZATIONS: false
PL_RESIZER_TIMING_OPTIMIZATIONS: false

GLB_RESIZER_DESIGN_OPTIMIZATIONS: false
GLB_RESIZER_TIMING_OPTIMIZATIONS: false

PL_RESIZER_BUFFER_INPUT_PORTS: false

RUN_FILL_INSERTION: false

# ------------------------------------------------------------
# Power Distribution Network
# ------------------------------------------------------------

FP_PDN_VPITCH: 20
FP_PDN_HPITCH: 20

PDN_MACRO_CONNECTIONS:
  - "counter_0 VPWR VGND VPWR VGND"
  - "counter_1 VPWR VGND VPWR VGND"
  - "counter_2 VPWR VGND VPWR VGND"
  - "counter_3 VPWR VGND VPWR VGND"

# ------------------------------------------------------------
# Macro Views and Placement
# ------------------------------------------------------------

MACROS:
  counter_8bit:
    gds:
      - dir::counter_8bit/final/gds/counter_8bit.gds

    lef:
      - dir::counter_8bit/final/lef/counter_8bit.lef

    nl:
      - dir::counter_8bit/final/nl/counter_8bit.nl.v

    lib:
      nom_typ_1p20V_25C:
        - dir::counter_8bit/final/lib/nom_typ_1p20V_25C/counter_8bit__nom_typ_1p20V_25C.lib

      nom_fast_1p32V_m40C:
        - dir::counter_8bit/final/lib/nom_fast_1p32V_m40C/counter_8bit__nom_fast_1p32V_m40C.lib

      nom_slow_1p08V_125C:
        - dir::counter_8bit/final/lib/nom_slow_1p08V_125C/counter_8bit__nom_slow_1p08V_125C.lib

    spef:
      nom:
        - dir::counter_8bit/final/spef/nom/counter_8bit.nom.spef

    instances:
      counter_0:
        location: [20, 20]
        orientation: N

      counter_1:
        location: [20, 105]
        orientation: N

      counter_2:
        location: [105, 20]
        orientation: N

      counter_3:
        location: [105, 105]
        orientation: N
```

ควรตรวจสอบชื่อไฟล์ Liberty และ SPEF กับไฟล์ที่ถูกสร้างจริง เพราะชื่ออาจแตกต่างกันตามเวอร์ชันของ flow

---

## 4.27 ตรวจสอบ Configuration ก่อนรัน

ตรวจสอบ YAML syntax โดยใช้ Python

```bash
python3 - <<'PY'
import yaml

with open("config.yaml", "r", encoding="utf-8") as f:
    config = yaml.safe_load(f)

print("DESIGN_NAME =", config.get("DESIGN_NAME"))
print("Macro types =", list(config.get("MACROS", {}).keys()))
print("PDN connections =", len(config.get("PDN_MACRO_CONNECTIONS", [])))
PY
```

ผลลัพธ์ควรคล้ายกับ

```text
DESIGN_NAME = counter_32bit
Macro types = ['counter_8bit']
PDN connections = 4
```

ตรวจสอบไฟล์ทั้งหมดที่ configuration อ้างอิง

```bash
for f in \
  counter_8bit/final/gds/counter_8bit.gds \
  counter_8bit/final/lef/counter_8bit.lef \
  counter_8bit/final/nl/counter_8bit.nl.v
do
    if [ -s "$f" ]; then
        echo "[OK] $f"
    else
        echo "[MISSING] $f"
    fi
done
```

ตรวจสอบ Liberty

```bash
find counter_8bit/final/lib -name "*.lib" -type f -print
```

ตรวจสอบ SPEF

```bash
find counter_8bit/final/spef -name "*.spef" -type f -print
```

---

## 4.28 รัน Top-level Flow

ตรวจสอบตำแหน่งปัจจุบัน

```bash
pwd
```

ต้องอยู่ที่

```text
exercise_4
```

รัน

```bash
librelane --pdk ihp-sg13g2 config.yaml
```

นี่คือคำสั่งที่ README กำหนดสำหรับการ implement Top-level 

---

## 4.29 สิ่งที่ต้องสังเกตระหว่างรัน

### 4.29.1 Synthesis และ Elaboration

เครื่องมือต้องสามารถ

- หา module `counter_32bit`
- หา black-box definition ของ `counter_8bit`
- resolve instance `counter_0` ถึง `counter_3`
- เชื่อมต่อ bus และ overflow chain ได้ครบ

ข้อผิดพลาดที่อาจพบ

```text
Module counter_8bit not found
Unknown module type
Unresolved blackbox
```

แนวทางแก้ไข

- ตรวจสอบ `MACROS.nl`
- ตรวจสอบชื่อ module ใน netlist
- ตรวจสอบชื่อ module ใน RTL
- ตรวจสอบ path ที่ใช้ `dir::`

### 4.29.2 Floorplanning

ตรวจสอบว่า Macro ทั้ง 4 ตัว

- อยู่ภายใน die area
- ไม่ overlap
- ไม่ชน die boundary
- มี channel ระหว่างกัน
- orientation ถูกต้อง

### 4.29.3 PDN Generation

ตรวจสอบว่า

- มี vertical straps
- มี horizontal straps
- straps ผ่าน Macro
- Macro power pins ถูกเชื่อม
- ไม่มี floating power net
- ไม่มี short ระหว่าง VPWR และ VGND

### 4.29.4 Routing

ตรวจสอบว่า signal nets ระหว่าง Macro สามารถ route ได้ โดยเฉพาะ

- `counter_0_ovf`
- `counter_1_ovf`
- `counter_2_ovf`
- `clk_i`
- `rst_ni`
- `en_i`
- `count_o[31:0]`
- `ovf_o`

---

## 4.30 ตรวจสอบผลลัพธ์หลัง Flow จบ

กำหนด run ล่าสุด

```bash
LATEST_RUN=$(ls -dt runs/* | head -1)
echo "$LATEST_RUN"
```

ค้นหา error

```bash
grep -RniE "error|critical|failed" "$LATEST_RUN" | head -100
```

ค้นหา warning สำคัญ

```bash
grep -RniE "unconnected|floating|violation|overlap|short" \
    "$LATEST_RUN" | head -100
```

ตรวจสอบ final files

```bash
find "$LATEST_RUN/final" -maxdepth 3 -type f | sort
```

ตรวจสอบ GDS

```bash
find "$LATEST_RUN/final" -name "*.gds" -type f -ls
```

ตรวจสอบ LEF

```bash
find "$LATEST_RUN/final" -name "*.lef" -type f -ls
```

---

## 4.31 เปิดผลลัพธ์ด้วย OpenROAD GUI

รูปแบบคำสั่งขึ้นกับ LibreLane version แต่สามารถเปิด run จาก interface ของ LibreLane หรือใช้ OpenROAD database ที่ flow สร้างไว้

ใน GUI ให้เปิด display control สำหรับ

- Macros
- Instances
- Nets
- Obstructions
- Power nets
- Routing tracks
- Metal layers
- Blockages
- Pins

ตรวจสอบภาพรวม

```text
+--------------------------------------------------+
|                                                  |
|   +---------------+      +---------------+       |
|   | counter_1     |      | counter_3     |       |
|   |               |      |               |       |
|   +---------------+      +---------------+       |
|                                                  |
|   +---------------+      +---------------+       |
|   | counter_0     |      | counter_2     |       |
|   |               |      |               |       |
|   +---------------+      +---------------+       |
|                                                  |
+--------------------------------------------------+
```

---

## 4.32 Checklist การตรวจสอบ Layout

### 4.32.1 Macro Placement

- Macro ทั้ง 4 ตัวปรากฏครบ
- ชื่อ instance ถูกต้อง
- ไม่มี Macro overlap
- ไม่มี Macro อยู่นอก die
- orientation เป็น `N`
- routing channel มีพื้นที่เพียงพอ

### 4.32.2 PDN

- TopMetal straps ปรากฏครบ
- VPWR และ VGND สลับตำแหน่งอย่างถูกต้อง
- strap พาดผ่านหรือเชื่อมถึง Macro
- มี via เชื่อมชั้นโลหะ
- ไม่มี open
- ไม่มี short

### 4.32.3 Signal Routing

- overflow chain เชื่อมต่อครบ
- clock เข้าถึง Macro ทั้งสี่ตัว
- reset เข้าถึง Macro ทั้งสี่ตัว
- output bus ไม่มี unrouted net
- ไม่มี routing congestion รุนแรง

### 4.32.4 Final Verification

- flow จบโดยไม่มี fatal error
- final GDS ถูกสร้าง
- DRC อยู่ในเกณฑ์ที่ Exercise กำหนด
- LVS ผ่านหรือไม่มี mismatch ที่ไม่คาดหมาย
- timing report ถูกสร้าง
- ไม่มี unconnected Macro power pins

---

# ส่วนที่ 3: การวิเคราะห์ปัญหาและการแก้ไข

## 4.33 ปัญหา: ไม่พบ Macro File

ตัวอย่างข้อความ

```text
File not found:
counter_8bit/final/lef/counter_8bit.lef
```

ตรวจสอบ

```bash
find counter_8bit/final -type f | sort
```

สาเหตุที่เป็นไปได้

- ยังไม่ได้สร้าง Macro
- ลืมคัดลอก `runs/<timestamp>/final`
- path ผิด
- ชื่อไฟล์ต่างจาก configuration
- รัน LibreLane จาก working directory ผิด

แนวทางแก้ไข

```bash
cd counter_8bit
LATEST_RUN=$(ls -dt runs/* | head -1)
rm -rf final
cp -a "$LATEST_RUN/final" ./final
cd ..
```

---

## 4.34 ปัญหา: Unknown Module `counter_8bit`

ตัวอย่าง

```text
ERROR: Module counter_8bit referenced in counter_32bit is not part of the design
```

ตรวจสอบชื่อ module

```bash
grep -Rni "module counter_8bit" \
    counter_8bit/final/nl \
    counter_8bit/*.sv
```

ตรวจสอบว่า `nl` ถูกประกาศใน `MACROS`

```yaml
nl:
  - dir::counter_8bit/final/nl/counter_8bit.nl.v
```

ตรวจสอบชื่อ top-level instance

```bash
grep -n "counter_8bit counter_" counter_32bit.sv
```

---

## 4.35 ปัญหา: Macro Instance Name ไม่ตรง

ตัวอย่าง RTL

```systemverilog
counter_8bit u_counter_0 (...);
```

แต่ configuration ใช้

```yaml
instances:
  counter_0:
```

ชื่อไม่ตรงกัน

แก้ให้เป็น

```yaml
instances:
  u_counter_0:
```

และ

```yaml
PDN_MACRO_CONNECTIONS:
  - "u_counter_0 VPWR VGND VPWR VGND"
```

---

## 4.36 ปัญหา: PDN Generation Failed

สาเหตุหลักที่ Exercise ชี้ไว้คือ top-level PDN pitch กว้างเกินไป ทำให้ strap ไม่เชื่อม Macro 

แก้โดยลด pitch

```yaml
FP_PDN_VPITCH: 20
FP_PDN_HPITCH: 20
```

หากยังไม่ผ่าน ให้ตรวจสอบเพิ่ม

- ตำแหน่ง Macro
- ขนาด Macro
- รูปร่าง power pin ใน LEF
- strap offset
- strap width
- spacing
- routing layer
- orientation

ดู power pin geometry

```bash
awk '
/PIN VPWR/,/END VPWR/
/PIN VGND/,/END VGND/
' counter_8bit/final/lef/counter_8bit.lef
```

---

## 4.37 ปัญหา: Macro Overlap

ถ้า Macro มีขนาดใหญ่กว่าที่คาด ตำแหน่งอาจทับกัน

ตรวจสอบขนาด

```bash
grep "SIZE" counter_8bit/final/lef/counter_8bit.lef
```

เพิ่มระยะห่าง เช่น

```yaml
instances:
  counter_0:
    location: [20, 20]
    orientation: N

  counter_1:
    location: [20, 115]
    orientation: N

  counter_2:
    location: [115, 20]
    orientation: N

  counter_3:
    location: [115, 115]
    orientation: N
```

หากเกิน die ให้ขยาย

```yaml
DIE_AREA: [0, 0, 200, 200]
```

การขยาย die ควรทำพร้อมการตรวจสอบ routing channel และ PDN pitch ใหม่

---

## 4.38 ปัญหา: Liberty Corner ไม่ตรง

ตัวอย่าง

```text
No liberty file defined for corner nom_typ_1p20V_25C
```

ตรวจสอบไฟล์จริง

```bash
find counter_8bit/final/lib -type f -name "*.lib" | sort
```

แก้ชื่อ key และ path ให้ตรงกับไฟล์ที่ถูกสร้าง

อย่าเดาชื่อ corner จากตัวอย่างโดยไม่ตรวจสอบ output จริง

---

## 4.39 ปัญหา: Power Pin Name ไม่ตรง

Macro บางชนิดอาจใช้ชื่อ

```text
VDD
VSS
VDD!
VSS!
VDDARRAY!
VSS!
```

แทน `VPWR` และ `VGND`

ตรวจสอบ LEF

```bash
grep -n "USE POWER\|USE GROUND" \
    counter_8bit/final/lef/counter_8bit.lef
```

ตรวจสอบ netlist

```bash
grep -RniE "VPWR|VGND|VDD|VSS" \
    counter_8bit/final/nl
```

จากนั้นแก้ `PDN_MACRO_CONNECTIONS` ให้ตรงกับชื่อจริง

---

## 4.40 ปัญหา: Routing Congestion

สาเหตุอาจมาจาก

- Macro วางชิดกันเกินไป
- output bus มีจำนวนมาก
- pin ของ Macro หันเข้าหาขอบ die
- PDN straps หนาและถี่เกินไป
- routing channel แคบ
- orientation ไม่เหมาะสม

แนวทางแก้

1. เพิ่มระยะห่างระหว่าง Macro
2. ขยาย die area
3. เปลี่ยน orientation
4. ปรับ PDN pitch หรือ width อย่างระมัดระวัง
5. วาง Macro ตามทิศทางของ signal flow
6. ปรับตำแหน่ง I/O pins

---

# ส่วนที่ 4: การทดลองเพิ่มเติม

## 4.41 ทดลองเปรียบเทียบ Hierarchical Method กับ Ring Method

สร้างผลลัพธ์สองชุด

### ชุด A: Hierarchical

```yaml
PDN_MULTILAYER: false
RT_MAX_LAYER: TopMetal1
```

### ชุด B: Ring

```yaml
FP_PDN_CORE_RING: true
```

บันทึกผลดังตาราง

| รายการ | Hierarchical | Ring |
|---|---:|---:|
| Macro width | | |
| Macro height | | |
| Macro area | | |
| จำนวน PDN layers | | |
| DRC violations | | |
| Worst slack | | |
| Routing congestion | | |
| ความง่ายในการเชื่อม Top-level | | |

วิเคราะห์ว่าเหตุใด Ring Method จึงอาจใช้พื้นที่มากกว่า แต่มี interface ด้าน power ที่ชัดเจนกว่า

---

## 4.42 ทดลองเปลี่ยน PDN Pitch

ทดลองค่า

```yaml
FP_PDN_VPITCH: 15
FP_PDN_HPITCH: 15
```

จากนั้นทดลอง

```yaml
FP_PDN_VPITCH: 30
FP_PDN_HPITCH: 30
```

และ

```yaml
FP_PDN_VPITCH: 50
FP_PDN_HPITCH: 50
```

บันทึกผล

| Pitch | PDN ผ่านหรือไม่ | จำนวน straps | Routing congestion | หมายเหตุ |
|---:|---|---:|---|---|
| 15 | | | | |
| 20 | | | | |
| 30 | | | | |
| 50 | | | | |

ตอบคำถาม

1. Pitch ใดทำให้ Macro เชื่อมต่อไฟได้ครบ
2. Pitch ใดเริ่มเกิด PDN error
3. Pitch ที่เล็กลงมีผลต่อ routing congestion อย่างไร
4. จำนวน straps มีผลต่อ IR drop อย่างไรในเชิงแนวคิด

---

## 4.43 ทดลองเปลี่ยน Macro Orientation

ทดลองเปลี่ยน

```yaml
counter_1:
  location: [20, 105]
  orientation: S
```

หรือ

```yaml
counter_2:
  location: [105, 20]
  orientation: E
```

ตรวจสอบ

- pin location เปลี่ยนอย่างไร
- overflow connection สั้นลงหรือยาวขึ้น
- power straps ยังเชื่อมต่อหรือไม่
- Macro อยู่ภายใน die หรือไม่
- routing congestion เปลี่ยนหรือไม่

---

## 4.44 ทดลองเปลี่ยน Macro Placement

วาง Macro ตาม signal flow เป็นแถวเดียว

```text
counter_0 → counter_1 → counter_2 → counter_3
```

เปรียบเทียบกับแบบ 2 × 2

```text
counter_1    counter_3

counter_0    counter_2
```

วิเคราะห์

- ความยาว overflow chain
- clock routing
- die aspect ratio
- congestion
- pin access
- PDN regularity

---

# ส่วนที่ 5: Bonus — การใช้งาน SRAM Macro ของ IHP

## 4.45 เหตุผลที่ใช้ SRAM Macro

หน่วยความจำที่สร้างจาก flip-flops ใช้พื้นที่และพลังงานสูงเมื่อความจุเพิ่มขึ้น

ตัวอย่างหน่วยความจำขนาด

$$1024 \times 8 = 8192\text{ bits}$$

ถ้าสร้างจาก flip-flop จะต้องใช้ storage elements อย่างน้อย 8192 ตัว ยังไม่รวม decoder, mux และ control logic

SRAM Macro ถูกออกแบบเป็นโครงสร้าง bitcell ที่มีความหนาแน่นสูง จึงเหมาะกว่ามากสำหรับ memory ขนาดใหญ่

Exercise เสนอ SRAM รุ่น `RM_IHPSG13_1P_1024x8_c2_bm_bist` ซึ่งเป็น SRAM ความกว้าง 8 บิต จำนวน 1024 words 

---

## 4.46 ค้นหา SRAM ใน PDK

PDK ถูกเก็บภายใต้ `~/.ciel` ตามคำแนะนำของ Exercise 

ค้นหา directory

```bash
find ~/.ciel -type d -path "*libs.ref/sg13g2_sram*" 2>/dev/null
```

ดูรายการ SRAM

```bash
find ~/.ciel -type f \
    -path "*libs.ref/sg13g2_sram/lef/*.lef" \
    | sort
```

ค้นหา Macro ที่กำหนด

```bash
find ~/.ciel -type f \
    -name "RM_IHPSG13_1P_1024x8_c2_bm_bist*" \
    | sort
```

---

## 4.47 ตัวอย่าง Configuration สำหรับ SRAM

```yaml
MACROS:
  RM_IHPSG13_1P_1024x8_c2_bm_bist:
    gds:
      - pdk_dir::libs.ref/sg13g2_sram/gds/RM_IHPSG13_1P_1024x8_c2_bm_bist.gds

    lef:
      - pdk_dir::libs.ref/sg13g2_sram/lef/RM_IHPSG13_1P_1024x8_c2_bm_bist.lef

    nl:
      - pdk_dir::libs.ref/sg13g2_sram/verilog/RM_IHPSG13_1P_1024x8_c2_bm_bist.v

    lib:
      nom_typ_1p20V_25C:
        - pdk_dir::libs.ref/sg13g2_sram/lib/RM_IHPSG13_1P_1024x8_c2_bm_bist_typ_1p20V_25C.lib

      nom_fast_1p32V_m40C:
        - pdk_dir::libs.ref/sg13g2_sram/lib/RM_IHPSG13_1P_1024x8_c2_bm_bist_fast_1p32V_m55C.lib

      nom_slow_1p08V_125C:
        - pdk_dir::libs.ref/sg13g2_sram/lib/RM_IHPSG13_1P_1024x8_c2_bm_bist_slow_1p08V_125C.lib

    instances:
      top.sram:
        location: [50, 50]
        orientation: E
```

ตัวอย่างนี้อ้างอิง Macro views จาก `pdk_dir::` โดยตรงตาม Exercise 

### ความแตกต่างระหว่าง `dir::` และ `pdk_dir::`

```yaml
dir::counter_8bit/...
```

หมายถึง path สัมพัทธ์กับ directory ของ design configuration

```yaml
pdk_dir::libs.ref/...
```

หมายถึง path สัมพัทธ์กับ root directory ของ PDK

---

## 4.48 การเชื่อม Power ของ SRAM

SRAM ตัวอย่างมี power supply มากกว่าหนึ่งชุด จึงต้องประกาศ

```yaml
PDN_MACRO_CONNECTIONS:
  - "top.sram VPWR VGND VDD! VSS!"
  - "top.sram VPWR VGND VDDARRAY! VSS!"
```

Exercise แสดงการเชื่อมทั้ง `VDD!` และ `VDDARRAY!` เข้ากับ top-level `VPWR` และเชื่อม `VSS!` เข้ากับ `VGND` 

ความหมายเชิงสถาปัตยกรรมคือ

- `VDD!` อาจเป็นไฟเลี้ยง peripheral/control circuitry
- `VDDARRAY!` เป็นไฟเลี้ยง memory array
- `VSS!` เป็น ground

แม้จะเชื่อมเข้ากับ supply เดียวกันใน Exercise แต่การออกแบบขั้นสูงอาจแยก supply domain เพื่อวิเคราะห์ noise, power gating หรือ retention

---

# ส่วนที่ 6: คำถามท้ายบท

## 4.49 คำถามทบทวน

1. Hard Macro แตกต่างจาก Soft IP อย่างไร
2. เพราะเหตุใด Top-level จึงต้องใช้ทั้ง GDS และ LEF
3. Liberty file มีหน้าที่อะไร
4. SPEF file มีข้อมูลประเภทใด
5. เหตุใด Hierarchical Method จึงต้องสงวน TopMetal2
6. Ring Method มีข้อดีและข้อเสียอย่างไร
7. `RT_MAX_LAYER` มีผลต่อ routing อย่างไร
8. `PDN_MULTILAYER: false` มีผลต่อ Macro PDN อย่างไร
9. เพราะเหตุใดชื่อ instance ใน RTL ต้องตรงกับ configuration
10. `PDN_MACRO_CONNECTIONS` เชื่อมข้อมูลใดเข้าด้วยกัน
11. เหตุใด PDN pitch ที่กว้างเกินไปจึงทำให้ Macro ไม่ได้รับไฟ
12. การลด PDN pitch ส่งผลต่อ signal routing อย่างไร
13. เพราะเหตุใด Top-level ของ Exercise จึงปิด CTS
14. เพราะเหตุใด `SYNTH_ELABORATE_ONLY` จึงเหมาะกับ Macro-only Top-level
15. การหมุน Macro มีผลต่อ pin access และ PDN อย่างไร
16. เหตุใด SRAM จึงเหมาะกับหน่วยความจำขนาดใหญ่มากกว่า flip-flop array
17. `dir::` และ `pdk_dir::` แตกต่างกันอย่างไร
18. ถ้า Macro มี power supply สองชุด ควรกำหนด `PDN_MACRO_CONNECTIONS` อย่างไร
19. ควรตรวจสอบไฟล์ใดเพื่อหาขนาดและตำแหน่ง pin ของ Macro
20. ควรตรวจสอบไฟล์ใดเพื่อวิเคราะห์ timing ของ Macro

---

## 4.50 งานที่ต้องส่ง

ให้ผู้เรียนส่งไฟล์และหลักฐานดังต่อไปนี้

### ส่วนที่ 1: Counter 8-bit Macro

1. ไฟล์ `counter_8bit/config.yaml`
2. ภาพ floorplan ของ Macro
3. ภาพ PDN ของ Macro
4. ไฟล์ GDS
5. ไฟล์ LEF
6. Gate-level netlist
7. Liberty files อย่างน้อย 3 corners
8. SPEF file
9. สรุปผล DRC/LVS
10. สรุป worst timing slack

### ส่วนที่ 2: Counter 32-bit Top-level

1. ไฟล์ `counter_32bit.sv`
2. ไฟล์ `config.yaml`
3. ภาพ Macro placement
4. ภาพ PDN ที่พาดและเชื่อม Macro
5. ภาพ routed design
6. รายงานจำนวน Macro instances
7. รายงาน unrouted nets
8. รายงาน DRC/LVS
9. รายงาน timing
10. Final GDS

### ส่วนที่ 3: วิเคราะห์ผล

เขียนรายงานสั้น 1–2 หน้าเปรียบเทียบ

- Hierarchical Method
- Ring Method
- ผลของ PDN pitch
- ผลของ Macro orientation
- ปัญหาที่พบและวิธีแก้ไข

---

## 4.51 Checklist ก่อนถือว่า Exercise ผ่าน

### Macro Generation

- [ ] `counter_8bit` ผ่าน LibreLane flow
- [ ] มี `counter_8bit.gds`
- [ ] มี `counter_8bit.lef`
- [ ] มี `counter_8bit.nl.v`
- [ ] มี Liberty ครบ corner ที่ต้องใช้
- [ ] มี SPEF
- [ ] พบขา `VPWR` และ `VGND` ใน LEF
- [ ] คัดลอก `final/` มาไว้ใน `counter_8bit/`

### Macro Integration

- [ ] Top-level หา Macro พบ
- [ ] Instance `counter_0` ถึง `counter_3` ปรากฏครบ
- [ ] Macro ไม่ overlap
- [ ] Macro อยู่ภายใน die
- [ ] `PDN_MACRO_CONNECTIONS` ครบ 4 instances
- [ ] Top-level PDN เชื่อม Macro ครบ
- [ ] ไม่มี floating power pin
- [ ] Signal routing ครบ
- [ ] Final GDS ถูกสร้าง
- [ ] ไม่มี fatal DRC/LVS error

---

## 4.52 สรุป

Exercise 4 แสดงกระบวนการสำคัญของการออกแบบ ASIC แบบลำดับชั้น ตั้งแต่การสร้าง RTL block ให้เป็น Hard Macro ไปจนถึงการนำ Macro หลายตัวมาประกอบเป็นระบบที่ใหญ่ขึ้น

หัวใจสำคัญของ Exercise ไม่ได้อยู่เพียงการประกาศ Macro ใน YAML แต่รวมถึงการจัดการข้อมูลหลายมุมมอง ได้แก่

- Logical view
- Physical abstract view
- Layout view
- Timing view
- Parasitic view
- Power connectivity

การบูรณาการ Macro ที่ถูกต้องต้องทำให้ชื่อ module, instance, file path, timing corner, power pin และ physical placement สอดคล้องกันทั้งหมด

แนวคิดเดียวกันนี้สามารถขยายไปสู่การออกแบบที่ซับซ้อนกว่า เช่น

- CPU subsystem
- SRAM-based SoC
- DSP accelerator
- Hierarchical RISC-V SoC
- Multi-power-domain design
- Full-chip integration
- Chiplet หรือ multi-die subsystem

ดังนั้น Exercise นี้จึงเป็นพื้นฐานโดยตรงสำหรับการเรียนรู้ Macro Integration, Hierarchical Physical Design และ Full-Chip Design ในบทปฏิบัติการขั้นถัดไป
