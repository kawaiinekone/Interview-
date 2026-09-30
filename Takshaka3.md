Takshaka: A Simple Guide to the 3-Stage RISC-V Core

By Shubhi Rai — Computer Systems Architecture study notes

This guide explains the Takshaka processor core, made by OR5 Labs (open source, MIT license). It's a 32-bit RISC-V core with a 3-stage pipeline. I studied its RTL code and manual and wrote down what I learned, in plain words.

---

1. What is Takshaka?

Takshaka is an open-source, 32-bit computer processor core developed by OR5 Labs under the open-source MIT license. It is designed around the modern RISC-V Instruction Set Architecture (ISA).


CPUs come in two extremes today:

1. Tiny, simple cores — cheap and low power, but slow. Each instruction takes 3–4 clock cycles because one unit does everything step by step (a multicycle core, like their Agni core).
2. Big, complex cores — laptops and phones. Very fast, but burn lots of power, take up big chip area, and run hot.

Takshaka sits in the sweet spot between them. It's built for embedded devices: motor controllers, drones, IoT gadgets, microcontrollers. It gives near-high-end speed while staying small and power-efficient.

The fast-food analogy (how the 3-stage pipeline works)

Think of a kitchen where every meal needs 3 steps:

1. Take the order ticket (Fetch, stage F)
2. Read the ticket and grab ingredients (Decode & Execute, stage X)
3. Cook it and serve it (Memory & Writeback, stage W)

- A multicycle CPU has ONE worker who does all 3 steps for meal 1 before touching meal 2. Simple, but only 1 meal every 3–4 ticks of the clock.
- Takshaka has 3 workers in a row. While worker 3 cooks meal 1, worker 2 preps meal 2, and worker 1 takes the order for meal 3. Once the line fills up, one finished meal comes out every single clock tick.

(Reference image: the full block diagram of Takshaka, drawn from the actual RTL files `takshaka_core.sv` and `takshaka_soc.sv`.)

---

2. Quick Facts

Feature	Spec	What it means in plain words	
ISA	RV32IMACB + Zicsr + Zcb + Zbc	Full 32-bit RISC-V with hardware multiply/divide, atomics, compressed instructions, and bit-manipulation	
Register width	32-bit	All registers and data buses carry 32 bits	
Pipeline	3 stages: Fetch → Execute → Memory/Writeback	One instruction finishes almost every clock tick, but a wrong branch guess only wastes 1 cycle	
Hazard handling	Full forwarding from W back to X	Results are passed straight to the next instruction — no load-use stalls at all	
Branch predictor	gshare table (256 × 2-bit) + 64-entry BTB + 8-entry RAS	Guesses branches so the CPU doesn't wait; trained from the execute stage	
Privilege modes	Machine (M), optional User (U)	Normal embedded mode + optional secure isolation	
Memory protection	Optional 8-region PMP (Smepmp)	Hardware guard that stops code from touching forbidden memory	
Debug	RISC-V External Debug 0.13 (JTAG) + triggers	Hardware breakpoints, single-stepping, register peeking	
Buses	Direct SRAM port + AXI4-Lite master	Can talk to on-chip memory or a standard SoC bus	
CoreMark	2.68 CoreMark/MHz (2.92 on some tests)	85% better cycle-efficiency than a multicycle core (1.43)	

---

3. The Pipeline, Stage by Stage

The core is split into 3 stages, separated by pipeline registers (F/X and X/W) that hold each instruction's data as it moves down the line.

Stage 1 — F: FETCH (get the instruction, guess the future)

What happens here: the core figures out which instruction comes next, fetches it from instruction memory, and expands compressed instructions.

- PC register: holds the address of the next instruction. It normally goes up by +4 bytes (a 32-bit instruction) or +2 (a compressed 16-bit one) — or jumps straight to a predicted target.
- Branch predictor — three helpers working together:
  - gshare BHT (256 × 2-bit): guesses taken or not-taken. It remembers the last 8 branch outcomes (the 8-bit history), mixes that with the PC using XOR, and uses the result to look up a 2-bit counter. If the counter leans "taken," the CPU bets on taken. Each counter is "saturating" — it takes a couple of wrong guesses to flip its mind, so one surprise doesn't destroy its memory.
  - BTB (64 entries): a speed-dial book of where branches jump to, so the target address is available instantly instead of being calculated.
  - RAS (8 entries): a mini stack of return addresses. When a function is called, its return address is pushed; when it returns, it's popped — so `ret` needs no calculation.
  - Training: the predictor is updated from branch results in Stage X, so it learns as the program runs.
- RVC expander: compressed instructions are 16-bit "zipped" versions that save memory. The expander sits at the front door and instantly unzips any 16-bit instruction into its full 32-bit form, so the decoder in Stage X only ever sees one size.
- Fetch control (the traffic cop):
  - If a 32-bit instruction is split across two memory words (an "overlapping page"), the controller does two quick reads and glues the halves together.
  - If something long is running (division, misaligned access), it freezes fetching so new instructions don't pile up.

