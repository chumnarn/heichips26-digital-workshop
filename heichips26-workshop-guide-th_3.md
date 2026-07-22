# คู่มือ Workshop: HeiChips 2026 LibreLane

**เวอร์ชัน:** 1.0  
**อัปเดตล่าสุด:** กรกฎาคม 2026  
**ระยะเวลา:** ประมาณ 4–6 ชั่วโมง  
**ระดับ:** ปานกลาง (Intermediate)  
**ผู้จัดทำ:** HeiChips 2026, Heidelberg University  
**ลิขสิทธิ์:** CC-BY-SA-4.0 Leo Moser / Apache 2.0 (ส่วน source code)

---

## ภาพรวม Workshop

Workshop นี้สอนให้ผู้เข้าร่วมออกแบบและ implement วงจรดิจิทัลด้วย **LibreLane** บน open-source PDK ของ IHP (กระบวนการผลิต SG13G2 BiCMOS 130nm) ซึ่งเป็น PDK เดียวกับที่ใช้ใน HeiChips 2026 Tapeout จริง

**หลังจากจบ Workshop นี้ ผู้เรียนจะสามารถ:**

- รัน LibreLane flow สำหรับออกแบบ digital circuit ตั้งแต่ต้นจนจบ
- ปรับแต่ง configuration variables เพื่อควบคุมพื้นที่, density และการวาง pin
- ควบคุม flow ให้รันเฉพาะบาง step เพื่อเพิ่มความเร็วในการ debug
- สร้างและรวม macro เข้ากับ top-level design
- ใช้ LibreLane Python API เพื่อสร้าง custom flow และ step
- implement full chip design ที่พร้อมส่ง tapeout

---

## ข้อกำหนดเบื้องต้น

### ความรู้ที่ต้องมี
- ความเข้าใจพื้นฐานเกี่ยวกับ digital logic (flip-flop, counter, register)
- เคยเขียน SystemVerilog หรือ Verilog มาบ้าง
- ใช้ command line (terminal / bash) ได้
- Python พื้นฐาน (สำหรับ Exercise 5)

### ซอฟต์แวร์และเครื่องมือ
- **Nix** — package manager สำหรับติดตั้ง toolchain (ติดตั้งให้แล้วใน HeiChips VM)
- **LibreLane** — ASIC implementation flow (จัดการผ่าน Nix)
- **OpenROAD** — physical design tool พร้อม GUI
- **KLayout** — layout viewer และ DRC/LVS tool
- **Yosys** — synthesis tool

