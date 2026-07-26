
# Lab 2 
# การกำหนดค่า Physical Design ด้วย LibreLane Configuration Variables

## 2.1 วัตถุประสงค์ของบทปฏิบัติการ

ในบทปฏิบัติการนี้ ผู้เรียนจะศึกษาวิธีควบคุมกระบวนการ RTL-to-GDSII ของ LibreLane ผ่านไฟล์ `config.yaml` โดยเน้นตัวแปรที่มีผลต่อ Floorplanning, Placement, Pin Placement และ Routing

เมื่อจบบทปฏิบัติการ ผู้เรียนควรสามารถ

1. กำหนดขนาด Die แบบคงที่ได้
2. อธิบายความแตกต่างระหว่าง Die Area, Core Area และ Placement Density ได้
3. ทดลองหาขนาด Die และ Placement Density ที่เหมาะสม
4. กำหนดตำแหน่งขาอินพุตและเอาต์พุตด้วยไฟล์ `pins.cfg`
5. ใช้ DEF Template เพื่อบังคับขนาด Die และตำแหน่งขา
6. สร้างพื้นที่ห้ามวางเซลล์แบบ Firm และ Soft Obstruction
7. ตรวจสอบผลลัพธ์ด้วย OpenROAD GUI
8. วิเคราะห์ความสัมพันธ์ระหว่างพื้นที่ ความหนาแน่น Congestion และความสามารถในการทำ Routing
9. ตรวจสอบผลการทำ DRC, LVS, Antenna Check และ Timing
10. ใช้ Configuration Variables อื่นเพื่อปรับแต่ง Synthesis และ Routing

---

## 2.2 แนวคิดพื้นฐาน

### 2.2.1 Configuration Variables คืออะไร

LibreLane ประกอบด้วยขั้นตอนย่อยหลายขั้นตอน เช่น

```text
RTL
 │
 ▼
Yosys Synthesis
 │
 ▼
Floorplanning
 │
 ▼
I/O Placement
 │
 ▼
Global Placement
 │
 ▼
Clock Tree Synthesis
 │
 ▼
Detailed Placement
 │
 ▼
Global Routing
 │
 ▼
Detailed Routing
 │
 ▼
DRC / LVS / Antenna Check
 │
 ▼
GDSII
```

แต่ละขั้นตอนมีพารามิเตอร์สำหรับควบคุมพฤติกรรมของเครื่องมือ ตัวอย่างเช่น

- ขนาด Die
- ระยะขอบ Core
- เป้าหมายความหนาแน่นของ Placement
- ตำแหน่งขา I/O
- กลยุทธ์การสังเคราะห์
- การยอมให้เกิด Global Routing Congestion
- จำนวนรอบการแก้ปัญหา Antenna
- พื้นที่ห้ามวาง Standard Cell

ตัวแปรเหล่านี้เขียนไว้ในไฟล์ `config.yaml` เพื่อให้การทดลองสามารถทำซ้ำได้ และช่วยให้ Configuration ของการออกแบบอยู่ภายใต้ Version Control

---

### 2.2.2 ความแตกต่างระหว่าง Die Area และ Core Area

**Die Area** คือขอบเขตทั้งหมดของชิปหรือ Macro

**Core Area** คือบริเวณภายใน Die ที่ LibreLane สร้าง Standard-cell Rows และอนุญาตให้วาง Standard Cells

โครงสร้างโดยประมาณเป็นดังนี้

```text
+---------------------------------------------------+
|                    Die Area                       |
|                                                   |
|   Die/Core Margin                                 |
|   +-------------------------------------------+   |
|   |                                           |   |
|   |                 Core Area                 |   |
|   |                                           |   |
|   |       Standard Cells และ Routing          |   |
|   |                                           |   |
|   +-------------------------------------------+   |
|                                                   |
+---------------------------------------------------+
```

ค่า Margin ใน `config.yaml` มีผลต่อระยะห่างระหว่างขอบ Die และขอบ Core

```yaml
TOP_MARGIN_MULT: 1
BOTTOM_MARGIN_MULT: 1
LEFT_MARGIN_MULT: 6
RIGHT_MARGIN_MULT: 6
```

ค่าเหล่านี้อยู่ใน Configuration เริ่มต้นของ Exercise 2 เพื่อช่วยลดพื้นที่ว่างบางส่วนและกำหนดระยะขอบด้านซ้ายและขวาให้เหมาะกับการวางขา I/O 

---

### 2.2.3 Placement Density

Placement Density แสดงสัดส่วนของพื้นที่วางเซลล์ที่เครื่องมือควรใช้ โดยประมาณสามารถมองได้ว่า

$$\text{Placement Density}\approx \frac{\text{พื้นที่ของ Standard Cells}} {\text{พื้นที่ที่ Global Placement ใช้งาน}} \times 100$$

ตัวแปรที่ใช้คือ

```yaml
PL_TARGET_DENSITY_PCT: 80
```

ค่า `80` หมายถึงกำหนดเป้าหมาย Placement Density เท่ากับ 80%

อย่างไรก็ตาม Placement Density ไม่ใช่ Global Core Utilization โดยตรง เครื่องมือ Placement อาจรวมพื้นที่ Padding และพื้นที่ที่ต้องใช้สำหรับการเคลื่อนย้ายเซลล์ระหว่าง Optimization ด้วย

หาก Density ต่ำเกินไป

- เซลล์อาจรวมตัวอยู่เพียงบริเวณหนึ่ง
- Die มีพื้นที่ว่างมาก
- Wirelength อาจเพิ่มขึ้น
- พื้นที่ Silicon ถูกใช้อย่างไม่มีประสิทธิภาพ

หาก Density สูงเกินไป

- เซลล์อยู่ชิดกันมาก
- มีพื้นที่สำหรับ Routing น้อย
- Global Routing อาจเกิด Congestion
- Detailed Routing อาจทำงานไม่สำเร็จ
- การแทรก Buffer หรือแก้ Antenna อาจทำได้ยาก
- อาจเกิด DRC Violation เพิ่มขึ้น

ดังนั้นการเลือก Density เป็นการหาสมดุลระหว่างพื้นที่ ความสามารถในการทำ Routing และ Timing

---

## 2.3 โครงสร้างไฟล์ของ Exercise 2

เข้าสู่โฟลเดอร์ Repository แล้วตรวจสอบไฟล์

```bash
cd heichips26-digital-workshop/exercise_2
tree
```

โครงสร้างหลักควรมีลักษณะดังนี้

```text
exercise_2/
├── config.yaml
├── README.md
├── src/
│   └── project.sv
├── def/
│   ├── tt_block_1x1_pgvdd.def
│   ├── tt_block_1x2_pgvdd.def
│   ├── tt_block_2x1_pgvdd.def
│   ├── tt_block_2x2_pgvdd.def
│   ├── tt_block_3x1_pgvdd.def
│   ├── tt_block_3x2_pgvdd.def
│   ├── tt_block_4x1_pgvdd.def
│   ├── tt_block_4x2_pgvdd.def
│   ├── tt_block_6x1_pgvdd.def
│   ├── tt_block_6x2_pgvdd.def
│   ├── tt_block_8x1_pgvdd.def
│   └── tt_block_8x2_pgvdd.def
└── img/
```

โฟลเดอร์ `def/` มี DEF Template หลายขนาด ตั้งแต่ 1×1 จนถึง 8×2 Tile เพื่อใช้ทดลอง Fixed Floorplan และ Predetermined Pin Placement 

---

## 2.4 ศึกษา RTL Design

เปิดไฟล์ RTL

```bash
sed -n '1,200p' src/project.sv
```

Top-level Module คือ

```systemverilog
module tt_um_example (
    input  wire [7:0] ui_in,
    output wire [7:0] uo_out,

    input  wire [7:0] uio_in,
    output wire [7:0] uio_out,
    output wire [7:0] uio_oe,

    input  wire       ena,
    input  wire       clk,
    input  wire       rst_n
);
```

