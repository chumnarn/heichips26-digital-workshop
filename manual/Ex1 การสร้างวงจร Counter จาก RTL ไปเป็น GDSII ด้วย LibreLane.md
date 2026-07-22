
# Lab 1: การสร้างวงจร Counter จาก RTL ไปเป็น GDSII ด้วย LibreLane

## 1. วัตถุประสงค์ของบทปฏิบัติการ

บทปฏิบัติการนี้เป็นการทดลองใช้งาน LibreLane ครั้งแรก โดยนำวงจรเคาน์เตอร์ขนาด 8 บิตที่เขียนด้วย SystemVerilog ผ่านกระบวนการออกแบบวงจรรวมดิจิทัลตั้งแต่ RTL จนถึงข้อมูลเลย์เอาต์ระดับ GDSII

เมื่อจบบทปฏิบัติการ ผู้เรียนจะสามารถ:

1. ตรวจสอบโครงสร้างของโครงการ LibreLane
2. อ่านและอธิบาย RTL ของวงจรเคาน์เตอร์ได้
3. อ่านและอธิบายไฟล์ `config.yaml` ได้
4. เรียกใช้ LibreLane กับ IHP SG13G2 PDK
5. ตรวจสอบผลการสังเคราะห์ การวางเซลล์ การสร้าง Clock Tree และการเดินสาย
6. เปิดดู Physical Design ด้วย OpenROAD GUI
7. เปิดดู GDSII ด้วย KLayout
8. ตรวจสอบรายงาน Timing, DRC, LVS และ Antenna
9. เข้าใจโครงสร้างไดเรกทอรี `runs/`
10. วิเคราะห์และแก้ไขปัญหาเบื้องต้นของกระบวนการ RTL-to-GDSII ได้

---

## 2. ภาพรวมของงานทดลอง

วงจรที่ใช้ในบทปฏิบัติการเป็นเคาน์เตอร์ไบนารีขนาด 8 บิต มีอินพุตและเอาต์พุตดังนี้

| สัญญาณ | ทิศทาง | ขนาด | หน้าที่ |
|---|---:|---:|---|
| `clk_i` | Input | 1 บิต | สัญญาณนาฬิกา |
| `rst_ni` | Input | 1 บิต | Reset แบบ Active-Low |
| `count_o` | Output | 8 บิต | ค่าปัจจุบันของเคาน์เตอร์ |

เมื่อ `rst_ni` มีค่าเป็นลอจิก `0` ค่า `count_o` จะถูกล้างเป็นศูนย์ที่ขอบขาขึ้นของสัญญาณนาฬิกา

เมื่อ `rst_ni` มีค่าเป็นลอจิก `1` เคาน์เตอร์จะเพิ่มค่าขึ้นครั้งละหนึ่งในทุกขอบขาขึ้นของ `clk_i`

ลำดับกระบวนการออกแบบโดยสรุปคือ

```text
SystemVerilog RTL
       │
       ▼
RTL Elaboration
       │
       ▼
Logic Synthesis
       │
       ▼
Floorplanning
       │
       ▼
Placement
       │
       ▼
Clock Tree Synthesis
       │
       ▼
Global Routing
       │
       ▼
Detailed Routing
       │
       ▼
Parasitic Extraction และ STA
       │
       ▼
DRC / LVS / Antenna
       │
       ▼
GDSII
```

---

## 3. เครื่องมือที่ใช้

บทปฏิบัติการนี้ใช้เครื่องมือหลักดังต่อไปนี้

- LibreLane สำหรับควบคุมกระบวนการ RTL-to-GDSII
- Yosys สำหรับ Logic Synthesis
- ABC สำหรับ Technology Mapping
- OpenROAD สำหรับ Floorplanning, Placement, CTS, Routing และ STA
- KLayout สำหรับเปิดดู GDSII และข้อมูลเลย์เอาต์
- Magic สำหรับงาน Physical Verification บางขั้นตอน
- Netgen สำหรับ LVS
- IHP Open PDK ตระกูล `ihp-sg13g2`

LibreLane เรียกใช้เครื่องมือเหล่านี้ตามลำดับที่กำหนดไว้ใน flow ผู้ใช้จึงไม่จำเป็นต้องเรียกแต่ละโปรแกรมด้วยตนเองในบทปฏิบัติการแรก

---

## 4. โครงสร้างไฟล์ของ Exercise 1

ภายในไดเรกทอรี `exercise_1` มีไฟล์สำคัญดังนี้

```text
exercise_1/
├── README.md
├── config.yaml
├── counter.sv
└── img/
    ├── openroad_gui_1.png
    ├── openroad_gui_2.png
    ├── openroad_gui_3.png
    └── klayout_1.png
```

หน้าที่ของแต่ละไฟล์คือ

### 4.1 ไฟล์ `counter.sv`

เป็น RTL ของวงจรเคาน์เตอร์ 8 บิต เขียนด้วย SystemVerilog

### 4.2 ไฟล์ `config.yaml`

เป็นไฟล์กำหนดค่าของ LibreLane เช่น ชื่อ top-level module, รายการ RTL, clock port และ clock period

### 4.3 ไฟล์ `README.md`

เป็นคำแนะนำเบื้องต้นสำหรับการรัน Exercise

### 4.4 ไดเรกทอรี `img/`

เก็บภาพตัวอย่างผลลัพธ์จาก OpenROAD GUI และ KLayout

---

# ส่วนที่ 1: เตรียมสภาพแวดล้อม

## 5. ดาวน์โหลด Repository

เปิด Terminal แล้วเลือกตำแหน่งที่ต้องการจัดเก็บโครงการ เช่น

```bash
cd ~
mkdir -p workshop
cd workshop
```

ดาวน์โหลด repository ด้วยคำสั่ง

```bash
git clone https://github.com/chumnarn/heichips26-digital-workshop.git
```

เข้าสู่ repository

```bash
cd heichips26-digital-workshop
```

ตรวจสอบรายการไฟล์

```bash
ls
```

จากนั้นเข้าสู่ Exercise 1

```bash
cd exercise_1
```

ตรวจสอบตำแหน่งปัจจุบัน

```bash
pwd
```

ผลลัพธ์ควรมีลักษณะคล้าย

```text
/home/<username>/workshop/heichips26-digital-workshop/exercise_1
```

ตรวจสอบไฟล์ภายในไดเรกทอรี

```bash
ls -la
```

ควรพบไฟล์อย่างน้อยดังนี้

```text
README.md
config.yaml
counter.sv
img
```

---

## 6. เปิด LibreLane Environment

วิธีเข้าสู่ LibreLane environment ขึ้นอยู่กับรูปแบบการติดตั้งที่ใช้ในชั้นเรียน

### 6.1 กรณีใช้ Nix

เข้าสู่ LibreLane environment จากไดเรกทอรีที่มีไฟล์ Nix configuration เช่น

```bash
nix-shell
```

หรือในระบบที่ใช้ Flake

```bash
nix develop
```

จากนั้นกลับเข้าสู่ Exercise 1

```bash
cd ~/workshop/heichips26-digital-workshop/exercise_1
```

### 6.2 กรณีใช้ Docker

หากสภาพแวดล้อมของหลักสูตรเตรียม Docker wrapper ไว้แล้ว ให้ใช้คำสั่งตามที่กำหนดใน Lab การติดตั้งเครื่องมือ

### 6.3 ตรวจสอบคำสั่ง LibreLane

ใช้คำสั่ง

```bash
librelane --version
```

หรือ

```bash
librelane --help
```

หากติดตั้งถูกต้อง โปรแกรมจะแสดงหมายเลขเวอร์ชันหรือรายการตัวเลือกของคำสั่ง

หากพบข้อความ

```text
librelane: command not found
```

แสดงว่ายังไม่ได้เข้าสู่ environment ที่ติดตั้ง LibreLane หรือ path ของโปรแกรมยังไม่ถูกตั้งค่า

---

## 7. ตรวจสอบเครื่องมือพื้นฐาน

ตรวจสอบเครื่องมือที่ LibreLane อาจเรียกใช้

```bash
yosys -V
```

```bash
openroad -version
```

```bash
klayout -v
```

บาง environment อาจไม่ได้เปิดให้เรียกเครื่องมือย่อยทั้งหมดโดยตรง แต่สามารถเรียกผ่าน LibreLane ได้ ดังนั้นข้อสำคัญที่สุดคือคำสั่งต่อไปนี้ต้องทำงานได้

