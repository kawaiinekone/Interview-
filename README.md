# Architectural Study & Systems Reference: Takshaka 3-Stage RISC-V Core

> **Target Core:** Takshaka RV32 Core (OpenR5 / OR5 Labs)  
> **Architecture Class:** 3-Stage In-Order Pipelined Core (F / X / W)  
> **Course / Academic Context:** Computer Systems Architecture (CSA)  
> **Author & Repository Maintainer:** Shubhi Rai  
> **Core Documentation Reference:** OR5 Labs Takshaka Technical Manual (`or5.org/takshaka`)  
> **Textbook Reference:** *RISC-V Architecture Tutorial: Complete Guide from Fundamentals to Advanced Implementation* by Nikhil Kumar Rajput  

---

## 1. Executive Summary & Plain-English Overview

### What is Takshaka?
**Takshaka** is an open-source, 32-bit computer processor core developed by OR5 Labs under the open-source MIT license. It is designed around the modern **RISC-V Instruction Set Architecture (ISA)**.

In modern electronics, computer processors generally fall into one of two extremes:
1. **Tiny, simple processors:** They run slowly because they take multiple clock ticks to finish a single instruction (known as multicycle cores).
2. **Big, complicated processors:** Found in laptops and smartphones, they run fast but consume massive power, take up large physical chip area, and run hot.

**Takshaka occupies the sweet spot:** It is engineered for devices like smart motor controllers, drones, embedded microcontrollers, and IoT hardware. It delivers near high-end processing speed while maintaining a tiny, power-efficient silicon footprint.

### The Core Analogy: The 3-Worker Fast-Food Assembly Line
Imagine a restaurant kitchen where each meal requires three steps:
1. **Take the order ticket** from the counter (Fetch: `F`).
2. **Read the ticket and gather ingredients** from the fridge (Decode & Operand Read: `X`).
3. **Cook the meal and hand it to the customer** (Execute, Memory, Writeback: `W`).

- **A Multicycle CPU (like Agni):** A single worker completes all three steps from start to finish before taking the next order. It is simple and avoids collisions, but the kitchen produces only one meal every 3 to 4 clock ticks.
- **Takshaka's 3-Stage Pipeline:** Three workers stand in a row. While Worker 3 cooks Meal 1, Worker 2 gathers ingredients for Meal 2, and Worker 1 takes the order for Meal 3. Once the line fills up, **a completed meal comes out on every single clock tick**.

---

## 2. Technical Profile at a Glance

| Architectural Metric | Specification / Implementation | Practical Meaning |
| :--- | :--- | :--- |
| **ISA Standard** | RV32IMACB_Zicsr_Zcb_Zbc (B = Zba + Zbb + Zbs) | 32-bit clean base with hardware math, atomics, bitwise logic, and compressed instructions. |
| **Register Width (XLEN)** | 32-bit | Word size for general-purpose registers and datapath buses. |
| **Pipeline Stages** | **3 Stages:** Fetch (`F`) → Execute (`X`) → Memory / Writeback (`W`) | Balances single-cycle throughput while keeping branch stall penalties down to 1 cycle. |
| **Hazard Resolution** | Full `W` → `X` Forwarding Path | Answers are fed immediately to the next instruction; **zero load-use delay stalls**. |
| **Branch Predictor** | 256-entry gshare BHT (8-bit global history) + 64-entry BTB + 8-entry RAS | Predicts loop/condition outcomes so the processor does not stall waiting for branch decisions. |
| **Privilege Modes** | Machine (M); optional User (U in SECURE config) | Supports standard embedded operational states and security isolation. |
| **Memory Protection** | Optional 8-region PMP + Smepmp (mseccfg) | Physical Memory Protection checking in hardware for secure enclaves. |
| **Debug Interface** | RISC-V External Debug 0.13 (JTAG DTM + DM) + Sdtrig triggers | Hardware breakpoints, single-stepping, and register inspection support. |
| **Bus Interfaces** | Native combinational IMEM/DMEM port; AXI4-Lite master bridge | Connects to on-chip SRAM or standard SoC interconnect fabrics. |
| **CoreMark Score** | **2.68 CoreMark / MHz** (Simulation) / **2.92** on specific tests | Demonstrates high cycle efficiency, beating multicycle cores (1.43) by over 85%. |

