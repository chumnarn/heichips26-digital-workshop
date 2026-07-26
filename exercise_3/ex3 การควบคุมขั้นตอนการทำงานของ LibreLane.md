
# Lab 3 
# การควบคุมขั้นตอนการทำงานของ LibreLane  
## Controlling the LibreLane Flow

---

## 1. บทนำ

กระบวนการออกแบบวงจรรวมแบบดิจิทัลจาก RTL ไปจนถึง GDSII ประกอบด้วยขั้นตอนจำนวนมาก เช่น

1. ตรวจสอบไวยากรณ์ RTL
2. สังเคราะห์วงจร
3. สร้าง Floorplan
4. วาง Power Distribution Network
5. วาง Standard Cell
6. สร้าง Clock Tree
7. เดินสายสัญญาณ
8. วิเคราะห์ Timing
9. ตรวจสอบ DRC
10. ตรวจสอบ LVS
11. สร้างไฟล์ GDSII

ในงานออกแบบขนาดเล็ก การรันกระบวนการทั้งหมดตั้งแต่ต้นจนจบทุกครั้งอาจไม่สร้างปัญหามากนัก แต่สำหรับวงจรขนาดใหญ่ การรัน Full Flow อาจใช้เวลานาน หากพบข้อผิดพลาดเฉพาะในขั้น Synthesis หรือ Placement การเริ่มใหม่ทั้งหมดจะทำให้เสียเวลาโดยไม่จำเป็น

LibreLane จึงมีความสามารถในการควบคุมลำดับการทำงานของ Flow ได้แก่

- รันจนถึงขั้นตอนที่กำหนดด้วย `--to`
- เริ่มรันจากขั้นตอนที่กำหนดด้วย `--from`
- ใช้ผลลัพธ์จาก Run ก่อนหน้าด้วย `--last-run`
- ระบุสถานะเริ่มต้นด้วย `--with-initial-state`
- ข้ามขั้นตอนที่ไม่ต้องการด้วย `--skip`
- เปิดผลลัพธ์ด้วย OpenROAD GUI

Exercise นี้ใช้วงจร Shift Register ขนาด 256 บิตเป็นกรณีศึกษา เพื่อให้เห็นความสัมพันธ์ระหว่างโครงสร้าง RTL กับลักษณะการจัดวางเซลล์ใน Physical Design อย่างชัดเจน citeturn212292view0turn763393view0turn763393view1

---

## 2. วัตถุประสงค์

หลังจากจบ Exercise นี้ ผู้เรียนจะสามารถ

1. อธิบายโครงสร้างของ Sequential Flow ใน LibreLane ได้
2. ตรวจสอบชื่อ Step ที่ใช้ใน Classic Flow ได้
3. สั่งรัน Flow จนถึง Step ที่ต้องการได้
4. เริ่ม Flow ต่อจากผลลัพธ์เดิมได้
5. ระบุ State ที่ต้องใช้เป็นอินพุตของ Step ได้
6. จำกัดช่วงการรันด้วย `--from` และ `--to` พร้อมกันได้
7. ข้าม Step ที่เลือกด้วย `--skip` ได้
8. เปิดฐานข้อมูล Physical Design ด้วย OpenROAD GUI ได้
9. เปรียบเทียบ Global Placement กับ Detailed Placement ได้
10. วิเคราะห์ความเสี่ยงของการข้าม Physical Verification ได้

---

## 3. ความรู้พื้นฐานที่ควรมี

ผู้เรียนควรมีพื้นฐานเรื่องต่อไปนี้

- SystemVerilog RTL
- Flip-flop และ Shift Register
- Logic Synthesis
- Floorplanning
- Placement
- Routing
- Static Timing Analysis
- DRC และ LVS
- การใช้ Terminal บน Linux
- โครงสร้างไฟล์ YAML
- การใช้งาน LibreLane ขั้นพื้นฐาน

---

## 4. โครงสร้างไฟล์ของ Exercise

เข้าสู่ Repository และตรวจสอบโครงสร้างของ `exercise_3`

```text
exercise_3/
├── README.md
├── config.yaml
├── shift_register.sv
└── img/
```

หน้าที่ของแต่ละไฟล์มีดังนี้

| ไฟล์ | หน้าที่ |
|---|---|
| `README.md` | คำสั่งและโจทย์หลักของ Exercise |
| `config.yaml` | Configuration สำหรับ LibreLane |
| `shift_register.sv` | RTL ของ Shift Register ขนาด 256 บิต |
| `img/` | ภาพประกอบผลลัพธ์จากเครื่องมือ |

Repository ต้นฉบับระบุว่า Exercise นี้มุ่งสาธิตการควบคุม Flow ด้วย `--to`, `--from` และ `--skip` โดยตรง citeturn212292view0

---

# ส่วนที่ 1 เตรียมสภาพแวดล้อม

## 5. ดาวน์โหลด Repository

เปิด Terminal และดาวน์โหลด Repository

```bash
git clone https://github.com/chumnarn/heichips26-digital-workshop.git
```

เข้าสู่โฟลเดอร์ Exercise

```bash
cd heichips26-digital-workshop/exercise_3
```

ตรวจสอบตำแหน่งปัจจุบัน

```bash
pwd
```

ตรวจสอบไฟล์

```bash
ls -la
```

ผลลัพธ์ควรพบอย่างน้อย

```text
README.md
config.yaml
shift_register.sv
img
```

---

## 6. เข้าสู่ LibreLane Environment

วิธีเข้าสู่ Environment ขึ้นอยู่กับการติดตั้งของระบบอบรม เช่น Nix หรือ Docker

ตัวอย่างกรณีใช้ Nix

```bash
nix-shell
```

หรือตาม Environment ของ Workshop

ตรวจสอบว่าเรียก LibreLane ได้

```bash
librelane --version
```

ตรวจสอบว่า PDK ถูกติดตั้ง

```bash
librelane --pdk ihp-sg13g2 --version
```

หากคำสั่งแรกแสดงหมายเลขเวอร์ชัน แสดงว่า LibreLane พร้อมใช้งาน

> ควรรันคำสั่งทั้งหมดของ Exercise จากภายในโฟลเดอร์ `exercise_3` เนื่องจาก `config.yaml` อ้างอิงไฟล์ RTL แบบสัมพันธ์กับ Design Directory

---

# ส่วนที่ 2 ศึกษาวงจร Shift Register

## 7. เปิดไฟล์ RTL

แสดงเนื้อหาของไฟล์

```bash
cat shift_register.sv
```

RTL หลักของ Exercise มีลักษณะดังนี้

```systemverilog
module shift_register (
    input  logic clk_i,
    input  logic in_i,
    input  logic rst_ni,
    output logic out_o
);

    localparam BITS = 256;

    logic [BITS-1:0] shift_reg;

    always_ff @(posedge clk_i, negedge rst_ni) begin
        if (!rst_ni) begin
            shift_reg <= '0;
        end else begin
            for (int i = 0; i < BITS; i++) begin
                if (i == 0) begin
                    shift_reg[i] <= in_i;
                end else begin
                    shift_reg[i] <= shift_reg[i-1];
                end
            end
        end
    end

    assign out_o = shift_reg[BITS-1];

endmodule
```