```bash
librelane --help
```

---

# ส่วนที่ 2: ศึกษา RTL ของวงจร Counter

## 8. เปิดดูไฟล์ `counter.sv`

ใช้คำสั่ง

```bash
cat counter.sv
```

RTL ที่ใช้มีโครงสร้างดังนี้

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

---

## 9. วิเคราะห์โครงสร้าง Module

ส่วนประกาศ module คือ

```systemverilog
module counter (
    input  logic       clk_i,
    input  logic       rst_ni,
    output logic [7:0] count_o
);
```

ชื่อ top-level module คือ

```text
counter
```

ชื่อนี้ต้องตรงกับค่าของ `DESIGN_NAME` ในไฟล์ `config.yaml`

อินพุต `clk_i` เป็นสัญญาณนาฬิกา ส่วน `rst_ni` เป็นสัญญาณรีเซตแบบ Active-Low สังเกตได้จากอักษร `_n` ในชื่อสัญญาณ และเงื่อนไข

```systemverilog
if (!rst_ni)
```

เมื่อ `rst_ni = 0` นิพจน์ `!rst_ni` จะเป็นจริง จึงเข้าสู่เงื่อนไขรีเซต

เอาต์พุต

```systemverilog
output logic [7:0] count_o
```

มีขนาด 8 บิต จึงแทนค่าได้ตั้งแต่

```text
0 ถึง 255
```

หรือในเลขฐานสิบหกตั้งแต่

```text
8'h00 ถึง 8'hFF
```

เมื่อค่าเพิ่มจาก `8'hFF` อีกหนึ่งครั้ง จะเกิดการวนกลับเป็น `8'h00` เนื่องจากรีจิสเตอร์มีขนาดเพียง 8 บิต

---

## 10. วิเคราะห์ Sequential Logic

บล็อกหลักของวงจรคือ

```systemverilog
always_ff @(posedge clk_i)
```

`always_ff` ใช้สำหรับอธิบาย Sequential Logic โดยเฉพาะ ในกรณีนี้วงจรทำงานที่ขอบขาขึ้นของสัญญาณนาฬิกา

ภายในบล็อกมีเงื่อนไข

```systemverilog
if (!rst_ni) begin
    count_o <= '0;
end
```

คำสั่ง

```systemverilog
count_o <= '0;
```

กำหนดค่าทุกบิตของ `count_o` เป็นศูนย์ ซึ่งสำหรับสัญญาณ 8 บิตมีค่าเท่ากับ

```systemverilog
count_o <= 8'b0000_0000;
```

กรณีไม่อยู่ในสถานะรีเซต วงจรทำงานดังนี้

```systemverilog
count_o <= count_o + 1;
```

ค่าปัจจุบันของเคาน์เตอร์จะเพิ่มขึ้นหนึ่งในทุกขอบขาขึ้นของ clock

---

## 11. เหตุผลที่ใช้ Nonblocking Assignment

ใน Sequential Logic ใช้ตัวดำเนินการ

```systemverilog
<=
```

ซึ่งเรียกว่า Nonblocking Assignment

การใช้ Nonblocking Assignment ทำให้การอัปเดตค่ารีจิสเตอร์เกิดขึ้นตามพฤติกรรมของ Flip-Flop จริง และช่วยป้องกันปัญหาลำดับการประมวลผลระหว่างรีจิสเตอร์หลายตัว

หลักการทั่วไปคือ

```text
Combinational Logic  → ใช้ =
Sequential Logic     → ใช้ <=
```

---

## 12. ลักษณะของ Reset

วงจรนี้ใช้ Synchronous Active-Low Reset เนื่องจาก `rst_ni` ถูกตรวจสอบภายใน

```systemverilog
always_ff @(posedge clk_i)
```

ไม่มี `negedge rst_ni` อยู่ใน sensitivity list

ดังนั้นเมื่อเปลี่ยน `rst_ni` จาก `1` เป็น `0` ค่า `count_o` จะยังไม่ถูกรีเซตทันที แต่จะถูกรีเซตเมื่อเกิดขอบขาขึ้นของ `clk_i` ครั้งถัดไป

เปรียบเทียบกับ Asynchronous Reset ซึ่งมักเขียนในรูป

```systemverilog
always_ff @(posedge clk_i or negedge rst_ni)
```

ในกรณีนั้นเอาต์พุตจะถูกรีเซตทันทีเมื่อ `rst_ni` ลดลงเป็นศูนย์ โดยไม่ต้องรอ clock

---

## 13. คาดการณ์ฮาร์ดแวร์หลังการสังเคราะห์

จาก RTL นี้ เครื่องมือสังเคราะห์ควรสร้างฮาร์ดแวร์หลักดังนี้

```text
8-bit Register
     ▲
     │
8-bit Adder (+1)
     ▲
     │
Current count_o
```

ในระดับ Standard Cell วงจรจะประกอบด้วยองค์ประกอบ เช่น

- Flip-Flop จำนวนประมาณ 8 ตัว
- Logic สำหรับ Incrementer
- Clock buffers
- Reset-related logic
- Buffer หรือ Inverter ตามความจำเป็น
- Tie cells และ physical-only cells ที่เพิ่มในขั้นตอน Physical Design

จำนวนเซลล์จริงอาจแตกต่างกันตามการ optimize ของ Yosys, ABC และไลบรารีมาตรฐานของ PDK

---

# ส่วนที่ 3: ศึกษา LibreLane Configuration

## 14. เปิดดูไฟล์ `config.yaml`

ใช้คำสั่ง

```bash
cat config.yaml
```

เนื้อหาหลักมีลักษณะดังนี้

```yaml
# LibreLane configuration file

DESIGN_NAME: counter
VERILOG_FILES: dir::counter.sv
CLOCK_PORT: clk_i
CLOCK_PERIOD: 10
```

---

## 15. ความหมายของ `DESIGN_NAME`

```yaml
DESIGN_NAME: counter
```

ตัวแปรนี้ระบุชื่อ top-level module

ค่าที่กำหนดต้องตรงกับชื่อ module ใน RTL

```systemverilog
module counter (
```

หากกำหนดชื่อไม่ตรงกัน เช่น

```yaml
DESIGN_NAME: counter_top
```

แต่ไม่มี module ชื่อ `counter_top` เครื่องมือจะไม่สามารถ elaborate design ได้และ flow จะหยุดในขั้นตอนต้น

---

## 16. ความหมายของ `VERILOG_FILES`

```yaml
VERILOG_FILES: dir::counter.sv
```

ตัวแปรนี้ระบุไฟล์ RTL ที่ LibreLane ต้องอ่าน

คำนำหน้า

```text
dir::
```

หมายถึงตำแหน่งของไฟล์อ้างอิงจากไดเรกทอรีที่เก็บ `config.yaml`

ดังนั้น

```yaml
VERILOG_FILES: dir::counter.sv
```

หมายถึงไฟล์

```text
exercise_1/counter.sv
```

สำหรับโครงการที่มีหลายไฟล์สามารถเขียนเป็นรายการได้ เช่น

```yaml
VERILOG_FILES:
  - dir::rtl/counter.sv
  - dir::rtl/control.sv
  - dir::rtl/top.sv
```

หรือใช้ wildcard เช่น

```yaml
VERILOG_FILES:
  - dir::rtl/*.sv
```

อย่างไรก็ตาม ในโครงการขนาดใหญ่ควรควบคุมลำดับไฟล์ให้ถูกต้อง โดยเฉพาะเมื่อใช้ package, interface หรือไฟล์ header

---

## 17. ความหมายของ `CLOCK_PORT`

```yaml
CLOCK_PORT: clk_i
```

กำหนดชื่อ clock port หลักของ design

ค่าต้องตรงกับชื่อสัญญาณใน top-level module

```systemverilog
input logic clk_i
```

LibreLane ใช้ข้อมูลนี้เพื่อสร้าง default timing constraint และใช้ในขั้นตอนต่าง ๆ เช่น

- Static Timing Analysis
- Clock Tree Synthesis
- Clock Routing
- Timing Optimization

หากระบุชื่อ clock ผิด LibreLane อาจแจ้งว่าไม่พบ clock port หรือไม่สามารถสร้าง clock constraint ได้