วงจรทำหน้าที่คูณข้อมูลอินพุต 8 บิตสองชุด

```systemverilog
logic [15:0] product;

always @(posedge clk, negedge rst_n) begin
    if (!rst_n) begin
        product <= '0;
    end else begin
        product <= ui_in * uio_in;
    end
end
```

ผลคูณมีขนาด 16 บิตและถูกแบ่งออกเป็นสองส่วน

```systemverilog
assign uo_out  = product[7:0];
assign uio_out = product[15:8];
```

ดังนั้น

```text
ui_in[7:0] × uio_in[7:0]
              │
              ▼
       product[15:0]
         │         │
         │         └── product[15:8] → uio_out[7:0]
         └──────────── product[7:0]  → uo_out[7:0]
```

สัญญาณ `uio_oe` ถูกกำหนดเป็นศูนย์ทั้งหมด

```systemverilog
assign uio_oe = '0;
```

จึงหมายความว่าเส้นทาง Bidirectional I/O ถูกตั้งให้เป็น Input Mode ตามนิยามของ Interface

```text
uio_oe = 0 → input
uio_oe = 1 → output
```

อินพุต `ena` ไม่ได้ถูกใช้ในฟังก์ชันหลัก จึงถูกรวมไว้ในสัญญาณ `_unused` เพื่อลดคำเตือนจากเครื่องมือตรวจ RTL

```systemverilog
wire _unused = ena;
```

วงจรนี้เหมาะกับการทดลอง Placement เนื่องจากตัวคูณ 8×8 บิตจะสร้าง Combinational Logic จำนวนมากกว่า Counter ขนาดเล็กอย่างชัดเจน ทำให้สามารถสังเกตผลของ Area และ Density ได้ง่ายขึ้น โครงสร้างและพฤติกรรมดังกล่าวอยู่ใน `src/project.sv` ของ Exercise 2 

---

## 2.5 ศึกษา Configuration เริ่มต้น

เปิดไฟล์

```bash
cat config.yaml
```

Configuration เริ่มต้นคือ

```yaml
# LibreLane configuration file

DESIGN_NAME: tt_um_example

VERILOG_FILES: dir::src/*.sv

CLOCK_PORT: clk

CLOCK_PERIOD: 10 # 10ns = 100MHz

# Reduce wasted space
TOP_MARGIN_MULT: 1
BOTTOM_MARGIN_MULT: 1
LEFT_MARGIN_MULT: 6
RIGHT_MARGIN_MULT: 6
```

### ความหมายของแต่ละตัวแปร

#### `DESIGN_NAME`

```yaml
DESIGN_NAME: tt_um_example
```

ระบุชื่อ Top-level Module ของ Design โดยต้องตรงกับชื่อ Module ใน `project.sv`

---

#### `VERILOG_FILES`

```yaml
VERILOG_FILES: dir::src/*.sv
```

ระบุไฟล์ RTL ที่ LibreLane ต้องนำไปสังเคราะห์

คำว่า `dir::` หมายถึงให้ตีความ Path โดยอ้างอิงจาก Directory ที่มีไฟล์ `config.yaml`

Wildcard `*.sv` หมายถึงอ่านไฟล์ SystemVerilog ทุกไฟล์ภายใน `src/`

---

#### `CLOCK_PORT`

```yaml
CLOCK_PORT: clk
```

ระบุชื่อ Clock Port ของ Top-level Module

ชื่อนี้ต้องตรงกับ

```systemverilog
input wire clk
```

---

#### `CLOCK_PERIOD`

```yaml
CLOCK_PERIOD: 10
```

กำหนด Clock Period เท่ากับ 10 ns

ดังนั้นความถี่เป้าหมายคือ

$$f=\frac{1}{T}$$

$$f=\frac{1}{10\text{ ns}}=100\text{ MHz}$$

---

## 2.6 ขั้นตอนที่ 1: ตรวจสอบ Environment

ตรวจสอบว่า LibreLane สามารถทำงานได้

```bash
librelane --version
```

ตรวจสอบตำแหน่งของคำสั่ง

```bash
which librelane
```

จากนั้นตรวจสอบว่ากำลังอยู่ใน Directory ที่ถูกต้อง

```bash
pwd
ls
```

ควรเห็นอย่างน้อย

```text
README.md
config.yaml
def
img
src
```

---

## 2.7 ขั้นตอนที่ 2: สำรอง Configuration เริ่มต้น

ก่อนแก้ไข Configuration ควรสำรองไฟล์เดิมไว้

```bash
cp config.yaml config.base.yaml
```

ตรวจสอบว่าไฟล์ถูกสร้างแล้ว

```bash
ls -l config*.yaml
```

ควรเห็น

```text
config.base.yaml
config.yaml
```

การสำรองนี้ช่วยให้สามารถย้อนกลับไปยัง Baseline ได้ทันที

```bash
cp config.base.yaml config.yaml
```

---

## 2.8 ขั้นตอนที่ 3: รัน Baseline Flow

เริ่มต้นด้วย Configuration เดิมเพื่อสร้างผลลัพธ์อ้างอิง

```bash
librelane --pdk ihp-sg13g2 config.yaml
```

คำสั่งนี้เลือกใช้ IHP SG13G2 PDK และรัน Classic Flow โดยใช้ `config.yaml` เป็น Configuration หลัก รูปแบบคำสั่งเดียวกันถูกใช้ใน Exercise 1 ของ Repository 

ระหว่างการรัน LibreLane จะดำเนินการโดยประมาณดังนี้

```text
Lint/Elaboration
        │
        ▼
Yosys Synthesis
        │
        ▼
Floorplan
        │
        ▼
IO Placement
        │
        ▼
Global Placement
        │
        ▼
Clock Tree Synthesis
        │
        ▼
Detailed Placement
        │
        ▼
Global Routing
        │
        ▼
Detailed Routing
        │
        ▼
Parasitic Extraction
        │
        ▼
STA / DRC / LVS / Antenna
```

หลัง Flow จบ ให้ตรวจสอบ Summary ว่า

```text
Antenna : Passed
LVS     : Passed
DRC     : Passed
```

คำเตือนบางรายการอาจไม่ทำให้ Flow ล้มเหลว แต่ผู้เรียนควรอ่าน Warning Summary ทุกครั้ง

---

## 2.9 ขั้นตอนที่ 4: ตรวจสอบ Run Directory

หลังรัน LibreLane จะสร้าง Directory ชื่อ `runs/` หรือ `run/` ตามเวอร์ชันและ Configuration ของ Environment

ตรวจสอบด้วย

```bash
find . -maxdepth 2 -type d | sort
```

หรือ

```bash
ls -lt runs 2>/dev/null
ls -lt run 2>/dev/null
```

แต่ละ Run จะมีชื่อคล้าย

```text
RUN_2026-07-20_08-15-30
```

ภายในประกอบด้วย

- Log ของ Flow
- Warning และ Error Report
- ผลลัพธ์ของแต่ละ Step
- Netlist หลัง Synthesis
- DEF และ ODB ระหว่างกระบวนการ
- Timing Report
- Routing Report
- GDSII
- DRC/LVS Report

แนวคิดของ Run Directory และ State ระหว่างแต่ละ Step อธิบายไว้ใน Exercise 1 ของ Repository 

---

# 2.10 การทดลองที่ 2.1: กำหนด Die Area แบบคงที่

## 2.10.1 Relative Sizing และ Absolute Sizing

โดยปกติ LibreLane สามารถคำนวณขนาด Die จากพื้นที่ของ Standard Cells และค่า Utilization ได้อัตโนมัติ เมื่อใช้

```yaml
FP_SIZING: relative
```

แต่หากผู้ออกแบบต้องการกำหนดขนาดตายตัว เช่น