### สภาพแวดล้อม
- ใช้ **HeiChips VM** ที่จัดเตรียมให้ หรือติดตั้ง Nix เองตาม [คู่มือ LibreLane](https://librelane.readthedocs.io/en/latest/getting_started/common/nix_installation/index.html)

---

## การตั้งค่าสภาพแวดล้อม

### ขั้นตอนการติดตั้ง

1. เปิด terminal และ clone repository ของ workshop:
   ```bash
   git clone https://github.com/HeiChips/heichips26-digital-workshop.git
   cd heichips26-digital-workshop
   ```

2. เปิด Nix shell เพื่อโหลด toolchain ทั้งหมด:
   ```bash
   nix-shell
   ```

   > 📝 **หมายเหตุ:** ต้องรันคำสั่งนี้ทุกครั้งที่เปิด terminal ใหม่

3. รอให้ Nix ดาวน์โหลดและติดตั้ง dependencies (ครั้งแรกอาจใช้เวลาสักครู่)

### ตรวจสอบว่าสภาพแวดล้อมพร้อมใช้งาน

```bash
librelane --version
```

> ✅ **จุดตรวจสอบ:** ถ้าเห็นหมายเลขเวอร์ชันของ LibreLane แสดงว่าพร้อมแล้ว

---

## Lab 1: สร้าง Counter เป็นครั้งแรก

**เวลาโดยประมาณ:** 45–60 นาที  
**Objective:** รัน LibreLane Classic flow ครั้งแรกบน 8-bit counter, ทำความเข้าใจแต่ละ step ของ flow, ดู layout ใน OpenROAD GUI และ KLayout, และสำรวจโครงสร้างของ run directory

---

### พื้นหลัง: LibreLane คืออะไร?

**LibreLane** คือ open-source ASIC implementation flow ที่แปลง RTL (Register Transfer Level) — โค้ด SystemVerilog/Verilog ที่เราเขียน — ไปเป็น **GDS** (Graphic Design System) ซึ่งเป็นไฟล์ layout ที่ส่งให้โรงงานผลิตชิปได้จริง

กระบวนการนี้เรียกว่า **RTL-to-GDS** หรือ **Place & Route (PnR)** ประกอบด้วย step หลัก ๆ ดังนี้:

```
RTL (counter.sv)
      │
      ▼  Yosys.Synthesis
  Gate-level Netlist  ← แปลงเป็น AND/OR/FF จริง ๆ
      │
      ▼  OpenROAD.Floorplan
  กำหนดขนาด Die Area + วาง Pin รอบขอบ
      │
      ▼  OpenROAD.Placement (Global → Detailed)
  วาง Standard Cell ลงใน Die Area
      │
      ▼  OpenROAD.CTS
  สร้าง Clock Tree ให้ clock ถึง FF ทุกตัวพร้อมกัน
      │
      ▼  OpenROAD.Routing (Global → Detailed)
  เดินสายโลหะเชื่อมต่อ cell ทั้งหมด
      │
      ▼  KLayout.DRC + OpenROAD.LVS
  ตรวจสอบ Design Rule และ Layout vs. Schematic
      │
      ▼
    GDS (counter.gds)  ← พร้อมส่งโรงงาน
```

**PDK (Process Design Kit)** คือชุดข้อมูลจากโรงงานผลิตที่บอกว่า:
- standard cell มีหน้าตาอย่างไร (LEF/GDS)
- ขนาดขั้นต่ำของสาย metal, via คืออะไร (Design Rules)
- ความเร็วและกำลังไฟของแต่ละ cell (Liberty files)

Workshop นี้ใช้ **IHP Open PDK (`ihp-sg13g2`)** — PDK จริงของโรงงาน IHP สำหรับกระบวนการผลิต SG13G2 (BiCMOS 130nm) ซึ่งเปิดให้ใช้งานฟรีและเป็น PDK เดียวกับที่ใช้ผลิต HeiChips 2026 chip จริง

> 📝 **หมายเหตุ:** PDK ส่วนใหญ่ในอุตสาหกรรมต้องเซ็น NDA (Non-Disclosure Agreement) กับโรงงานก่อนถึงจะใช้ได้ IHP Open PDK เป็นหนึ่งในไม่กี่ PDK ที่เปิดให้ใช้ได้โดยไม่ต้องมี NDA

---

### ทำความเข้าใจ Source Files

#### ไฟล์ที่ 1: `counter.sv` — Design ของเรา

```systemverilog
// A simple 8-bit counter
module counter (
    input  logic       clk_i,   // Clock input (i = input)
    input  logic       rst_ni,  // Reset แบบ active-low (n = negative, i = input)
    output logic [7:0] count_o  // ผลลัพธ์ 8-bit (o = output)
);

    always_ff @(posedge clk_i) begin
        if (!rst_ni) begin
            count_o <= '0;       // Reset: ล้างค่าเป็น 0 ทุก bit
        end else begin
            count_o <= count_o + 1;  // นับขึ้นทุก rising edge ของ clock
        end
    end

endmodule
```

**อธิบาย naming convention:** suffix `_i` = input port, `_o` = output port, `_n` = active-low signal — เป็น convention ที่นิยมใช้ใน chip design เพื่อให้อ่าน port list แล้วรู้ทิศทางและ polarity ได้ทันที

**วงจรนี้ทำงานอย่างไร:** ทุก rising edge ของ `clk_i` ถ้า `rst_ni = 0` (active) จะ reset เป็น 0, ถ้า `rst_ni = 1` จะนับขึ้น 1 ช่วง 8-bit หมายความว่า counter จะนับ 0→1→2→…→254→255→0→... วนไปเรื่อย ๆ

#### ไฟล์ที่ 2: `config.yaml` — บอก LibreLane ว่าต้องทำอะไร

```yaml
DESIGN_NAME: counter       # ชื่อ top-level module ใน Verilog (ต้องตรงกันทุกตัวอักษร)
VERILOG_FILES: dir::counter.sv  # ไฟล์ source (dir:: = "ใน directory เดียวกับ config นี้")
CLOCK_PORT: clk_i          # ชื่อ port ที่รับ clock — ใช้ทำ CTS และ timing analysis
CLOCK_PERIOD: 10           # คาบ clock เป็น ns → 10ns = 100 MHz
```

> 📝 **หมายเหตุ:** `dir::` เป็น path prefix พิเศษของ LibreLane แปลว่า "directory ที่ไฟล์ config.yaml อยู่" ถ้าใช้ path ปกติจะต้อง specify full path ซึ่งยุ่งยากกว่า

`CLOCK_PERIOD: 10` หมายความว่าเราบอก LibreLane ให้ implement design นี้ให้ทำงานได้ที่ **100 MHz** (= 1/10ns) LibreLane จะใช้ค่านี้ตั้งค่า timing constraints ใน SDC file และตรวจสอบว่า setup/hold time ทุก flip-flop ผ่านที่ความถี่นี้

---

### 1.1 รัน LibreLane Classic Flow

#### ขั้นตอนการรัน

1. ตรวจสอบว่าอยู่ใน `nix-shell` แล้ว (prompt จะแสดงต่างออกไป หรือรันใหม่ได้):
   ```bash
   nix-shell
   ```

2. เปลี่ยน directory ไปที่ exercise 1:
   ```bash
   cd exercise_1
   ```

3. รัน LibreLane:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml
   ```

   ความหมายของ argument แต่ละตัว:
   - `--pdk ihp-sg13g2` — เลือกใช้ IHP Open PDK (ถ้าไม่ระบุจะใช้ `sky130A` ซึ่งเป็น default)
   - `config.yaml` — ไฟล์ configuration ที่บอกว่า design ชื่ออะไร ไฟล์ source อยู่ที่ไหน

#### สิ่งที่เกิดขึ้นระหว่างรัน

LibreLane จะพิมพ์ log ออกมาต่อเนื่องบน terminal สังเกตชื่อ step ที่กำลังทำงาน เช่น:

```
[Step  1/76] Yosys.JsonHeader          ← อ่านข้อมูล design
[Step  5/76] Yosys.Synthesis           ← Synthesize RTL เป็น gate netlist
[Step 13/76] OpenROAD.Floorplan        ← กำหนดขนาด chip
[Step 22/76] OpenROAD.GlobalPlacement  ← วาง cell คร่าว ๆ
[Step 28/76] OpenROAD.DetailedPlacement← จัดวาง cell ให้แม่นยำ
[Step 32/76] OpenROAD.GlobalRouting    ← เดินสายคร่าว ๆ
[Step 36/76] OpenROAD.DetailedRouting  ← เดินสายละเอียด
[Step 60/76] KLayout.DRC               ← ตรวจ Design Rules
[Step 65/76] OpenROAD.LVS             ← ตรวจ Layout vs Schematic
```

> 📝 **หมายเหตุ:** ครั้งแรกที่รัน LibreLane จะดาวน์โหลด PDK ลงใน `~/.ciel` อัตโนมัติ อาจใช้เวลาสักครู่ขึ้นอยู่กับความเร็ว internet ครั้งต่อไปจะเร็วกว่ามากเพราะ cache แล้ว

#### ผลลัพธ์ที่ถูกต้อง

เมื่อ flow ทำงานเสร็จ (ประมาณ 5–15 นาที) จะเห็นสรุปที่ท้าย log:

```
* Antenna
Passed ✅

* LVS
Passed ✅

* DRC
Passed ✅
```

อาจมี warning บ้าง (เช่น unconstrained path) ซึ่งสำหรับ exercise นี้สามารถ ignore ได้

> ✅ **จุดตรวจสอบ:** ต้องเห็น Passed ✅ ทั้ง 3 รายการก่อนไปต่อ ถ้ามี error ให้ดูส่วน "การแก้ไขปัญหา" ด้านล่าง

---

### 1.2 ดู Design ใน OpenROAD GUI

OpenROAD เป็น open-source physical design tool ที่ LibreLane ใช้ทำ floorplan, placement, CTS และ routing มี GUI ที่ช่วยให้เราดูและ debug design ได้

#### เปิด OpenROAD GUI

1. รันคำสั่ง (ใน directory เดิม `exercise_1/`):
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml --last-run --flow OpenInOpenROAD
   ```

   - `--last-run` — บอกให้ใช้ run directory ล่าสุด ไม่ต้องรัน flow ใหม่ตั้งแต่ต้น
   - `--flow OpenInOpenROAD` — เลือก flow พิเศษที่แค่เปิด GUI โดยโหลด state จาก run ที่เสร็จแล้ว

2. หน้าต่าง OpenROAD จะเปิดขึ้น มี 3 ส่วนหลัก:
   - **Display Control** (แผง ซ้าย) — รายการ layer และ object ที่แสดง/ซ่อนได้
   - **Canvas** (ตรงกลาง) — layout ของ design
   - **Inspector** (แผง ขวา) — รายละเอียดของสิ่งที่เลือกใน canvas

#### ทำความเข้าใจ Layout ที่เห็น

สิ่งที่เห็นใน canvas คือ **routed design** ของ 8-bit counter หลังจากผ่านทุก step แล้ว:

- **สีฟ้าอ่อนแนวตั้ง** (TopMetal1) และ **สีชมพูแนวนอน** (TopMetal2) — Power straps สำหรับจ่ายไฟ VDD และ GND ให้ทั่ว chip
- **เส้นสีเขียวแนวนอน** — Signal routing บน Metal3
- **เส้นสีแดงแนวตั้ง** — Signal routing บน Metal2  
- **เส้นสีน้ำเงินเข้ม** — Signal routing บน Metal1 (ใกล้ transistor มากที่สุด)
- **จุดสี่เหลี่ยมเล็ก ๆ** — Standard cells (AND gate, flip-flop, buffer ฯลฯ)
- **เส้นสีเขียวรอบขอบ** — Pin ของ design (clk_i, rst_ni, count_o[7:0])

**IHP SG13G2 มี metal layer ทั้งหมด 7 ชั้น:** Metal1–Metal5 สำหรับ signal routing, TopMetal1 และ TopMetal2 สำหรับ power distribution (ใหญ่กว่าและหนากว่า)

#### สำรวจ Inspector

1. คลิกที่ net หรือ cell ใดก็ได้ใน canvas เพื่อดูข้อมูลใน **Inspector** ทางขวา

2. ทดลองคลิกที่เส้นสีต่าง ๆ — Inspector จะบอกว่าเป็น net ชื่ออะไร ประเภทไหน (signal/power/clock) และเชื่อมต่อกับ pin ไหนบ้าง

3. ทดลองปิด/เปิด layer ใน **Display Control** เพื่อดูแต่ละ metal layer แยกกัน

#### ดู Clock Tree Viewer

Clock Tree Synthesis (CTS) คือ step ที่สร้างโครงสร้างต้นไม้ (tree) ของ buffer เพื่อกระจาย clock signal ให้ถึง flip-flop ทุกตัวในเวลาใกล้เคียงกันที่สุด (minimize clock skew)

1. ไปที่เมนูบน: **Windows → Clock Tree Viewer**

2. กดปุ่ม **Update** ในแผง Clock Tree Viewer

3. จะเห็น tree structure ดังนี้:
   - **สามเหลี่ยมสีแดง** (บนสุด) — Root buffer ที่รับ clock จาก port `clk_i`
   - **สามเหลี่ยมสีน้ำเงิน** — Buffer ระดับกลางที่ clone และกระจาย clock ต่อ
   - **สี่เหลี่ยมสีแดงเล็ก ๆ** (ล่างสุด) — Leaf flip-flop ทั้ง 8 ตัว (หนึ่งตัวต่อ bit ของ counter)

   ตัวเลข `0.00 ns` และ `0.10 ns` ทางซ้ายของ tree คือ **arrival time** ของ clock ที่แต่ละชั้น — ออกแบบให้ flip-flop ทุกตัวได้รับ clock ในเวลาใกล้เคียงกันมากที่สุด

4. สังเกตว่าใน layout (canvas) เส้น clock จะเปลี่ยนสี ทำให้เห็น clock routing ได้ชัดเจน

#### ดู Timing Report

Timing analysis ตรวจสอบว่าแต่ละ data path ระหว่าง flip-flop ทำงานทัน clock period ที่กำหนด (10ns = 100MHz) หรือไม่

1. ไปที่เมนูบน: **Windows → Timing Report**

2. กดปุ่ม **Update**

3. จะเห็นตารางรายการ timing path ทุกเส้นในออกแบบ แต่ละแถวประกอบด้วย:
   - **Capture Clock** — ชื่อ clock (clk_i)
   - **Required** — เวลาที่ data ต้องมาถึงก่อน (= clock period − setup time margin)
   - **Arrival** — เวลาที่ data จริง ๆ มาถึง
   - **Slack** — ส่วนต่าง = Required − Arrival → **ต้องเป็นค่าบวกเสมอ** (path นั้นไม่ violation)

   ตัวอย่างจาก design นี้: Slack ≈ 7.1–7.3 ns หมายความว่า design ของเรา margin เหลือมากถึง 7ns — counter ง่าย ๆ ไม่มีทางเป็นปัญหา timing

4. คลิกที่ path ใดก็ได้ใน timing table — layout จะ highlight เส้นทางนั้นให้เห็น

> 💡 **เคล็ดลับ:** ถ้าต้องการ export layout เป็นภาพความละเอียดสูง พิมพ์คำสั่งนี้ในช่อง **Scripting** ด้านล่าง:
> ```tcl
> save_image counter_layout.png -width 4096
> ```
> สำหรับ clock tree ใช้: `save_clocktree_image clock_tree.png`

---

### 1.3 ดู Design ใน KLayout

KLayout เป็น layout viewer และ editor ที่โหลด GDS file — ไฟล์ geometry จริงที่มีพิกัดของทุก polygon บนทุก metal layer ซึ่งเป็นสิ่งที่ส่งให้โรงงานผลิต

**ความแตกต่างระหว่าง OpenROAD GUI กับ KLayout:**

| | OpenROAD GUI | KLayout |
|---|---|---|
| ไฟล์ที่เปิด | ODB (OpenDB) — database ภายใน | GDS/LEF/DEF — geometry จริง |
| ข้อมูลที่เห็น | Netlist, timing, placement | Layer geometry แบบ physical |
| ใช้สำหรับ | Debug flow, ดู timing | ตรวจ DRC/LVS, ดู layout ก่อนส่งโรงงาน |

#### เปิด KLayout

1. รันคำสั่ง:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml --last-run --flow OpenInKLayout
   ```

2. KLayout จะเปิดขึ้นพร้อม GDS ของ counter โดยอัตโนมัติ มี 3 ส่วน:
   - **Cells** (แผงซ้ายบน) — hierarchy ของ cell ทั้งหมดใน design
   - **Canvas** (ตรงกลาง) — layout geometry แต่ละ layer
   - **Layers** (แผงขวา) — รายการ layer ทั้งหมดของ `ihp-sg13g2`

#### ทำความเข้าใจ Cell Hierarchy

ใน **Cells panel** จะเห็น:
- `counter` — top-level design ของเรา
- `sg13g2_dfrbpq_1`, `sg13g2_and2_1`, `sg13g2_buf_1` ฯลฯ — Standard cells จาก IHP PDK ที่ Yosys เลือกมาใช้

Standard cell เหล่านี้คือ building block ที่โรงงานออกแบบไว้ล่วงหน้า: flip-flop (`dfr` = D Flip-flop with Reset), AND gate (`and2`), buffer (`buf`) เป็นต้น ตัวเลขท้ายชื่อเช่น `_1`, `_2`, `_8` คือ drive strength (ความสามารถในการขับ load)

#### สำรวจ Layer

ใน **Layers panel** ทางขวา จะเห็น layer ทั้งหมดของ IHP SG13G2:
- `Activ.drawing` — บริเวณ active silicon (NMOS/PMOS)
- `GatPoly.drawing` — Gate polysilicon
- `Metal1.drawing` ถึง `TopMetal2.drawing` — สาย metal แต่ละชั้น
- `Via1.drawing` ถึง `TopVia2.drawing` — Via เชื่อมระหว่าง metal layer

คลิก checkbox หน้า layer เพื่อ show/hide และดูว่าแต่ละ layer อยู่ที่ไหนใน design

#### ทดลองใช้งาน

1. **ซูมเข้า:** scroll wheel หรือ `Ctrl+scroll` — ซูมเข้าจนเห็น standard cell แต่ละตัวได้

2. **วัดขนาด:** คลิกปุ่ม **Ruler** ในแถบเครื่องมือ แล้วลากวัดขนาด cell หรือ feature ที่สนใจ ตัวเลขจะแสดงเป็น µm

3. **เลือก cell:** ใช้ปุ่ม **Select** แล้วคลิก cell ใด ๆ เพื่อดูขอบเขตและตำแหน่ง

4. **ดู hierarchy:** คลิกที่ชื่อ cell ใน Cells panel เพื่อ zoom ไปยัง cell นั้นใน canvas

> 💡 **เคล็ดลับ:** กดปุ่ม `F` เพื่อ Fit ทั้ง design เข้าหน้าจอ หรือ `Shift+F` เพื่อ Fit เฉพาะที่เลือก

---

### 1.4 สำรวจโครงสร้างโฟลเดอร์ `run/`

ทุกครั้งที่รัน LibreLane จะสร้างโฟลเดอร์ใหม่ใน `run/` เพื่อเก็บผลลัพธ์ของแต่ละ step ทำให้สามารถย้อนกลับไปดูหรือ debug ได้ทุกเวลา

#### โครงสร้างของ Run Directory

```bash
ls run/
# RUN_2026-07-18_09-30-00   ← timestamp ของการรัน
```

```bash
ls run/RUN_2026-07-18_09-30-00/
```

จะเห็นโฟลเดอร์ตามลำดับ step เช่น:
```
00-flow-start/
05-yosys-synthesis/
13-openroad-floorplan/
22-openroad-globalplacement/
28-openroad-detailedplacement/
32-openroad-globalrouting/
36-openroad-detailedrouting/
60-klayout-drc/
65-openroad-lvs/
...
final/              ← ผลลัพธ์สุดท้าย (GDS, LEF, netlist ฯลฯ)
flow.log            ← log ทั้งหมดของ flow
errors.log          ← เฉพาะ error
warnings.log        ← เฉพาะ warning
```

#### ดูผลลัพธ์ภายใน Step

แต่ละโฟลเดอร์ step จะมีโครงสร้างเดียวกัน:

```bash
ls run/RUN_2026-07-18_09-30-00/13-openroad-floorplan/
```

```
state_in.json    ← state ก่อนรัน step นี้ (input)
state_out.json   ← state หลังรัน step นี้ (output)
counter.odb      ← OpenDB file (database ของ design ณ จุดนี้)
counter.nl.v     ← Unpowered netlist
counter.pnl.v    ← Powered netlist (มี VDD/GND)
run.log          ← log เฉพาะของ step นี้
```

1. ดู state หลัง floorplan:
   ```bash
   cat run/RUN_2026-07-18_09-30-00/13-openroad-floorplan/state_out.json
   ```
   State เป็น JSON ที่ map ชนิดของ file (เช่น `odb`, `nl`, `def`) ไปยัง path ของไฟล์นั้น และมี metrics ต่าง ๆ เช่น die area, cell count

2. ดูผลลัพธ์สุดท้าย:
   ```bash
   ls run/RUN_2026-07-18_09-30-00/final/
   ```
   ```
   gds/     ← GDS file ที่ส่งโรงงานได้
   lef/     ← Abstract view สำหรับใช้เป็น macro
   nl/      ← Netlist
   lib/     ← Timing library (Liberty format)
   spef/    ← Extracted parasitics
   ```

> 💡 **เคล็ดลับสำหรับการ Debug:** เมื่อ flow ล้มเหลวที่ step ใด ให้เข้าไปดูโฟลเดอร์ของ step นั้น อ่าน `run.log` เพื่อดู error message และดู `state_in.json` เพื่อดูว่า input เข้ามาถูกต้องหรือไม่

---

### 1.5 ทำความเข้าใจ Metrics สำคัญ

หลังจาก flow เสร็จ LibreLane จะรายงาน metrics สำคัญ บางส่วนที่ควรรู้:

| Metric | ความหมาย | ค่าที่ดี |
|---|---|---|
| **Antenna Violations** | ผล DRC ด้าน antenna effect | 0 violations |
| **LVS** | Layout ตรงกับ netlist หรือไม่ | Passed |
| **DRC** | Layout ตาม design rules หรือไม่ | 0 violations |
| **Cell Count** | จำนวน standard cell ที่ใช้ | ยิ่งน้อยยิ่งประหยัด area |
| **Worst Slack** | timing margin ที่แย่ที่สุด | ต้องเป็นบวก (> 0) |
| **Total Power** | กำลังไฟที่ใช้ | ขึ้นกับ spec |

สำหรับ counter ง่าย ๆ ใน exercise นี้ทุก metric ควรผ่านโดยไม่มีปัญหา

---

### การแก้ไขปัญหา

| อาการ | สาเหตุที่เป็นไปได้ | วิธีแก้ |
|---|---|---|
| `command not found: librelane` | ยังไม่ได้เข้า `nix-shell` | รัน `nix-shell` ก่อนแล้วลองใหม่ |
| PDK ดาวน์โหลดช้าหรือค้าง | network ช้า | รอ หรือตรวจ internet connection |
| `DESIGN_NAME not found` | ชื่อ module ใน Verilog ไม่ตรงกับ `DESIGN_NAME` | ตรวจว่า `module counter` ใน `.sv` ตรงกับ `DESIGN_NAME: counter` ใน config |
| `DRC: X violations` | layout ไม่ตาม design rule | ดู log ใน `run/<timestamp>/60-klayout-drc/` |
| `LVS: FAILED` | layout ไม่ตรงกับ netlist | มักเกิดจาก power connection ผิด ดู `run/<timestamp>/65-openroad-lvs/` |
| OpenROAD GUI หรือ KLayout ไม่เปิด | display environment ผิด | ตรวจว่าใช้ desktop session ที่มี X display หรือ Wayland |
| `Error: No run directory found` | ลืมใส่ `--last-run` หรือยังไม่เคยรัน | รันโดยไม่มี `--last-run` ก่อน แล้วค่อยเปิด GUI |

---

## Lab 2: ปรับแต่ง Configuration Variables

**เวลาโดยประมาณ:** 60–90 นาที  
**Objective:** เข้าใจและปรับแต่ง configuration variable ของ LibreLane เพื่อควบคุม die area, placement density, การวาง pin, DEF template, และ placement obstruction

---

### พื้นหลัง: Design และ Config เริ่มต้นของ Exercise 2

Exercise นี้ใช้ design ที่ซับซ้อนกว่า Lab 1 — เป็น **multiplier** ที่ใช้ interface เดียวกับ **Tiny Tapeout** (platform สำหรับ tapeout รวมกลุ่มที่ให้นักศึกษาและ maker ส่ง design ได้ฟรี/ราคาถูก)

#### ไฟล์ `src/project.sv` — Design ของ Exercise 2

```systemverilog
module tt_um_example (
    input  wire [7:0] ui_in,    // Dedicated inputs (8-bit)
    output wire [7:0] uo_out,   // Dedicated outputs (8-bit)
    input  wire [7:0] uio_in,   // Bidirectional IOs: Input path
    output wire [7:0] uio_out,  // Bidirectional IOs: Output path
    output wire [7:0] uio_oe,   // Bidirectional IOs: Output enable
    input  wire       ena,      // Always 1 เมื่อ design ถูก power on
    input  wire       clk,      // Clock
    input  wire       rst_n     // Reset (active-low)
);
    logic [15:0] product;

    always @(posedge clk, negedge rst_n) begin
        if (!rst_n)
            product <= '0;
        else
            product <= ui_in * uio_in;  // คูณ 8x8 bit → ผล 16-bit
    end

    assign uo_out  = product[7:0];   // ครึ่งล่างของผลคูณ
    assign uio_out = product[15:8];  // ครึ่งบนของผลคูณ
    assign uio_oe  = '0;             // uio เป็น input ทั้งหมด
endmodule
```

Design นี้ใหญ่กว่า counter ใน Lab 1 มาก เพราะการคูณ 8x8 bit ต้องการ logic gate จำนวนมาก ทำให้การปรับแต่ง `DIE_AREA` และ `PL_TARGET_DENSITY_PCT` มีความหมายมากขึ้น

#### ไฟล์ `config.yaml` เริ่มต้น

```yaml
DESIGN_NAME: tt_um_example
VERILOG_FILES: dir::src/*.sv     # โหลดทุกไฟล์ .sv ใน src/
CLOCK_PORT: clk
CLOCK_PERIOD: 10                 # 100 MHz

# ปรับ margin รอบขอบ die ให้บาง (ประหยัด area)
TOP_MARGIN_MULT: 1
BOTTOM_MARGIN_MULT: 1
LEFT_MARGIN_MULT: 6
RIGHT_MARGIN_MULT: 6
```

> 📝 **หมายเหตุ:** `VERILOG_FILES: dir::src/*.sv` ใช้ wildcard `*` โหลดทุกไฟล์ `.sv` ใน `src/` ซึ่งสะดวกกว่าการระบุชื่อไฟล์ทีละตัว

1. เปลี่ยน directory ไปที่ exercise 2:
   ```bash
   cd exercise_2
   ```

---

### 2.1 กำหนด Die Area และ Placement Density

#### Die Area คืออะไร?

**Die Area** คือขนาดของ chip ที่เราจะออกแบบ กำหนดเป็น `[x1, y1, x2, y2]` หน่วย µm โดย x1,y1 คือมุมล่างซ้าย (origin) และ x2,y2 คือมุมขวาบน

โดย default LibreLane ใช้ `FP_SIZING: relative` ซึ่งคำนวณขนาด die อัตโนมัติจากจำนวน cell และ density ที่กำหนด แต่ในงานจริงเราอาจต้องการกำหนดขนาดแน่นอน เช่น ให้พอดีกับ slot บน chip ที่แบ่งพื้นที่ไว้แล้ว

#### Placement Density คืออะไร?

`PL_TARGET_DENSITY_PCT` คือ **สัดส่วนของพื้นที่ที่ต้องการให้มี standard cell อยู่** ภายใน die area ที่กำหนด

สิ่งสำคัญที่ต้องเข้าใจ: ค่านี้ **ไม่ได้บังคับให้กระจาย cell ทั่ว die** — มันบอก placer ว่า "วาง cell ให้แน่นแค่ไหน" ถ้า die ใหญ่มากแต่ density สูง cell จะกระจุกกันเป็นก้อน ไม่กระจาย

ลองทดสอบเพื่อเห็นผลกับตา:

#### ขั้นตอน: ทดสอบ Die Area และ Density

1. เปิดไฟล์ `config.yaml` ด้วย text editor แล้วเพิ่มบรรทัดเหล่านี้:
   ```yaml
   FP_SIZING: absolute
   DIE_AREA: [0, 0, 150, 150]
   PL_TARGET_DENSITY_PCT: 80
   ```

   `config.yaml` ทั้งหมดควรมีหน้าตาดังนี้:
   ```yaml
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

2. รัน flow:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml
   ```

3. เปิดดูผลใน OpenROAD GUI:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml --last-run --flow OpenInOpenROAD
   ```

   จะเห็น standard cell วางกระจุกกันเป็นก้อนกลางๆ die area ที่กว้าง 150×150 µm ทั้งที่ design มีขนาดเล็กกว่ามาก — นี่คือผลของ density 80% บน die ที่ใหญ่เกินความจำเป็น

   > 📝 **หมายเหตุ:** pin บน die ขนาด 150×150 µm จะอยู่ริมขอบ ส่วน cell จะกระจุกอยู่ตรงกลาง ทำให้ routing wire ยาวและ congested

#### ทำความเข้าใจผ่านภาพ: ผล Die 150×150, Density 80%

สังเกตจาก OpenROAD GUI ที่เห็น:
- **สีแดง/เขียว** บนขอบ คือ I/O pin ของ design (ui_in, uo_out, uio_in ฯลฯ)
- **กลุ่ม cell** อยู่ตรงกลาง — ไม่กระจายทั่ว die
- **สีฟ้าอ่อนแนวตั้ง** คือ TopMetal1 power strap พาด die ทั้งหมด
- routing ต้องยืดข้ามพื้นที่ว่างมากเกินจำเป็น

#### แบบฝึกหัด: หาขนาด Die และ Density ที่เหมาะสมที่สุด

เป้าหมายคือหาค่าที่ **die เล็กที่สุด** และ **density สูงที่สุด** ที่ flow ยังผ่านได้

กรอบความคิด:
- ถ้า die เล็กเกินไป → placer จะ error "cannot fit cells"
- ถ้า density สูงเกินไป → router จะ error "routing congestion"
- ค่าที่ดีอยู่ระหว่างสองขอบนี้

ลองปรับทีละขั้น เช่น เริ่มที่ `[0, 0, 100, 100]` density 50% แล้วค่อย ๆ ลด die หรือเพิ่ม density:

```yaml
# ลองค่าต่าง ๆ:
DIE_AREA: [0, 0, 100, 100]   # เล็กลง
PL_TARGET_DENSITY_PCT: 60    # เพิ่ม density

# ถ้าผ่าน ลองต่อ:
DIE_AREA: [0, 0, 80, 80]
PL_TARGET_DENSITY_PCT: 65
```

> ⚠️ **คำเตือน:** ถ้า placement fail จะเห็น error คล้ายนี้: `[ERROR] Unable to place all instances` หรือ `Placement failed` ให้เพิ่ม `DIE_AREA` หรือลด `PL_TARGET_DENSITY_PCT`

> ⚠️ **คำเตือน:** ถ้า routing fail จะเห็น error คล้ายนี้: `[ERROR] Routing congestion` ให้เพิ่ม `DIE_AREA` หรือลด `PL_TARGET_DENSITY_PCT` หรือเพิ่ม `GRT_ALLOW_CONGESTION: true` เพื่อให้ flow ผ่านไปก่อน

> ✅ **จุดตรวจสอบ:** เมื่อหาค่าที่เหมาะสมได้แล้ว flow ควรจบด้วย `DRC Passed ✅`, `LVS Passed ✅`

---

### 2.2 กำหนดตำแหน่ง Pin เอง

#### I/O Placement ใน LibreLane ทำงานอย่างไร?

ใน Classic flow มี 3 step ที่เกี่ยวกับ pin placement เรียงกัน:

```
OpenROAD.GlobalPlacementSkipIO   ← วาง cell โดยไม่สนใจ pin ก่อน
        ↓
OpenROAD.IOPlacement             ← วาง pin ให้ใกล้ cell ที่ต้องการมากที่สุด
        ↓
OpenROAD.GlobalPlacement         ← วาง cell อีกรอบ โดยคำนึงถึงตำแหน่ง pin แล้ว
```

ผลลัพธ์ default คือ pin จะกระจายรอบ die ตามที่ router เห็นว่าเหมาะสม ดังที่เห็นใน openroad_1.png — pin อยู่หลายด้าน

แต่ในงานจริง เช่น ออกแบบ macro สำหรับรวมเข้า top-level design เราอาจต้องการควบคุมว่า pin อยู่ด้านไหน เช่น **input ซ้าย, output ขวา** เพื่อให้ routing เป็นระเบียบ

#### ทำความเข้าใจ Port ของ Design

เปิด `src/project.sv` และดู port list:

```
Inputs:  ui_in[7:0], uio_in[7:0], ena, clk, rst_n   → จะวางทางซ้าย (#W)
Outputs: uo_out[7:0], uio_out[7:0], uio_oe[7:0]     → จะวางทางขวา (#E)
```

> 📝 **หมายเหตุ:** `uio_oe` คือ output enable ของ bidirectional port — แม้ชื่อจะมี "io" แต่จริง ๆ เป็น output signal ให้วางทางขวา

#### ขั้นตอน: สร้างไฟล์ `pins.cfg`

1. สร้างไฟล์ใหม่ชื่อ `pins.cfg` ในโฟลเดอร์ `exercise_2/`:
   ```bash
   touch pins.cfg
   ```
   หรือใช้ text editor สร้างไฟล์ใหม่

2. เพิ่มเนื้อหาต่อไปนี้ใน `pins.cfg`:
   ```
   #N

   #S

   #E
   uo_out[0]
   uo_out[1]
   uo_out[2]
   uo_out[3]
   uo_out[4]
   uo_out[5]
   uo_out[6]
   uo_out[7]
   uio_out[0]
   uio_out[1]
   uio_out[2]
   uio_out[3]
   uio_out[4]
   uio_out[5]
   uio_out[6]
   uio_out[7]
   uio_oe[0]
   uio_oe[1]
   uio_oe[2]
   uio_oe[3]
   uio_oe[4]
   uio_oe[5]
   uio_oe[6]
   uio_oe[7]

   #W
   clk
   rst_n
   ena
   ui_in[0]
   ui_in[1]
   ui_in[2]
   ui_in[3]
   ui_in[4]
   ui_in[5]
   ui_in[6]
   ui_in[7]
   uio_in[0]
   uio_in[1]
   uio_in[2]
   uio_in[3]
   uio_in[4]
   uio_in[5]
   uio_in[6]
   uio_in[7]
   ```

   **อธิบาย syntax ของ `pins.cfg`:**
   - `#N`, `#S`, `#E`, `#W` — compass direction: North (บน), South (ล่าง), East (ขวา), West (ซ้าย)
   - ชื่อ pin ที่ตามมาหลัง direction header จะถูกวางบนด้านนั้น
   - `#N` และ `#S` ว่างเปล่า = ไม่มี pin วางด้านบนและล่าง
   - ต้องระบุ pin bit by bit เช่น `ui_in[0]`, `ui_in[1]`, ... ไม่ใช่ `ui_in[7:0]`

3. เพิ่มบรรทัดนี้ใน `config.yaml` (ใต้ `DIE_AREA`):
   ```yaml
   FP_PIN_ORDER_CFG: dir::pins.cfg
   ```

4. รัน flow:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml
   ```

5. เปิดดูผลใน OpenROAD GUI:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml --last-run --flow OpenInOpenROAD
   ```

#### ผลลัพธ์ที่คาดหวัง

หน้าตาใน OpenROAD GUI ควรเปลี่ยนจาก pin กระจายรอบด้าน เป็นแบบที่เห็นใน `openroad_2.png`:
- **ด้านซ้าย (West)** — เห็นลูกศรสีเขียวชี้เข้า → input pin ทั้งหมด (clk, rst_n, ui_in, uio_in)
- **ด้านขวา (East)** — เห็นลูกศรสีเขียวชี้ออก → output pin ทั้งหมด (uo_out, uio_out, uio_oe)
- **ด้านบน/ล่าง** — ว่างเปล่า ไม่มี pin

> ✅ **จุดตรวจสอบ:** เปิด Inspector แล้วคลิกที่ pin ทางซ้ายควรเห็น `DIRECTION: INPUT` ส่วนทางขวาควรเห็น `DIRECTION: OUTPUT`

---

### 2.3 ใช้ DEF Template (รูปแบบ Tiny Tapeout)

#### DEF Template คืออะไร?

**DEF (Design Exchange Format)** คือ format ไฟล์ที่เก็บข้อมูล physical design เช่น die area, pin placement, row definitions และ track grid

**DEF Template** คือไฟล์ DEF ที่กำหนด die area และ pin ไว้ล่วงหน้าแล้ว — LibreLane จะนำ pin placement จาก template มาใช้แทน การกำหนดเอง ทำให้ design เราพอดีกับ "slot" ที่กำหนดไว้

ในกรณีของ **Tiny Tapeout** แต่ละ user ได้พื้นที่เป็น tile ขนาดมาตรฐาน (1x1, 2x1, 2x2, ฯลฯ) และ pin ของทุก tile ต้องอยู่ตรงตำแหน่งเดียวกันเสมอ เพื่อให้ chip-level ที่รวม tile ทั้งหมดเชื่อมต่อกันได้ถูกต้อง

ดูไฟล์ template ที่มีใน `def/`:

| ชื่อไฟล์ | DIE_AREA ที่ต้องใส่ใน config |
|---|---|
| `tt_block_1x1_pgvdd.def` | `[0, 0, 202.08, 154.98]` |
| `tt_block_1x2_pgvdd.def` | `[0, 0, 202.08, 313.74]` |
| `tt_block_2x1_pgvdd.def` | `[0, 0, 419.52, 154.98]` |
| `tt_block_2x2_pgvdd.def` | `[0, 0, 419.52, 313.74]` |
| `tt_block_3x1_pgvdd.def` | `[0, 0, 636.96, 154.98]` |
| `tt_block_4x2_pgvdd.def` | `[0, 0, 854.40, 313.74]` |
| `tt_block_8x2_pgvdd.def` | `[0, 0, 1724.16, 313.74]` |

> 📝 **หมายเหตุ:** ชื่อ `pgvdd` หมายถึง "power gated with VDD" — template นี้มี power gating infrastructure รวมอยู่ด้วย ซึ่งเป็น feature ของ Tiny Tapeout chip

#### เหตุใดต้องระบุ `DIE_AREA` ซ้ำ?

ค่า die area อยู่ในไฟล์ `.def` แล้ว แต่ LibreLane ต้องการให้ระบุใน `config.yaml` เพื่อตรวจสอบความสอดคล้อง — ถ้าค่าไม่ตรงกัน LibreLane จะ error ทันที ป้องกันการใช้ template ผิดขนาดโดยไม่รู้ตัว

#### ขั้นตอน: ใช้ DEF Template 1x1

1. ลบหรือ comment out `FP_PIN_ORDER_CFG` ออกจาก `config.yaml` ก่อน (เพราะ template มี pin placement ของตัวเองอยู่แล้ว):
   ```yaml
   # FP_PIN_ORDER_CFG: dir::pins.cfg   ← comment out ด้วย #
   ```

2. แก้ไข `config.yaml` ให้เป็นดังนี้:
   ```yaml
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

3. รัน flow:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml
   ```

4. ดูผลใน OpenROAD GUI:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml --last-run --flow OpenInOpenROAD
   ```

#### ทำความเข้าใจภาพผลลัพธ์ (openroad_3.png)

สิ่งที่เห็นในภาพเมื่อใช้ DEF template:
- **Pin บนด้านบน** — สามเหลี่ยมเล็ก ๆ แน่น ๆ แถวเดียว คือ pin ของ `uo_out`, `uio_out`, `uio_oe` — ตำแหน่งนี้กำหนดโดย template ไม่ใช่เรา
- **Pin บนด้านล่าง** — `ui_in`, `uio_in`, `clk`, `rst_n`, `ena`
- **แถบสีฟ้าอ่อนแนวตั้ง** — Power strap ของ TopMetal1 (VPWR)
- Die ขนาด 202.08×154.98 µm แน่นกว่าการใช้ 150×150 µm ก่อนหน้า

> 📝 **หมายเหตุ:** สังเกตว่า pin ของ Tiny Tapeout template จะอยู่บนและล่าง — ต่างจาก pins.cfg ที่เราสร้างเองซึ่งวางซ้าย-ขวา นี่เป็น convention ของ Tiny Tapeout chip

#### แบบฝึกหัด: ลองขนาด Template อื่น

ลองเปลี่ยนเป็น template 2x1 (กว้างขึ้น 2 เท่า):

```yaml
DIE_AREA: [0, 0, 419.52, 154.98]
FP_DEF_TEMPLATE: dir::def/tt_block_2x1_pgvdd.def
```

รัน flow และสังเกตว่า cell กระจาย (density ลดลง) เพราะ area ใหญ่ขึ้นแต่ logic เท่าเดิม

> ✅ **จุดตรวจสอบ:** ถ้า `DIE_AREA` ไม่ตรงกับ template จะเห็น error: `[ERROR] DEF template die area mismatch` — ให้ตรวจตัวเลขในตารางด้านบน

---

### 2.4 วาง Placement Obstruction

#### Obstruction คืออะไร และต่างจาก Soft Blockage อย่างไร?

ในการออกแบบ chip จริง พื้นที่บน die ไม่ได้ว่างทั้งหมด — อาจมี macro block อยู่แล้ว หรือต้องการพื้นที่ว่างสำหรับ routing พิเศษ LibreLane รองรับสองแบบ:

| ประเภท | Variable | ผล |
|---|---|---|
| **Hard Obstruction** | `FP_OBSTRUCTIONS` | ห้าม cell วางในพื้นที่นี้เด็ดขาด ไม่มีแม้แต่ standard cell row ถูกสร้าง |
| **Soft Blockage** | `PL_SOFT_OBSTRUCTIONS` | ห้าม cell วางในช่วง initial placement แต่ buffer หรือ antenna fix cell อาจวางได้ภายหลัง |

ใช้ `FP_OBSTRUCTIONS` เมื่อต้องการพื้นที่ที่ **ว่างสนิท** (เช่น จะวาง macro ทีหลัง)
ใช้ `PL_SOFT_OBSTRUCTIONS` เมื่อต้องการ "ดัน" cell ออก แต่ยังยืดหยุ่นได้

#### ขั้นตอน: วาง Hard Obstruction

ทดลองใช้กับ 1x1 template (ที่ run ได้ผ่านใน 2.3):

1. เพิ่มบรรทัดต่อไปนี้ใน `config.yaml`:
   ```yaml
   FP_OBSTRUCTIONS:
     - [30, 30, 40, 40]
     - [120, 100, 150, 115]
     - [100, 20, 180, 30]
   ```

   รูปแบบของแต่ละ entry คือ `[x1, y1, x2, y2]` หน่วยเป็น **µm** โดย:
   - `[30, 30, 40, 40]` — สี่เหลี่ยม 10×10 µm ที่มุม (30,30)
   - `[120, 100, 150, 115]` — สี่เหลี่ยม 30×15 µm ทางขวาบน
   - `[100, 20, 180, 30]` — สี่เหลี่ยม 80×10 µm แถบนอนกว้าง

2. `config.yaml` ทั้งหมด ณ จุดนี้ควรเป็น:
   ```yaml
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

   FP_OBSTRUCTIONS:
     - [30, 30, 40, 40]
     - [120, 100, 150, 115]
     - [100, 20, 180, 30]
   ```

3. รัน flow:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml
   ```

4. ดูผลใน OpenROAD GUI:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml --last-run --flow OpenInOpenROAD
   ```

#### ทำความเข้าใจภาพผลลัพธ์ (openroad_4.png)

ใน OpenROAD GUI จะเห็น:
- **พื้นที่ว่าง** ขนาดสี่เหลี่ยมตรงตำแหน่งที่กำหนด — ไม่มี cell วางอยู่เลย
- **เส้น routing** วนหลีกรอบ obstruction แทนที่จะตัดผ่าน
- Cell ที่เหลือถูก "บีบ" ไปอยู่ในพื้นที่ที่เหลือ

**วิธีตรวจสอบตำแหน่ง obstruction ใน GUI:**
ไปที่ **Display Control → Blockages** แล้วเปิด checkbox จะเห็น obstruction ถูก highlight ชัดขึ้น

> ⚠️ **คำเตือน:** ถ้าวาง obstruction มากเกินไปจน routing ผ่านไม่ได้ จะเห็น error: `[ERROR] Antenna fix failed` หรือ `Routing DRC violations` ให้ลด obstruction หรือเพิ่มขนาด die

#### แบบฝึกหัด: ใส่ Obstruction ให้มากที่สุดโดยไม่ทำให้ flow พัง

ลองเพิ่ม obstruction หลาย ๆ ชิ้น ทั้งเล็กและใหญ่ และดูว่าขีดจำกัดอยู่ที่ไหน:

```yaml
FP_OBSTRUCTIONS:
  - [30, 30, 40, 40]
  - [60, 30, 70, 40]
  - [90, 30, 100, 40]
  - [120, 100, 150, 115]
  - [100, 20, 180, 30]
  - [50, 80, 80, 100]   # เพิ่มใหม่ — จะ flow ยังผ่านไหม?
```

---

### 2.5 Configuration Variables อื่น ๆ ที่ควรรู้

นอกจาก variable ที่ใช้ใน exercise นี้ ยังมีอีกหลายตัวที่มีประโยชน์ในงานจริง:

| Variable | ความหมาย | ตัวอย่างการใช้ |
|---|---|---|
| `SYNTH_STRATEGY` | เลือก strategy ของ Yosys: `AREA 0`–`AREA 4` หรือ `DELAY 0`–`DELAY 4` | `SYNTH_STRATEGY: "DELAY 2"` — optimize timing |
| `VERILOG_DEFINES` | ส่ง define เข้า preprocessor ตอน synthesis | `VERILOG_DEFINES: ["SIM=1", "WIDTH=8"]` |
| `GRT_ALLOW_CONGESTION` | อนุญาตให้ global routing ผ่านพื้นที่ congested | `GRT_ALLOW_CONGESTION: true` |
| `GRT_ANTENNA_ITERS` | จำนวนรอบการแก้ antenna violation | `GRT_ANTENNA_ITERS: 5` |
| `RSZ_DONT_TOUCH_RX` | regex ชื่อ net/cell ที่ห้าม resizer แตะ | `RSZ_DONT_TOUCH_RX: "clk_buf*"` |
| `PL_RESIZER_HOLD_SLACK_MARGIN` | เพิ่ม margin สำหรับ hold time | `PL_RESIZER_HOLD_SLACK_MARGIN: 0.05` |
| `FP_PDN_VPITCH` | ระยะห่างระหว่าง vertical power strap | `FP_PDN_VPITCH: 20` |
| `FP_PDN_HPITCH` | ระยะห่างระหว่าง horizontal power strap | `FP_PDN_HPITCH: 20` |

> 💡 **เคล็ดลับ:** ดูรายการ variable ทั้งหมดพร้อมคำอธิบายได้ที่: [LibreLane Step Config Vars](https://librelane.readthedocs.io/en/latest/reference/step_config_vars.html) — แนะนำให้ bookmark ไว้ เพราะจะใช้บ่อยมากในการออกแบบจริง

---

### การแก้ไขปัญหา

| อาการ | สาเหตุ | วิธีแก้ |
|---|---|---|
| `[ERROR] Unable to place all instances` | Die เล็กเกินไปหรือ density สูงเกินไป | เพิ่ม `DIE_AREA` หรือลด `PL_TARGET_DENSITY_PCT` |
| `[ERROR] Routing congestion` | Cell แน่นเกินจนเดินสายไม่ผ่าน | ลด `PL_TARGET_DENSITY_PCT` ลง 5–10% หรือเพิ่ม `GRT_ALLOW_CONGESTION: true` |
| `[ERROR] DEF template die area mismatch` | `DIE_AREA` ใน config ไม่ตรงกับที่อยู่ใน `.def` | ดูตารางขนาด template ด้านบนและแก้ไขให้ตรง |
| `[ERROR] Pin X not found in design` | ชื่อ pin ใน `pins.cfg` ไม่ตรงกับชื่อใน Verilog | ตรวจสอบ port name ใน `src/project.sv` ให้ตรงทุกตัวอักษร |
| `[ERROR] Cannot find pins.cfg` | Path ของ `FP_PIN_ORDER_CFG` ผิด | ตรวจว่าใช้ `dir::pins.cfg` และสร้างไฟล์ไว้ใน directory เดียวกับ `config.yaml` |
| `[ERROR] Antenna fix failed` | Obstruction ขวางเส้นทาง routing ทั้งหมด | ลดจำนวนหรือขนาด obstruction |
| Warning: `unconstrained path` | มี path ที่ไม่มี timing constraint | ปกติ ignore ได้สำหรับ exercise นี้ |

---

## Lab 3: ควบคุม Flow

**เวลาโดยประมาณ:** 45–60 นาที  
**Objective:** เข้าใจโครงสร้าง Classic flow ทุก step, ใช้ `--to` / `--from` / `--skip` เพื่อรัน flow เฉพาะส่วน, อ่าน state file และเชื่อมต่อ step ได้อย่างถูกต้อง

---

### พื้นหลัง: ทำไมต้องควบคุม Flow?

ใน Lab 1–2 เรารัน flow ตั้งแต่ต้นจนจบทุกครั้ง สำหรับ design เล็ก ๆ ที่ใช้เวลา 5–10 นาทีก็ไม่เป็นปัญหา แต่ในงานจริง:

- design ขนาด chip จริงใช้เวลา **หลายชั่วโมง** ต่อ run
- `KLayout.DRC` เพียง step เดียวบางครั้งใช้เวลา **30–60 นาที**
- เมื่อ debug ปัญหา synthesis เราไม่จำเป็นต้องรอ routing เสร็จ
- เมื่อแก้ bug เล็กน้อยหลัง placement ก็ไม่ต้อง synthesize ใหม่ตั้งแต่ต้น

LibreLane จึงมีกลไก **partial run** ให้เลือกได้ว่าจะรันจากไหนถึงไหน และ step ไหนจะข้ามไป

#### โครงสร้าง Classic Flow ทั้งหมด

Classic flow ประกอบด้วย step ต่าง ๆ ตามลำดับนี้ — แต่ละ step รับ **state_in** และส่งออก **state_out**:

```
Phase 1: Lint & Synthesis
  Verilator.Lint                ← ตรวจ syntax/style ของ Verilog
  Checker.LintTimingConstructs  ← ตรวจ construct ที่อาจมีปัญหา timing
  Checker.LintErrors / Warnings
  Yosys.JsonHeader              ← อ่าน metadata ของ design
  Yosys.Synthesis               ← แปลง RTL → gate-level netlist  ★
  Checker.YosysUnmappedCells
  Checker.YosysSynthChecks
  Checker.NetlistAssignStatements

Phase 2: Pre-PnR Checks
  OpenROAD.CheckSDCFiles
  OpenROAD.CheckMacroInstances
  OpenROAD.STAPrePNR            ← Timing analysis ก่อน Place & Route

Phase 3: Floorplan
  OpenROAD.Floorplan            ← กำหนด die area, row, track  ★
  OpenROAD.DumpRCValues
  Odb.CheckMacroAntennaProperties
  Odb.SetPowerConnections
  Odb.ManualMacroPlacement
  OpenROAD.CutRows
  OpenROAD.TapEndcapInsertion   ← แทรก tap cell และ end cap cell
  Odb.AddPDNObstructions
  OpenROAD.GeneratePDN          ← สร้าง Power Distribution Network  ★
  Odb.RemovePDNObstructions
  Odb.AddRoutingObstructions

Phase 4: Placement
  OpenROAD.GlobalPlacementSkipIO ← วาง cell คร่าว ๆ (ยังไม่สนใจ I/O pin)
  OpenROAD.IOPlacement           ← วาง I/O pin  ★
  Odb.CustomIOPlacement          ← ใช้ pins.cfg (ถ้ามี)
  Odb.ApplyDEFTemplate           ← ใช้ DEF template (ถ้ามี)
  OpenROAD.GlobalPlacement       ← วาง cell โดยคำนึงถึง pin แล้ว  ★
  Odb.WriteVerilogHeader
  Checker.PowerGridViolations
  OpenROAD.STAMidPNR
  OpenROAD.RepairDesignPostGPL   ← แก้ timing/fanout หลัง global placement
  Odb.ManualGlobalPlacement
  OpenROAD.DetailedPlacement     ← snap cell เข้า grid อย่างแม่นยำ  ★

Phase 5: Clock Tree Synthesis
  OpenROAD.CTS                   ← สร้าง clock tree  ★
  OpenROAD.STAMidPNR
  OpenROAD.ResizerTimingPostCTS  ← ปรับ timing หลัง CTS
  OpenROAD.STAMidPNR

Phase 6: Routing
  OpenROAD.GlobalRouting         ← เดินสายคร่าว ๆ  ★
  OpenROAD.CheckAntennas
  OpenROAD.RepairDesignPostGRT
  Odb.DiodesOnPorts
  Odb.HeuristicDiodeInsertion
  OpenROAD.RepairAntennas
  OpenROAD.ResizerTimingPostGRT
  OpenROAD.STAMidPNR
  OpenROAD.DetailedRouting       ← เดินสายละเอียด (ช้าที่สุด)  ★
  Odb.RemoveRoutingObstructions
  OpenROAD.CheckAntennas
  Checker.TrDRC
  ...

Phase 7: Post-Route Checks & GDS
  OpenROAD.FillInsertion
  OpenROAD.RCX                   ← Resistance/Capacitance extraction
  OpenROAD.STAPostPNR            ← Final timing sign-off
  Magic.StreamOut                ← export GDS
  KLayout.StreamOut
  KLayout.DRC                    ← Design Rule Check (ช้ามาก)  ★
  Netgen.LVS                     ← Layout vs Schematic
  ...
  Misc.ReportManufacturability   ← step สุดท้าย
```

> 📝 **หมายเหตุ:** Step ที่มี ★ คือ step สำคัญที่มักใช้เป็นจุด `--to` / `--from`

#### Design ของ Exercise 3: 256-bit Shift Register

```systemverilog
module shift_register (
    input  logic    clk_i,
    input  logic    in_i,    // serial input
    input  logic    rst_ni,
    output logic    out_o    // serial output (bit สุดท้าย)
);
    localparam BITS = 256;

    logic [BITS-1:0] shift_reg;

    always_ff @(posedge clk_i, negedge rst_ni) begin
        if (!rst_ni) begin
            shift_reg <= '0;
        end else begin
            for (int i = 0; i < BITS; i++) begin
                if (i == 0)
                    shift_reg[i] <= in_i;           // รับ bit ใหม่จาก input
                else
                    shift_reg[i] <= shift_reg[i-1]; // เลื่อน bit ไปทางขวา
            end
        end
    end

    assign out_o = shift_reg[BITS-1]; // output bit ที่ถูก shift มา 256 รอบ

endmodule
```

Design นี้คือสายโซ่ flip-flop (D-FF) 256 ตัวต่ออนุกรมกัน เมื่อ synthesize แล้ว Yosys จะสร้าง flip-flop จริง 256 ตัว แต่ละตัวเชื่อม Q→D ของตัวถัดไป ความพิเศษคือเมื่อดูใน OpenROAD หลัง global placement จะเห็นรูปร่างคล้าย "งูขด" เพราะ placer พยายามวาง FF ที่เชื่อมกันให้อยู่ใกล้กัน

#### Config ของ Exercise 3

```yaml
DESIGN_NAME: shift_register
VERILOG_FILES: dir::shift_register.sv
CLOCK_PORT: clk_i
CLOCK_PERIOD: 10                    # 100 MHz

FP_SIZING: absolute
DIE_AREA: [0, 0, 250, 125]
PL_TARGET_DENSITY_PCT: 60

# shift register ยาว 256 FF ต้องการ hold buffer จำนวนมาก
# ค่า default (25%) ไม่พอ ต้องเพิ่มเป็น 100%
GRT_RESIZER_HOLD_MAX_BUFFER_PCT: 100
PL_RESIZER_HOLD_MAX_BUFFER_PCT: 100
```

> 📝 **หมายเหตุ:** `GRT_RESIZER_HOLD_MAX_BUFFER_PCT: 100` หมายถึงอนุญาตให้ resizer แทรก hold buffer ได้มากถึง 100% ของจำนวน instance ที่มี (256 FF → แทรกได้สูงสุด 256 buffer) ค่า default คือ 25% ซึ่งไม่พอสำหรับ shift register ยาว ๆ

1. เปลี่ยน directory ไปที่ exercise 3:
   ```bash
   cd exercise_3
   ```

---

### 3.1 รัน Flow ถึง Step ที่กำหนด (`--to`)

#### ทำความเข้าใจ `--to`

flag `--to StepName` บอกให้ LibreLane **หยุดทำงานหลังจาก step นั้นเสร็จ** และ skip step ที่เหลือทั้งหมด มีประโยชน์มากเมื่อ:
- ยังแก้ bug อยู่ใน synthesis และยังไม่ต้องการรอ PnR
- ต้องการตรวจสอบ netlist หลัง synthesis ก่อนไปต่อ
- ต้องการดู placement ก่อนที่ routing จะเริ่ม

#### ขั้นตอน 3.1.1 — รันถึง Synthesis

1. รัน flow ถึงแค่ `Yosys.Synthesis`:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml --to Yosys.Synthesis
   ```
   shorthand ที่เหมือนกัน:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml -T Yosys.Synthesis
   ```

2. สังเกต terminal output — จะเห็น step ก่อน `Yosys.Synthesis` รันตามปกติ และทุก step หลังจากนั้นจะแสดง `[SKIPPED]`:
   ```
   [Step  1/76] Verilator.Lint             ✓
   [Step  5/76] Yosys.JsonHeader           ✓
   [Step  6/76] Yosys.Synthesis            ✓  ← หยุดที่นี่
   [Step  7/76] Checker.YosysUnmappedCells [SKIPPED]
   [Step  8/76] Checker.YosysSynthChecks   [SKIPPED]
   ...
   ```

3. ดูไฟล์ netlist ที่ได้จาก synthesis:
   ```bash
   ls run/RUN_*/06-yosys-synthesis/
   ```
   จะเห็นไฟล์ `shift_register.nl.v` (synthesized netlist) ที่มี `sg13g2_dfrbpq_1` (D-FF cell จาก IHP PDK) จำนวน 256 ตัว

   ลองตรวจสอบ:
   ```bash
   grep -c "dfrbpq" run/RUN_*/06-yosys-synthesis/shift_register.nl.v
   ```
   ควรได้ตัวเลขประมาณ 256 (จำนวน FF)

> ✅ **จุดตรวจสอบ:** ถ้าเห็น `[SKIPPED]` สำหรับ step หลัง `Yosys.Synthesis` แสดงว่า `--to` ทำงานถูกต้อง

#### ขั้นตอน 3.1.2 — รันถึง Global Placement แล้วดู "งูขด"

1. รัน flow ถึง `OpenROAD.GlobalPlacement`:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml -T OpenROAD.GlobalPlacement
   ```

   > 📝 **หมายเหตุ:** LibreLane จะสร้าง run directory ใหม่ทุกครั้ง เพราะไม่ได้ใส่ `--last-run` — ทำให้มีประวัติการรันแยกจากกัน

2. เปิด OpenROAD GUI:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml --last-run --flow OpenInOpenROAD
   ```

#### ทำความเข้าใจภาพ Global Placement ของ Shift Register

สิ่งที่เห็นใน OpenROAD GUI (ภาพ `openroad_1.png`):

- **กลุ่มสีน้ำเงิน** กระจายเป็น "เกาะ ๆ" ทั่ว die — แต่ละเกาะคือกลุ่มของ FF ที่ placer วางให้อยู่ใกล้กัน
- **เส้นสีชมพูแนวนอน** หนา — Power strap TopMetal2 (VPWR/VGND)
- **เส้นสีฟ้าอ่อนแนวตั้ง** — Power strap TopMetal1
- **ลูกศรสีเขียว** ทางซ้ายและขวา — I/O pin ของ design (in_i, out_o, clk_i, rst_ni)

สิ่งสำคัญที่ต้องสังเกต: **cell ยังไม่ "snap" เข้า standard cell row** — บางตัวลอยอยู่กลางพื้นที่ว่าง ไม่ตรงกับ grid ของ row นี่คือผลของ *Global Placement* ซึ่งวางแบบ "roughly" เพื่อลด wirelength ก่อน แต่ยังไม่แม่นยำ

ความแตกต่างระหว่าง Global Placement กับ Detailed Placement:

| | Global Placement | Detailed Placement |
|---|---|---|
| ผล | cell วางคร่าว ๆ อาจทับกัน | cell snap เข้า row grid แม่นยำ |
| เป้าหมาย | ลด total wirelength | แก้ overlap + จัดให้ตรง grid |
| ความเร็ว | เร็ว | ช้ากว่า |

---

### 3.2 รัน Flow จาก Step ที่กำหนด (`--from`)

#### ทำความเข้าใจ `--from` และ State File

flag `--from StepName` บอกให้ LibreLane **เริ่มรันจาก step นั้น** โดย skip step ก่อนหน้าทั้งหมด

แต่มีสิ่งที่ต้องระบุเพิ่ม: **state file** ที่จะใช้เป็น input ของ step แรก

เหตุผล: ใน run directory เดียวกันอาจมี `OpenROAD.GlobalPlacement` หลายครั้ง (เช่น ถ้ารันหลายรอบ) LibreLane จึงไม่รู้ว่าจะเอา state ไหนมาใช้ — เราต้องระบุเองด้วย `-i path/to/state_in.json`

#### ขั้นตอน: รัน Global Placement → Detailed Placement เท่านั้น

1. ดู run directory ล่าสุด:
   ```bash
   ls run/
   ```
   ตัวอย่าง output: `RUN_2026-07-18_10-30-00`

2. หา state_in.json ของ step `OpenROAD.GlobalPlacement`:
   ```bash
   ls run/RUN_2026-07-18_10-30-00/ | grep globalplacement
   ```
   จะเห็นโฟลเดอร์ชื่อประมาณ `28-openroad-globalplacement` (เลขหน้าอาจต่างกัน)

3. ตรวจสอบว่า `state_in.json` มีอยู่:
   ```bash
   ls run/RUN_2026-07-18_10-30-00/28-openroad-globalplacement/
   ```
   ```
   state_in.json    ← สิ่งที่เราต้องการ
   state_out.json
   shift_register.odb
   ...
   ```

4. ดูเนื้อหาของ `state_in.json` เพื่อทำความเข้าใจ:
   ```bash
   cat run/RUN_2026-07-18_10-30-00/28-openroad-globalplacement/state_in.json
   ```
   จะเห็น JSON ที่ map ชนิดไฟล์ไปยัง path เช่น:
   ```json
   {
     "odb": ".../.../shift_register.odb",
     "nl": ".../.../shift_register.nl.v",
     "pnl": ".../.../shift_register.pnl.v",
     "sdc": ".../.../shift_register.sdc",
     "metrics": { "design__instance__count": 256, ... }
   }
   ```
   State คือ snapshot ของ design ณ จุดนั้น — มีทั้ง path ของไฟล์และ metrics

5. รัน flow ตั้งแต่ `OpenROAD.GlobalPlacement` ถึง `OpenROAD.DetailedPlacement`:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml \
     --last-run \
     --from OpenROAD.GlobalPlacement \
     --with-initial-state run/RUN_2026-07-18_10-30-00/28-openroad-globalplacement/state_in.json \
     --to OpenROAD.DetailedPlacement
   ```

   shorthand ที่เหมือนกัน:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml \
     --last-run \
     -F OpenROAD.GlobalPlacement \
     -i run/RUN_2026-07-18_10-30-00/28-openroad-globalplacement/state_in.json \
     -T OpenROAD.DetailedPlacement
   ```

   > 📝 **หมายเหตุ:** แทนที่ `RUN_2026-07-18_10-30-00` ด้วย folder จริงที่เห็นจาก `ls run/`

6. สังเกต terminal output:
   ```
   [Step  1/76] Verilator.Lint             [SKIPPED]
   ...
   [Step 28/76] OpenROAD.GlobalPlacement   ✓  ← เริ่มที่นี่
   [Step 29/76] Odb.WriteVerilogHeader     ✓
   [Step 30/76] Checker.PowerGridViolations✓
   [Step 31/76] OpenROAD.STAMidPNR         ✓
   [Step 32/76] OpenROAD.RepairDesignPostGPL ✓
   [Step 33/76] Odb.ManualGlobalPlacement  ✓
   [Step 34/76] OpenROAD.DetailedPlacement ✓  ← หยุดที่นี่
   [Step 35/76] OpenROAD.CTS              [SKIPPED]
   ...
   ```

7. เปิด OpenROAD GUI เพื่อเปรียบเทียบกับก่อนหน้า:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml --last-run --flow OpenInOpenROAD
   ```

   ตอนนี้ cell ทุกตัวจะ **snap เข้า standard cell row** อย่างเป็นระเบียบ — ไม่มี cell ลอยผิดตำแหน่งอีกต่อไป

> ✅ **จุดตรวจสอบ:** ใน OpenROAD GUI หลัง Detailed Placement cell ทุกตัวควรเรียงเป็นแถว ไม่มีช่องว่างผิดปกติภายใน row

#### เปรียบเทียบ: ก่อนและหลัง Detailed Placement

| สิ่งที่เห็นใน GUI | หลัง Global Placement | หลัง Detailed Placement |
|---|---|---|
| ตำแหน่ง cell | คร่าว ๆ อาจไม่ตรง row | snap เข้า row grid ทุกตัว |
| ความทับซ้อน | อาจมี overlap เล็กน้อย | ไม่มี overlap เลย |
| routing | ยังไม่มี (แค่ placement) | ยังไม่มี |
| สี cell | น้ำเงิน กระจุกเป็นเกาะ | น้ำเงิน เรียงเป็นแถวในแต่ละ row |

---

### 3.3 ข้าม Step ที่ต้องการ (`--skip`)

#### ทำความเข้าใจ `--skip`

flag `--skip StepName` บอกให้ข้าม step นั้นในขณะที่รัน step อื่นทั้งหมดตามปกติ แตกต่างจาก `--to` ตรงที่ `--skip` ไม่หยุด flow — เพียงแค่ข้าม step ที่ระบุแล้วรันต่อ

ใช้ `--skip` เมื่อ:
- ต้องการข้าม step ที่ใช้เวลานานระหว่าง debug เช่น `KLayout.DRC`
- step นั้นมี error ที่รู้อยู่แล้วแต่ยังไม่ต้องการแก้ตอนนี้
- ต้องการดูผลขั้น final โดยไม่รอ verification

#### ขั้นตอน: รัน Full Flow โดย Skip DRC

1. รัน full flow แต่ข้าม `KLayout.DRC`:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml --skip KLayout.DRC
   ```
   shorthand:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml -S KLayout.DRC
   ```

2. สังเกตว่า flow รันทุก step ยกเว้น `KLayout.DRC` ที่จะแสดง `[SKIPPED]`

3. เวลารวมจะลดลงอย่างมีนัยสำคัญ — DRC เป็น step ที่ใช้เวลานานสุดใน flow

#### ข้าม Step หลายตัวพร้อมกัน

สามารถใช้ `--skip` หลายครั้งเพื่อข้ามหลาย step พร้อมกัน:

```bash
librelane --pdk ihp-sg13g2 config.yaml \
  --skip KLayout.DRC \
  --skip Magic.DRC \
  --skip Netgen.LVS
```

หรือใช้ shorthand:
```bash
librelane --pdk ihp-sg13g2 config.yaml -S KLayout.DRC -S Magic.DRC -S Netgen.LVS
```

> ⚠️ **คำเตือน:** อย่าข้าม DRC หรือ LVS ใน design ที่จะส่ง tapeout เพราะ DRC violations ทำให้โรงงานปฏิเสธไฟล์ และ LVS mismatch หมายความว่า chip จะทำงานไม่ตรงกับ schematic

---

### 3.4 รวมทุก Flag: สถานการณ์จริงในการ Debug

ในงานจริง มักใช้ flag หลายตัวร่วมกัน ดูตัวอย่างสถานการณ์:

#### สถานการณ์ที่ 1: แก้ Bug ใน Synthesis แล้วต้องการดูผลเร็ว

```bash
# รัน synthesis ใหม่ + placement แต่ข้าม DRC/LVS
librelane --pdk ihp-sg13g2 config.yaml \
  --to OpenROAD.DetailedPlacement \
  --skip KLayout.DRC
```

#### สถานการณ์ที่ 2: แก้ Config เล็กน้อย (เช่น density) แล้วรัน placement ใหม่

```bash
# หา state หลัง synthesis จาก run ล่าสุด
TIMESTAMP=$(ls -t run/ | head -1)
SYNTH_STATE="run/$TIMESTAMP/06-yosys-synthesis/state_out.json"

# รัน ตั้งแต่ floorplan ถึง detailed placement โดยข้าม DRC
librelane --pdk ihp-sg13g2 config.yaml \
  --last-run \
  -F OpenROAD.Floorplan \
  -i $SYNTH_STATE \
  -T OpenROAD.DetailedPlacement
```

#### สถานการณ์ที่ 3: ดู Final GDS โดยไม่รอ DRC

```bash
# รัน full flow แต่ข้ามทุก verification step ที่ช้า
librelane --pdk ihp-sg13g2 config.yaml \
  -S KLayout.DRC \
  -S Magic.DRC \
  -S Netgen.LVS
```

---

### 3.5 สรุป Flag ทั้งหมดและการใช้งาน

| Flag | Shorthand | ความหมาย | ตัวอย่าง |
|---|---|---|---|
| `--to StepName` | `-T` | รันถึง step นี้แล้วหยุด | `-T Yosys.Synthesis` |
| `--from StepName` | `-F` | เริ่มรันจาก step นี้ (ต้องใช้คู่กับ `-i`) | `-F OpenROAD.GlobalPlacement` |
| `--with-initial-state path` | `-i` | ระบุ state file ที่ใช้เป็น input ของ step แรก | `-i run/.../state_in.json` |
| `--skip StepName` | `-S` | ข้าม step นี้ไปในขณะที่รัน step อื่นปกติ | `-S KLayout.DRC` |
| `--last-run` | — | ใช้ run directory ล่าสุด แทนที่จะสร้างใหม่ | `--last-run` |
| `--flow FlowName` | — | เลือก flow พิเศษแทน Classic | `--flow OpenInOpenROAD` |

> 💡 **เคล็ดลับ:** Step name ต้องพิมพ์ถูกต้องทั้ง module และชื่อ เช่น `OpenROAD.GlobalPlacement` (ตัว O ใหญ่, ตัว G ใหญ่) ถ้าสะกดผิดจะได้ error `[ERROR] Step 'xxx' not found in flow`

---

### การแก้ไขปัญหา

| อาการ | สาเหตุ | วิธีแก้ |
|---|---|---|
| `[ERROR] Step 'X' not found in flow` | สะกดชื่อ step ผิด | ดูชื่อ step ที่ถูกต้องจากตาราง Classic flow ด้านบน |
| `[ERROR] No run directory found` | ใส่ `--last-run` แต่ยังไม่เคยรัน | รัน flow ปกติก่อนหนึ่งครั้ง |
| `[ERROR] state_in.json not found` | Path ของ `-i` ผิด | ตรวจสอบ timestamp และชื่อโฟลเดอร์ step ให้ตรง |
| Flow รัน step ที่ควร skip | ลืมใส่ `-S` หรือสะกดผิด | ตรวจสอบ flag และชื่อ step |
| `--from` แต่ flow รันตั้งแต่ต้น | ลืมใส่ `--last-run` คู่กับ `--from` | ต้องใส่ `--last-run` ด้วยเสมอเมื่อใช้ `--from` |
| `GRT_RESIZER hold buffer` error | Hold buffer quota เกิน | เพิ่ม `GRT_RESIZER_HOLD_MAX_BUFFER_PCT: 100` ใน config |
| FF count ไม่ครบ 256 | Yosys optimize FF ออกบางส่วน | ปกติ Yosys อาจ merge FF ที่ไม่ใช้ ถ้า logic ใช้ครบจะได้ครบ |

---

## Lab 4: การใช้ Macro

**เวลาโดยประมาณ:** 75–105 นาที  
**Objective:** เข้าใจแนวคิด hard macro และ PDN integration, implement `counter_8bit` เป็น macro ด้วยสองวิธี (Hierarchical และ Ring), copy ไฟล์ final มาใช้งาน, และรวม macro 4 ตัวเป็น `counter_32bit` ที่ทำงานได้จริง

---

### พื้นหลัง: Hard Macro คืออะไรและทำไมต้องใช้?

**Hard macro** คือ block วงจรที่ผ่านกระบวนการ place & route จนได้ GDS layout สมบูรณ์แล้ว — พร้อมใช้เป็น black box ใน design ระดับสูงขึ้นได้ทันที ไม่ต้อง synthesize หรือ route ใหม่

ในงานจริง macro ถูกใช้เพื่อ:
- **IP Reuse** — ออกแบบ block หนึ่งครั้ง ใช้ซ้ำหลายที่บน chip เดียวกันหรือใน project ต่าง ๆ
- **Third-party IP** — เช่น SRAM จากโรงงาน, PLL, ADC ที่โรงงาน/บริษัทอื่นออกแบบและรับประกันแล้ว
- **Timing closure** — macro ที่ sign-off แล้วมี timing ที่รับประกัน ทำให้ top-level timing analysis ง่ายขึ้น
- **PDK standard cells** — IO pad, ESD cell, decap cell ล้วนเป็น macro ชนิดหนึ่ง

#### ไฟล์ที่ประกอบขึ้นเป็น Macro

เมื่อ implement macro สำเร็จ LibreLane จะสร้างไฟล์ในโฟลเดอร์ `final/` ซึ่งแต่ละชนิดมีหน้าที่ต่างกัน:

| ไฟล์ | นามสกุล | หน้าที่ | ใช้ใน |
|---|---|---|---|
| **GDS** | `.gds` | Layout geometry จริง ทุก polygon ทุก layer | KLayout, ส่งโรงงาน |
| **LEF** | `.lef` | Abstract view — แค่ขอบเขตและ pin location (ไม่เห็น internal) | Router ที่ top-level ใช้ routing รอบ macro |
| **Netlist** | `.nl.v` | Gate-level netlist (unpowered) | LVS, formal verification |
| **Liberty** | `.lib` | Timing model: input/output delay, setup/hold, power | Static Timing Analysis |
| **SPEF** | `.spef` | Extracted parasitics (resistance, capacitance จริงจาก layout) | Post-layout timing sign-off |

> 📝 **หมายเหตุ:** Top-level router เห็นแค่ LEF (abstract) ไม่เห็น GDS ภายใน — นี่คือเหตุผลที่ route ผ่าน macro ไม่ได้ ต้องวนรอบเท่านั้น

#### ทำความเข้าใจ Design ทั้งหมดของ Lab 4

```
exercise_4/
├── counter_8bit/           ← Part 1: implement เป็น macro
│   ├── counter_8bit.sv     ← source code ของ macro
│   └── config.yaml         ← config สำหรับ implement macro
├── counter_32bit.sv        ← top-level design ที่ใช้ macro 4 ตัว
├── config.yaml             ← config ของ top-level
└── top.sv                  ← (bonus) design สำหรับ SRAM macro
```

#### Source Code: `counter_8bit.sv`

```systemverilog
module counter_8bit (
    input  logic       clk_i,
    input  logic       rst_ni,
    input  logic       en_i,      // Enable — นับเมื่อ en_i = 1 เท่านั้น

    output logic [7:0] count_o,
    output logic       ovf_o      // Overflow — เป็น 1 เมื่อ count ล้นจาก 255→0
);
    always_ff @(posedge clk_i) begin
        if (!rst_ni) begin
            count_o <= '0;
        end else begin
            if (en_i) begin
                {ovf_o, count_o} <= count_o + 1;
                // บวก 1 เข้า count_o และเก็บ carry bit ไว้ที่ ovf_o
                // เมื่อ count_o = 255 และ en_i = 1:
                //   255 + 1 = 256 = 9'b1_0000_0000
                //   ovf_o = 1, count_o = 0
            end
        end
    end
endmodule
```

> 📝 **หมายเหตุ:** `{ovf_o, count_o} <= count_o + 1` คือ **concatenation assignment** — นำ 9-bit ผลลัพธ์ (1 overflow bit + 8 data bit) ใส่ใน `{ovf_o, count_o}` ในครั้งเดียว

#### Source Code: `counter_32bit.sv` — Top-level ที่ใช้ Macro

```systemverilog
module counter_32bit (
    input  logic        clk_i,
    input  logic        rst_ni,
    input  logic        en_i,
    output logic [31:0] count_o,
    output logic        ovf_o
);
    logic counter_0_ovf, counter_1_ovf, counter_2_ovf;

    // counter_0: นับ bit[7:0] — enable เมื่อ en_i จาก outside
    counter_8bit counter_0 (.clk_i, .rst_ni, .en_i(en_i),
        .count_o(count_o[7:0]),   .ovf_o(counter_0_ovf));

    // counter_1: นับ bit[15:8] — enable เมื่อ counter_0 overflow
    counter_8bit counter_1 (.clk_i, .rst_ni, .en_i(counter_0_ovf),
        .count_o(count_o[15:8]),  .ovf_o(counter_1_ovf));

    // counter_2: นับ bit[23:16] — enable เมื่อ counter_1 overflow
    counter_8bit counter_2 (.clk_i, .rst_ni, .en_i(counter_1_ovf),
        .count_o(count_o[23:16]), .ovf_o(counter_2_ovf));

    // counter_3: นับ bit[31:24] — enable เมื่อ counter_2 overflow
    counter_8bit counter_3 (.clk_i, .rst_ni, .en_i(counter_2_ovf),
        .count_o(count_o[31:24]), .ovf_o(ovf_o));
endmodule
```

นี่คือ **Ripple Carry Counter** — overflow ของ stage หนึ่งเป็น enable ของ stage ถัดไป ทำให้เราใช้ counter_8bit ซ้ำ 4 ครั้งได้โดยไม่ต้องเขียน logic ใหม่

1. เปลี่ยน directory ไปที่ exercise 4:
   ```bash
   cd exercise_4
   ```

---

### 4.1 เลือก PDN Integration Method และ Implement Macro

#### Power Distribution Network (PDN) คืออะไร?

**PDN** คือโครงข่ายสาย metal ที่ใช้กระจายแรงดันไฟ (VPWR) และ ground (VGND) ให้กับ standard cell ทุกตัวใน chip

IHP SG13G2 มี metal layer ทั้งหมด 7 ชั้น LibreLane แบ่งการใช้งานดังนี้:

```
TopMetal2  ← แนวตั้ง  — ใช้สำหรับ Power strap ของ top-level (ใหญ่สุด)
TopMetal1  ← แนวนอน  — ใช้สำหรับ Power strap
Metal5–1   ← Signal routing และ local power rails ของ standard cell
```

**ปัญหา:** เมื่อรวม macro เข้า top-level ต้องเชื่อม VPWR/VGND ของ macro เข้ากับ PDN ของ top-level ซึ่งทำได้ 2 วิธี:

#### วิธีที่ 1: Hierarchical Method

```
Top-level PDN:
  TopMetal2 (แนวตั้ง) ←── power strap หลัก
  TopMetal1 (แนวนอน) ←── power strap หลัก
                            ↕ via ตรงที่ overlap กับ macro PDN
Macro PDN:
  TopMetal1 (แนวนอน) ←── macro ใช้แค่ชั้นเดียว
  (TopMetal2 สงวนไว้ให้ top-level เดินผ่าน)
```

**ข้อดี:** ไม่ต้องใช้พื้นที่เพิ่มสำหรับ ring  
**ข้อเสีย:** macro ต้องหยุดใช้ TopMetal2 — signal routing ใน macro มีเพียง Metal1–TopMetal1 (ลด routing resource)  
**Config:** `PDN_MULTILAYER: false` + `RT_MAX_LAYER: TopMetal1`

#### วิธีที่ 2: Ring Method

```
Top-level PDN:
  TopMetal2 (แนวตั้ง) ←── power strap หลัก
  TopMetal1 (แนวนอน) ←── power strap หลัก
                            ↓ เชื่อมกับ ring รอบ macro
Macro PDN:
  [TopMetal2 + TopMetal1 ring รอบขอบ macro]  ←── power ring
  TopMetal2 (แนวตั้ง) ←── ใช้ได้เต็มที่ภายใน macro
  TopMetal1 (แนวนอน) ←── ใช้ได้เต็มที่ภายใน macro
```

**ข้อดี:** macro ใช้ metal layer ได้เต็มทั้ง 7 ชั้น — routing ภายใน macro ยืดหยุ่นกว่า  
**ข้อเสีย:** ต้องใช้พื้นที่เพิ่มรอบขอบ macro สำหรับ ring  
**Config:** `FP_PDN_CORE_RING: true`

#### ภาพจาก OpenROAD: Default PDN ของ `counter_8bit` (ก่อนเลือก method)

ใน `openroad_1.png` สังเกตว่า:
- **สีฟ้าอ่อนแนวตั้ง (TopMetal1)** — vertical power strap ของ macro
- **สีชมพูแนวนอน (TopMetal2)** — horizontal power strap ของ macro
- ทั้งคู่ครอบพื้นที่ทั้ง die ของ macro 50×50 µm

> 📝 **หมายเหตุ:** ภาพนี้แสดง default PDN ที่ใช้ทั้ง TopMetal1 และ TopMetal2 — ถ้าเลือก Hierarchical method เราจะตัด TopMetal2 ออกจาก macro

---

#### ขั้นตอน 4.1.1: Implement Macro ด้วย Hierarchical Method

1. เข้าไปใน directory ของ macro:
   ```bash
   cd counter_8bit
   ```

2. ดู config เริ่มต้น:
   ```bash
   cat config.yaml
   ```
   ```yaml
   DESIGN_NAME: counter_8bit
   VERILOG_FILES: dir::counter_8bit.sv
   CLOCK_PORT: clk_i
   CLOCK_PERIOD: 10           # 100 MHz

   FP_SIZING: absolute
   DIE_AREA: [0, 0, 50, 50]  # 50×50 µm — พอดีกับ counter ขนาดเล็ก
   PL_TARGET_DENSITY_PCT: 60

   TOP_MARGIN_MULT: 1
   BOTTOM_MARGIN_MULT: 1
   LEFT_MARGIN_MULT: 6
   RIGHT_MARGIN_MULT: 6
   ```

3. แก้ไข `config.yaml` เพิ่ม Hierarchical method config:
   ```yaml
   DESIGN_NAME: counter_8bit
   VERILOG_FILES: dir::counter_8bit.sv
   CLOCK_PORT: clk_i
   CLOCK_PERIOD: 10

   FP_SIZING: absolute
   DIE_AREA: [0, 0, 50, 50]
   PL_TARGET_DENSITY_PCT: 60

   TOP_MARGIN_MULT: 1
   BOTTOM_MARGIN_MULT: 1
   LEFT_MARGIN_MULT: 6
   RIGHT_MARGIN_MULT: 6

   # Hierarchical PDN method
   PDN_MULTILAYER: false    # ใช้แค่ชั้นเดียว (TopMetal1) ไม่ใช้ TopMetal2
   RT_MAX_LAYER: TopMetal1  # ห้าม router ใช้ TopMetal2 — สงวนไว้ให้ top-level
   ```

4. รัน flow:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml
   ```

5. รอจนเสร็จ และตรวจสอบผลลัพธ์:
   ```
   * Antenna  Passed ✅
   * LVS      Passed ✅
   * DRC      Passed ✅
   ```

6. ดูผลใน OpenROAD GUI:
   ```bash
   librelane --pdk ihp-sg13g2 config.yaml --last-run --flow OpenInOpenROAD
   ```
   สังเกตว่าตอนนี้มีเพียง TopMetal1 (แนวนอน/ฟ้าอ่อน) เป็น power strap — TopMetal2 หายไปแล้ว

> ✅ **จุดตรวจสอบ:** DRC, LVS ต้องผ่านทั้งคู่ก่อน copy final

#### ขั้นตอน 4.1.2 (ทางเลือก): Implement ด้วย Ring Method

ถ้าต้องการลองวิธี Ring ให้แก้ไข config เป็น:
```yaml
# Ring PDN method (แทนที่ค่าจาก Hierarchical)
FP_PDN_CORE_RING: true
# ลบ PDN_MULTILAYER และ RT_MAX_LAYER ออก หรือ comment out
```

แล้วรัน flow ใหม่ ดูความแตกต่างของ power structure ใน OpenROAD GUI

---

#### ขั้นตอน 4.1.3: Copy ไฟล์ `final/` ออกมา

หลังจาก flow ผ่านแล้ว ต้อง copy โฟลเดอร์ `final/` ออกมาไว้ใน `counter_8bit/` เพื่อให้ top-level design ใช้งานได้:

1. หา run directory ล่าสุด:
   ```bash
   ls -t run/ | head -1
   ```
   สมมติได้ `RUN_2026-07-18_11-00-00`

2. ตรวจสอบว่า `final/` มีไฟล์ครบ:
   ```bash
   ls run/RUN_2026-07-18_11-00-00/final/
   ```
   ควรเห็น:
   ```
   gds/    lef/    nl/    lib/    spef/
   ```

3. Copy `final/` เข้ามาใน `counter_8bit/`:
   ```bash
   cp -r run/RUN_2026-07-18_11-00-00/final/ .
   ```

4. ตรวจสอบโครงสร้างที่ได้:
   ```bash
   ls final/
   # gds/  lef/  lib/  nl/  spef/

   ls final/gds/
   # counter_8bit.gds

   ls final/lef/
   # counter_8bit.lef

   ls final/lib/
   # nom_fast_1p32V_m40C/  nom_slow_1p08V_125C/  nom_typ_1p20V_25C/

   ls final/lib/nom_typ_1p20V_25C/
   # counter_8bit__nom_typ_1p20V_25C.lib

   ls final/spef/nom/
   # counter_8bit.nom.spef
   ```

> ✅ **จุดตรวจสอบ:** ต้องมี `counter_8bit/final/gds/`, `counter_8bit/final/lef/`, `counter_8bit/final/nl/`, `counter_8bit/final/lib/` (3 corners), และ `counter_8bit/final/spef/nom/` ก่อนไปขั้นตอนถัดไป

5. กลับไปที่ `exercise_4/`:
   ```bash
   cd ..
   ```

---

### 4.2 รวม Macro 4 ตัวเข้าใน Top-level Design

#### ทำความเข้าใจ `config.yaml` ของ Top-level

ดู config เริ่มต้นของ `exercise_4/config.yaml`:

```yaml
DESIGN_NAME: counter_32bit
VERILOG_FILES: dir::counter_32bit.sv
CLOCK_PORT: clk_i
CLOCK_PERIOD: 10

FP_SIZING: absolute
DIE_AREA: [0, 0, 175, 175]       # ใหญ่พอสำหรับ macro 4 ตัว (50×50 µm) + routing
PL_TARGET_DENSITY_PCT: 60

# Top-level เป็น "wrapper" เท่านั้น ไม่มี standard cell ใหม่
SYNTH_ELABORATE_ONLY: true        # synthesis แค่ elaborate (parse netlist) ไม่ synthesize ใหม่
FP_PDN_ENABLE_RAILS: false        # ไม่สร้าง power rail สำหรับ standard cell (เพราะไม่มี cell)
RUN_CTS: false                    # ไม่ต้องทำ CTS ที่ top-level (macro มี clock tree ของตัวเอง)
PL_RESIZER_DESIGN_OPTIMIZATIONS: false   # ไม่ต้อง resize ที่ top-level
PL_RESIZER_TIMING_OPTIMIZATIONS: false
GLB_RESIZER_DESIGN_OPTIMIZATIONS: false
GLB_RESIZER_TIMING_OPTIMIZATIONS: false
PL_RESIZER_BUFFER_INPUT_PORTS: false
RUN_FILL_INSERTION: false         # ไม่ต้อง fill ที่ top-level
```

> 📝 **หมายเหตุ:** `SYNTH_ELABORATE_ONLY: true` ทำให้ Yosys แค่อ่าน Verilog และสร้าง module hierarchy โดยไม่แปลง logic เป็น gate — เพราะ logic ทั้งหมดอยู่ใน macro แล้ว top-level แค่เชื่อม wire

#### ขั้นตอน 4.2.1: เพิ่ม MACROS Block ใน config

แก้ไขไฟล์ `exercise_4/config.yaml` เพิ่ม MACROS block ต่อจากที่มีอยู่:

```yaml
DESIGN_NAME: counter_32bit
VERILOG_FILES: dir::counter_32bit.sv
CLOCK_PORT: clk_i
CLOCK_PERIOD: 10

FP_SIZING: absolute
DIE_AREA: [0, 0, 175, 175]
PL_TARGET_DENSITY_PCT: 60

SYNTH_ELABORATE_ONLY: true
FP_PDN_ENABLE_RAILS: false
RUN_CTS: false
PL_RESIZER_DESIGN_OPTIMIZATIONS: false
PL_RESIZER_TIMING_OPTIMIZATIONS: false
GLB_RESIZER_DESIGN_OPTIMIZATIONS: false
GLB_RESIZER_TIMING_OPTIMIZATIONS: false
PL_RESIZER_BUFFER_INPUT_PORTS: false
RUN_FILL_INSERTION: false

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
        location: [20, 20]     # มุมซ้ายล่าง
        orientation: N
      counter_1:
        location: [20, 105]    # มุมซ้ายบน
        orientation: N
      counter_2:
        location: [105, 20]    # มุมขวาล่าง
        orientation: N
      counter_3:
        location: [105, 105]   # มุมขวาบน
        orientation: N
```

**อธิบาย key ใน MACROS block:**

| Key | ความหมาย |
|---|---|
| `counter_8bit` (ชื่อ key) | ต้องตรงกับชื่อ module ใน Verilog (`module counter_8bit`) |
| `gds` | ไฟล์ GDS ของ macro — ใช้ตอน stream out GDS สุดท้าย |
| `lef` | Abstract view — router ใช้รู้ขอบเขตและ pin location |
| `nl` | Gate-level netlist — ใช้ใน LVS |
| `lib` | Timing model 3 corners: typical (25°C, 1.2V), fast (-40°C, 1.32V), slow (125°C, 1.08V) |
| `spef` | Extracted parasitics — ใช้ใน post-layout STA |
| `instances` | รายชื่อ instance ในภาษา Verilog พร้อม location (µm) และ orientation |

**Orientation ที่ใช้บ่อย:**
- `N` (North) — ปกติ ไม่มี flip หรือ rotate
- `S` (South) — หมุน 180°
- `FN` (Flip North) — mirror แนวแกน Y
- `FS` (Flip South) — mirror แนวแกน Y แล้วหมุน 180°

#### ขั้นตอน 4.2.2: เพิ่ม PDN Connections

เพิ่มต่อจาก MACROS block:

```yaml
PDN_MACRO_CONNECTIONS:
  - "counter_0 VPWR VGND VPWR VGND"
  - "counter_1 VPWR VGND VPWR VGND"
  - "counter_2 VPWR VGND VPWR VGND"
  - "counter_3 VPWR VGND VPWR VGND"
```

รูปแบบของแต่ละ entry: `"<instance_name> <macro_power_pin> <macro_gnd_pin> <top_power_net> <top_gnd_net>"`

ความหมาย: "เชื่อม VPWR pin ของ counter_0 เข้ากับ net VPWR ของ top-level และเชื่อม VGND pin เข้ากับ net VGND"

#### ขั้นตอน 4.2.3: ลด PDN Strap Pitch

ถ้ารัน flow แล้วได้ error ระหว่าง PDN generation ให้เพิ่ม:

```yaml
FP_PDN_VPITCH: 20    # default ~40 µm — ลดเหลือ 20 µm ให้ strap หนาแน่นขึ้น
FP_PDN_HPITCH: 20    # ทำให้ macro มีโอกาสถูก strap ผ่านมากขึ้น
```

> 📝 **หมายเหตุ:** pitch คือระยะห่างระหว่าง power strap ที่อยู่ติดกัน ยิ่ง pitch น้อย strap ยิ่งถี่ ทำให้ macro ที่ขนาดเล็ก (50×50 µm) มีโอกาสมีอย่างน้อย 1 strap ผ่านกลาง — จำเป็นสำหรับการเชื่อม power

#### ขั้นตอน 4.2.4: รัน Top-level Flow

```bash
librelane --pdk ihp-sg13g2 config.yaml
```

> ⚠️ **คำเตือน:** ถ้าได้ error `[ERROR] No PDN stripe covers macro counter_X` ให้เพิ่ม `FP_PDN_VPITCH: 20` และ `FP_PDN_HPITCH: 20` แล้วรันใหม่

#### ขั้นตอน 4.2.5: ดูผลลัพธ์ใน OpenROAD GUI

```bash
librelane --pdk ihp-sg13g2 config.yaml --last-run --flow OpenInOpenROAD
```

#### ทำความเข้าใจภาพผลลัพธ์ทั้งสองวิธี

**Ring Method** (`openroad_2.png`):
- เห็น macro 4 ตัว (`counter_0`–`counter_3`) วางตาม location ที่กำหนด
- **สีชมพู/ม่วง** รอบขอบ macro — power ring (TopMetal1 + TopMetal2)
- **Power strap สีเขียวและฟ้า** พาดระหว่าง macro บน top-level
- สังเกตว่า ring ทำให้ macro มีขอบหนากว่าตัว cell จริง

**Hierarchical Method** (`openroad_3.png`):
- layout คล้ายกัน แต่ **ไม่มี ring** รอบขอบ macro
- **TopMetal2 strap แนวตั้ง** จาก top-level พาดลงมาผ่านกลาง macro โดยตรง
- ประหยัด area มากกว่า แต่ routing ภายใน macro ถูกจำกัดที่ TopMetal1

ทั้งสองวิธีให้ผลที่ DRC และ LVS ผ่านเหมือนกัน ความแตกต่างอยู่ที่ area efficiency และ routing flexibility ภายใน macro

> ✅ **จุดตรวจสอบ:** ใน OpenROAD GUI ควรเห็น instance label `counter_0`, `counter_1`, `counter_2`, `counter_3` แสดงในตำแหน่งที่กำหนด

---

### 4.3 แบบฝึกหัดเพิ่มเติม: SRAM Macro จาก IHP PDK

Exercise นี้ท้าทายกว่า — ลองรวม SRAM macro ที่มาจาก IHP PDK โดยตรง (ไม่ต้อง implement เอง)

#### ทำไมต้องใช้ SRAM?

SRAM (Static RAM) ที่ออกแบบเป็น macro โดยโรงงาน มีความหนาแน่นสูงกว่าการสร้าง memory จาก flip-flop มาก ตัวอย่าง:
- SRAM 1024×8 bit = 8192 bit ใน area ประมาณ 100×100 µm
- ถ้าใช้ flip-flop แทน: 8192 FF × ~1.5 µm² ≈ พื้นที่มากกว่า 10 เท่า

#### สำรวจ SRAM ที่มีใน PDK

```bash
ls ~/.ciel/libs.ref/sg13g2_sram/
# doc/  gds/  lef/  lib/  verilog/

ls ~/.ciel/libs.ref/sg13g2_sram/lef/
# RM_IHPSG13_1P_1024x8_c2_bm_bist.lef  ← SRAM 1024 words × 8-bit พร้อม BIST
# RM_IHPSG13_1P_256x48_c2_bm_bist.lef  ← อื่น ๆ
# ...
```

#### ดูข้อมูลของ SRAM ที่เลือก

```bash
cat ~/.ciel/libs.ref/sg13g2_sram/doc/RM_IHPSG13_1P_1024x8_c2_bm_bist.txt
```

SRAM นี้มี port สำคัญ:

| Port | ความหมาย |
|---|---|
| `A_CLK` | Clock |
| `A_MEN` | Memory enable (active high) |
| `A_WEN` | Write enable |
| `A_REN` | Read enable |
| `A_ADDR[9:0]` | Address (10-bit → 1024 locations) |
| `A_DIN[7:0]` | Data input |
| `A_DOUT[7:0]` | Data output |
| `A_BM[7:0]` | Byte mask (1 = write bit) |
| `A_DLY` | Delay setting — **ต้องผูกไว้กับ 1 เสมอ** |
| `A_BIST_*` | Built-In Self-Test port |

#### ดู `top.sv` — Design สำหรับ SRAM Exercise

ไฟล์ `top.sv` ที่มีให้เป็นตัวอย่างการใช้ SRAM:
```systemverilog
module top (input logic clk_i, rst_ni, ...);
    // สร้าง counter เพื่อ generate address
    logic [9:0] mem_addr;
    always_ff @(posedge clk_i) begin
        if (!rst_ni) mem_addr <= '0;
        else         mem_addr <= mem_addr + 1; // sweep address 0→1023→0→...
    end

    // เขียน pattern 0xDE, 0xAD, 0xBE, 0xEF วนซ้ำตาม address[1:0]
    logic mem_din;
    always_comb begin
        case (mem_addr[1:0])
            2'b00: mem_din = 8'hDE;
            2'b01: mem_din = 8'hAD;
            2'b10: mem_din = 8'hBE;
            2'b11: mem_din = 8'hEF;
        endcase
    end

    // Instance ของ SRAM macro
    RM_IHPSG13_1P_1024x8_c2_bm_bist sram (
        .A_CLK(clk_i), .A_MEN(1'b1), .A_WEN(1'b1), .A_REN(1'b1),
        .A_ADDR(mem_addr), .A_DIN(mem_din), .A_BM(8'hFF),
        .A_BIST_CLK(clk_i), .A_BIST_EN(bist_en_i), ...
        .A_DOUT(dout_i),
        .A_DLY(1'b1)  // สำคัญมาก: ผูก A_DLY = 1 เสมอ!
    );
endmodule
```

#### Config สำหรับรวม SRAM Macro

สร้างไฟล์ config ใหม่สำหรับ top design ที่ใช้ SRAM (หรือแก้ไข `config.yaml` ที่มี):

```yaml
DESIGN_NAME: top
VERILOG_FILES: dir::top.sv
CLOCK_PORT: clk_i
CLOCK_PERIOD: 10

FP_SIZING: absolute
DIE_AREA: [0, 0, 300, 300]   # SRAM ใหญ่กว่า counter_8bit มาก ต้องการพื้นที่มากขึ้น
PL_TARGET_DENSITY_PCT: 60

SYNTH_ELABORATE_ONLY: true
FP_PDN_ENABLE_RAILS: false
RUN_CTS: false
PL_RESIZER_DESIGN_OPTIMIZATIONS: false
PL_RESIZER_TIMING_OPTIMIZATIONS: false
GLB_RESIZER_DESIGN_OPTIMIZATIONS: false
GLB_RESIZER_TIMING_OPTIMIZATIONS: false
PL_RESIZER_BUFFER_INPUT_PORTS: false
RUN_FILL_INSERTION: false

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
        orientation: E    # หมุน 90° เพื่อให้ pin อยู่ทางซ้าย

PDN_MACRO_CONNECTIONS:
  - "top.sram VPWR VGND VDD! VSS!"        # core power
  - "top.sram VPWR VGND VDDARRAY! VSS!"   # array power (SRAM มี 2 domain)

FP_PDN_VPITCH: 20
FP_PDN_HPITCH: 20
```

> 📝 **หมายเหตุ:** `pdk_dir::` คือ path prefix สำหรับ reference ไฟล์จาก PDK root (`~/.ciel/`) แทนที่จะเป็น directory ของ config ต่างจาก `dir::` ที่ใช้กับไฟล์ใน project เรา

> ⚠️ **คำเตือน:** SRAM มี 2 power domain คือ `VDD!` (core logic) และ `VDDARRAY!` (memory array) ต้องเชื่อมทั้งคู่ ถ้าขาดตัวใดตัวหนึ่งจะ LVS fail

---

### การแก้ไขปัญหา

| อาการ | สาเหตุ | วิธีแก้ |
|---|---|---|
| `[ERROR] Cannot find macro GDS/LEF` | Path ใน MACROS block ผิด | ตรวจว่าไฟล์ใน `counter_8bit/final/` มีจริง และ path ถูกต้อง |
| `final/` directory ไม่มีหลัง cp | ลืม copy หรือ copy ผิดที่ | ตรวจว่า flow ผ่านแล้ว และ cp จาก `run/RUN_.../final/` ลงมาใน `counter_8bit/` |
| `[ERROR] No PDN stripe covers macro` | PDN strap pitch ห่างเกิน macro ไม่มี strap ผ่าน | เพิ่ม `FP_PDN_VPITCH: 20` และ `FP_PDN_HPITCH: 20` |
| `[ERROR] Macro overlap` | Macro วางทับกัน | เช็คว่า location ห่างกันอย่างน้อย 50 µm (ขนาด macro) + margin |
| LVS fail: `net VPWR unconnected` | ลืม `PDN_MACRO_CONNECTIONS` หรือชื่อผิด | ตรวจชื่อ instance และ power pin ใน LEF ของ macro |
| SRAM LVS fail: `VDD! missing` | SRAM มี 2 power domain แต่ connect แค่ 1 | ต้องมี entry สำหรับ `VDD!` และ `VDDARRAY!` ทั้งคู่ |
| DRC fail ใน macro | ใช้ Hierarchical method แต่ macro routing ใช้ TopMetal2 | ตรวจว่า `RT_MAX_LAYER: TopMetal1` อยู่ใน macro config ก่อน implement |
| `[ERROR] Module counter_8bit not found` | ชื่อ key ใน MACROS ไม่ตรงกับชื่อ module | key ใน `MACROS:` ต้องตรงกับ `module counter_8bit` ใน Verilog ทุกตัวอักษร |

---

## Lab 5: LibreLane Python API

**เวลาโดยประมาณ:** 60–90 นาที  
**Objective:** เข้าใจโครงสร้าง LibreLane Python API, รัน Classic flow ด้วย Python script, สร้าง custom flow class ด้วย inheritance, สร้าง custom step พร้อม config variable, และต่อยอดเป็น step ที่ใช้งานได้จริง

---

### พื้นหลัง: ทำไมต้องใช้ Python API?

การรัน `librelane --pdk ihp-sg13g2 config.yaml` จากบรรทัดคำสั่งสะดวกสำหรับการใช้งานทั่วไป แต่มีข้อจำกัดเมื่อต้องการ:

- **Parameter sweep** — รัน flow หลายรอบอัตโนมัติด้วย config ต่างกัน เช่น ทดสอบ `DIE_AREA` หลายขนาดแล้วเปรียบเทียบ area และ timing
- **Parallel runs** — รัน design หลาย variant พร้อมกันบน machine ที่มี CPU หลาย core
- **Custom step logic** — เพิ่ม script ประมวลผลผลลัพธ์, ส่ง notification, หรือเชื่อมกับ CI/CD pipeline
- **Flow customization** — ใส่, ลบ, หรือแทน step ใน flow ตามความต้องการเฉพาะของ project

#### โครงสร้าง Python API หลัก ๆ

```
librelane/
├── flows/
│   ├── flow.py          ← class Flow (ABC) — base ของทุก flow
│   ├── sequential.py    ← class SequentialFlow — flow แบบ step-by-step
│   └── classic.py       ← class Classic(SequentialFlow) — flow มาตรฐาน
├── steps/
│   └── step.py          ← class Step (ABC) — base ของทุก step
├── config/
│   └── variable.py      ← class Variable — นิยาม config variable
├── state/
│   └── state.py         ← class State — state ที่ส่งระหว่าง step
└── logging.py           ← info(), warn(), err() ฯลฯ
```

**ความสัมพันธ์หลัก:**

```
Flow
 └─ มี Steps: List[Type[Step]]
     └─ แต่ละ Step มี:
         ├─ id: str          ("Module.StepName")
         ├─ name: str        (ชื่อสำหรับแสดงผล)
         ├─ config_vars      (Variables ที่ step นี้ต้องการ)
         ├─ inputs           (DesignFormat ที่รับเข้ามา)
         ├─ outputs          (DesignFormat ที่ส่งออกไป)
         └─ run(state_in) → (ViewsUpdate, MetricsUpdate)
```

1. เปลี่ยน directory ไปที่ exercise 5:
   ```bash
   cd exercise_5
   ```

---

### 5.1 อ่านและรัน `flow.py` ที่มีให้

#### ทำความเข้าใจไฟล์ `flow.py` ทีละบรรทัด

```python
# ─── imports ─────────────────────────────────────────────────────
import os
from typing import List, Type, Tuple

# Import classes ที่จำเป็น — ทั้งหมดนี้จะใช้ในขั้นตอนต่อไป
from librelane.steps import Step
from librelane.steps.step import ViewsUpdate, MetricsUpdate
from librelane.flows import Flow, FlowError
from librelane.flows.classic import Classic
from librelane.state.state import State
from librelane.config.variable import Variable
from librelane.logging import info
# ─────────────────────────────────────────────────────────────────

def main():
    # กำหนด source files — เป็น list ของ path (relative to design_dir)
    verilog_files = [
        "counter.sv"
    ]

    # flow_cfg เทียบเท่ากับ config.yaml — key คือชื่อ variable, value คือค่า
    flow_cfg = {
        "DESIGN_NAME"           : "counter",
        "VERILOG_FILES"         : verilog_files,
        "CLOCK_PORT"            : "clk_i",
        "CLOCK_PERIOD"          : 10,          # หน่วย ns → 100 MHz
        "FP_SIZING"             : "absolute",
        "DIE_AREA"              : [0, 0, 75, 75],  # หน่วย µm
        "PL_TARGET_DENSITY_PCT" : 60,
    }

    # สร้าง flow object
    # - flow_cfg : dict ของ config variables (เทียบเท่า config.yaml)
    # - design_dir : directory ที่มี source files (. = current dir)
    # - pdk_root : None ให้ LibreLane หา PDK เองใน ~/.ciel
    # - pdk : ชื่อ PDK family
    flow = Classic(
        flow_cfg,
        design_dir = ".",
        pdk_root   = None,
        pdk        = "ihp-sg13g2",
    )

    # เริ่มรัน flow — เทียบเท่ากับคำสั่ง librelane บน terminal
    # flow.start() returns State object ของผลลัพธ์สุดท้าย
    flow.start()

if __name__ == "__main__":
    main()
```

**สิ่งสำคัญที่ต้องสังเกต:**
- `flow_cfg` เป็น Python `dict` แทน YAML file — ทำให้ใส่ logic Python ได้โดยตรง
- `Classic(flow_cfg, ...)` สร้าง flow object แต่ยังไม่รัน
- `flow.start()` คือจุดที่เริ่มรัน flow จริง ๆ และ return `State` ของผลลัพธ์

#### ขั้นตอน: เปิดใช้ PDK และรัน flow.py

1. เปิดใช้งาน PDK ก่อน (จำเป็นเมื่อใช้ API แทน CLI):
   ```bash
   ciel enable --pdk-family ihp-sg13g2 cb7daaa8901016cf7c5d272dfa322c41f024931f
   ```

   > 📝 **หมายเหตุ:** เมื่อใช้คำสั่ง `librelane` บน terminal ตัว CLI จัดการ PDK ให้อัตโนมัติ แต่เมื่อใช้ Python API ต้อง enable PDK เองก่อนเสมอ

2. รันสคริปต์:
   ```bash
   python3 flow.py
   ```

3. ผลลัพธ์บน terminal จะเหมือนกับการรัน `librelane --pdk ihp-sg13g2 config.yaml` ทุกประการ รวมถึง progress bar และ summary สุดท้าย

> ✅ **จุดตรวจสอบ:** ต้องเห็น `DRC Passed ✅`, `LVS Passed ✅` เหมือน Lab 1

#### ประโยชน์ของการใช้ Python: Parameter Sweep

ตัวอย่างการรันหลาย config อัตโนมัติ (ไม่ต้องทำใน exercise นี้ — แค่ดูแนวคิด):

```python
# ทดสอบ DieArea หลายขนาดอัตโนมัติ
die_sizes = [(60, 60), (70, 70), (75, 75), (80, 80)]

for w, h in die_sizes:
    cfg = {
        "DESIGN_NAME": "counter",
        "VERILOG_FILES": ["counter.sv"],
        "CLOCK_PORT": "clk_i",
        "CLOCK_PERIOD": 10,
        "FP_SIZING": "absolute",
        "DIE_AREA": [0, 0, w, h],
        "PL_TARGET_DENSITY_PCT": 60,
    }
    flow = Classic(cfg, design_dir=".", pdk_root=None, pdk="ihp-sg13g2")
    result = flow.start(tag=f"run_{w}x{h}")  # tag กำหนดชื่อ run directory
    print(f"Die {w}x{h}: completed")
```

---

### 5.2 สร้าง Custom Flow ด้วย Inheritance

#### ทำความเข้าใจ Flow Factory และ Decorator

`@Flow.factory.register()` คือ Python decorator ที่:
1. รับ class ที่อยู่ข้างล่าง
2. ลงทะเบียนใน flow registry ของ LibreLane ด้วย class name เป็น key
3. Return class นั้นกลับมา (class ไม่เปลี่ยน)

หลังจาก register แล้ว สามารถเรียกใช้ flow ด้วยชื่อ string ได้ เช่น `Flow.factory.get("HeiChipsFlow")`

#### ขั้นตอน: สร้าง HeiChipsFlow

เปิด `flow.py` ด้วย text editor และเพิ่ม class นี้ **ก่อน** `def main():`

```python
# ─── Custom Flow Definition ───────────────────────────────────────
@Flow.factory.register()          # ลงทะเบียนใน LibreLane flow registry
class HeiChipsFlow(Classic):      # inherit ทุกอย่างจาก Classic
    """
    HeiChips 2026 custom flow — based on Classic with room to customize.
    """
    # กำหนด Steps list — ตอนนี้ใช้ Steps เดิมของ Classic ทั้งหมด
    Steps = Classic.Steps
# ─────────────────────────────────────────────────────────────────
```

แก้ไข `main()` ให้ใช้ `HeiChipsFlow` แทน `Classic`:

```python
def main():
    verilog_files = ["counter.sv"]

    flow_cfg = {
        "DESIGN_NAME"           : "counter",
        "VERILOG_FILES"         : verilog_files,
        "CLOCK_PORT"            : "clk_i",
        "CLOCK_PERIOD"          : 10,
        "FP_SIZING"             : "absolute",
        "DIE_AREA"              : [0, 0, 75, 75],
        "PL_TARGET_DENSITY_PCT" : 60,
    }

    # เปลี่ยนจาก Classic เป็น HeiChipsFlow
    flow = HeiChipsFlow(
        flow_cfg,
        design_dir = ".",
        pdk_root   = None,
        pdk        = "ihp-sg13g2",
    )

    flow.start()
```

รันอีกครั้ง:
```bash
python3 flow.py
```

ผลลัพธ์ควรเหมือนเดิมทุกประการ — เพราะ `HeiChipsFlow` ยังใช้ `Steps = Classic.Steps` ทั้งหมด แต่ตอนนี้เรามี class ของตัวเองที่พร้อมปรับแต่งได้แล้ว

> ✅ **จุดตรวจสอบ:** Title bar ที่ terminal header อาจแสดงว่ากำลังรัน "HeiChipsFlow" แทน "Classic"

---

### 5.3 สร้าง Custom Step — HelloWorldStep

#### ทำความเข้าใจโครงสร้างของ Step

Step ทุกตัวใน LibreLane มีส่วนประกอบดังนี้:

```python
@Step.factory.register()
class MyStep(Step):
    # ── Identity ──────────────────────────────────
    id   = "Module.StepName"   # ชื่อที่ใช้ใน --to, --from, --skip
    name = "Display Name"      # ชื่อที่แสดงบน terminal และ progress bar

    # ── Config Variables ──────────────────────────
    config_vars = [
        Variable(
            "VAR_NAME",        # ชื่อ variable (ตัวใหญ่)
            type,              # Python type: str, int, float, bool, Path, ...
            "Description",     # คำอธิบาย (สำหรับ documentation)
            default=...,       # ค่า default (ถ้าไม่มีจะเป็น required)
        ),
    ]

    # ── State I/O ─────────────────────────────────
    inputs  = [DesignFormat.ODB, DesignFormat.SDC]  # ไฟล์ที่ต้องมีใน state_in
    outputs = [DesignFormat.ODB]                    # ไฟล์ที่จะเพิ่มใน state_out

    # ── Main Logic ────────────────────────────────
    def run(self, state_in: State, **kwargs) -> Tuple[ViewsUpdate, MetricsUpdate]:
        # ทำงานอะไรก็ได้ที่นี่
        # state_in["odb"] → path ของ ODB file จาก step ก่อนหน้า
        # self.config["VAR_NAME"] → ค่าของ config variable
        # self.step_dir → directory ของ step นี้ใน run folder

        views_update  = {}  # dict ของ DesignFormat → Path (ถ้ามี output ใหม่)
        metrics_update = {}  # dict ของ metric name → value
        return views_update, metrics_update
```

**`ViewsUpdate`** คือ dict ที่ map `DesignFormat` → `Path` — บอกว่า step นี้สร้างไฟล์ใหม่อะไรที่จะส่งต่อไปยัง step ถัดไป

**`MetricsUpdate`** คือ dict ที่ map `str` → `value` — บอกว่า step นี้วัดค่าอะไรได้บ้าง (เช่น cell count, wire length ฯลฯ)

ถ้า step ไม่สร้างไฟล์ใหม่และไม่มี metric → `return {}, {}`

#### ขั้นตอน: เพิ่ม HelloWorldStep ใน flow.py

เพิ่ม class นี้ **ก่อน** `HeiChipsFlow` (เพราะ HeiChipsFlow จะอ้างอิง HelloWorldStep):

```python
# ─── Custom Step Definition ───────────────────────────────────────
@Step.factory.register()
class HelloWorldStep(Step):
    """
    A simple custom step that prints a greeting message.
    ใช้สำหรับสาธิต custom step — ไม่ได้แก้ไข state ใด ๆ
    """

    # Step identity — ต้องไม่ซ้ำกับ step อื่นในทั้ง flow
    id   = "HeiChips.HelloWorld"
    name = "Hello World!"

    # Config variable เดียว — เป็น string พร้อม default value
    config_vars = [
        Variable(
            "HEICHIPS_SAY",                 # ชื่อ — ใช้ใน flow_cfg และ self.config
            str,                            # type: ต้องเป็น string
            "A string of what to say.",     # description
            default="How are you?",         # ค่า default ถ้าไม่ได้ระบุ
        ),
    ]

    # Step นี้ไม่ต้องการ input file และไม่สร้าง output file
    inputs  = []
    outputs = []

    def run(self, state_in: State, **kwargs) -> Tuple[ViewsUpdate, MetricsUpdate]:
        # อ่านค่า config variable ด้วย self.config["KEY"]
        message = self.config["HEICHIPS_SAY"]

        # info() พิมพ์ข้อความที่ระดับ INFO — จะแสดงบน terminal
        info(f"Hello World! {message}")

        # ไม่มี views หรือ metrics ที่ต้องอัปเดต
        return {}, {}
# ─────────────────────────────────────────────────────────────────
```

#### ขั้นตอน: เพิ่ม HelloWorldStep เข้า HeiChipsFlow

แก้ไข `HeiChipsFlow` ให้เพิ่ม step ใหม่:

```python
@Flow.factory.register()
class HeiChipsFlow(Classic):
    """
    HeiChips 2026 custom flow — Classic + HelloWorldStep at the end.
    """
    # Classic.Steps + [HelloWorldStep] = ทุก step เดิม + step ใหม่ที่ท้ายสุด
    Steps = Classic.Steps + [HelloWorldStep]
```

#### ขั้นตอน: รันและดูผลลัพธ์

```bash
python3 flow.py
```

สังเกตที่ท้าย terminal output หลังจาก flow เสร็จ:

```
──────────────────────── Hello World! ────────────────────────
[19:51:23] VERBOSE  Running 'HeiChips.HelloWorld' at         step.py:1140
                    'runs/RUN_.../76-heichips-helloworld'…
[19:51:23] INFO     Hello World! How are you?
[19:51:23] INFO     Saving views to '...'
[19:51:23] INFO     Flow complete.
```

> ✅ **จุดตรวจสอบ:** ต้องเห็นบรรทัด `Hello World! How are you?` ที่ท้าย output ก่อน `Flow complete.`

#### ขั้นตอน: ปรับค่า HEICHIPS_SAY ผ่าน flow_cfg

เพิ่ม key ใน `flow_cfg` ใน `main()`:

```python
flow_cfg = {
    "DESIGN_NAME"           : "counter",
    "VERILOG_FILES"         : verilog_files,
    "CLOCK_PORT"            : "clk_i",
    "CLOCK_PERIOD"          : 10,
    "FP_SIZING"             : "absolute",
    "DIE_AREA"              : [0, 0, 75, 75],
    "PL_TARGET_DENSITY_PCT" : 60,

    # ค่าใหม่สำหรับ HelloWorldStep
    "HEICHIPS_SAY"          : "สวัสดี HeiChips 2026!",
}
```

รันอีกครั้ง:
```bash
python3 flow.py
```

ผลลัพธ์ที่คาดหวัง:
```
──────────────────────── Hello World! ────────────────────────
[INFO] Hello World! สวัสดี HeiChips 2026!
[INFO] Flow complete.
```

> 📝 **หมายเหตุ:** `HEICHIPS_SAY` เป็น config variable เหมือนกับ `DIE_AREA` หรือ `CLOCK_PERIOD` ทุกประการ — สามารถใส่ใน `flow_cfg` (Python API) หรือ `config.yaml` (CLI) ก็ได้

---

### 5.4 ต่อยอด: Step ที่แก้ไข State จริง

HelloWorldStep เป็น step ที่ไม่ทำอะไรกับ design จริง ๆ แต่ step จริงจะอ่านและเขียน design files Step ต่อไปนี้แสดงตัวอย่าง step ที่ **อ่านข้อมูลจาก state** และ **รายงาน metric**:

```python
@Step.factory.register()
class ReportCellCountStep(Step):
    """
    Custom step ที่นับ cell ใน netlist และรายงานเป็น metric
    """

    id   = "HeiChips.ReportCellCount"
    name = "Report Cell Count"

    # ต้องการ netlist (nl) เป็น input
    inputs  = [DesignFormat.NL]
    outputs = []   # ไม่สร้างไฟล์ใหม่

    config_vars = []  # ไม่มี config variable เพิ่มเติม

    def run(self, state_in: State, **kwargs) -> Tuple[ViewsUpdate, MetricsUpdate]:
        # อ่าน path ของ netlist file จาก state
        nl_path = state_in[DesignFormat.NL]

        # นับจำนวน cell ใน netlist โดย grep หาบรรทัดที่มี "sg13g2_"
        import subprocess
        result = subprocess.run(
            ["grep", "-c", "sg13g2_", str(nl_path)],
            capture_output=True, text=True
        )
        cell_count = int(result.stdout.strip()) if result.returncode == 0 else 0

        # แสดงผลบน terminal
        info(f"Total standard cells in netlist: {cell_count}")

        # รายงานเป็น metric — จะบันทึกใน state_out.json
        metrics = {
            "design__instance__count__custom": cell_count
        }
        return {}, metrics
```

เพิ่ม `ReportCellCountStep` ใน `HeiChipsFlow`:

```python
@Flow.factory.register()
class HeiChipsFlow(Classic):
    Steps = Classic.Steps + [ReportCellCountStep, HelloWorldStep]
    #                         ↑ รันก่อน            ↑ รันทีหลัง
```

> 📝 **หมายเหตุ:** ต้อง import `DesignFormat` ด้วย:
> ```python
> from librelane.state import DesignFormat
> ```

---

### 5.5 สรุป flow.py ฉบับสมบูรณ์

ด้านล่างคือ `flow.py` ที่รวมทุกอย่างที่เรียนในบทนี้ไว้ด้วยกัน:

```python
# flow.py — HeiChips 2026 LibreLane Python API Example
from typing import Tuple
from librelane.steps import Step
from librelane.steps.step import ViewsUpdate, MetricsUpdate
from librelane.flows import Flow
from librelane.flows.classic import Classic
from librelane.state import DesignFormat
from librelane.state.state import State
from librelane.config.variable import Variable
from librelane.logging import info


# ─── Step 1: Hello World step ────────────────────────────────────
@Step.factory.register()
class HelloWorldStep(Step):
    """Prints a customizable greeting at the end of the flow."""

    id   = "HeiChips.HelloWorld"
    name = "Hello World!"

    config_vars = [
        Variable(
            "HEICHIPS_SAY",
            str,
            "A string of what to say.",
            default="How are you?",
        ),
    ]

    inputs  = []
    outputs = []

    def run(self, state_in: State, **kwargs) -> Tuple[ViewsUpdate, MetricsUpdate]:
        info(f"Hello World! {self.config['HEICHIPS_SAY']}")
        return {}, {}


# ─── Step 2: Cell count reporter ─────────────────────────────────
@Step.factory.register()
class ReportCellCountStep(Step):
    """Counts standard cells in the synthesized netlist."""

    id   = "HeiChips.ReportCellCount"
    name = "Report Cell Count"

    inputs  = [DesignFormat.NL]
    outputs = []
    config_vars = []

    def run(self, state_in: State, **kwargs) -> Tuple[ViewsUpdate, MetricsUpdate]:
        import subprocess
        nl_path = state_in[DesignFormat.NL]
        result = subprocess.run(
            ["grep", "-c", "sg13g2_", str(nl_path)],
            capture_output=True, text=True
        )
        cell_count = int(result.stdout.strip()) if result.returncode == 0 else 0
        info(f"Standard cell count: {cell_count}")
        return {}, {"design__instance__count__custom": cell_count}


# ─── Custom Flow ──────────────────────────────────────────────────
@Flow.factory.register()
class HeiChipsFlow(Classic):
    """Classic flow + custom reporting steps."""
    Steps = Classic.Steps + [ReportCellCountStep, HelloWorldStep]


# ─── Main ─────────────────────────────────────────────────────────
def main():
    verilog_files = ["counter.sv"]

    flow_cfg = {
        "DESIGN_NAME"           : "counter",
        "VERILOG_FILES"         : verilog_files,
        "CLOCK_PORT"            : "clk_i",
        "CLOCK_PERIOD"          : 10,
        "FP_SIZING"             : "absolute",
        "DIE_AREA"              : [0, 0, 75, 75],
        "PL_TARGET_DENSITY_PCT" : 60,
        "HEICHIPS_SAY"          : "สวัสดี HeiChips 2026!",
    }

    flow = HeiChipsFlow(
        flow_cfg,
        design_dir = ".",
        pdk_root   = None,
        pdk        = "ihp-sg13g2",
    )

    flow.start()


if __name__ == "__main__":
    main()
```

---

### 5.6 แนวทางขยายต่อ: Substitution API

LibreLane มี `Substitutions` API ที่ทรงพลังกว่าการ `+` step เข้า list เพียงอย่างเดียว — ให้ **แทนที่**, **ลบ**, **แทรกก่อน/หลัง** step ที่มีอยู่:

```python
@Flow.factory.register()
class MyAdvancedFlow(Classic):
    """
    ตัวอย่าง Substitutions: แทน, ลบ, และแทรก step
    """
    Substitutions = {
        # แทนที่ KLayout.DRC ด้วย step ของตัวเอง
        "KLayout.DRC": MyCustomDRCStep,

        # ลบ step ที่ไม่ต้องการออก (None = ลบ)
        "Magic.DRC": None,

        # แทรก HelloWorldStep หลังจาก Yosys.Synthesis
        "+Yosys.Synthesis": HelloWorldStep,

        # แทรก ReportCellCountStep ก่อน KLayout.DRC
        "-KLayout.DRC": ReportCellCountStep,
    }
```

> 💡 **เคล็ดลับ:** อ่าน [Writing Custom Flows](https://librelane.readthedocs.io/en/latest/usage/writing_custom_flows.html) ใน LibreLane documentation สำหรับ Substitutions API แบบเต็ม

---

### การแก้ไขปัญหา

| อาการ | สาเหตุ | วิธีแก้ |
|---|---|---|
| `ModuleNotFoundError: librelane` | ยังไม่ได้เข้า `nix-shell` | รัน `nix-shell` ก่อนแล้วลองใหม่ |
| `[ERROR] PDK not found` | ลืม enable PDK | รัน `ciel enable --pdk-family ihp-sg13g2 <hash>` |
| `[ERROR] Step 'HeiChips.HelloWorld' already registered` | รัน script ซ้ำใน Python session เดิม | Python session ใหม่ (รัน `python3 flow.py` ใหม่ทั้งหมด) |
| `KeyError: 'HEICHIPS_SAY'` | ใส่ key ใน flow_cfg แต่ Variable ไม่ได้ declare ใน config_vars | ตรวจว่า `Variable("HEICHIPS_SAY", ...)` อยู่ใน `config_vars` ของ step |
| `TypeError: run() missing argument` | signature ของ `run()` ไม่ถูกต้อง | ต้องเป็น `def run(self, state_in: State, **kwargs)` |
| `AttributeError: 'NoneType' has no attribute` | `state_in[DesignFormat.NL]` เป็น None | ตรวจว่า step ก่อนหน้า produce DesignFormat.NL ออกมาจริง |
| Step ไม่แสดงบน terminal | ลืมใส่ `@Step.factory.register()` decorator | เพิ่ม decorator ก่อน class definition |
| `HeiChipsFlow` ไม่ถูก register | ลืม `@Flow.factory.register()` | เพิ่ม decorator ก่อน class HeiChipsFlow |
| Flow รัน Classic แทน HeiChipsFlow | ใน `main()` ยังสร้าง `Classic(...)` อยู่ | เปลี่ยนเป็น `HeiChipsFlow(...)` |

---

## Bonus: Full Chip Design

**เวลาโดยประมาณ:** 60–90 นาที  
**Objective:** เข้าใจสถาปัตยกรรมของ full chip ที่มี I/O pad ring, seal ring, filler cell และ bondpad, รัน LibreLane `Chip` flow จนได้ GDS ที่พร้อมส่ง tapeout จริง, แก้ไข `chip_core.sv` เพื่อใส่ design ของตัวเอง

---

### พื้นหลัง: ความแตกต่างระหว่าง Classic Flow กับ Chip Flow

ใน Lab 1–5 เราใช้ `Classic` flow ซึ่งสร้าง **chip core** — วงจร logic ที่ route เสร็จ แต่ยังไม่สามารถส่งโรงงานผลิตจริงได้ เพราะขาดองค์ประกอบสำคัญ:

```
Classic Flow ผลิต:
┌─────────────────────────────────┐
│  Core die (logic + routing)     │  ← ได้จาก Lab 1–5
└─────────────────────────────────┘

Chip Flow เพิ่ม:
┌──────────────────────────────────────┐
│ ░░░░░░ Seal Ring ░░░░░░░░░░░░░░░░░░ │  ← ชั้นนอกสุด
│ ░  ┌─┬─┬─┬─┬─┬─┬─┬─┬─┐  ░░░░░░░  │
│ ░  │  I/O Pad Ring      │  ░░░░░░░  │  ← pad cell รอบ chip
│ ░  │  ┌──────────────┐  │  ░░░░░░░  │
│ ░  │  │              │  │  ░░░░░░░  │
│ ░  │  │  Core Logic  │  │  ░░░░░░░  │  ← เหมือน Classic
│ ░  │  │  (PDN ring)  │  │  ░░░░░░░  │
│ ░  │  └──────────────┘  │  ░░░░░░░  │
│ ░  └─┴─┴─┴─┴─┴─┴─┴─┴─┘  ░░░░░░░  │
│ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
└──────────────────────────────────────┘
```

#### Step เพิ่มเติมใน Chip Flow

| Step | หน้าที่ | ผลลัพธ์ |
|---|---|---|
| `OpenROAD.PadRing` | สร้าง I/O pad ring จาก pad cell ใน PDK | DEF ที่มี pad cell วางรอบ die |
| `KLayout.SealRing` | วาด seal ring geometry รอบ chip | GDS layer สำหรับ seal ring |
| `KLayout.Filler` | เติม filler cell ใน I/O ring ที่ว่าง | GDS filler geometry |
| `KLayout.Density` | ตรวจ density ของแต่ละ layer ให้ครบตาม DRC | Density check report |

#### Chip Flow กับ `meta:` block ใน config.yaml

`Chip` flow ถูกเลือกด้วย block พิเศษที่ **ต้นไฟล์** `config.yaml`:

```yaml
meta:
  version: 3
  flow: Chip                  # ← เลือกใช้ Chip flow แทน Classic
  substituting_steps:
    Checker.IllegalOverlap: null   # ลบ step นี้ออก (null = ลบ)
    KLayout.DRC: null
    Checker.KLayoutDRC: null
    Magic.DRC: null
    Checker.MagicDRC: null
```

`meta:` block มีสองหน้าที่หลัก: เลือก flow และ Substitute step โดยใช้ syntax เดียวกับ `Substitutions` ของ Python API (null = ลบ, class name = แทนที่)

> 📝 **หมายเหตุ:** DRC ทั้ง KLayout และ Magic ถูก skip ใน exercise นี้เพื่อประหยัดเวลา ในงาน tapeout จริงต้องรัน DRC เสมอ

---

### ทำความเข้าใจโครงสร้าง Design

#### สถาปัตยกรรม Chip สองชั้น

Design ใน bonus แบ่งเป็นสองระดับ:

```
chip_top.sv       ← wrapper ที่ instantiate pad cell และ chip_core
    └── chip_core.sv  ← วงจร logic จริง (แก้ไขที่นี่เพื่อใส่ design ของตัวเอง)
```

**เหตุผลที่แยก:** `chip_top.sv` เป็น boilerplate ที่ไม่ต้องแตะ — เชื่อม pad กับ core ด้วย naming convention ที่ตายตัว ส่วน `chip_core.sv` คือ "canvas" สำหรับ design ของเรา

#### `chip_top.sv` — Pad Wrapper

```systemverilog
module chip_top #(
    parameter NUM_INPUT_PADS  = 10,   // digital input: input_PAD[9:0]
    parameter NUM_OUTPUT_PADS = 8,    // digital output: output_PAD[7:0]
    parameter NUM_BIDIR_PADS  = 8,    // bidirectional: bidir_PAD[7:0]
    parameter NUM_ANALOG_PADS = 8     // analog: analog_PAD[7:0]
)(
    inout wire clk_PAD,               // clock pad
    inout wire rst_n_PAD,             // reset pad
    inout wire [9:0]  input_PAD,
    inout wire [7:0]  output_PAD,
    inout wire [7:0]  bidir_PAD,
    inout wire [7:0]  analog_PAD
);
```

**I/O pad cell ที่ใช้จาก IHP PDK:**

| Cell name | หน้าที่ | Signal direction |
|---|---|---|
| `sg13g2_IOPadIn` | Digital input pad (Schmitt trigger) | `pad` → `p2c` (pad-to-core) |
| `sg13g2_IOPadOut30mA` | Digital output pad 30mA | `c2p` (core-to-pad) → `pad` |
| `sg13g2_IOPadInOut30mA` | Bidirectional pad 30mA | ทั้งสองทิศ + `c2p_en` |
| `sg13g2_IOPadAnalog` | Analog pad (ไม่มี buffer) | `padres` ↔ `pad` |
| `sg13g2_IOPadVdd` / `sg13g2_IOPadVss` | Core power pad | — |
| `sg13g2_IOPadIOVdd` / `sg13g2_IOPadIOVss` | I/O ring power pad | — |

#### `chip_core.sv` — Core Logic (แก้ไขที่นี่)

```systemverilog
module chip_core #(
    parameter NUM_INPUT_PADS,
    parameter NUM_OUTPUT_PADS,
    parameter NUM_BIDIR_PADS,
    parameter NUM_ANALOG_PADS
)(
    input  logic clk,                           // จาก clk_PAD ผ่าน clk_pad/p2c
    input  logic rst_n,                         // จาก rst_n_PAD
    input  wire  [NUM_INPUT_PADS-1:0]  input_in,   // จาก input_PAD[9:0]
    output wire  [NUM_OUTPUT_PADS-1:0] output_out,  // ไปยัง output_PAD[7:0]
    input  wire  [NUM_BIDIR_PADS-1:0]  bidir_in,
    output wire  [NUM_BIDIR_PADS-1:0]  bidir_out,
    output wire  [NUM_BIDIR_PADS-1:0]  bidir_oe,   // output enable (1=output)
    inout  wire  [NUM_ANALOG_PADS-1:0] analog
);
    // ─── Default design: 8-bit counter ───────────────────────────
    assign bidir_oe = '1;              // bidir ทั้งหมดเป็น output

    logic [NUM_BIDIR_PADS-1:0] count;

    always_ff @(posedge clk) begin
        if (!rst_n)       count <= '0;
        else if (&input_in) count <= count + 1;  // นับเมื่อ input_in ทุก bit = 1
    end

    assign bidir_out  = count;    // แสดงบน bidir pad
    assign output_out = count;    // แสดงบน output pad ด้วย
endmodule
```

#### `bondpad_70x70_novias` — Wire Bond Pad

Bondpad คือ metal square ขนาด **70×70 µm** ที่วางทับบน I/O pad — ใช้สำหรับ **wire bonding** (การต่อสายทองระหว่าง chip และ package ด้วยกระบวนการ bonding)

ไฟล์ bondpad ที่มีใน `ip/bondpad_70x70_novias/`:
- `gds/` — geometry ของ bondpad บน Metal2–TopMetal2
- `lef/` — abstract view ขนาด 70×70 µm
- `vh/` — Verilog header (stub ว่าง เพราะ bondpad ไม่มี logic)

> 📝 **หมายเหตุ:** bondpad ยังไม่ได้รวมอยู่ใน IHP Open PDK อย่างเป็นทางการ จึงต้องระบุเป็น `EXTRA_GDS` และ `EXTRA_LEFS` และใช้ `IGNORE_DISCONNECTED_MODULES` เพราะ bondpad ไม่มี power connection

#### `chip_top.sdc` — Timing Constraints สำหรับ Full Chip

SDC (Synopsys Design Constraints) กำหนด timing สำหรับ pad ring:

```tcl
# Clock ที่ chip_top level มาจาก clk_PAD ผ่าน IOPadIn cell
set clock_port clk_PAD
# CLOCK_NET ชี้ไปที่ output ของ IOPadIn: clk_pad/p2c
# (แทนที่จะชี้ไปที่ port โดยตรง เพราะ clock ผ่าน pad cell แล้ว)
```

ความแตกต่างจาก SDC ของ core design (Lab 1–5): ต้องระบุ `CLOCK_NET` แยกจาก `CLOCK_PORT` เพราะ clock signal ผ่าน I/O pad cell ก่อนถึง core — ทำให้ LibreLane รู้ว่าต้อง propagate clock ผ่าน pad cell ด้วย

---

### ทำความเข้าใจ `config.yaml` ส่วนสำคัญ

```yaml
# ─── Pad Ring Layout ──────────────────────────────────────────────
# แต่ละด้านระบุ instance name ของ pad ตามลำดับจากซ้ายไปขวา (South/North)
# หรือจากล่างไปบน (West/East)
PAD_SOUTH: [clk_pad, rst_n_pad,
            "bidirs\\[0\\].bidir_pad", ..., "bidirs\\[7\\].bidir_pad"]
PAD_EAST:  ["analogs\\[0\\].analog_pad", ..., "vdd_pads\\[0\\].vdd_pad", ...]
PAD_NORTH: ["outputs\\[7\\].output_pad", ..., "outputs\\[0\\].output_pad",
            "iovdd_pads\\[0\\].iovdd_pad", "iovss_pads\\[0\\].iovss_pad"]
PAD_WEST:  ["inputs\\[9\\].input_pad", ..., "inputs\\[0\\].input_pad"]

# ─── Die และ Core Area ────────────────────────────────────────────
FP_SIZING: absolute
DIE_AREA:  [0, 0, 1600, 1600]    # chip ทั้งหมด: 1600×1600 µm (= 1.6×1.6 mm)
CORE_AREA: [365, 365, 1235, 1235] # พื้นที่ logic: 870×870 µm
                                   # margin ~365 µm ทุกด้านสำหรับ pad ring

# ─── Power/Ground ─────────────────────────────────────────────────
VDD_NETS: [VDD]   # ชื่อ net ไฟ (ต่างจาก VPWR ที่ใช้ใน Lab 1–4)
GND_NETS: [VSS]   # ชื่อ net กราวด์

# ─── Clock ────────────────────────────────────────────────────────
CLOCK_PORT:   clk_PAD        # port บน chip_top
CLOCK_NET:    clk_pad/p2c    # internal signal หลังผ่าน pad cell
CLOCK_PERIOD: 20             # 20ns = 50 MHz (ใช้ period ยาวกว่า Lab 1 เพราะ
                             # มี pad cell อยู่บน clock path เพิ่ม delay)

# ─── PDN Core Ring ────────────────────────────────────────────────
PDN_CORE_RING: True          # สร้าง power ring รอบ core ที่ใหญ่กว่า
PDN_CORE_RING_VWIDTH: 15     # ความกว้าง vertical strap: 15 µm
PDN_CORE_RING_HWIDTH: 15     # ความกว้าง horizontal strap: 15 µm
PDN_CORE_RING_VSPACING: 5    # ระยะห่าง (metal slotting rule: >30µm ต้อง slot)
PDN_CORE_RING_HSPACING: 5

PDN_CORE_RING_CONNECT_TO_PADS: True   # เชื่อม ring เข้ากับ power pad โดยตรง
PDN_ENABLE_PINS: False        # ปิด power pin ที่ขอบ core (เพราะใช้ ring แทน)

# ─── Density ──────────────────────────────────────────────────────
PL_TARGET_DENSITY_PCT: 10    # chip ขนาดใหญ่ (870×870 µm) แต่ logic น้อยมาก
                             # 10% density ทำให้ flow ผ่านได้ง่าย
GRT_ALLOW_CONGESTION: true   # อนุญาต routing congestion (จำเป็นสำหรับ chip ใหญ่)
```

---

### ขั้นตอน B.1: รัน Chip Flow

1. เปลี่ยน directory ไปที่ bonus:
   ```bash
   cd bonus
   ```

2. รัน Chip flow:
   ```bash
   librelane config.yaml --pdk ihp-sg13g2
   ```

   > 📝 **หมายเหตุ:** สังเกตลำดับ argument — `config.yaml` อยู่ก่อน `--pdk` ต่างจาก Lab ก่อนหน้าที่ใช้ `librelane --pdk ihp-sg13g2 config.yaml` ทั้งสองลำดับใช้ได้เหมือนกัน

3. Chip flow ใช้เวลานานกว่า Classic (อาจถึง 30–60 นาที) เพราะมี step เพิ่ม เช่น filler generation ที่ต้องเติม cell ทุก micro meter

4. ผลลัพธ์ที่ต้องเห็นเมื่อเสร็จ:
   ```
   * Antenna
   Passed ✅

   * LVS
   Passed ✅
   ```

   > 📝 **หมายเหตุ:** DRC ถูก skip ใน config (`KLayout.DRC: null`) ดังนั้นจะไม่แสดง DRC result — นี่คือการตั้งใจเพื่อประหยัดเวลาใน workshop

> ✅ **จุดตรวจสอบ:** Antenna ✅ และ LVS ✅ ต้องผ่านก่อนดำเนินการต่อ

---

### ขั้นตอน B.2: ดูผลใน OpenROAD GUI

```bash
librelane --pdk ihp-sg13g2 config.yaml --last-run --flow OpenInOpenROAD
```

#### ทำความเข้าใจภาพ `openroad_1.png`

ใน OpenROAD GUI จะเห็น chip layout แบ่งเป็นส่วนต่าง ๆ ชัดเจน:

**Canvas (กลาง):**
- **สีเขียวเข้ม** — core area (870×870 µm) ที่มี standard cell วางอยู่ เป็นพื้นที่ logic จริง ๆ ที่ density เพียง 10% ทำให้ดูว่างมาก
- **สีชมพู/ม่วง** แถบใหญ่รอบ core — I/O pad cell ring (TopMetal1/TopMetal2 power strap ผ่าน pad area)
- **สีฟ้าอ่อน** แนวตั้ง — TopMetal1 core power ring ส่วนที่ผ่านจาก pad ring เข้าสู่ core
- **สีแดง** เส้นเล็ก ๆ ที่ขอบ — pin และ via connection ระหว่าง pad cell กับ core PDN

**Hierarchy Browser (ขวา):**
- `<top>` → `chip_top` — instance ทั้งหมด 236 ตัว
- **Input pad** 12, **Output pad** 8, **Input/output...** 16 (bidir) — pad cell จำนวนจริงจาก `generate` block
- **Pad spacer** 136 — filler cell ที่เติมช่องว่างระหว่าง pad ให้ต่อเนื่อง
- **Clock buffer** 3, **Inverter** 3, **Sequential** 8 — standard cell ใน chip_core
- **Multi-Input** 45 — logic gate จาก chip_core (AND, NAND, NOR ฯลฯ)

> 💡 **เคล็ดลับ:** คลิก instance ใน Hierarchy Browser เพื่อ highlight ใน canvas และดูว่า pad แต่ละตัวอยู่ตำแหน่งไหน

---

### ขั้นตอน B.3: ดูผลใน KLayout

```bash
librelane --pdk ihp-sg13g2 config.yaml --last-run --flow OpenInKLayout
```

#### ทำความเข้าใจภาพ `klayout_1.png`

KLayout แสดง GDS จริงที่พร้อมส่งโรงงาน — ทุก polygon ทุก layer มีอยู่ครบ:

**Cells panel (ซ้าย):** เห็น hierarchy ทั้งหมดรวม:
- `chip_top` — top-level
- `bondpad_70x70_novias` — bondpad ที่เพิ่มมาจาก `EXTRA_GDS`
- `sealring` — seal ring geometry ที่ `KLayout.SealRing` สร้าง
- `sg13g2_Corner` — corner cell ของ seal ring
- `sg13g2_Filler1000`, `sg13g2_Filler2000`, `sg13g2_Filler400` — filler cell ที่ `KLayout.Filler` เติม
- `sg13g2_IOPadAnalog`, `sg13g2_IOPadIn`, `sg13g2_IOPadOut30mA`, `sg13g2_IOPadInOut30mA` — pad cell

**Canvas:** สังเกตชั้นต่าง ๆ จากนอกสุดเข้าใน:
1. **Seal ring** — เส้น pattern ที่ขอบนอกสุดของ chip ทำจาก metal หลายชั้น ป้องกันการแตกและ moisture ingress
2. **I/O pad ring** (สีส้ม/เหลือง) — pad cell และ bondpad รอบ chip
3. **Core power ring** (สีม่วงเข้ม) — TopMetal1/TopMetal2 strap ที่รับไฟจาก power pad
4. **Core area** — standard cell พร้อม routing

**Layers panel (ขวา):** layer ทั้งหมดของ IHP SG13G2 ตั้งแต่ `Substrate.drawing` ถึง `TopMetal2.drawing` พร้อม filler layer (`Activ.filler`, `Metal1.filler` ฯลฯ)

> 💡 **เคล็ดลับ:** ใน KLayout กด `F` เพื่อ zoom fit และ `Shift+F` เพื่อ fit เฉพาะที่เลือก ลองซูมเข้า seal ring เพื่อดู geometry ละเอียด

---

### ขั้นตอน B.4: ใส่ Design ของตัวเองใน `chip_core.sv`

นี่คือแบบฝึกหัดสุดท้าย — แก้ไข `chip_core.sv` เพื่อใส่ logic ของตัวเองแทน counter เริ่มต้น

#### ทำความเข้าใจ Interface ของ chip_core

```
Inputs ที่มีให้ใช้:
  clk              — clock จาก pad
  rst_n            — reset (active low) จาก pad
  input_in[9:0]    — 10 bit digital input จาก input pad
  bidir_in[7:0]    — 8 bit จาก bidirectional pad (ถ้าตั้งเป็น input)

Outputs ที่ต้องขับ:
  output_out[7:0]  — 8 bit digital output ไปยัง output pad
  bidir_out[7:0]   — 8 bit bidirectional output
  bidir_oe[7:0]    — output enable (1 = output, 0 = input)
  analog[7:0]      — analog pad (ไม่มี buffer — ใช้เชื่อม analog circuit)
```

#### ตัวอย่างที่ 1: PWM Generator อย่างง่าย

```systemverilog
module chip_core #(
    parameter NUM_INPUT_PADS,
    parameter NUM_OUTPUT_PADS,
    parameter NUM_BIDIR_PADS,
    parameter NUM_ANALOG_PADS
)(
    input  logic clk, rst_n,
    input  wire  [NUM_INPUT_PADS-1:0]  input_in,
    output wire  [NUM_OUTPUT_PADS-1:0] output_out,
    input  wire  [NUM_BIDIR_PADS-1:0]  bidir_in,
    output wire  [NUM_BIDIR_PADS-1:0]  bidir_out,
    output wire  [NUM_BIDIR_PADS-1:0]  bidir_oe,
    inout  wire  [NUM_ANALOG_PADS-1:0] analog
);
    assign bidir_oe = '0;  // bidir เป็น input ทั้งหมด (ไม่ใช้)

    // PWM: ใช้ input_in[7:0] เป็น duty cycle (0–255)
    logic [7:0] counter;
    logic [7:0] duty = input_in[7:0];  // ปรับ duty cycle จาก input pad

    always_ff @(posedge clk) begin
        if (!rst_n) counter <= '0;
        else        counter <= counter + 1;
    end

    // output_out[0] = PWM signal
    assign output_out[0] = (counter < duty) ? 1'b1 : 1'b0;
    assign output_out[7:1] = '0;
    assign bidir_out = '0;
endmodule
```

#### ตัวอย่างที่ 2: LFSR (Linear Feedback Shift Register) — Pseudo-Random Generator

```systemverilog
module chip_core #(...)(
    input  logic clk, rst_n,
    ...
    output wire  [NUM_OUTPUT_PADS-1:0] output_out,
    output wire  [NUM_BIDIR_PADS-1:0]  bidir_out,
    output wire  [NUM_BIDIR_PADS-1:0]  bidir_oe,
    ...
);
    assign bidir_oe = '1;  // bidir เป็น output

    // 8-bit LFSR: polynomial x^8 + x^6 + x^5 + x^4 + 1
    logic [7:0] lfsr;
    wire feedback = lfsr[7] ^ lfsr[5] ^ lfsr[4] ^ lfsr[3];

    always_ff @(posedge clk) begin
        if (!rst_n) lfsr <= 8'hFF;  // seed ≠ 0
        else        lfsr <= {lfsr[6:0], feedback};
    end

    assign output_out = lfsr;
    assign bidir_out  = lfsr;
