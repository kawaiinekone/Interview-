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
A physical register is a parallel bank of edge-triggered D flip-flops that holds binary voltages (0V ground or nominal Vdd):
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
* **CPU Implementations:** Program Counter (PC), General-Purpose Registers (`x0` through `x31`), Pipeline Latches.

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
* **On the sharp rising edge:** Transistor gates settle into valid digital logic states (0 or 1) across setup-time windows, and sequential storage elements latch the inputs.

---

## 5. Single-Cycle vs. 3-Stage Pipeline: The Structural Trade-Off

### The Single-Cycle Datapath (Mini-Takshaka)
The processor completes an instruction's full lifecycle in **one uninterrupted clock period**.

```text
One Single Continuous Clock Period (Tclk = 20ns → Fmax = 50 MHz):
├─────────────────────────────────────────────────────────────────────────────────────────────┤
[ Fetch: IMEM ] ──► [ Decode ] ──► [ Reg Read ] ──► [ ALU AGU ] ──► [ Data RAM ] ──► [ Reg Commit ]
```

#### Real-Life Analogy: The Single-Operator Bakery
* One baker mixes flour, kneads dough, bakes the loaf in the oven, packages it, and places it on the counter for a customer.
* **Advantage:** No scheduling conflicts or order mix-ups. There are **zero pipeline hazards** because customer 2 never steps up until customer 1 takes their loaf.
* **The Structural Flaw:** The baker is bound to the slowest order. If a customer only asks for a glass of water (`addi`), they must wait the full baking time of a multicourse pastry (`lw`) before the next customer is served.
* **Electrical Reality:**
  ```text
  Clock Period (Tclk) >= T_PC + T_IMEM + T_Decode + T_RegRead + T_ALU + T_DMEM + T_WBMux + T_Setup
  ```
  Because signals must traverse all five operational blocks sequentially within one clock tick, the operating frequency is limited (Fmax ≈ 50 MHz).

---

### The 3-Stage Pipelined Datapath (Takshaka)
Takshaka segments the datapath into three balanced, synchronous stages separated by clocked registers:

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
* Baker 1 prepares ingredients (Fetch).
* Baker 2 kneads and cuts (Execute).
* Baker 3 monitors baking and packages (Memory/Writeback).

```text
Cycle 1: [ Instruction 1: Fetch   ]
Cycle 2: [ Instruction 2: Fetch   ]  [ Instruction 1: Execute ]
Cycle 3: [ Instruction 3: Fetch   ]  [ Instruction 2: Execute ]  [ Instruction 1: Writeback ]
```

* **The Gain:** Each stage's critical path is roughly one-third the length of the single-cycle design, allowing a **≈ 3x higher clock frequency (Fmax >= 200 MHz)**.
* **The Challenge:** Multiple instructions in-flight require hazard control:
  * **W -> X Forwarding Bypasses:** Hardware bypass wires route results straight from stage W's output back to stage X's input, eliminating load-use stalls.
  * **Branch Prediction:** A 256-entry gshare branch history table predicts loop trajectories to prevent 1-cycle pipeline flushes.

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
* **Function:** A 32-bit register holding the memory address of the instruction currently being executed.
* **Sequential Stepping:** For linear program flow, an adder adds 4 bytes (advancing past a standard 32-bit instruction word):
  ```text
  Next PC = Current PC + 4
  ```
* **Branch Redirection:** For conditional branches (`beq`), an input multiplexer routes the calculated jump target address instead:

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
* **Function:** Pure combinational decoding that parses the 32-bit instruction word from instruction memory into hardware control lines.

```text
                         32-bit Instruction Word (from imem_rdata)
                                            │
    ┌────────────┬────────────┬─────────────┼────────────┬────────────┬────────────┐
    ▼            ▼            ▼             ▼            ▼            ▼            ▼
 funct7         rs2          rs1         funct3          rd         opcode      imm_gen
                        [11:7]        [6:0]      Sign-Ext
```

* **Control Assignments:**
  * Configures the ALU operation (`ADD`, `SUB`, `AND`, `OR`, `SLT`) based on `funct3` and `funct7`.
  * Asserts `reg_write` to allow results into the register file.
  * Asserts `dmem_we` to allow writes into Data Memory.

---