---

## 3. Deep-Dive Pipeline Architecture (Block Diagram Walkthrough)

Based on the official RTL implementation (`rtl/takshaka_core.sv`), the core is divided into three physical stages separated by inter-stage pipeline registers (`F/X` and `X/W`):

```
+-------------------------------------------------------------------------------------------------------------------------+
|                                          TAKSHAKA 3-STAGE PIPELINE BLOCK DIAGRAM                                        |
+------------------------------------+-------------------------------------------+----------------------------------------+
|          STAGE 1: F - FETCH        |             STAGE 2: X - EXECUTE          |        STAGE 3: W - MEMORY / WB        |
|     (PC, prediction, RVC expand)   |          (decode, read, execute, address) |           (memory access, writeback)   |
+------------------------------------+-------------------------------------------+----------------------------------------+
|                                    |                                           |                                        |
|  [ INSTRUCTION BUS ]               |  [ DECODER ]        [ IMMEDIATE GEN ]     |  [ LOAD / STORE ]                      |
|  imem_addr [31:0]                  |  RV32IMACB, Zbc, Zcb  I/S/B/U/J formats   |  byte-lane align                       |
|  imem_rdata [31:0]                 |  A, URET inline                           |  misaligned: 2 beats                   |
|  native port, combinational IMEM   |                                           |  AMO RMW, LR/SC                        |
|                  │                 |                                           |                    │                   |
|                  ▼                 |  [ REGISTER FILE ]  [ ALU + BRANCH ]      |                    ▼                   |
|  [ PC REGISTER ]                   |  32 x 32-bit        Zba/Zbb/Zbs/Zbc       |  [ WRITEBACK MUX ]                     |
|  pc (+2 / +4 / predicted)          |  2 read / 1 write   branch compare        |  ALU / MEM / PC+4                      |
|                  ▲                 |  written from W     address (AGU)         |  MUL-DIV / CSR                         |
|                  │                 |                                           |                    │                   |
|  [ BRANCH PREDICTOR ]              |  [ MUL / DIV ]      [ CSR + TRAPS ]       |                    │                   |
|  gshare BHT: 256 x 2-bit           |  multi-cycle        Zicsr, Zihpm          |                    │                   |
|  global history: 8 bits            |  stalls front end   precise traps, IRQs   |                    │                   |
|  BTB: 64 entries                   |                     resolve in X          |                    │                   |
|  RAS: 8 entries (trained from X)   |                                           |                    │                   |
|                                    |  [ PMP (SECURE) ]   [ TRIGGERS ]          |                    │                   |
|  [ RVC EXPANDER ]                  |  8 regions          Sdtrig x 2            |                    │                   |
|  16-bit -> 32-bit (C + Zcb)        |  TOR / NA4 / NAPOT  mcontrol6             |                    │                   |
|                                    |  fetch + load/store PC breakpoint         |                    │                   |
|  [ FETCH CONTROL ]                 |  M/U, user deleg.   load/store watchpoint |                    │                   |
|  straddling 32-bit fetch           |                                           |                    │                   |
|  stall: mul/div or misaligned      |                                           |                    │                   |
+──────────────────┬─────────────────+─────────────────────┬─────────────────────+────────────────────┬───────────────────+
                   │                                       │                                          │
                   │ ─── F/X Pipeline Register ──────────> │ ─── X/W Pipeline Register ─────────────> │
                   │                                       │                                          │
                   │ <── redirect / trap vector (flush) ── │                                          │
                   │                                       │ <── forwards W -> X (no load-use stall) ─│
                   │                                       │ <── writeback -> register file ──────────│
```

