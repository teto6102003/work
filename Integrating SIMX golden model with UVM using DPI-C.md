# Integrating SIMX golden model with UVM using DPI-C

**Created:** 2/28/2026 14:57:16  
**Updated:** 2/28/2026 18:10:36  
**Exported:** 5/13/2026 16:15:10  
**Link:** [https://claude.ai/chat/42ba0438-08df-44b8-a9c2-f96e2c1d88bd](https://claude.ai/chat/42ba0438-08df-44b8-a9c2-f96e2c1d88bd)  

## Prompt:
2/28/2026, 3:23:22 PM

> File: Makefile

> File: simx_dpi.cpp

> File: simx_pkg.sv

> File: test_top.sv

> File: vortex_config.sv

act as expert in Digital ic design and verification specially in UVM and GPGPUs like vortex from this repo :"https://github.com/vortexgpgpu/vortex" study it deeply and all info. you can find in reserch papers about that.
we are a team working to build a full UVM for vortex , 
we work on linux ubuntu 2022 and questasim 2021 for linux ,
and I'm resposable for the Golden model part ,
we choose the 'simx' as a golden model which is inside vortex repo ,
but it (simx) was written by C++ and rtl for vortex is systemverilog and uvm is systemverilog too, 
i searched in this part and find thing called DPI-c used in these cases ,which is a MakeFile compile all Simx files + third-party files (in C++ too) and DPI-c file (C++) + DPI-c file which is simx_dpi.cpp .
NOW , i need from you to answer in details and deeply :
* tell me, Is their other Golden models in the repo or at any other place for Vortex rather than SIMX , and tell me Why simx is the best ?
* is it normal to write the Golden models as an behavorail models and using C++ insteade of systemverilog or other language ??
* tell me all possible ways to integrate or import or solve the probelm of SIMX written in C++ and UVM in SV , and also the pros and cons for each way even the old ways ??
* why in the end we go to DPI-c to solve this problem ??
* what is DPI-c ?? is it a translator from C++ to SV or a bridge or what ??
* How will help to import golden model inside UVM ?? 
* (with simple example for write and read on simx with uvm) what it actually do ?? what's the importance for Makefile and simx_dpi.cpp and each file in my work



## Response:
2/28/2026, 5:26:33 PM



````plaintext
Thought process: Examining uploaded files to comprehend project scope.

The user wants a deep, detailed explanation about several topics related to Vortex GPGPU, UVM, golden models, and DPI-C. Let me provide a comprehensive answer based on my knowledge.

Let me first look at the uploaded files to understand what they have.
````

````plaintext
View: Understanding the DPI-C implementation
````

````plaintext
Thought process: Marshaled comprehensive technical framework for UVM-SIMX integration strategy.

Now let me look at the other files.
````

````plaintext
View: Reading simx_pkg.sv
````

````plaintext
View: Reading Makefile
````

````plaintext
Thought process: Synthesized comprehensive technical framework for UVM-SIMX integration strategy.

Now I have a thorough understanding of all the files. Let me write a comprehensive, deep answer covering all the questions asked.
````

Now I have a complete picture of your work. Let me give you the deep, expert answer you need.

---

# Complete Deep Guide: SimX as Golden Model + DPI-C in Vortex UVM

---

## 1. Are There Other Golden Models for Vortex? Why is SimX the Best?

### Other potential golden models in/around the repo:

**A) Spike (RISC-V ISA Simulator)**
Found at: `third_party/` or used externally. Spike is the official RISC-V Foundation ISA reference simulator. It simulates RISC-V instructions architecturally-correctly. However, it knows nothing about Vortex's GPU-specific extensions: warps, thread groups, barriers, texture units, shared memory, or vector execution. It would only validate scalar RISC-V behavior.

**B) LLVM/Clang Compiler Emulation**
The Vortex compiler toolchain (`kernel/`) can emit and trace instructions. Some teams use the compiler's built-in emulator mode as a lightweight golden check. But this is not an architectural model — it doesn't simulate memory hierarchy, divergence, or multi-warp scheduling.

**C) OpenCL/POCL Reference**
Vortex targets OpenCL. POCL (Portable Computing Language) can run the same OpenCL kernels on CPU using reference implementations. But it doesn't model Vortex's microarchitecture at all — it's pure algorithm-level comparison.

**D) Verilator model (RTL itself at higher speed)**
Some teams run a subset of the RTL through Verilator and treat the Verilator result as a "fast reference." This is dangerous: you'd be comparing RTL to itself, defeating the whole purpose of a golden model.

**E) SimX — the actual functional simulator inside the repo**
Located at `sim/simx/`. Written in C++ by the Vortex team themselves. This IS the architectural golden model Vortex was designed with.

### Why SimX is THE correct and best choice:

| Criterion | SimX | Spike | POCL |
|---|---|---|---|
| Models Vortex warps/threads | ✅ Exactly | ❌ No | ❌ No |
| Models Vortex ISA extensions | ✅ Exactly | ❌ No | ❌ No |
| Models memory hierarchy | ✅ Cache + DRAM | ❌ Flat | ❌ OS-level |
| Same team wrote it | ✅ Yes | ❌ No | ❌ No |
| DCR register model | ✅ Yes | ❌ No | ❌ No |
| Shared memory / barriers | ✅ Yes | ❌ No | ❌ Approximate |
| Used to validate RTL in CI | ✅ Yes | ❌ No | ❌ No |

SimX simulates instruction-by-instruction execution of every warp, every thread, across all cores, with the exact same memory map (`0x80000000` startup address), DCRs, and boot sequence that the RTL uses. This is the definition of a proper golden model for RTL verification.

---

## 2. Is It Normal to Write Golden Models in C++ Instead of SystemVerilog?

**Yes, absolutely normal and in fact the industry standard approach.** Here is the full reasoning:

**Why C++ (or C) is preferred for golden models:**

1. **Speed of execution**: A behavioral C++ model runs 100x–1000x faster than an RTL simulation. The golden model needs to predict results before or during simulation. If it were written in SV, it would be as slow as the DUT itself — useless.

2. **Abstraction level**: Golden models are *behavioral*, not *structural*. They don't model clock cycles, pipeline stages, or timing. They model *what should happen* (what output should a certain input produce?), not *how it happens* in hardware. C++ is perfectly suited for this abstraction level.

3. **Algorithm-friendly**: Complex GPU programs involve memory allocation, file I/O, floating point operations, exception handling. Implementing all this in synthesizable or even simulation SV is painful and error-prone.

4. **Real industry examples**:
   - ARM uses **Cycle Models** in C++ (Fast Models) as golden references for Cortex verification
   - Intel uses C++ ISS (Instruction Set Simulators) like SoftSIM for Core validation
   - NVIDIA uses C++ functional simulators to validate GPU shader behavior
   - RISC-V community uses Spike (C++) as the ISA golden reference

5. **SystemC is a middle ground**: SystemC (a C++ library from Accellera) lets you model hardware at transaction or cycle-accurate level in C++. Some teams use SystemC as a golden model. But for Vortex, SimX is already written and fully trusted — there's no reason to rewrite it in SystemC.

**The only time golden models are written in SV** is when the model is *very simple* (like a simple FIFO or a register file) and the team wants everything in one language. For anything as complex as a GPU ISA, C++ wins.

---

## 3. All Possible Ways to Integrate C++ (SimX) with SV/UVM

Here is every known integration approach, with deep pros/cons:

---

### **Method 1: DPI-C (Direct Programming Interface – C)**
The IEEE 1800-2012 standard mechanism to call C/C++ from SV and vice versa.

**Pros:**
- IEEE standard, supported by all major simulators (QuestaSim, VCS, Xcelium)
- Zero serialization overhead — direct in-process function calls
- C++ objects can live in heap, SV just holds a pointer or calls functions
- Full access to C++ STL, classes, data structures
- Bidirectional: SV can call C (import), C can call SV (export)
- Works natively in QuestaSim 2021 — no plugins needed

**Cons:**
- Debugging cross-language issues (C++ exceptions vs SV errors) can be tricky
- Data type mapping must be explicit (svOpenArrayHandle, chandle, etc.)
- Shared library (`.so`) must be compiled with `-fPIC`
- Thread safety: only one simulation thread can be in DPI at a time (unless you use `context` tasks carefully)

---

### **Method 2: PLI (Programming Language Interface) — Old**
The original Verilog extension mechanism from the 1990s (VPI/PLI). Uses `vpi_get_value()`, `vpi_put_value()` to access simulator objects.

**Pros:**
- Works with very old simulators
- Can introspect simulation hierarchy (read any net or register by name)

**Cons:**
- Extremely verbose and painful API
- No C++ class-level integration, only C structs
- Deprecated in favor of DPI-C
- Much slower (every access goes through simulator kernel)
- Nobody writes new PLI code anymore

---

### **Method 3: SystemC + TLM-2.0**
Write the golden model in SystemC, then co-simulate with SV using a SystemC-SV bridge (supported by QuestaSim via the `sccom`/`sclink` flow).

**Pros:**
- Clean object-oriented model
- TLM-2.0 gives transaction-level communication standard
- Good for both cycle-accurate and behavioral models
- QuestaSim has native SystemC integration

**Cons:**
- SimX is already in plain C++. You'd need to wrap it in SystemC sc_module — extra work
- SystemC co-simulation adds overhead (delta cycles, TLM blocking calls)
- More complex build system (need Accellera SystemC library)
- Overkill for what you need

---

### **Method 4: Shared Memory / Pipes / Sockets (IPC)**
Run SimX as a *separate process*, and exchange data with the SV simulator via OS-level IPC: shared memory segments, Unix pipes, or TCP/UDP sockets.

**Pros:**
- Complete language isolation — SimX doesn't need to know about SV at all
- Can restart SimX independently
- Good for multi-machine setups

**Cons:**
- Latency: every comparison requires IPC round-trip (slow for cycle-by-cycle comparison)
- Synchronization complexity: how do you know when SimX result is ready?
- Serialization: you must marshal/unmarshal all data structures
- Error handling across process boundaries is complex
- Overkill for same-machine simulation

---

### **Method 5: File-Based Exchange**
Run SimX offline, dump results to file (`.txt`, `.hex`, `.bin`). Then read the file in SV during simulation and compare.

**Pros:**
- Dead simple to implement
- No integration complexity
- Works with any tool

**Cons:**
- Post-mortem only — cannot do cycle-by-cycle comparison
- File I/O is slow for large memories
- No runtime feedback — you can't stop RTL simulation early when divergence is detected
- Error localization is harder (you only know the final state differs)

---

### **Method 6: HVL/HDL Co-simulation via e/Specman or URM**
Old Cadence approach using `e` language (Specman Elite) with C extensions via `HDL import`. Essentially a predecessor to UVM's DPI-C approach.

**Pros:**
- Was industry standard in 2000s (pre-UVM)

**Cons:**
- `e` is proprietary Cadence language, dying
- No open-source support
- Completely replaced by UVM + DPI-C

---

### **Method 7: MATLAB/Simulink with HDL Verifier**
Generate a SystemVerilog testbench from MATLAB model and compare with RTL.

**Pros:**
- Good for DSP/signal processing golden models
- Automatic comparison infrastructure

**Cons:**
- Licensing costs are extreme
- Not at all suitable for a GPGPU ISA model
- MATLAB cannot model RISC-V execution semantics properly

---

## 4. Why Do We End Up Choosing DPI-C?

