# 01 - Hardware Fundamentals, Real-Life Mental Models & Datapath Walkthrough

## 1. What is RTL (Register Transfer Level)?

RTL stands for:
```text
Register Transfer Level
```

In plain words:
> **RTL describes what physical state is retained in silicon flip-flops, where voltages travel across conductive copper traces, and how logic gates transform those bit patterns between clock edges.**

### The Real-Life Analogy: The Assembly Line Factory
Consider an industrial manufacturing workshop:
* **Workers at workbenches (Combinational Logic):** They take raw items, cut them, weld them, or drill holes. They hold nothing overnight; if you stop feeding them materials, they produce nothing.
* **Storage Bins / Lockers (Registers):** Stable shelves that hold components securely.
* **The Whistle / Shift Bell (Clock Signal):** At the sound of the bell, workers grab raw parts from Bin A, process them across the bench, and deposit the finished item into Bin B.

```text
Software Program (Python / C / Java)
              ↓
   "Tell the factory what product to build"
              ↓
Assembly Language (RISC-V ISA)
              ↓
   add x5, x6, x7  (A standardized work ticket)
              ↓
RTL (SystemVerilog / Verilog Hardware Description)
              ↓
   The physical blueprint specifying the benches, conveyor tracks, and storage bins
```

* **Software** is an abstract sequence of algorithmic operations executed by an underlying host.
* **RTL** is a structural description of digital hardware circuits synthesized into silicon gates, interconnects, and flip-flops.

---

## 2. What Does "Register Transfer" Actually Mean?

The term identifies the atomic loop of synchronous digital design:

### 1. Register
A physical register is a parallel bank of edge-triggered D flip-flops that holds binary voltages ($0\text{V}$ ground or nominal $V_{dd}$):
```text
Register A = 10 (0x0000000A)
Register B = 20 (0x00000014)
```

### 2. Transfer
Transfer is the directed movement of digital electrical charges across metal bus tracks through switching logic into another register:

```text
Register A (10) ────┐
                    ├──► [ ALU: Carry-Lookahead Adder ] ────► (30) ────► Register C
Register B (20) ────┘
```

The perpetual hardware execution sequence:
```text
State Storage (Source Registers)
       ↓
Data Bus Propagation (Metal Conductors)
       ↓
Combinational Processing (ALU / Shifters / Logic Gates)
       ↓
Writeback Bus Routing (Multiplexers)
       ↓
State Latching (Destination Registers on posedge clk)
```

This cyclic transfer of binary words defines the **Register Transfer Level**.

---

## 3. Combinational Logic vs. Sequential Logic

Every digital microarchitecture is divided into two distinct circuit domains:

```text
              ┌────────────────────────────────────────────────────────┐
              │               THE DIGITAL PROCESSOR CORE               │
              └───────────────────────────┬────────────────────────────┘
                                          │
                  ┌───────────────────────┴────────────────────────┐
                  ▼                                                ▼
        Combinational Logic                             Sequential Logic
        (Stateless / Immediate Propagation)             (Stateful / Clock Synchronized)
```

### Combinational Logic: The Light Switch Matrix
* Output voltages depend directly and immediately on present input voltages, modulated only by gate propagation delays.
* The circuit contains no feedback storage or memory.
* **Real-Life Example:** A set of mechanical wall switches configured in parallel. Flip switch A, and the bulb illuminates instantly; turn it off, and it goes dark immediately.
* **CPU Implementations:** Arithmetic adders, decoders, sign-extension blocks, multiplexers (MUX).

```text
Input A (10) ───┐
                ├──► [ Combinational Adder Array ] ───► Result Output (30)
Input B (20) ───┘
```
*If Input A fluctuates from 10 to 15, the output changes to 35 after the electrical signals propagate through the adder's logic gates.*