---

## 18. ความหมายของ `CLOCK_PERIOD`

```yaml
CLOCK_PERIOD: 10
```

ค่ามีหน่วยเป็นนาโนวินาที

ความถี่สัญญาณนาฬิกาคำนวณจาก

```text
f = 1 / T
```

เมื่อ

```text
T = 10 ns
```

ดังนั้น

```text
f = 1 / 10 ns
  = 100 MHz
```

ตารางตัวอย่างความสัมพันธ์ระหว่าง period และ frequency

| Clock period | Clock frequency |
|---:|---:|
| 20 ns | 50 MHz |
| 10 ns | 100 MHz |
| 5 ns | 200 MHz |
| 2 ns | 500 MHz |

การกำหนด period ที่สั้นลงหมายถึง constraint ที่เข้มงวดขึ้น เครื่องมือต้องพยายามทำให้ combinational path ทำงานได้เร็วขึ้น ซึ่งอาจทำให้

- ใช้เซลล์ขนาดใหญ่ขึ้น
- ใช้ buffer เพิ่มขึ้น
- พื้นที่เพิ่มขึ้น
- กำลังไฟเพิ่มขึ้น
- Routing ยากขึ้น
- เกิด timing violation ได้ง่ายขึ้น

---

# ส่วนที่ 4: รัน RTL-to-GDSII Flow

## 19. ตรวจสอบตำแหน่งก่อนรัน

ต้องอยู่ในไดเรกทอรี `exercise_1`

```bash
pwd
```

ตรวจสอบไฟล์

```bash
ls
```

ควรพบ

```text
config.yaml
counter.sv
```

---

## 20. เรียกใช้ LibreLane

ใช้คำสั่ง

```bash
librelane --pdk ihp-sg13g2 config.yaml
```

ความหมายของแต่ละส่วนคือ

```text
librelane
```

เรียกโปรแกรม LibreLane

```text
--pdk ihp-sg13g2
```

เลือกใช้ IHP SG13G2 PDK

```text
config.yaml
```

ระบุไฟล์ configuration ของ design

หากไม่ได้ระบุ flow เพิ่มเติม LibreLane จะใช้ Classic flow ซึ่งประกอบด้วยขั้นตอนตั้งแต่ synthesis ไปจนถึง signoff checks ตามการตั้งค่าของ LibreLane รุ่นที่ติดตั้ง

---

## 21. การดาวน์โหลด PDK ครั้งแรก

หากเป็นการใช้งาน `ihp-sg13g2` ครั้งแรก LibreLane อาจดาวน์โหลดและเตรียม PDK ก่อนเริ่ม flow

โดยทั่วไป PDK ที่ LibreLane จัดการอัตโนมัติจะถูกเก็บภายใต้

```text
~/.ciel
```

ตรวจสอบได้ด้วย

```bash
ls ~/.ciel
```

ในขั้นตอนนี้จำเป็นต้องมี

- การเชื่อมต่อเครือข่าย
- พื้นที่ว่างเพียงพอ
- สิทธิ์เขียนไฟล์ใน home directory

หาก PDK มีอยู่แล้ว LibreLane จะใช้เวอร์ชันที่ตรงกับ environment โดยไม่ต้องดาวน์โหลดใหม่ทั้งหมด

---

## 22. ข้อความที่ปรากฏระหว่างรัน

ระหว่างรันจะเห็นข้อความจำนวนมากจากขั้นตอนต่าง ๆ ตัวอย่างกลุ่มขั้นตอนที่อาจพบ ได้แก่

```text
Yosys.JsonHeader
Yosys.Synthesis
OpenROAD.CheckSDCFiles
OpenROAD.Floorplan
OpenROAD.IOPlacement
OpenROAD.GeneratePDN
OpenROAD.GlobalPlacement
OpenROAD.DetailedPlacement
OpenROAD.CTS
OpenROAD.GlobalRouting
OpenROAD.DetailedRouting
OpenROAD.RCX
OpenROAD.STA
KLayout.StreamOut
Magic.DRC
Netgen.LVS
```

ชื่อและหมายเลขลำดับของแต่ละขั้นตอนอาจแตกต่างกันตามเวอร์ชัน LibreLane และ configuration

ไม่ควรตัดสินว่า flow ล้มเหลวจากข้อความ `WARNING` เพียงอย่างเดียว ต้องตรวจสอบสถานะสุดท้ายและรายงาน error ประกอบ

---

## 23. ขั้นตอน Logic Synthesis

ในขั้นตอน Synthesis เครื่องมือจะ

1. อ่านไฟล์ `counter.sv`
2. ตรวจสอบ syntax
3. Elaborate top module ชื่อ `counter`
4. แปลง RTL เป็นโครงสร้าง logic
5. Optimize logic
6. Map logic ลง standard cells ของ IHP SG13G2
7. สร้าง gate-level netlist

สำหรับเคาน์เตอร์ 8 บิต เครื่องมือควรอนุมาน

- รีจิสเตอร์ 8 บิต
- วงจรบวกหนึ่ง
- Logic ที่เกี่ยวข้องกับ reset

ไฟล์ผลลัพธ์อาจพบในไดเรกทอรีของขั้นตอน Yosys เช่น

```text
runs/<RUN_TAG>/<step>-yosys-synthesis/
```

---

## 24. ขั้นตอน Floorplanning

Floorplanning กำหนด

- ขอบเขต die
- ขอบเขต core
- ตำแหน่งแถว standard cell
- พื้นที่สำหรับการวางเซลล์
- โครงสร้าง power distribution เบื้องต้น
- ตำแหน่ง I/O pin

เนื่องจาก counter มีขนาดเล็ก พื้นที่ core ที่ได้จะมีขนาดไม่ใหญ่มาก แต่ยังต้องมีพื้นที่ขั้นต่ำเพื่อรองรับ

- Standard cells
- Tap cells
- Filler cells
- Clock buffers
- Power rails
- Routing tracks

---

## 25. ขั้นตอน Placement

Placement เป็นการกำหนดตำแหน่งของ standard cell แต่ละตัวภายใน core

แบ่งเป็นสองช่วงหลัก

### 25.1 Global Placement

วางตำแหน่งโดยประมาณเพื่อให้

- ความยาวสายรวมต่ำ
- Congestion ต่ำ
- Timing ดี
- Density เหมาะสม

### 25.2 Detailed Placement

ปรับตำแหน่งให้เซลล์อยู่บน legal site และไม่มีการซ้อนทับกัน

หลัง Detailed Placement ทุก standard cell ต้องวางอยู่ในตำแหน่งที่ถูกต้องตามข้อกำหนดของ standard-cell row

---

## 26. ขั้นตอน Clock Tree Synthesis

Clock Tree Synthesis หรือ CTS สร้างเครือข่ายกระจายสัญญาณ `clk_i` ไปยัง Flip-Flop ทั้งแปดตัว

เป้าหมายสำคัญคือ

- ลด Clock Skew
- ควบคุม Clock Latency
- ควบคุม Slew
- ลด fanout ต่อ buffer
- ทำให้ทุก sequential endpoint ได้รับ clock ที่ใกล้เคียงกัน

แม้ว่าวงจรนี้มี Flip-Flop เพียงแปดตัว เครื่องมืออาจยังเพิ่ม Clock Buffer หลายระดับตามกฎและ strategy ของ PDK

โครงสร้างเชิงแนวคิดคือ

```text
clk_i
  │
  ▼
Root Clock Buffer
  │
  ├── Clock Buffer ──► Flip-Flops
  │
  └── Clock Buffer ──► Flip-Flops
```

---

## 27. ขั้นตอน Routing

Routing แบ่งเป็นสองส่วน

### 27.1 Global Routing

ค้นหาเส้นทางโดยประมาณสำหรับแต่ละ net และประเมิน congestion

### 27.2 Detailed Routing

สร้าง geometry จริงบน metal layers โดยคำนึงถึงกฎการผลิต เช่น

- Minimum width
- Minimum spacing
- Via enclosure
- Via spacing
- End-of-line spacing
- Routing direction ของแต่ละชั้นโลหะ

เมื่อ Detailed Routing สำเร็จ ทุก signal net ต้องถูกเชื่อมต่อครบ และไม่มี routing short หรือ open ที่ร้ายแรง