- มีข้อจำกัดด้านพื้นที่
- ต้องวาง Macro ลงในช่องที่กำหนดไว้
- ต้องเชื่อมต่อกับ Top-level Floorplan
- ต้องใช้ Tile Template
- ต้องส่ง Design เข้าโครงการแบบ Fixed Slot

ให้ใช้

```yaml
FP_SIZING: absolute
```

Exercise 2 กำหนดให้เริ่มจาก Die ขนาด 150 µm × 150 µm และ Placement Density 80% 

---

## 2.10.2 แก้ไขไฟล์ `config.yaml`

เพิ่มตัวแปรต่อไปนี้ท้ายไฟล์

```yaml
FP_SIZING: absolute
DIE_AREA: [0, 0, 150, 150]
PL_TARGET_DENSITY_PCT: 80
```

ไฟล์ฉบับสมบูรณ์ควรเป็น

```yaml
# LibreLane configuration file

DESIGN_NAME: tt_um_example

VERILOG_FILES: dir::src/*.sv

CLOCK_PORT: clk
CLOCK_PERIOD: 10

TOP_MARGIN_MULT: 1
BOTTOM_MARGIN_MULT: 1
LEFT_MARGIN_MULT: 6
RIGHT_MARGIN_MULT: 6

FP_SIZING: absolute
DIE_AREA: [0, 0, 150, 150]
PL_TARGET_DENSITY_PCT: 80
```

---

## 2.10.3 ความหมายของ `DIE_AREA`

รูปแบบของตัวแปรคือ

```yaml
DIE_AREA: [x_min, y_min, x_max, y_max]
```

สำหรับ

```yaml
DIE_AREA: [0, 0, 150, 150]
```

จะได้

```text
จุดมุมซ้ายล่าง = (0, 0) µm
จุดมุมขวาบน  = (150, 150) µm
ความกว้าง     = 150 - 0 = 150 µm
ความสูง       = 150 - 0 = 150 µm
```

พื้นที่ Die เท่ากับ

$$A_{\text{die}}=150\times150$$

$$A_{\text{die}}=22,500\ \mu m^2$$

หรือ

$$A_{\text{die}}=0.0225\ mm^2$$

ข้อควรระวังคือ `DIE_AREA` ไม่ได้ระบุเป็น `[x, y, width, height]` แต่ระบุเป็นพิกัดมุมซ้ายล่างและมุมขวาบน

---

## 2.10.4 รัน Flow

เพื่อแยกผลการทดลองจาก Baseline สามารถกำหนด Run Tag

```bash
librelane \
  --pdk ihp-sg13g2 \
  --run-tag exp2_1_area150_density80 \
  config.yaml
```

หาก LibreLane เวอร์ชันที่ใช้ไม่รองรับ `--run-tag` ให้ใช้คำสั่งปกติ

```bash
librelane --pdk ihp-sg13g2 config.yaml
```

---

## 2.10.5 เปิดผลลัพธ์ใน OpenROAD GUI

```bash
librelane \
  --pdk ihp-sg13g2 \
  config.yaml \
  --last-run \
  --flow OpenInOpenROAD
```

`--last-run` สั่งให้ LibreLane ใช้ State จาก Run ล่าสุด ส่วน `OpenInOpenROAD` ใช้เปิดผลลัพธ์ใน OpenROAD GUI โดยไม่รัน Physical Design ใหม่ รูปแบบคำสั่งนี้ใช้ใน Exercise 1 เช่นกัน 

---

## 2.10.6 สิ่งที่ต้องสังเกต

ใน OpenROAD GUI ให้ตรวจสอบ

1. ขอบเขต Die
2. ขอบเขต Core
3. ตำแหน่ง Standard Cells
4. พื้นที่ว่างภายใน Core
5. การกระจายตัวของเซลล์
6. ตำแหน่งขาอินพุตและเอาต์พุต
7. Routing Tracks
8. Power Straps
9. Clock Tree
10. Routing Congestion

เมื่อ Die มีขนาดใหญ่เมื่อเทียบกับ Design แต่ตั้ง Target Density สูง เซลล์อาจถูกรวมตัวเป็นกลุ่มอยู่บริเวณกึ่งกลาง แทนที่จะกระจายเต็มพื้นที่ Die ซึ่งเป็นพฤติกรรมที่ Exercise 2 ต้องการให้สังเกต 
---

# 2.11 การทดลองหาขนาด Die และ Density ที่เหมาะสม

## 2.11.1 จุดประสงค์

เป้าหมายคือหาค่าที่

- ใช้พื้นที่น้อย
- Placement ผ่าน
- CTS ผ่าน
- Routing ผ่าน
- DRC ผ่าน
- LVS ผ่าน
- Antenna Check ผ่าน
- Timing อยู่ในเกณฑ์
- ไม่มี Congestion รุนแรง

ไม่ควรพิจารณาเฉพาะว่าคำสั่งจบโดยไม่มี Error เท่านั้น

---

## 2.11.2 วิธีทดลองแบบเป็นระบบ

เริ่มต้นด้วยการคง Density ไว้ที่ 80% แล้วลดขนาด Die

ตัวอย่างชุดทดลอง

```text
150 × 150 µm
140 × 140 µm
130 × 130 µm
120 × 120 µm
110 × 110 µm
100 × 100 µm
```

ตัวอย่าง Configuration

```yaml
FP_SIZING: absolute
DIE_AREA: [0, 0, 130, 130]
PL_TARGET_DENSITY_PCT: 80
```

เมื่อพบขนาดต่ำสุดที่ Flow ยังผ่าน ให้ตรึงขนาดนั้นไว้ แล้วเปลี่ยน Density

```text
60%
65%
70%
75%
80%
85%
90%
```

ไม่แนะนำให้เริ่มจาก 100% เพราะ Physical Design ต้องใช้พื้นที่สำหรับ

- Routing
- Buffer Insertion
- Clock Buffers
- Hold-fix Buffers
- Antenna Diodes
- Cell Resizing
- Legalization

---

## 2.11.3 ตารางบันทึกผล

ให้ผู้เรียนบันทึกผลในรูปแบบต่อไปนี้

| ครั้งที่ | Die Area | Density | Placement | Routing | DRC | LVS | Antenna | WNS | หมายเหตุ |
|---:|---|---:|---|---|---|---|---|---:|---|
| 1 | 150×150 | 80% | Pass | Pass | Pass | Pass | Pass |  | Baseline |
| 2 | 140×140 | 80% |  |  |  |  |  |  |  |
| 3 | 130×130 | 80% |  |  |  |  |  |  |  |
| 4 | 120×120 | 80% |  |  |  |  |  |  |  |
| 5 |  | 85% |  |  |  |  |  |  |  |
| 6 |  | 90% |  |  |  |  |  |  |  |

---

## 2.11.4 เกณฑ์ตัดสิน Configuration ที่เหมาะสม

Configuration ที่เหมาะสมควรมีคุณสมบัติ

```text
Synthesis      : ผ่าน
Floorplan      : ผ่าน
Placement      : ผ่าน
CTS            : ผ่าน
Global Route   : ไม่มี Congestion รุนแรง
Detailed Route : ผ่าน
STA            : WNS ≥ 0 หรืออยู่ในเกณฑ์ที่ยอมรับ
DRC            : 0
LVS            : ผ่าน
Antenna        : ผ่าน
```

การเลือก Die ที่เล็กที่สุดโดย Routing ผ่านอย่างเฉียดฉิวอาจไม่ใช่คำตอบที่ดีที่สุด เพราะ Design อาจไม่มี Margin เพียงพอสำหรับการแก้ไข RTL หรือ Timing ในอนาคต

---

# 2.12 การทดลองที่ 2.2: กำหนดตำแหน่งขา I/O

## 2.12.1 พฤติกรรมเริ่มต้นของ LibreLane