ไฟล์จริงกำหนด `BITS` เท่ากับ 256 และใช้ลูปสร้างพฤติกรรมการเลื่อนข้อมูลผ่านรีจิสเตอร์ทุกบิต citeturn763393view1

---

## 8. วิเคราะห์ Interface ของวงจร

| Signal | ทิศทาง | ความกว้าง | ความหมาย |
|---|---:|---:|---|
| `clk_i` | Input | 1 บิต | Clock |
| `in_i` | Input | 1 บิต | Serial data input |
| `rst_ni` | Input | 1 บิต | Active-low asynchronous reset |
| `out_o` | Output | 1 บิต | Serial data output |

Suffix ของชื่อสัญญาณมีความหมายดังนี้

- `_i` หมายถึง Input
- `_o` หมายถึง Output
- `_n` หมายถึง Active-low
- `rst_ni` จึงหมายถึง Reset ซึ่งเป็น Input และทำงานเมื่อค่าเป็นลอจิก 0

---

## 9. วิเคราะห์พฤติกรรมของ Shift Register

วงจรประกอบด้วยรีจิสเตอร์ 256 บิต

```systemverilog
logic [BITS-1:0] shift_reg;
```

เมื่อเกิดขอบขาขึ้นของ Clock และ Reset ไม่ทำงาน

```systemverilog
shift_reg[0] <= in_i;
shift_reg[1] <= shift_reg[0];
shift_reg[2] <= shift_reg[1];
...
shift_reg[255] <= shift_reg[254];
```

ข้อมูลที่เข้าทาง `in_i` จะปรากฏที่ `out_o` หลังจากผ่านขอบขาขึ้นของ Clock จำนวน 256 ครั้ง

```systemverilog
assign out_o = shift_reg[BITS-1];
```

โครงสร้างเชิงฮาร์ดแวร์จึงเทียบเท่ากับ D Flip-flop จำนวน 256 ตัวต่ออนุกรม

```text
          FF0       FF1       FF2                  FF255
in_i ───► D Q ────► D Q ────► D Q ─── ... ─────► D Q ───► out_o
           ▲          ▲          ▲                    ▲
           └──────────┴──────────┴──── clk_i ─────────┘
```

โครงสร้างลักษณะนี้ทำให้ Placement มีแนวโน้มจัดเซลล์เป็นสายยาว เนื่องจาก Data Path เป็นการเชื่อมต่อแบบต่อเนื่องจาก Flip-flop ตัวหนึ่งไปยังตัวถัดไป

---

## 10. วิเคราะห์ Reset

Sensitivity list คือ

```systemverilog
always_ff @(posedge clk_i, negedge rst_ni)
```

หมายความว่า Block จะทำงานเมื่อ

- `clk_i` เปลี่ยนจาก 0 เป็น 1 หรือ
- `rst_ni` เปลี่ยนจาก 1 เป็น 0

เมื่อ Reset ทำงาน

```systemverilog
if (!rst_ni) begin
    shift_reg <= '0;
end
```

รีจิสเตอร์ทั้งหมดจะถูกกำหนดเป็นศูนย์โดยไม่ต้องรอขอบ Clock

จึงเป็นวงจรแบบ

```text
Asynchronous active-low reset
```

---

## 11. เหตุผลที่ใช้ Nonblocking Assignment

ภายใน Sequential Logic ใช้

```systemverilog
<=
```

แทน

```systemverilog
=
```

Nonblocking assignment ทำให้รีจิสเตอร์ทุกตัวอ่านค่าก่อนขอบ Clock เดียวกัน แล้วอัปเดตค่าพร้อมกันภายหลัง

ตัวอย่าง

```systemverilog
shift_reg[i] <= shift_reg[i-1];
```

หมายถึง Flip-flop ลำดับที่ `i` รับค่าก่อนหน้าใน Flip-flop ลำดับที่ `i-1` ไม่ใช่ค่าที่เพิ่งอัปเดตภายในลูปเดียวกัน

หากใช้ Blocking Assignment อาจทำให้ Simulation แสดงพฤติกรรมคล้ายข้อมูลเคลื่อนผ่านหลาย Stage ภายใน Clock รอบเดียว ซึ่งไม่ตรงกับวงจร Flip-flop จริง

---

# ส่วนที่ 3 ศึกษา LibreLane Configuration

## 12. เปิดไฟล์ `config.yaml`

```bash
cat config.yaml
```

Configuration ของ Exercise คือ

```yaml
DESIGN_NAME: shift_register

VERILOG_FILES: dir::shift_register.sv

CLOCK_PORT: clk_i
CLOCK_PERIOD: 10 # 10ns = 100 MHz

FP_SIZING: absolute
DIE_AREA: [0, 0, 250, 125]

PL_TARGET_DENSITY_PCT: 60

# We need to allow the same number of hold buffers as there are instances (D-FFs)
GRT_RESIZER_HOLD_MAX_BUFFER_PCT: 100
PL_RESIZER_HOLD_MAX_BUFFER_PCT: 100
```

ค่าดังกล่าวตรงกับ Configuration ใน Repository citeturn763393view0

---

## 13. อธิบาย Configuration ทีละรายการ

### 13.1 `DESIGN_NAME`

```yaml
DESIGN_NAME: shift_register
```

กำหนดชื่อ Top-level Module

ค่าต้องตรงกับชื่อ Module ใน RTL

```systemverilog
module shift_register (
```

หากชื่อไม่ตรงกัน LibreLane หรือ Yosys จะไม่สามารถเลือก Top Module ได้อย่างถูกต้อง

---

### 13.2 `VERILOG_FILES`

```yaml
VERILOG_FILES: dir::shift_register.sv
```

กำหนดไฟล์ RTL ที่ต้องนำเข้า

`dir::` หมายถึงให้ตีความ Path โดยอ้างอิงจาก Design Directory ซึ่งเป็นตำแหน่งของ `config.yaml`

แนวทางนี้ช่วยให้ Configuration เคลื่อนย้ายไปยังเครื่องอื่นได้ง่ายกว่าการใช้ Absolute Path

---

### 13.3 `CLOCK_PORT`

```yaml
CLOCK_PORT: clk_i
```

กำหนดชื่อ Clock Port หลักของวงจร

LibreLane ใช้ข้อมูลนี้ในการ

- สร้าง Clock Constraint
- วิเคราะห์ Setup Timing
- วิเคราะห์ Hold Timing
- สร้าง Clock Tree
- ประเมิน Clock Skew
- วิเคราะห์ Timing หลัง Routing

ค่าต้องตรงกับพอร์ตใน RTL

```systemverilog
input logic clk_i
```

---

### 13.4 `CLOCK_PERIOD`

```yaml
CLOCK_PERIOD: 10
```

หน่วยเป็นนาโนวินาที

คำนวณความถี่ได้จาก

$$f = \frac{1}{T}$$

เมื่อ

$$T = 10\text{ ns}$$

ดังนั้น

$$f = \frac{1}{10\times10^{-9}}  = 100\times10^6   = 100\text{ MHz}$$

วงจรจึงถูกออกแบบโดยมีเป้าหมาย Clock 100 MHz

---

### 13.5 `FP_SIZING`

```yaml
FP_SIZING: absolute
```

กำหนดให้ใช้ขนาด Die แบบ Absolute โดยระบุพิกัดตรงใน `DIE_AREA`

