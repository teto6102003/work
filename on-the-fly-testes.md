# on-the-fly-testes

**Created:** 3/30/2026 17:08:14  
**Updated:** 3/30/2026 17:08:58  
**Exported:** 5/13/2026 16:12:29  
**Link:** [https://claude.ai/chat/f23ff818-9140-4ee0-96e5-1e06dde9bc10](https://claude.ai/chat/f23ff818-9140-4ee0-96e5-1e06dde9bc10)  

## Prompt:
3/30/2026, 5:08:18 PM

act as expert in Digital ic design and verification specially in UVM and GPGPUs like vortex from this repo :"https://github.com/vortexgpgpu/vortex" study it deeply and all info. you can find in reserch papers about that. we are a team working to build a full UVM for vortex , we work on linux ubuntu 2022 and questasim 2021 for linux , and I'm resposable for the Golden model part , we choose the 'simx' as a golden model which is inside vortex repo , but it (simx) was written by C++ and rtl for vortex is systemverilog and uvm is systemverilog too, i searched in this part and find thing called DPI-c used in these cases ,which is a MakeFile compile all Simx files + third-party files (in C++ too) and DPI-c file (C++) + DPI-c file which is simx_dpi.cpp  .
 and now i am trying some on-the-fly testes on simx to show what happened .
now i tried the alu otf test only  , and need to apply the other testes (hex tests) {pipleine , branch , simt , fpu , isu , mwarp , mext } , i wonder if for every one i need to make .sv file custmized for that test or not



## Response:
3/30/2026, 5:08:58 PM



````plaintext
Thought process: Analyzed testbench reusability across multiple hex test files.

The user is asking whether they need a separate .sv testbench file for each hex test, or if they can reuse a generic one. Let me think about this carefully.

Looking at the existing test files and the hex files provided, the user has:
- `test_otf_full.sv` - a generic on-the-fly testbench that uses `+HEX=` and `+RESULT_ADDR=` plusargs
- Various hex files: test_alu.hex, test_branch.hex, test_fpu.hex, test_lsu.hex, test_mext.hex, test_mwarp.hex, test_pipeline.hex, test_simt.hex


- The testbench already accepts plusargs like `+HEX=`, `+RESULT_ADDR=`, `+STEP=`, `+TIMEOUT=`, so it's designed to be reusable across different tests. Each hex file stores results at different memory addresses, so I just need to invoke the same testbench with different plusargs for each test rather than creating separate files. I see each test file maps to a specific memory address through the `lui` instruction, so they can all run with the same testbench by just swapping the hex file and result address parameters. I should document this pattern and provide a Makefile or script to automate running all the tests with their respective configurations.
````

## You Don't Need Separate `.sv` Files

Your existing `test_otf_full.sv` is already generic via plusargs. You can run **all hex tests** with it by just changing `+HEX=` and `+RESULT_ADDR=`.

### Result Addresses from the Hex Files

Decoding the first `lui` instruction in each hex file reveals where results are stored:

| Test | First Instruction | Result Address |
|---|---|---|
| `test_alu.hex` | `80010A37` → `lui x20, 0x80010` | `0x80010000` |
| `test_lsu.hex` | `80020A37` → `lui x20, 0x80020` | `0x80020000` |
| `test_mext.hex` | `80030A37` → `lui x20, 0x80030` | `0x80030000` |
| `test_fpu.hex` | `80040A37` → `lui x20, 0x80040` | `0x80040000` |
| `test_simt.hex` | `80050A37` → `lui x20, 0x80050` | `0x80050000` |
| `test_mwarp.hex` | `80060A37` → `lui x20, 0x80060` | `0x80060000` |
| `test_branch.hex` | `80070A37` → `lui x20, 0x80070` | `0x80070000` |
| `test_pipeline.hex` | `80080A37` → `lui x20, 0x80080` | `0x80080000` |

### Add to Your Makefile

```makefile
# ── all on-the-fly hex tests ─────────────────────────────────────────────────
HEX_TESTS := alu lsu mext fpu simt mwarp branch pipeline

# Map test name → result address (must match the hex file's lui x20, ...)
RESULT_ADDR_alu      := 0x80010000
RESULT_ADDR_lsu      := 0x80020000
RESULT_ADDR_mext     := 0x80030000
RESULT_ADDR_fpu      := 0x80040000
RESULT_ADDR_simt     := 0x80050000
RESULT_ADDR_mwarp    := 0x80060000
RESULT_ADDR_branch   := 0x80070000
RESULT_ADDR_pipeline := 0x80080000

# Watch words: how many 32-bit result words each test writes
WATCH_WORDS_alu      := 20
WATCH_WORDS_lsu      := 16
WATCH_WORDS_mext     := 16
WATCH_WORDS_fpu      := 16
WATCH_WORDS_simt     := 8
WATCH_WORDS_mwarp    := 8
WATCH_WORDS_branch   := 12
WATCH_WORDS_pipeline := 10

# Generic rule — run any test:  make otf_alu  /  make otf_fpu  / etc.
otf_%: $(DPI_LIB)
	@echo "=== OTF test: $* ============================================"
	$(VLIB) work
	$(VLOG) -sv $(VLOG_UVM_FLAGS) \
	        $(RTL_INCLUDE) $(SV_SOURCES) $(SV_PKG) \
	        test_otf_full.sv
	$(VSIM) $(VSIM_FLAGS) test_otf_full \
	        +HEX=$(CURDIR)/test_$*.hex \
	        +RESULT_ADDR=$(RESULT_ADDR_$*) \
	        +WATCH_WORDS=$(WATCH_WORDS_$*) \
	        +STEP=$(STEP_CYCLES) \
	        +TIMEOUT=$(TIMEOUT) \
	        -do "run -all; quit" \
	        | tee logs/otf_$*.log
	@echo "=== Done: $* exit=$$? ======================================="

# Run all hex tests in sequence
otf_all: $(DPI_LIB)
	@mkdir -p logs
	@for t in $(HEX_TESTS); do \
	    $(MAKE) otf_$$t || echo "FAILED: $$t"; \
	done
	@echo "=== All OTF tests complete ==================================="

.PHONY: otf_all $(addprefix otf_,$(HEX_TESTS))
```

### Usage

```bash
# Single test
make otf_alu
make otf_fpu
make otf_branch

# All tests at once
make otf_all

# With custom step size / timeout
make otf_pipeline STEP_CYCLES=5 TIMEOUT=200000
```

### One Small Fix Needed in `test_otf_full.sv`

The `simx_load_hex` DPI function in `simx_dpi.cpp` reads the `@address` line and also calls `dcr_write` at the end — so you don't need to call `simx_dcr_write` separately for hex files. But your testbench currently calls it before loading. Change the hex test flow to:

```systemverilog
// Step 3: Load hex (this sets startup addr internally)
if (simx_load_hex(hex_file) != 0) begin ... end

// Step 2.5: DCR is already set inside simx_load_hex → just confirm:
simx_dcr_write(32'h001, startup_addr[31:0]);
```

This order is fine as-is since `simx_load_hex` overwrites the DCR anyway. No `.sv` changes needed.





---
Powered by [Claude Exporter](https://www.ai-chat-exporter.net)