---

## 28. Parasitic Extraction และ Static Timing Analysis

หลัง Routing เครื่องมือจะประมาณหรือสกัดค่า parasitic ของสาย ได้แก่

- Resistance
- Capacitance

ค่าดังกล่าวมีผลต่อ

- Cell delay
- Net delay
- Slew
- Setup timing
- Hold timing

Static Timing Analysis จะตรวจสอบว่าเส้นทางข้อมูลทำงานทันภายใน clock period ที่กำหนดหรือไม่

สำหรับ clock period 10 ns เป้าหมายคือให้ setup slack ไม่ติดลบ

แนวคิดของ Setup Slack คือ

```text
Setup Slack = Required Arrival Time - Actual Arrival Time
```

ถ้า

```text
Slack > 0
```

หมายถึงผ่าน timing

ถ้า

```text
Slack = 0
```

หมายถึงพอดีกับข้อกำหนด

ถ้า

```text
Slack < 0
```

หมายถึงเกิด timing violation

---

## 29. Physical Verification

หลังสร้าง layout แล้ว LibreLane จะตรวจสอบประเด็นสำคัญดังนี้

### 29.1 DRC

Design Rule Check ตรวจสอบว่า geometry ของ layout เป็นไปตามกฎการผลิตหรือไม่ เช่น

- ความกว้างต่ำสุด
- ระยะห่างต่ำสุด
- Via enclosure
- Metal overlap
- Density rules

ผลที่ต้องการคือ

```text
DRC Passed
```

หรือจำนวน violation ที่ยอมรับได้ตามเกณฑ์ของโครงการ แต่สำหรับ Exercise นี้ควรคาดหวังว่าไม่มี violation

### 29.2 LVS

Layout Versus Schematic เปรียบเทียบ connectivity ของ layout กับ netlist

LVS ตรวจสอบว่า

- จำนวนอุปกรณ์สอดคล้องกัน
- จำนวน net สอดคล้องกัน
- การเชื่อมต่อ pin ถูกต้อง
- ไม่มี short
- ไม่มี open
- ไม่มีอุปกรณ์หายหรือเกิน

ผลที่ต้องการคือ

```text
LVS Passed
```

### 29.3 Antenna Check

Antenna check ตรวจสอบความเสี่ยงจากประจุที่สะสมบนสายโลหะระหว่างกระบวนการผลิต ซึ่งอาจทำลาย gate oxide

ผลที่ต้องการคือ

```text
Antenna Passed
```

---

## 30. ตรวจสอบผลลัพธ์เมื่อ Flow สิ้นสุด

เมื่อ flow สำเร็จ ส่วนสรุปท้าย log ควรแสดงผลในลักษณะ

```text
Antenna
Passed

LVS
Passed

DRC
Passed
```

อาจมี warning บางรายการที่ไม่ทำให้ flow ล้มเหลว อย่างไรก็ตามควรอ่าน warning ทุกครั้ง ไม่ควรถือว่า warning ทุกชนิดสามารถละเลยได้

ตรวจสอบ exit status ของคำสั่งล่าสุดได้ด้วย

```bash
echo $?
```

ค่าที่ต้องการคือ

```text
0
```

ค่าอื่นที่ไม่ใช่ศูนย์มักหมายถึงคำสั่งสิ้นสุดด้วยข้อผิดพลาด

---

# ส่วนที่ 5: ตรวจสอบไดเรกทอรีผลลัพธ์

## 31. ค้นหาไดเรกทอรี Run

หลังรัน LibreLane จะสร้างไดเรกทอรีผลลัพธ์ โดยชื่ออาจเป็น `runs/` หรือ `run/` ตามเวอร์ชันและรูปแบบของโครงการ

ตรวจสอบด้วย

```bash
ls
```

จากนั้นทดลอง

```bash
ls runs
```

หากไม่พบ ให้ทดลอง

```bash
ls run
```

ตัวอย่างชื่อ run tag

```text
RUN_2026-07-20_08-30-15
```

รายชื่อจริงจะอ้างอิงวันที่และเวลาที่เริ่มรัน

---

## 32. หา Run ล่าสุด

กรณีใช้ไดเรกทอรี `runs`

```bash
ls -td runs/RUN_* | head -1
```

เก็บ path ของ run ล่าสุดลงตัวแปร

```bash
LATEST_RUN=$(ls -td runs/RUN_* | head -1)
```

แสดงค่า

```bash
echo "$LATEST_RUN"
```

กรณีระบบใช้ `run`

```bash
LATEST_RUN=$(ls -td run/RUN_* | head -1)
```

---

## 33. ตรวจสอบไฟล์ระดับบนของ Run

```bash
ls -la "$LATEST_RUN"
```

อาจพบไฟล์ เช่น

```text
config.yaml
resolved.json
state_out.json
warning.log
error.log
final/
```

ชื่อไฟล์จริงขึ้นอยู่กับเวอร์ชัน LibreLane

ค้นหาไฟล์ทั้งหมดแบบย่อ

```bash
find "$LATEST_RUN" -maxdepth 2 -type f | sort | less
```

ออกจาก `less` ด้วยปุ่ม

```text
q
```

---

## 34. ตรวจสอบ Warning และ Error

ค้นหาไฟล์ที่เกี่ยวข้อง

```bash
find "$LATEST_RUN" -type f \
  \( -iname "*warning*" -o -iname "*error*" \) \
  -print
```

อ่าน warning

```bash
find "$LATEST_RUN" -type f -iname "*warning*" -exec sh -c '
  echo "===== $1 ====="
  cat "$1"
' sh {} \;
```

อ่าน error

```bash
find "$LATEST_RUN" -type f -iname "*error*" -exec sh -c '
  echo "===== $1 ====="
  cat "$1"
' sh {} \;
```

หาก flow สำเร็จ ไฟล์ error อาจว่างหรือไม่มี critical error

---

## 35. ค้นหา Gate-Level Netlist

```bash
find "$LATEST_RUN" -type f \
  \( -name "*.nl.v" -o -name "*.pnl.v" -o -name "*netlist*.v" \) \
  -print
```

ความหมายโดยทั่วไปคือ

- `.nl.v` อาจเป็น netlist ที่ยังไม่รวม power connections
- `.pnl.v` อาจเป็น powered netlist ที่เพิ่ม power และ ground connections
- ชื่อและรูปแบบขึ้นกับขั้นตอนของ flow

เปิดดูส่วนต้นของ netlist

```bash
NETLIST=$(find "$LATEST_RUN" -type f -name "*.nl.v" | tail -1)
head -80 "$NETLIST"
```

สังเกตว่า RTL ระดับพฤติกรรม เช่น

```systemverilog
count_o <= count_o + 1;
```

ถูกแทนด้วย instance ของ standard cell แล้ว

---

## 36. ค้นหาไฟล์ OpenDB

```bash
find "$LATEST_RUN" -type f -name "*.odb" -print
```

ไฟล์ `.odb` เก็บฐานข้อมูล physical design ของ OpenROAD เช่น

- ตำแหน่งเซลล์
- Floorplan
- Net connectivity
- Routing
- Clock tree
- Timing-related data

OpenROAD GUI ใช้ไฟล์นี้เพื่อเปิดดูและ debug design

---

## 37. ค้นหาไฟล์ DEF

```bash
find "$LATEST_RUN" -type f -name "*.def" -print
```

DEF หรือ Design Exchange Format เก็บข้อมูล physical implementation เช่น

- Die area
- Component placement
- Pin placement
- Routing
- Vias
- Special nets

ดูส่วนต้นของไฟล์

```bash
DEF_FILE=$(find "$LATEST_RUN" -type f -name "*.def" | tail -1)
head -60 "$DEF_FILE"
```

---

## 38. ค้นหาไฟล์ GDSII

```bash
find "$LATEST_RUN" -type f \
  \( -name "*.gds" -o -name "*.gdsii" \) \
  -print
```

GDSII เป็นข้อมูล geometry สำหรับส่งต่อไปยังการตรวจสอบขั้นสุดท้ายหรือกระบวนการผลิต

ตัวอย่างการตรวจสอบขนาดไฟล์

```bash
GDS_FILE=$(find "$LATEST_RUN" -type f -name "*.gds" | tail -1)
ls -lh "$GDS_FILE"
```

---