### Stage 1: F - FETCH (Instruction Acquisition & Speculation)
* **PC Register:** Holds the instruction address (`imem_addr [31:0]`). Increments sequentially by `+4` (standard word) or `+2` (compressed halfword), or updates to a predicted target address.
* **Branch Predictor Subsystem:**
  * **256-Entry gshare BHT:** Uses an 8-bit global branch history XOR-ed with the PC bits to index 2-bit saturating counters, predicting taken/not-taken outcomes.
  * **64-Entry Branch Target Buffer (BTB):** Caches destination target addresses for direct jumps and branches.
  * **8-Entry Return Address Stack (RAS):** Predicts subroutine return addresses (`ret`) to eliminate call/return delays.
  * **Training Feedback:** The branch predictor is trained directly from branch resolution in the Execute (`X`) stage.
* **RVC Expander (`takshaka_rvc.sv`):** Sits directly in the fetch path. It inspects incoming 16-bit instructions (C and Zcb extensions) and translates them into equivalent 32-bit instructions before passing them to the decoder.
* **Fetch Control:** Manages instruction alignment. If a 32-bit instruction straddles across a word boundary, the fetch controller coordinates a second beat. It also halts instruction fetching during multi-cycle stalls (such as division or misaligned memory access).

### Stage 2: X - EXECUTE (Decode, Arithmetic, and Control Resolution)
* **Instruction Decoder (`takshaka_decode.sv`):** Decodes full RV32IMACB, Zbc, and Zcb instruction profiles.
* **Immediate Generator:** Extracts and sign-extends 12-bit, 20-bit, or branch offsets across standard I, S, B, U, and J instruction formats.
* **Register File:** Contains thirty-two 32-bit registers (`x0` through `x31`). Provides two independent combinational read ports (for `rs1` and `rs2`) and one synchronous write port written from the Writeback (`W`) stage.
* **ALU & Branch Unit:** Executes standard integer arithmetic, logic operations, bit-manipulation extensions (Zba, Zbb, Zbs, Zbc), branch comparisons, and memory address generation (AGU).
* **Multi-Cycle Multiplier / Divider Unit (`takshaka_muldiv.sv`):**
  * Multiplication executes in a single pipelined cycle.
  * Division executes iteratively across 32 shift-subtract steps plus a sign-correction cycle.
  * While active, this unit asserts a stall line back to the Fetch stage.
* **CSR & Trap Unit:** Manages Control and Status Registers (`Zicsr`), performance counters (`Zihpm`), and resolves exceptions/interrupts. All branch directions, traps, and jump targets resolve here.
* **Redirect / Pipeline Flush:** If an instruction in `X` determines that a branch was mispredicted or an interrupt occurred, it asserts a redirect signal to the PC and flushes the single younger instruction currently in Stage `F` (costing exactly 1 bubble cycle).
* **SECURE Modules (Optional Build):** Includes Physical Memory Protection (8 PMP regions supporting TOR, NA4, and NAPOT addressing) and hardware debug watchpoint triggers (Sdtrig).

### Stage 3: W - MEMORY / WRITEBACK (Data Memory Access & Retirement)
* **Load / Store Unit:** Drives the external data memory interface (`dmem_addr`, `dmem_wdata`, `dmem_rdata`, `dmem_be`).
  * Handles byte-lane masking (`lb`, `lh`, `lw`, `sb`, `sh`, `sw`).
  * Supports Atomic Memory Operations (`AMO RMW`, `LR.W`, `SC.W`).
  * Manages hardware misaligned memory accesses across two distinct clock cycles.
* **Writeback MUX:** Selects the final result to commit to the destination register (`rd`) among five sources:
  1. ALU / Bit-manipulation calculation
  2. Data memory load (`MEM`)
  3. Link register return address (`PC + 4`)
  4. Multiplier / Divider result
  5. CSR read data
* **Forwarding Path (`W` → `X`):** The output of the Writeback MUX connects directly to the operand multiplexers of the ALU in Stage `X`. This allows data from loads and computations to be used immediately in the next cycle without a load-use stall.

---

