
Lab 2: RISC-V Formal Verification Framework
===========================================

Adding Your Core:
------------------

The `cores/example` directory includes a set of files to attach the riscv-formal interface without CSRs to your core.
- `rvfi_wrapper.sv`: The top file that connects a core to riscv-formal checks.
- `otter_rvfi.v`:    A module that drives the riscv-formal interface for a core.
- `rvfi_defines.vh`: Defines useful macros for attaching RVFI to a core.

### Adding RVFI

Begin by copying your verilog source files into the `cores/example/rtl` directory.
In the top-module of your core (e.g. `otter_mcu.v` or something similar), instantiate the `otter_rvfi.v` module.
The purpose of each input to the `otter_rvfi.v` module are documented inside it.

```verilog

    //=========================//
    // INSIDE YOUR TOP MODULE:
    //=========================//

    //----------------------------------------------------------------------------//
    // RISC-V Formal Interface: Set of Bindings Used in Sim to Verify Correctness
    //----------------------------------------------------------------------------//

`ifdef RISCV_FORMAL

    otter_rvfi u_otter_rvfi (
        .i_clk             (/* clk signal */),
        .i_rst             (/* active-high reset */),
        .i_valid           (/* valid signal (1 if a new instruction will be fetched next cycle) */),
        .i_excp            (/* 1 if an interrupt occurs this cycle */),
        .i_trap            (/* 1 if a synchronous trap occurs this cycle */),

        .i_instrn          (/* instruction signal */),

        .i_is_reg_type     (/* add, sub, etc. */),
        .i_is_imm_type     (/* addi, slti, etc. */),
        .i_is_load         (/* lb, lh, lw, etc. */),
        .i_is_store        (/* sb, sh, sw */),
        .i_is_branch       (/* beq, bnez, etc. */),
        .i_is_jal          (/* jal */),
        .i_is_jalr         (/* jalr */),
        .i_is_csr_write    (/* csrrw (can be wired zero if unimplemented) */),

        .i_br_taken        (/* 1 if branch taken */),
        .i_pc_addr         (/* current pc address */),
        .i_br_tgt_addr     (/* jump/branch instruction target address */),
        .i_epc_addr        (/* exception address, mepc or mtvec depending on type */),

        .i_rfile_we        (/* rfile write-enable */),
        .i_rfile_w_data    (/* rfile write data */),
        .i_rfile_r_rs1     (/* rfile rs1 read data */),
        .i_rfile_r_rs2     (/* rfile rs2 read data */),

        .i_dmem_sel        (/* byte mask for data memory */),
        .i_dmem_addr       (/* address read from/written to in dmem */),
        .i_dmem_r_data     (/* data read from dmem */),
        .i_dmem_w_data     (/* data written to dmem */),

        `RVFI_INTERCONNECTS
        ._dummy(1'b0)
    );

`endif

```

Once added, use the `RVFI_OUTPUTS` macro to add the the RVFI signals to the outputs of your top module.

```verilog

//=================================================//
// YOUR TOP MODULE SHOULD LOOK SOMETHING LIKE THIS
//=================================================//

`include "rvfi_defines.vh"

module otter_mcu #(
    parameter RESET_VEC = 32'h0
) (
    input             i_clk,
    input             i_rst, 
    input      [31:0] i_intrpt, 

`ifdef RISCV_FORMAL
    `RVFI_OUTPUTS
`endif

    input      [31:0] i_imem_r_data,
    output     [31:0] o_imem_addr,

    input      [31:0] i_dmem_r_data,
    output reg        o_dmem_re,
    output reg        o_dmem_we,
    output     [3:0]  o_dmem_sel,
    output     [31:0] o_dmem_addr,
    output     [31:0] o_dmem_w_data
);
```

Inside `rvfi_wrapper.sv`, update the mcu instantiation to match the name and signals used by your core.

```verilog

//=================================//
// UPDATE THIS TO MATCH YOUR CORE:
//=================================//

(* keep *) `rvformal_rand_reg [31:0] w_intrpt;
(* keep *) `rvformal_rand_reg [31:0] w_imem_r_data;
(* keep *) `rvformal_rand_reg [31:0] w_dmem_r_data;

(* keep *) wire [31:0] w_imem_addr;
(* keep *) wire        w_dmem_re;
(* keep *) wire        w_dmem_we;
(* keep *) wire [ 3:0] w_dmem_sel;
(* keep *) wire [31:0] w_dmem_addr;
(* keep *) wire [31:0] w_dmem_w_data;

otter_mcu # (
    .RESET_VEC('0)
) u_otter_mcu (
    .i_clk          (clock),
    .i_rst          (reset),
    .i_intrpt       (w_intrpt),

    `RVFI_INTERCONNECTS

    .i_imem_r_data  (w_imem_r_data),
    .o_imem_addr    (w_imem_addr),

    .i_dmem_r_data  (w_dmem_r_data),
    .o_dmem_re      (w_dmem_re),
    .o_dmem_we      (w_dmem_we),
    .o_dmem_sel     (w_dmem_sel),
    .o_dmem_addr    (w_dmem_addr),
    .o_dmem_w_data  (w_dmem_w_data)
);
```

With those additions, use `genchecks.py` to generate the checks for the core, and try running the checks with `make`.

```bash
# Enter the cores directory
cd cores
./genchecks.py --corename example --cfgname checks --basedir ~/riscv-formal

cd example
make -C checks -j(nprocs)
```

Dicussion
---------

After getting the checks running, look at the counterexample waveforms for at least three failing tests

For each, include a screenshot of the counterexample and discuss:

1. What assertion in the check failed; what is it testing for?
2. What behavior caused the failing example, was it a misinterpretation of the spec, wiring issue, etc.?
3. What changes do you think would fix the issue, and why?

*Note: if you are lucky enough to not fail three checks, you may introduce a bug into one of your modules and discuss the check that detects the fault.*

Deliverables
------------

Please include the following in your submission:

1. Your modified verilog file that instantiates and connects to the `otter_rvfi`.
2. The updated `rvfi_wrapper.sv` that connects to your core
3. A separate `.pdf` that contains your responses to the discussion questions.