endmodule
```

#### รัน Flow หลังแก้ไข

เมื่อแก้ `chip_core.sv` แล้ว รัน flow ใหม่:

```bash
librelane config.yaml --pdk ihp-sg13g2
```

> 💡 **เคล็ดลับ:** ถ้าต้องการรัน synthesis ก่อนเพื่อตรวจสอบว่า Verilog ถูกต้องโดยไม่รอ routing ทั้งหมด:
> ```bash
> librelane config.yaml --pdk ihp-sg13g2 -T Yosys.Synthesis
> ```

---

### ขั้นตอน B.5: ตรวจสอบผลลัพธ์สุดท้าย

เมื่อ flow เสร็จสมบูรณ์แล้ว ตรวจสอบ output files ใน `final/`:

```bash
ls runs/RUN_<timestamp>/final/
# gds/   lef/   nl/   lib/   spef/
```

ไฟล์ที่สำคัญสำหรับ tapeout:

| ไฟล์ | ความสำคัญ |
|---|---|
| `final/gds/chip_top.gds` | ไฟล์หลักที่ส่งให้โรงงาน |
| `final/lef/chip_top.lef` | ใช้ถ้า chip นี้เป็น macro ใน design ใหญ่กว่า |
| `final/nl/chip_top.nl.v` | Netlist สำหรับ formal verification |

> ✅ **จุดตรวจสอบสุดท้าย:** เปิด `final/gds/chip_top.gds` ใน KLayout และตรวจว่า seal ring อยู่รอบขอบทุกด้าน, pad cell ครบทุกตำแหน่ง, และ core area ไม่ว่างเกินไปจนผิดปกติ

---

### การแก้ไขปัญหา

| อาการ | สาเหตุ | วิธีแก้ |
|---|---|---|
| `[ERROR] Pad instance 'X' not found` | ชื่อ pad ใน `PAD_SOUTH/EAST/NORTH/WEST` ไม่ตรงกับ instance ใน Verilog | ตรวจ `chip_top.sv` ว่า instance ชื่ออะไร (เช่น `clk_pad`, `bidirs[0].bidir_pad`) |
| `[ERROR] Pad ring not closed` | pad ไม่ครบรอบ มีช่องว่าง | เพิ่ม pad spacer หรือตรวจว่า pad list ทุกด้านครบ |
| `[ERROR] Power ring connection failed` | `PDN_CORE_RING_CONNECT_TO_PADS: True` แต่ power pad ไม่มี | ตรวจว่า `vdd_pads` และ `vss_pads` อยู่ใน PAD_EAST |
| LVS fail: `VDDARRAY missing` | เกิดเฉพาะถ้ามี SRAM macro | เพิ่ม `PDN_MACRO_CONNECTIONS` สำหรับ VDDARRAY |
| `[ERROR] bondpad GDS not found` | Path ใน `EXTRA_GDS` ผิด | ตรวจว่า `ip/bondpad_70x70_novias/gds/` อยู่ใน bonus directory |
| Density check fail | layer บางชั้นมี density ต่ำเกินกำหนด | เพิ่ม `PL_TARGET_DENSITY_PCT` หรือเพิ่ม fill cell ด้วย variable `FILL_*` |
| Core logic ไม่ปรากฏใน GDS | แก้ `chip_core.sv` ผิด module name | ตรวจว่า module name ยังเป็น `chip_core` ไม่ใช่ชื่ออื่น |
| `GRT_ALLOW_CONGESTION` warning | routing congested แต่ flow ยังผ่าน | ปกติ ignore ได้ ถ้าต้องการแก้ให้ลด `PL_TARGET_DENSITY_PCT` |

---

## สรุปและ Key Takeaways

### สิ่งที่ได้เรียนรู้

- **LibreLane** เป็น open-source ASIC implementation flow ที่ครอบคลุมตั้งแต่ RTL ถึง GDS
- **`config.yaml`** คือจุดศูนย์กลางในการควบคุม flow — ปรับได้ทุกอย่างตั้งแต่ขนาด die จนถึง PDN
- **`--to`, `--from`, `--skip`** ช่วยให้ debug ได้เร็วขึ้นมากโดยไม่ต้องรัน flow ทั้งหมดซ้ำ
- **Macro** ช่วยให้ reuse design ได้และลดเวลา implementation ของ design ขนาดใหญ่
- **Python API** เปิดโอกาสให้ extend LibreLane ด้วย custom flow และ step ของตัวเอง
- **IHP Open PDK (`ihp-sg13g2`)** เป็น PDK จริงที่ใช้ผลิต chip ใน HeiChips 2026

### ขั้นตอนถัดไป

- ดูรายการ configuration variable ทั้งหมด: [LibreLane Docs](https://librelane.readthedocs.io/en/latest/reference/step_config_vars.html)
- เรียนรู้การสร้าง custom flow: [Writing Custom Flows](https://librelane.readthedocs.io/en/latest/usage/writing_custom_flows.html)
- ทดลอง implement design ของตัวเองและรวมเข้ากับ HeiChips 2026 Hackathon
- ศึกษา [IHP Open PDK](https://github.com/IHP-GmbH/IHP-Open-PDK) เพื่อทำความเข้าใจ process ที่ใช้ผลิตจริง
- อ่านเกี่ยวกับ [Tiny Tapeout](https://tinytapeout.com/) สำหรับโอกาส tapeout ถัดไป

---

## ภาคผนวก

### คำสั่ง LibreLane ที่ใช้บ่อย

```bash
# รัน Classic flow ปกติ
librelane --pdk ihp-sg13g2 config.yaml