The answer is a process of elimination combined with specific requirements:

1. **SimX already exists** in C++. We don't rewrite it.
2. **We need tight integration** with UVM — the scoreboard needs SimX results *during* simulation, not after it from a file.
3. **We need speed** — IPC methods add unacceptable latency.
4. **We need standard tool support** — QuestaSim 2021 supports DPI-C perfectly.
5. **We need bidirectional control** — SV must be able to initialize SimX, write memory into it, run it, and then read back expected values.
6. **SystemC is unnecessary overhead** — SimX is plain C++, not a SystemC model. Wrapping it would add complexity without benefit.

DPI-C is the only mechanism that gives you:
- **Direct function call from SV to C++** (zero overhead, same process)
- **In-process memory access** (no serialization)
- **Standard IEEE support** across QuestaSim/VCS/Xcelium
- **Minimal wrapper code** — you only need `extern "C"` and `svdpi.h`

---

## 5. What Exactly IS DPI-C?

DPI-C is **not** a translator and **not** a code generator. It is a **direct binary interface bridge** — a standardized calling convention that allows the SystemVerilog simulator to call compiled C/C++ functions that live in a shared library (`.so` on Linux), as if they were native SV functions.

Here's what happens at the hardware/OS level:

```
 QuestaSim (vsim process)
 ┌───────────────────────────────────────────┐
 │  SV Simulator Kernel                      │
 │  ┌──────────────────────────────────────┐ │
 │  │  UVM Testbench (SV bytecode/IR)      │ │
 │  │                                      │ │
 │  │  simx_init(2, 4, 4)  ──────────────────┼──► DPI-C call dispatch
 │  └──────────────────────────────────────┘ │        │
 │                                            │        ▼
 │  DPI-C Runtime Layer                       │  dlopen("simx_model.so")
 │  ┌──────────────────────────────────────┐ │  dlsym("simx_init")
 │  │  Argument marshaling:               │ │        │
 │  │  SV int → C int                     │ │        ▼
 │  │  SV string → char*                  │ │  Jump to C++ function
 │  │  SV byte[] → svOpenArrayHandle      │ │
 │  └──────────────────────────────────────┘ │
 │                                            │
 │  Loaded .so: simx_model.so                 │
 │  ┌──────────────────────────────────────┐ │
 │  │  simx_init() ← actual C++ function  │ │
 │  │  SimX Processor object               │ │
 │  │  RAM object                          │ │
 │  │  Arch object                         │ │
 │  └──────────────────────────────────────┘ │
 └───────────────────────────────────────────┘
```

Key concepts:
- `extern "C"` disables C++ name mangling so the simulator can find functions by their plain names
- `svdpi.h` provides the data type definitions for the interface (`svOpenArrayHandle`, `svBitVecVal`, `chandle`, etc.)
- `-sv_lib simx_model` tells vsim to `dlopen` the `.so` at startup
- The SV `import "DPI-C"` declaration tells the SV compiler "this function lives in a .so — trust me"
- Data type mapping is automatic for primitives (`int`, `longint`, `byte`), and manual (via handles) for arrays/structs

**DPI-C is NOT:**
- A compiler
- A code generator
- A translator
- A language wrapper like SWIG or ctypes
- A process boundary (everything is same-process, same memory space)

---

## 6. How DPI-C Enables Using SimX Inside UVM

In UVM, the golden model is typically a **passive component** that sits alongside the scoreboard. Here's the full architectural picture of how it all connects:

```
 UVM Environment
 ┌─────────────────────────────────────────────────────────────┐
 │                                                             │
 │   Agent                                                     │
 │   ┌──────────┐   Analysis Port                             │
 │   │ Monitor  │──────────────────────┐                      │
 │   └──────────┘                      │                      │
 │                                     ▼                      │
 │                              ┌─────────────┐               │
 │                              │  Scoreboard │               │
 │                              │             │               │
 │                              │  actual  ◄──┘               │
 │                              │  vs                         │
 │                              │  expected ◄──────────────┐  │
 │                              └─────────────┘             │  │
 │                                                          │  │
 │   ┌─────────────────────────────────────────────────┐   │  │
 │   │  simx_golden_model (UVM Component)              │   │  │
 │   │                                                 │   │  │
 │   │  run_phase():                                   │   │  │
 │   │    simx_init(cores, warps, threads) ──► C++    │   │  │
 │   │    simx_write_mem(addr, size, data) ──► C++    │   │  │
 │   │    simx_dcr_write(addr, val)        ──► C++    │   │  │
 │   │    simx_run()                       ──► C++    │   │  │
 │   │    simx_read_mem(addr, size, data)  ──► C++    │   │  │
 │   │    ap.write(expected_txn)           ───────────┘   │  │
 │   └─────────────────────────────────────────────────┘  │  │
 └────────────────────────────────────────────────────────┘  │
```

The `simx_golden_model` component calls the DPI-C functions (which call into the compiled SimX library), runs the same program as the RTL DUT, reads back what the memory should contain after execution, packages that into a UVM transaction, and sends it to the scoreboard. The scoreboard compares it against what the RTL monitor captured.

---

## 7. Deep Walkthrough: Write/Read Example with UVM + SimX + DPI-C

Let me walk through a complete write-then-read scenario showing exactly what every layer does.

### Scenario: Vector Addition Kernel

You have a simple kernel that adds two arrays (A + B → C). You want to verify that after RTL executes this kernel, memory at the result address matches what SimX predicted.

---

### Step 1 — The SV Package declares the DPI bridge (`simx_pkg.sv`)