## 39. ค้นหารายงาน Timing

```bash
find "$LATEST_RUN" -type f \
  \( -iname "*timing*" -o -iname "*sta*" -o -iname "*wns*" \) \
  -print
```

ค้นหาคำว่า slack ในรายงาน

```bash
grep -Rni "slack" "$LATEST_RUN" | head -30
```

ค้นหา WNS

```bash
grep -Rni "WNS" "$LATEST_RUN" | head -20
```

ค้นหา TNS

```bash
grep -Rni "TNS" "$LATEST_RUN" | head -20
```

ความหมายคือ

- WNS: Worst Negative Slack
- TNS: Total Negative Slack

ค่าที่ต้องการสำหรับ design ที่ผ่าน timing คือ

```text
WNS >= 0
TNS = 0
```

รูปแบบรายงานจริงอาจแสดงค่าและชื่อ metric แตกต่างกันเล็กน้อย

---

## 40. ค้นหารายงาน Area

```bash
grep -Rni \
  -e "Design area" \
  -e "Chip area" \
  -e "Core area" \
  -e "utilization" \
  "$LATEST_RUN" | head -40
```

ข้อมูลที่ควรสังเกต ได้แก่

- Standard-cell area
- Core area
- Die area
- Core utilization
- จำนวน instance
- จำนวน sequential cells
- จำนวน combinational cells

---

## 41. ค้นหารายงาน DRC, LVS และ Antenna

### DRC

```bash
find "$LATEST_RUN" -type f -iname "*drc*" -print
```

```bash
grep -Rni \
  -e "violation" \
  -e "violations" \
  "$LATEST_RUN" | grep -i drc | head -30
```

### LVS

```bash
find "$LATEST_RUN" -type f -iname "*lvs*" -print
```

```bash
grep -Rni \
  -e "circuits match" \
  -e "mismatch" \
  -e "LVS" \
  "$LATEST_RUN" | head -30
```

### Antenna

```bash
find "$LATEST_RUN" -type f -iname "*antenna*" -print
```

```bash
grep -Rni "antenna" "$LATEST_RUN" | tail -30
```

---

# ส่วนที่ 6: เปิด Design ด้วย OpenROAD GUI

## 42. เปิด Run ล่าสุดใน OpenROAD

จากไดเรกทอรี `exercise_1` ใช้คำสั่ง

```bash
librelane \
  --pdk ihp-sg13g2 \
  config.yaml \
  --last-run \
  --flow OpenInOpenROAD
```

ตัวเลือก

```text
--last-run
```

สั่งให้ LibreLane ใช้ run ล่าสุดแทนการสร้าง physical implementation ใหม่

ตัวเลือก

```text
--flow OpenInOpenROAD
```

เลือก flow สำหรับเปิดฐานข้อมูล design ใน OpenROAD GUI

---

## 43. ส่วนประกอบของ OpenROAD GUI

เมื่อ GUI เปิดขึ้น โดยทั่วไปจะพบ

- พื้นที่แสดง layout อยู่ตรงกลาง
- Display Control อยู่ด้านซ้าย
- Inspector หรือ Object Browser อยู่ด้านขวา
- เมนูควบคุมอยู่ด้านบน
- Console หรือ Scripting panel อยู่ด้านล่างหรือเปิดเพิ่มได้

หากไม่เห็น layout ให้ใช้คำสั่งหรือปุ่ม

```text
Zoom Fit
```

หรือกดปุ่มที่มีสัญลักษณ์ fit-to-window

---

## 44. ตรวจสอบ Die และ Core Area

สังเกตกรอบภายนอกและพื้นที่ภายใน

- Die area คือขอบเขตทั้งหมดของ block
- Core area คือพื้นที่วาง standard cells
- Standard-cell rows อยู่ภายใน core
- I/O pins มักอยู่ตามขอบเขต block
- Power straps และ rails เชื่อมต่อภายในพื้นที่ core

ให้ลองเปิดและปิดการแสดงผลของ

- Rows
- Instances
- Pins
- Nets
- Tracks
- Routing layers
- Power nets

เพื่อทำความเข้าใจองค์ประกอบของ layout

---

## 45. ตรวจสอบ Standard Cells

ขยายภาพเข้าไปในบริเวณ core

คลิกเลือก standard cell แล้วดูข้อมูลจาก Inspector เช่น

- Instance name
- Master cell name
- Location
- Orientation
- Connected nets
- Pin names

พยายามค้นหาเซลล์ประเภท

- Flip-Flop
- Inverter
- Buffer
- Clock Buffer
- Logic Gate
- Filler
- Tap Cell

จำนวน Flip-Flop ที่เกี่ยวข้องกับ `count_o` ควรสอดคล้องกับขนาดเคาน์เตอร์ 8 บิต แม้เครื่องมืออาจ optimize หรือ map รูปแบบเซลล์แตกต่างจากที่คาดไว้

---

## 46. เปิด Clock Tree Viewer

จากเมนูเลือก

```text
Windows → Clock Tree Viewer
```

จากนั้นกด

```text
Update
```

เลือก clock ที่เกี่ยวข้องกับ `clk_i`

Clock Tree Viewer จะแสดง

- Clock root
- Clock buffers
- ลำดับชั้นของ clock tree
- Clock sinks
- Flip-Flop endpoints

สำหรับวงจร counter ขนาดเล็ก Clock Tree มีจำนวนกิ่งไม่มาก แต่ยังช่วยให้เห็นการกระจาย clock ไปยังรีจิสเตอร์ทั้งแปดบิต

---

## 47. วิเคราะห์ Clock Tree

ประเด็นที่ควรตรวจสอบ ได้แก่

1. Clock root เชื่อมต่อจาก `clk_i`
2. มี buffer ถูกแทรกใน clock network
3. Clock sinks เชื่อมต่อไปยัง Flip-Flop
4. ไม่มี sink ที่ไม่ได้รับ clock
5. โครงสร้าง tree ไม่ลึกเกินความจำเป็น
6. Routing ของ clock แยกออกจาก data nets ได้ชัดเจน

Clock Tree Synthesis ไม่ได้เพียงเชื่อมสาย clock แต่ต้องควบคุมคุณภาพของ clock network ทั้ง skew, latency และ slew

---

## 48. เปิด Timing Report

จากเมนูเลือก

```text
Windows → Timing Report
```

จากนั้นกด

```text
Update
```

เลือก timing path ที่ต้องการ

หน้าต่าง timing report อาจแสดง

- Startpoint
- Endpoint
- Clock
- Path delay
- Cell delay
- Net delay
- Required time
- Arrival time
- Slack

คลิก path เพื่อ highlight เส้นทางบน layout

---

## 49. วิเคราะห์ Timing Path

เส้นทางของวงจร counter โดยทั่วไปมีลักษณะ

```text
Flip-Flop Q
    │
    ▼
Incrementer Logic
    │
    ▼
Flip-Flop D
```

ข้อมูลเริ่มต้นจากเอาต์พุตของ Flip-Flop ชุดหนึ่ง ผ่าน logic สำหรับบวกหนึ่ง แล้วกลับเข้าสู่อินพุต D ของ Flip-Flop

Critical path อาจอยู่ที่บิตสูงของ counter เพราะ carry ต้อง propagate ผ่าน logic หลายระดับ

อย่างไรก็ตาม เครื่องมือ synthesis อาจสร้างโครงสร้าง incrementer ที่ optimize แล้ว จึงไม่ควรสรุปจาก RTL เพียงอย่างเดียว ต้องตรวจสอบ timing report จริง

---

## 50. เปิด Congestion และ Heatmap

ใน OpenROAD GUI สามารถทดลองเปิด heatmap เช่น

- Placement density
- Routing congestion
- Power density
- IR drop หาก flow มีข้อมูลรองรับ

สำหรับ design ขนาดเล็ก heatmap อาจไม่มีบริเวณวิกฤตชัดเจน แต่เป็นพื้นฐานสำคัญสำหรับการ debug design ขนาดใหญ่

---

## 51. บันทึกภาพจาก OpenROAD

ตั้งพื้นหลังเป็นสีขาวจาก

```text
Display Control → Misc → Background
```

เปิด Scripting หรือ Tcl console แล้วใช้

```tcl
save_image counter_openroad.png -width 4096
```

สำหรับบันทึกภาพ clock tree อาจใช้

