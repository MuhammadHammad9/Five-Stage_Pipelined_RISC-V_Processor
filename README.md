# Five-Stage Pipelined RISC-V Processor

![Language](https://img.shields.io/badge/Language-Verilog_HDL-blue?style=flat-square)
![Architecture](https://img.shields.io/badge/Architecture-RISC--V_RV32I-orange?style=flat-square)
![Status](https://img.shields.io/badge/Status-Verified-success?style=flat-square)

---

## 📖 Abstract

This project presents the design and implementation of a **32-bit Five-Stage Pipelined Processor** based on the **RISC-V Instruction Set Architecture (ISA)**. The processor is fully implemented in **Verilog HDL** and targets the **RV32I base integer instruction set**.

A major focus of this work is ensuring **correct program execution in a pipelined environment**, where multiple instructions overlap in execution. To address this challenge, the processor integrates a dedicated **Hazard Management Unit** that handles **data hazards**, **load-use hazards**, and **control hazards**. Techniques such as **data forwarding**, **pipeline stalling**, and **instruction flushing** are used to reduce pipeline stalls while maintaining correctness, thereby improving overall performance and approaching an ideal **CPI of 1**.

---

## 👥 Authors

- **Muhammad Hammad** (u2023420)  
- **Muhammad Qasim** (u2023488)  

**Department of Computer Engineering** **Ghulam Ishaq Khan Institute of Engineering Sciences and Technology (GIKI), Pakistan**

---

## 🏗️ Architectural Design

The processor follows a classic **five-stage pipeline architecture**, where instruction execution is divided into independent stages separated by pipeline registers (`IF/ID`, `ID/EX`, `EX/MEM`, `MEM/WB`). This structure allows up to five instructions to be active simultaneously.

### Detailed Pipeline Stages

#### 1. Instruction Fetch (IF)
* **Function:** Fetches the instruction from Instruction Memory using the Program Counter (PC).
* **Key Components:** PC, Instruction Memory, PC Adder, PC Mux.
* **Operation:** The PC addresses the memory to retrieve the machine code. The PC is typically updated to `PC + 4`. However, if a branch or jump is taken in the Execute stage, the PC Mux updates the PC to the target address (`PCTarget`).

#### 2. Instruction Decode (ID)
* **Function:** Decodes the 32-bit instruction and reads source operands.
* **Key Components:** Register File (`RegFile`), Control Unit, Sign Extender.
* **Operation:** * **Control Unit:** Generates signals (RegWrite, ALUSrc, MemWrite, etc.) based on the Opcode.
    * **Register File:** Asynchronously reads data from source registers `Rs1` and `Rs2`.
    * **Sign Extender:** Converts 12-bit (I-Type/S-Type/B-Type) immediates into 32-bit values for arithmetic operations.

#### 3. Execute (EX)
* **Function:** Performs arithmetic/logical operations and calculates branch addresses.
* **Key Components:** ALU, ALU Control, Branch Adder, Forwarding Multiplexers.
* **Operation:** * **ALU:** Executes the operation (`add`, `sub`, `xor`, etc.) on operands selected by the Forwarding Multiplexers.
    * **Branch Logic:** The Branch Adder computes `PC + Imm` to determine where to jump if a branch condition (e.g., `Zero` flag from ALU) is met.

#### 4. Memory Access (MEM)
* **Function:** Interfaces with Data Memory.
* **Key Components:** Data Memory.
* **Operation:** * **Load (`lw`):** Reads data from the address computed by the ALU.
    * **Store (`sw`):** Writes data (`Rs2`) to the address computed by the ALU.
    * **Others:** For R-type or I-type arithmetic, this stage simply passes the ALU result to the next register.

#### 5. Write Back (WB)
* **Function:** Updates the Register File.
* **Key Components:** Result Mux.
* **Operation:** Selects the final result based on the instruction type (ALU Result for arithmetic, Read Data for loads) and writes it back to the destination register (`Rd`).

---

## 📜 Supported Instruction Set

The processor supports a representative subset of the **RV32I instruction set**, covering arithmetic, memory, and control-flow operations.

| Type | Instructions | Opcode | Description |
|-----:|-------------|--------|-------------|
| **R-Type** | `add`, `sub`, `and`, `or`, `slt` | `0110011` | Register-Register Arithmetic/Logical |
| **I-Type** | `addi` | `0010011` | Immediate Arithmetic |
| **I-Type** | `lw` | `0000011` | Load Word from Memory |
| **S-Type** | `sw` | `0100011` | Store Word to Memory |
| **B-Type** | `beq` | `1100011` | Branch if Equal |

---

## ⚠️ Hazard Management Unit

The **Hazard Management Unit (`hazard_unit.v`)** is the core component responsible for maintaining correct execution. It automatically detects and resolves three major classes of hazards.

### 1. Data Hazards (Forwarding)

**Problem:** An instruction in the **Execute (EX)** stage requires a value that is currently being computed in the **Memory (MEM)** or **Write-Back (WB)** stage (RAW Hazard).

**Solution:** **Forwarding (Bypassing)**. The unit steers the ALU input multiplexers to grab the data from the pipeline registers rather than the stale value from the Register File.

* **Forwarding Logic:**
    * **`ForwardAE = 2'b10`:** Forward from **EX/MEM** (The instruction immediately preceding current one).
    * **`ForwardAE = 2'b01`:** Forward from **MEM/WB** (The instruction 2 cycles ahead).

### 2. Load-Use Hazards (Stalling)

**Problem:** If a `lw` instruction is immediately followed by an instruction that uses the loaded value, forwarding is impossible because the data is only available after the MEM stage.

**Solution:** **Pipeline Stall (Bubble)**. The hazard unit detects this dependency and pauses the early pipeline stages for one cycle.

**Detection Logic:**
The hazard exists if the instruction in the Execute stage (`EX`) is a load (checking `ResultSrc` or `MemRead`) and its destination register matches either source register of the instruction in Decode (`ID`).

```verilog
// Detect Load-Use Hazard
// ResultSrcE[0] is typically 1 for Load instructions
wire lwStall;

assign lwStall = ResultSrcE[0] & ((RdE == Rs1D) | (RdE == Rs2D));

// Apply Stall Signals
assign StallF = lwStall; // Freeze PC
assign StallD = lwStall; // Freeze IF/ID Register
assign FlushE = lwStall; // Flush ID/EX Register (Insert NOP)
