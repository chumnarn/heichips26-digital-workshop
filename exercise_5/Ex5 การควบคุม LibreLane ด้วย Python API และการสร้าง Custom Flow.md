
# Lab 5: การควบคุม LibreLane ด้วย Python API และการสร้าง Custom Flow

## 5.1 วัตถุประสงค์ของบทปฏิบัติการ

บทปฏิบัติการนี้แนะนำการควบคุมกระบวนการออกแบบวงจรรวมแบบ RTL-to-GDSII ผ่าน **LibreLane Python API** แทนการเรียก flow ผ่าน command-line interface เพียงอย่างเดียว

หลังจากทำบทปฏิบัติการนี้เสร็จ ผู้เรียนจะสามารถ

1. อธิบายองค์ประกอบพื้นฐานของ LibreLane API ได้แก่ Configuration, Flow, Step และ State
2. เขียน Python script เพื่อเริ่มต้น `Classic` flow
3. กำหนดค่าออกแบบ เช่น RTL source, clock constraint, die area และ placement density ผ่าน Python dictionary
4. สร้าง Flow class ใหม่โดยสืบทอดจาก `Classic`
5. ลงทะเบียน Flow ใหม่กับ LibreLane Flow Factory
6. สร้าง Custom Step โดยสืบทอดจากคลาส `Step`
7. เพิ่ม configuration variable ของตนเองให้กับ Custom Step
8. แทรก Custom Step เข้าไปในลำดับ RTL-to-GDSII flow
9. ตรวจสอบ run directory, log, state และผลลัพธ์ที่เกิดจากแต่ละ step
10. อธิบายแนวทางต่อยอด Python API สำหรับ flow automation และ design-space exploration

---

## 5.2 แนวคิดพื้นฐานของ LibreLane API

LibreLane ไม่ได้เป็นเพียงโปรแกรม command-line สำหรับรัน RTL-to-GDSII flow แต่ยังเปิดให้ผู้ใช้งานเข้าถึงองค์ประกอบภายในผ่าน Python API ได้ด้วย

โมดูลสำคัญของ API ประกอบด้วย

| โมดูล | หน้าที่ |
|---|---|
| `librelane.config` | จัดการ configuration variables |
| `librelane.flows` | นิยามและควบคุม flow |
| `librelane.steps` | นิยามขั้นตอนย่อยของ flow |
| `librelane.state` | จัดเก็บ design views และ metrics ระหว่างแต่ละ step |
| `librelane.logging` | แสดงข้อความและสถานะการทำงาน |

LibreLane ระบุว่าองค์ประกอบที่อยู่ในเอกสาร API เป็น programming interface หลักสำหรับโครงสร้างพื้นฐานของ LibreLane โดยแบ่งออกเป็นโมดูล Configuration, Flow, Logging, State และ Step อย่างชัดเจน 

### 5.2.1 Configuration

Configuration คือข้อมูลที่กำหนดว่า flow จะประมวลผลวงจรอย่างไร เช่น

- ชื่อ top-level module
- รายชื่อไฟล์ RTL
- ชื่อ clock port
- clock period
- ขนาด die
- placement density
- routing layers
- timing constraints
- PDK ที่ต้องการใช้

ใน Lab นี้ configuration ถูกสร้างเป็น Python dictionary ชื่อ `flow_cfg`

### 5.2.2 Flow

Flow คือผู้ควบคุมลำดับการทำงานของ Step หลายขั้นตอน เช่น

```text
RTL
 │
 ▼
Synthesis
 │
 ▼
Floorplan
 │
 ▼
Placement
 │
 ▼
Clock Tree Synthesis
 │
 ▼
Routing
 │
 ▼
Physical Verification
 │
 ▼
GDSII
```

`Classic` เป็น flow มาตรฐานของ LibreLane ซึ่งประกอบด้วยขั้นตอน RTL-to-GDSII ที่จัดเรียงไว้แล้ว

### 5.2.3 Step

Step คือหน่วยงานย่อยหนึ่งหน่วยใน flow ตัวอย่างเช่น

- Yosys synthesis
- Floorplanning
- IO placement
- Global placement
- Detailed placement
- Clock tree synthesis
- Global routing
- Detailed routing
- DRC
- LVS

แต่ละ Step รับ `State` จากขั้นตอนก่อนหน้า ประมวลผลข้อมูล แล้วส่ง `State` ใหม่ออกไปยังขั้นตอนถัดไป

### 5.2.4 State

State เป็นโครงสร้างข้อมูลที่ส่งผ่านระหว่าง Step โดยทั่วไปประกอบด้วยสองส่วนสำคัญ

1. **Design Views**

   ตัวอย่างเช่น

   - RTL
   - synthesized netlist
   - OpenDB database
   - DEF
   - LEF
   - GDSII
   - SDF
   - SPEF

2. **Metrics**

   ตัวอย่างเช่น

   - cell area
   - instance count
   - wire length
   - worst slack
   - total negative slack
   - DRC violation count
   - utilization

แนวคิดการส่ง State สามารถเขียนเป็นภาพได้ดังนี้

```text
Initial State
     │
     ▼
┌───────────────┐
│ Step 1        │
│ Synthesis     │
└───────┬───────┘
        │ State 1
        ▼
┌───────────────┐
│ Step 2        │
│ Floorplan     │
└───────┬───────┘
        │ State 2
        ▼
┌───────────────┐
│ Step 3        │
│ Placement     │
└───────┬───────┘
        │ State 3
        ▼
      ...
```

---

## 5.3 โครงสร้างไฟล์ของบทปฏิบัติการ

ภายใน `exercise_5` มีไฟล์หลักดังนี้

```text
exercise_5/
├── README.md
├── counter.sv
└── flow.py
```

หน้าที่ของแต่ละไฟล์คือ

| ไฟล์ | หน้าที่ |
|---|---|
| `README.md` | คำอธิบายและโจทย์ของบทปฏิบัติการ |
| `counter.sv` | RTL ของวงจรตัวนับ 8 บิต |
| `flow.py` | Python script สำหรับกำหนดค่าและเริ่ม LibreLane flow |

Repository ต้นฉบับของ Lab นี้มีไฟล์ดังกล่าวสามไฟล์ และระบุให้เริ่ม flow ด้วยคำสั่ง `python3 flow.py` 

---

## 5.4 วงจรที่ใช้ในบทปฏิบัติการ

ไฟล์ `counter.sv` เป็นวงจรตัวนับแบบ synchronous ขนาด 8 บิต

```systemverilog
// A simple 8-bit counter
module counter (
    input  logic       clk_i,
    input  logic       rst_ni,
    output logic [7:0] count_o
);

    always_ff @(posedge clk_i) begin
        if (!rst_ni) begin
            count_o <= '0;
        end else begin
            count_o <= count_o + 1;
        end
    end

endmodule
```

RTL ใน repository ใช้ `always_ff` ทำงานที่ขอบขาขึ้นของ `clk_i` และตรวจสอบ reset แบบ active-low ภายใน sequential block เมื่อ `rst_ni` มีค่าเป็นศูนย์ เอาต์พุต `count_o` จะถูกล้างเป็นศูนย์ มิฉะนั้นค่าจะเพิ่มขึ้นหนึ่งทุก clock cycle 

### 5.4.1 รายละเอียดพอร์ต

| พอร์ต | ทิศทาง | ขนาด | ความหมาย |
|---|---|---:|---|
| `clk_i` | Input | 1 บิต | สัญญาณนาฬิกา |
| `rst_ni` | Input | 1 บิต | Reset แบบ active-low |
| `count_o` | Output | 8 บิต | ค่าปัจจุบันของตัวนับ |

### 5.4.2 พฤติกรรมของวงจร

เมื่อ `rst_ni = 0`