### Sequential Logic: The Camera Shutter
* Stores and stabilizes binary data across discrete intervals.
* State transitions occur strictly when triggered by an active timing signal (the clock edge).
* **Real-Life Example:** A photographer taking snapshots. While the shutter is closed, subjects move around unpredictably. When the shutter fires (the clock tick), the camera freezes the scene into a permanent frame until the next exposure.
* **CPU Implementations:** Program Counter (PC), General-Purpose Registers (`x0–x31`), Pipeline Latches.

```text
                     ┌─────────────────────────────┐
Data Input (30) ────►│ Physical D Flip-Flop Array  │────► Stored Output Q (Stable)
                     └──────────────┬──────────────┘
                                    ▲
         Active Clock Edge: ────────┘ (Latches input only on rising edge)
```

---

## 4. The Digital Clock: Orchestrating Silicon Propagation

Digital processors rely on synchronous timing signals:

```text
       Clock Period (Tclk)
      |<─────────────────────────>|
      ┌─────────────┐             ┌─────────────┐             ┌─────────────┐
      │  Logic High │             │  Logic High │             │  Logic High │
──────┘             └─────────────┘             └─────────────┘             └─────
      ▲                           ▲                           ▲
  Rising Edge 1               Rising Edge 2               Rising Edge 3
  (Time t = 0ns)              (Time t = 10ns)             (Time t = 20ns)
```

### The Metronome Analogy
Musicians in an orchestra match tempo against a conductor's baton:
* **During the interval between ticks:** Electrical signals ripple through silicon gates, switching between high and low voltages. Internal nets experience intermediate electrical noise and propagation delay.
* **On the sharp rising edge:** Transistor gates settle into valid digital logic states ($0$ or $1$) across setup-time windows, and sequential storage elements latch the inputs.

---

## 5. Single-Cycle vs. 3-Stage Pipeline: The Structural Trade-Off

### The Single-Cycle Datapath (Mini-Takshaka)
The processor completes an instruction's full lifecycle in **one uninterrupted clock period**[span_0](start_span)[span_0](end_span).

```text
One Single Continuous Clock Period (Tclk = 20ns → Fmax = 50 MHz):
├─────────────────────────────────────────────────────────────────────────────────────────────┤
[ Fetch: IMEM ] ──► [ Decode ] ──► [ Reg Read ] ──► [ ALU AGU ] ──► [ Data RAM ] ──► [ Reg Commit ]
```

#### Real-Life Analogy: The Single-Operator Bakery
* One baker mixes flour, kneads dough, bakes the loaf in the oven, packages it, and places it on the counter for a customer.
* **Advantage:** No scheduling conflicts or order mix-ups. There are **zero pipeline hazards** because customer 2 never steps up until customer 1 takes their loaf[span_1](start_span)[span_1](end_span).
* **The Structural Flaw:** The baker is bound to the slowest order. If a customer only asks for a glass of water (`addi`), they must wait the full baking time of a multicourse pastry (`lw`) before the next customer is served[span_2](start_span)[span_2](end_span).
* **Electrical Reality:**
  $$\text{Clock Period } (T_{\text{clk}}) \ge T_{\text{PC}} + T_{\text{IMEM}} + T_{\text{Decode}} + T_{\text{RegRead}} + T_{\text{ALU}} + T_{\text{DMEM}} + T_{\text{WBMux}} + T_{\text{Setup}}$$
  Because signals must traverse all five operational blocks sequentially within one clock tick, the operating frequency is limited ($F_{\max} \approx 50\text{ MHz}$)[span_3](start_span)[span_3](end_span).

---

### The 3-Stage Pipelined Datapath (Takshaka)
Takshaka segments the datapath into three balanced, synchronous stages separated by clocked registers[span_4](start_span)[span_4](end_span)[span_5](start_span)[span_5](end_span):

