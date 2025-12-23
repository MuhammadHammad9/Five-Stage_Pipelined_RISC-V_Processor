# Five-Stage Pipelined RISC-V Processor

![Language](https://img.shields.io/badge/Language-Verilog_HDL-blue?style=flat-square) ![Architecture](https://img.shields.io/badge/Architecture-RISC--V_RV32I-orange?style=flat-square) ![Status](https://img.shields.io/badge/Status-Verified-success?style=flat-square)

## 📖 Abstract

This project involves the design and implementation of a **32-bit Five-Stage Pipelined Processor** based on the RISC-V Instruction Set Architecture (ISA). Implemented in **Verilog HDL**, the processor targets the **RV32I** base integer instruction set and features a robust **Hazard Management Unit** to maintain correct program execution.

The core engineering challenge addressed in this design is the resolution of pipeline hazards. The processor employs **Data Forwarding**, **Load-Use Stalling**, and **Control Flushing** to minimize pipeline stalls while ensuring data integrity.

## 👥 Authors

* **Muhammad Hammad** (u2023420)
* **Muhammad Qasim** (u2023488)

*Department of Computer Engineering, Ghulam Ishaq Khan Institute of Engineering Sciences and Technology (GIKI), Pakistan.*

---

## 🏗️ Architectural Design

The processor divides instruction execution into five distinct stages, separated by pipeline registers to isolate data and ensure synchronization. 

1.  **Instruction Fetch (IF):** Fetches instructions from memory and handles the Program Counter (PC) updates (Next PC or Branch Target).
2.  **Instruction Decode (ID):** Decodes instructions, reads source operands from the Register File, and handles sign extension.
3.  **Execute (EX):** Performs ALU operations and calculates branch target addresses. This stage also contains the **Forwarding Muxes**.
4.  **Memory Access (MEM):** Interfaces with Data Memory for Load (`lw`) and Store (`sw`) operations.
5.  **Write Back (WB):** Selects the final result (ALU output or Memory data) to write back to the Register File.

### Supported Instruction Set

The core supports a functional subset of the RV32I ISA:

| Type | Instructions | Opcode | Description |
| :--- | :--- | :--- | :--- |
| **R-Type** | `add`, `sub`, `and`, `or`, `slt` | `0110011` | Arithmetic & Logical operations |
| **I-Type** | `addi` | `0010011` | Immediate arithmetic |
| **I-Type** | `lw` | `0000011` | Load Word from Memory |
| **S-Type** | `sw` | `0100011` | Store Word to Memory |
| **B-Type** | `beq` | `1100011` | Branch if Equal |

---

## ⚠️ Hazard Management Unit

The most critical component of this design is the `hazard_unit.v`, which automatically detects and resolves three types of hazards.

### 1. Data Hazards (Forwarding)
* **Problem:** An instruction in the EX stage needs data computed by an instruction currently in the MEM or WB stage (Read-After-Write).
* **Solution:** The Hazard Unit controls 3-way Muxes in the EX stage to bypass the Register File.
    * **ForwardA/B = `10`:** Forwards from **EX/MEM** pipeline register (Most recent ALU result).
    * **ForwardA/B = `01`:** Forwards from **MEM/WB** pipeline register (Value pending write-back).

### 2. Load-Use Hazards (Stalling)
* **Problem:** An instruction attempts to read a register immediately after a `lw` instruction loads it. Forwarding is impossible because the data is still being read from memory.
* **Solution:** The Hazard Unit detects this condition:

    ```verilog
    if (MemtoRegE & (RD_E != 0) & ((RD_E == Rs1_D) | (RD_E == Rs2_D)))
    ```

    It then asserts a Stall:
    1.  **PCWrite = 0:** Freezes the PC.
    2.  **IF_ID_Write = 0:** Freezes the Fetch stage register.
    3.  **ID_EX_Flush = 1:** Injects a NOP (bubble) into the Execute stage.

### 3. Control Hazards (Flushing)
* **Problem:** The pipeline fetches instructions sequentially, but a `beq` (Branch) instruction might change the PC flow.
* **Solution:** Branch outcomes are evaluated in the EX stage. If a branch is taken, the `PCSrcE` signal triggers a **Flush** of the IF/ID register, discarding the erroneously fetched instructions.

---

## 🧪 Verification

The design is verified using a self-checking testbench `tb_Pipeline_top.v` using ModelSim/Vivado.

### Hazard Test Program
The `Instruction_Memory.v` contains a hardcoded assembly program specifically designed to stress-test the hazard logic:

```assembly
0: addi x7, x0, 2      // Init x7 = 2
1: addi x6, x0, 0x123  // Init Base Address
2: sw   x7, 0(x6)      // Store 2 to Mem[0x123]
3: lw   x8, 0(x6)      // Load 2 from Mem to x8
4: add  x9, x8, x7     // HAZARD: Load-Use on x8. Requires STALL.
5: addi x4, x0, 10     // Normal execution
6: slli x9, x9, 1      // Shift x9

---

### Simulation Results
Waveform analysis confirms the following behaviors:

* **Stall Insertion:** The `ID_EX_Flush` signal goes High for one cycle when instruction 4 (`add`) follows instruction 3 (`lw`), confirming Load-Use detection.
* **Data Integrity:** The final register values reflect the correct sequential execution of the program, verifying that forwarding and flushing logic functioned correctly.

**Final Register Check (from Testbench):**

* `[PASS]` Register x5 contains 5
* `[PASS]` Register x6 contains 3
* `[PASS]` Register x7 contains 8

---

## 🚀 Getting Started

### Clone the Repository
```bash
git clone [https://github.com/your-username/RISCV-Pipeline-Processor.git](https://github.com/your-username/RISCV-Pipeline-Processor.git)

### Open Project
1. Open the files in **Vivado** or **ModelSim**.
2. Set `tb_Pipeline_top.v` as the Top-Level Simulation Module.

### Simulation
1. Run Behavioral Simulation for **200ns**.
2. Observe the `[PASS]` / `[FAIL]` signals in the Tcl Console.

## 📂 File Structure

* `Pipeline_top.v` - Top-level wrapper connecting all stages.
* `hazard_unit.v` - Logic for Forwarding, Stalling, and Flushing.
* `fetch_cycle.v` - Instruction Fetch stage and PC logic.
* `decode_cycle.v` - Decode stage, Control Unit, and Register File.
* `execute_cycle.v` - ALU, Muxes, and Branch Logic.
* `memory_cycle.v` - Data Memory access.
* `writeback_cycle.v` - Result selection.
* `tb_Pipeline_top.v` - Testbench for verification.

## 🔮 Future Work

* Implementation of **Dynamic Branch Prediction** (2-bit saturating counter).
* Integration of **Instruction and Data Caches (L1)**.
* Expansion of ISA to support the **'M' extension** (Multiplication/Division).