```text
count_o ← 0
```

เมื่อ `rst_ni = 1`

```text
count_o ← count_o + 1
```

หากตัวนับมีค่า `8'hFF` แล้วได้รับ clock อีกหนึ่งรอบ ค่าจะ overflow กลับเป็น `8'h00`

### 5.4.3 หมายเหตุเกี่ยวกับ Reset

แม้ชื่อพอร์ต `rst_ni` ลงท้ายด้วย `ni` ซึ่งมักใช้แทน active-low input แต่ RTL นี้ไม่ได้ใช้ reset ใน sensitivity list ดังนี้

```systemverilog
always_ff @(posedge clk_i or negedge rst_ni)
```

ดังนั้นวงจรนี้เป็น **synchronous active-low reset** ไม่ใช่ asynchronous reset

---

## 5.5 การเตรียมสภาพแวดล้อม

### ขั้นตอนที่ 1 เข้า LibreLane development environment

เปลี่ยนไปยัง directory หลักของ workshop

```bash
cd ~/heichips26-digital-workshop
```

เข้าสู่สภาพแวดล้อมที่ติดตั้ง LibreLane แล้ว ตัวอย่างสำหรับ Nix environment คือ

```bash
nix-shell
```

หรือในบาง installation อาจใช้

```bash
nix develop
```

ตรวจสอบว่า LibreLane พร้อมใช้งาน

```bash
librelane --version
```

ตรวจสอบ Python

```bash
python3 --version
```

ตรวจสอบว่า Python สามารถ import LibreLane ได้

```bash
python3 -c "import librelane; print('LibreLane Python API is available')"
```

ผลที่ควรได้คือ

```text
LibreLane Python API is available
```

### ขั้นตอนที่ 2 ตรวจสอบ Ciel

```bash
ciel --version
```

แสดงรายการ PDK ที่มีอยู่

```bash
ciel list
```

หรือ

```bash
ciel list --pdk-family ihp-sg13g2
```

### ขั้นตอนที่ 3 Enable IHP SG13G2 PDK

เอกสารของ exercise ระบุให้ enable PDK revision ต่อไปนี้ก่อนรัน Python script

```bash
ciel enable \
  --pdk-family ihp-sg13g2 \
  cb7daaa8901016cf7c5d272dfa322c41f024931f
```

ขั้นตอนนี้จำเป็นเพื่อให้ LibreLane สามารถค้นหาไฟล์เทคโนโลยีและ standard-cell library ของ `ihp-sg13g2` ได้ 

> หมายเหตุ: PDK revision ที่เหมาะสมอาจเปลี่ยนแปลงตาม environment ของ workshop หาก revision ดังกล่าวไม่มีอยู่ ให้ตรวจสอบ revision ที่ติดตั้งด้วย `ciel list`

### ขั้นตอนที่ 4 เข้า directory ของ Lab

```bash
cd exercise_5
```

ตรวจสอบไฟล์

```bash
ls -la
```

ควรเห็นอย่างน้อย

```text
README.md
counter.sv
flow.py
```

---

## 5.6 ตรวจสอบ RTL ก่อนเริ่ม Physical Design

ก่อนเรียก flow ควรตรวจสอบ syntax ของ RTL เพื่อลดเวลาในการ debug

### ขั้นตอนที่ 1 ตรวจสอบด้วย Verilator

```bash
verilator --lint-only --Wall --sv counter.sv
```

หากไม่มี error สำคัญ คำสั่งอาจไม่แสดงข้อความใด หรือแสดงเฉพาะ warning ที่ไม่ทำให้กระบวนการหยุด

### ขั้นตอนที่ 2 ตรวจสอบด้วย Yosys

```bash
yosys -p "read_verilog -sv counter.sv; hierarchy -check -top counter; proc; check"
```

ผลที่ควรสังเกตคือ

```text
Found and reported 0 problems.
```

### ขั้นตอนที่ 3 ตรวจสอบชื่อ Top Module

ชื่อ module ใน RTL คือ

```systemverilog
module counter
```

ดังนั้นค่า configuration ต่อไปนี้ต้องตรงกัน

```python
"DESIGN_NAME": "counter"
```

หากชื่อไม่ตรงกัน Yosys จะไม่สามารถเลือก top-level design ได้

---

## 5.7 วิเคราะห์ไฟล์ `flow.py` ฉบับเริ่มต้น

ไฟล์เริ่มต้นประกอบด้วยส่วนหลักดังนี้

```python
import os

from typing import List, Type, Tuple

from librelane.steps import Step
from librelane.steps.step import ViewsUpdate, MetricsUpdate

from librelane.flows import Flow, FlowError
from librelane.flows.classic import Classic

from librelane.state.state import State
from librelane.config.variable import Variable
from librelane.logging import info
```

### 5.7.1 Import มาตรฐานของ Python

```python
import os
```

ใช้สำหรับจัดการ pathname, environment variables และระบบไฟล์ แม้ใน script เริ่มต้นยังไม่ได้เรียกใช้โดยตรง แต่สามารถนำไปใช้สร้าง absolute path หรืออ่าน environment variable ได้

```python
from typing import List, Type, Tuple
```

ใช้สำหรับ type annotation

### 5.7.2 Import คลาส Step

```python
from librelane.steps import Step
```

นำเข้าคลาสแม่สำหรับสร้าง Custom Step

```python
from librelane.steps.step import ViewsUpdate, MetricsUpdate
```

เป็นชนิดข้อมูลของค่าที่คืนจากเมธอด `run()`

### 5.7.3 Import คลาส Flow

```python
from librelane.flows import Flow, FlowError
```

- `Flow` เป็นคลาสฐานของ flow
- `FlowError` เป็น exception ที่เกี่ยวข้องกับการทำงานของ flow

### 5.7.4 Import Classic Flow

```python
from librelane.flows.classic import Classic
```

ใช้เรียก RTL-to-GDSII flow มาตรฐาน หรือใช้เป็นคลาสแม่ของ Custom Flow

### 5.7.5 Import State

```python
from librelane.state.state import State
```

ใช้ประกาศชนิดของ state ที่ Custom Step รับเข้ามา

### 5.7.6 Import Variable

```python
from librelane.config.variable import Variable
```

ใช้ประกาศ configuration variable ใหม่ของ Custom Step

### 5.7.7 Import Logging Function

```python
from librelane.logging import info
```

ใช้แสดงข้อความในรูปแบบเดียวกับ log ของ LibreLane

---

## 5.8 การกำหนด Source Files

ภายในฟังก์ชัน `main()` มีการกำหนดไฟล์ RTL ดังนี้

```python
verilog_files = [
    "counter.sv"
]
```

รายการนี้ถูกส่งให้ LibreLane ผ่าน configuration variable `VERILOG_FILES`

ข้อดีของการแยกรายการ source file ออกมาต่างหากคือ

- เพิ่ม RTL หลายไฟล์ได้ง่าย
- สร้างรายการไฟล์จาก Python ได้
- ตรวจสอบการมีอยู่ของไฟล์ก่อนเริ่ม flow ได้
- เลือก source file ตามเงื่อนไขได้
- ใช้ glob หรือ recursive file search ได้

ตัวอย่างการเพิ่มหลายไฟล์

```python
verilog_files = [
    "rtl/counter.sv",
    "rtl/counter_control.sv",
    "rtl/counter_top.sv",
]
```

---

## 5.9 การกำหนด Flow Configuration

Configuration เริ่มต้นใน Lab คือ

```python
flow_cfg = {
    # Design
    "DESIGN_NAME": "counter",

    # Sources
    "VERILOG_FILES": verilog_files,

    # Clock
    "CLOCK_PORT": "clk_i",
    "CLOCK_PERIOD": 10,

    # Die area
    "FP_SIZING": "absolute",
    "DIE_AREA": [0, 0, 75, 75],
    "PL_TARGET_DENSITY_PCT": 60,
}
```