```tcl
save_clocktree_image
```

ตำแหน่งไฟล์ขึ้นอยู่กับ working directory ของ OpenROAD ควรตรวจสอบด้วย

```tcl
pwd
```

---

# ส่วนที่ 7: เปิด GDSII ด้วย KLayout

## 52. เปิด Run ล่าสุดใน KLayout

ปิด OpenROAD GUI ก่อนหากต้องการลดการใช้หน่วยความจำ แล้วใช้คำสั่ง

```bash
librelane \
  --pdk ihp-sg13g2 \
  config.yaml \
  --last-run \
  --flow OpenInKLayout
```

LibreLane จะเลือกไฟล์ layout จาก run ล่าสุดและเปิดด้วย KLayout

---

## 53. ส่วนประกอบของ KLayout

หน้าต่าง KLayout โดยทั่วไปประกอบด้วย

- Layout viewer อยู่ตรงกลาง
- Cell hierarchy อยู่ด้านซ้าย
- Layer list อยู่ด้านขวา
- Toolbar สำหรับ zoom, select และ ruler
- Status bar แสดงพิกัดและข้อมูลวัตถุ

หากไม่เห็น layout ทั้งหมด ให้ใช้

```text
Zoom Fit
```

---

## 54. ตรวจสอบ Cell Hierarchy

ใน Cell hierarchy ให้หา top cell ชื่อ

```text
counter
```

หรือชื่อที่ flow ใช้เป็น top layout

ขยาย hierarchy เพื่อดู

- Standard-cell instances
- Filler cells
- Tap cells
- Decap cells หากมี
- Physical-only cells
- Routing geometry

KLayout แสดงข้อมูล geometry ตาม GDSII ส่วน OpenROAD แสดงฐานข้อมูล implementation ที่มีข้อมูลเชิง logical และ timing เพิ่มเติม

---

## 55. ตรวจสอบ Layer

ทดลองเปิดและปิด layer ทีละกลุ่ม เช่น

- Active และ diffusion-related layers
- Poly
- Contact
- Metal 1
- Metal 2
- Metal 3
- Upper metals
- Via layers
- Pin หรือ label layers

เมื่อปิด layer ระดับบน จะเห็นโครงสร้าง standard cell ชัดขึ้น

เมื่อเปิดเฉพาะ metal layers จะเห็นโครงข่าย routing และ power distribution

หมายเลขและชื่อ layer จริงขึ้นอยู่กับ layer map ของ IHP SG13G2 PDK

---

## 56. ตรวจสอบ Power Distribution Network

เปิด layer ที่เกี่ยวข้องกับโลหะและสังเกต

- Power rails ภายใน standard-cell rows
- Power stripes หรือ straps
- การเชื่อมต่อ VDD
- การเชื่อมต่อ VSS
- Via arrays ระหว่างโลหะต่างระดับ

Power Distribution Network ต้องกระจายพลังงานไปยัง standard cell ทุกตัวโดยมีความต้านทานและแรงดันตกอยู่ในระดับที่ยอมรับได้

สำหรับ design ขนาดเล็ก PDN อาจดูมีขนาดใหญ่เมื่อเทียบกับ logic เพราะโครงสร้างพื้นฐานยังต้องเป็นไปตามกฎของ PDK

---

## 57. ใช้เครื่องมือวัดระยะ

เลือก Ruler Tool แล้ววัด

- ความกว้าง die
- ความสูง die
- ระยะจาก die boundary ถึง core
- ความกว้าง power stripe
- ระยะห่างระหว่าง stripe
- ขนาด standard cell โดยประมาณ

บันทึกค่าที่วัดได้ลงในรายงาน

---

## 58. เปรียบเทียบ OpenROAD กับ KLayout

| ประเด็น | OpenROAD GUI | KLayout |
|---|---|---|
| ฐานข้อมูลหลัก | OpenDB/ODB | GDS, DEF, LEF |
| Logical connectivity | เห็นได้ชัด | จำกัด |
| Timing analysis | รองรับ | ไม่ใช่หน้าที่หลัก |
| Clock tree viewer | มี | ไม่มีโดยตรง |
| Placement debug | เหมาะมาก | ดู geometry ได้ |
| Routing visualization | มี | มี |
| GDS inspection | ไม่ใช่จุดเด่นหลัก | เหมาะมาก |
| Layer inspection | มี | ละเอียด |
| Measurement | มี | มี |
| Final mask geometry | ดูได้บางระดับ | เหมาะสำหรับตรวจสอบ |

---

# ส่วนที่ 8: วิเคราะห์ผลการทดลอง

## 59. ตรวจสอบจำนวน Sequential Cells

ค้นหารายงาน synthesis หรือ metrics

```bash
grep -Rni \
  -e "sequential" \
  -e "flip-flop" \
  -e "dff" \
  "$LATEST_RUN" | head -40
```

บันทึกจำนวน Sequential Cells ที่พบ

คำถามสำหรับวิเคราะห์:

1. จำนวน Flip-Flop ตรงกับ 8 บิตหรือไม่
2. Reset ถูก implement ด้วย Flip-Flop ที่มี reset pin หรือ logic ภายนอก
3. มี Flip-Flop ชนิดใดจาก standard-cell library
4. มี clock buffer กี่ตัว
5. จำนวน logic cells ของ incrementer เท่าใด

---

## 60. วิเคราะห์ Critical Path

จาก OpenROAD Timing Report หรือรายงาน STA ให้บันทึก

```text
Startpoint:
Endpoint:
Path group:
Data arrival time:
Data required time:
Slack:
```

จากนั้นตอบคำถาม

1. Critical path เริ่มจาก Flip-Flop ตัวใด
2. สิ้นสุดที่ Flip-Flop ตัวใด
3. ผ่าน combinational cell กี่ระดับ
4. Cell delay หรือ net delay มีสัดส่วนมากกว่า
5. Timing ผ่านข้อกำหนด 10 ns หรือไม่

---

## 61. วิเคราะห์พื้นที่

บันทึกข้อมูลต่อไปนี้

```text
Die width:
Die height:
Die area:
Core width:
Core height:
Core area:
Standard-cell area:
Core utilization:
Number of instances:
```

อธิบายว่าเหตุใด design ขนาดเล็กจึงอาจมี utilization ต่ำ

สาเหตุที่เป็นไปได้ ได้แก่

- มี minimum die size
- ต้องเว้นพื้นที่สำหรับ routing
- ต้องมี power grid
- ต้องมี tap และ filler cells
- ต้องรักษาระยะห่างตาม design rules
- เครื่องมือใช้ค่าพื้นฐานที่ออกแบบมาสำหรับ design ทั่วไป

---

## 62. วิเคราะห์ DRC

บันทึกผล

```text
DRC tool:
Number of violations:
Result:
```

หากไม่มี violation ให้อธิบายว่า

```text
Layout ผ่านกฎทางเรขาคณิตที่ flow ตรวจสอบภายใต้ rule deck ที่ใช้งาน
```

ไม่ควรตีความว่า layout ผ่านทุกข้อกำหนดสำหรับการผลิตโดยอัตโนมัติ เพราะเกณฑ์ tapeout จริงอาจต้องใช้ rule deck, waiver และ signoff procedure เพิ่มเติม

---

## 63. วิเคราะห์ LVS

บันทึกผล

```text
Layout netlist:
Schematic netlist:
Number of devices:
Number of nets:
Result:
```

ผลที่ต้องการคือ layout และ schematic มี connectivity สอดคล้องกัน

หาก LVS ผ่าน หมายความว่า physical layout มีโครงสร้างทางไฟฟ้าตรงกับ netlist อ้างอิงตามขอบเขตที่ tool ตรวจสอบ

---

## 64. วิเคราะห์ Antenna

บันทึกผล

```text
Number of antenna violations before repair:
Number of antenna violations after repair:
Final result:
```

หากมีการแทรก antenna diode ให้ตรวจสอบจาก netlist หรือ placement ว่ามีเซลล์ดังกล่าวหรือไม่

---

# ส่วนที่ 9: การทดลองเพิ่มเติม

## 65. Experiment 1: เปลี่ยน Clock Period เป็น 20 ns

สำรองไฟล์เดิม

```bash
cp config.yaml config_10ns.yaml
```

แก้ไข