```text
   Stage F (Fetch)             Stage X (Execute)                 Stage W (Memory/WB)
┌──────────────────┐         ┌─────────────────────────┐       ┌──────────────────────┐
│ PC & Instruction │──►[R]──►│ Decoder, RegFile Read,  │──►[R]►│ Data Memory (SRAM)   │
│ Memory Interface │         │ Execution ALU & AGU     │       │ & Writeback Commit   │
└──────────────────┘         └─────────────────────────┘       └──────────────────────┘
                        ▲                                 ▲
                Pipeline Register                 Pipeline Register
```

#### Real-Life Analogy: The Conveyor Assembly Line
* Baker 1 prepares ingredients (Fetch)[span_6](start_span)[span_6](end_span).
* Baker 2 kneads and cuts (Execute)[span_7](start_span)[span_7](end_span).
* Baker 3 monitors baking and packages (Memory/Writeback)[span_8](start_span)[span_8](end_span).

```text
Cycle 1: [ Instruction 1: Fetch   ]
Cycle 2: [ Instruction 2: Fetch   ]  [ Instruction 1: Execute ]
Cycle 3: [ Instruction 3: Fetch   ]  [ Instruction 2: Execute ]  [ Instruction 1: Writeback ]
```

* **The Gain:** Each stage's critical path is roughly one-third the length of the single-cycle design, allowing a **$\approx 3\times$ higher clock frequency ($F_{\max} \ge 200\text{ MHz}$)**[span_9](start_span)[span_9](end_span)[span_10](start_span)[span_10](end_span).
* **The Challenge:** Multiple instructions in-flight require hazard control:
  * **$W \to X$ Forwarding Bypasses:** Hardware bypass wires route results straight from stage W's output back to stage X's input, eliminating load-use stalls[span_11](start_span)[span_11](end_span).
  * **Branch Prediction:** A 256-entry gshare branch history table predicts loop trajectories to prevent 1-cycle pipeline flushes[span_12](start_span)[span_12](end_span).

---

## 6. Detailed Hardware Anatomy of the 5 Core Blocks

```text
             ┌──────────────────────────────────────────────────────────────────────────┐
             │                   COMPLETE DATAPATH HARDWARE TOPOLOGY                    │
             └────────────────────────────────────┬─────────────────────────────────────┘
                                                  │
       ┌──────────────────┬───────────────────────┼───────────────────────┬──────────────────┐
       ▼                  ▼                       ▼                       ▼                  ▼
 1. Program         2. Instruction          3. Register             4. Arithmetic      5. Writeback
    Counter (PC)       Decoder                 File (RF)               Logic Unit         Multiplexer
    Address Engine     Control Logic           Storage Array           (ALU / AGU)        Commit Unit
```

---

### Block 1: The Program Counter (PC)
* **Function:** A 32-bit register holding the memory address of the instruction currently being executed[span_13](start_span)[span_13](end_span)[span_14](start_span)[span_14](end_span).
* **Sequential Stepping:** For linear program flow, an adder adds 4 bytes (advancing past a standard 32-bit instruction word)[span_15](start_span)[span_15](end_span)[span_16](start_span)[span_16](end_span):
  $$\text{PC}_{\text{next}} = \text{PC} + 4$$
* **Branch Redirection:** For conditional branches (`beq`), an input multiplexer routes the calculated jump target address instead[span_17](start_span)[span_17](end_span):

```text
                ┌──────────────────┐
 Branch Target ─►│                  │
 (Calculated)   │  Multiplexer     ├────► [ PC Register: 32 Flip-Flops ] ────► imem_addr [31:0]
  PC + 4 Adder ─►│                  │                     │
                └────────┬─────────┘                     │
                         ▲                               ▼
                   branch_taken                 +4 Incrementor Adder
```

---

### Block 2: The Instruction Decoder
* **Function:** Pure combinational decoding that parses the 32-bit instruction word from instruction memory into hardware control lines[span_18](start_span)[span_18](end_span)[span_19](start_span)[span_19](end_span).

```text
                         32-bit Instruction Word (from imem_rdata)
                                            │
    ┌────────────┬────────────┬─────────────┼────────────┬────────────┬────────────┐
    ▼            ▼            ▼             ▼            ▼            ▼            ▼
 funct7         rs2          rs1         funct3          rd         opcode      imm_gen
                        [11:7]        [6:0]      Sign-Ext
```