หากไม่ใช้โหมด Absolute เครื่องมืออาจคำนวณขนาด Floorplan จากจำนวนเซลล์และค่า Utilization

---

### 13.6 `DIE_AREA`

```yaml
DIE_AREA: [0, 0, 250, 125]
```

รูปแบบคือ

```text
[x_min, y_min, x_max, y_max]
```

ดังนั้น

- ความกว้าง = \(250-0=250\) µm
- ความสูง = \(125-0=125\) µm

พื้นที่ Die คือ

$$A = 250 \times 125   = 31{,}250\ \mu m^2$$

หรือ

$$A = 0.03125\ mm^2$$

---

### 13.7 `PL_TARGET_DENSITY_PCT`

```yaml
PL_TARGET_DENSITY_PCT: 60
```

กำหนด Target Placement Density เท่ากับ 60%

แนวคิดโดยประมาณคือ

$$Density = \frac{\text{พื้นที่ของเซลล์ที่วาง}} {\text{พื้นที่ที่สามารถวางเซลล์ได้}}\times 100$$

การไม่กำหนด Density สูงเกินไปช่วยเหลือพื้นที่สำหรับ

- Cell movement
- Buffer insertion
- Clock-tree cells
- Hold buffers
- Routing channels
- Timing optimization

---

### 13.8 Hold Buffer Limits

```yaml
GRT_RESIZER_HOLD_MAX_BUFFER_PCT: 100
PL_RESIZER_HOLD_MAX_BUFFER_PCT: 100
```

วงจร Shift Register มี Data Path ระหว่าง Flip-flop ที่สั้นมาก

```text
FF Q ──► FF D
```

Data Path ที่สั้นอาจทำให้เกิด Hold Violation ได้ เพราะข้อมูลใหม่อาจเดินทางถึงปลายทางเร็วเกินไป

เครื่องมือจึงอาจต้องแทรก Delay Buffer หรือ Hold Buffer จำนวนมาก การกำหนดค่าสูงถึง 100% เปิดโอกาสให้ Resizer เพิ่ม Buffer ได้ในจำนวนมากเมื่อเทียบกับจำนวน Instance เดิม

ตัวแปรทั้งสองใช้ในช่วงที่ต่างกัน

- `PL_RESIZER_HOLD_MAX_BUFFER_PCT` จำกัดการเพิ่ม Hold Buffer ในช่วง Placement optimization
- `GRT_RESIZER_HOLD_MAX_BUFFER_PCT` จำกัดการเพิ่ม Hold Bufferในช่วงที่สัมพันธ์กับ Global Routing optimization

---

# ส่วนที่ 4 ทำความเข้าใจ Flow, Step และ State

## 14. ความหมายของ Flow

Flow คือชุดของขั้นตอนที่นำข้อมูลการออกแบบจาก Input ไปสร้าง Output ตามลำดับ

```text
RTL
 │
 ▼
Lint
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
CTS
 │
 ▼
Routing
 │
 ▼
Signoff
 │
 ▼
GDSII
```

LibreLane ใช้ `Classic` เป็น Flow มาตรฐานสำหรับงาน RTL-to-GDSII และ Classic Flow มีลักษณะเป็น Sequential Flow กล่าวคือ Step ต่าง ๆ ทำงานเรียงตามลำดับ โดย Output State ของ Step ก่อนหน้าถูกส่งเป็น Input State ของ Step ถัดไป citeturn212292view0turn763393view3

---

## 15. ความหมายของ Step

Step คือหน่วยการทำงานหนึ่งหน่วยใน Flow เช่น

```text
Yosys.Synthesis
OpenROAD.Floorplan
OpenROAD.GlobalPlacement
OpenROAD.DetailedPlacement
OpenROAD.CTS
OpenROAD.GlobalRouting
KLayout.DRC
Netgen.LVS
```

ชื่อ Step ประกอบด้วย

```text
Tool-or-Namespace.StepName
```

ตัวอย่าง

```text
Yosys.Synthesis
```

หมายถึง Step สำหรับ Logic Synthesis ด้วย Yosys

```text
OpenROAD.GlobalPlacement
```

หมายถึง Step สำหรับ Global Placement ด้วย OpenROAD

---

## 16. ความหมายของ State

State คือชุดข้อมูลที่แสดงสถานะปัจจุบันของ Design หลังผ่าน Step ใด Step หนึ่ง

State อาจประกอบด้วยเส้นทางของไฟล์ เช่น

- RTL
- Synthesized netlist
- SDC
- LEF
- DEF
- ODB
- GDS
- SPEF
- Timing reports
- Metrics

รวมถึงค่าตัวชี้วัดของ Design

```text
area
cell count
wire length
setup slack
hold slack
DRC count
```

ในแต่ละ Step มักพบไฟล์สำคัญ

```text
state_in.json
state_out.json
```

ความหมายคือ

```text
state_in.json
    สถานะก่อน Step เริ่มทำงาน

state_out.json
    สถานะหลัง Step ทำงานสำเร็จ
```

LibreLane อธิบาย State โดยแนวคิดว่าเป็นการจับคู่ชนิดของ Design Format กับไฟล์บนดิสก์ รวมถึง Metrics ของการออกแบบ citeturn212292view0

---

# ส่วนที่ 5 ทดลองรันจนถึง Step ที่กำหนด

## 17. รันจนถึงขั้น Synthesis

ใช้คำสั่ง

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --to Yosys.Synthesis
```

หรือรูปแบบย่อ

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  -T Yosys.Synthesis
```

คำสั่งนี้หมายถึง

```text
เริ่มจาก Step แรกของ Classic Flow
และหยุดหลัง Yosys.Synthesis เสร็จสมบูรณ์
```

ตัวเลือก

```text
--to
```

และ

```text
-T
```

มีความหมายเดียวกันในบริบทของ Exercise นี้ citeturn212292view0

---

## 18. สังเกตข้อความใน Terminal

เมื่อ Flow ทำงาน เครื่องมือจะ

1. โหลด `config.yaml`
2. โหลด PDK `ihp-sg13g2`
3. ตรวจสอบ RTL
4. ทำ Synthesis ด้วย Yosys
5. ข้าม Step หลังจาก `Yosys.Synthesis`
6. หยุด Flow

ใน Log ควรเห็น Step หลัง Synthesis ถูกระบุว่า Skipped หรือไม่ถูกเรียกทำงาน

แนวคิดสำคัญคือ `--to` รวม Step เป้าหมายด้วย ดังนั้น `Yosys.Synthesis` จะถูกรัน ไม่ได้หยุดก่อน Step นี้

---

## 19. ตรวจสอบ Run Directory

```bash
ls runs
```

จะพบ Directory ที่มี Timestamp เช่น

```text
runs/RUN_2026-07-20_07-30-00/
```

ชื่อจริงขึ้นอยู่กับเวอร์ชันและเวลาที่รัน

กำหนดตัวแปรเพื่อความสะดวก

```bash
RUN_DIR=$(find runs -mindepth 1 -maxdepth 1 -type d \
  -printf '%T@ %p\n' | sort -nr | head -1 | cut -d' ' -f2-)
```

ตรวจสอบค่า

```bash
echo "$RUN_DIR"
```

ดูโครงสร้าง

```bash
find "$RUN_DIR" -maxdepth 2 -type d | sort
```

ค้นหา Directory ของ Synthesis