Stage 2 — X: EXECUTE (decode, calculate, decide)

- Decoder: understands the full RV32IMACB + Zbc + Zcb instruction set.
- Immediate generator: pulls the constant number out of I/S/B/U/J format instructions and sign-extends it.
- Register file: the 32 general-purpose registers (x0–x31). Two read ports (rs1, rs2) work at the same time; one write port gets updated from Stage W.
- ALU + branch unit: does arithmetic, logic, bit-manipulation (Zba/Zbb/Zbs/Zbc), branch comparisons, and memory address calculation.
- Multiplier/Divider:
  - Multiplication is fast: built from combinational logic gates, it finishes in 1 cycle.
  - Division is slow: like long division on paper, the hardware shifts and subtracts one bit at a time — 32 steps for 32 bits, plus 1 extra step to fix the sign = 33 cycles. While it runs, it sends a "freeze" signal back to Fetch.
- CSR & trap unit (the control room):
  - CSRs are special registers that act like dashboard gauges: cycle counters, error flags, interrupt settings.
  - If something goes wrong (illegal instruction, timer, external interrupt), this unit takes over and jumps to a handler routine.
  - All branch decisions, jump targets, and traps are finalized here in Stage X.
- Redirect / flush (the undo button): if a branch was guessed wrong, Stage X tells the PC the right address and throws away the wrong instruction sitting in Stage F. Because the pipeline is only 3 stages deep, this costs exactly 1 wasted cycle.
- SECURE build extras: 8-region PMP (a security guard that blocks forbidden memory accesses) and debug triggers (watchpoints that fire when the PC hits an address or a variable is written — like a hardware wiretap for debugging).

Stage 3 — W: MEMORY / WRITEBACK (touch data, finish up)

- Load/Store unit: the only part that talks to data memory (`dmem_*` signals).
  - Handles byte/halfword/word loads and stores with byte-lane masking (`lb/lh/lw/sb/sh/sw`).
  - Supports atomic operations (AMO, LR/SC) for multi-core sync.
  - Handles misaligned accesses in hardware (see section 6).
- Writeback mux: picks what gets written into the destination register — ALU result, loaded data, return address (PC+4), mul/div result, or CSR value.
- Forwarding wire (W → X): the writeback output connects straight back to the ALU inputs in Stage X. A value produced now can be used by the very next instruction — no waiting.

---

4. How Hazards Are Handled

Problem	Hardware fix	
Instruction needs the result of a previous instruction (RAW dependency)	Forwarding wire W → X feeds the result in directly — zero load-use stalls	
Branch guessed wrong	1-cycle flush: discard the one wrong instruction in F, restart from the correct address	
Division takes 33 cycles	Divider stalls the Fetch stage until it's done	
Misaligned load/store	Split into 2 sequential memory beats; Fetch stalls 1 cycle	
Atomic memory op	Two-beat read-modify-write; pipeline stalls during the bus access	

Why there's no load-use penalty

In a classic 5-stage pipeline, an instruction right after a `lw` must stall 1 cycle because loaded data only exists at the end of the memory stage. In Takshaka:

1. `lw` is in Stage W, reading data from memory.
2. The next instruction is in Stage X needing that data.
3. The W→X forwarding wire hands the data straight into the ALU input — same cycle.
4. No bubble, no stall, no scoreboard logic needed.

---

5. The 32 Registers

Register	ABI name	Job	Survives calls?	
x0	zero	always 0; writes ignored	permanent	
x1	ra	return address (set by `jal`/`jalr`)	No (caller-saved)	
x2	sp	stack pointer, 16-byte aligned	Yes (callee-saved)	
x3	gp	points to global data	—	
x4	tp	points to thread-local storage	—	
x5–x7	t0–t2	scratch temporaries	No	
x8	s0/fp	saved register / frame pointer	Yes	
x9	s1	saved register	Yes	
x10–x11	a0–a1	arguments + return values	No	
x12–x17	a2–a7	more arguments	No	
x18–x27	s2–s11	more saved registers	Yes	
x28–x31	t3–t6	more temporaries	No	

---

6. Load/Store Design and Misaligned Access

Takshaka follows RISC-V's strict rule: only load/store instructions touch memory. `add`, `sub`, `and` etc. work purely on registers.

Normally a 32-bit word must start at an address divisible by 4.

- Standard cores: a misaligned access (e.g. address `0x1001`) triggers an alignment trap, and slow software has to fix it up.
- Takshaka: the Load/Store unit notices the misalignment and automatically splits it into two aligned accesses across two clock cycles, gluing the bytes together in hardware. No trap, no software fixup.

---

7. Instruction Set Extensions Supported

Takshaka implements RV32IMACB + Zicsr + Zcb + Zbc:

1. I — Base Integer: the mandatory core: loads, stores, add/sub, branches, jumps.
2. M — Multiply/Divide: `mul/mulh` in 1 cycle; `div/rem` in 33 cycles.
3. A — Atomics: `amoadd.w`, `amoswap.w`, `lr.w`/`sc.w` for multi-core synchronization.
4. C + Zcb — Compressed: 16-bit instructions, expanded in Stage F; shrinks code size 25–30%.
5. B (Zba, Zbb, Zbs) + Zbc — Bit manipulation:
   - Zba: shift-and-add in one cycle (`sh1add`, `sh2add`, `sh3add`) — handy for address math.
   - Zbb: `clz` (count leading zeros), `cpop` (count 1-bits), `rev8` (byte reverse), min/max.
   - Zbs: single-bit set/clear/invert/extract (`bset`, `bclr`, `binv`, `bext`).
   - Zbc: `clmul` (carry-less multiply) — speeds up CRC and cryptography.

---

8. Performance (CoreMark/MHz)

CoreMark/MHz measures how much work a core does per clock cycle — so it compares architecture, not clock speed. All numbers come from Verilator RTL simulation (1,000 iterations; 100 for 3-stage cores).

```
Multicycle core (Agni):       1.43 CoreMark/MHz   (CPI > 2.5)
5-stage pipeline (Gandiva):   2.34 CoreMark/MHz   (2–3 cycle branch penalty)
3-stage pipeline (Takshaka):  2.68 CoreMark/MHz   (1 cycle branch penalty)
Out-of-order core (Chakra):   3.84 CoreMark/MHz   (superscalar, runs several at once)
```

Why Takshaka hits 2.68:

1. Pipelining: 1 instruction retires per cycle instead of 1 per 3–4 cycles in a multicycle core.
2. Cheap mispredictions: CoreMark code is full of branches and loops. A 5-stage core loses 2–3 cycles per wrong guess; Takshaka loses only 1, because branches resolve early (in X) and only F has to be flushed.
3. Zero load-use stalls thanks to W→X forwarding.

Compared to industry multicycle cores

The lab's older multicycle cores (Agni 1.4269, Surya 1.3746, Kavacha 1.3181) already beat industry multicycle references like PicoRV32 (0.5531) by up to 2.58×. Takshaka builds on those datapaths and adds pipelined concurrency on top.

A thought of mine: the missing "wake-up" benchmark

Benchmarks like CoreMark run the CPU at 100% load. But real embedded devices (IoT sensors, appliances) sleep almost all the time and only wake up when an interrupt arrives. So a useful real-world metric would be interrupt wake-up latency: how many cycles from "signal arrives" to "first handler instruction runs?" A short 3-stage pipeline like Takshaka should wake up faster than a deep 5-stage core — measuring that would show off its responsiveness in low-power gadgets.

---

9. How It's Verified and Turned into Silicon

Verification

- Golden model co-simulation: the RTL runs the same program trace side-by-side with a reference Python ISA model (`tools/golden_rv32im.py`), comparing PC and register state instruction by instruction.
- RVFI (RISC-V Formal Interface): checks the retirement trace, traps, and that writes to x0 always read as zero.
- Self-checking test suites: cover instructions, JTAG debug, interrupts, and even running FreeRTOS multitasking.

Physical design (OpenROAD flow)

1. Synthesis (Yosys): SystemVerilog RTL → standard logic gates.
2. Floorplanning & placement: arrange macros and cells on the chip area.
3. Clock tree synthesis: build a balanced clock distribution network so every flip-flop gets the clock at the same time (minimizing skew).
4. Routing (TritonRoute): draw the metal wires connecting everything.
5. Static timing analysis (OpenSTA): check timing slack and compute the max safe clock frequency (Fmax).
6. GDSII output: the final mask layout sent to a foundry for manufacturing.

---

10. Things That Confused Me (and how I made sense of them)

1. Why is x0 hardwired to zero?
   My first thought: why waste one of 32 registers on a constant?
   The answer: it saves hardware. `addi rd, rs, 0` becomes a copy, `addi x0, x0, 0` becomes a nop, `sub rd, x0, rs` becomes a negate. One simple adder does five jobs — no extra instructions needed.

2. Why does a 3-stage core need branch prediction? Isn't that for big CPUs?
   My first thought: prediction is only for Intel-class monsters.
   The answer: even in 3 stages, the CPU has already fetched the next instruction while a branch is being resolved. Wrong guess = 1 wasted cycle. In a 10,000-iteration loop that's 10,000 wasted ticks. A 256-entry table guessing right means loops run without stumbling.

3. Load/store vs. Python variables.
   My first thought: `x = a + b` just "happens" in memory.
   The answer: RAM is physically far away across a bus. You can't do math out there. You `lw` the values into registers, add them in the ALU, and `sw` the result back.

4. Forwarding feels like cheating the clock.
   My first thought: if instruction 1 writes a register in stage 3, instruction 2 must wait until it's fully done.
   The answer: Takshaka just runs a wire from stage 3's output straight to stage 2's input. Like handing a tool directly to a teammate instead of putting it back in the toolbox first.

---

References: OR5 Labs Takshaka Technical Manual (or5.org/takshaka), "RISC-V Architecture Tutorial" by Nikhil Kumar Rajput, and the official RISC-V ISA specifications. Diagrams sourced from the repo's RTL files and benchmark runs.