ใน Classic Flow การวาง Standard Cells และ I/O Pins เกี่ยวข้องกับลำดับขั้นตอนโดยประมาณดังนี้

```text
OpenROAD.GlobalPlacementSkipIO
              │
              ▼
OpenROAD.IOPlacement
              │
              ▼
OpenROAD.GlobalPlacement
```

ขั้นแรก Standard Cells ถูกวางโดยยังไม่คำนึงถึง I/O Pins จากนั้นจึงวางขา I/O และปรับ Placement อีกครั้งโดยพิจารณาตำแหน่งขาแล้ว ลำดับดังกล่าวระบุไว้ในคำอธิบาย Exercise 2 

ในงานจริง ตำแหน่งขามักถูกกำหนดโดย

- Interface กับ Macro ข้างเคียง
- Top-level Bus Routing
-ตำแหน่ง SRAM หรือ Analog Macro
- ทิศทางการไหลของข้อมูล
- Feedthrough Path
- Packaging หรือ Pad Ring
- Floorplan ของระบบระดับบน

---

## 2.12.2 กำหนดเป้าหมาย

ในการทดลองนี้กำหนดให้

- อินพุตทั้งหมดอยู่ด้านซ้าย
- เอาต์พุตทั้งหมดอยู่ด้านขวา
- Clock และ Reset อยู่ด้านล่าง
- สัญญาณควบคุม `ena` อยู่ด้านล่าง

อินพุตของวงจรคือ

```text
ui_in[7:0]
uio_in[7:0]
ena
clk
rst_n
```

เอาต์พุตคือ

```text
uo_out[7:0]
uio_out[7:0]
uio_oe[7:0]
```

รายชื่อเหล่านี้มาจาก Top-level Interface ใน `project.sv` 

---

## 2.12.3 สร้างไฟล์ `pins.cfg`

สร้างไฟล์

```bash
nano pins.cfg
```

หรือ

```bash
cat > pins.cfg <<'EOF'
#N
clk
rst_n
ena

#S

#E
uo_out.*
uio_out.*
uio_oe.*

#W
ui_in.*
uio_in.*
EOF
```

อย่างไรก็ตาม รูปแบบที่แนะนำตามโจทย์ “Input ซ้าย Output ขวา” และแยก Clock/Reset ไว้ด้านล่าง คือ

```text
#N

#S
clk
rst_n
ena

#E
uo_out.*
uio_out.*
uio_oe.*

#W
ui_in.*
uio_in.*
```

ความหมายของ Section คือ

```text
#N = North หรือด้านบน
#S = South หรือด้านล่าง
#E = East หรือด้านขวา
#W = West หรือด้านซ้าย
```

เครื่องหมาย `.*` ใช้จับ Bus Bit ทั้งหมด เช่น

```text
ui_in.*
```

ครอบคลุม

```text
ui_in[0]
ui_in[1]
...
ui_in[7]
```

---

## 2.12.4 เพิ่ม Configuration

เพิ่มใน `config.yaml`

```yaml
FP_PIN_ORDER_CFG: dir::pins.cfg
```

Configuration ที่เกี่ยวข้องควรเป็น

```yaml
FP_SIZING: absolute
DIE_AREA: [0, 0, 150, 150]
PL_TARGET_DENSITY_PCT: 80

FP_PIN_ORDER_CFG: dir::pins.cfg
```

Exercise 2 กำหนดให้ชี้ `FP_PIN_ORDER_CFG` ไปยังไฟล์ Pin Placement Configuration ด้วยรูปแบบ `dir::pins.cfg` 

---

## 2.12.5 รัน Flow ใหม่

```bash
librelane \
  --pdk ihp-sg13g2 \
  --run-tag exp2_2_custom_pins \
  config.yaml
```

หรือ

```bash
librelane --pdk ihp-sg13g2 config.yaml
```

---

## 2.12.6 ตรวจสอบใน OpenROAD GUI

```bash
librelane \
  --pdk ihp-sg13g2 \
  config.yaml \
  --last-run \
  --flow OpenInOpenROAD
```

ใน GUI ให้ซูมบริเวณขอบ Die และตรวจสอบว่า

```text
ด้านซ้าย  : ui_in[7:0], uio_in[7:0]
ด้านขวา  : uo_out[7:0], uio_out[7:0], uio_oe[7:0]
ด้านล่าง : clk, rst_n, ena
```

---

## 2.12.7 ปัญหาที่พบบ่อย

### ชื่อ Pin ไม่ตรงกับ RTL

ตัวอย่างที่ผิด

```text
ui_in
```

แต่เครื่องมืออาจต้องการจับ Bus Bit จึงควรใช้

```text
ui_in.*
```

---

### ลืมใส่ Pin บางตัว

หากไฟล์ `pins.cfg` ไม่ครอบคลุมทุก Port เครื่องมืออาจ

- แจ้ง Error
- วาง Pin ที่เหลืออัตโนมัติ
- ทำให้ผลไม่ตรงตามที่คาด

ควรตรวจรายชื่อ Port จาก RTL ทุกครั้ง

```bash
grep -n "input\|output" src/project.sv
```

---

### Pin หนาแน่นเกินไปในด้านเดียว

ด้านขวาของ Design มี Output รวม 24 เส้น

```text
uo_out  = 8
uio_out = 8
uio_oe  = 8
รวม      = 24 pins
```

หาก Die มีความสูงน้อยเกินไป Pin Pitch อาจไม่เพียงพอ เครื่องมืออาจแจ้งว่าไม่สามารถวาง Pins ได้ครบ

วิธีแก้คือ

- เพิ่มความสูง Die
- กระจาย Output ไปสองด้าน
- ปรับ Pin Placement Constraints
- ลดจำนวน Pin ที่อยู่ด้านเดียวกัน

---

# 2.13 การทดลองที่ 2.3: ใช้ DEF Template

## 2.13.1 DEF คืออะไร

DEF ย่อมาจาก Design Exchange Format เป็นไฟล์ที่ใช้เก็บข้อมูล Physical Design เช่น

- Die Area
- Rows
- Tracks
- Pin Locations
- Pin Directions
- Components
- Nets
- Routing Geometry
- Blockages
- Special Nets

DEF Template ใช้กำหนดโครงสร้าง Physical เริ่มต้น โดยเฉพาะ

- ขนาด Die คงที่
- ตำแหน่ง Pin คงที่
- Routing Grid คงที่
- Tile Interface คงที่
- Compatibility กับระบบระดับบน

Exercise 2 ใช้ Tiny Tapeout-style DEF Templates ซึ่งอยู่ในโฟลเดอร์ `def/` 

---

## 2.13.2 เลือก Template ขนาด 1×1

ใช้ไฟล์

```text
def/tt_block_1x1_pgvdd.def
```

ตรวจสอบส่วนต้นของไฟล์

```bash
head -n 40 def/tt_block_1x1_pgvdd.def
```

ค้นหา `DIEAREA`

```bash
grep -n "DIEAREA" def/tt_block_1x1_pgvdd.def
```

ค่าที่ได้อาจอยู่ในหน่วย Database Unit ไม่ใช่ไมโครเมตรโดยตรง จึงควรตรวจค่า `UNITS DISTANCE MICRONS` ด้วย

```bash
grep -n "UNITS\|DIEAREA" def/tt_block_1x1_pgvdd.def
```

---

## 2.13.3 แก้ไข Configuration

นำ Configuration ของ Custom Pin Placement ออกหรือ Comment ก่อน เพื่อไม่ให้ขัดกับ Pin Location ใน DEF Template

```yaml
# FP_PIN_ORDER_CFG: dir::pins.cfg
```

จากนั้นเพิ่ม

```yaml
FP_SIZING: absolute
DIE_AREA: [0, 0, 202.08, 154.98]
FP_DEF_TEMPLATE: dir::def/tt_block_1x1_pgvdd.def
```

Configuration ตัวอย่าง