* **Control Assignments:**
  * Configures the ALU operation (`ADD`, `SUB`, `AND`, `OR`, `SLT`) based on `funct3` and `funct7`[span_20](start_span)[span_20](end_span)[span_21](start_span)[span_21](end_span).
  * Asserts `reg_write` to allow results into the register file[span_22](start_span)[span_22](end_span).
  * Asserts `dmem_we` to allow writes into Data Memory[span_23](start_span)[span_23](end_span).

---

### Block 3: The Register File (`x0` to `x31`)
* **Function:** Multi-ported static RAM array housing thirty-two 32-bit general-purpose registers[span_24](start_span)[span_24](end_span)[span_25](start_span)[span_25](end_span).
* **Interface:**
  * **Two Read Ports:** Driven by index inputs `rs1` and `rs2`. Combinational multiplexer trees output data words on `rf_rdata1` and `rf_rdata2` without waiting for a clock edge[span_26](start_span)[span_26](end_span)[span_27](start_span)[span_27](end_span).
  * **One Synchronous Write Port:** Driven by destination index `rd` and data bus `wb_data`. Updates flip-flops on the rising clock edge if `reg_write == 1`[span_28](start_span)[span_28](end_span)[span_29](start_span)[span_29](end_span)[span_30](start_span)[span_30](end_span).
* **Hardwired Zero Register (`x0`):** Grounded to $0\text{V}$ (`32'h00000000`)[span_31](start_span)[span_31](end_span)[span_32](start_span)[span_32](end_span). Reads always yield 0, and writes targeting register `0` are ignored[span_33](start_span)[span_33](end_span)[span_34](start_span)[span_34](end_span).

```text
                   ┌───────────────────────────────────────┐
       rs1 [4:0] ─►│                                       │─► rf_rdata1 [31:0] (Combinational)
       rs2 [4:0] ─►│          REGISTER FILE                │─► rf_rdata2 [31:0] (Combinational)
        rd [4:0] ─►│       32 x 32-bit Registers           │
   wb_data [31:0] ─►│      (x0 Hardwired Ground)            │
   reg_write ───►│                                       │
                   └───────────────────┬───────────────────┘
                                       ▲
                                  posedge clk (Synchronous Commit)
```

---

### Block 4: The Arithmetic Logic Unit (ALU / AGU)
* **Function:** High-speed parallel arithmetic and bitwise logic engine[span_35](start_span)[span_35](end_span)[span_36](start_span)[span_36](end_span).
* **Dual Multiplexing:**
  * Input A always receives `rf_rdata1`[span_37](start_span)[span_37](end_span).
  * Input B is selected by a multiplexer: receives `rf_rdata2` for register-register operations (`add`), or sign-extended immediate data (`imm`) for calculations like `addi` and memory offsets (`lw`/`sw`)[span_38](start_span)[span_38](end_span).
* **Address Generation Unit (AGU):** For loads and stores, the ALU computes the target effective memory address[span_39](start_span)[span_39](end_span)[span_40](start_span)[span_40](end_span):
  $$\text{Effective Address} = \text{Register}(rs1) + \text{Sign-Extended Immediate}$$

```text
  rf_rdata1 [31:0] ──────────────────────┐
                                         ├──► ┌──────────────────────┐
  rf_rdata2 [31:0] ──────┐               │    │                      │
                         ├──► [ MUX ] ───┴──► │ 32-bit Parallel ALU  ├────► alu_result [31:0]
  Immediate (imm) ───────┘                    │ (Adder/Sub/Logic)    │
                                              │                      ├──► branch_taken (Condition Flag)
  ALU Operation Selection Control ──────────► └──────────────────────┘
  (From Instruction Decoder)
```

---