```systemverilog
package simx_pkg;
    import uvm_pkg::*;
    `include "uvm_macros.svh"

    // Tell SV: "these functions exist in an external .so"
    // SV side sees them as normal SV functions
    import "DPI-C" context function int simx_init(
        input int num_cores,
        input int num_warps,
        input int num_threads
    );

    import "DPI-C" context function int simx_load_bin(
        input string filepath,
        input longint load_addr
    );

    import "DPI-C" context function void simx_write_mem(
        input longint  addr,
        input int      size,
        input byte     data[]
    );

    import "DPI-C" context function void simx_read_mem(
        input longint  addr,
        input int      size,
        inout byte     data[]
    );

    import "DPI-C" context function void simx_dcr_write(
        input int addr,
        input int value
    );

    import "DPI-C" context function int  simx_run();
    import "DPI-C" context function void simx_cleanup();

endpackage
```

**What this does:** The SV compiler sees these as function prototypes. At link time (when vsim loads the `.so`), it resolves them. When SV code calls `simx_init(2,4,4)`, the simulator's DPI runtime marshals the arguments and jumps directly to the C++ function.

---

### Step 2 — The DPI wrapper in C++ (`simx_dpi.cpp`)

Looking at your actual file, the key write/read functions are:

```cpp
// Called from SV: simx_write_mem(addr, size, data[])
void simx_write_mem(uint64_t addr, int size, const svOpenArrayHandle data) {
    // svGetArrayPtr: gets a raw C pointer to the SV byte array's memory
    uint8_t* src = (uint8_t*)svGetArrayPtr(data);
    
    // Calls SimX's RAM object directly — same object SimX Processor uses
    g_ram->write(src, addr, size);
}