```bash
find "$RUN_DIR" -maxdepth 1 -type d \
  -iname '*yosys*synthesis*'
```

---

## 20. ตรวจสอบผลลัพธ์ Synthesis

ค้นหา Netlist

```bash
find "$RUN_DIR" -type f \
  \( -name '*.v' -o -name '*.nl.v' -o -name '*.json' \) \
  | sort
```

ค้นหารายงาน

```bash
find "$RUN_DIR" -type f \
  \( -name '*.rpt' -o -name '*.log' \) \
  | sort
```

ค้นหาจำนวนเซลล์ในรายงาน

```bash
grep -RniE "Number of cells|cell count|DFF|flip-flop" \
  "$RUN_DIR" | head -50
```

คาดว่าผล Synthesis จะมี Flip-flop จำนวนใกล้เคียง 256 ตัว รวมกับเซลล์อื่น เช่น Buffer, Inverter หรือ Logic สำหรับ Reset ทั้งนี้จำนวนจริงขึ้นอยู่กับ Mapping และ Standard-cell Library

---

## 21. แบบฝึกตรวจสอบ Synthesis

ตอบคำถามต่อไปนี้

1. Yosys สร้าง Flip-flop จำนวนเท่าใด
2. Flip-flop ที่เลือกมีชื่อ Standard Cell ใด
3. Reset ถูก Mapping เป็น Flip-flop ที่มี Reset Pin หรือสร้าง Logic เพิ่มเติม
4. มี Buffer หรือ Inverter เกิดขึ้นหรือไม่
5. จำนวน Net ใน Netlist หลัง Synthesis เท่าใด
6. Critical Path อยู่ระหว่างพอร์ตหรือรีจิสเตอร์ใด

---

# ส่วนที่ 6 รันจนถึง Global Placement

## 22. เริ่ม Run ใหม่จนถึง Global Placement

สั่งรัน

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --to OpenROAD.GlobalPlacement
```

หรือ

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  -T OpenROAD.GlobalPlacement
```

Flow จะทำงานตั้งแต่ต้นจนถึง Global Placement

ขั้นตอนก่อนหน้าอาจประกอบด้วย

```text
RTL checking
Synthesis
Timing constraint preparation
Floorplanning
Tap/endcap insertion
PDN generation
Initial placement
Placement optimization
Global placement
```

ลำดับและชื่อ Step ย่อยอาจแตกต่างกันตาม LibreLane Version ดังนั้นไม่ควรยึดหมายเลขลำดับของ Step เป็นค่าตายตัว

---

## 23. ตรวจสอบ Run ล่าสุด

```bash
ls -lt runs
```

หรือตั้งค่า `RUN_DIR` ใหม่

```bash
RUN_DIR=$(find runs -mindepth 1 -maxdepth 1 -type d \
  -printf '%T@ %p\n' | sort -nr | head -1 | cut -d' ' -f2-)
```

```bash
echo "$RUN_DIR"
```

ค้นหา Global Placement Directory

```bash
find "$RUN_DIR" -maxdepth 1 -type d \
  -iname '*openroad*globalplacement*'
```

ตัวอย่างชื่อที่อาจพบ

```text
28-openroad-globalplacement
```

เลข `28` เป็นลำดับ Step ของ Run นั้น ไม่ควรถือว่าเหมือนกันทุกเวอร์ชันหรือทุก Configuration

---

# ส่วนที่ 7 เปิด OpenROAD GUI

## 24. เปิดผลจาก Run ล่าสุด

ใช้คำสั่ง

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --last-run \
  --flow OpenInOpenROAD
```

คำสั่งนี้มีองค์ประกอบสำคัญ

| Argument | ความหมาย |
|---|---|
| `--pdk ihp-sg13g2` | เลือก PDK |
| `config.yaml` | โหลด Design Configuration |
| `--last-run` | ใช้ Run Directory ล่าสุด |
| `--flow OpenInOpenROAD` | เปิดฐานข้อมูลด้วย OpenROAD GUI |

คำสั่งนี้เป็นวิธีที่ Exercise กำหนดให้ใช้เปิดผลหลัง Global Placement citeturn212292view0

---

## 25. สิ่งที่ควรสังเกตใน OpenROAD GUI

เมื่อ GUI เปิดขึ้น ให้ตรวจสอบ

### 25.1 Die Boundary

กรอบ Die ควรมีอัตราส่วนประมาณ

```text
250 µm × 125 µm
```

### 25.2 Standard-cell Rows

จะเห็นแถวสำหรับวาง Standard Cell เรียงในแนวนอน

### 25.3 Placement ของ Flip-flop

เซลล์ของ Shift Register อาจเรียงตัวเป็นเส้นยาวหรือคดไปมา คล้ายงู เนื่องจาก Netlist มีลักษณะเป็นสายโซ่

```text
FF0 → FF1 → FF2 → ... → FF255
```

Placement Engine พยายามวางเซลล์ที่เชื่อมต่อกันใกล้กัน เพื่อลด

- Estimated wire length
- Congestion
- Delay
- Routing complexity

Repository อธิบายว่าลักษณะคล้ายงูเกิดจากวงจรเป็น Shift Register ซึ่งประกอบด้วยสายโซ่ Flip-flop ยาวต่อเนื่อง citeturn212292view0

---

## 26. Global Placement คืออะไร

Global Placement คำนวณตำแหน่งโดยประมาณของ Instance ทุกตัว โดยมุ่งลด Cost Function เช่น

$$Cost = \alpha \cdot Wirelength +\beta \cdot DensityPenalty +\gamma \cdot TimingPenalty +\delta \cdot CongestionPenalty$$

ผล Global Placement อาจยังมีลักษณะดังนี้

- เซลล์ซ้อนทับกันบางส่วน
- เซลล์ยังไม่ตรง Placement Site
- พิกัดอาจยังไม่ Legal
- Orientation อาจยังไม่สมบูรณ์
- เซลล์ยังไม่ถูกจัดลง Standard-cell Row อย่างถูกต้องทั้งหมด

จึงยังไม่ใช่ Placement ที่พร้อมสำหรับ Detailed Routing

---

# ส่วนที่ 8 เริ่ม Flow จาก Step ที่กำหนด

## 27. เหตุผลที่ `--from` เพียงอย่างเดียวไม่พอ

คำสั่งต่อไปนี้ดูเหมือนจะเพียงพอ

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --from OpenROAD.GlobalPlacement
```

แต่ Flow ที่เริ่มจากกลางทางจำเป็นต้องรู้ด้วยว่า Design ก่อนเข้าสู่ Step นั้นมีสถานะอย่างไร

ข้อมูลที่ต้องใช้ เช่น

- Synthesized netlist
- Constraints
- Floorplan database
- PDN
- Placement database
- Metrics จาก Step ก่อนหน้า

ดังนั้นต้องระบุ

1. Run Directory ที่ต้องการใช้
2. Initial State ที่จะป้อนเข้าสู่ Step

Exercise จึงใช้ `--last-run` ร่วมกับ `--with-initial-state` citeturn212292view0

---

## 28. ค้นหา State ของ Global Placement

กำหนด Global Placement Directory โดยไม่ยึดเลข Step

```bash
GP_DIR=$(find "$RUN_DIR" -maxdepth 1 -type d \
  -iname '*openroad*globalplacement*' | head -1)
```