### Block 5: The Writeback Multiplexer (WB MUX)
* **Function:** Output steering circuit that chooses which operational result commits to the register file[span_41](start_span)[span_41](end_span)[span_42](start_span)[span_42](end_span)[span_43](start_span)[span_43](end_span).

```text
                         ┌───────────────────────┐
  alu_result [31:0] ────►│     Writeback MUX     │
  (Computation Result)   │                       ├────► wb_data [31:0] ────► Register File (rd)
  dmem_rdata [31:0] ────►│  (Selected by is_lw)  │
  (Data Memory Read)     └───────────┬───────────┘
                                     ▲
                           is_lw Control Strobe
```

---

## 7. Complete Hardware Walkthrough: Tracing `add x5, x6, x7`

Consider an addition instruction executing in silicon[span_44](start_span)[span_44](end_span):

```asm
add x5, x6, x7       # Architectural Intent: Register x5 <= Register x6 + Register x7
```

### Initial Register State
* `Register x6` contains integer **`10`** (`0x0000000A`)
* `Register x7` contains integer **`20`** (`0x00000014`)
* Program Counter (`PC`) points to **`0x00001000`**

---

### Step 1: Instruction Fetch (F)
```text
PC Register Value (0x00001000)
       │
       ▼ imem_addr
┌─────────────────────────────────┐
│     Instruction Memory (IMEM)   │
└────────────────┬────────────────┘
                 ▼ imem_rdata = 0x007302B3
```
1. The PC register outputs `0x00001000` onto the instruction address lines[span_45](start_span)[span_45](end_span).
2. The instruction memory decodes the address and presents the machine code word `0x007302B3` onto `imem_rdata`[span_46](start_span)[span_46](end_span).
3. Concurrently, an adder computes $\text{PC} + 4 = \text{0x00001004}$[span_47](start_span)[span_47](end_span).

---

### Step 2: Instruction Decode & Field Demux (D)
The binary word `0x007302B3` passes to the decoder[span_48](start_span)[span_48](end_span):

```text
Binary Bitfield Breakdown:
┌──────────────┬──────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│ funct7       │ rs2          │ rs1          │ funct3       │ rd           │ opcode       │
│      │      │      │      │ [11:7]       │ [6:0]        │
├──────────────┼──────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ 0000000      │ 00111        │ 00110        │ 000          │ 00101        │ 0110011      │
│ (Base Math)  │ (Reg x7)     │ (Reg x6)     │ (ADD/SUB)    │ (Reg x5)     │ (R-Type)     │
└──────────────┴──────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
```

The decoder establishes the control lines:
* Sets source indices `rs1 = 5'd6` and `rs2 = 5'd7`[span_49](start_span)[span_49](end_span).
* Sets destination index `rd = 5'd5`[span_50](start_span)[span_50](end_span).
* Selects the ALU ADD operation[span_51](start_span)[span_51](end_span)[span_52](start_span)[span_52](end_span).
* Asserts `reg_write = 1` and deasserts `dmem_we = 0`[span_53](start_span)[span_53](end_span).

---

### Step 3: Register File Read Operations
Source addresses `rs1 = 6` and `rs2 = 7` route into the register file's read address ports:

```text
Register File Internal Memory Array
┌─────────────────────────┐
│ Entry 6  (x6): 0x0000000A ├──────────────► rf_rdata1 = 10
│ Entry 7  (x7): 0x00000014 ├──────────────► rf_rdata2 = 20
└─────────────────────────┘
```
The internal multiplexers resolve the addresses, placing `10` onto `rf_rdata1` and `20` onto `rf_rdata2`[span_54](start_span)[span_54](end_span)[span_55](start_span)[span_55](end_span).

---

### Step 4: Arithmetic Execution
Because this is an R-type instruction, the ALU input multiplexer selects the register operand `rf_rdata2`:

```text
rf_rdata1 (10) ────────┐
                       ├──► ┌────────────────────────┐
rf_rdata2 (20) ────────┘    │  32-bit Carry-Lookahead │────► alu_result = 30 (0x0000001E)
                            │  Adder Engine          │
ALU Control (ADD) ────────► └────────────────────────┘
```
The ALU's carry-lookahead adder computes $10 + 20 = 30$[span_56](start_span)[span_56](end_span).

---

### Step 5: Writeback Multiplexing & Sequential State Retirement
```text
alu_result (30) ────► ┌─────────────────┐
                      │  Writeback MUX  ├────► wb_data = 30 (0x0000001E)
dmem_rdata (X)  ────► └────────┬────────┘
                               ▲
                         is_lw = 0
```

1. The Writeback MUX selects `alu_result` because this is an arithmetic calculation rather than a memory read[span_57](start_span)[span_57](end_span).
2. Value `30` travels across the writeback bus to destination write port `rd = 5` of the register file[span_58](start_span)[span_58](end_span).
3. **The Active Rising Clock Edge Arrives:**
   * Register `x5` latches `30` into its flip-flops[span_59](start_span)[span_59](end_span).
   * The Program Counter latches `0x00001004`[span_60](start_span)[span_60](end_span).

The instruction completes, and the next instruction cycle begins[span_61](start_span)[span_61](end_span).

---

## 8. Summary Comparison: Architectural Evolution

| Microarchitectural Metric | Mini-Takshaka (Single-Cycle Baseline)[span_62](start_span)[span_62](end_span) | Ibex Core (2-Stage Pipeline)[span_63](start_span)[span_63](end_span) | Takshaka RV32 (3-Stage Pipeline Target)[span_64](start_span)[span_64](end_span) |
| :--- | :--- | :--- | :--- |
| **Pipeline Depth** | 1 Contiguous Stage (No inter-stage latches)[span_65](start_span)[span_65](end_span) | 2 Stages (`IF` and `ID/EX`)[span_66](start_span)[span_66](end_span) | 3 Concurrent Stages (`F -> X -> W`)[span_67](start_span)[span_67](end_span) |
| **Cycles Per Instruction (CPI)** | **1.0** (All instructions retire in 1 tick)[span_68](start_span)[span_68](end_span) | **~1.2** (Interlocked branch/load delays)[span_69](start_span)[span_69](end_span) | **~1.0** (Scalar pipelined throughput)[span_70](start_span)[span_70](end_span) |
| **Operating Frequency ($F_{\max}$)** | **Low (~50 MHz)** (Full critical path)[span_71](start_span)[span_71](end_span) | **Moderate (~120 MHz)** (Balanced 2-stage)[span_72](start_span)[span_72](end_span) | **High (~200+ MHz)** (Isolated stages)[span_73](start_span)[span_73](end_span)[span_74](start_span)[span_74](end_span) |
| **Hazard Resolution Scheme** | **Zero Hazards** (Sequential execution)[span_75](start_span)[span_75](end_span) | Interlocked structural stalling[span_76](start_span)[span_76](end_span) | **Full $W \to X$ Forwarding** (Zero load-use stalls)[span_77](start_span)[span_77](end_span) |
| **Branch Penalty** | **0 Cycles** (Combinational PC calculation)[span_78](start_span)[span_78](end_span) | 1 Cycle Stall Bubble[span_79](start_span)[span_79](end_span) | **1 Cycle Flush** (Mitigated by 256-entry gshare)[span_80](start_span)[span_80](end_span) |
| **CoreMark Efficiency** | Low (Constrained by clock speed)[span_81](start_span)[span_81](end_span) | ~1.43 CoreMark/MHz[span_82](start_span)[span_82](end_span) | **2.68 CoreMark/MHz**[span_83](start_span)[span_83](end_span) |

### Architectural Conclusion
By analyzing how signals, multiplexers, and flip-flops interact during a single clock cycle, the trade-offs of deeper pipelining, forwarding bypasses, and branch prediction in advanced cores like **Takshaka** become clear[span_84](start_span)[span_84](end_span)[span_85](start_span)[span_85](end_span).