# รันถึง step ที่ระบุ
librelane --pdk ihp-sg13g2 config.yaml -T Yosys.Synthesis

# รันจาก step ที่ระบุ
librelane --pdk ihp-sg13g2 config.yaml -F OpenROAD.GlobalPlacement -i runs/.../state_in.json

# ข้าม step
librelane --pdk ihp-sg13g2 config.yaml -S KLayout.DRC

# เปิด OpenROAD GUI
librelane --pdk ihp-sg13g2 config.yaml --last-run --flow OpenInOpenROAD

# เปิด KLayout
librelane --pdk ihp-sg13g2 config.yaml --last-run --flow OpenInKLayout
```

### Glossary คำศัพท์

| คำศัพท์ | ความหมาย |
|---|---|
| **PDK** | Process Design Kit — ชุดข้อมูลจากโรงงานผลิต |
| **RTL** | Register Transfer Level — ระดับการเขียน Verilog/SystemVerilog |
| **GDS** | Graphic Design System — format ไฟล์ layout ที่ส่งโรงงาน |
| **DRC** | Design Rule Check — ตรวจสอบว่า layout ตามกฎโรงงาน |
| **LVS** | Layout vs. Schematic — ตรวจสอบว่า layout ตรงกับ netlist |
| **PDN** | Power Distribution Network — ระบบจ่ายไฟให้ chip |
| **Macro** | Block ที่ implement แล้ว พร้อมใช้งาน |
| **Standard Cell** | หน่วยพื้นฐานเช่น AND, OR, flip-flop จาก library |
| **Floorplan** | การกำหนดขนาดและพื้นที่ของ chip |
| **Place & Route** | การวาง cell และเดินสายเชื่อมต่อ |
| **Tapeout** | การส่งไฟล์ GDS ให้โรงงานเพื่อผลิต chip จริง |

### โครงสร้างโฟลเดอร์ Repository

```
heichips26-digital-workshop/
├── exercise_1/          # Lab 1: 8-bit counter
│   ├── counter.sv
│   ├── config.yaml
│   └── img/
├── exercise_2/          # Lab 2: Configuration variables
│   ├── src/
│   ├── def/             # DEF templates (Tiny Tapeout)
│   └── config.yaml
├── exercise_3/          # Lab 3: Flow control
│   ├── shift_register.sv
│   └── config.yaml
├── exercise_4/          # Lab 4: Macros
│   ├── counter_8bit/    # Macro source
│   ├── counter_32bit.sv
│   ├── top.sv
│   └── config.yaml
├── exercise_5/          # Lab 5: Python API
│   ├── counter.sv
│   └── flow.py
└── bonus/               # Full chip design
    ├── src/
    ├── chip_top.sdc
    └── config.yaml
```

### Links ที่มีประโยชน์

- [LibreLane Documentation](https://librelane.readthedocs.io)
- [LibreLane GitHub](https://github.com/librelane/librelane)
- [IHP Open PDK](https://github.com/IHP-GmbH/IHP-Open-PDK)
- [OpenROAD Project](https://github.com/The-OpenROAD-Project/OpenROAD)
- [KLayout](https://www.klayout.de)
- [HeiChips 2026](https://heichips.github.io)
- [Tiny Tapeout](https://tinytapeout.com)
