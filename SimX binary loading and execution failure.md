# SimX binary loading and execution failure

**Created:** 4/30/2026 5:24:15  
**Updated:** 4/30/2026 20:59:09  
**Exported:** 5/13/2026 16:26:36  
**Link:** [https://claude.ai/chat/3ee4ff65-b2ec-4938-862b-99ba72b44bc0](https://claude.ai/chat/3ee4ff65-b2ec-4938-862b-99ba72b44bc0)  

## Prompt:
4/30/2026, 5:24:17 AM

i need your help to solve the problems facing me when trying to run all uvm .
i have a problem when trying to pass the program_simple.hex to simx 
it gives me that error 
{
# [TEST] Binary file: /home/stev_teto_22/vortex_uvm/uvm_env/ref_model/program_simple.bin
# [TEST] Load address: 0x0000000080000000
# [TEST] Waveform dumping enabled
#
# --- Configuration ---
#   Cores:        2
#   Warps:        4
#   Threads:      4
#   Startup Addr: 0x0000000080000000
#   Load Addr:    0x0000000080000000
#   Result Addr:  0x0000000080010000
#   DCR Addr0:    0x00000001 (corrected)
#   DCR Addr1:    0x00000002 (corrected)
#
# [TEST] Step 1: Initializing SimX...
# [SimX-DPI] ========================================
# [SimX-DPI] Initializing SimX Golden Model
# [SimX-DPI] Cores=2, Warps=4, Threads=4
# [SimX-DPI] Architecture created successfully
# [SimX-DPI] Creating RAM: capacity=0x100000000, page_size=0x1000
# [SimX-DPI] RAM created successfully
# [SimX-DPI] Attaching RAM to processor...
# [SimX-DPI] RAM attached successfully
# [SimX-DPI] Verifying RAM...
# [SimX-DPI] RAM verification PASSED
# [SimX-DPI] Initialization successful
# [SimX-DPI] ========================================
#
# [TEST] Step 2: Configuring DCRs...
# [TEST] Writing DCR 0x00000001 = 0x80000000
# [SimX-DPI] DCR Write: addr=0x1, value=0x80000000
# [SimX-DPI] DCR write successful
# [TEST] DCR configuration complete
#
# [TEST] Step 3: Loading kernel binary...
# [SimX-DPI] Loaded '/home/stev_teto_22/vortex_uvm/uvm_env/ref_model/program_simple.bin' (52 bytes) at 0x80000000
# [TEST] Kernel loaded successfully
#
# [TEST] Step 4: Running SimX to completion...
# [SimX-DPI] ========================================
# [SimX-DPI] Running processor to completion...
# [SimX-DPI] Startup address: 0x80000000
# lookup of 0x6365705380000000 failed.
# [SimX-DPI] Execution finished
# [SimX-DPI] Exit code: -1
# [SimX-DPI] ========================================
# ** Error: [TEST] Execution failed with exit code: -1
#    Time: 0 ns  Scope: test_bin File: test_[bin.sv](http://bin.sv) Line: 114
#
# [TEST] Step 5: Verifying results...
# [SimX-DPI] Read 1024 bytes from 0x80010000
# [TEST] Read 1024 bytes from result area at 0x0000000080010000
# [TEST] First 16 bytes: 0d f0 ad ba 0d f0 ad ba 0d f0 ad ba 0d f0 ad ba
# [SimX-DPI] ========================================
# [SimX-DPI] Cleaning up SimX...
# [SimX-DPI] Processor deleted
# [SimX-DPI] RAM deleted
# [SimX-DPI] Arch deleted
# [SimX-DPI] Cleanup complete
# [SimX-DPI] ========================================
#
# ================================================================================
#   Test FAILED (exit code: -1)
# ================================================================================
# End time: 05:07:49 on Apr 30,2026, Elapsed time: 0:00:02
# Errors: 1, Warnings: 0
}
that is the output i need to understand it + how i can solve it



## Response:
4/30/2026, 5:25:21 AM



````plaintext
Thought process: Diagnosed XLEN_64 mismatch causing uninitialized startup address corruption.

The error message "lookup of 0x6365705380000000 failed" is the key clue here. Let me analyze this.

`0x6365705380000000` - let me decode this:
- `0x63` = 'c'
- `0x65` = 'e'
- `0x70` = 'p'
- `0x53` = 'S'

So it reads "cepS" + `0x80000000` - this looks like the startup address `0x80000000` is being concatenated with some stack/PC value incorrectly.

Wait, let me look at this more carefully. The address `0x6365705380000000` - in little endian:
- bytes: 00 00 00 80 53 70 65 63
- ASCII of 53 70 65 63: 'S', 'p', 'e', 'c' = "Spec" reversed = "cepS"

This looks like a string "Spec" followed by `0x80000000`. This is a classic sign of a stack corruption or wrong startup address interpretation.

Actually, looking at the hex: `0x6365705380000000`
- 0x63 = 'c'
- 0x65 = 'e' 
- 0x70 = 'p'
- 0x53 = 'S'
- then 0x80000000

Hmm, this might be related to XLEN_64 vs XLEN_32 issue. The startup address in 64-bit mode should be `0x080000000` not `0x80000000`.

Wait, let me re-read the error. "lookup of 0x6365705380000000 failed." - this is in the RAM lookup. The processor is trying to fetch from address `0x6365705380000000` which is clearly garbage.

Looking at the Makefile:
`​`​`
ARCH_CONFIGS  = -DNUM_CLUSTERS=1
ARCH_CONFIGS += -DNUM_CORES=2
ARCH_CONFIGS += -DNUM_WARPS=4
ARCH_CONFIGS += -DNUM_THREADS=4
ARCH_CONFIGS += -DXLEN_64
`​`​`

The DPI is compiled with XLEN_64! But the program is compiled for RV32 (startup address 0x80000000). In XLEN_64 mode, the startup address is 0x080000000 (9 hex digits).

But more importantly, the issue is that the program is being loaded correctly at 0x80000000, but the processor is fetching from 0x6365705380000000 which is garbage.

Let me think about this differently. The bytes `63 65 70 53` in ASCII are 'c', 'e', 'p', 'S'. In little-endian 32-bit, this would be the word `0x53706563` = "Spec". 

Actually wait - `0x6365705380000000` - let me look at this as bytes in memory order:
00 00 00 80 53 70 65 63

…
````

## Root Cause Analysis

The error `lookup of 0x6365705380000000 failed` is the key. Let me decode it:

```
0x6365705380000000
  ↑↑↑↑↑↑↑↑
  63 65 70 53  →  ASCII: 'c','e','p','S'  →  "Spec" garbage in upper 32 bits
              80000000  ←  your actual startup address
```

**The processor fetched from a corrupted 64-bit address because DCR_STARTUP_ADDR1 (the upper 32 bits) was never written.**

### Why This Happens

Your `Makefile` compiles SimX with `XLEN_64`:
```makefile
ARCH_CONFIGS += -DXLEN_64   # ← SimX runs in 64-bit mode
```

In XLEN_64 mode, SimX reads the full 64-bit PC as:
```
PC = (DCR_STARTUP_ADDR1 << 32) | DCR_STARTUP_ADDR0
   = (UNINITIALIZED_GARBAGE << 32) | 0x80000000
   = 0x6365705380000000   ← exactly what you see
```

Your test only writes `DCR 0x001 = 0x80000000` but **never writes DCR 0x002 (ADDR1)**.

---

## Fix 1: Immediate Fix in `test_bin.sv`

Add the missing DCR write. Find the DCR configuration block in `test_bin.sv` around line where `Writing DCR 0x00000001` happens and add:

```systemverilog
// Write DCR_STARTUP_ADDR0 (lower 32 bits)
simx_dcr_write(32'h001, 32'h80000000);

// ADD THIS — Write DCR_STARTUP_ADDR1 (upper 32 bits = 0 for RV32 programs)
simx_dcr_write(32'h002, 32'h00000000);
```

---

## Fix 2: Makefile XLEN Consistency

Your `program_simple.bin` is a **32-bit RISC-V program** (startup at `0x80000000` — RV32 convention). Your SimX is compiled as **64-bit**. Either:

**Option A (recommended):** Match SimX to the program — remove `XLEN_64` from your Makefile:

```makefile
# In ref_model/Makefile, change:
ARCH_CONFIGS  = -DNUM_CLUSTERS=1
ARCH_CONFIGS += -DNUM_CORES=2       # keep or change as needed
ARCH_CONFIGS += -DNUM_WARPS=4
ARCH_CONFIGS += -DNUM_THREADS=4
# ARCH_CONFIGS += -DXLEN_64        # ← REMOVE or comment out for RV32 programs
```

**Option B:** Keep XLEN_64 but use the correct 64-bit startup address `0x080000000` (note: 9 hex digits) and write both DCRs:

```systemverilog
simx_dcr_write(32'h001, 32'h80000000);   // ADDR0: lower 32 bits
simx_dcr_write(32'h002, 32'h00000008);   // ADDR1: upper 32 bits of 0x080000000
```

---

## Fix 3: SimX Integration in UVM (`simx_pkg.sv`)

Looking at your `simx_pkg.sv` `run_phase`, it only writes `DCR 0x001`:

```systemverilog
// In simx_pkg.sv run_phase, find:
simx_dcr_write(32'h001, cfg.startup_addr[31:0]);

// ADD after it:
simx_dcr_write(32'h002, cfg.startup_addr[63:32]);  // ← upper 32 bits
simx_dcr_write(32'h003, 32'h0);  // ARG0
simx_dcr_write(32'h004, 32'h0);  // ARG1
simx_dcr_write(32'h005, 32'h0);  // MPM_CLASS
```

---

## Fix 4: XLEN Consistency in `vortex_config.sv`

Your `vortex_config_pkg.sv` has a conditional:

```systemverilog
`ifdef XLEN_64
    parameter VX_MEM_ADDR_WIDTH = 42;
    parameter AXI_ADDR_WIDTH    = 48;
`else
    parameter VX_MEM_ADDR_WIDTH = 26;   // ← this is what runs with RV32 programs
    parameter AXI_ADDR_WIDTH    = 32;
`endif
```

Make sure the define used when compiling UVM matches what the Makefile uses for SimX. In `run_vortex_uvm_enhanced.sh`, the RTL compile currently does NOT pass `+define+XLEN_64` by default (only `FPU_FPNEW`, `NUM_*` etc.), so RTL and UVM are RV32. But SimX in the Makefile is XLEN_64. **This mismatch is the core problem.**

---

## Summary of All Changes Needed

| File | Location | Change |
|------|----------|--------|
| `ref_model/Makefile` | `ARCH_CONFIGS` | Remove `-DXLEN_64` for RV32 programs |
| `ref_model/test_bin.sv` | DCR setup block | Add `simx_dcr_write(32'h002, 32'h0)` |
| `uvm_env/ref_model/simx_pkg.sv` | `run_phase` after DCR write | Add ADDR1, ARG0, ARG1, MPM_CLASS DCR writes |

The single root cause is **XLEN mismatch + missing DCR_STARTUP_ADDR1 write**. Fix the Makefile `XLEN_64` first, then add the missing DCR write — that resolves the `lookup of 0x6365705380000000 failed` error completely.



## Prompt:
4/30/2026, 12:18:26 PM

> File: outptu.txt

now i run this command  ./scripts/run_vortex_uvm_[enhanced.sh](http://enhanced.sh)     --test=vortex_smoke_test     --program=$(pwd)/uvm_env/agents/host_agent/program_simple.hex     --clean
and that is the output report "output.txt" .
study it hard and tell me what is missing and what is hidden errors , problems , missing things or not done as expected and also find the case of that from the project files



## Response:
4/30/2026, 12:25:32 PM



````plaintext
Thought process

Let me read the output file first.
````

````plaintext
View: Read the output file
````

````plaintext
Thought process

Let me read more of the truncated portion of the file to understand the full picture.
````

````plaintext
View: Read the truncated middle section
````

````plaintext
View: Read middle simulation section
````

````plaintext
View: Read the rest of simulation output
````

````plaintext
View: Read the key post-reset execution section
````

````plaintext
Thought process: Analyzed test failures, traced configuration mismatches, identified verification gaps.

Now I have a comprehensive picture. Let me analyze all the issues:

1. **CRITICAL: RESET is absurdly too long** - The VX_schedule RESET messages go from ~85000ns to ~5615000ns (5.615ms). That's 561 clock cycles at 10ns period. But looking at the parameter: `RESET_CYCLES = vortex_config_pkg::RTL_RESET_DELAY * 50` = 8 * 50 = 400 cycles. At 10ns = 4000ns. But we see the last RESET message at ~5615000 (but that's from the tb_top timestamping). Actually at 5615000ns with 10ns period that's 561,500... wait, let me recalculate.

Wait - the messages go up to 5615000ns. That's 5615 µs. At 10ns clock period, that's 561,500 clock cycles. But the total_cycles reported is only 163. This doesn't add up until I understand the RESET is holding for a very long time (from 5ns to 5615000ns).

Actually wait - looking more carefully: the `VX_schedule[0] RESET` messages span from 85000ns to well past 3,600,000ns. Given 10ns period, 85000ns = 8500 cycles into simulation. The reset is lasting until the very end (5615000ns = 561,500 cycles).

But wait - `Total Cycles: 163` is what tb_cycle_count shows. And the test passes. Let me re-examine.

Actually, looking at the `TB_TOP` reset sequence:
`​`​`
RESET_CYCLES = vortex_config_pkg::RTL_RESET_DELAY * 50; // 400 cycles
`​`​`
And `RTL_RESET_DELAY = 8`, so `RESET_CYCLES = 400 cycles = 4000ns`.

But from the output, the DCRs are written at ns 45000, 65000, 85000, 105000, 125000, 135000 (when "all 5 base DCRs written") and then reset is still asserted for a very long time.

Actually wait - `repeat(RESET_CYCLES - 15) @(posedge clk)` = 385 cycles after the 5-cycle initial + 10 cycles for DCR writes (5 DCRs × 2 clocks each = 10 cycles), so total = 5 + 10 + 385 = 400 cycles, releasing at 4000ns.

But the VX_schedule RESET messages continue past 85000ns!! Wait - actually I think they continue indefinitely. Let me understand why.

Looking at the RTL more carefully: the `VX_schedule.sv` prints `RESET` every 10ns even after reset is released. …
````

Now I have a complete picture. Let me present a structured analysis of every issue found.

---

## Complete Issue Analysis

### CRITICAL — Simulation is a Vacuous PASS

**Issue 1: Test "passes" with 1 instruction in 163 cycles — meaningless verification**

```
Cycles: 163    Instructions: 1    Memory Reads: 1    Memory Writes: 0
```

The DUT fetches 1 instruction (first CSR read from `program_simple.hex`), then `busy=0`. The TB_TOP `always_ff` immediately fires:
```systemverilog
// vortex_tb_top.sv — the problem:
if (tb_execution_started && !tb_execution_complete && !vif.status_if.busy) begin
    tb_execution_complete <= 1;   // ← fires instantly because busy=0 after vx_wstop
```
One AXI read → `tb_execution_started=1` → `busy=0` → `tb_execution_complete=1` in 2 clock cycles. No actual program outcome was verified.

---

**Issue 2: Golden Model (SimX) ran but performed ZERO comparisons — completely bypassed**

```
SimX Enabled: YES    SimX Ran: YES    Comparisons: 0    Passed: 0    Failed: 0
```

Followed by **128 identical warnings** (lines 1967–1970):
```
UVM_WARNING [SCOREBOARD] Result addr 0x801003e0 not written by DUT — skipping
```

**Root cause:** In `vortex_config.sv`, `set_defaults_from_vx_config()`:
```systemverilog
result_base_addr = startup_addr + 64'h100000;  // = 0x80100000
result_size_bytes = 1024;                       // = 128 × 8-byte words
```

`program_simple.hex` contains **only CSR reads and `vx_wstop`** — it writes nothing to memory. The scoreboard compares 128 addresses at `0x80100000`, none of which the DUT ever touched → 128 warnings, 0 real comparisons.

The fix requires either a program that stores results to `result_base_addr`, or changing the comparison strategy. The `vortex_functional_mem_vseq.sv` already has the right program reference (`program_with_store.hex`) but it's not being used here.

---

### HIGH — Hidden Real Failures Masked as Passing

**Issue 3: Status Monitor contradicts the test result**

```
STATUS_MON:  Execution Started: YES    Execution Completed: NO
             Busy Transitions: 1       Idle Transitions: 0
             Total Cycles: 0           Total Instructions: 0
```

But the test reports PASS. The contradiction:
- `detect_state_transitions()` in `status_monitor.sv` never saw `ebreak_detected` rising edge
- BUT `run_final_comparison()` was called (SimX Ran=YES), meaning the scoreboard DID receive a transaction with `ebreak_detected=1`

The scoreboard receives status transactions via `ap.write()`. Looking at `monitor_status()`:
```systemverilog
// status_monitor.sv line ~95 (monitor_status task):
if (vif.monitor_cb.ebreak_detected && !prev_ebreak) continue;  // SKIPS it!
```
`monitor_status()` explicitly skips the ebreak cycle. So the **only** path to the scoreboard is through `detect_state_transitions()`, which sets `execution_completed=1`. But the flag shows 0. This is a **UVM phase race**: `detect_state_transitions()` fires `ap.write()` but the objection drops before the `execution_completed=1` assignment completes, then `check_phase()` reads the stale value.

**Issue 4: NUM_CORES redefinition in SimX build — wrong architecture**

Line 17 of the output:
```
-DNUM_CLUSTERS=1 -DNUM_CORES=2 -DNUM_WARPS=4 -DNUM_THREADS=4 -DXLEN_32   ← from Makefile ARCH_CONFIGS
-DNUM_CLUSTERS=1 -DNUM_CORES=1 -DNUM_WARPS=4 -DNUM_THREADS=4             ← from EXTRA_CXXFLAGS (script)
<command-line>: warning: "NUM_CORES" redefined   (line 20)
```

In `ref_model/Makefile`:
```makefile
ARCH_CONFIGS  = -DNUM_CLUSTERS=1
ARCH_CONFIGS += -DNUM_CORES=2        # ← HARDCODED 2
ARCH_CONFIGS += -DNUM_WARPS=4
ARCH_CONFIGS += -DNUM_THREADS=4
ARCH_CONFIGS += -DXLEN_32
```

The script passes `EXTRA_CXXFLAGS="-DNUM_CLUSTERS=1 -DNUM_CORES=1 ..."`, causing `NUM_CORES` to be defined twice. Second definition wins (NUM_CORES=1 matches RTL), but the Makefile has a hardcoded `NUM_CORES=2` that causes a warning and indicates the Makefile is not coordinated with the script.

**Issue 5: AXI Driver ID pool is broken — Available IDs: 0/0**

Line 1982: `Available IDs: 0 / 0`

In `axi_driver.sv`:
```systemverilog
max_ids = (1 << cfg.AXI_ID_WIDTH);
id_pool = new[max_ids];
```

From `vortex_config.sv`:
```systemverilog
parameter VX_MEM_TAG_WIDTH = 50;  // comment: "will be parameterized later"
...
AXI_ID_WIDTH = VX_MEM_TAG_WIDTH;  // = 50 ← causes (1 << 50) = 1 quadrillion IDs
```

`1 << 50` overflows in a 32-bit integer → `max_ids = 0`. The driver silently creates a zero-size pool. This doesn't crash because the DUT (not the TB) is the AXI master, so the driver never allocates IDs. But it's a latent bug that will crash any test where the UVM agent drives AXI transactions.

**Issue 6: Host Monitor — 0 Kernel Completions despite 1 launch detected**

```
HOST_MON:   Kernel Launches: 1    Kernel Completions: 0
HOST_DRV:   Programs Loaded: 0    DCR Writes: 0    Kernels Launched: 0
```

The Host Driver is completely idle (all zeros) because TB_TOP handles DCR initialization directly. The Host Monitor detected the kernel launch (busy rising edge) but missed completion. In `host_monitor.sv::monitor_kernel_execution()`:

```systemverilog
if (kernel_running && (!status_vif.monitor_cb.busy || status_vif.monitor_cb.ebreak_detected))
```

This check happens every clock. Since busy goes from 1→0 in only ~3 cycles (the program is extremely short), the monitor likely missed the falling edge due to not being scheduled in time during UVM startup.

---

### MEDIUM — Noisy Output and Hidden Issues

**Issue 7: 400+ lines of RTL RESET trace spam — hides real debug info**

Lines 998–1966: `*** VX_schedule[0] RESET: startup_addr=0x80000000 ***` repeated ~400 times (once per clock cycle during the 400-cycle reset period). This is caused by:
- `+define+DEBUG_LEVEL=3` is NOT passed to RTL `vlog` (the script only passes `FPU_FPNEW`, `NUM_*`, etc.)
- But `VX_config.vh` defines `DEBUG_LEVEL 3` as default, which is included during RTL compile
- Without `+define+NDEBUG`, all TRACE(3, ...) macros fire

In `run





---
Powered by [Claude Exporter](https://www.ai-chat-exporter.net)