## 4. Hardware Hazard Management

```
Hazard Type               Hardware Resolution Mechanism
---------------------------------------------------------------------------------------------------------
Data Dependency (RAW)     Full W -> X forwarding bypass; zero load-use stalls
Branch Misprediction      1-cycle pipeline flush from X to F; PC redirects to correct branch target
Multi-Cycle Math (DIV)    Iterative 32-cycle shift-subtract stalls front-end fetch until completion
Misaligned Load / Store   Split into 2 sequential memory beats in W; stalls front-end fetch for 1 cycle
Atomic Operation (AMO)    Atomic two-beat read-modify-write sequence stalls pipeline during bus access
```

### Full Forwarding vs. The Load-Use Hazard
In standard 5-stage pipelines, an instruction that immediately follows a load (`lw`) must stall for 1 cycle because memory data is not ready until the end of the MEM stage.

In Takshaka:
1. Instruction 1 (`lw`) enters Stage `W` and reads data from the memory bus.
2. Instruction 2 arrives in Stage `X` requiring that data.
3. The `W → X` forwarding wire feeds the memory data directly into the ALU input multiplexer during that same clock cycle.
4. **Result:** Takshaka eliminates the load-use penalty entirely without needing a complex instruction scoreboard.

---

## 5. Register File Architecture & ABI Conventions

Takshaka implements 32 general-purpose registers (`x0` through `x31`), each 32 bits wide:

| Register | ABI Name | Calling Role | Hardware Operational Rule |
| :--- | :--- | :--- | :--- |
| `x0` | `zero` | Permanent Zero | Hardwired to electrical ground (0V). Writes are discarded. |
| `x1` | `ra` | Return Address | Holds return target; written automatically by `jal`/`jalr`. Caller-saved. |
| `x2` | `sp` | Stack Pointer | Tracks top of the stack. Aligned to 16 bytes. Callee-saved. |
| `x3` | `gp` | Global Pointer | Points to base of static global data segment. |
| `x4` | `tp` | Thread Pointer | Points to thread-local storage blocks. |
| `x5–x7` | `t0–t2` | Temporaries | Scratchpad registers. Functions overwrite without saving. Caller-saved. |
| `x8` | `s0 / fp` | Saved / Frame | Callee-saved register or frame pointer. |
| `x9` | `s1` | Saved Register | Preserved across calls by callee. |
| `x10–x11` | `a0–a1` | Args / Returns | Primary arguments and function return values. Caller-saved. |
| `x12–x17` | `a2–a7` | Function Args | Function arguments 3 through 8. Caller-saved. |
| `x18–x27` | `s2–s11` | Saved Registers | Callee-saved; must be saved to stack if used by subroutines. |
| `x28–x31` | `t3–t6` | Temporaries | Additional caller-saved scratchpad registers. |

---

## 6. Strict Load/Store Architecture & Misaligned Support

Takshaka follows the strict RISC-V **Load/Store design**:
* Arithmetic instructions (`add`, `sub`, `and`, `or`) operate **only on register operands**.
* External memory is never read or written directly inside an arithmetic operation.
* Memory is accessed exclusively through dedicated load (`lw`, `lh`, `lb`) and store (`sw`, `sh`, `sb`) instructions.

### Hardware Misaligned Access Handling
Normally, a 32-bit word access must be aligned to an address divisible by 4 (`addr[1:0] == 00`).
* **Standard Cores:** Accessing a word across boundary lines (e.g., address `0x1001`) triggers an alignment exception trap that must be handled slowly in software.
* **Takshaka Hardware Support:** Takshaka detects misaligned addresses in the Load/Store unit in Stage `W`. It automatically splits the transfer into two consecutive aligned memory accesses and reassembles the byte lanes across an extra clock cycle, avoiding software trap overhead.

---

## 7. Supported Instruction Set Extensions (RV32IMACB)

Takshaka implements the **RV32IMACB_Zicsr_Zcb_Zbc** feature profile:

1. **RV32I (Base Integer):** The minimal standard 32-bit integer ISA (load, store, add, branch, jumps).
2. **M Extension (Multiply/Divide):** Single-cycle hardware multiplication (`mul`, `mulh`) and iterative shift-subtract division (`div`, `rem`).
3. **A Extension (Atomics):** Atomic memory operations (`amoadd.w`, `amoswap.w`, etc.) and Load-Reserved/Store-Conditional (`lr.w`/`sc.w`) for multi-core synchronization.
4. **C & Zcb Extensions (Compressed Instructions):** 16-bit compressed instructions expanded in Stage `F` via `takshaka_rvc.sv`, reducing binary code footprint by 25–30%.
5. **B Extension (Zba, Zbb, Zbs) & Zbc (Bit Manipulation):**
   * **Zba:** Address generation instructions (`sh1add`, `sh2add`, `sh3add`) that shift and add in a single ALU cycle.
   * **Zbb:** Bit operations like count leading zeros (`clz`), population count (`cpop`), byte-swap (`rev8`), and min/max.
   * **Zbs:** Single-bit manipulation (`bset`, `bclr`, `binv`, `bext`).
   * **Zbc:** Carry-less multiplication (`clmul`) for accelerated CRC calculations and cryptography.

---

## 8. Benchmark Evaluation & Performance Metrics

On our laboratory's Verilator RTL simulation benchmarks:

```
Benchmark Evaluation (CoreMark / MHz across Pipeline Architectures):
- Multicycle (Agni):           1.43 CoreMark / MHz  [Sequential FSM, CPI > 2.5]
- 5-Stage Pipeline (Gandiva):  2.34 CoreMark / MHz  [Classic pipeline, 2-3 cycle branch penalty]
- 3-Stage Pipeline (Takshaka): 2.68 CoreMark / MHz  [Short pipeline, 1-cycle branch penalty]
- Advanced (Chakra OoO):       3.84 CoreMark / MHz  [Superscalar / Out-of-Order execution]
```

### Why Takshaka Scores 2.68 CoreMark/MHz
1. **Pipelined Throughput vs. Multicycle:** Unlike multicycle cores that take 3 to 4 cycles per instruction, Takshaka retires close to 1 instruction per cycle on standard code.
2. **Low Branch Penalty vs. 5-Stage:** CoreMark code is dense with conditional branches and loops. In a 5-stage core, branch mispredictions flush multiple stages. In Takshaka, branches resolve in Stage `X`, meaning a misprediction flushes only Stage `F` (a 1-cycle penalty).
3. **Zero Load-Use Stalls:** Forwarding from `W` to `X` prevents pipeline bubbles during memory-load dependencies.

---

## 9. Verification & Silicon Physical Design Flow

### Verification Architecture
* **Golden Model Co-Simulation:** Simulation runs program traces side-by-side on Takshaka RTL and a golden Python ISA model (`tools/golden_rv32im.py`), verifying PC addresses and register state retirement instruction by instruction.
* **RVFI (RISC-V Formal Interface):** Validates retirement trace continuity, traps, and ensures writes to `x0` always evaluate to zero.
* **Self-Checking Directed Suites:** Tests verify instruction behavior, JTAG debug, hardware interrupts, and FreeRTOS multitasking execution.

### Physical Implementation via OpenROAD Flow Scripts (ORFS)
Takshaka is synthesized into physical silicon using the open-source OpenROAD flow:
1. **Logic Synthesis (Yosys):** Compiles SystemVerilog RTL into standard logic gates.
2. **Floorplanning & Placement:** Organizes macros, core boundaries, and standard cell sites.
3. **Clock Tree Synthesis (CTS):** Synthesizes balanced clock distribution networks to minimize clock skew.
4. **Routing (TritonRoute):** Connects signal nets across physical metal layers.
5. **Static Timing Analysis (OpenSTA):** Evaluates timing slack and computes maximum operating frequency ($F_{\max}$).
6. **GDSII Generation:** Emits final layout masks ready for foundry manufacturing.

---
