# RISC-V Single Cycle Core

## 📦 Core Component Breakdown

The hardware is modularly isolated into separate Verilog source blocks to maintain synthesis cleanliness and verification clarity:

### 1. Program Counter (PC)
* **Type:** Synchronous Sequential Logic
* **Description:** A 32-bit edge-triggered D-flip-flop register tracking the execution pointer. On every rising edge of `clk`, it transitions to the next target instruction address dictated by the instruction alignment routing logic. Instantly resets to a clear baseline on activation of the `rst` line.

### 2. Program Counter Adder (PC Adder)
* **Type:** Pure Combinational Logic
* **Description:** A dedicated 32-bit high-speed ripple-carry/look-ahead adder configured to constantly compute $PC + 4$. This derives the standard sequential instruction address parallel to the active cycle execution without introducing pipeline stall states.

### 3. Instruction Memory (`Instruction_Memory`)
* **Type:** Combinational Read Memory (Simulation Mode)
* **Description:** Houses the pre-compiled raw machine binary. Maps an incoming 32-bit address from the Program Counter to instantly output the corresponding 32-bit instruction word on the exact same cycle. Can be pre-loaded with execution code tables via Verilog `$readmemh` directives.

### 4. Register File (`Register_File`)
* **Type:** Hybrid Combinational-Sequential Module
* **Description:** Contains an internal array of 32 separate 32-bit wide registers. Implements **asynchronous dual-read ports** (`rs1`, `rs2`) allowing instantaneous source extraction to feed the ALU execution stage immediately, and a **synchronous single-write port** (`rd`) that securely commits data on the active edge when the `RegWrite` line is asserted by the Control Decoder.

### 5. Immediate Generator (`Imm_Gen`)
* **Type:** Pure Combinational Logic
* **Description:** Acts as an architectural parsing router. Extracts scattered immediate literal data bits from shifting fields inside the 32-bit instruction word. Synthesizes and expands them into a standard uniform signed 32-bit representation based on the runtime instruction type classification (**I-Type**, **S-Type**, **B-Type**, **U-Type**, **J-Type**).

### 6. Control Unit Top (Wrapper)
* **Type:** Logic Control Matrix
* **Description:** The central orchestration engine of the core. It decodes execution fields and dictates how multiplexers route internal paths and how memories behave. Internally decomposes into:
  * **Main Decoder:** Maps the instruction's 7-bit primary `opcode` field to drive core path selectors (`RegWrite`, `ALUSrc`, `MemWrite`, `ResultSrc`, `Branch`).
  * **ALU Decoder:** Intercepts the execution typing fields (`funct3`, `funct7`) paired with `ALUOp` tracking bits to isolate exactly which hardware operation the ALU must instantiate.

### 7. Arithmetic Logic Unit (ALU)
* **Type:** Pure Combinational High-Speed Execution Block
* **Description:** The computational heavy-lifter. Takes two 32-bit input operands and executes an operation derived by the incoming multi-bit `ALUControl` bus. Operates basic arithmetic (`ADD`, `SUB`), bitwise logic primitives (`AND`, `OR`, `XOR`), and logical shifts. Outputs a primary 32-bit calculated value along with a conditional status flag (`Zero`) utilized to resolve hardware branch decisions.

### 8. Data Memory (`Data_Memory`)
* **Type:** Synchronous Read/Write Data Matrix
* **Description:** System execution memory. Provides volatile workspace storage for running processes. Read accesses are combinational or managed immediately within the operational cycle, while execution write-backs (`sw`) latch data to memory on the active rising edge when `MemWrite` is dynamically enabled.

### 9. Multiplexers (MUX blocks)
* **Type:** Combinational Steering Logic
* **Description:** Distributed 2-to-1 routing paths deployed across the datapath. Key implementations include routing choice selectors between immediate generation offsets vs register data inputs before entering the ALU, and selecting whether the final write-back destination path loads raw data from the memory array or computation results directly out of the ALU.

---

## 🚀 Instruction Pipeline Step-by-Step Execution Sequence

To understand how the hardware processes data seamlessly in one clock cycle, observe how an **Arithmetic Immediate** instruction (`addi x5, x6, 25`) migrates structurally through the modules:

1. **Instruction Fetch:** The `PC` outputs the active instruction pointer. `Instruction_Memory` intercepts this value and outputs the machine code hex representation for `addi x5, x6, 25`.
2. **Instruction Decode & Register Read:** * The instruction bits are divided. `rs1` (pointing to `x6`) is sent to the `Register_File`, which instantly drops the contents of `x6` onto data bus line 1.
   * Simultaneously, the `Control_Unit` catches the 7-bit opcode, reading it as an I-type operation. It raises `ALUSrc` to 1 (telling the input MUX to select the immediate path) and sets `RegWrite` to 1.
3. **Immediate Generation:** The `Imm_Gen` block intercepts the upper instruction bits, pulls out the literal field string `25`, extends the sign bits to a full 32-bit representation, and sends it directly to the second port of the ALU input MUX.
4. **ALU Compute Block:** The ALU accepts the raw data out of register `x6`, accepts the sign-extended value `25` routed through the selection MUX, and runs an internal addition operation (`ADD`) dictated by the `ALUControl` bus line.
5. **Memory & Write-Back Commit:** The calculation outcome lands outside the input of the `Data_Memory`. Because it is an arithmetic immediate instruction, `MemWrite` remains strictly unasserted (`0`). The multiplexer routes the raw ALU calculation directly back around to the `Register_File` write-port. On the immediate next active rising edge of the system clock, the data value settles permanently inside register `x5`.

---

## 💻 Simulation & Behavioral Verification

This hardware core is verified extensively using behavioral testbenches to validate operational correctness.

### Verification Environment Recommendations
* **HDL Simulators:** Siemens QuestaSim / ModelSim, AMD Vivado Design Suite, or Icarus Verilog (`iverilog`).
* **Waveform Viewers:** GTKWave or Vivado Simulation Waveform Viewer.

### Running Simulation (Example via Icarus Verilog & GTKWave)
To compile the core modules and verify operations using a standard testbench wrapper:

```bash
# Compile all source modules and testbench
iverilog -o riscv_core_sim tb_riscv_top.v pc.v pc_adder.v instruction_memory.v register_file.v imm_gen.v control_unit.v main_decoder.v alu_decoder.v alu.v data_memory.v mux2.v

# Execute the simulation binary to generate a VCD waveform file
vvp riscv_core_sim

# View the structural trace execution paths visually
gtkwave waveforms.vcd