```yaml
# LibreLane configuration file

DESIGN_NAME: tt_um_example
VERILOG_FILES: dir::src/*.sv

CLOCK_PORT: clk
CLOCK_PERIOD: 10

TOP_MARGIN_MULT: 1
BOTTOM_MARGIN_MULT: 1
LEFT_MARGIN_MULT: 6
RIGHT_MARGIN_MULT: 6

FP_SIZING: absolute
DIE_AREA: [0, 0, 202.08, 154.98]

FP_DEF_TEMPLATE: dir::def/tt_block_1x1_pgvdd.def
```

ขนาด `202.08 × 154.98 µm` ต้องตรงกับ Die Area ของ Template 1×1 ตามตัวอย่างใน Exercise 2 มิฉะนั้น LibreLane จะรายงานความไม่ตรงกัน 

---

## 2.13.4 รัน Flow

```bash
librelane \
  --pdk ihp-sg13g2 \
  --run-tag exp2_3_def_1x1 \
  config.yaml
```

---

## 2.13.5 ตรวจสอบผลลัพธ์

เปิด OpenROAD GUI

```bash
librelane \
  --pdk ihp-sg13g2 \
  config.yaml \
  --last-run \
  --flow OpenInOpenROAD
```

ตรวจสอบ

- Die Area ตรงกับ Template
- Pin อยู่ในตำแหน่งที่กำหนด
- ชื่อ Pin ตรงกับ Top-level Ports
- Standard Cells อยู่ภายใน Core
- ไม่มี Pin ซ้อนกัน
- Routing เชื่อมต่อถึง Pin ได้
- Power Distribution ไม่ขัดกับ Boundary

---

## 2.13.6 เหตุใดต้องระบุทั้ง `DIE_AREA` และ `FP_DEF_TEMPLATE`

แม้ DEF Template มีข้อมูล Die Area อยู่แล้ว แต่ LibreLane ยังคงใช้ `DIE_AREA` เป็น Configuration Constraint และตรวจสอบความสอดคล้องระหว่าง

```text
DIE_AREA ใน config.yaml
          กับ
DIEAREA ใน DEF Template
```

หากค่าไม่ตรงกัน Flow จะหยุดเพื่อป้องกันการสร้าง Layout ที่มี Boundary ไม่สอดคล้องกัน

---

## 2.13.7 ทดลอง Template ขนาดอื่น

ตัวอย่าง Template ที่มีให้ ได้แก่

```text
tt_block_1x2_pgvdd.def
tt_block_2x1_pgvdd.def
tt_block_2x2_pgvdd.def
tt_block_3x1_pgvdd.def
tt_block_3x2_pgvdd.def
tt_block_4x1_pgvdd.def
tt_block_4x2_pgvdd.def
tt_block_6x1_pgvdd.def
tt_block_6x2_pgvdd.def
tt_block_8x1_pgvdd.def
tt_block_8x2_pgvdd.def
```

ก่อนเปลี่ยน Template ต้องตรวจ `DIEAREA` ของไฟล์นั้น แล้วแก้ `DIE_AREA` ให้ตรงกัน

ตัวอย่าง

```bash
for file in def/*.def; do
    echo "===== $file ====="
    grep -E "UNITS DISTANCE MICRONS|DIEAREA" "$file"
done
```

---

# 2.14 การทดลองที่ 2.4: สร้าง Placement Obstruction

## 2.14.1 Placement Obstruction คืออะไร

Placement Obstruction คือพื้นที่ที่ไม่อนุญาตหรือไม่ต้องการให้วาง Standard Cells

ใช้ในกรณี

- กันพื้นที่สำหรับ Macro
- กันพื้นที่สำหรับ Analog Block
- สร้าง Routing Channel
- กันพื้นที่รอบ Clock Trunk
- กันพื้นที่บริเวณ Feedthrough
- เตรียมพื้นที่สำหรับ ECO
- ลด Congestion เฉพาะจุด
- หลีกเลี่ยงบริเวณที่มี Power Straps หนาแน่น

Exercise 2 แนะนำตัวแปรสองชนิด 

```yaml
FP_OBSTRUCTIONS
PL_SOFT_OBSTRUCTIONS
```

---

## 2.14.2 Firm Obstruction

`FP_OBSTRUCTIONS` เป็น Obstruction ระดับ Floorplanning

พื้นที่ที่กำหนดจะไม่มี Placement Sites หรือ Standard-cell Rows ถูกสร้างขึ้น จึงไม่มีการวางเซลล์ในพื้นที่นั้นตลอด Flow

ตัวอย่าง

```yaml
FP_OBSTRUCTIONS:
  - [30, 30, 40, 40]
  - [120, 100, 150, 115]
  - [100, 20, 180, 30]
```

แต่ละรายการอยู่ในรูป

```text
[x_min, y_min, x_max, y_max]
```

---

## 2.14.3 Soft Obstruction

`PL_SOFT_OBSTRUCTIONS` ใช้ห้ามวางเซลล์ในช่วง Initial Placement แต่ในขั้นตอนหลัง เครื่องมืออาจใช้พื้นที่นั้นสำหรับ

- Buffer Insertion
- Hold Fix
- Antenna Diode
- Resizing
- Timing Optimization

ตัวอย่าง

```yaml
PL_SOFT_OBSTRUCTIONS:
  - [50, 50, 70, 70]
```

Soft Obstruction จึงเหมาะกับพื้นที่ที่ต้องการหลีกเลี่ยงในขั้นแรก แต่ยังยอมให้เครื่องมือใช้ได้เมื่อจำเป็น

---

## 2.14.4 เพิ่ม Firm Obstruction

ใช้ DEF Template ขนาด 1×1 แล้วเพิ่ม

```yaml
FP_OBSTRUCTIONS:
  - [30, 30, 40, 40]
  - [120, 100, 150, 115]
  - [100, 20, 180, 30]
```

Configuration ส่วน Physical Design จะเป็น

```yaml
FP_SIZING: absolute
DIE_AREA: [0, 0, 202.08, 154.98]
FP_DEF_TEMPLATE: dir::def/tt_block_1x1_pgvdd.def

FP_OBSTRUCTIONS:
  - [30, 30, 40, 40]
  - [120, 100, 150, 115]
  - [100, 20, 180, 30]
```

---

## 2.14.5 ตรวจสอบพิกัดก่อนรัน

ทุก Rectangle ต้องเป็นไปตามเงื่อนไข

$$x_{\min}<x_{\max}$$

$$y_{\min}<y_{\max}$$

และควรอยู่ภายใน Die

$$0\leq x\leq202.08$$

$$0\leq y\leq154.98$$

ตัวอย่างแรก

```text
[30, 30, 40, 40]
```

มีขนาด

```text
กว้าง = 40 - 30 = 10 µm
สูง   = 40 - 30 = 10 µm
พื้นที่ = 100 µm²
```

ตัวอย่างที่สาม

```text
[100, 20, 180, 30]
```

มีขนาด

```text
กว้าง = 80 µm
สูง   = 10 µm
พื้นที่ = 800 µm²
```

---

## 2.14.6 รัน Flow

```bash
librelane \
  --pdk ihp-sg13g2 \
  --run-tag exp2_4_firm_obstruction \
  config.yaml
```

---

## 2.14.7 ตรวจสอบผล

เปิด OpenROAD GUI

```bash
librelane \
  --pdk ihp-sg13g2 \
  config.yaml \
  --last-run \
  --flow OpenInOpenROAD
```

ตรวจสอบว่าบริเวณ Obstruction ไม่มี Standard Cells

ให้เปิดหรือปิดการแสดงผล

- Rows
- Placement Blockages
- Instances
- Routing
- Congestion Heatmap

เพื่อดูผลของ Obstruction อย่างชัดเจน

---

## 2.14.8 ทดลองเพิ่มจำนวนช่องว่าง

เพิ่ม Obstruction ทีละส่วนและรัน Flow ใหม่

