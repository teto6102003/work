# Integrating golden model simx with UVM

**Created:** 4/1/2026 0:22:11  
**Updated:** 4/1/2026 14:07:33  
**Exported:** 5/13/2026 16:15:31  
**Link:** [https://claude.ai/chat/bab85908-4cf6-47fa-8afd-28fc5adb78fd](https://claude.ai/chat/bab85908-4cf6-47fa-8afd-28fc5adb78fd)  

## Prompt:
4/1/2026, 12:22:15 AM

Now i need to integrate the golden model simx inside uvm , and need to now how i can compine between the makefile for golden model and the scripte that run full uvm



## Response:
4/1/2026, 12:23:41 AM



````plaintext
Thought process: Architected SimX DPI-C integration strategy combining Makefile build with UVM simulation script.

The user wants to integrate the SimX golden model (DPI-C) into the UVM environment and understand how to combine the Makefile (for building the SimX DPI shared library) with the run_vortex_uvm_enhanced.sh script (for running the full UVM simulation).

Let me analyze what we have:

1. **SimX DPI**: The `uvm_env/ref_model/Makefile` builds `simx_model.so` from `simx_dpi.cpp`
2. **UVM Run Script**: `scripts/run_vortex_uvm_enhanced.sh` runs the full UVM simulation


3. The DPI imports are already declared in the SystemVerilog packages, and the scoreboard has SimX integration ready to go. The main integration steps are building the shared library, passing it to vsim with the `-sv_lib` flag, and enabling SimX in the configuration with the appropriate path settings.

Looking at the existing run script, I notice it references DPI flags but the library path seems misconfigured—it's pointing to a directory instead of the actual `simx_model.so` file. The Makefile in the ref_model directory builds this shared library, so I need to ensure the build runs first and then pass the correct path to the simulator. The key distinction is that we're using DPI-C for the UVM integration rather than the standalone simx executable approach. the actual simx binary

Looking more carefully at the scoreboard:
- `simx_init()` - initializes the processor
- `simx_load_bin()` - loads program
- `simx_run()` - runs to completion
- `cfg.simx_enable` controls whether simx is used
- `cfg.program_path` is used to load the program

The integration plan:
1. **Build step**: Use Makefile to build `simx_model.so`
2. **Link step**: Pass `-sv_lib simx_model` to vsim (already partially done in the script with VORTEX_DPI_LIB)
3. **Runtime**: Pass `+SIMX` or `+SIMX_PATH` plusargs

Let me write a clear integration guide and modified script.

Key changes needed:
1. Fix the `VORTEX_DPI_LIB` path in the script to point to `uvm_env/ref_model/simx_model`

Now I need to add command-line flags to control simx behavior, including a `--simx` option and optional path specification, then ensure the build runs before simulation and the appropriate plusargs get passed to vsim. Looking at how the scoreboard uses `cfg.program_path` to load the binary through the DPI interface, I should verify that the configuration properly connects these pieces together.
````

## SimX DPI Integration into UVM — Complete Guide

### Understanding the Integration Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Integration Flow                             │
├──────────────────────┬──────────────────────────────────────────┤
│  Makefile            │  run_vortex_uvm_enhanced.sh              │
│  (ref_model/)        │  (scripts/)                              │
│                      │                                           │
│  1. Compile          │  1. Call make build → simx_model.so      │
│     simx_dpi.cpp     │  2. Compile RTL + UVM                    │
│  2. Link SimX objs   │  3. vsim -sv_lib simx_model              │
│  3. Output:          │     +SIMX +SIMX_PATH=...                 │
│     simx_model.so    │  4. UVM scoreboard calls DPI functions   │
└──────────────────────┴──────────────────────────────────────────┘
```

---

### Step 1 — Fix the Makefile `VORTEX_HOME` and Output Path

The Makefile already builds `simx_model.so`. You only need to make sure it is reachable from the script. Add one target to copy the `.so` to a known location:

```makefile
# Add at the end of uvm_env/ref_model/Makefile

# ── Install target: copy .so to flists/ so the run script finds it ──
INSTALL_DIR ?= $(PROJECT_ROOT)/flists
PROJECT_ROOT ?= $(shell cd ../..; pwd)

.PHONY: install
install: $(DPI_LIB)
	@echo "=== Installing DPI lib to $(INSTALL_DIR) ==="
	cp $(DPI_LIB) $(INSTALL_DIR)/simx_model.so
	@echo "=== Done: $(INSTALL_DIR)/simx_model.so ==="

# Full build + install in one command
.PHONY: deploy
deploy: install
```

---

### Step 2 — Fix `vortex_config.sv` `is_valid()` for DPI Mode

The current `is_valid()` rejects `simx_enable=1` with empty `simx_path`. In DPI-C mode the `.so` is linked at vsim time, not at runtime via a path. Add a bypass flag:

```systemverilog
// In uvm_env/vortex_config.sv — add field after simx_path:
rand bit            simx_use_dpi;      // 1 = DPI-C mode (linked .so), 0 = exec mode

// In set_defaults_from_vx_config():
simx_use_dpi = 1;   // default: DPI-C mode

// In apply_plusargs():
if ($test$plusargs("SIMX_DPI") || $test$plusargs("simx_dpi"))
    simx_use_dpi = 1;

// Replace the simx_path validation in is_valid():
if (simx_enable && !simx_use_dpi && simx_path == "") begin
    `uvm_error("VORTEX_CFG",
        "simx enabled in exec mode but simx_path not set — pass +SIMX_PATH=<path>")
    valid = 0;
end
// DPI mode: simx_path is informational only, not required
```

---

### Step 3 — Integrate into `run_vortex_uvm_enhanced.sh`

Replace the relevant sections of the script:

```bash
#!/usr/bin/env bash
################################################################################
# run_vortex_uvm_enhanced.sh  — with SimX DPI integration
################################################################################

set -e
set -o pipefail

# ── paths ─────────────────────────────────────────────────────────────────────
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"
FLISTS_DIR="$PROJECT_ROOT/flists"
REF_MODEL_DIR="$PROJECT_ROOT/uvm_env/ref_model"
SIMX_LIB_NAME="simx_model"
SIMX_LIB_SO="$FLISTS_DIR/${SIMX_LIB_NAME}.so"

# ── new flags (add to argument parsing) ───────────────────────────────────────
USE_SIMX=0
SIMX_CORES=1        # must match --cores
SIMX_WARPS=4        # must match --warps
SIMX_THREADS=4      # must match --threads
REBUILD_SIMX=0

# ── parse new args (add these cases to existing for loop) ─────────────────────
for arg in "$@"; do
    case $arg in
        --simx)           USE_SIMX=1 ;;
        --rebuild-simx)   REBUILD_SIMX=1; USE_SIMX=1 ;;
        # existing args unchanged ...
    esac
done

# sync simx arch with requested arch
SIMX_CORES=$NUM_CORES
SIMX_WARPS=$NUM_WARPS
SIMX_THREADS=$NUM_THREADS
```

---

### Step 4 — Add the SimX Build Section

Insert this block **after the environment checks** and **before compilation**:

```bash
################################################################################
# SimX DPI Library Build
################################################################################

build_simx_dpi() {
    print_header "Building SimX DPI Shared Library"

    # Check SimX objects exist
    if [[ ! -d "$VORTEX_HOME/sim/simx/obj" ]]; then
        print_error "SimX not built. Run first:"
        echo "  cd $VORTEX_HOME/sim/simx && make"
        exit 1
    fi

    # Check ramulator
    if [[ ! -f "$VORTEX_HOME/third_party/ramulator/libramulator.so" && \
          ! -f "$VORTEX_HOME/third_party/ramulator/libramulator.a"  ]]; then
        print_warning "Ramulator library not found — SimX may fail to link"
    fi

    cd "$REF_MODEL_DIR" || exit 1

    # Override Makefile variables for current arch
    make build \
        VORTEX_HOME="$VORTEX_HOME"  \
        ARCH_CONFIGS="-DNUM_CLUSTERS=$NUM_CLUSTERS \
                      -DNUM_CORES=$SIMX_CORES       \
                      -DNUM_WARPS=$SIMX_WARPS        \
                      -DNUM_THREADS=$SIMX_THREADS"  \
        2>&1 | tee "$RESULTS_RUN_DIR/logs/simx_build.log"

    if [[ $? -ne 0 ]]; then
        print_error "SimX DPI build failed — check logs/simx_build.log"
        exit 1
    fi

    # Install to flists/
    make install \
        INSTALL_DIR="$FLISTS_DIR" \
        PROJECT_ROOT="$PROJECT_ROOT" \
        2>&1 | tee -a "$RESULTS_RUN_DIR/logs/simx_build.log"

    if [[ ! -f "$SIMX_LIB_SO" ]]; then
        print_error "simx_model.so not found at $SIMX_LIB_SO after build"
        exit 1
    fi

    print_success "SimX DPI library built: $SIMX_LIB_SO"
    cd "$FLISTS_DIR" || exit 1
}

# ── decide whether to build ───────────────────────────────────────────────────
if [[ $USE_SIMX -eq 1 ]]; then
    if [[ $REBUILD_SIMX -eq 1 || ! -f "$SIMX_LIB_SO" ]]; then
        build_simx_dpi
    else
        print_success "Reusing existing SimX DPI lib: $SIMX_LIB_SO"
    fi
fi
```

---

### Step 5 — Update the DPI Flag and Simulation Options

Replace the existing DPI_FLAG block and SIM_OPTS section:

```bash
################################################################################
# DPI Library Flags
################################################################################

DPI_FLAG=""

# UVM DPI (always required)
UVM_DPI_LIB="$QUESTA_HOME/uvm-1.2/linux_x86_64/uvm_dpi"
if [[ -f "${UVM_DPI_LIB}.so" ]]; then
    DPI_FLAG="$DPI_FLAG -sv_lib ${UVM_DPI_LIB}"
    print_success "UVM DPI: ${UVM_DPI_LIB}.so"
else
    print_error "UVM DPI not found at ${UVM_DPI_LIB}.so — simulation will crash"
    exit 1
fi

# SimX DPI (only when --simx requested)
if [[ $USE_SIMX -eq 1 ]]; then
    if [[ -f "$SIMX_LIB_SO" ]]; then
        # vsim -sv_lib takes path WITHOUT .so extension
        DPI_FLAG="$DPI_FLAG -sv_lib ${SIMX_LIB_SO%.so}"
        print_success "SimX DPI: $SIMX_LIB_SO"

        # Set LD_LIBRARY_PATH so ramulator.so is found at runtime
        export LD_LIBRARY_PATH="$VORTEX_HOME/third_party/ramulator:${LD_LIBRARY_PATH:-}"
    else
        print_error "SimX .so not found: $SIMX_LIB_SO"
        print_error "Run with --rebuild-simx to build it first"
        exit 1
    fi
fi

################################################################################
# Simulation Options — add SimX plusargs
################################################################################

SIM_OPTS="+UVM_TESTNAME=$TEST_NAME"
SIM_OPTS="$SIM_OPTS +NUM_CLUSTERS=$NUM_CLUSTERS"
SIM_OPTS="$SIM_OPTS +NUM_CORES=$NUM_CORES"
SIM_OPTS="$SIM_OPTS +NUM_WARPS=$NUM_WARPS"
SIM_OPTS="$SIM_OPTS +NUM_THREADS=$NUM_THREADS"
SIM_OPTS="$SIM_OPTS +TIMEOUT=$TIMEOUT_CYCLES"
SIM_OPTS="$SIM_OPTS +STARTUP_ADDR=$STARTUP_ADDR_HEX"

if [[ "$MEMORY_INTERFACE" == "axi" ]]; then
    SIM_OPTS="$SIM_OPTS +USE_AXI_WRAPPER"
fi

if [[ -n "$PROGRAM_HEX" ]]; then
    SIM_OPTS="$SIM_OPTS +PROGRAM=$PROGRAM_HEX"
fi

# ── SimX specific ──────────────────────────────────────────────────────────────
if [[ $USE_SIMX -eq 1 ]]; then
    # +SIMX enables simx_enable=1 in vortex_config.apply_plusargs()
    SIM_OPTS="$SIM_OPTS +SIMX"
    # +SIMX_DPI tells is_valid() not to require a binary path
    SIM_OPTS="$SIM_OPTS +SIMX_DPI"
    # +SIMX_PATH is informational; set to the .so for traceability
    SIM_OPTS="$SIM_OPTS +SIMX_PATH=$SIMX_LIB_SO"
    print_info "SimX golden model: ENABLED"
else
    SIM_OPTS="$SIM_OPTS +DISABLE_SIMX"
    print_info "SimX golden model: DISABLED"
fi

if [[ $NO_WAVES -eq 0 ]]; then
    WAVE_FILE="$RESULTS_RUN_DIR/waves/${TEST_NAME}_${MEMORY_INTERFACE}.vcd"
    SIM_OPTS="$SIM_OPTS +WAVE=$WAVE_FILE"
else
    SIM_OPTS="$SIM_OPTS +NO_WAVES"
fi

[[ $VERBOSE -eq 1 ]] && SIM_OPTS="$SIM_OPTS +VERBOSE"
```

---

### Step 6 — Update the vsim Call

```bash
################################################################################
# Simulation
################################################################################

LOG_FILE="$RESULTS_RUN_DIR/logs/simulation.log"

if [[ "$SIMULATOR" == "questa" ]]; then
    if [[ $GUI_MODE -eq 1 ]]; then
        vsim vortex_tb_top $SIM_OPTS $DPI_FLAG \
            -do "add wave -r /*; run -all"
    else
        vsim -c vortex_tb_top $SIM_OPTS $DPI_FLAG \
            -do "run -all; quit -f" \
            2>&1 | tee "$LOG_FILE"
    fi
fi
```

---

### Step 7 — Full Usage Examples

```bash
# Sanity test — no SimX
./scripts/run_vortex_uvm_enhanced.sh \
    --test=vortex_sanity_test

# Smoke test with AXI, no SimX
./scripts/run_vortex_uvm_enhanced.sh \
    --test=vortex_smoke_test \
    --program=rv32ui-p-add \
    --interface=axi \
    --cores=1 --warps=4 --threads=4

# Smoke test WITH SimX (builds .so if missing)
./scripts/run_vortex_uvm_enhanced.sh \
    --test=vortex_smoke_test \
    --program=rv32ui-p-add \
    --interface=axi \
    --simx

# Force rebuild SimX .so then run
./scripts/run_vortex_uvm_enhanced.sh \
    --test=vortex_smoke_test \
    --program=rv32ui-p-add \
    --interface=axi \
    --rebuild-simx

# Multi-core with SimX
./scripts/run_vortex_uvm_enhanced.sh \
    --test=vortex_smoke_test \
    --program=vecadd \
    --cores=2 --warps=4 --threads=4 \
    --interface=axi \
    --simx
```

---

### Step 8 — Critical Constraint: Architecture Must Match

The SimX DPI library is compiled with hardcoded `NUM_CORES`, `NUM_WARPS`, `NUM_THREADS`. These **must match** the RTL compile-time defines. The script already syncs `SIMX_CORES=$NUM_CORES` etc., and passes them to the Makefile via `ARCH_CONFIGS`. This ensures the `Arch(num_threads, num_warps, num_cores)` object created in `simx_init()` matches what the RTL was compiled with.

```
RTL compile:  +define+NUM_CORES=2 +define+NUM_WARPS=4 +define+NUM_THREADS=4
SimX .so:     -DNUM_CORES=2 -DNUM_WARPS=4 -DNUM_THREADS=4   ← must match
vsim plusarg: +NUM_CORES=2 +NUM_WARPS=4 +NUM_THREADS=4
```

If they diverge, `simx_init()` returns non-zero and the scoreboard disables itself gracefully with a `UVM_ERROR`.



## Prompt:
4/1/2026, 9:30:08 AM

please tell at every step where i must apply it specifically



## Response:
4/1/2026, 9:31:26 AM



````plaintext
Thought process: Recognized need for granular, location-specific implementation instructions.

The user wants me to specify exactly where each step should be applied - which file, which line/section, what to add/modify.
````

## Exact Location for Every Change

---

### Step 1 — `uvm_env/ref_model/Makefile`

**Location:** End of the file, after the `distclean` target (around line 195, after `rm -rf modelsim.ini`)

```makefile
# ── Add these lines at the very end of the file ──────────────────────────────

# Install target: copy .so to flists/ so the run script finds it
INSTALL_DIR  ?= $(shell cd ../..; pwd)/flists
PROJECT_ROOT ?= $(shell cd ../..; pwd)

.PHONY: install
install: $(DPI_LIB)
	@echo "=== Installing DPI lib to $(INSTALL_DIR) ==="
	mkdir -p $(INSTALL_DIR)
	cp $(DPI_LIB) $(INSTALL_DIR)/simx_model.so
	@echo "=== Done: $(INSTALL_DIR)/simx_model.so ==="

.PHONY: deploy
deploy: install
```

---

### Step 2 — `uvm_env/vortex_config.sv`

**Location A:** Inside `class vortex_config`, after the `simx_trace_file` field declaration (around line where `string simx_trace_file;` appears)

```systemverilog
    rand bit            simx_debug_enable;
    rand int unsigned   simx_timeout_cycles;
    string              simx_trace_file;
    // ── ADD THIS LINE ────────────────────────────────────────────────────────
    rand bit            simx_use_dpi;      // 1 = linked .so, 0 = exec binary
```

**Location B:** Inside `set_defaults_from_vx_config()`, after the line `simx_trace_file = "simx_trace.log";`

```systemverilog
        simx_trace_file    = "simx_trace.log";
        // ── ADD THIS LINE ────────────────────────────────────────────────────
        simx_use_dpi       = 1;   // default: DPI-C linked .so mode
```

**Location C:** Inside `apply_plusargs()`, after the block that handles `+DISABLE_SIMX`

```systemverilog
        if ($test$plusargs("DISABLE_SIMX") || $test$plusargs("disable_simx"))
            simx_enable = 0;

        // ── ADD THIS BLOCK ───────────────────────────────────────────────────
        if ($test$plusargs("SIMX_DPI") || $test$plusargs("simx_dpi"))
            simx_use_dpi = 1;
        // ─────────────────────────────────────────────────────────────────────
```

**Location D:** Inside `is_valid()`, find this exact existing block and replace it:

```systemverilog
        // FIND and REPLACE this existing block:
        if (simx_enable && simx_path == "") begin
            `uvm_error("VORTEX_CFG", "simx enabled but simx_path not set — pass +SIMX_PATH=<path>")
            valid = 0;
        end

        // REPLACE WITH:
        if (simx_enable && !simx_use_dpi && simx_path == "") begin
            `uvm_error("VORTEX_CFG",
                "simx enabled in exec mode but simx_path not set — pass +SIMX_PATH=<path>")
            valid = 0;
        end
```

---

### Step 3 — `scripts/run_vortex_uvm_enhanced.sh`

**Location A:** After the line `VERBOSE=0` in the Default Configuration section (around line 100)

```bash
# Simulation options
VERBOSE=0

# ── ADD THESE LINES immediately after VERBOSE=0 ──────────────────────────────
USE_SIMX=0
REBUILD_SIMX=0
# ─────────────────────────────────────────────────────────────────────────────
```

**Location B:** After the line that sets `SCRIPT_DIR` and `PROJECT_ROOT` at the top of the file (around line 60–65)

```bash
SCRIPT_DIR="$(cd "$(dirname "$SCRIPT_PATH")" && pwd)"
PROJECT_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"
FLISTS_DIR="$PROJECT_ROOT/flists"

# ── ADD THESE LINES immediately after FLISTS_DIR ─────────────────────────────
REF_MODEL_DIR="$PROJECT_ROOT/uvm_env/ref_model"
SIMX_LIB_NAME="simx_model"
SIMX_LIB_SO="$FLISTS_DIR/${SIMX_LIB_NAME}.so"
# ─────────────────────────────────────────────────────────────────────────────
```

**Location C:** Inside the `for arg in "$@"` loop, add two new cases. Find the line `--help|-h)      usage ;;` and add before it:

```bash
        --help|-h)      usage ;;
        # ── ADD THESE TWO CASES before the *) default case ───────────────────
        --simx)         USE_SIMX=1 ;;
        --rebuild-simx) REBUILD_SIMX=1; USE_SIMX=1 ;;
        # ─────────────────────────────────────────────────────────────────────
        *)
            print_error "Unknown option: $arg"
```

**Location D:** After the argument parsing `for` loop ends and before the `Validate Inputs` section. Find the line `if [[ -z "$TEST_NAME" ]]; then` and insert before it:

```bash
# ── ADD THIS BLOCK before "Validate Inputs" ───────────────────────────────────
# Sync SimX arch with requested arch (must happen after arg parsing)
SIMX_CORES=$NUM_CORES
SIMX_WARPS=$NUM_WARPS
SIMX_THREADS=$NUM_THREADS
# ─────────────────────────────────────────────────────────────────────────────

if [[ -z "$TEST_NAME" ]]; then
```

---

### Step 4 — `scripts/run_vortex_uvm_enhanced.sh` (SimX Build Function)

**Location:** After the `Environment Checks` section ends (after the line `print_success "Simulator: Questa/ModelSim"` or the VCS equivalent) and **before** the `Create Results Directory` section. Find the line `print_header "Setting Up Results Directory"` and insert the entire function before it:

```bash
# ── ADD THIS ENTIRE FUNCTION before "Setting Up Results Directory" ────────────

build_simx_dpi() {
    print_header "Building SimX DPI Shared Library"

    if [[ ! -d "$VORTEX_HOME/sim/simx/obj" ]]; then
        print_error "SimX objects not found. Build SimX first:"
        echo "  cd $VORTEX_HOME/sim/simx && make"
        exit 1
    fi

    cd "$REF_MODEL_DIR" || exit 1

    make build \
        VORTEX_HOME="$VORTEX_HOME"  \
        ARCH_CONFIGS="-DNUM_CLUSTERS=$NUM_CLUSTERS \
                      -DNUM_CORES=$SIMX_CORES       \
                      -DNUM_WARPS=$SIMX_WARPS        \
                      -DNUM_THREADS=$SIMX_THREADS"  \
        2>&1 | tee "$RESULTS_RUN_DIR/logs/simx_build.log"

    if [[ $? -ne 0 ]]; then
        print_error "SimX DPI build failed — see logs/simx_build.log"
        exit 1
    fi

    make install \
        INSTALL_DIR="$FLISTS_DIR" \
        PROJECT_ROOT="$PROJECT_ROOT" \
        2>&1 | tee -a "$RESULTS_RUN_DIR/logs/simx_build.log"

    if [[ ! -f "$SIMX_LIB_SO" ]]; then
        print_error "simx_model.so not found at $SIMX_LIB_SO after install"
        exit 1
    fi

    print_success "SimX DPI library ready: $SIMX_LIB_SO"
    cd "$FLISTS_DIR" || exit 1
}

if [[ $USE_SIMX -eq 1 ]]; then
    if [[ $REBUILD_SIMX -eq 1 || ! -f "$SIMX_LIB_SO" ]]; then
        build_simx_dpi
    else
        print_success "Reusing existing SimX DPI lib: $SIMX_LIB_SO"
    fi
fi

# ─────────────────────────────────────────────────────────────────────────────

print_header "Setting Up Results Directory"   # ← this line already exists
```

> **Note:** The `build_simx_dpi` function references `$RESULTS_RUN_DIR/logs/` which is created in the "Setting Up Results Directory" section. Because of this, move the Results Directory block **above** the SimX build call, or change the log path to `/tmp/simx_build.log` for the build step. The cleanest fix: move only the `mkdir -p` call for logs earlier:

```bash
# ── ADD just these two lines right before the build_simx_dpi function call ───
mkdir -p "$RESULTS_RUN_DIR/logs"
# ─────────────────────────────────────────────────────────────────────────────
if [[ $USE_SIMX -eq 1 ]]; then
```

---

### Step 5 — `scripts/run_vortex_uvm_enhanced.sh` (DPI Flag Block)

**Location:** Find the existing DPI_FLAG block that starts with `# ── FIX B ─` (around line 160–185 in the script). **Replace the entire existing block** with:

```bash
# ── REPLACE the entire existing DPI_FLAG block with this ─────────────────────

DPI_FLAG=""

# UVM DPI — always required
UVM_DPI_LIB="$QUESTA_HOME/uvm-1.2/linux_x86_64/uvm_dpi"
if [[ -f "${UVM_DPI_LIB}.so" ]]; then
    DPI_FLAG="$DPI_FLAG -sv_lib ${UVM_DPI_LIB}"
    print_success "UVM DPI loaded: ${UVM_DPI_LIB}.so"
else
    print_error "UVM DPI not found at ${UVM_DPI_LIB}.so — simulation WILL crash"
    exit 1
fi

# SimX DPI — only when --simx flag was passed
if [[ $USE_SIMX -eq 1 ]]; then
    if [[ -f "$SIMX_LIB_SO" ]]; then
        # vsim -sv_lib takes path WITHOUT the .so extension
        DPI_FLAG="$DPI_FLAG -sv_lib ${SIMX_LIB_SO%.so}"
        print_success "SimX DPI loaded: $SIMX_LIB_SO"
        # Ramulator shared lib must be on LD_LIBRARY_PATH at vsim runtime
        export LD_LIBRARY_PATH="$VORTEX_HOME/third_party/ramulator:${LD_LIBRARY_PATH:-}"
    else
        print_error "SimX .so missing: $SIMX_LIB_SO"
        print_error "Run with --rebuild-simx to build it"
        exit 1
    fi
fi

# Vortex DPI (optional trace functions) — keep existing behaviour
VORTEX_DPI_LIB="$FLISTS_DIR/vortex_dpi"
if [[ -f "${VORTEX_DPI_LIB}.so" ]]; then
    DPI_FLAG="$DPI_FLAG -sv_lib ${VORTEX_DPI_LIB}"
    print_success "Vortex DPI loaded: ${VORTEX_DPI_LIB}.so"
fi

# ─────────────────────────────────────────────────────────────────────────────
```

---

### Step 6 — `scripts/run_vortex_uvm_enhanced.sh` (SIM_OPTS Block)

**Location:** Find the existing `SIM_OPTS` section that starts with `SIM_OPTS="+UVM_TESTNAME=$TEST_NAME"`. After the existing `+VERBOSE` line at the end of that block, **add**:

```bash
# This line already exists:
if [[ $VERBOSE -eq 1 ]]; then
    SIM_OPTS="$SIM_OPTS +VERBOSE"
fi

# ── ADD THIS BLOCK immediately after the VERBOSE block ───────────────────────

# SimX plusargs
if [[ $USE_SIMX -eq 1 ]]; then
    SIM_OPTS="$SIM_OPTS +SIMX"
    SIM_OPTS="$SIM_OPTS +SIMX_DPI"
    SIM_OPTS="$SIM_OPTS +SIMX_PATH=$SIMX_LIB_SO"
    print_info "SimX golden model: ENABLED (DPI-C mode)"
else
    SIM_OPTS="$SIM_OPTS +DISABLE_SIMX"
    print_info "SimX golden model: DISABLED"
fi

# ─────────────────────────────────────────────────────────────────────────────
```

---

### Step 7 — `scripts/run_vortex_uvm_enhanced.sh` (vsim call — no change needed)

The existing vsim call already uses `$SIM_OPTS $DPI_FLAG` so **no change is needed** there. Just verify the existing line looks like:

```bash
vsim -c vortex_tb_top $SIM_OPTS $DPI_FLAG \
    -do "run -all; quit -f" \
    2>&1 | tee "$LOG_FILE"
```

If it does, you are done.

---

### Step 8 — `scripts/run_vortex_uvm_enhanced.sh` (`usage()` function)

**Location:** Inside the `usage()` function, after the existing `--no-tcu` line, add:

```bash
    --no-tcu                 Disable TCU

    # ── ADD THESE TWO LINES ──────────────────────────────────────────────────
    --simx                   Enable SimX golden model (builds .so if needed)
    --rebuild-simx           Force rebuild of simx_model.so then run with SimX
    # ─────────────────────────────────────────────────────────────────────────
    --help                   Show this help
```

---

### Summary Table

| Step | File | Where exactly |
|------|------|---------------|
| 1 | `uvm_env/ref_model/Makefile` | After last line (`distclean` target) |
| 2A | `uvm_env/vortex_config.sv` | After `string simx_trace_file;` field |
| 2B | `uvm_env/vortex_config.sv` | After `simx_trace_file = "simx_trace.log";` in `set_defaults` |
| 2C | `uvm_env/vortex_config.sv` | After `+DISABLE_SIMX` block in `apply_plusargs` |
| 2D | `uvm_env/vortex_config.sv` | Replace old `simx_path == ""` check in `is_valid` |
| 3A | `scripts/run_vortex_uvm_enhanced.sh` | After `VERBOSE=0` default |
| 3B | `scripts/run_vortex_uvm_enhanced.sh` | After `FLISTS_DIR=...` at top |
| 3C | `scripts/run_vortex_uvm_enhanced.sh` | Before `--help` in arg loop |
| 3D | `scripts/run_vortex_uvm_enhanced.sh` | Before first `if [[ -z "$TEST_NAME" ]]` |
| 4 | `scripts/run_vortex_uvm_enhanced.sh` | Before `print_header "Setting Up Results Directory"` |
| 5 | `scripts/run_vortex_uvm_enhanced.sh` | Replace entire old DPI_FLAG block |
| 6 | `scripts/run_vortex_uvm_enhanced.sh` | After `+VERBOSE` block in SIM_OPTS section |
| 7 | `scripts/run_vortex_uvm_enhanced.sh` | No change — vsim call already correct |
| 8 | `scripts/run_vortex_uvm_enhanced.sh` | Inside `usage()` after `--no-tcu` |



## Prompt:
4/1/2026, 2:06:29 PM

> File: 

i did all the 8 steps you said i must did to integrate simx inside uvm .
then i run this command :  ./scripts/run_vortex_uvm_enhanced.sh     --test=vortex_smoke_test     --program=$(pwd)/uvm_env/agents/host_agent/program_simple.hex     --clean
then that is the resulte , study it deeply and tell me if the simx totally integrated or not and if not tell me what is the problem.



## Response:
4/1/2026, 2:07:33 PM



````plaintext
Thought process: Analyzed simulation logs and diagnosed SimX integration incompleteness.

Let me analyze the simulation results carefully to determine if SimX is integrated or not.

Key indicators from the log:

1. **Scoreboard report:**
`​`​`
║  SimX Enabled       : NO                 ║
║  SimX Ran           : NO                 ║
`​`​`

2. **No `--simx` flag was passed** in the command:
`​`​`bash
./scripts/run_vortex_uvm_enhanced.sh \
    --test=vortex_smoke_test \
    --program=$(pwd)/uvm_env/agents/host_agent/program_simple.hex \
    --clean
`​`​`
The user did NOT pass `--simx` flag.


The scoreboard shows zero comparisons were performed since SimX wasn't enabled, and there's a warning from the DCR monitor indicating it didn't receive the startup configuration through the UVM path because the testbench writes DCRs directly instead.

The status monitor also flagged that execution started but never completed with an EBREAK signal through its clocking interface. Despite these warnings, the smoke test itself passed—it detected EBREAK and ran for 154 cycles with just a single instruction, suggesting this is an extremely minimal test program of only 64 bytes.

The reason SimX didn't run is straightforward: the command was executed without the `--simx` flag, so `USE_SIMX` was set to 0 and `+DISABLE_SIMX` was passed to the simulation.

To actually enable SimX, the user needs to rebuild the SimX `.so` library first, then run with the `--simx` flag. There's also a disconnect between how the TB_TOP signals execution completion and how the status monitor tracks it internally, which explains why the monitor sees no EBREAK while the test itself detects it. tracking.

Let me be more specific about the problems:

**Problem 1: SimX not enabled** - The command didn't include `--simx`
**Problem 2: Even if `--simx` was added, the simx_model.so needs to be built first**

The test results show:
- TEST PASSED (the basic smoke test works)
- SimX: NOT enabled/ran

So the integration steps were applied correctly to the script/config files, but SimX was never actually invoked because:
1. `--simx` flag was not passed in the command
2. The `simx_model.so` may not have been built yet

The program being tested is just a minimal 64-byte hex file, which explains the extremely low IPC of 0.006 and why most of the 154 cycles are spent in reset overhead. To properly verify the SimX integration, the user needs to build the `.so` file first, then run the test with the `--simx` flag and check that the scoreboard output shows both SimX enabled and executed.
````

## SimX Integration Status: NOT Integrated Yet

The test passed but SimX was **never invoked**. Here is the proof from your log:

```
║  SimX Enabled       : NO                 ║
║  SimX Ran           : NO                 ║
║  Comparisons        : 0                  ║
║  Passed             : 0                  ║
```

---

## Root Cause Analysis

### Problem 1 — You Did Not Pass `--simx` Flag (Primary Cause)

Your command was:

```bash
./scripts/run_vortex_uvm_enhanced.sh \
    --test=vortex_smoke_test \
    --program=$(pwd)/uvm_env/agents/host_agent/program_simple.hex \
    --clean
```

No `--simx` flag → script sets `USE_SIMX=0` → passes `+DISABLE_SIMX` to vsim → `vortex_config.apply_plusargs()` sets `simx_enable=0` → scoreboard skips all SimX calls.

---

### Problem 2 — `simx_model.so` Was Never Built

Before you can use `--simx`, the shared library must exist. Check:

```bash
ls -lh /home/stev_teto_22/vortex_uvm/flists/simx_model.so
```

If that file does not exist, SimX was never built. You need to build it first.

---

### Problem 3 — Program Is Too Trivial to Verify Anything (Secondary)

```
Program:  64 bytes
Instructions: 1
Memory Reads: 1 (AXI AR)
IPC: 0.006
```

`program_simple.hex` contains essentially just one instruction (likely `ebreak`). Even when SimX runs, there is nothing meaningful to compare because the RTL and SimX will both trivially agree on 1 instruction. You need a real program like `rv32ui-p-add` to validate SimX comparison logic.

---

### Problem 4 — Two False Warnings That Will Persist (Not Blocking)

```
UVM_WARNING DCR_MON: Test ended without startup configuration
UVM_WARNING STATUS_MON: Execution started but did not complete (no EBREAK)
```

These are expected because:

- **DCR_MON warning**: TB_TOP writes DCRs directly to the interface, bypassing the UVM DCR agent's monitor clocking block. The monitor never sees the writes. This is by design (TB_TOP DCR edition) and is not a real error.
- **STATUS_MON warning**: The status monitor tracks EBREAK through `vif.status_if.ebreak_detected` via its clocking block, but `ebreak_detected` is driven combinationally only after `tb_execution_complete && axi_channels_idle`. The monitor sees `busy` went high but the combinational ebreak signal was not sampled before the test ended. Again not blocking.

---

## What You Must Do Now — Step by Step

### Step A — Build SimX First

```bash
cd /home/stev_teto_22/vortex_uvm/uvm_env/ref_model

# Verify SimX objects exist
ls $VORTEX_HOME/sim/simx/obj/*.o | head -5

# Build the .so
make build \
    VORTEX_HOME=$VORTEX_HOME \
    ARCH_CONFIGS="-DNUM_CLUSTERS=1 -DNUM_CORES=1 -DNUM_WARPS=4 -DNUM_THREADS=4"

# Install to flists/
make install \
    INSTALL_DIR=/home/stev_teto_22/vortex_uvm/flists

# Verify it exists
ls -lh /home/stev_teto_22/vortex_uvm/flists/simx_model.so
```

Expected output:
```
-rwxr-xr-x 1 stev_teto_22 ... 45M Apr 01 12:50 simx_model.so
```

### Step B — Run With a Real Program and `--simx`

```bash
cd /home/stev_teto_22/vortex_uvm

./scripts/run_vortex_uvm_enhanced.sh \
    --test=vortex_smoke_test \
    --program=rv32ui-p-add \
    --interface=axi \
    --cores=1 --warps=4 --threads=4 \
    --simx
```

### Step C — What a Successful SimX Run Looks Like

If SimX is correctly integrated, your scoreboard output must show:

```
║  SimX Enabled       : YES                ║   ← must be YES
║  SimX Ran           : YES                ║   ← must be YES
║  Comparisons        : N                  ║   ← N > 0
║  Passed             : N                  ║   ← equals Comparisons
║  Failed             : 0                  ║   ← must be 0
```

And in the log you will see SimX DPI calls:

```
[SimX-DPI] Initializing SimX Golden Model
[SimX-DPI] Cores=1, Warps=4, Threads=4
[SimX-DPI] RAM created successfully
[SimX-DPI] RAM attached successfully
[SimX-DPI] Initialization successful
[SimX-DPI] Running processor to completion...
[SimX-DPI] Execution finished
[SimX-DPI] Exit code: 0
```

---

## Diagnostic Checklist

Run these checks before retrying:

```bash
# 1. SimX source objects exist?
ls $VORTEX_HOME/sim/simx/obj/*.o 2>/dev/null | wc -l
# Must be > 0, otherwise: cd $VORTEX_HOME/sim/simx && make

# 2. Ramulator built?
ls $VORTEX_HOME/third_party/ramulator/libramulator.* 2>/dev/null
# Must show .a or .so file

# 3. .so built and installed?
ls -lh /home/stev_teto_22/vortex_uvm/flists/simx_model.so 2>/dev/null
# Must exist

# 4. .so has the required symbols?
nm -D /home/stev_teto_22/vortex_uvm/flists/simx_model.so | grep simx_init
# Must show: T simx_init

# 5. No missing shared library dependencies?
ldd /home/stev_teto_22/vortex_uvm/flists/simx_model.so | grep "not found"
# Must show nothing (empty output = good)
```

Run Step A first, share the output of the `nm` and `ldd` commands, and then run Step B. Only then can we confirm SimX is truly integrated.





---
Powered by [Claude Exporter](https://www.ai-chat-exporter.net)