# Integrating SimX golden model in UVM verification

**Created:** 4/3/2026 4:46:05  
**Updated:** 4/3/2026 4:48:44  
**Exported:** 5/13/2026 16:25:59  
**Link:** [https://claude.ai/chat/1ec63862-7747-4cae-b697-d7075033c617](https://claude.ai/chat/1ec63862-7747-4cae-b697-d7075033c617)  

## Prompt:
4/3/2026, 4:46:08 AM

act as expert in digital IC verification working in UVM . i need to integrate simx / golden model in the uvm . 
fully integration , that (simx) is the ref_model for the vortex we will use . what i must to do .
study first all uvm files deeply to tell me how i can do that .



## Response:
4/3/2026, 4:48:44 AM



````plaintext
Thought process: Analyzed UVM architecture and identified SimX integration gaps systematically.

The user wants to fully integrate SimX (the golden model) into their UVM environment for the Vortex GPGPU. Let me analyze the existing code carefully.

From studying the files:

1. **SimX DPI library** (`uvm_env/ref_model/simx_dpi.cpp`) - C++ DPI wrapper that interfaces with Vortex's SimX simulator
2. **SimX package** (`uvm_env/ref_model/simx_pkg.sv`) - SV package with DPI imports and a `simx_golden_model` UVM component


3. **Scoreboard** (`uvm_env/vortex_scoreboard.sv`) - Already has SimX DPI imports and comparison logic, but SimX is currently disabled
4. **Config** (`uvm_env/vortex_config.sv`) - Contains the SimX configuration flags and path settings
5. **Env package** (`uvm_env/vortex_env_pkg.sv`) - Imports the DPI functions at the package level
6. **Smoke test** (`uvm_tests/vortex_smoke_test.sv`) - Currently disables SimX during test execution

The root cause is that the SimX shared library isn't being compiled or linked into the simulation. The DPI imports exist but the actual `simx_model.so` file isn't being passed to vsim, so the function lookups fail. I need to build the shared library as part of the compilation flow, add the `-sv_lib simx_model` flag to vsim, enable SimX in the test configuration, and make sure the program gets loaded into SimX before the scoreboard comparison runs after EBREAK. I'm seeing a mismatch between what the scoreboard expects and what the test actually provides—the smoke test loads a hex file, but the scoreboard's `simx_load_bin` call expects a binary file, so I need to either convert the hex file or switch to `simx_load_hex` depending on what format the program is actually in.

The real flow should be: the UVM testbench loads the hex file into the memory model, SimX loads the same program into its own RAM, the RTL executes, and when it hits an EBREAK instruction, the scoreboard runs a final comparison between the RTL's memory state and SimX's state.

To make this work, I need to ensure the simx_model.so library gets built in the run script, pick the right loader function based on the program file format, make sure simx_enable is set correctly when the library is available, and handle how DCR writes flow from the TB through the scoreboard to SimX—the DCR monitor observes writes and forwards them, but I need to verify the agent is properly connected.

The trickier part is memory mirroring: when the RTL writes to AXI, the scoreboard mirrors those writes to SimX's RAM, but if SimX is running its own program, it already has the correct memory state, so the comparison should focus on the result memory region rather than everything.

Actually, I'm realizing there's a fundamental issue with the approach. The scoreboard loads the program into SimX upfront, then mirrors DUT memory writes as they happen, and finally runs SimX at the end. But this means SimX would execute with a RAM that's already been modified by DUT writes, which defeats the purpose of having a golden reference. The correct flow should be: load program into SimX, run it to completion for the golden result, run RTL separately, then compare—not interleave the writes. then compare against what the DUT produced. But looking at the scoreboard's write_mem function, it mirrors DUT stores to SimX RAM before simx_run() is even called, which corrupts the program code in SimX. The real issue is that Option A—loading the program, running SimX to get golden results upfront, then comparing against the DUT—would be cleaner than trying to mirror writes during execution.

Actually, I need to reconsider the read logic. The scoreboard only uses SimX memory for comparisons after simx_ran becomes true, and it falls back to shadow_memory otherwise. So the design intent seems to be: don't write DUT stores to SimX at all, only to shadow_memory, then run SimX independently on the clean program and compare the results afterward. The shadow_memory is populated by DUT writes, SimX memory is populated by simx_run().

The `simx_write_mem` calls before simx_run are problematic. They're writing DUT store data to SimX RAM. If the program writes results to specific addresses, these get written to SimX RAM. Then when simx_run() runs, it finds those addresses already set. That would make SimX produce wrong results for reads from those addresses.

I need to reconsider the design. For a GPU program like the simple test with just EBREAK, there are no stores. For more complex programs with stores, the DUT stores go to memory while SimX stores should also go to those same addresses when SimX runs the same program, then we compare the final state.

The cleanest approach is to load the program into SimX at start, let the DUT run and capture stores in shadow_memory, then on EBREAK call simx_run() so SimX executes the program independently and produces its own results, finally comparing shadow_memory against simx_read_mem for the result region.

The `simx_write_mem` calls should be removed from the scoreboard's `write_mem`/`write_axi` functions since they corrupt SimX's independent execution. Looking at what needs to change, the main issue is that simx_load_hex reads 32-bit hex words per line but the Vortex hex file uses byte format, so I need to fix how the hex file is loaded into SimX. is byte format (from objcopy --verilog-data-width=1)

**2. Fix vortex_scoreboard.sv**:
- Remove `simx_write_mem` from `write_mem`/`write_axi` (those corrupt SimX)
- Keep shadow_memory for DUT writes
- On EBREAK: run SimX, compare result regions
- Fix program loading: use hex for .hex files

**3. Fix vortex_smoke_test.sv**:
- Enable SimX

**4. Fix run script**:
- Build simx_model.so
- Pass -sv_lib to vsim

**5. Fix simx_pkg.sv**:
- The `simx_golden_model` class is unused currently
- Either integrate it or remove it

Now I'm looking at the `simx_load_hex` function in simx_dpi.cpp. It's parsing 32-bit words from each line of the hex file, but our objcopy output uses single-byte tokens per line, so this loader won't handle our format correctly.

The mem_model.sv loader is smarter—it detects the format by checking token length: 2 characters for byte format, 8 for word format. I should apply similar logic to simx_load_hex, or alternatively just use binary files with simx_load_bin instead.

Actually, the cleaner path is to have the scoreboard write program bytes directly to SimX RAM via simx_write_mem during initialization, then set the startup address with simx_dcr_write. This sidesteps the hex parsing entirely, though we'd need a hook from mem_model to the scoreboard to coordinate that. Since we don't have that, the next best option is either adding a new DPI function for Verilog hex format or fixing the existing hex loader. The real solution is to think through the full integration: the scoreboard initializes SimX with architecture parameters, mirrors DCR writes to SimX for things like startup address, and handles program loading through a consistent interface.

For program loading specifically, I could either add a scoreboard function that reads from mem_model and mirrors to SimX, let the test call SimX directly, or fix simx_load_hex to handle the Verilog byte format so the scoreboard can just call it with the program path. The last option feels cleanest, but I need to verify the timing works out.

Looking at what `program_simple.hex` actually contains—it's a Verilog hex file from objcopy with byte-width formatting, so it has address markers like `@00000000` followed by space-separated byte values. The scoreboard currently calls `simx_load_bin`, so I need to make it detect the file extension and call the appropriate loader function instead.

Now I'm laying out the concrete changes needed across the codebase: fix the hex loader in the DPI layer to handle Verilog byte format, remove the contaminating SimX RAM writes from the scoreboard's memory and AXI handlers, update the program loading logic to detect `.hex` files, conditionally enable SimX in the test configuration, and add the necessary build and simulation steps to the run script.

I'm also thinking through the bootstrap initialization flow—the scoreboard calls `simx_init_exit_code_register()` which installs bootstrap code at `startup_addr - 16`, then the program loads at `startup_addr`, and when SimX runs it starts at the bootstrap address, initializes the exit code register in x3, and jumps to the program entry point. This should work fine as long as the bootstrap address is properly accessible in memory.

For the result comparison, the scoreboard checks the region from `result_base_addr` to `result_base_addr + result_size_bytes`. In the simple test case with just EBREAK, the program doesn't actually write anything to the result region, so the comparison would find empty memory on both the DUT and SimX sides.

Now I'm thinking through how to structure my answer for the user. I should give them a clear action plan: what the current state is, what needs to change in each file, and how to unify the scripts into a single Makefile. I'll be concrete about the specific modifications needed.

I'm also noticing that `simx_pkg.sv` has a `simx_golden_model` class that's never actually used—the scoreboard already has all the SimX logic embedded in it. So I need to decide whether to keep the embedded approach or refactor to use the separate class.

Looking at the actual error from the test run, the real problem is clear: `simx_model.so` was never built or linked into the simulation. The run script looks for a DPI library path that doesn't exist, so it gets skipped entirely, and there's no reference to `simx_model.so` anywhere in the command. That's the root cause I need to fix.
````

## Full SimX Integration Analysis & Action Plan

After studying all your UVM files deeply, here's what I found and exactly what needs to change.

---

## Current State Assessment

**What exists but is broken/incomplete:**

1. `vortex_env_pkg.sv` — DPI imports ✓ but functions resolve to nothing (no `.so` linked)
2. `vortex_scoreboard.sv` — SimX logic exists but has a **fundamental design flaw** (explained below)
3. `vortex_smoke_test.sv` — `cfg.simx_enable = 0` hardcoded — SimX disabled
4. `run_vortex_uvm_enhanced.sh` — Looks for `vortex_dpi`, never references `simx_model.so`
5. `simx_pkg.sv` — `simx_golden_model` class exists but is never instantiated (dead code)
6. `simx_dpi.cpp` — `simx_load_hex()` reads 32-bit word-per-line format, but your hex files are **byte-per-token verilog format** (from `objcopy --verilog-data-width=1`)

**The fundamental scoreboard flaw:**

In `vortex_scoreboard.sv::write_mem()` and `write_axi()`:
```systemverilog
if (cfg.simx_enable) begin
    simx_write_mem(64'(tr.addr), 8, wr);  // ← WRONG: contaminates SimX RAM
end
```

You're mirroring DUT memory **stores** into SimX RAM *before* `simx_run()`. When SimX runs later, it finds its RAM already modified by DUT output — making comparison meaningless. SimX must run on a **clean** independent copy of the program.

---

## Changes Required — File by File

### Change 1: `uvm_env/ref_model/simx_dpi.cpp`

Fix `simx_load_hex` to handle verilog byte-format (byte per token, space separated):

```cpp
// REPLACE the entire simx_load_hex function with this:
int simx_load_hex(const char* filepath) {
    if (!g_initialized || !g_ram) {
        std::cerr << "[SimX-DPI] Error: not initialized" << std::endl;
        return -1;
    }

    std::ifstream file(filepath);
    if (!file.is_open()) {
        std::cerr << "[SimX-DPI] Cannot open hex file: " << filepath << std::endl;
        return -1;
    }

    std::string line;
    uint64_t current_addr = 0;
    int bytes_loaded = 0;
    bool format_detected = false;
    bool is_byte_format  = true;   // objcopy --verilog-data-width=1

    while (std::getline(file, line)) {
        // Strip CR/LF
        while (!line.empty() && (line.back() == '\r' || line.back() == '\n'))
            line.pop_back();
        if (line.empty() || line[0] == '/' || line[0] == '#') continue;

        if (line[0] == '@') {
            // Address marker
            current_addr = std::stoull(line.substr(1), nullptr, 16);
            if (bytes_loaded == 0) g_startup_addr = current_addr;
            format_detected = false;   // re-detect on next data line
            continue;
        }

        // Tokenise on whitespace
        std::istringstream iss(line);
        std::string token;
        bool first_token = true;

        while (iss >> token) {
            if (first_token && !format_detected) {
                is_byte_format  = (token.size() <= 2);
                format_detected = true;
            }
            first_token = false;

            if (is_byte_format) {
                uint8_t b = (uint8_t)std::stoul(token, nullptr, 16);
                g_ram->write(&b, current_addr++, 1);
                bytes_loaded++;
            } else {
                uint32_t w = std::stoul(token, nullptr, 16);
                uint8_t bytes[4] = {
                    (uint8_t)(w),
                    (uint8_t)(w >> 8),
                    (uint8_t)(w >> 16),
                    (uint8_t)(w >> 24)
                };
                g_ram->write(bytes, current_addr, 4);
                current_addr  += 4;
                bytes_loaded  += 4;
            }
        }
    }

    std::cout << "[SimX-DPI] Loaded " << bytes_loaded
              << " bytes from hex '" << filepath
              << "' startup=0x" << std::hex << g_startup_addr << std::dec << std::endl;

    // Apply startup address to DCR
    g_processor->dcr_write(VX_DCR_BASE_STARTUP_ADDR0, g_startup_addr & 0xFFFFFFFF);
    return (bytes_loaded > 0) ? 0 : -1;
}
```

Also add `#include <sstream>` at the top of the file (after existing includes).

---

### Change 2: `uvm_env/vortex_scoreboard.sv`

Three specific edits:

**Edit A** — `write_mem()`: Remove SimX contamination, keep shadow only:

```systemverilog
// FIND this block in write_mem():
    if (tr.rw) begin
      shadow_memory[tr.addr] = tr.data;
      if (cfg.simx_enable) begin          // ← DELETE these 4 lines
        byte unsigned wr[];  wr = new[8];
        for (int i = 0; i < 8; i++) wr[i] = tr.data[i*8 +: 8];
        simx_write_mem(64'(tr.addr), 8, wr);
      end                                  // ← DELETE
    end

// REPLACE with:
    if (tr.rw) begin
      shadow_memory[tr.addr] = tr.data;   // DUT shadow only — SimX runs independently
    end
```

**Edit B** — `write_axi()`: Same fix:

```systemverilog
// FIND this block in write_axi():
        shadow_memory[baddr[31:0]] = bdata;
        if (cfg.simx_enable) begin          // ← DELETE these 4 lines
          byte unsigned wr[];  wr = new[8];
          for (int i = 0; i < 8; i++) wr[i] = bdata[i*8 +: 8];
          simx_write_mem(baddr, 8, wr);
        end                                  // ← DELETE

// REPLACE with:
        shadow_memory[baddr[31:0]] = bdata; // DUT shadow only
```

**Edit C** — `run_phase()`: Fix program loading to detect hex vs bin:

```systemverilog
// FIND this block in run_phase():
    // Pre-load program into SimX RAM
    if (cfg.program_path != "") begin
      status = simx_load_bin(cfg.program_path, 64'(cfg.startup_addr));
      if (status != 0)
        `uvm_error("SCOREBOARD",
          $sformatf("simx_load_bin('%s') failed", cfg.program_path))

// REPLACE with:
    // Pre-load program into SimX RAM (detect format from extension)
    if (cfg.program_path != "") begin
      string path = cfg.program_path;
      int    path_len = path.len();
      bit    is_hex;

      // Detect .hex extension (last 4 chars)
      is_hex = (path_len > 4) &&
               (path.substr(path_len-4, path_len-1) == ".hex");

      if (is_hex) begin
        status = simx_load_hex(cfg.program_path);
        if (status != 0)
          `uvm_error("SCOREBOARD",
            $sformatf("simx_load_hex('%s') failed", cfg.program_path))
        else
          `uvm_info("SCOREBOARD",
            $sformatf("SimX: loaded hex '%s'", cfg.program_path), UVM_MEDIUM)
      end else begin
        status = simx_load_bin(cfg.program_path, 64'(cfg.startup_addr));
        if (status != 0)
          `uvm_error("SCOREBOARD",
            $sformatf("simx_load_bin('%s') failed", cfg.program_path))
        else
          `uvm_info("SCOREBOARD",
            $sformatf("SimX: loaded bin '%s' @ 0x%0h",
              cfg.program_path, cfg.startup_addr), UVM_MEDIUM)
      end
```

---

### Change 3: `uvm_tests/vortex_smoke_test.sv`

In `customize_config()`, change SimX enable:

```systemverilog
// FIND:
        cfg.simx_enable       = 0;

// REPLACE with:
        // Enable SimX only if the shared library was actually linked
        // (controlled by +SIMX plusarg or always-on if .so built)
        if ($test$plusargs("NO_SIMX"))
            cfg.simx_enable = 0;
        else
            cfg.simx_enable = 1;
```

---

### Change 4: `scripts/run_vortex_uvm_enhanced.sh`

Add SimX build and linking. Find the **DPI_FLAG** section and replace:

```bash
# FIND this block (around line with "Vortex DPI"):
VORTEX_DPI_LIB="$FLISTS_DIR/vortex_dpi"

# --- UVM DPI (REQUIRED) ---
UVM_DPI_LIB="$QUESTA_HOME/uvm-1.2/linux_x86_64/uvm_dpi"

DPI_FLAG=""

# Add UVM DPI FIRST (critical)
if [[ -f "${UVM_DPI_LIB}.so" ]]; then
    DPI_FLAG="$DPI_FLAG -sv_lib ${UVM_DPI_LIB}"
    print_success "UVM DPI loaded: ${UVM_DPI_LIB}.so"
else
    print_error "UVM DPI not found! This WILL crash simulation."
fi

# Add Vortex DPI if exists
if [[ -f "${VORTEX_DPI_LIB}.so" ]]; then
    DPI_FLAG="$DPI_FLAG -sv_lib ${VORTEX_DPI_LIB}"
    print_success "Vortex DPI loaded: ${VORTEX_DPI_LIB}.so"
else
    print_warning "Vortex DPI not found (optional)"
fi
```

Replace with:

```bash
# ── DPI LIBRARY PATHS ────────────────────────────────────────────────────────
UVM_DPI_LIB="$QUESTA_HOME/uvm-1.2/linux_x86_64/uvm_dpi"
SIMX_REF_DIR="$PROJECT_ROOT/uvm_env/ref_model"
SIMX_MODEL_LIB="$SIMX_REF_DIR/simx_model"

DPI_FLAG=""
SIMX_ENABLED=0

# --- UVM DPI (REQUIRED) ---
if [[ -f "${UVM_DPI_LIB}.so" ]]; then
    DPI_FLAG="$DPI_FLAG -sv_lib ${UVM_DPI_LIB}"
    print_success "UVM DPI: ${UVM_DPI_LIB}.so"
else
    print_error "UVM DPI not found! Simulation will crash."
fi

# --- SimX Golden Model (build if needed) ---
print_header "SimX Golden Model"

if [[ -z "$VORTEX_HOME" ]]; then
    print_warning "VORTEX_HOME not set — skipping SimX build"
elif [[ ! -d "$VORTEX_HOME/sim/simx/obj" ]]; then
    print_warning "SimX not built (no obj/ in $VORTEX_HOME/sim/simx)"
    print_info  "Build SimX first: cd \$VORTEX_HOME/sim/simx && make"
else
    print_info "Building SimX DPI library..."
    (
        cd "$SIMX_REF_DIR" || exit 1
        ARCH_FLAGS="-DNUM_CLUSTERS=${NUM_CLUSTERS} -DNUM_CORES=${NUM_CORES}"
        ARCH_FLAGS="$ARCH_FLAGS -DNUM_WARPS=${NUM_WARPS} -DNUM_THREADS=${NUM_THREADS}"
        make build \
            VORTEX_HOME="$VORTEX_HOME" \
            QUESTA_HOME="$QUESTA_HOME" \
            EXTRA_CXXFLAGS="$ARCH_FLAGS" 2>&1
    )
    if [[ $? -eq 0 && -f "${SIMX_MODEL_LIB}.so" ]]; then
        DPI_FLAG="$DPI_FLAG -sv_lib ${SIMX_MODEL_LIB}"
        SIMX_ENABLED=1
        print_success "SimX DPI built and linked: simx_model.so"
    else
        print_warning "SimX DPI build failed — running without golden model"
    fi
fi

# Add NO_SIMX plusarg if SimX not available
if [[ $SIMX_ENABLED -eq 0 ]]; then
    SIM_OPTS="$SIM_OPTS +NO_SIMX"
    print_info "SimX disabled (add +NO_SIMX to suppress this)"
fi
# ─────────────────────────────────────────────────────────────────────────────
```

Also update the `Makefile` (`uvm_env/ref_model/Makefile`) to accept `EXTRA_CXXFLAGS`:

```makefile
# FIND this line:
CXXFLAGS     += $(ARCH_CONFIGS)

# ADD after it:
CXXFLAGS     += $(EXTRA_CXXFLAGS)
```

---

## Combined Build Script

Create `scripts/build_and_run.sh` that replaces both:

```bash
#!/usr/bin/env bash
################################################################################
# build_and_run.sh - Single entry point: builds SimX .so + compiles RTL/UVM + runs
################################################################################
set -e
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# Forward all args to the enhanced runner (which now builds SimX internally)
exec "$SCRIPT_DIR/run_vortex_uvm_enhanced.sh" "$@"
```

The key point: the run script now handles SimX build internally. The standalone `ref_model/Makefile` remains for direct SimX-only testing.

---

## Rebuild Order

After all changes, your build sequence becomes:

```
1. make -C $VORTEX_HOME/sim/simx          (build simx core — one time)
2. scripts/build_and_run.sh               (builds simx_model.so + compiles RTL/UVM + runs)
```

---

## Quick Validation

After changes, the log should show instead of warnings:
```
✓ SimX DPI built and linked: simx_model.so
UVM_INFO [SCOREBOARD] Initialising SimX: cores=1 warps=4 threads=4
UVM_INFO [SCOREBOARD] SimX: loaded hex 'program_simple.hex'
UVM_INFO [SCOREBOARD] EBREAK detected — running SimX and comparing results.
UVM_INFO [SCOREBOARD] SimX done — exit code = 0
```

The most critical single change is **removing `simx_write_mem` from `write_mem`/`write_axi`** — without that fix, all comparisons are invalid even when the `.so` is linked.





---
Powered by [Claude Exporter](https://www.ai-chat-exporter.net)