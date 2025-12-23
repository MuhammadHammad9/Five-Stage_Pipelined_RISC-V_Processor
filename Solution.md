# ✅ Solution: Data Forwarding Unit

We solve the data hazard problem without stopping the processor by adding **bypass paths**. The result is tapped from the pipeline registers (`EX/MEM` or `MEM/WB`) and fed directly back to the ALU inputs.

## 🧠 Logic Breakdown

The **Forwarding Unit** controls two multiplexers (`SrcAE_Mux`, `SrcBE_Mux`) that select the ALU operands. It checks three conditions:
1.  **Dependency:** Does the Source Register (`Rs`) of the current instruction match the Destination Register (`Rd`) of a future instruction?
2.  **Validity:** Is the future instruction actually writing to a register (`RegWrite == 1`)?
3.  **Zero Check:** Is the destination register `x0`? (Writes to `x0` are always ignored).

### Detailed Forwarding Conditions

| Hazard Source | Pipeline Register | Logic Condition (Verilog) | Mux Action |
| :--- | :--- | :--- | :--- |
| **Execution Hazard** | EX/MEM | `if (Rs_E == Rd_M) && RegWrite_M` | `Forward = 10` (Use ALU Result from MEM) |
| **Memory Hazard** | MEM/WB | `if (Rs_E == Rd_W) && RegWrite_W` | `Forward = 01` (Use Result from WB) |
| **No Hazard** | N/A | `else` | `Forward = 00` (Use RegFile Output) |

> **Critical Priority:** If both the MEM stage and WB stage contain writes to the same register (a "Double Data Hazard"), the MEM stage result is more recent. Therefore, the **MEM hazard logic must take precedence** over the WB hazard logic.

### 💻 Verilog Implementation

```verilog
always @(*) begin
    // Forwarding for Input A (Rs1)
    if ((rs1_E == rd_M) && RegWrite_M && (rd_M != 0)) 
        ForwardAE = 2'b10;  // Priority 1: Forward most recent result (EX/MEM)
    else if ((rs1_E == rd_W) && RegWrite_W && (rd_W != 0)) 
        ForwardAE = 2'b01;  // Priority 2: Forward older result (MEM/WB)
    else 
        ForwardAE = 2'b00;  // No hazard: Use Register File

    // Forwarding for Input B (Rs2)
    if ((rs2_E == rd_M) && RegWrite_M && (rd_M != 0)) 
        ForwardBE = 2'b10; 
    else if ((rs2_E == rd_W) && RegWrite_W && (rd_W != 0)) 
        ForwardBE = 2'b01; 
    else 
        ForwardBE = 2'b00; 
end