ตรวจสอบ

```bash
echo "$GP_DIR"
```

ดูไฟล์ State

```bash
ls -l "$GP_DIR"/state_*.json
```

ควรพบ

```text
state_in.json
state_out.json
```

---

## 29. เลือก `state_in.json` หรือ `state_out.json`

Exercise กำหนดตัวอย่างการเริ่มที่ Step `OpenROAD.GlobalPlacement` โดยใช้

```text
GlobalPlacement/state_in.json
```

แนวคิดคือ

```text
--from OpenROAD.GlobalPlacement
```

สั่งให้ Global Placement ทำงานอีกครั้ง ดังนั้น Initial State ต้องเป็นสถานะก่อนเข้า Global Placement

```text
state_in.json
```

หากต้องการเริ่มจาก Step ถัดจาก Global Placement โดยไม่รัน Global Placement ซ้ำ หลักการทั่วไปจะเป็นการใช้ Output State ของ Global Placement และกำหนด `--from` เป็น Step ถัดไป อย่างไรก็ตามต้องตรวจสอบชื่อ Step และรูปแบบที่ LibreLane Version นั้นรองรับ

---

## 30. รันจาก Global Placement

คำสั่งรูปแบบเต็ม

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --last-run \
  --from OpenROAD.GlobalPlacement \
  --with-initial-state "$GP_DIR/state_in.json"
```

รูปแบบย่อ

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --last-run \
  -F OpenROAD.GlobalPlacement \
  -i "$GP_DIR/state_in.json"
```

ความหมายของ Argument

| Argument | ความหมาย |
|---|---|
| `--last-run` | ใช้ Run ล่าสุดเป็นบริบท |
| `--from` หรือ `-F` | เริ่ม执行จาก Step ที่ระบุ |
| `--with-initial-state` หรือ `-i` | โหลด State เริ่มต้นจาก JSON |

Repository แสดงทั้งรูปแบบเต็มและรูปแบบย่อของคำสั่งนี้ citeturn212292view0

---

## 31. ผลที่เกิดขึ้นหากไม่กำหนด `--to`

คำสั่งในหัวข้อก่อนหน้าจะเริ่มจาก Global Placement แล้วรันต่อไปจนจบ Flow

```text
Global Placement
      ↓
Detailed Placement
      ↓
CTS
      ↓
Routing
      ↓
Signoff
      ↓
GDSII
```

หากต้องการทดลองเฉพาะ Detailed Placement การปล่อยให้ Flow ทำงานจนจบจะใช้เวลาเกินความจำเป็น

จึงต้องใช้ `--from` และ `--to` ร่วมกัน

---

# ส่วนที่ 9 จำกัดช่วงการรัน

## 32. รันจาก Global Placement ถึง Detailed Placement

ใช้คำสั่ง

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --last-run \
  --from OpenROAD.GlobalPlacement \
  --with-initial-state "$GP_DIR/state_in.json" \
  --to OpenROAD.DetailedPlacement
```

รูปแบบหลายบรรทัดช่วยให้ตรวจสอบ Argument ได้ง่าย

ความหมายคือ

```text
โหลด State ก่อน Global Placement
        ↓
รัน Global Placement
        ↓
รัน Step ระหว่างทางที่จำเป็น
        ↓
รัน Detailed Placement
        ↓
หยุด
```

คำสั่งนี้ตรงกับหลักการใน Exercise ที่ใช้ `--from` และ `--to` เพื่อจำกัดช่วงของ Flow citeturn212292view0

---

## 33. ตรวจสอบ Step ที่ถูกข้าม

ใน Terminal ควรเห็นว่า

- Step ก่อน `OpenROAD.GlobalPlacement` ถูกข้าม
- `OpenROAD.GlobalPlacement` ถูกเรียก
- `OpenROAD.DetailedPlacement` ถูกเรียก
- Step หลัง `OpenROAD.DetailedPlacement` ถูกข้ามหรือไม่ถูกรัน

ตรวจสอบ Directory ที่เพิ่มขึ้น

```bash
find "$RUN_DIR" -maxdepth 1 -type d | sort
```

ค้นหา Detailed Placement

```bash
DP_DIR=$(find "$RUN_DIR" -maxdepth 1 -type d \
  -iname '*openroad*detailedplacement*' | head -1)
```

```bash
echo "$DP_DIR"
```

---

# ส่วนที่ 10 เปรียบเทียบ Global และ Detailed Placement

## 34. เปิดผลหลัง Detailed Placement

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --last-run \
  --flow OpenInOpenROAD
```

ซูมเข้าไปยังบริเวณ Standard Cell แล้วตรวจสอบ

- เซลล์อยู่บน Placement Row หรือไม่
- เซลล์ยังซ้อนทับกันหรือไม่
- Orientation สอดคล้องกับ Row หรือไม่
- ตำแหน่งเซลล์ถูก Snap ลง Grid หรือไม่

---

## 35. Detailed Placement คืออะไร

Detailed Placement หรือ Legalization ทำหน้าที่แปลงตำแหน่งโดยประมาณให้เป็นตำแหน่งที่ถูกต้องตามกฎของ Standard-cell Placement

หน้าที่หลักประกอบด้วย

1. กำจัดการซ้อนทับของเซลล์
2. จัดเซลล์ลง Placement Row
3. Snap พิกัดลง Placement Site
4. ปรับ Orientation ให้เข้ากับ Power Rail
5. รักษาลำดับเซลล์ให้ใกล้เคียง Global Placement
6. ลดการเปลี่ยนแปลง Wire Length ให้มากที่สุด
7. ตรวจสอบความถูกต้องของ Placement

---

## 36. ตารางเปรียบเทียบ Placement

| ประเด็น | Global Placement | Detailed Placement |
|---|---|---|
| จุดประสงค์ | หาตำแหน่งโดยประมาณ | ทำตำแหน่งให้ถูกกฎ |
| Cell overlap | อาจมี | ต้องไม่มี |
| Placement grid | อาจไม่ตรง | ต้องตรง |
| Standard-cell rows | ยังไม่สมบูรณ์ | วางถูก Row |
| ความละเอียด | ระดับภาพรวม | ระดับ Legal Site |
| พร้อมทำ CTS | ยังไม่สมบูรณ์ | พร้อมมากขึ้น |
| พร้อม Routing | ยังไม่พร้อม | เป็นฐานสำหรับขั้นต่อไป |

---

## 37. เหตุผลที่ Shift Register เหมาะกับ Exercise นี้

วงจร Shift Register มีคุณสมบัติที่ช่วยให้เห็นผลของ Placement ได้ชัด

### 37.1 Connectivity เป็นเส้นตรง

```text
FF0 → FF1 → FF2 → ... → FF255
```

### 37.2 Data Path ระหว่าง Stage สั้น

แต่ละ Stage เชื่อมต่อโดยตรงไปยัง Stage ถัดไป

### 37.3 Clock มี Fanout สูง

Clock ขับ Flip-flop ทุกตัว จึงเหมาะกับการศึกษาขั้น CTS ใน Exercise ต่อไป

### 37.4 มีโอกาสเกิด Hold Violation

Data Path ที่สั้นมากทำให้ Hold Timing เป็นประเด็นสำคัญ

### 37.5 ลักษณะ Placement สัมพันธ์กับ RTL โดยตรง