ไฟล์ต้นฉบับกำหนดวงจร `counter`, ใช้ `counter.sv`, กำหนด clock period 10 ns, die area 75 × 75 µm และ target placement density 60% 

### 5.9.1 `DESIGN_NAME`

```python
"DESIGN_NAME": "counter"
```

กำหนดชื่อ top-level module

ชื่อนี้ต้องตรงกับ

```systemverilog
module counter
```

### 5.9.2 `VERILOG_FILES`

```python
"VERILOG_FILES": verilog_files
```

กำหนดรายการไฟล์ Verilog หรือ SystemVerilog ที่ใช้เป็น input ของ synthesis

### 5.9.3 `CLOCK_PORT`

```python
"CLOCK_PORT": "clk_i"
```

กำหนดชื่อ top-level clock port

ชื่อต้องตรงกับพอร์ตใน RTL

```systemverilog
input logic clk_i
```

### 5.9.4 `CLOCK_PERIOD`

```python
"CLOCK_PERIOD": 10
```

หน่วยโดยทั่วไปคือ nanosecond

ดังนั้น

$$T_{clk}=10\text{ ns}$$

ความถี่เป้าหมายคือ

$$f_{clk}=\frac{1}{T_{clk}} =\frac{1}{10\text{ ns}}=100\text{ MHz}$$

### 5.9.5 `FP_SIZING`

```python
"FP_SIZING": "absolute"
```

กำหนดให้ใช้ขนาด die แบบระบุพิกัดโดยตรง แทนการให้ LibreLane คำนวณจาก utilization

### 5.9.6 `DIE_AREA`

```python
"DIE_AREA": [0, 0, 75, 75]
```

รูปแบบคือ

```text
[x_min, y_min, x_max, y_max]
```

ดังนั้น

```text
lower-left  = (0, 0)
upper-right = (75, 75)
```

ขนาด die คือ

$$W = 75-0 = 75\ \mu m$$

$$H = 75-0 = 75\ \mu m$$

$$A = 75 \times 75 = 5{,}625\ \mu m^2$$

### 5.9.7 `PL_TARGET_DENSITY_PCT`

```python
"PL_TARGET_DENSITY_PCT": 60
```

กำหนด target density สำหรับ global placement เป็น 60%

ค่าที่สูงเกินไปอาจทำให้

- placement congestion สูง
- legalization ยาก
- routing congestion สูง
- DRC เพิ่มขึ้น
- timing optimization มีพื้นที่ไม่เพียงพอ

ค่าที่ต่ำเกินไปอาจทำให้

- die มีขนาดใหญ่เกินความจำเป็น
- wire length เพิ่มขึ้น
- clock tree ยาวขึ้น
- area efficiency ต่ำ

---

## 5.10 การสร้างและเริ่ม Classic Flow ผ่าน API

โค้ดเริ่มต้นสร้าง object ของ `Classic`

```python
flow = Classic(
    flow_cfg,
    design_dir=".",
    pdk_root=None,
    pdk="ihp-sg13g2",
)
```

### 5.10.1 `flow_cfg`

ส่ง Python dictionary ที่มี configuration variables ทั้งหมด

### 5.10.2 `design_dir`

```python
design_dir="."
```

กำหนดให้ current directory เป็นฐานสำหรับค้นหา source files และสร้างผลลัพธ์

เนื่องจาก `VERILOG_FILES` กำหนดเป็น

```python
"counter.sv"
```

LibreLane จึงค้นหาไฟล์นี้จาก directory ปัจจุบัน

### 5.10.3 `pdk_root`

```python
pdk_root=None
```

ให้ LibreLane ตรวจหา PDK root จาก environment หรือ installation configuration

### 5.10.4 `pdk`

```python
pdk="ihp-sg13g2"
```

เลือก IHP SG13G2 PDK

### 5.10.5 เริ่ม Flow

```python
flow.start()
```

ควรใช้ `start()` แทนการเรียก `run()` โดยตรง เนื่องจาก `start()` ทำ preprocessing และการจัดการที่จำเป็นก่อนเข้าสู่ core flow เอกสาร LibreLane ระบุชัดว่าไม่ควรเรียก `Flow.run()` จากภายนอกคลาส และไม่ควร override `start()` 

### 5.10.6 Python Entry Point

```python
if __name__ == "__main__":
    main()
```

เงื่อนไขนี้ทำให้ฟังก์ชัน `main()` ถูกเรียกเมื่อรันไฟล์โดยตรง

```bash
python3 flow.py
```

แต่จะไม่เรียกโดยอัตโนมัติเมื่อไฟล์ถูก import เป็น module จาก Python script อื่น

---

## 5.11 รัน Classic Flow ครั้งแรก

### ขั้นตอนที่ 1 ตรวจสอบ current directory

```bash
pwd
```

ควรอยู่ใน

```text
.../heichips26-digital-workshop/exercise_5
```

### ขั้นตอนที่ 2 ตรวจสอบ syntax ของ Python

```bash
python3 -m py_compile flow.py
```

หากไม่มี syntax error คำสั่งจะไม่แสดง error

### ขั้นตอนที่ 3 เริ่ม Flow

```bash
python3 flow.py
```

ระหว่างการทำงาน LibreLane จะ

1. โหลด configuration
2. โหลด PDK
3. ตรวจสอบ RTL source
4. สังเคราะห์วงจรด้วย Yosys
5. สร้าง floorplan
6. สร้าง power distribution network
7. วาง I/O pins
8. ทำ global placement
9. ทำ detailed placement
10. สร้าง clock tree
11. ทำ global routing
12. ทำ detailed routing
13. สร้าง signoff views
14. ตรวจสอบ DRC และ LVS ตาม steps ที่เปิดใช้งาน
15. เขียนผลลัพธ์ลง run directory

### ขั้นตอนที่ 4 ตรวจสอบ Run Directory

```bash
ls -lt runs
```

ควรพบ directory ลักษณะดังนี้

```text
RUN_YYYY-MM-DD_HH-MM-SS
```

กำหนดตัวแปรสำหรับ run ล่าสุด

```bash
LATEST_RUN=$(ls -td runs/RUN_* | head -1)
echo "$LATEST_RUN"
```

### ขั้นตอนที่ 5 ตรวจสอบโครงสร้างผลลัพธ์

```bash
find "$LATEST_RUN" -maxdepth 1 -type d | sort
```

แต่ละ Step จะมี directory แยกตามลำดับและ Step ID เช่น

```text
01-yosys-synthesis
02-openroad-checksdcfiles
...
```

ชื่อและจำนวน Step อาจแตกต่างตามเวอร์ชัน LibreLane และ PDK configuration

### ขั้นตอนที่ 6 ตรวจสอบ log

ค้นหา error

```bash
grep -Rni "ERROR" "$LATEST_RUN" | head -30
```

ค้นหา warning

```bash
grep -Rni "WARNING" "$LATEST_RUN" | head -30
```

ตรวจสอบว่า flow เสร็จสมบูรณ์

```bash
grep -Rni "Flow complete" "$LATEST_RUN" | tail
```

---

## 5.12 การสร้าง Custom Flow

เป้าหมายถัดไปคือสร้าง Flow ใหม่ชื่อ `HeiChipsFlow` โดยสืบทอดจาก `Classic`

เพิ่มโค้ดต่อไปนี้ก่อนฟังก์ชัน `main()`

```python
@Flow.factory.register()
class HeiChipsFlow(Classic):
    """
    A flow that inherits the steps from the Classic flow.
    """

    Steps = Classic.Steps
```