```yaml
CLOCK_PERIOD: 20
```

รันใหม่

```bash
librelane --pdk ihp-sg13g2 config.yaml
```

เปรียบเทียบกับผล 10 ns ในหัวข้อต่อไปนี้

- WNS
- TNS
- จำนวนเซลล์
- จำนวน buffer
- พื้นที่
- กำลังไฟ หากมีรายงาน
- Clock tree
- Runtime

Clock period 20 ns เท่ากับความถี่

```text
50 MHz
```

constraint ที่ผ่อนคลายขึ้นมักช่วยให้ timing closure ง่ายขึ้น

---

## 66. Experiment 2: เปลี่ยน Clock Period เป็น 5 ns

แก้ไข

```yaml
CLOCK_PERIOD: 5
```

รันใหม่

```bash
librelane --pdk ihp-sg13g2 config.yaml
```

Clock period 5 ns เท่ากับ

```text
200 MHz
```

ตรวจสอบว่า

- Timing ยังผ่านหรือไม่
- จำนวน buffer เพิ่มขึ้นหรือไม่
- เครื่องมือเลือกเซลล์ขนาดใหญ่ขึ้นหรือไม่
- พื้นที่เพิ่มขึ้นหรือไม่
- Routing มีความซับซ้อนขึ้นหรือไม่

หลังทดลองให้คืนไฟล์

```bash
cp config_10ns.yaml config.yaml
```

---

## 67. Experiment 3: เปลี่ยน Counter เป็น 16 บิต

แก้ไขเอาต์พุตจาก

```systemverilog
output logic [7:0] count_o
```

เป็น

```systemverilog
output logic [15:0] count_o
```

ส่วนคำสั่งเพิ่มค่ายังคงใช้

```systemverilog
count_o <= count_o + 1;
```

รัน LibreLane ใหม่

```bash
librelane --pdk ihp-sg13g2 config.yaml
```

เปรียบเทียบ

- จำนวน Flip-Flop
- จำนวน combinational cells
- Critical path
- Standard-cell area
- Core utilization
- Routing
- Clock sinks

คาดว่าจำนวน Flip-Flop จะเพิ่มจากประมาณ 8 เป็นประมาณ 16 ตัว และ incrementer จะมีขนาดใหญ่ขึ้น

---

## 68. Experiment 4: เปลี่ยน Reset เป็น Asynchronous Reset

แก้ RTL เป็น

```systemverilog
always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
        count_o <= '0;
    end else begin
        count_o <= count_o + 1;
    end
end
```

รัน flow ใหม่แล้วตรวจสอบว่า standard-cell mapping เปลี่ยนหรือไม่

ประเด็นสำหรับอภิปราย:

1. Library มี Flip-Flop แบบ asynchronous reset หรือไม่
2. Tool ใช้ reset pin ของ Flip-Flop โดยตรงหรือไม่
3. Reset network ถูก buffer อย่างไร
4. Timing checks ที่เกี่ยวข้องกับ reset เปลี่ยนหรือไม่
5. Recovery และ removal checks ปรากฏหรือไม่

---

# ส่วนที่ 10: การแก้ปัญหาเบื้องต้น

## 69. ปัญหา `librelane: command not found`

อาการ

```text
bash: librelane: command not found
```

สาเหตุที่เป็นไปได้

- ยังไม่ได้เข้าสู่ Nix shell
- Environment ไม่ถูก activate
- LibreLane ไม่ได้ติดตั้ง
- PATH ไม่ถูกต้อง

แนวทางแก้ไข

```bash
which librelane
```

หากไม่แสดง path ให้กลับไปเปิด environment ตาม Lab การติดตั้งเครื่องมือ

---

## 70. ปัญหาไม่พบ `config.yaml`

อาการ

```text
No such file or directory: config.yaml
```

ตรวจสอบตำแหน่ง

```bash
pwd
```

ตรวจสอบไฟล์

```bash
ls -la
```

เข้าสู่ไดเรกทอรีให้ถูกต้อง

```bash
cd ~/workshop/heichips26-digital-workshop/exercise_1
```

แล้วรันใหม่

```bash
librelane --pdk ihp-sg13g2 config.yaml
```

---

## 71. ปัญหาไม่พบ RTL

อาการอาจระบุว่าไม่พบ

```text
counter.sv
```

ตรวจสอบ

```bash
ls -l counter.sv
```

ตรวจสอบ `VERILOG_FILES`

```bash
grep VERILOG_FILES config.yaml
```

ค่าที่ถูกต้องคือ

```yaml
VERILOG_FILES: dir::counter.sv
```

Linux แยกตัวพิมพ์เล็กและตัวพิมพ์ใหญ่ ดังนั้น `Counter.sv` และ `counter.sv` ถือเป็นคนละไฟล์

---

## 72. ปัญหาไม่พบ Top Module

อาการอาจมีข้อความว่าไม่พบ module `counter`

ตรวจสอบชื่อ module

```bash
grep -n "module" counter.sv
```

ตรวจสอบ configuration

```bash
grep DESIGN_NAME config.yaml
```

ทั้งสองส่วนต้องตรงกัน

```yaml
DESIGN_NAME: counter
```

```systemverilog
module counter (
```

---

## 73. ปัญหา Syntax Error ที่ `always_ff`

หากเครื่องมืออ่านไฟล์เป็น Verilog แทน SystemVerilog อาจไม่รู้จัก

```systemverilog
logic
always_ff
```

ตรวจสอบว่าไฟล์ใช้นามสกุล

```text
.sv
```

ไม่ใช่

```text
.v
```

และ `VERILOG_FILES` อ้างถึงไฟล์ `.sv` ที่ถูกต้อง

---

## 74. ปัญหาไม่พบ Clock Port

ตรวจสอบ

```yaml
CLOCK_PORT: clk_i
```

และ RTL

```systemverilog
input logic clk_i
```

ชื่อต้องตรงกันทุกตัวอักษร

ตรวจสอบว่า clock port อยู่ที่ top-level module ไม่ใช่เฉพาะ submodule

---

## 75. ปัญหาดาวน์โหลด PDK ไม่สำเร็จ

ตรวจสอบเครือข่าย

```bash
ping -c 3 github.com
```

ตรวจสอบพื้นที่ว่าง

```bash
df -h
```

ตรวจสอบสิทธิ์เขียน home directory

```bash
touch ~/librelane_write_test
rm ~/librelane_write_test
```

ตรวจสอบไดเรกทอรี Ciel

```bash
ls -la ~/.ciel
```

หากไฟล์ PDK ดาวน์โหลดไม่สมบูรณ์ อาจต้องลบเฉพาะ installation ที่เสียและให้ LibreLane ดาวน์โหลดใหม่ตามแนวทางของ environment ที่ใช้ ไม่ควรลบ `~/.ciel` ทั้งหมดโดยไม่ตรวจสอบ เพราะอาจมี PDK หลายชุดที่ใช้งานอยู่

---

## 76. ปัญหา OpenROAD GUI ไม่เปิด

ตรวจสอบ environment variable

```bash
echo "$DISPLAY"
```

กรณีใช้ WSL2 หรือ remote server ต้องมีระบบแสดงผล GUI เช่น

- WSLg
- X11 forwarding
- VNC
- Remote desktop

ทดลองตรวจสอบโปรแกรม GUI อื่น

```bash
klayout
```

หากรันผ่าน SSH ให้เชื่อมต่อด้วย X forwarding ตามการตั้งค่าของระบบ เช่น

```bash
ssh -X user@hostname
```

---

## 77. ปัญหา `--last-run` ไม่พบ Run

อาการเกิดเมื่อยังไม่มี run ที่สำเร็จหรือไม่มี run directory

ตรวจสอบ

```bash
ls runs
```

หรือ

```bash
ls run
```

ต้องรัน Classic flow ก่อน

```bash
librelane --pdk ihp-sg13g2 config.yaml
```

จากนั้นจึงเปิดด้วย

```bash
librelane \
  --pdk ihp-sg13g2 \
  config.yaml \
  --last-run \
  --flow OpenInOpenROAD
```

---

## 78. ปัญหา Timing Violation

ตรวจสอบรายงาน timing ก่อน

```bash
grep -Rni "slack" "$LATEST_RUN" | head -30
```

แนวทางวิเคราะห์ ได้แก่

