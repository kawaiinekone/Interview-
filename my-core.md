# Mini-Takshaka: 32-Bit Single-Cycle RISC-V Processor Core
*Synthesizable RV32I Baseline Implementation & Architectural Verification*

---

## 1. Architectural Overview & Design Philosophy

**Mini-Takshaka** is a synthesizable, single-cycle 32-bit RISC-V processor implementing the fundamental base integer instruction set (`RV32I`). It serves as an empirical reference core designed to explore datapath timing, register file interactions, and instruction retirement before analyzing advanced pipelined microarchitectures like Takshaka.

### What Does "Single-Cycle" Mean in Hardware?
In this architecture, **every instruction completes its entire lifecycle in exactly one clock cycle (CPI = 1.0)**.
* An instruction is fetched from memory, decoded into control signals, executed through the arithmetic logic unit (ALU), and committed to memory or the register file within a single tick of the clock.
* **Zero Pipeline Hazards:** Because each instruction completely finishes and commits its state before the next instruction begins, the datapath is physically immune to pipeline hazards:
  * **No Data Hazards (RAW):** An instruction never attempts to read an uncommitted register value because no other instruction is currently in flight[cite: 2, 3].
  * **No Load-Use Stalls:** A memory load (`lw`) writes back to the register file combinationally/synchronously within the cycle; subsequent instructions read the updated register file directly.
  * **No Branch Bubbles:** Branch target evaluation occurs in the same cycle as condition evaluation, updating the Program Counter (`PC`) without pipeline flushes.

### The Critical Path Bottleneck: Why This Baseline Matters
While a single-cycle core has zero hazards, its maximum operating frequency ($F_{\max}$) is constrained by the cumulative propagation delay of all hardware blocks in series:

$$\text{Critical Path} = T_{\text{PC}} + T_{\text{IMEM}} + T_{\text{Decode}} + T_{\text{RegRead}} + T_{\text{ALU}} + T_{\text{DMEM}} + T_{\text{WBMux}} + T_{\text{Setup}}$$

Because the clock period must accommodate the slowest instruction (`lw`), the clock frequency remains low (~50 MHz). This physical limit forms the exact engineering rationale for pipelining in cores like Takshaka[cite: 1, 3].

---

## 2. Core Datapath Mechanics (Q&A Architectural Breakdown)

### How is an instruction fetched?
* **RTL Module:** Program Counter (`pc`) register and Instruction Memory (`IMEM`) interface.
* **Execution:** The `pc` register asserts its address onto `imem_addr[31:0]`. The memory block performs a combinational read, returning the 32-bit machine word on `imem_rdata[31:0]`. Concurrently, a parallel 32-bit adder computes `pc + 4` to determine the sequential return address.

### How is the instruction decoded?
* **RTL Module:** Hardwired combinational decoder[cite: 1, 3].
* **Field Extraction:** The 32-bit instruction word is sliced into standard RISC-V fields:
  * `opcode`: `imem_rdata[6:0]` (Determines instruction class: R, I, S, B)[cite: 2]
  * `rd`: `imem_rdata[11:7]` (Destination register index)[cite: 2]
  * `funct3`: `imem_rdata[14:12]` (Operation sub-category)[cite: 2]
  * `rs1`: `imem_rdata[19:15]` (Source register 1 index)[cite: 2]
  * `rs2`: `imem_rdata[24:20]` (Source register 2 index)[cite: 2]
  * `funct7`: `imem_rdata[31:25]` (Arithmetic mode selector)[cite: 2]
* **Immediate Generation:** The Immediate Generator sign-extends the extracted constants into full 32-bit words:
  * `imm_i` for `addi`, `lw`
  * `imm_s` for `sw`
  * `imm_b` for `beq`

### How are registers read and written?
* **Structure:** A register file containing 32 general-purpose 32-bit registers (`x0` through `x31`)[cite: 2, 3].
* **Asynchronous Dual Read:** Indices `rs1` and `rs2` drive internal multiplexer arrays to output `rf_rdata1` and `rf_rdata2` combinationally within nanoseconds[cite: 1, 3].
* **The Zero Register (`x0`):** Hardwired to electrical zero (`32'b0`)[cite: 1, 2]. Reads from `x0` always yield 0; writes to `x0` are safely ignored[cite: 1, 2].
* **Synchronous Single Write:** On the rising clock edge, if `reg_write` is asserted by the decoder, `wb_data` is latched into the physical flip-flops of register `rd`.

### How does the ALU perform operations?
* **Structure:** 32-bit parallel arithmetic and logic execution block.
* **Operand Multiplexing:** Operand A receives `rf_rdata1`. An input multiplexer selects Operand B between `rf_rdata2` (for R-type instructions) and the sign-extended immediate (for I-type/S-type instructions)[cite: 3].
* **Address Generation Unit (AGU):** For memory operations (`lw` and `sw`), the ALU computes the effective memory target address[cite: 2, 3]:
  $$\text{Effective Address} = \text{Register}(rs1) + \text{Immediate Offset}$$

### How are branches handled?
* **Condition Evaluation:** A dedicated comparator in the ALU evaluates whether `rf_rdata1 == rf_rdata2` for `beq` instructions.
* **Target Calculation:** A branch adder computes `Branch Target = PC + imm_b` in parallel.
* **PC Update:** If the branch condition is true, `branch_taken` asserts, and the PC multiplexer loads `branch_target` on the next clock edge.

### How does the core communicate with data memory?
* **Store Instructions (`sw`):** The effective address from the ALU drives `dmem_addr`, and data from `rf_rdata2` drives `dmem_wdata`. The write-enable line `dmem_we` asserts, writing the 32-bit word into RAM on the clock edge.
* **Load Instructions (`lw`):** The effective address drives `dmem_addr`, and the RAM returns the stored data onto `dmem_rdata`.

### How is writeback handled?
* **Writeback MUX:** A multiplexer chooses what commits to register `rd`:
  ```verilog
  wb_data = (opcode == LW) ? dmem_rdata : alu_result;
  
Retirement: At the rising clock edge, wb_data commits into register rd, while the PC commits pc_next, retiring the instruction.
---

## 3. Deep Example Trace: The Lifecycle of `add x3, x1, x2`

Consider the execution of:
```asm
add x3, x1, x2    # Hardware Target: x3 = x1 + x2
```
Initial State: x1 = 15 (0x0000000F), x2 = 25 (0x00000019), PC = 0x00000008

Machine Instruction: 0x002081B3.