Repository กำหนดแนวทางให้ลงทะเบียน `HeiChipsFlow` ด้วย `@Flow.factory.register()` และเริ่มจากการใช้รายการ Step เดียวกับ `Classic.Steps` 

### 5.12.1 Decorator `@Flow.factory.register()`

Decorator นี้ลงทะเบียนคลาสใหม่ไว้ใน Flow Factory ของ LibreLane

ประโยชน์คือ

- LibreLane สามารถค้นหา Flow จากชื่อหรือชนิดได้
- รองรับ plugin architecture
- ทำให้ custom code ใช้รูปแบบเดียวกับ built-in flow

### 5.12.2 การสืบทอดจาก `Classic`

```python
class HeiChipsFlow(Classic):
```

หมายความว่า `HeiChipsFlow` ได้รับพฤติกรรมและโครงสร้างพื้นฐานจาก `Classic`

### 5.12.3 การใช้รายการ Step เดิม

```python
Steps = Classic.Steps
```

Custom Flow ในขั้นนี้ยังมีลำดับ Step เหมือน `Classic` ทุกประการ

ดังนั้นโครงสร้างคือ

```text
Classic
   ▲
   │ inherits
   │
HeiChipsFlow
```

---

## 5.13 เปลี่ยนจาก Classic เป็น HeiChipsFlow

แก้ไขส่วนสร้าง flow object จาก

```python
flow = Classic(
    flow_cfg,
    design_dir=".",
    pdk_root=None,
    pdk="ihp-sg13g2",
)
```

เป็น

```python
flow = HeiChipsFlow(
    flow_cfg,
    design_dir=".",
    pdk_root=None,
    pdk="ihp-sg13g2",
)
```

รันอีกครั้ง

```bash
python3 flow.py
```

เนื่องจาก

```python
Steps = Classic.Steps
```

ผลลัพธ์เชิงฟังก์ชันควรเหมือนกับการใช้ `Classic`

ตรวจสอบ run ล่าสุด

```bash
LATEST_RUN=$(ls -td runs/RUN_* | head -1)
echo "$LATEST_RUN"
```

หลักฐานสำคัญที่ต้องตรวจสอบคือ

- synthesis สำเร็จ
- placement สำเร็จ
- routing สำเร็จ
- signoff steps ทำงาน
- ไม่มี unhandled Python exception
- flow จบด้วยข้อความ `Flow complete`

---

## 5.14 การสร้าง Custom Step

ต่อไปจะสร้าง Step ชื่อ `HelloWorldStep`

เพิ่มโค้ดต่อไปนี้ก่อนคลาส `HeiChipsFlow`

```python
@Step.factory.register()
class HelloWorldStep(Step):
    """
    Prints a configurable Hello World message.
    """

    id = "HeiChips.HelloWorld"
    name = "Hello World!"

    config_vars = [
        Variable(
            "HEICHIPS_SAY",
            str,
            "A string of what to say.",
            default="How are you?",
        ),
    ]

    inputs = []
    outputs = []

    def run(
        self,
        state_in: State,
        **kwargs,
    ) -> Tuple[ViewsUpdate, MetricsUpdate]:

        info(f"Hello World! {self.config['HEICHIPS_SAY']}")

        return {}, {}
```

โครงสร้างนี้สอดคล้องกับโจทย์ใน repository ซึ่งกำหนด Step ID เป็น `HeiChips.HelloWorld`, ชื่อ `Hello World!`, มีตัวแปรชนิด string ชื่อ `HEICHIPS_SAY` และคืนค่า view update กับ metrics update เป็น dictionary ว่าง 

---

## 5.15 วิเคราะห์องค์ประกอบของ Custom Step

### 5.15.1 ลงทะเบียน Step

```python
@Step.factory.register()
```

ลงทะเบียน Custom Step เข้ากับ Step Factory

### 5.15.2 สืบทอดจาก Step

```python
class HelloWorldStep(Step):
```

ทำให้คลาสได้รับ interface และ lifecycle ของ LibreLane Step

### 5.15.3 Step ID

```python
id = "HeiChips.HelloWorld"
```

ID ควรมี namespace เพื่อหลีกเลี่ยงการชนกับ built-in Step หรือ plugin อื่น

รูปแบบที่แนะนำคือ

```text
Organization.StepName
```

ตัวอย่าง

```text
HeiChips.HelloWorld
HeiChips.GenerateReport
MyCompany.PowerCheck
University.ExportMetrics
```

### 5.15.4 Step Name

```python
name = "Hello World!"
```

เป็นชื่อที่ใช้แสดงบน terminal และ progress output

### 5.15.5 Configuration Variable

```python
Variable(
    "HEICHIPS_SAY",
    str,
    "A string of what to say.",
    default="How are you?",
)
```

องค์ประกอบคือ

| ค่า | ความหมาย |
|---|---|
| `"HEICHIPS_SAY"` | ชื่อตัวแปร |
| `str` | ชนิดข้อมูล |
| Description | คำอธิบายตัวแปร |
| `default` | ค่าเริ่มต้นเมื่อผู้ใช้ไม่ได้กำหนด |

### 5.15.6 Input Views

```python
inputs = []
```

Step นี้ไม่ต้องการ design view ใดเป็น input

ใน Step ที่ประมวลผล layout อาจประกาศ input เช่น OpenDB หรือ GDSII

### 5.15.7 Output Views

```python
outputs = []
```

Step นี้ไม่สร้าง design view ใหม่

### 5.15.8 เมธอด `run()`

```python
def run(
    self,
    state_in: State,
    **kwargs,
) -> Tuple[ViewsUpdate, MetricsUpdate]:
```

`state_in` คือ State จาก Step ก่อนหน้า

ค่าที่คืนประกอบด้วย

```python
return views_update, metrics_update
```

สำหรับ HelloWorldStep ไม่มีการแก้ไขทั้งสองส่วน จึงคืนค่า

```python
return {}, {}
```

### 5.15.9 อ่าน Configuration

```python
self.config["HEICHIPS_SAY"]
```

ใช้เข้าถึงค่าของ configuration variable ที่ผ่านการ validate แล้ว

### 5.15.10 แสดงข้อความด้วย LibreLane Logger

```python
info(f"Hello World! {self.config['HEICHIPS_SAY']}")
```

ใช้ `info()` แทน `print()` เพื่อให้ข้อความ

- มี timestamp
- มี log level
- ใช้รูปแบบเดียวกับ LibreLane
- ถูกบันทึกใน step log
- ควบคุม verbosity ได้

---

## 5.16 เพิ่ม Custom Step เข้า HeiChipsFlow

แก้ไขคลาส `HeiChipsFlow` เป็น

```python
@Flow.factory.register()
class HeiChipsFlow(Classic):
    """
    Classic flow followed by a custom Hello World step.
    """

    Steps = Classic.Steps + [HelloWorldStep]
```

การใช้

```python
Classic.Steps + [HelloWorldStep]
```

หมายถึงสร้างรายการใหม่ซึ่งประกอบด้วย Step ทั้งหมดของ Classic แล้วต่อท้ายด้วย `HelloWorldStep`

```text
Classic Step 1
      │
Classic Step 2
      │
     ...
      │
Classic Final Step
      │
HelloWorldStep
```

Repository ระบุว่าเมื่อใช้รูปแบบนี้ Custom Step จะถูกแทรกไว้ท้าย Step ทั้งหมดของ Classic flow 

---

## 5.17 ไฟล์ `flow.py` ฉบับสมบูรณ์

