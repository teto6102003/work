# Infinite errors in simx_model UVM integration

**Created:** 4/7/2026 13:10:42  
**Updated:** 4/7/2026 13:13:38  
**Exported:** 5/13/2026 16:26:19  
**Link:** [https://claude.ai/chat/892ad779-766e-41e5-89ce-287e535151ca](https://claude.ai/chat/892ad779-766e-41e5-89ce-287e535151ca)  

## Prompt:
4/7/2026, 1:12:12 PM

> File: 

inside that project can you access the other chats ?? i need from you to study the previous chats in that project deeply .  i did all changes and steps in the last chat to fully integrate simx_model.so inside uvm .
i made your fixes all of them that in "Debugging simx_model integration in UVM chat" and ues the program that you generate in Fix 5 , but the output is infinite of errors may be RTL enter inside loop . 
what's the problem and how i can fix it , and if the problem related by the  RTL or SIMX or any specific part , tell me
the part of infinite repeated result :



## Response:
4/7/2026, 1:13:38 PM



````plaintext
Thought process: حدّد السبب الجذري: تحميل البرنامج متأخر جداً.

The user is getting infinite errors from the RTL after integrating simx_model.so. The errors are:
1. "misaligned memory access, wid=x, PC=0xx, addr=0xx, wsize=x!" - X-state propagation
2. "runtime error: push and pop in same cycle not supported!" - IPDOM stack corruption
3. "runtime error: reading an empty stack!" - Stack underflow
4. "invalid request mask" - X-states in request masks
5. "invalid writeback register" - X-states in register file


The `x` values in the error messages indicate X-state propagation from the start of simulation, likely because the program memory wasn't loaded before reset was released, the DCR startup address wasn't set up, or memory responses are returning X values. At 11505 ns (around 1150 cycles in), this is well past the 400-cycle reset period, so the X-states are persisting through execution.

Looking more closely, the fact that all errors show `wid=x, PC=0xx` suggests this is happening right at the beginning—the PC register itself was never initialized properly, which means `base_dcrs.startup_addr` is X and the DUT has no valid starting address to fetch from.

The real problem is the timing: DCR writes happen during reset cycles 5-14, then reset releases around cycle 400. But the smoke test loads the program in `run_phase` after waiting for reset to complete, so by the time the DUT starts fetching from 0x80000000, memory is still empty and returns zeros or X. The DUT then executes garbage instructions and explodes into X-states. The solution is to load the program before reset deasserts, not after. The mem_model is just a software associative array, so loading data into it at any point during simulation is fine—what matters is that the data exists before the DUT actually tries to read from those addresses. The MEM responder in tb_top waits for reset to deassert, then starts servicing memory requests, so as long as `load_program()` completes before the DUT makes its first memory access, the timing should work out.

But looking at the actual sequence: reset holds until around cycle 400, the DUT immediately starts fetching after that, yet the UVM test's `run_phase` doesn't kick in until much later (around cycle 19500 based on the tb_top comments). That's a huge gap where the DUT is fetching from an empty memory.

The solution is to load the program in the `tb_top` initial block before reset releases, pulling the file path from the `+PROGRAM` plusarg, just like how the DCR writes happen early. The old backup had this in the memory model's initial block, but the current version removed it expecting the test to handle it—which doesn't work with the timing.

I need to add the hex file loading right after memory creation, using the base address of `0x80000000` where the memory model expects programs to be loaded. Though I should double-check if the startup address might differ from this assumption. Still writing the startup address logic...

The hex files have `@00000000` as their base, so when I add the startup address offset of `0x80000000`, the addresses line up correctly at `0x80000000`. The test's `load_program()` can then verify or skip loading since the memory is already populated from the initial block.

The real issue is timing: the program needs to load before reset deasserts at cycle 400, but the UVM run_phase starts much later during elaboration. Moving the load into the tb_top initial block solves this. In AXI wrapper mode, the responder reads directly from the memory model when handling AXI transactions, so having the program pre-loaded ensures data is available immediately.
````

## Root Cause Analysis

The errors (`wid=x, PC=0xx, addr=0xx`) all show **X-state propagation** — the DUT is executing garbage. This is a **timing issue**, not an RTL or SimX bug.

**The sequence of events:**
1. `t=0`: Reset asserted, TB_TOP DCR writes happen (cycles 5-14)  
2. `t≈4000ns` (cycle 400): **Reset releases → DUT starts fetching from 0x80000000**
3. `t≈19500ns` (cycle ~1950): UVM elaboration finishes, `run_phase` starts, `load_program()` called
4. `t≈11500ns` (cycle 1150): **RTL explodes** — fetching from empty memory → zeros/X → pipeline corruption

**Program loads ~1000 cycles AFTER the DUT starts executing. Memory is empty, DUT fetches zeros, X-states cascade through the pipeline.**

---

## Fix: Load Program in TB_TOP at Time-0

In `tb/vortex_tb_top.sv`, find the memory model initial block and add program loading **before reset deasserts**:

**Find this section:**
```systemverilog
initial begin
    memory = mem_model::type_id::create("memory");
    $display("[TB_TOP @ %0t] Memory model created", $time);

    uvm_config_db#(mem_model)::set(null, "*",             "mem_model", memory);
    // ... other set calls ...

    begin
        mem_model test_get;
        #1;
        if (uvm_config_db#(mem_model)::get(null, "*", "mem_model", test_get))
            $display("[TB_TOP @ %0t] ✓ mem_model verified in config_db", $time);
    end
end
```

**Replace with:**
```systemverilog
initial begin
    memory = mem_model::type_id::create("memory");
    $display("[TB_TOP @ %0t] Memory model created", $time);

    uvm_config_db#(mem_model)::set(null, "*",             "mem_model", memory);
    uvm_config_db#(mem_model)::set(null, "uvm_test_top*", "mem_model", memory);
    uvm_config_db#(mem_model)::set(uvm_root::get(), "*",  "mem_model", memory);

    // =========================================================
    // FIX: Load program at TIME-0, BEFORE reset deasserts.
    // The DUT starts fetching at cycle 400 (reset release).
    // UVM run_phase only starts at ~cycle 1950 (after elaboration).
    // Without this, memory is empty when DUT first fetches → X-state explosion.
    // =========================================================
    begin
        string prog_file = "";
        bit [63:0] load_addr = 64'h80000000;
        bit [63:0] tmp_addr;
        int n_bytes;

        if ($value$plusargs("PROGRAM=%s", prog_file) ||
            $value$plusargs("HEX=%s",     prog_file)) begin

            // Use same startup addr as DCR (strips 0x prefix already handled)
            if ($value$plusargs("STARTUP_ADDR=%h", tmp_addr))
                load_addr = tmp_addr;
            else
                load_addr = 64'h80000000;

            n_bytes = memory.load_hex_file(prog_file, load_addr);
            if (n_bytes > 0)
                $display("[TB_TOP @ %0t] ✓ Program pre-loaded: %s (%0d bytes @ 0x%h)",
                         $time, prog_file, n_bytes, load_addr);
            else
                $error("[TB_TOP @ %0t] ✗ Failed to pre-load program: %s", $time, prog_file);
        end else begin
            $display("[TB_TOP @ %0t] ⚠ No +PROGRAM specified — memory empty at reset!", $time);
        end
    end

    begin
        mem_model test_get;
        #1;
        if (uvm_config_db#(mem_model)::get(null, "*", "mem_model", test_get))
            $display("[TB_TOP @ %0t] ✓ mem_model verified in config_db", $time);
        else
            $error("[TB_TOP @ %0t] ✗ mem_model NOT in config_db!", $time);
    end
end
```

---

## Secondary Fix: Smoke Test load_program() Becomes Verification-Only

In `uvm_tests/vortex_smoke_test.sv`, the `load_program()` task in `run_phase` now only verifies (data already there), so change its failure from `uvm_fatal` to `uvm_warning` for the "no PROGRAM" case since tb_top handles it:

**In `load_program()` task, find:**
```systemverilog
if (!$value$plusargs("PROGRAM=%s", hex_file)) begin
    `uvm_fatal(get_type_name(), {
        "No +PROGRAM specified!\n",
        ...
    })
end
```

**Change to:**
```systemverilog
if (!$value$plusargs("PROGRAM=%s", hex_file) &&
    !$value$plusargs("HEX=%s", hex_file)) begin
    `uvm_info(get_type_name(),
        "No +PROGRAM — relying on TB_TOP pre-load (this is fine)", UVM_MEDIUM)
    return;  // TB_TOP already loaded it
end
```

---

## Why This Fixes the Infinite Errors

| Issue | Root Cause | After Fix |
|---|---|---|
| `wid=x, PC=0xx` | DUT fetches zeros from empty memory | Memory populated at t=0 |
| `misaligned access wid=x` | X from PC propagates to LSU address | Valid instructions decoded |
| `push/pop same cycle` | SIMT stack corrupted by X-state warp mask | Clean warp state from valid fetch |
| `reading empty stack` | Spurious SPLIT/JOIN from X-state decode | No spurious control flow |

The program must be in memory **at or before** cycle 400 (when reset releases). Loading it in `run_phase` at cycle ~1950 is always too late.





---
Powered by [Claude Exporter](https://www.ai-chat-exporter.net)