ผู้เรียนสามารถเชื่อมโยง

```text
RTL structure
    ↓
Netlist topology
    ↓
Physical placement pattern
```

ได้อย่างชัดเจน

---

# ส่วนที่ 11 ข้าม Step ที่ไม่ต้องการ

## 38. ข้าม KLayout DRC

ใช้คำสั่ง

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --skip KLayout.DRC
```

รูปแบบย่อ

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  -S KLayout.DRC
```

คำสั่งนี้สั่งให้ Flow รันตามปกติ แต่ไม่เรียก Step `KLayout.DRC` citeturn212292view0

---

## 39. เหตุผลที่บางครั้งต้องข้าม Step

การข้าม Step อาจมีประโยชน์ในช่วง Development เช่น

- ต้องการตรวจสอบว่า Flow ผ่าน Routing ได้หรือไม่
- กำลังแก้ปัญหา Timing
- ยังไม่ต้องการรอ DRC ที่ใช้เวลานาน
- ต้องการสร้าง Intermediate Result อย่างรวดเร็ว
- กำลัง Debug Step อื่นที่ไม่เกี่ยวข้องกับ DRC
- ใช้ Iteration รอบแรกเพื่อประเมิน Area หรือ Congestion

---

## 40. ความเสี่ยงของการข้าม DRC

DRC ย่อมาจาก Design Rule Checking ทำหน้าที่ตรวจสอบว่า Layout สอดคล้องกับกฎกระบวนการผลิตหรือไม่ เช่น

- Minimum width
- Minimum spacing
- Enclosure
- Extension
- Via rules
- Density
- Metal overlap
- Off-grid geometry

หากข้าม `KLayout.DRC` การได้ไฟล์ GDSII ไม่ได้หมายความว่า Layout พร้อมผลิต

ต้องแยกให้ชัดเจนระหว่าง

```text
Flow completed
```

กับ

```text
Design passed signoff
```

การออกแบบที่พร้อมส่งผลิตควรผ่านอย่างน้อย

- DRC
- LVS
- Antenna checks
- Setup timing
- Hold timing
- Power integrity checks ตามระดับ Signoff
- Configuration และ Waiver review

ดังนั้น `--skip KLayout.DRC` ควรใช้เฉพาะการทดลองหรือ Debug ไม่ควรใช้เป็นผล Signoff ขั้นสุดท้าย

---

# ส่วนที่ 12 ตรวจสอบ `state_in.json`

## 41. เปิด State File

แสดงส่วนต้นของไฟล์

```bash
head -80 "$GP_DIR/state_in.json"
```

หรือใช้ `jq`

```bash
jq . "$GP_DIR/state_in.json" | less
```

ตรวจสอบ Key ระดับบน

```bash
jq 'keys' "$GP_DIR/state_in.json"
```

รูปแบบภายในอาจแตกต่างตามเวอร์ชัน แต่โดยแนวคิดจะมีข้อมูลเกี่ยวกับ

- Design views
- File paths
- Metrics
- Metadata

---

## 42. ค้นหาไฟล์ Physical Design ใน State

```bash
grep -nE '\.odb|\.def|\.lef|\.sdc|\.v|\.spef|metrics' \
  "$GP_DIR/state_in.json" | head -50
```

ให้ผู้เรียนพิจารณาว่า ก่อน Global Placement จำเป็นต้องมีข้อมูลใดบ้าง

ตัวอย่าง

```text
Netlist
Floorplan database
Technology LEF
Cell LEF
Timing constraints
Power information
Placement rows
```

---

## 43. State ทำให้ Resume Flow ได้อย่างไร

หากไม่มี State การเริ่มกลาง Flow จะไม่ทราบว่า

- Netlist ปัจจุบันอยู่ที่ใด
- Floorplan ใดต้องโหลด
- Constraints ชุดใดถูกใช้
- Metrics ก่อนหน้าเป็นอย่างไร
- DEF หรือ ODB ใดเป็นฉบับล่าสุด

State จึงทำหน้าที่คล้าย Snapshot หรือ Contract ระหว่าง Step

```text
Step N
  state_in
      │
      ▼
  execute tool
      │
      ▼
  state_out
      │
      ▼
Step N+1
```

สถาปัตยกรรมของ LibreLane แยก Flow และ Step ออกจากกัน โดย Step รับ State เข้าและสร้าง State ออก ซึ่งเป็นพื้นฐานสำคัญของการสร้าง Custom Flow ด้วย Python API citeturn123089search1turn763393view4

---

# ส่วนที่ 13 Workflow ที่แนะนำสำหรับการ Debug

## 44. กรณี Debug Synthesis

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --to Yosys.Synthesis
```

ตรวจสอบ

- Syntax
- Inferred latches
- Undriven nets
- Multiple drivers
- Cell count
- Area
- Timing หลัง Synthesis
- Netlist structure

---

## 45. กรณี Debug Floorplan

กำหนด `--to` เป็น Floorplan Step ที่ตรงกับเวอร์ชันที่ใช้

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --to <Floorplan-Step-Name>
```

จากนั้นเปิด OpenROAD GUI

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --last-run \
  --flow OpenInOpenROAD
```

ตรวจสอบ

- Die area
- Core area
- Standard-cell rows
- Pin positions
- Tap cells
- Endcaps
- PDN

---

## 46. กรณี Debug Placement

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --to OpenROAD.GlobalPlacement
```

ตรวจสอบ Placement แล้วรันเฉพาะช่วงต่อไป

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --last-run \
  --from OpenROAD.GlobalPlacement \
  --with-initial-state "$GP_DIR/state_in.json" \
  --to OpenROAD.DetailedPlacement
```

---

## 47. กรณี Debug Signoff โดยไม่รันใหม่ทั้งหมด

แนวคิดคือ

1. เลือก State ก่อน Signoff Step
2. ใช้ `--last-run`
3. ใช้ `--from`
4. ระบุ Initial State
5. ใช้ `--to` หากต้องการตรวจสอบเพียงบาง Step

ตัวอย่างเชิงแนวคิด

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --last-run \
  --from <Signoff-Step> \
  --with-initial-state <path-to-state_in.json> \
  --to <Target-Step>
```

ต้องใช้ชื่อ Step ที่มีอยู่จริงใน LibreLane Version นั้น

---

# ส่วนที่ 14 ข้อควรระวัง

## 48. อย่ายึดหมายเลข Step Directory แบบตายตัว

ตัวอย่างใน Repository ใช้ Path ลักษณะ

```text
runs/<time_stamp>/28-openroad-globalplacement/state_in.json
```

แต่เลข `28` อาจเปลี่ยนได้จาก

- LibreLane Version
- Classic Flow Version
- PDK
- Configuration
- Step ที่ถูก Enable หรือ Disable
- Plugin
- Custom Flow

ควรค้นหา Directory ด้วยชื่อ

```bash
find "$RUN_DIR" -maxdepth 1 -type d \
  -iname '*openroad*globalplacement*'
```

แทนการเขียนเลขคงที่

---

## 49. ตรวจสอบว่า `--last-run` ชี้ไปยัง Run ที่ถูกต้อง

หากมี Run ใหม่เกิดขึ้นหลังจาก Run ที่ต้องการ `--last-run` อาจเลือก Run คนละชุด