```python
from typing import Tuple

from librelane.config.variable import Variable
from librelane.flows import Flow
from librelane.flows.classic import Classic
from librelane.logging import info
from librelane.state.state import State
from librelane.steps import Step
from librelane.steps.step import MetricsUpdate, ViewsUpdate


@Step.factory.register()
class HelloWorldStep(Step):
    """
    A simple custom step that prints a configurable message.
    """

    id = "HeiChips.HelloWorld"
    name = "Hello World!"

    config_vars = [
        Variable(
            "HEICHIPS_SAY",
            str,
            "A string appended to the Hello World message.",
            default="How are you?",
        ),
    ]

    inputs = []
    outputs = []

    def run(
        self,
        state_in: State,
        **kwargs,
    ) -> Tuple[ViewsUpdate, MetricsUpdate]:
        """
        Execute the custom step.

        This demonstration step does not modify design views or metrics.
        """

        message = self.config["HEICHIPS_SAY"]
        info(f"Hello World! {message}")

        return {}, {}


@Flow.factory.register()
class HeiChipsFlow(Classic):
    """
    Classic RTL-to-GDSII flow followed by HelloWorldStep.
    """

    Steps = Classic.Steps + [HelloWorldStep]


def main() -> None:
    """
    Configure and execute the HeiChips custom LibreLane flow.
    """

    verilog_files = [
        "counter.sv",
    ]

    flow_cfg = {
        # Design
        "DESIGN_NAME": "counter",

        # RTL sources
        "VERILOG_FILES": verilog_files,

        # Clock constraint
        "CLOCK_PORT": "clk_i",
        "CLOCK_PERIOD": 10,

        # Floorplanning
        "FP_SIZING": "absolute",
        "DIE_AREA": [0, 0, 75, 75],

        # Placement
        "PL_TARGET_DENSITY_PCT": 60,
    }

    flow = HeiChipsFlow(
        flow_cfg,
        design_dir=".",
        pdk_root=None,
        pdk="ihp-sg13g2",
    )

    flow.start()


if __name__ == "__main__":
    main()
```

---

## 5.18 รัน Custom Flow ด้วยค่า Default

ตรวจสอบ syntax

```bash
python3 -m py_compile flow.py
```

เริ่ม flow

```bash
python3 flow.py
```

เมื่อ Classic flow ทำงานครบแล้ว ควรพบข้อความลักษณะดังนี้

```text
──────────────────────── Hello World! ────────────────────────
VERBOSE  Running 'HeiChips.HelloWorld' at
         'runs/RUN_.../...-heichips-helloworld'
INFO     Hello World! How are you?
INFO     Saving views to '...'
INFO     Flow complete.
```

เวลา เลขลำดับ Step และ pathname จะแตกต่างกันในแต่ละครั้ง

### ตรวจสอบว่า Custom Step ถูกสร้างจริง

```bash
LATEST_RUN=$(ls -td runs/RUN_* | head -1)

find "$LATEST_RUN" \
  -maxdepth 1 \
  -type d \
  -iname "*helloworld*"
```

ควรพบ directory ลักษณะดังนี้

```text
runs/RUN_.../NN-heichips-helloworld
```

### ค้นหาข้อความจาก Log

```bash
grep -Rni "Hello World" "$LATEST_RUN"
```

ควรพบ

```text
Hello World! How are you?
```

---

## 5.19 ปรับแต่ง Custom Step ผ่าน Configuration

เพิ่มตัวแปรต่อไปนี้ใน `flow_cfg`

```python
"HEICHIPS_SAY": "Howdy!",
```

ตัวอย่าง

```python
flow_cfg = {
    "DESIGN_NAME": "counter",
    "VERILOG_FILES": verilog_files,

    "CLOCK_PORT": "clk_i",
    "CLOCK_PERIOD": 10,

    "FP_SIZING": "absolute",
    "DIE_AREA": [0, 0, 75, 75],
    "PL_TARGET_DENSITY_PCT": 60,

    "HEICHIPS_SAY": "Howdy!",
}
```

รันใหม่

```bash
python3 flow.py
```

ควรเห็น

```text
INFO Hello World! Howdy!
```

แสดงว่า LibreLane

1. มองเห็น configuration variable ของ Custom Step
2. ตรวจสอบชนิดข้อมูลเป็น `str`
3. ส่งค่าไปยัง `self.config`
4. ทำให้ Step เปลี่ยนพฤติกรรมได้โดยไม่ต้องแก้ logic ภายใน `run()`

---

## 5.20 ตรวจสอบการ Validate ชนิดข้อมูล

ทดลองกำหนดค่าผิดชนิด

```python
"HEICHIPS_SAY": 1234,
```

จากนั้นรัน

```bash
python3 flow.py
```

สิ่งที่ต้องสังเกตคือ LibreLane อาจแปลงค่าได้หรือรายงาน configuration validation error ขึ้นอยู่กับกฎการ coercion ของ LibreLane version ที่ใช้อยู่

แนวปฏิบัติที่ถูกต้องคือกำหนดค่าให้ตรงชนิด

```python
"HEICHIPS_SAY": "1234",
```

การประกาศชนิดข้อมูลช่วยลดข้อผิดพลาดที่ตรวจพบยากใน flow ขนาดใหญ่

---

## 5.21 การรับ Final State จาก `flow.start()`

เอกสาร LibreLane ระบุว่า `Flow.start()` คืน tuple ซึ่งประกอบด้วย final state และรายการ Step objects ที่สร้างระหว่างการรัน 

สามารถแก้ไขจาก

```python
flow.start()
```

เป็น

```python
final_state, step_objects = flow.start()
```

จากนั้นเพิ่ม

```python
info(f"Total executed steps: {len(step_objects)}")
```

ตัวอย่าง

```python
final_state, step_objects = flow.start()

info(f"Total executed steps: {len(step_objects)}")
info(f"Final metric count: {len(final_state.metrics)}")
```

ข้อดีของการรับค่าเหล่านี้คือ

- วิเคราะห์ metrics หลัง flow จบ
- สร้างสรุปผลอัตโนมัติ
- เปรียบเทียบหลาย runs
- ส่งข้อมูลเข้าสู่ฐานข้อมูล
- ตรวจสอบ Step ที่ถูกเรียกจริง
- สร้าง CI pass/fail criteria

---

## 5.22 เพิ่ม Metric จาก Custom Step

`HelloWorldStep` ปัจจุบันคืน metrics update ว่าง

```python
return {}, {}
```

สามารถทดลองเพิ่ม metric ได้ดังนี้

```python
def run(
    self,
    state_in: State,
    **kwargs,
) -> Tuple[ViewsUpdate, MetricsUpdate]:

    message = self.config["HEICHIPS_SAY"]
    info(f"Hello World! {message}")

    metrics = {
        "heichips__message__length": len(message),
    }

    return {}, metrics
```

ตัวอย่างเช่น เมื่อ

```python
"HEICHIPS_SAY": "Howdy!"
```

จะได้

```text
heichips__message__length = 6
```

แนวคิดนี้สามารถต่อยอดเป็น Custom Step สำหรับ

- นับจำนวน violation
- คำนวณ utilization
- ตรวจสอบไฟล์ output
- สร้าง quality score
- ตรวจสอบ timing threshold
- ตรวจสอบพื้นที่วงจร
- สรุป signoff status

---

## 5.23 เพิ่มการตรวจสอบ Input File

ปรับปรุง `main()` ให้ตรวจสอบไฟล์ก่อนเริ่ม flow

```python
from pathlib import Path
```

จากนั้นเพิ่ม

```python
for source_file in verilog_files:
    source_path = Path(source_file)

    if not source_path.is_file():
        raise FileNotFoundError(
            f"RTL source file does not exist: {source_path}"
        )
```

ตัวอย่างฉบับรวม

```python
def main() -> None:
    verilog_files = [
        "counter.sv",
    ]

    for source_file in verilog_files:
        source_path = Path(source_file)

        if not source_path.is_file():
            raise FileNotFoundError(
                f"RTL source file does not exist: {source_path}"
            )

    flow_cfg = {
        # ...
    }
```