### Block 3: The Register File (`x0` to `x31`)
* **Function:** Multi-ported static RAM array housing thirty-two 32-bit general-purpose registers.
* **Interface:**
  * **Two Read Ports:** Driven by index inputs `rs1` and `rs2`. Combinational multiplexer trees output data words on `rf_rdata1` and `rf_rdata2` without waiting for a clock edge.
  * **One Synchronous Write Port:** Driven by destination index `rd` and data bus `wb_data`. Updates flip-flops on the rising clock edge if `reg_write == 1`.
* **Hardwired Zero Register (`x0`):** Grounded to 0V (`32'h00000000`). Reads always yield 0, and writes targeting register 0 are ignored.

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
* **Function:** High-speed parallel arithmetic and bitwise logic engine.
* **Dual Multiplexing:**
  * Input A always receives `rf_rdata1`.
  * Input B is selected by a multiplexer: receives `rf_rdata2` for register-register operations (`add`), or sign-extended immediate data (`imm`) for calculations like `addi` and memory offsets (`lw`/`sw`).
* **Address Generation Unit (AGU):** For loads and stores, the ALU computes the target effective memory address:
  ```text
  Effective Address = Register(rs1) + Sign-Extended Immediate
  ```

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
* **Function:** Output steering circuit that chooses which operational result commits to the register file.

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

Consider an addition instruction executing in silicon:

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
1. The PC register outputs `0x00001000` onto the instruction address lines.
2. The instruction memory decodes the address and presents the machine code word `0x007302B3` onto `imem_rdata`.
3. Concurrently, an adder computes `PC + 4 = 0x00001004`.

---

### Step 2: Instruction Decode & Field Demux (D)
The binary word `0x007302B3` passes to the decoder:

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
* Sets source indices `rs1 = 5'd6` and `rs2 = 5'd7`.
* Sets destination index `rd = 5'd5`.
* Selects the ALU ADD operation.
* Asserts `reg_write = 1` and deasserts `dmem_we = 0`.

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
The internal multiplexers resolve the addresses, placing `10` onto `rf_rdata1` and `20` onto `rf_rdata2`.

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
The ALU's carry-lookahead adder computes `10 + 20 = 30`.

---

### Step 5: Writeback Multiplexing & Sequential State Retirement
```text
alu_result (30) ────► ┌─────────────────┐
                      │  Writeback MUX  ├────► wb_data = 30 (0x0000001E)
dmem_rdata (X)  ────► └────────┬────────┘
                               ▲
                         is_lw = 0
```

1. The Writeback MUX selects `alu_result` because this is an arithmetic calculation rather than a memory read.
2. Value `30` travels across the writeback bus to destination write port `rd = 5` of the register file.
3. **The Active Rising Clock Edge Arrives:**
   * Register `x5` latches `30` into its flip-flops.
   * The Program Counter latches `0x00001004`.

The instruction completes, and the next instruction cycle begins.

---

## 8. Summary Comparison: Architectural Evolution

| Microarchitectural Metric | Mini-Takshaka (Single-Cycle Baseline) | Ibex Core (2-Stage Pipeline) | Takshaka RV32 (3-Stage Pipeline Target) |
| :--- | :--- | :--- | :--- |
| **Pipeline Depth** | 1 Contiguous Stage (No inter-stage latches) | 2 Stages (`IF` and `ID/EX`) | 3 Concurrent Stages (`F -> X -> W`) |
| **Cycles Per Instruction (CPI)** | **1.0** (All instructions retire in 1 tick) | **~1.2** (Interlocked branch/load delays) | **~1.0** (Scalar pipelined throughput) |
| **Operating Frequency (Fmax)** | **Low (~50 MHz)** (Full critical path) | **Moderate (~120 MHz)** (Balanced 2-stage) | **High (~200+ MHz)** (Isolated stages) |
| **Hazard Resolution Scheme** | **Zero Hazards** (Sequential execution) | Interlocked structural stalling | **Full W -> X Forwarding** (Zero load-use stalls) |
| **Branch Penalty** | **0 Cycles** (Combinational PC calculation) | 1 Cycle Stall Bubble | **1 Cycle Flush** (Mitigated by 256-entry gshare) |
| **CoreMark Efficiency** | Low (Constrained by clock speed) | ~1.43 CoreMark/MHz | **2.68 CoreMark/MHz** |

### Architectural Conclusion
By analyzing how signals, multiplexers, and flip-flops interact during a single clock cycle, the trade-offs of deeper pipelining, forwarding bypasses, and branch prediction in advanced cores like **Takshaka** become clear.