ตรวจสอบด้วย

```bash
ls -lt runs
```

และตรวจสอบ State Path ก่อนรัน

```bash
realpath "$GP_DIR/state_in.json"
```

---

## 50. Configuration ต้องสอดคล้องกับ State

ไม่ควรนำ State จาก Design หรือ Configuration หนึ่งไปใช้กับอีก Design โดยไม่ตรวจสอบ เพราะอาจมีความไม่สอดคล้อง เช่น

- Top module เปลี่ยน
- RTL เปลี่ยน
- Clock เปลี่ยน
- Die area เปลี่ยน
- PDK เปลี่ยน
- Standard-cell library เปลี่ยน
- Pin configuration เปลี่ยน
- Macro configuration เปลี่ยน

หากแก้ไข Configuration ที่มีผลต่อ Step ก่อนหน้า ควรเริ่ม Flow จาก Step ที่ได้รับผลกระทบหรือก่อนหน้านั้น

---

## 51. การเปลี่ยน RTL ต้องรัน Synthesis ใหม่

หากแก้ `shift_register.sv` แล้วเริ่มจาก Global Placement โดยใช้ State เก่า เครื่องมือจะยังใช้ Netlist เดิม

ดังนั้นเมื่อ RTL เปลี่ยน ควรเริ่มจากอย่างช้าที่สุด

```text
Yosys.Synthesis
```

และในหลายกรณีควรสร้าง Run ใหม่ตั้งแต่ต้นเพื่อป้องกัน State ไม่สอดคล้องกัน

---

# ส่วนที่ 15 Troubleshooting

## 52. Error: ไม่พบ `config.yaml`

อาการ

```text
No such file or directory: config.yaml
```

ตรวจสอบ

```bash
pwd
ls -l config.yaml
```

แก้ไขโดยเข้าสู่ Directory ที่ถูกต้อง

```bash
cd heichips26-digital-workshop/exercise_3
```

---

## 53. Error: ไม่พบ PDK

อาการอาจมีข้อความเกี่ยวกับ

```text
ihp-sg13g2
PDK not found
```

ตรวจสอบ Environment และ PDK Installation

```bash
librelane --version
```

ตรวจสอบว่ากำลังอยู่ใน Nix Shell หรือ Container ที่ Workshop เตรียมไว้

---

## 54. Error: ไม่พบ Step

อาการ

```text
Step OpenROAD.GlobalPlacement not found
```

สาเหตุที่เป็นไปได้

- ใช้ชื่อผิด
- ตัวพิมพ์ใหญ่และเล็กไม่ตรง
- LibreLane Version ต่างจาก Exercise
- ใช้ Flow ที่ไม่มี Step นี้

ควรตรวจสอบรายการ Step ของ Classic Flow จากเอกสารหรือ Source ของเวอร์ชันที่ติดตั้ง เนื่องจาก Classic Flow ประกาศชุด Step ไว้เป็นลำดับภายใน Flow citeturn763393view3turn123089search3

---

## 55. Error: ไม่พบ `state_in.json`

ตรวจสอบตัวแปร

```bash
echo "$RUN_DIR"
echo "$GP_DIR"
```

ค้นหาใหม่

```bash
find runs -type f -name state_in.json | sort
```

ตรวจสอบว่า Global Placement เคยทำงานสำเร็จแล้ว

---

## 56. Error: State ไม่ตรงกับ Step

หากเลือก State ผิดช่วง เครื่องมืออาจแจ้งว่า Design Format ที่ต้องการไม่มีอยู่

หลักการเลือกคือ

```text
ต้องการรัน Step X ซ้ำ
    ใช้ state_in.json ของ Step X
```

ตัวอย่าง

```text
ต้องการรัน OpenROAD.GlobalPlacement
    ใช้ GlobalPlacement/state_in.json
```

---

## 57. OpenROAD GUI ไม่เปิด

สาเหตุที่เป็นไปได้

- ไม่มี Graphical Display
- SSH ไม่ได้ Forward X11
- WSLg ไม่พร้อม
- Container ไม่ได้ส่งผ่าน Display
- Run ยังไม่มีฐานข้อมูล OpenROAD

ตรวจสอบ

```bash
echo "$DISPLAY"
```

กรณี SSH อาจต้องใช้ X11 forwarding ตามนโยบายของระบบ

```bash
ssh -X user@host
```

---

## 58. Flow ใช้ Run ผิดชุด

ตรวจสอบรายการ Run

```bash
find runs -mindepth 1 -maxdepth 1 -type d \
  -printf '%TY-%Tm-%Td %TH:%TM:%TS %p\n' | sort
```

ไม่ควรสมมติว่า Run ล่าสุดเป็น Run ที่ต้องการเสมอ โดยเฉพาะเมื่อเปิดหลาย Terminal พร้อมกัน

---

## 59. Hold Violation จำนวนมาก

Shift Register มีเส้นทางรีจิสเตอร์ต่อรีจิสเตอร์ที่สั้นมาก จึงอาจพบ Hold Violation

ตรวจสอบ

- Hold timing report
- จำนวน Hold Buffer
- Placement density
- Clock skew
- Min-delay paths
- ค่า Buffer limit ใน Configuration

Configuration ของ Exercise จึงเปิดเพดาน Hold Buffer ไว้สูงถึง 100% citeturn763393view0

---

# ส่วนที่ 16 แบบฝึกปฏิบัติ

## 60. แบบฝึกที่ 1: เปลี่ยนความยาว Shift Register

แก้ไข

```systemverilog
localparam BITS = 256;
```

เป็น

```systemverilog
localparam BITS = 64;
```

รันจนถึง Synthesis

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --to Yosys.Synthesis
```

เปรียบเทียบ

- จำนวน Flip-flop
- Cell area
- Net count
- Runtime
- Timing

จากนั้นทดลองค่า

```text
BITS = 64
BITS = 128
BITS = 256
BITS = 512
```

---

## 61. แบบฝึกที่ 2: เปรียบเทียบ Placement

สำหรับแต่ละค่า `BITS`

1. รันถึง Global Placement
2. เปิด OpenROAD GUI
3. บันทึกลักษณะการจัดวาง
4. เปรียบเทียบ Wire Length
5. เปรียบเทียบ Congestion
6. เปรียบเทียบจำนวน Hold Buffer

ตั้งสมมติฐานก่อนทดลองว่า Placement จะเปลี่ยนอย่างไรเมื่อจำนวน Stage เพิ่มขึ้น

---

## 62. แบบฝึกที่ 3: เปลี่ยน Die Area

ทดลอง

```yaml
DIE_AREA: [0, 0, 200, 100]
```

และ

```yaml
DIE_AREA: [0, 0, 400, 200]
```

สังเกตผลต่อ

- Placement density
- Cell spreading
- Estimated wire length
- Congestion
- Timing
- Runtime

อภิปรายว่า Die ที่ใหญ่ขึ้นไม่ได้ทำให้ Timing ดีขึ้นเสมอ เพราะระยะห่างระหว่างเซลล์อาจเพิ่มขึ้น

---

## 63. แบบฝึกที่ 4: เปลี่ยน Placement Density

ทดลอง

```yaml
PL_TARGET_DENSITY_PCT: 40
```

```yaml
PL_TARGET_DENSITY_PCT: 60
```

```yaml
PL_TARGET_DENSITY_PCT: 75
```

เปรียบเทียบ

- Cell distribution
- Congestion
- Legalization
- Buffer insertion
- Wire length
- Timing

---

## 64. แบบฝึกที่ 5: ตรวจสอบ State

ใช้

```bash
jq . "$GP_DIR/state_in.json" > state_in_pretty.json
```

ค้นหา

```bash
grep -nE 'netlist|odb|def|sdc|metric|area|slack' \
  state_in_pretty.json