ข้อดีคือรายงานปัญหาได้ก่อนเสียเวลาโหลด PDK และเตรียม flow

---

## 5.24 การกำหนด Path ที่ไม่ขึ้นกับ Current Directory

Script เริ่มต้นใช้

```python
design_dir="."
```

วิธีนี้ต้องรันจาก `exercise_5` เท่านั้น

ถ้ารันจาก directory อื่น

```bash
python3 exercise_5/flow.py
```

Python พบ script แต่ `counter.sv` อาจไม่ถูกค้นพบ เพราะ current directory ไม่ใช่ `exercise_5`

วิธีที่แข็งแรงกว่าคือใช้ pathname ของตัว script

```python
from pathlib import Path

DESIGN_DIR = Path(__file__).resolve().parent
```

กำหนด source file

```python
verilog_files = [
    DESIGN_DIR / "counter.sv",
]
```

และสร้าง flow

```python
flow = HeiChipsFlow(
    flow_cfg,
    design_dir=DESIGN_DIR,
    pdk_root=None,
    pdk="ihp-sg13g2",
)
```

ทำให้เรียกจาก directory ใดก็ได้

```bash
python3 exercise_5/flow.py
```

---

## 5.25 การจัดการ Exception

สามารถครอบ `flow.start()` ด้วย `try-except`

```python
try:
    final_state, step_objects = flow.start()
except FlowError as error:
    info(f"LibreLane flow failed: {error}")
    raise
```

ตัวอย่างที่เหมาะสมยิ่งขึ้น

```python
import sys

try:
    final_state, step_objects = flow.start()
except FlowError as error:
    print(f"ERROR: LibreLane flow failed: {error}", file=sys.stderr)
    raise SystemExit(1) from error
```

อย่างไรก็ตาม ไม่ควรดัก exception แล้วปล่อยให้โปรแกรมจบด้วย exit code 0 เพราะระบบ CI อาจเข้าใจผิดว่า flow สำเร็จ

---

## 5.26 Flow ฉบับปรับปรุงสำหรับ Automation

```python
from pathlib import Path
from typing import Tuple

from librelane.config.variable import Variable
from librelane.flows import Flow, FlowError
from librelane.flows.classic import Classic
from librelane.logging import info
from librelane.state.state import State
from librelane.steps import Step
from librelane.steps.step import MetricsUpdate, ViewsUpdate


DESIGN_DIR = Path(__file__).resolve().parent


@Step.factory.register()
class HelloWorldStep(Step):
    """
    Print a message and record its length as a custom metric.
    """

    id = "HeiChips.HelloWorld"
    name = "Hello World!"

    config_vars = [
        Variable(
            "HEICHIPS_SAY",
            str,
            "Text appended to the Hello World message.",
            default="How are you?",
        ),
    ]

    inputs = []
    outputs = []

    def run(
        self,
        state_in: State,
        **kwargs,
    ) -> Tuple[ViewsUpdate, MetricsUpdate]:

        message = self.config["HEICHIPS_SAY"]

        info(f"Hello World! {message}")

        metrics = {
            "heichips__message__length": len(message),
        }

        return {}, metrics


@Flow.factory.register()
class HeiChipsFlow(Classic):
    """
    Classic flow with an additional post-processing step.
    """

    Steps = Classic.Steps + [HelloWorldStep]


def validate_sources(source_files: list[Path]) -> None:
    """
    Verify that all RTL source files exist.
    """

    for source_file in source_files:
        if not source_file.is_file():
            raise FileNotFoundError(
                f"RTL source file does not exist: {source_file}"
            )


def main() -> None:
    """
    Run the custom LibreLane flow.
    """

    verilog_files = [
        DESIGN_DIR / "counter.sv",
    ]

    validate_sources(verilog_files)

    flow_cfg = {
        "DESIGN_NAME": "counter",
        "VERILOG_FILES": verilog_files,

        "CLOCK_PORT": "clk_i",
        "CLOCK_PERIOD": 10,

        "FP_SIZING": "absolute",
        "DIE_AREA": [0, 0, 75, 75],
        "PL_TARGET_DENSITY_PCT": 60,

        "HEICHIPS_SAY": "LibreLane API flow completed.",
    }

    flow = HeiChipsFlow(
        flow_cfg,
        design_dir=DESIGN_DIR,
        pdk_root=None,
        pdk="ihp-sg13g2",
    )

    try:
        final_state, step_objects = flow.start()
    except FlowError as error:
        raise SystemExit(
            f"LibreLane flow failed: {error}"
        ) from error

    info(f"Executed steps: {len(step_objects)}")
    info(f"Final metrics: {len(final_state.metrics)}")


if __name__ == "__main__":
    main()
```

---

## 5.27 ตรวจสอบ Physical Design Results

หลัง flow สำเร็จ กำหนด run ล่าสุด

```bash
LATEST_RUN=$(ls -td runs/RUN_* | head -1)
echo "$LATEST_RUN"
```

### 5.27.1 ค้นหา GDSII

```bash
find "$LATEST_RUN" -type f \( -name "*.gds" -o -name "*.gdsii" \)
```

### 5.27.2 ค้นหา DEF

```bash
find "$LATEST_RUN" -type f -name "*.def"
```

### 5.27.3 ค้นหา Netlist

```bash
find "$LATEST_RUN" -type f \
  \( -name "*.v" -o -name "*.nl.v" \)
```

### 5.27.4 ค้นหา SDF

```bash
find "$LATEST_RUN" -type f -name "*.sdf"
```

### 5.27.5 ค้นหา SPEF

```bash
find "$LATEST_RUN" -type f -name "*.spef"
```

### 5.27.6 ค้นหา Metrics

```bash
find "$LATEST_RUN" -type f \
  \( -name "*metric*" -o -name "resolved.json" \)
```

ชื่อและตำแหน่งไฟล์อาจแตกต่างกันตาม LibreLane version

---

## 5.28 เปิด Layout เพื่อตรวจสอบ

ค้นหา GDSII ล่าสุด

```bash
GDS_FILE=$(find "$LATEST_RUN" -type f -name "*.gds" | tail -1)
echo "$GDS_FILE"
```

เปิดด้วย KLayout

```bash
klayout "$GDS_FILE"
```

สิ่งที่ควรตรวจสอบ

- die boundary
- core boundary
- standard-cell rows
- placement ของ flip-flops และ logic cells
- power stripes หรือ power rails
- clock routing
- signal routing
- I/O pins
- fill cells
- metal density structures หากมี

สำหรับวงจร counter ขนาดเล็ก พื้นที่ว่างภายใน die อาจมีมาก เนื่องจากกำหนด die area แบบ absolute ที่ 75 × 75 µm

---

## 5.29 การทดลองเปลี่ยน Clock Period

### การทดลองที่ 1: 100 MHz

```python
"CLOCK_PERIOD": 10
```

$$f=100\text{ MHz}$$

### การทดลองที่ 2: 200 MHz

```python
"CLOCK_PERIOD": 5
```

$$f=200\text{ MHz}$$

### การทดลองที่ 3: 50 MHz

```python
"CLOCK_PERIOD": 20
```

$$f=50\text{ MHz}$$

สำหรับแต่ละค่าให้รัน flow และเปรียบเทียบ

- worst slack
- total negative slack
- cell count
- buffer count
- clock tree size
- routing length
- runtime

ตารางบันทึกผล

| Clock Period | Frequency | WNS | TNS | Cell Area | Result |
|---:|---:|---:|---:|---:|---|
| 20 ns | 50 MHz | | | | |
| 10 ns | 100 MHz | | | | |
| 5 ns | 200 MHz | | | | |

---

## 5.30 การทดลองเปลี่ยน Die Area

ทดลองค่าต่อไปนี้

```python
"DIE_AREA": [0, 0, 50, 50]
```