1. Clock period เข้มงวดเกินไปหรือไม่
2. Critical path ผ่าน logic กี่ระดับ
3. Net delay สูงหรือไม่
4. Slew หรือ capacitance สูงหรือไม่
5. Congestion มีผลต่อ routing หรือไม่
6. Clock uncertainty มีค่ามากหรือไม่

สำหรับการทดลองเบื้องต้นสามารถเพิ่ม clock period เช่น

```yaml
CLOCK_PERIOD: 20
```

แต่ในงานออกแบบจริงควรแก้สาเหตุของ critical path ไม่ใช่เพียงลดความถี่เสมอไป

---

## 79. ปัญหา DRC Violation

ค้นหา DRC report

```bash
find "$LATEST_RUN" -type f -iname "*drc*" -print
```

อ่านรายละเอียด

```bash
grep -Rni "violation" "$LATEST_RUN" | head -50
```

สาเหตุที่เป็นไปได้

- Routing congestion
- Pin placement ไม่เหมาะสม
- PDN geometry
- Via placement
- Minimum spacing
- Minimum area
- Metal density
- Fill-related rule

เปิด marker ใน KLayout หรือ OpenROAD เพื่อระบุตำแหน่ง violation

---

## 80. ปัญหา LVS Mismatch

สาเหตุทั่วไป ได้แก่

- Net ขาด
- Net short
- Power pin ไม่เชื่อมต่อ
- Ground pin ไม่เชื่อมต่อ
- Top-level port ไม่ตรงกัน
- Powered netlist ไม่ตรงกับ layout
- Cell extraction model ไม่ตรงกัน

ค้นหารายงาน

```bash
find "$LATEST_RUN" -type f -iname "*lvs*" -print
```

อ่านคำว่า mismatch

```bash
grep -Rni "mismatch" "$LATEST_RUN" | head -40
```

ควรเริ่มวิเคราะห์จาก net หรือ device แรกที่รายงานว่าต่างกัน เพราะ mismatch รายการหลังอาจเป็นผลต่อเนื่องจากปัญหาแรก

---

# ส่วนที่ 11: คำถามท้ายบทปฏิบัติการ

## 81. คำถามความเข้าใจ

1. `DESIGN_NAME` ต้องสัมพันธ์กับส่วนใดของ RTL
2. `CLOCK_PERIOD: 10` หมายถึงความถี่เท่าใด
3. เหตุใด `rst_ni` จึงเรียกว่า Active-Low Reset
4. Reset ใน RTL นี้เป็น synchronous หรือ asynchronous
5. `always_ff` ต่างจาก `always_comb` อย่างไร
6. เหตุใด Sequential Logic จึงควรใช้ Nonblocking Assignment
7. เคาน์เตอร์ 8 บิตนับได้สูงสุดเท่าใด
8. เมื่อเคาน์เตอร์มีค่า `8'hFF` แล้วเพิ่มอีกหนึ่งจะเกิดอะไรขึ้น
9. Clock Tree Synthesis มีวัตถุประสงค์อะไร
10. DRC และ LVS ตรวจสอบสิ่งเดียวกันหรือไม่
11. OpenROAD GUI และ KLayout เหมาะกับงานต่างกันอย่างไร
12. ไฟล์ `.odb`, `.def` และ `.gds` ต่างกันอย่างไร
13. ค่า WNS ติดลบหมายถึงอะไร
14. เหตุใด layout ของวงจรขนาดเล็กจึงยังมี physical-only cells
15. การผ่าน LVS รับประกันว่า timing ผ่านหรือไม่ เพราะเหตุใด

---

## 82. งานที่ต้องส่ง

ผู้เรียนต้องส่งผลลัพธ์ดังต่อไปนี้

### 82.1 Source files

```text
counter.sv
config.yaml
```

### 82.2 Command log

บันทึกคำสั่งหลักที่ใช้ ได้แก่

```bash
librelane --pdk ihp-sg13g2 config.yaml
```

```bash
librelane \
  --pdk ihp-sg13g2 \
  config.yaml \
  --last-run \
  --flow OpenInOpenROAD
```

```bash
librelane \
  --pdk ihp-sg13g2 \
  config.yaml \
  --last-run \
  --flow OpenInKLayout
```

### 82.3 ภาพผลลัพธ์

อย่างน้อยสี่ภาพ

1. Layout ใน OpenROAD GUI
2. Clock Tree Viewer
3. Timing Report หรือ critical path
4. Layout ใน KLayout

### 82.4 ตารางสรุปผล

| รายการ | ค่าที่วัดได้ |
|---|---|
| Design name | |
| PDK | |
| Clock port | |
| Clock period | |
| Target frequency | |
| Number of sequential cells | |
| Number of combinational cells | |
| Number of clock buffers | |
| Die area | |
| Core area | |
| Standard-cell area | |
| Core utilization | |
| Setup WNS | |
| Hold WNS | |
| TNS | |
| DRC violations | |
| LVS result | |
| Antenna result | |
| GDSII file | |

### 82.5 บทวิเคราะห์

เขียนบทวิเคราะห์ความยาวประมาณหนึ่งถึงสองหน้า โดยอธิบาย

- RTL ถูกแปลงเป็นฮาร์ดแวร์อย่างไร
- Clock tree มีโครงสร้างอย่างไร
- Critical path อยู่บริเวณใด
- Timing ผ่านหรือไม่
- Physical verification ผ่านหรือไม่
- สิ่งที่เรียนรู้จาก OpenROAD และ KLayout
- ปัญหาที่พบและวิธีแก้ไข

---

# ส่วนที่ 12: Checklist ก่อนจบ Lab

ตรวจสอบรายการต่อไปนี้

```text
[ ] อยู่ในไดเรกทอรี exercise_1
[ ] พบไฟล์ counter.sv
[ ] พบไฟล์ config.yaml
[ ] LibreLane ทำงานได้
[ ] เลือก PDK ihp-sg13g2
[ ] Synthesis สำเร็จ
[ ] Floorplan สำเร็จ
[ ] Placement สำเร็จ
[ ] CTS สำเร็จ
[ ] Routing สำเร็จ
[ ] Timing report ถูกสร้าง
[ ] DRC ผ่าน
[ ] LVS ผ่าน
[ ] Antenna check ผ่าน
[ ] พบไฟล์ ODB
[ ] พบไฟล์ DEF
[ ] พบไฟล์ GDSII
[ ] เปิด design ใน OpenROAD GUI ได้
[ ] เปิด design ใน KLayout ได้
[ ] ตรวจสอบ Clock Tree แล้ว
[ ] ตรวจสอบ Critical Path แล้ว
[ ] บันทึกผลลงในตารางแล้ว
```

---

# 13. สรุป

บทปฏิบัติการนี้แสดงกระบวนการพื้นฐานของการนำ RTL ไปเป็น Physical Layout ด้วย LibreLane โดยเริ่มจากวงจรเคาน์เตอร์ 8 บิตที่มีโครงสร้างไม่ซับซ้อน

แม้ตัว RTL จะมีเพียงไม่กี่บรรทัด แต่การสร้างชิปจริงต้องผ่านกระบวนการหลายขั้นตอน ได้แก่

```text
RTL
→ Synthesis
→ Floorplanning
→ Placement
→ Clock Tree Synthesis
→ Routing
→ Timing Analysis
→ Physical Verification
→ GDSII
```

ประเด็นสำคัญที่ผู้เรียนควรเข้าใจคือ LibreLane ไม่ได้เป็นเพียงโปรแกรมแปลงไฟล์ แต่เป็นระบบ orchestration ที่ควบคุมเครื่องมือ EDA หลายตัวให้ทำงานร่วมกันผ่าน state และ artifacts ของแต่ละขั้นตอน

ไดเรกทอรี run จึงเป็นแหล่งข้อมูลสำคัญสำหรับการเรียนรู้และ debug เพราะภายในมีทั้ง

- Input state
- Output state
- Netlist
- DEF
- ODB
- GDSII
- Timing reports
- DRC reports
- LVS reports
- Warning และ error logs

ความสามารถในการอ่านไฟล์เหล่านี้และเชื่อมโยงกลับไปยัง RTL, constraints และ physical layout เป็นพื้นฐานสำคัญของการออกแบบ ASIC ในบทปฏิบัติการขั้นต่อไป