ตัวอย่าง

```yaml
FP_OBSTRUCTIONS:
  - [20, 20, 30, 30]
  - [40, 20, 50, 30]
  - [60, 20, 70, 30]
  - [80, 20, 90, 30]
  - [100, 20, 110, 30]
```

สิ่งที่ต้องวิเคราะห์คือ

- Placement ยังผ่านหรือไม่
- Cell Density ในพื้นที่ที่เหลือเพิ่มขึ้นเท่าใด
- Routing Congestion ย้ายไปบริเวณใด
- Timing แย่ลงหรือไม่
- Wirelength เพิ่มขึ้นหรือไม่
- Detailed Routing ยังผ่านหรือไม่

เป้าหมายของ Exercise คือสร้าง Obstruction ให้มากที่สุดโดยไม่ทำให้ Flow ล้มเหลว แต่ในงานจริง Obstruction ต้องมีเหตุผลทางสถาปัตยกรรม ไม่ควรสร้างเพียงเพื่อให้ Layout มีช่องว่างจำนวนมาก

---

# 2.15 การทดลองที่ 2.5: Configuration Variables อื่นที่สำคัญ

Exercise 2 แนะนำตัวแปรเพิ่มเติม ได้แก่ `SYNTH_STRATEGY`, `VERILOG_DEFINES`, `GRT_ALLOW_CONGESTION`, `GRT_ANTENNA_ITERS` และ `RSZ_DONT_TOUCH_RX` 

---

## 2.15.1 `SYNTH_STRATEGY`

ใช้เลือกกลยุทธ์การสังเคราะห์ เช่น เน้นพื้นที่หรือเน้น Timing

แนวคิดทั่วไป

```text
Area-oriented strategy
    ลดจำนวน Gate
    ลดพื้นที่
    อาจทำให้ Critical Path ยาวขึ้น

Delay-oriented strategy
    ลด Delay
    เพิ่ม Buffer หรือ Logic Duplication ได้
    อาจใช้พื้นที่และกำลังมากขึ้น
```

ก่อนใช้ต้องตรวจสอบค่าที่ LibreLane เวอร์ชันปัจจุบันรองรับ

```bash
librelane --help
```

หรือดู Configuration Reference ของ Environment ที่ติดตั้งอยู่

การทดลองควรเปรียบเทียบ

- Standard-cell Area
- Cell Count
- WNS/TNS
- Buffer Count
- Total Power
- Routing Congestion

---

## 2.15.2 `VERILOG_DEFINES`

ใช้ส่ง Preprocessor Definition ไปยังขั้นตอนอ่าน RTL และ Synthesis

ตัวอย่างใน RTL

```systemverilog
`ifdef FAST_MODE
    // implementation สำหรับความเร็ว
`else
    // implementation สำหรับพื้นที่
`endif
```

ใน Configuration อาจกำหนดเป็นรายการตามรูปแบบที่ LibreLane เวอร์ชันนั้นรองรับ เช่น

```yaml
VERILOG_DEFINES:
  - FAST_MODE
```

ประโยชน์คือสามารถใช้ RTL ชุดเดียวเพื่อสร้างหลาย Configuration โดยไม่แก้ Source Code

---

## 2.15.3 `GRT_ALLOW_CONGESTION`

Global Router อาจหยุด Flow เมื่อพบ Congestion ที่ไม่สามารถแก้ได้

การกำหนด

```yaml
GRT_ALLOW_CONGESTION: true
```

อนุญาตให้ Flow ดำเนินต่อแม้ Global Routing ยังรายงาน Congestion

แต่ไม่ได้หมายความว่า Design ใช้งานได้แน่นอน เพราะ Detailed Routing อาจยังล้มเหลวหรือเกิด DRC จำนวนมาก

ควรใช้เพื่อ

- Debug
- ศึกษาพื้นที่ Congestion
- เปิด GUI เพื่อตรวจ Heatmap
- ทดลองว่าขั้น Detailed Routing สามารถแก้ปัญหาได้หรือไม่

ไม่ควรใช้เพื่อซ่อนปัญหา Congestion ใน Final Signoff

---

## 2.15.4 `GRT_ANTENNA_ITERS`

ใช้กำหนดจำนวนรอบการแก้ Antenna Violation ระหว่าง Routing

การเพิ่มจำนวน Iteration อาจช่วยลด Antenna Violation แต่มีผลข้างเคียงได้ เช่น

- เพิ่ม Diode
- เพิ่ม Routing Detour
- เพิ่ม Capacitance
- เพิ่ม Area
- กระทบ Timing
- เพิ่ม Runtime

จึงควรเพิ่มเมื่อ Antenna Check ไม่ผ่านจริง ไม่ควรกำหนดค่าสูงโดยไม่มีเหตุผล

---

## 2.15.5 `RSZ_DONT_TOUCH_RX`

ใช้กำหนด Regular Expression เพื่อระบุ Net หรือ Instance ที่ไม่ต้องการให้ Resizer เปลี่ยนแปลง

ตัวอย่างการใช้งานเชิงแนวคิด

```yaml
RSZ_DONT_TOUCH_RX: "^critical_instance.*"
```

เหมาะกับ

- Clock Generation Circuit
- Synchronizer
- Hand-crafted Datapath
- Constant Network
- Macro Interface
- Logic ที่ต้องรักษา Structure
- Instance ที่ใช้สำหรับ Formal Equivalence หรือ ECO

การใช้ `dont_touch` มากเกินไปจะจำกัดความสามารถของเครื่องมือในการแก้ Timing และ Electrical Violation

---

# 2.16 การตรวจสอบ Timing

เปิด OpenROAD GUI

```bash
librelane \
  --pdk ihp-sg13g2 \
  config.yaml \
  --last-run \
  --flow OpenInOpenROAD
```

เปิด

```text
Windows → Timing Report
```

กด `Update` และตรวจสอบ Critical Path

ค่าที่สำคัญ ได้แก่

```text
Arrival Time
Required Time
Slack
Startpoint
Endpoint
Path Group
Logic Delay
Net Delay
```

นิยามของ Slack คือ

$$\text{Slack} = \text{Required Time} - \text{Arrival Time}$$

ถ้า

```text
Slack > 0  → Timing ผ่าน
Slack = 0  → อยู่ที่ขอบ Constraint
Slack < 0  → Timing Violation
```

ควรเปรียบเทียบ Timing ระหว่าง Configuration เช่น

```text
Die ใหญ่ / Density ต่ำ
เทียบกับ
Die เล็ก / Density สูง
```

Density สูงอาจลดระยะทางบางส่วน แต่หากเกิด Congestion Routing อาจอ้อมมากขึ้นและทำให้ Net Delay เพิ่มขึ้น

---

# 2.17 การตรวจสอบ Clock Tree

ใน OpenROAD GUI เปิด

```text
Windows → Clock Tree Viewer
```

กด `Update`

ตรวจสอบ

- Clock Root
- Clock Buffers
- Clock Leaves
- Clock Depth
- Clock Path
- จำนวน Register ที่ Clock ขับ
- การกระจายของ Clock Buffers
- Clock Routing

วงจร `tt_um_example` มี Register `product[15:0]` จำนวน 16 บิต ดังนั้น Clock Tree ต้องขับ Flip-flop อย่างน้อย 16 ตัวหลัง Synthesis ทั้งนี้จำนวนจริงอาจเปลี่ยนจากการ Optimization

---

# 2.18 การตรวจสอบด้วย KLayout

เปิด Final Layout

```bash
librelane \
  --pdk ihp-sg13g2 \
  config.yaml \
  --last-run \
  --flow OpenInKLayout