```python
"DIE_AREA": [0, 0, 75, 75]
```

```python
"DIE_AREA": [0, 0, 100, 100]
```

บันทึกผล

| Die Size | Area | Placement | Routing | DRC | หมายเหตุ |
|---|---:|---|---|---:|---|
| 50 × 50 µm | 2,500 µm² | | | | |
| 75 × 75 µm | 5,625 µm² | | | | |
| 100 × 100 µm | 10,000 µm² | | | | |

อภิปรายว่า die ที่เล็กลงอาจเพิ่ม congestion แต่สำหรับวงจรขนาดเล็กมาก ความแตกต่างอาจไม่ชัดเจนจนกว่าจะลดพื้นที่ต่ำกว่าข้อจำกัดของ standard-cell rows, PDN หรือ pin placement

---

## 5.31 การทดลองเปลี่ยน Placement Density

ทดลอง

```python
"PL_TARGET_DENSITY_PCT": 40
```

```python
"PL_TARGET_DENSITY_PCT": 60
```

```python
"PL_TARGET_DENSITY_PCT": 80
```

เปรียบเทียบ

| Density | Global Placement | Detailed Placement | Routing | Runtime |
|---:|---|---|---|---:|
| 40% | | | | |
| 60% | | | | |
| 80% | | | | |

ข้อสังเกตสำคัญคือ `PL_TARGET_DENSITY_PCT` เป็นเป้าหมายของ placement ไม่ได้หมายความว่า standard cells จะใช้พื้นที่ die เท่ากับค่านั้นเสมอ โดยเฉพาะเมื่อใช้ fixed die area ที่ใหญ่กว่าพื้นที่ logic มาก

---

## 5.32 การแทรก Custom Step ในตำแหน่งอื่น

ตัวอย่างปัจจุบันเพิ่ม Step ต่อท้าย

```python
Steps = Classic.Steps + [HelloWorldStep]
```

ในงานจริงอาจต้องการแทรก Step ก่อนหรือหลังขั้นตอนเฉพาะ เช่น

- หลัง synthesis เพื่อวิเคราะห์ netlist
- หลัง placement เพื่อคำนวณ congestion
- หลัง routing เพื่อตรวจสอบ routed database
- หลัง signoff เพื่อสร้าง report

แนวทางพื้นฐานคือสร้างรายการ Step ใหม่จาก `Classic.Steps`

ตัวอย่างเชิงแนวคิด

```python
custom_steps = []

for step in Classic.Steps:
    custom_steps.append(step)

    if step.id == "Yosys.Synthesis":
        custom_steps.append(HelloWorldStep)

Steps = custom_steps
```

เนื่องจากชื่อ Step ID อาจเปลี่ยนหรือมีหลาย variant ควรตรวจสอบรายการ `Classic.Steps` ของ LibreLane version ที่ใช้งานจริงก่อน

---

## 5.33 การประยุกต์ใช้ LibreLane API

Python API เหมาะสำหรับงานที่ซับซ้อนกว่าการรัน configuration เดียว เช่น

### 5.33.1 Design-Space Exploration

ทดลองหลายค่าโดยอัตโนมัติ

```text
CLOCK_PERIOD = 5, 7.5, 10 ns
DIE_AREA     = 50×50, 75×75, 100×100 µm
DENSITY      = 40%, 50%, 60%, 70%
```

แล้วเลือกผลที่ดีที่สุดจาก

- timing
- area
- power
- congestion
- DRC
- runtime

### 5.33.2 Parallel Flows

เริ่มหลาย flow พร้อมกันเพื่อทดลอง configuration หลายชุด

ต้องคำนึงถึง

- CPU cores
- memory
- disk space
- tool license หากมี
- run directory collision

LibreLane มีแนวทางสำหรับ asynchronous step execution แต่เอกสารเตือนว่า Flow object ไม่เป็น thread-safe และควรใช้กลไกที่ LibreLane เตรียมไว้เมื่อรัน Step แบบขนาน 

### 5.33.3 Automatic Result Collection

หลัง flow จบ Python สามารถ

- อ่าน final metrics
- เขียน CSV
- เขียน JSON
- สร้างกราฟ
- อัปโหลดผลเข้าสู่ฐานข้อมูล
- เปรียบเทียบกับ baseline
- สร้าง regression dashboard

### 5.33.4 CI/CD Integration

ใช้ exit code และ metrics เป็นเงื่อนไข เช่น

```text
Fail เมื่อ DRC > 0
Fail เมื่อ LVS ไม่ผ่าน
Fail เมื่อ WNS < -0.10 ns
Fail เมื่อ area เพิ่มมากกว่า 5%
```

### 5.33.5 Custom Signoff Policy

สร้าง Custom Step เพื่อรวมสถานะ

```text
Synthesis      PASS
Placement      PASS
Routing        PASS
STA            PASS
DRC            PASS
LVS            PASS
Overall        PASS
```

---

## 5.34 ปัญหาที่พบบ่อยและแนวทางแก้ไข

### ปัญหา 1: `ModuleNotFoundError: No module named 'librelane'`

สาเหตุ

- ยังไม่ได้เข้า Nix shell
- Python ที่เรียกไม่ใช่ Python ใน LibreLane environment

ตรวจสอบ

```bash
which python3
python3 -c "import librelane; print(librelane)"
```

แก้ไขโดยเข้าสู่ environment ที่ติดตั้ง LibreLane ก่อน

---

### ปัญหา 2: ไม่พบ PDK

ตัวอย่างข้อความ

```text
PDK ihp-sg13g2 was not found
```

ตรวจสอบ

```bash
ciel list --pdk-family ihp-sg13g2
```

Enable revision ที่ workshop กำหนด

```bash
ciel enable \
  --pdk-family ihp-sg13g2 \
  cb7daaa8901016cf7c5d272dfa322c41f024931f
```

---

### ปัญหา 3: ไม่พบ `counter.sv`

สาเหตุ

- รัน script จาก directory ผิด
- `design_dir` ไม่ตรงกับตำแหน่ง RTL
- filename ผิด

ตรวจสอบ

```bash
pwd
ls -l counter.sv
```

หรือเปลี่ยนไปใช้ absolute path จาก `Path(__file__)`

---

### ปัญหา 4: ไม่พบ Top Module

ตัวอย่าง

```text
Module counter not found
```

ตรวจสอบว่า RTL มี

```systemverilog
module counter
```

และ configuration มี

```python
"DESIGN_NAME": "counter"
```

ตัวพิมพ์เล็กและตัวพิมพ์ใหญ่ต้องตรงกัน

---

### ปัญหา 5: ไม่พบ Clock Port

ตรวจสอบ RTL

```systemverilog
input logic clk_i
```

และ configuration

```python
"CLOCK_PORT": "clk_i"
```

หากกำหนดเป็น `clk` แต่ RTL ใช้ `clk_i` จะเกิด constraint error หรือ clock ไม่ถูกสร้าง

---

### ปัญหา 6: Custom Step ไม่ทำงาน

ตรวจสอบสามจุด

1. มี decorator

```python
@Step.factory.register()
```

2. มี Custom Step ในรายการ Flow

```python
Steps = Classic.Steps + [HelloWorldStep]
```

3. สร้าง object จาก `HeiChipsFlow`

```python
flow = HeiChipsFlow(...)
```

ไม่ใช่

```python
flow = Classic(...)
```

---

### ปัญหา 7: Configuration Variable ไม่เป็นที่รู้จัก

ตรวจสอบว่า `HEICHIPS_SAY` ถูกประกาศใน `config_vars`

```python
config_vars = [
    Variable(
        "HEICHIPS_SAY",
        str,
        "...",
        default="How are you?",
    ),
]
```