// Called from SV: simx_read_mem(addr, size, data[])
void simx_read_mem(uint64_t addr, int size, const svOpenArrayHandle data) {
    uint8_t* dest = (uint8_t*)svGetArrayPtr(data);
    
    // Reads from SimX's RAM — populated after simx_run()
    g_ram->read(dest, addr, size);
}
```

**What `svOpenArrayHandle` is:** A DPI-C opaque handle to a SV dynamic or open array. `svGetArrayPtr()` returns a raw C pointer to the underlying data. This is a zero-copy mechanism — no data is duplicated between SV and C++ memory spaces. They literally share the same bytes.

---

### Step 3 — The UVM Golden Model Component (`simx_pkg.sv` golden_model class)

Here's an extended version showing a real vector addition comparison:

```systemverilog
class simx_golden_model extends uvm_component;
    `uvm_component_utils(simx_golden_model)

    vortex_config cfg;
    uvm_analysis_port #(vortex_result_txn) result_ap;

    function new(string name, uvm_component parent);
        super.new(name, parent);
        result_ap = new("result_ap", this);
    endfunction

    function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        if (!uvm_config_db#(vortex_config)::get(this, "", "cfg", cfg))
            `uvm_fatal("SIMX", "Config not found!")
    endfunction

    task run_phase(uvm_phase phase);
        int        ret;
        byte       result_data[];
        vortex_result_txn expected_txn;

        // ── PHASE 1: Initialize SimX ─────────────────────────────────────────
        // This calls simx_init() in C++, which creates:
        //   g_arch = new Arch(num_threads, num_warps, num_cores)
        //   g_ram  = new RAM(4GB, 4KB pages)
        //   g_processor = new Processor(*g_arch)
        //   g_processor->attach_ram(g_ram)
        ret = simx_init(cfg.num_cores, cfg.num_warps, cfg.num_threads);
        if (ret != 0) `uvm_fatal("SIMX", "simx_init failed")
        `uvm_info("SIMX", "SimX initialized", UVM_LOW)

        // ── PHASE 2: Load Kernel Binary ──────────────────────────────────────
        // Writes the RISC-V binary (ELF flat binary) into g_ram at 0x80000000
        // This is the same .bin that the RTL loads from flash/ROM
        ret = simx_load_bin(cfg.kernel_path, 64'h80000000);
        if (ret != 0) `uvm_fatal("SIMX", "simx_load_bin failed")

        // ── PHASE 3: Write Input Data ────────────────────────────────────────
        // Suppose our kernel reads array A from 0x90000000
        // We push the same stimulus data into SimX's RAM
        // that we wrote into the RTL DUT's memory
        begin
            byte input_A[];
            input_A = new[cfg.data_size];
            // Copy stimulus from config object
            foreach(input_A[i]) input_A[i] = cfg.input_A[i];

            // DPI-C call: SV byte[] → C uint8_t* → g_ram->write()
            simx_write_mem(64'h90000000, cfg.data_size, input_A);
        end

        begin
            byte input_B[];
            input_B = new[cfg.data_size];
            foreach(input_B[i]) input_B[i] = cfg.input_B[i];
            simx_write_mem(64'h90010000, cfg.data_size, input_B);
        end

        // ── PHASE 4: Configure DCRs ──────────────────────────────────────────
        // DCRs are Device Configuration Registers — they tell the processor
        // where to start, how many warps to use, etc.
        // Must match exactly what the testbench writes to the RTL DUT
        simx_dcr_write(32'h0004, 32'h80000000); // STARTUP_ADDR0
        simx_dcr_write(32'h0006, 32'h00000001); // NUM_THREADS
        simx_dcr_write(32'h0008, 32'h00000001); // NUM_WARPS

        // ── PHASE 5: Run SimX to Completion ─────────────────────────────────
        // This calls g_processor->run() which executes every RISC-V instruction
        // of every warp of every thread until the program exits.
        // SimX is FAST — a 1000-instruction kernel runs in microseconds here.
        // When it returns, g_ram contains the final state of all memory,
        // including the output array C at 0x90020000.
        ret = simx_run();
        `uvm_info("SIMX", $sformatf("SimX run complete, exitcode=%0d", ret), UVM_LOW)

        // ── PHASE 6: Read Back Expected Results ─────────────────────────────
        // SimX has finished. g_ram[0x90020000] now contains expected output.
        // We read it back into a SV byte array.
        result_data = new[cfg.data_size];
        simx_read_mem(64'h90020000, cfg.data_size, result_data);

        // ── PHASE 7: Package and Send to Scoreboard ──────────────────────────
        expected_txn = vortex_result_txn::type_id::create("expected_txn");
        expected_txn.data      = result_data;
        expected_txn.base_addr = 64'h90020000;
        expected_txn.is_golden = 1; // Tag as golden model result

        // The scoreboard receives this via analysis port
        // and compares with what the RTL monitor captured
        result_ap.write(expected_txn);

    endtask

    function void report_phase(uvm_phase phase);
        // CRITICAL: Free all C++ objects (Processor, RAM, Arch)
        // Without this, you get memory leaks between test runs
        simx_cleanup();
    endfunction
endclass
```

---

### Step 4 — The Scoreboard receives from both sides

```systemverilog
class vortex_scoreboard extends uvm_scoreboard;
    `uvm_component_utils(vortex_scoreboard)

    // Receives from RTL monitor (actual results)
    uvm_analysis_imp #(vortex_result_txn, vortex_scoreboard) rtl_imp;
    // Receives from SimX golden model (expected results)
    uvm_analysis_imp #(vortex_result_txn, vortex_scoreboard) golden_imp;

    vortex_result_txn golden_result;
    vortex_result_txn rtl_result;

    function void write_golden(vortex_result_txn txn);
        golden_result = txn;
        compare_if_ready();
    endfunction

    function void write_rtl(vortex_result_txn txn);
        rtl_result = txn;
        compare_if_ready();
    endfunction

    function void compare_if_ready();
        if (golden_result == null || rtl_result == null) return;
        
        foreach (golden_result.data[i]) begin
            if (golden_result.data[i] !== rtl_result.data[i]) begin
                `uvm_error("SB", $sformatf(
                    "MISMATCH at byte %0d: expected 0x%02x, got 0x%02x",
                    i, golden_result.data[i], rtl_result.data[i]))
            end
        end
        `uvm_info("SB", "Memory comparison PASSED", UVM_LOW)
    endfunction
endclass
```

---

## 8. What Every File in Your Work Does — Deep Explanation

### `simx_dpi.cpp` — The Bridge Layer

This is the most critical file. It does three things:

1. **Owns the SimX state**: `g_processor`, `g_ram`, `g_arch` are file-scope static globals. They persist between DPI-C calls because they live in the `.so`'s BSS/data segment, not on the SV call stack. This is how SimX remembers its state between `simx_init()`, `simx_write_mem()`, `simx_run()`, and `simx_read_mem()` — they all operate on the same global objects.

2. **Bridges data types**: The `svOpenArrayHandle` → `svGetArrayPtr()` pattern is how SV `byte[]` (dynamic array) becomes a raw `uint8_t*` in C++. This is zero-copy — the SV simulator and C++ code look at the same physical memory addresses.

3. **Provides `extern "C"` linkage**: Without this, the C++ compiler mangles function names (`simx_init` becomes `_Z10simx_initi` or similar). The `extern "C"` block forces plain C symbol names so the SV `import "DPI-C"` declarations can find them.

**Important detail in your file**: The `simx_init_exit_code_register()` function injects a bootstrap sequence at `0x80000000 - 16`. This is necessary because SimX's `Processor::run()` uses `x3` as the exit code register, but `x3` is not initialized by the RISC-V startup convention. The bootstrap sets `x3 = 0`, then jumps to the real entry point. This is elegant low-level RISC-V code:
- `addi x3, x0, 0` — zero the exit register
- `auipc x1, 0` — PC-relative: x1 = current PC
- `jalr x0, x1, 16` — jump to current PC + 16 = original startup address

### `simx_pkg.sv` — The SV-Side Interface

This package serves two purposes:
1. **DPI-C declarations** (`import "DPI-C"`) — tells the SV compiler the function signatures
2. **UVM component class** — `simx_golden_model` wraps the DPI calls in proper UVM lifecycle phases

**Note a discrepancy between your two files**: `simx_pkg.sv` declares `simx_init` with a `kernel_file` string parameter and no `simx_run()`, while `simx_dpi.cpp` has `simx_init(num_cores, num_warps, num_threads)` and a separate `simx_run()`. You will need to align these — either update the `simx_pkg.sv` DPI declarations to match what `simx_dpi.cpp` actually exports, or add a `kernel_file` parameter to the C++ side.

### `Makefile` — The Build Orchestrator

The Makefile does something non-obvious that's worth understanding deeply:

```makefile
$(CXX) $(CXXFLAGS) simx_dpi.cpp $(SIMX_OBJS) $(LDFLAGS) -o $(DPI_LIB)
```

This line:
1. Compiles `simx_dpi.cpp` to object code
2. Links it with `$(SIMX_OBJS)` — the pre-compiled SimX object files (`processor.o`, `core.o`, `warp.o`, etc.)
3. Links with `softfloat.a` — the Berkeley SoftFloat library (SimX uses this for IEEE 754 FP emulation)
4. Links with `libramulator` — the DRAM timing model (SimX models DRAM latency using Ramulator)
5. Produces `simx_model.so` — a shared library loadable by vsim

**Why `$(SIMX_OBJS)` instead of compiling SimX source directly?**
SimX has complex inter-file dependencies and its own build system. The Makefile assumes you've already run `make` in `$(VORTEX_HOME)/sim/simx` to produce the `.o` files. You link those `.o` files directly into your `.so`. This is the correct pattern — don't recompile SimX, just link its output.

**The `-Wl,-rpath` flag**: This embeds the Ramulator library search path into the `.so` itself, so vsim can find it at runtime without needing to set `LD_LIBRARY_PATH` manually (though the Makefile also sets that as a belt-and-suspenders measure).

**`-DNUM_CORES=2 -DNUM_WARPS=4 -DNUM_THREADS=4`**: These defines must **exactly match** your RTL configuration (`vortex_config.sv`). SimX reads these at compile time to configure the Arch object correctly. If they don't match, SimX will model a different processor than your RTL, and comparisons will fail even for correct RTL.

### `vortex_config.sv` — The Configuration Object

This is standard UVM config object pattern. It acts as a single source of truth: the same `num_cores`, `num_warps`, `num_threads` values that drive the RTL plusargs and the SimX initialization come from one place, preventing mismatches.

The `simx_enable` flag lets you disable golden model comparison without restructuring the testbench — useful for early-stage debugging when you just want the RTL to run.

### `test_top.sv` — The Top-Level Testbench

This wires together the UVM phases and instantiates the DUT + UVM components. The key line in any test_top using DPI-C is the module-level `import` or the fact that vsim is invoked with `-sv_lib simx_model`, which causes the simulator to load your `.so` before any SV code runs.

---

## Summary: The Complete Data Flow Diagram

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │                     QuestaSim Process Memory                        │
 │                                                                     │
 │  SV Simulation                         C++ (simx_model.so)         │
 │  ┌─────────────────────┐               ┌───────────────────────┐   │
 │  │ UVM Test            │               │ global state:         │   │
 │  │ build_phase()       │  simx_init()  │  g_arch               │   │
 │  │   creates golden ──────────────────►│  g_ram (4GB)          │   │
 │  │   model component   │◄──────────────│  g_processor          │   │
 │  │                     │  returns 0    │                       │   │
 │  │ run_phase()         │               │                       │   │
 │  │                     │  simx_load_   │                       │   │
 │  │   load kernel ──────────────────────►bin("kernel.bin",      │   │
 │  │                     │               │   0x80000000)         │   │
 │  │                     │               │  g_ram[0x80000000]=   │   │
 │  │                     │               │  [kernel bytes]       │   │
 │  │                     │               │                       │   │
 │  │   write input ──────────────────────►simx_write_mem(        │   │
 │  │   data              │               │  0x90000000,          │   │
 │  │   [0xAB,0xCD,...]   │  zero-copy    │  size, ptr)           │   │
 │  │                     │               │  g_ram[0x90000000]=   │   │
 │  │                     │               │  [0xAB,0xCD,...]      │   │
 │  │                     │               │                       │   │
 │  │   run SimX ─────────────────────────►simx_run()             │   │
 │  │                     │               │  g_processor->run()   │   │
 │  │                     │               │  [executes all        │   │
 │  │                     │               │   RISC-V instrs       │   │
 │  │                     │               │   for all warps]      │   │
 │  │                     │               │  g_ram[0x90020000]=   │   │
 │  │                     │  returns 0    │  [result data]        │   │
 │  │                     │◄──────────────│                       │   │
 │  │                     │               │                       │   │
 │  │   read results ─────────────────────►simx_read_mem(         │   │
 │  │   result_data[]     │               │  0x90020000,          │   │
 │  │   [expected vals]  ◄────────────────│  size, ptr)           │   │
 │  │                     │  zero-copy    │  copies from g_ram    │   │
 │  │                     │               │  to SV array          │   │
 │  │   ap.write(txn)     │               │                       │   │
 │  │   ──────────────────►  Scoreboard   │                       │   │
 │  │                     │  compares     │                       │   │
 │  │                     │  vs RTL       │                       │   │
 │  │ report_phase()      │               │                       │   │
 │  │   simx_cleanup() ───────────────────►delete g_processor     │   │
 │  │                     │               │delete g_ram           │   │
 │  └─────────────────────┘               │delete g_arch          │   │
 │                                        └───────────────────────┘   │
 └─────────────────────────────────────────────────────────────────────┘
```

---

## Critical Issues to Fix in Your Current Code

Before you move forward, there are **two mismatches** between `simx_dpi.cpp` and `simx_pkg.sv` that will cause linker errors or runtime failures:

1. **`simx_init` signature mismatch**: Your `.sv` package declares `simx_init(string kernel_file, int num_cores, int num_warps, int num_threads)` but your `.cpp` implements `simx_init(int num_cores, int num_warps, int num_threads)`. Fix: remove `kernel_file` from the SV declaration and use `simx_load_bin` separately, OR add `kernel_file` as a parameter to the C++ function.

2. **Missing `simx_run` import**: Your `simx_pkg.sv` has `simx_step(0)` as a workaround for run-to-completion, but you have a proper `simx_run()` in the C++. Import `simx_run()` in the SV package and call it directly.

3. **`simx_write_mem` uses `const svOpenArrayHandle`**: For `inout` or `output` arrays (like in `simx_read_mem`), use non-const `svOpenArrayHandle`. For input-only arrays (like `simx_write_mem`), const is correct. Make sure the SV `input`/`inout` direction matches.



## Prompt:
2/28/2026, 3:54:51 PM

> File: 

help me to convert this message to presentation slide 
and for graphs and photos needed , just full describe it as nano banana can generate it



## Response:
2/28/2026, 4:01:34 PM



````plaintext
Thought process: Reviewed presentation conversion requirements and visual description approach.

The user wants to convert the document into a presentation. Let me read the PPTX skill first.
````

````plaintext
View: Reading PPTX skill
````

````plaintext
View: Reading pptxgenjs guide for creating from scratch
````

````plaintext
Thought process: Architected comprehensive technical presentation with GPU verification focus.

Now I have a clear picture. Let me create a professional, visually rich presentation based on the document. This is a technical presentation about SimX as a Golden Model + DPI-C for Vortex GPGPU UVM verification.

I'll plan the slides:
1. Title Slide - "SimX Golden Model & DPI-C Integration for Vortex UVM"
2. Agenda/Overview
3. What is Vortex GPGPU? (context)
4. Golden Model Options - Comparison table
5. Why SimX is the Best Golden Model
6. Is C++ Normal for Golden Models? (yes, industry standard)


7. Integration Methods Overview (all 7 methods)
8. DPI-C Deep Dive - What IS it?
9. DPI-C Architecture Diagram
10. How DPI-C Enables SimX in UVM
11. UVM Environment Architecture
12. Complete Data Flow - Write/Read Example
13. File-by-File Breakdown
14. Critical Issues / Next Steps
15. Conclusion

For the color scheme, I'm going with a tech-forward dark palette—"Midnight Executive" with navy and teal accents that fits the GPU verification context. Primary color is deep navy, with teal as the secondary accent.

Now I'm writing the pptxgenjs script to build out roughly 15 slides. I need to be careful with the syntax—no hash symbols in color codes, proper shadow objects, and bullet formatting. I'll use descriptive text placeholders for images throughout.

Building the presentation structure starting with a title slide using the dark navy background with decorative circuit elements, then moving into the agenda, golden model comparison, and why SimX stands out... Now I'm setting up the color palette and typography—dark navy backgrounds with electric cyan accents, keeping everything in Calibri for cross-platform compatibility. I'll use labeled placeholder boxes with descriptions for the diagrams since the user wants clear visual explanations rather than actual images.

Writing the presentation script...
````

Now I'll build the full presentation. Let me design it with a professional dark GPU/tech aesthetic.

````plaintext
Bash Tool: Check pptxgenjs availability
````

````plaintext
Create File: Creating full presentation script for SimX DPI-C UVM talk
````

````plaintext
Bash Tool: Build the presentation
````

````plaintext
Thought process: Debugged presentation formatting and resolved color configuration issues.

The presentation was created. The warnings are about "00000000" colors - those are from the transparent borders I tried to use. Let me fix those by removing the transparent border approach - instead of using `line:{color:"00000000"}`, I should just remove the line property or use the same color as the fill to hide borders. Let me fix and rebuild quickly.

Actually, the file was created successfully. The warnings are minor (just invalid 8-char colors getting replaced with black). The visual issue is that borders I wanted to hide might show as black instead. Let me fix those specific instances.
````

The file built but has some border issues. Let me fix the transparent border workarounds quickly.

````plaintext
Bash Tool: Check remaining transparent color issues
````

````plaintext
Bash Tool: Rebuild and check output
````

Clean build. Now let me do a quick content QA:

````plaintext
Bash Tool: QA check of presentation content
````

````plaintext
Bash Tool: Count slides and copy to output
````

All 14 slides confirmed. Let me do a visual render check:

````plaintext
Bash Tool: Convert to PDF for visual QA
````

````plaintext
Bash Tool: Convert to images for visual inspection
````

````plaintext
View: Visually inspect title slide
````

````plaintext
View: Check agenda slide
````

````plaintext
View: Check comparison and table slides
````

````plaintext
View: Check slides 7 and 8 - DPI-C concept and architecture
````

````plaintext
View: Check architecture diagram and data flow slides
````

````plaintext
View: Check critical issues and conclusion slides
````

````plaintext
View: Check conclusion slide
````

Everything looks sharp and professional. The presentation is ready.

````plaintext
Present File: /mnt/user-data/outputs/Vortex_SimX_DPI_UVM.pptx
````

Here's your **14-slide presentation** with a dark GPU/tech aesthetic (deep navy + electric cyan palette). Here's the slide breakdown:

| # | Slide | Layout |
|---|---|---|
| 1 | **Title** | Dark full-bleed + circuit board GPU chip art |
| 2 | **Agenda** | 6 numbered cards (2 columns) |
| 3 | **Golden Model Options** | 4-column card comparison (Spike/POCL/Verilator/SimX) |
| 4 | **Why SimX — Deep Table** | Color-coded ✓/✗ comparison table, SimX column highlighted green |
| 5 | **C++ for Golden Models** | Two-column: "Why C++" reasons + "Industry Precedent" ARM/Intel/NVIDIA |
| 6 | **All 7 Integration Methods** | Card grid with color-coded tags (CHOSEN/LEGACY/DEAD) |
| 7 | **What is DPI-C?** | Red "NOT" banner + Green "IS" banner + 3-step flow diagram |
| 8 | **DPI-C Architecture** | Full process memory diagram: SV kernel → DPI Runtime → .so |
| 9 | **UVM Environment** | Block diagram: Agent → Monitor → Scoreboard + SimX golden model |
| 10 | **7-Phase Data Flow** | Color-coded phases with SV ↔ C++ split per card |
| 11 | **simx_dpi.cpp Deep-Dive** | 4 code sections with explanations |
| 12 | **simx_pkg.sv + Makefile** | Side-by-side with highlighted critical mismatch warning |
| 13 | **Critical Issues to Fix** | 3 issues with red/orange severity badges and exact fixes |
| 14 | **Key Takeaways** | Icon circles + orange warning for action items |

> **For slides 8–9 (architecture diagrams):** Each slide has a detailed image description embedded in comments in the script — pass those descriptions directly to an AI image generator (Midjourney, DALL-E, etc.) and drop the result in as a background or side panel.

> File: /mnt/user-data/outputs/Vortex_SimX_DPI_UVM.pptx



## Prompt:
2/28/2026, 6:00:07 PM

Add more details and explaination in the part of the project files what it's and what it do deeply 
also reveiw the example , i mean by the example how i can use functions in simx_dpi.cpp inside uvm test and then how these functions executed on simx files and the how the output send to uvm again and that is it .

Add more details and explaination in the part of the project files what it's and what it do deeply 
also reveiw the example , i mean by the example how i can use functions in simx_dpi.cpp inside uvm test and then how these functions executed on simx files and the how the output send to uvm again and that is it .



## Response:
2/28/2026, 6:10:36 PM



````plaintext
Thought process: Architected deeper file documentation and clarified DPI-to-SimX-to-UVM execution flow.

The user wants me to enhance the presentation with:
1. More details on the project files (simx_dpi.cpp, simx_pkg.sv, Makefile, vortex_config.sv) - deeper explanation
2. A clearer example walkthrough showing: UVM test calls functions → functions execute in SimX → output returns to UVM

Let me read the existing presentation script and build enhanced slides. I'll need to add more file deep-dive slides and improve the example flow slides.

Let me first check what files are available and then build the enhanced presentation.
````

````plaintext
View: Read the existing build script to understand current slide structure
````

````plaintext
Thought process

Let me check the uploaded files to understand what's available for deeper content.
````

````plaintext
Bash Tool: Check uploaded files
````

````plaintext
Bash Tool: Read the full simx_dpi.cpp file
````

````plaintext
Bash Tool: Read the full simx_pkg.sv file
````

````plaintext
Bash Tool: Read config and test_top files
````

````plaintext
Thought process: Architected comprehensive file deep-dives and traced complete DPI-C execution flow.

Now I have all the files. Let me plan the new slides to add:

1. **File Deep Dive - simx_dpi.cpp** - Much more detailed, going through each function with actual code
2. **File Deep Dive - simx_pkg.sv** - Show the actual declarations, the mismatch, what needs to be fixed
3. **File Deep Dive - vortex_config.sv** - Show the rich config with all fields
4. **File Deep Dive - Makefile** - The build system
5. **File Deep Dive - test_top.sv** - The actual test module


6. **Complete Example Flow** - Step by step: UVM calls → DPI-C bridge → SimX executes → results return

I'll keep slides 1-12 as they are and significantly expand the file deep-dive section with a thorough example walkthrough. My new structure will have simx_dpi.cpp split across two slides—one covering the file overview, global state, and initialization logic, another handling memory operations, DCR functions, and execution control.

Then I'm moving into the SystemVerilog side with simx_pkg.sv showing the actual DPI declarations and UVM class structure, followed by vortex_config.sv's configuration object and test_top.sv's test module. The example walkthrough spans four slides tracing the complete flow from UVM test invocation through DPI-C handoff, SimX processing, and results returning back to the testbench. I'll finish with slides on critical issues that need fixing and key takeaways, keeping the first 10 slides intact.
````

Now I have all the real file contents. Let me build a comprehensive enhanced version with deep file explanations and a proper end-to-end example walkthrough.

````plaintext
Bash Tool: Backup original and check size
````

````plaintext
Create File: Build enhanced presentation v2 with deep file dives and complete example walkthrough
````

````plaintext
Bash Tool: Build the v2 presentation
````

````plaintext
Bash Tool: Count slides
````

````plaintext
Bash Tool: Convert to PDF and images for visual QA
````

````plaintext
View: Check section divider slide
````

````plaintext
View: Check simx_dpi.cpp deep dive slide
````

````plaintext
View: Check memory functions slide
````

````plaintext
View: Check bootstrap slide
````

````plaintext
View: Check example layers slides
````

````plaintext
View: Check SimX execution layer
````

````plaintext
View: Check results return slide and call stack
````

````plaintext
View: Check conclusion slide
````

Everything looks excellent. Let me copy the final file to outputs.

````plaintext
Bash Tool: Copy to outputs
````

````plaintext
Present File: /mnt/user-data/outputs/Vortex_SimX_DPI_v2_DeepGuide.pptx
````

Here's your **v2 — 27-slide deep guide**, rebuilt from the actual source files. Here's what's new:

**File Deep-Dive section (Slides 11–18):**
- **Slide 12** — `simx_dpi.cpp` Part 1: Why it's *the bridge layer*, why static globals (not local vars!), full annotated `simx_init()` code
- **Slide 13** — `simx_dpi.cpp` Part 2: `simx_write_mem()` / `simx_read_mem()` with exact `svOpenArrayHandle` / `svGetArrayPtr()` explanation, `simx_run()` and `simx_dcr_write()`
- **Slide 14** — `simx_dpi.cpp` Part 3: The x3 bootstrap problem explained with the real bytes, memory layout diagram
- **Slide 15** — `simx_pkg.sv`: DPI declarations, the UVM class structure, the 2 mismatches shown in actual code with red/green highlights
- **Slide 16** — `vortex_config.sv`: All 5 config categories (Architecture, ISA, Cache, Test/SimX, Memory Map), the `set_defaults_from_vx_config()` danger
- **Slide 17** — `Makefile`: The core build line annotated, 4 annotated flag blocks, the `-sv_lib simx_model` key flag
- **Slide 18** — `test_top.sv`: Shows the correct DPI declarations (reference for fixing simx_pkg.sv), 6-step execution sequence

**Complete Example section (Slides 19–25):**
- **Slide 20** — Layer 1: Your UVM `run_phase` code, exactly what SV does when it calls the function
- **Slide 21** — Layer 2: DPI-C bridge — symbol resolution → argument marshaling → CPU jump (3 sub-steps)
- **Slide 22** — Layer 3: Every line of `simx_write_mem()` annotated with sidebar explanations
- **Slide 23** — Layer 3b: `simx_run()` internals + **before/after memory table** showing g_ram state
- **Slide 24** — Layer 4: `simx_read_mem()` → transaction packaging → `ap.write()` → scoreboard compare → cleanup warning
- **Slide 25** — Full call stack: 6 layers from SV to `RAM::write()` and back, all in one diagram

> File: /mnt/user-data/outputs/Vortex_SimX_DPI_v2_DeepGuide.pptx



## Prompt:
undefined





---
Powered by [Claude Exporter](https://www.ai-chat-exporter.net)