```

ตอบคำถาม

1. State อ้างอิง Netlist ไฟล์ใด
2. State อ้างอิง Physical Database ไฟล์ใด
3. มี Timing Constraint หรือไม่
4. มี Metrics ใดบ้าง
5. Path เป็น Absolute หรือ Relative

---

## 65. แบบฝึกที่ 6: ข้าม DRC แล้วตรวจสอบ Deliverables

รัน

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --skip KLayout.DRC
```

จากนั้นตรวจสอบ

- Flow จบหรือไม่
- มี GDSII หรือไม่
- มีรายงาน KLayout DRC หรือไม่
- Summary ระบุว่า Step ถูกข้ามหรือไม่
- Design สามารถถือว่า Signoff ได้หรือไม่

คำตอบสำคัญคือ แม้มี GDSII ก็ยังไม่ควรถือว่า Design ผ่าน Physical Verification

---

# ส่วนที่ 17 คำถามทบทวน

## 66. คำถามเชิงแนวคิด

1. เพราะเหตุใดการรัน Full Flow ทุกครั้งจึงไม่มีประสิทธิภาพในการ Debug
2. `--to` รวม Step ที่ระบุหรือหยุดก่อน Step นั้น
3. เหตุใด `--from` จึงต้องใช้ State
4. `state_in.json` ต่างจาก `state_out.json` อย่างไร
5. เหตุใดไม่ควรยึดเลขลำดับ Directory ของ Step
6. Global Placement ต่างจาก Detailed Placement อย่างไร
7. เหตุใด Placement ของ Shift Register จึงคล้ายสายโซ่
8. เหตุใด Shift Register จึงมีโอกาสเกิด Hold Violation
9. การข้าม KLayout DRC มีประโยชน์ในกรณีใด
10. เหตุใด Flow Complete จึงไม่เท่ากับ Signoff Complete
11. เมื่อแก้ RTL ควรเริ่ม Flow ใหม่จาก Step ใด
12. เมื่อแก้เฉพาะ Routing Parameter ควรเริ่ม Flow จากช่วงใด
13. State ช่วยลดเวลา Iteration อย่างไร
14. การใช้ State เก่ากับ Configuration ใหม่มีความเสี่ยงอย่างไร
15. Custom Flow แตกต่างจากการใช้ `--from`, `--to` และ `--skip` อย่างไร

---

# ส่วนที่ 18 สรุปคำสั่ง

## 67. Command Cheat Sheet

### รันถึง Synthesis

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --to Yosys.Synthesis
```

หรือ

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  -T Yosys.Synthesis
```

### รันถึง Global Placement

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --to OpenROAD.GlobalPlacement
```

### เปิด OpenROAD GUI

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --last-run \
  --flow OpenInOpenROAD
```

### ค้นหา Run ล่าสุด

```bash
RUN_DIR=$(find runs -mindepth 1 -maxdepth 1 -type d \
  -printf '%T@ %p\n' | sort -nr | head -1 | cut -d' ' -f2-)
```

### ค้นหา Global Placement Directory

```bash
GP_DIR=$(find "$RUN_DIR" -maxdepth 1 -type d \
  -iname '*openroad*globalplacement*' | head -1)
```

### รันจาก Global Placement

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --last-run \
  --from OpenROAD.GlobalPlacement \
  --with-initial-state "$GP_DIR/state_in.json"
```

### รันเฉพาะ Global Placement ถึง Detailed Placement

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --last-run \
  --from OpenROAD.GlobalPlacement \
  --with-initial-state "$GP_DIR/state_in.json" \
  --to OpenROAD.DetailedPlacement
```

### ข้าม KLayout DRC

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --skip KLayout.DRC
```

หรือ

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  -S KLayout.DRC
```

---

# ส่วนที่ 19 Checklist ผลการทดลอง

## 68. Checklist สำหรับผู้เรียน

### Environment

- [ ] ดาวน์โหลด Repository สำเร็จ
- [ ] เข้าสู่ `exercise_3`
- [ ] เรียก `librelane --version` ได้
- [ ] ใช้ PDK `ihp-sg13g2` ได้

### RTL และ Configuration

- [ ] เข้าใจโครงสร้าง Shift Register 256 บิต
- [ ] ระบุ Clock และ Reset ได้
- [ ] อธิบาย `CLOCK_PERIOD: 10` ได้
- [ ] คำนวณขนาด Die ได้
- [ ] อธิบายเหตุผลของ Hold Buffer Limit ได้

### Flow Control

- [ ] รันถึง `Yosys.Synthesis` ได้
- [ ] รันถึง `OpenROAD.GlobalPlacement` ได้
- [ ] เปิด OpenROAD GUI ได้
- [ ] ค้นหา `state_in.json` ได้
- [ ] เริ่ม Flow ด้วย `--from` ได้
- [ ] จำกัด Flow ด้วย `--from` และ `--to` ได้
- [ ] ข้าม `KLayout.DRC` ได้

### Analysis

- [ ] เปรียบเทียบ Global กับ Detailed Placement ได้
- [ ] อธิบายรูปแบบ Placement ของ Shift Register ได้
- [ ] อธิบายความเสี่ยงของการข้าม DRC ได้
- [ ] อธิบายความสัมพันธ์ระหว่าง Step และ State ได้

---

# 69. สรุป

Exercise นี้แสดงให้เห็นว่า LibreLane ไม่จำเป็นต้องรันกระบวนการ RTL-to-GDSII ทั้งหมดทุกครั้ง ผู้ใช้งานสามารถควบคุมช่วงการทำงานได้อย่างละเอียดด้วย

```text
--to
--from
--last-run
--with-initial-state
--skip
```

แนวคิดสำคัญที่สุดคือทุก Step ไม่ได้ทำงานอย่างโดดเดี่ยว แต่รับ State จาก Step ก่อนหน้าและสร้าง State ใหม่สำหรับ Step ถัดไป

```text
Configuration + State In
          │
          ▼
        Step
          │
          ▼
       State Out
```

เมื่อเข้าใจกลไกนี้ ผู้ออกแบบจะสามารถ

- ลดเวลา Iteration
- Debug เฉพาะช่วง
- เปรียบเทียบ Configuration
- ตรวจสอบ Intermediate Results
- พัฒนา Custom Flow
- ใช้ทรัพยากรคอมพิวเตอร์ได้มีประสิทธิภาพมากขึ้น

อย่างไรก็ตาม การข้าม Step โดยเฉพาะ Physical Verification ต้องทำด้วยความระมัดระวัง ผลลัพธ์ที่ Flow สร้าง GDSII สำเร็จไม่ได้รับประกันว่า Layout ถูกต้องตามกฎการผลิต การ Signoff ขั้นสุดท้ายยังต้องผ่าน DRC, LVS, Timing และการตรวจสอบที่เกี่ยวข้องทั้งหมด