```

ใน KLayout ให้ตรวจสอบ

- Die Boundary
- Metal Layers
- Vias
- Standard Cells
- Pin Shapes
- Power Distribution
- Routing รอบ Obstructions
- Routing เข้าสู่ I/O Pins
- Hierarchy ของ Cell
- Layout Density

OpenROAD GUI เหมาะกับการวิเคราะห์ Database และ Timing ส่วน KLayout เหมาะกับการดู Geometry และ GDS/LEF/DEF การเปิด KLayout ผ่าน LibreLane รูปแบบนี้ใช้ใน Exercise 1 ของ Repository 

---

# 2.19 การวิเคราะห์ Congestion

## 2.19.1 Congestion คืออะไร

Routing Congestion เกิดเมื่อความต้องการใช้ Routing Tracks มากกว่าความจุของพื้นที่

$$\text{Congestion Ratio} = \frac{\text{Routing Demand}}{\text{Routing Capacity}}$$

หาก Ratio มากกว่า 1 แสดงว่าความต้องการเกินความสามารถของพื้นที่

สาเหตุทั่วไป ได้แก่

- Placement Density สูง
- Pin หนาแน่น
- Fanout สูง
- Macro ขวางทาง Routing
- Obstruction มากเกินไป
- Die เล็กเกินไป
- Cell Cluster รวมตัวหนาแน่น
- Routing Layer ถูกจำกัด
- Bus จำนวนมากต้องผ่าน Channel แคบ

---

## 2.19.2 วิธีลด Congestion

สามารถทดลอง

1. เพิ่ม Die Area
2. ลด `PL_TARGET_DENSITY_PCT`
3. ลดจำนวน Obstruction
4. เปลี่ยนตำแหน่ง I/O Pins
5. กระจาย Pins ออกหลายด้าน
6. เพิ่ม Routing Channel
7. ปรับ Synthesis Strategy
8. ลด Fanout สูง
9. Pipeline Datapath
10. เปลี่ยน Floorplan

---

# 2.20 แนวทางจัดเก็บ Configuration สำหรับแต่ละการทดลอง

ไม่ควรแก้ `config.yaml` ซ้ำโดยไม่มีสำเนา เพราะจะทำให้ย้อนกลับและเปรียบเทียบผลได้ยาก

แนะนำให้สร้างไฟล์แยก

```text
config.base.yaml
config.area150_density80.yaml
config.custom_pins.yaml
config.def_1x1.yaml
config.firm_obstruction.yaml
config.soft_obstruction.yaml
```

ตัวอย่าง

```bash
cp config.base.yaml config.area150_density80.yaml
cp config.base.yaml config.custom_pins.yaml
cp config.base.yaml config.def_1x1.yaml
```

รันแต่ละไฟล์

```bash
librelane --pdk ihp-sg13g2 config.area150_density80.yaml
```

ข้อควรระวัง: เมื่อใช้ `dir::` Path จะอ้างอิงจาก Directory ของ Configuration File ดังนั้นควรเก็บ Configuration ทั้งหมดไว้ใน `exercise_2/` หรือปรับ Path ให้ถูกต้อง

---

# 2.21 การอ่าน Log และค้นหาปัญหา

ค้นหา Error

```bash
grep -Rni "error" runs 2>/dev/null | head -n 50
grep -Rni "error" run 2>/dev/null | head -n 50
```

ค้นหา Warning

```bash
grep -Rni "warning" runs 2>/dev/null | head -n 50
```

ค้นหา Congestion

```bash
grep -Rni "congestion" runs 2>/dev/null | head -n 50
```

ค้นหา Antenna

```bash
grep -Rni "antenna" runs 2>/dev/null | head -n 50
```

ค้นหา Slack

```bash
grep -Rni "slack" runs 2>/dev/null | head -n 50
```

ค้นหา DRC

```bash
grep -Rni "drc" runs 2>/dev/null | head -n 50
```

ชื่อ Directory และ Report อาจแตกต่างตาม LibreLane Version จึงควรเริ่มจาก

```bash
find runs -maxdepth 3 -type f | less
```

---

# 2.22 Troubleshooting

## ปัญหา 1: Top-level Module Not Found

ข้อความอาจคล้าย

```text
Module tt_um_example not found
```

ตรวจสอบ

```yaml
DESIGN_NAME: tt_um_example
```

และ

```systemverilog
module tt_um_example (
```

ชื่อต้องตรงกันทุกตัวอักษร

---

## ปัญหา 2: Clock Port Not Found

ข้อความอาจคล้าย

```text
Clock port clk not found
```

ตรวจสอบ

```yaml
CLOCK_PORT: clk
```

และ

```systemverilog
input wire clk
```

---

## ปัญหา 3: ไม่พบไฟล์ RTL

ตรวจสอบ

```yaml
VERILOG_FILES: dir::src/*.sv
```

ทดสอบจาก Shell

```bash
ls src/*.sv
```

---

## ปัญหา 4: Die Area เล็กเกินไป

อาการ

- Floorplan ล้มเหลว
- Placement ไม่สามารถวางเซลล์ครบ
- Legalization ล้มเหลว
- Utilization สูงเกินไป

แนวทางแก้

```yaml
DIE_AREA: [0, 0, 170, 170]
```

หรือปรับ Density ให้ต่ำลง

```yaml
PL_TARGET_DENSITY_PCT: 65
```

---

## ปัญหา 5: Routing Congestion

อาการ

- Global Routing แจ้ง Overflow
- Detailed Routing ล้มเหลว
- DRC Violation จำนวนมาก
- Routing อ้อม Obstruction มาก

แนวทางแก้

- เพิ่ม Die Area
- ลด Density
- ลด Obstruction
- กระจาย Pin
- ตรวจ Congestion Heatmap

---

## ปัญหา 6: DEF Template และ Die Area ไม่ตรงกัน

อาการ

```text
DEF template die area does not match configured DIE_AREA
```

ตรวจสอบ

```bash
grep -E "UNITS DISTANCE MICRONS|DIEAREA" \
  def/tt_block_1x1_pgvdd.def
```

แล้วแก้

```yaml
DIE_AREA: [0, 0, 202.08, 154.98]
```

ให้ตรงกับ Template ที่เลือก

---

## ปัญหา 7: Pin ใน Template ไม่ตรงกับ RTL

ตรวจรายชื่อ Pin ใน RTL

```bash
grep -n "input\|output" src/project.sv
```

ตรวจรายชื่อ Pin ใน DEF

```bash
grep -n "^ *- " def/tt_block_1x1_pgvdd.def | head -n 100
```

หากชื่อ Interface ไม่ตรงกัน จะไม่สามารถ Apply Template ได้สมบูรณ์

---

## ปัญหา 8: Pin Placement ไม่ครบ

ตรวจ `pins.cfg`

```bash
cat pins.cfg
```

ตรวจว่า Bus ใช้ Pattern ที่ครอบคลุมทุก Bit เช่น

```text
ui_in.*
uo_out.*
```

---

## ปัญหา 9: Obstruction อยู่นอก Die

ตัวอย่างที่ผิด

```yaml
DIE_AREA: [0, 0, 150, 150]

FP_OBSTRUCTIONS:
  - [120, 100, 180, 160]
```

เนื่องจาก `x_max=180` และ `y_max=160` เกินขอบ Die

แก้เป็น

```yaml
FP_OBSTRUCTIONS:
  - [120, 100, 145, 145]
```

---

## ปัญหา 10: YAML Syntax Error

ตัวอย่างที่ผิด

```yaml
FP_OBSTRUCTIONS:
- [30, 30, 40, 40]
  - [120, 100, 150, 115]
```

ควรเขียนให้ Indentation สม่ำเสมอ

```yaml
FP_OBSTRUCTIONS:
  - [30, 30, 40, 40]
  - [120, 100, 150, 115]
```

ห้ามใช้ Tab ใน YAML ควรใช้ Space

---

# 2.23 แบบฝึกหัดท้ายบท

## แบบฝึกหัด 2.1

กำหนด Die Area เป็น

```yaml
DIE_AREA: [0, 0, 140, 140]
```

แล้วทดลอง Density

```text
60%, 70%, 80%, 90%
```

บันทึก

- Placement Status
- Routing Status
- WNS
- DRC
- LVS
- Antenna
- Runtime

---

## แบบฝึกหัด 2.2

เปลี่ยน Pin Placement ให้

```text
ui_in      → West
uio_in     → South
uo_out     → East
uio_out    → North
uio_oe     → North
clk/rst_n  → South
ena        → West
```

วิเคราะห์ผลต่อ Wirelength และ Congestion

---

## แบบฝึกหัด 2.3

ทดลอง DEF Template อย่างน้อยสามขนาด

```text
1×1
2×1
2×2
```

เปรียบเทียบ

- Die Area
- Core Area
- Cell Density
- Wirelength
- Timing
- Routing Congestion
- Runtime

---

## แบบฝึกหัด 2.4

สร้าง Firm Obstruction อย่างน้อยห้าพื้นที่ โดย Flow ต้องยังผ่าน

บันทึกพิกัดทุก Rectangle และอธิบายผลกระทบต่อ Placement

---

## แบบฝึกหัด 2.5

เปลี่ยน Firm Obstruction ชุดเดียวกันให้เป็น Soft Obstruction แล้วเปรียบเทียบ

```yaml
FP_OBSTRUCTIONS:
```

กับ

```yaml
PL_SOFT_OBSTRUCTIONS:
```

ตรวจว่ามี Buffer หรือ Standard Cell ถูกวางในพื้นที่ Soft Obstruction หลัง Timing Optimization หรือไม่

---

## แบบฝึกหัด 2.6

ลด Clock Period จาก

```yaml
CLOCK_PERIOD: 10
```

เป็น

```yaml
CLOCK_PERIOD: 8
```

และ

```yaml
CLOCK_PERIOD: 5
```

คำนวณความถี่

$$T=8\text{ ns}\Rightarrow f=125\text{ MHz}$$

$$T=5\text{ ns}\Rightarrow f=200\text{ MHz}$$

เปรียบเทียบ

- WNS
- TNS
- Cell Area
- Buffer Count
- Routing Congestion
- Power
- Runtime

---

# 2.24 คำถามวิเคราะห์

1. เพราะเหตุใด Placement Density 100% จึงไม่เหมาะกับ Physical Design จริง
2. Die Area และ Placement Density มีความสัมพันธ์กันอย่างไร
3. เหตุใด Die ที่ใหญ่เกินไปจึงไม่ใช่คำตอบที่ดีที่สุดเสมอ
4. Firm Obstruction แตกต่างจาก Soft Obstruction อย่างไร
5. เหตุใด DEF Template จึงเหมาะกับระบบ Tile-based
6. เหตุใด `DIE_AREA` ต้องตรงกับ `DIEAREA` ใน DEF Template
7. Pin Placement มีผลต่อ Timing อย่างไร
8. Pin Placement มีผลต่อ Routing Congestion อย่างไร
9. การเพิ่ม Antenna Iteration มีผลข้างเคียงอะไร
10. `GRT_ALLOW_CONGESTION` ควรใช้ใน Final Signoff หรือไม่ เพราะเหตุใด
11. เหตุใดการใช้ `dont_touch` จำนวนมากจึงอาจทำให้ Timing Closure ยากขึ้น
12. Configuration ใดควรจัดเก็บใน Version Control
13. Metrics ใดควรใช้ตัดสินว่า Floorplan หนึ่งดีกว่าอีก Floorplan
14. เหตุใดต้องเก็บ Baseline Run ก่อนเริ่ม Optimization
15. เหตุใดการที่ Flow จบโดยไม่มี Error จึงยังไม่เพียงพอสำหรับ Signoff

---

# 2.25 ผลลัพธ์ที่ต้องส่ง

ผู้เรียนต้องส่งไฟล์และหลักฐานดังต่อไปนี้

```text
exercise_2_submission/
├── config.base.yaml
├── config.area_density.yaml
├── config.custom_pins.yaml
├── config.def_template.yaml
├── config.obstructions.yaml
├── pins.cfg
├── screenshots/
│   ├── baseline.png
│   ├── area_density.png
│   ├── custom_pins.png
│   ├── def_template.png
│   ├── firm_obstruction.png
│   ├── congestion.png
│   └── timing_report.png
├── reports/
│   ├── area_density_results.csv
│   ├── timing_summary.txt
│   ├── routing_summary.txt
│   └── signoff_summary.txt
└── report.md
```

รายงานควรประกอบด้วย

1. วัตถุประสงค์
2. Environment และ LibreLane Version
3. PDK ที่ใช้
4. RTL Design Summary
5. Baseline Results
6. Area–Density Experiments
7. Custom Pin Placement
8. DEF Template Experiment
9. Firm/Soft Obstruction Comparison
10. Timing Analysis
11. Congestion Analysis
12. DRC/LVS/Antenna Results
13. Configuration ที่เลือกเป็น Final
14. เหตุผลในการเลือก
15. ปัญหาและวิธีแก้
16. สรุปสิ่งที่เรียนรู้

---

# 2.26 Checklist ก่อนจบบทปฏิบัติการ

```text
[ ] LibreLane ทำงานได้
[ ] ใช้ PDK ihp-sg13g2
[ ] Top-level Module คือ tt_um_example
[ ] Baseline Flow ผ่าน
[ ] กำหนด FP_SIZING เป็น absolute ได้
[ ] กำหนด DIE_AREA ได้ถูกต้อง
[ ] ทดลอง PL_TARGET_DENSITY_PCT หลายค่า
[ ] เปิดผลใน OpenROAD GUI ได้
[ ] สร้าง pins.cfg ได้
[ ] Input Pins อยู่ด้านที่กำหนด
[ ] Output Pins อยู่ด้านที่กำหนด
[ ] ใช้ DEF Template ได้
[ ] DIE_AREA ตรงกับ DEF Template
[ ] สร้าง Firm Obstruction ได้
[ ] ทดลอง Soft Obstruction แล้ว
[ ] ตรวจ Routing Congestion แล้ว
[ ] ตรวจ Timing Report แล้ว
[ ] ตรวจ Clock Tree แล้ว
[ ] เปิด GDS ใน KLayout ได้
[ ] DRC ผ่าน
[ ] LVS ผ่าน
[ ] Antenna Check ผ่าน
[ ] บันทึก Configuration และผลการทดลองครบ
```

---

# 2.27 สรุป

Exercise 2 แสดงให้เห็นว่า RTL ที่เหมือนกันสามารถให้ Physical Layout ที่แตกต่างกันอย่างมากเมื่อเปลี่ยน Configuration Variables

ตัวแปรสำคัญที่ศึกษาในบทนี้คือ

```yaml
FP_SIZING
DIE_AREA
PL_TARGET_DENSITY_PCT
FP_PIN_ORDER_CFG
FP_DEF_TEMPLATE
FP_OBSTRUCTIONS
PL_SOFT_OBSTRUCTIONS
SYNTH_STRATEGY
VERILOG_DEFINES
GRT_ALLOW_CONGESTION
GRT_ANTENNA_ITERS
RSZ_DONT_TOUCH_RX
```

ความสำเร็จของ Physical Design ไม่ได้ขึ้นกับพื้นที่หรือ Timing เพียงค่าเดียว แต่ต้องพิจารณาร่วมกันทั้ง

```text
Area
Timing
Power
Placement Density
Wirelength
Congestion
Routability
DRC
LVS
Antenna
Manufacturing Constraints
```

หัวใจสำคัญของการทำ Floorplan และ Placement Optimization คือการทดลองอย่างเป็นระบบ เก็บ Baseline เปลี่ยนตัวแปรครั้งละน้อย บันทึก Metrics และตัดสินจากข้อมูล แทนการปรับค่าด้วยการคาดเดา