และ Custom Step อยู่ใน `Steps` ของ Flow เพื่อให้ LibreLane รวบรวมและ validate configuration variables ของ Step นั้น

---

### ปัญหา 8: Python Indentation Error

ใช้ช่องว่างสี่ช่องอย่างสม่ำเสมอ และหลีกเลี่ยงการผสม tab กับ space

ตรวจสอบ

```bash
python3 -m py_compile flow.py
```

---

### ปัญหา 9: Die Area เล็กเกินไป

อาการที่อาจพบ

- floorplan failure
- PDN generation failure
- placement legalization failure
- high congestion
- routing failure

เพิ่มพื้นที่ เช่น

```python
"DIE_AREA": [0, 0, 100, 100]
```

---

### ปัญหา 10: Flow ใช้ผลจาก Run เก่า

แต่ละ invocation ควรสร้าง run directory ใหม่ ตรวจสอบ run ล่าสุดด้วย

```bash
ls -td runs/RUN_* | head
```

อย่าสรุปผลจาก directory เก่าโดยดูเฉพาะชื่อไฟล์โดยไม่ตรวจ timestamp

---

## 5.35 คำถามท้ายบทปฏิบัติการ

1. LibreLane Flow แตกต่างจาก LibreLane Step อย่างไร
2. State มีหน้าที่อะไรในกระบวนการส่งข้อมูลระหว่าง Step
3. เพราะเหตุใดจึงควรเรียก `flow.start()` แทน `flow.run()`
4. `@Flow.factory.register()` มีหน้าที่อะไร
5. `@Step.factory.register()` มีหน้าที่อะไร
6. เพราะเหตุใด Step ID ควรมี namespace เช่น `HeiChips.HelloWorld`
7. `inputs = []` และ `outputs = []` หมายความว่าอย่างไร
8. เหตุใด `HelloWorldStep` จึงคืนค่า `({}, {})`
9. หากต้องการสร้าง metric ใหม่ควรคืนค่าใน tuple ส่วนใด
10. `CLOCK_PERIOD = 10` สอดคล้องกับความถี่เท่าใด
11. `DIE_AREA = [0, 0, 75, 75]` มีพื้นที่เท่าใด
12. เหตุใด `DESIGN_NAME` ต้องตรงกับชื่อ module ใน RTL
13. การใช้ absolute path มีข้อดีกว่า relative path อย่างไร
14. Custom Step สามารถนำไปใช้กับงาน signoff automation ได้อย่างไร
15. Python API เหมาะกับ design-space exploration อย่างไร

---

## 5.36 แบบฝึกหัดเพิ่มเติม

### แบบฝึกหัด 5.1 เปลี่ยนข้อความ

กำหนด

```python
"HEICHIPS_SAY": "Welcome to HeiChips 2026!"
```

รัน flow และแนบ log ที่แสดงข้อความใหม่

### แบบฝึกหัด 5.2 เพิ่มตัวแปรชื่อผู้เรียน

เพิ่ม configuration variable

```python
Variable(
    "STUDENT_NAME",
    str,
    "Name of the student.",
    default="Unknown",
)
```

ให้ Custom Step แสดง

```text
Hello <STUDENT_NAME>! <HEICHIPS_SAY>
```

### แบบฝึกหัด 5.3 เพิ่ม Boolean Variable

เพิ่มตัวแปร

```python
Variable(
    "HEICHIPS_VERBOSE",
    bool,
    "Enable detailed custom messages.",
    default=False,
)
```

แสดงรายละเอียดเพิ่มเติมเฉพาะเมื่อค่าตัวแปรเป็น `True`

### แบบฝึกหัด 5.4 เพิ่ม Metric

สร้าง metric

```text
heichips__custom_step__executed = 1
```

และตรวจสอบว่า metric ถูกเพิ่มใน final state

### แบบฝึกหัด 5.5 สร้าง Source Checker Step

สร้าง `RTLSourceCheckStep` เพื่อตรวจสอบว่าไฟล์ทุกไฟล์ใน `VERILOG_FILES` มีอยู่จริง

### แบบฝึกหัด 5.6 สร้าง Post-Flow Summary

หลัง `flow.start()` ให้แสดง

- จำนวน Step
- จำนวน metrics
- ชื่อ design
- clock period
- PDK
- run status

### แบบฝึกหัด 5.7 เปรียบเทียบ Clock Constraint

รันที่ 50, 100 และ 200 MHz แล้วสรุป timing กับ area

### แบบฝึกหัด 5.8 เปรียบเทียบ Placement Density

ทดลอง 40%, 60% และ 80% แล้วบันทึก congestion หรือ routing result

---

## 5.37 สิ่งที่ต้องส่ง

ผู้เรียนต้องส่งไฟล์และหลักฐานดังต่อไปนี้

```text
lab5_submission/
├── counter.sv
├── flow.py
├── terminal_log.txt
├── run_summary.md
├── metrics_summary.csv
└── screenshots/
    ├── custom_step_log.png
    ├── final_layout.png
    └── run_directory.png
```

### เนื้อหาของ `run_summary.md`

ควรประกอบด้วย

1. ชื่อผู้เรียน
2. วันที่ทำ Lab
3. LibreLane version
4. PDK และ revision
5. ค่า configuration ที่ใช้
6. ผลของ Classic flow
7. ผลของ HeiChipsFlow
8. ข้อความจาก HelloWorldStep
9. timing summary
10. DRC/LVS summary
11. ปัญหาที่พบ
12. วิธีแก้ไข
13. สิ่งที่ได้เรียนรู้

---

## 5.38 เกณฑ์การประเมิน

| หัวข้อ | คะแนน |
|---|---:|
| เตรียม environment และ PDK ถูกต้อง | 10 |
| อธิบาย `flow.py` ได้ถูกต้อง | 15 |
| รัน Classic flow สำเร็จ | 15 |
| สร้างและลงทะเบียน HeiChipsFlow | 15 |
| สร้าง HelloWorldStep | 20 |
| ปรับค่า `HEICHIPS_SAY` ได้ | 10 |
| ตรวจสอบ log และผลลัพธ์ | 10 |
| สรุปผลและตอบคำถาม | 5 |
| **รวม** | **100** |

---

## 5.39 สรุป

บทปฏิบัติการนี้แสดงให้เห็นว่า LibreLane สามารถควบคุมผ่าน Python ได้โดยตรง ไม่จำกัดเฉพาะการเรียก command-line flow หรือใช้ไฟล์ configuration เท่านั้น

องค์ประกอบสำคัญที่ผู้เรียนได้ใช้งานคือ

```text
Python Configuration
        │
        ▼
   HeiChipsFlow
        │
        ├── Classic RTL-to-GDSII Steps
        │
        └── HelloWorldStep
                 │
                 ▼
          Configurable Message
```

หลักการสำคัญประกอบด้วย

- Configuration เป็นตัวกำหนดพารามิเตอร์การออกแบบ
- Flow เป็นผู้ควบคุมลำดับ Step
- Step เป็นหน่วยประมวลผลย่อย
- State ส่ง design views และ metrics ระหว่าง Step
- Factory registration ทำให้ LibreLane รู้จัก Custom Flow และ Custom Step
- `Flow.start()` เป็น entry point ที่ถูกต้องสำหรับเริ่ม flow
- Python API ช่วยให้สร้าง automation, regression, custom reporting และ design-space exploration ได้

แม้ `HelloWorldStep` จะเป็นตัวอย่างอย่างง่ายและไม่ได้แก้ไข design state แต่โครงสร้างเดียวกันนี้สามารถพัฒนาเป็น Step สำหรับตรวจสอบ RTL, วิเคราะห์ netlist, ประเมิน timing, ตรวจสอบ physical design, รวบรวม metrics หรือสร้าง signoff report อัตโนมัติได้


