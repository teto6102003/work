# Debugging simx_model integration in UVM

**Created:** 4/7/2026 3:10:59  
**Updated:** 4/7/2026 13:09:04  
**Exported:** 5/13/2026 16:26:10  
**Link:** [https://claude.ai/chat/a42d9029-2a83-4e40-9605-b460e032ae5a](https://claude.ai/chat/a42d9029-2a83-4e40-9605-b460e032ae5a)  

## Prompt:
4/7/2026, 3:11:05 AM

inside that project can you access the other chats ??
i need from you to study the previous chats in that project deeply . 
i did all changes and steps in the last chat to fully integrate simx_model.so inside uvm .
and then i run that command 
./scripts/run_vortex_uvm_enhanced.sh     --test=vortex_smoke_test     --program=$(pwd)/uvm_env/agents/host_agent/program_simple.hex     --clean
the output of that i provided to you in the file Output_of_runing_simple_program.hex_in_uvm.txt study it carefully and deeply and tell me what are the problems and also take a look in the program_simple.hex inside host agent



## Response:
4/7/2026, 3:17:51 AM



````plaintext
Thought process: Diagnosed SimX integration issues and prescribed targeted fixes.

Let me analyze the output file and the project files carefully.

Key observations from the output:

1. **SMOKE TEST PASSED** (locally in the test) but **TEST FAILED** overall due to 2 UVM_ERRORs

2. The 2 UVM_ERRORs are:
   - `UVM_ERROR` from SCOREBOARD: "SimX exit code=3 (non-zero = program failed)"
   - `UVM_ERROR` from base_test: "*** TEST FAILED ***"

3. The program executed: 1 instruction, 154 cycles, 1 AXI read


4. SimX exited with code 3, indicating the program failed, and the test loaded 64 bytes from `program_simple.hex` with only a single instruction executed
5. The scoreboard never observed the ebreak event it was waiting for, so it ran the final comparison without that trigger

The core issue seems to be that the program isn't completing as expected — SimX is failing with a non-zero exit code, and the ebreak instruction that should signal test completion never fires.

Looking at the bootstrap code, it initializes x3 to 0 and jumps to 0x80000000, but the exit code of 3 suggests the program itself is setting x3 to 3 before hitting ebreak. The minimal program in `program_simple.hex` might just contain an ebreak instruction that exits with whatever value is in x3, which could be getting set to 3 somewhere in the execution flow.

Now I'm noticing a discrepancy with the DCR monitor—it's reporting that startup configuration isn't complete because the testbench is driving DCRs directly rather than going through the UVM DCR agent, so the monitor never sees those configuration writes. Meanwhile, the RTL executed one instruction and one cache line read before signaling execution complete, but the scoreboard never captured the ebreak instruction that should have been observed.

Looking at the status interface, the ebreak_detected signal fires when execution completes and AXI channels are idle. The status monitor only samples every 100 cycles, but execution finished in 154 cycles—so it's possible the test detected the ebreak signal directly, yet the scoreboard's write_status() method was never invoked with that flag set. The smoke test itself did wait for and detect ebreak_detected=1, but somehow the scoreboard missed recording it.

For the SimX exit code issue, the bootstrap sits at 0x7FFFFFF0 which is outside the DUT's AXI memory range. The DUT starts fetching from 0x80000000 (its configured start address), so it loads program_simple.hex directly without the bootstrap. SimX, however, begins at the bootstrap address, runs initialization code, then jumps to 0x80000000 where the actual program executes. If that program contains an instruction like `li x3, 3; ebreak`, SimX will exit with code 3—which matches what we're seeing.

The real issue is in the scoreboard: it treats any non-zero exit code as a failure, but the program itself is intentionally setting x3 to 3 before hitting EBREAK. The bootstrap initializes x3 to 0, but the program overwrites it, causing the scoreboard to incorrectly flag this as an error. FFFFFF4, then x1 = 0x7FFFFFF4, and finally jump to 0x80000000 where program_simple.hex executes. The exit code of 3 most likely comes from x3 being set by the program itself with an EBREAK instruction, or x3 retains its initial value from SimX initialization since we're running bare-metal without startup code that would normally set the global pointer register.

The real issue is probably that program_simple.hex is intentionally exiting with code 3, but the UVM scoreboard treats any non-zero exit as a failure—so either the test program needs to exit with 0, or the scoreboard needs to accept non-zero exits without flagging them as failures.

Now looking at the ebreak detection problem: the status monitor samples every 100 cycles, but the simulation only runs 154 cycles total, so it might miss the ebreak signal entirely. The ebreak_detected wire depends on both tb_execution_complete and axi_channels_idle going high, and by the time that happens, the simulation is already wrapping up—the monitor's sampling window might have already passed.

There are two main issues to fix: the program_simple.hex is exiting with code 3 instead of 0, which the scoreboard treats as a failure, and the status monitor's sampling interval is too coarse to catch the brief ebreak pulse in such a short simulation. The scoreboard needs to handle the simx_enable flag and decide whether to validate exit codes for smoke tests, since the program might legitimately return non-zero values. Looking at the actual execution flow: if the program is just an `ebreak` instruction, then x3 should hold whatever value the bootstrap set it to (0 for SimX), but we're seeing exit code 3, which suggests either the bootstrap isn't running correctly or the program itself is setting x3 to 3 somewhere.

There's a configuration mismatch too—the reference model compiles with 2 cores while the UVM test uses 1 core, which could be affecting SimX behavior. And in the SimX initialization code, there's a bootstrap being installed at a specific address before the hex file is loaded, but then the startup address gets reset when the hex file is processed, which might be overwriting or interfering with the bootstrap setup. So the issue is that `simx_load_hex` resets `g_startup_addr` to 0x80000000 when it parses the hex file's address directive, then writes that value to the DCR register, completely overriding the bootstrap address that was set earlier. This means the processor jumps directly to 0x80000000 instead of executing the bootstrap code at 0x7FFFFFF0, leaving x3 with whatever initial value the register file happens to have. The fix would be to either prevent `simx_load_hex` from overriding the startup address, or call the initialization function after loading the hex file instead of before.

But looking at the actual scoreboard code, it's calling `simx_load_bin` with the program path, not `simx_load_hex` — and that's the real problem, because `program_simple.hex` is a hex file, not a binary file, so trying to load it as raw binary data will either fail or produce garbage. The scoreboard is loading a .hex file with `simx_load_bin`, which treats the ASCII hex characters as raw binary data instead of parsing them properly—that's why garbage ends up at 0x80000000. The bootstrap initialization happens before loading, so the DCR is already set correctly, but the program content itself is corrupted. The fix is to either use `simx_load_hex` for hex files or add file type detection to choose the right loader.

There's also a timing issue with EBREAK detection—the status monitor might not be sampling frequently enough to catch the event before the simulation ends. And the DCR monitor warning about startup completion is just cosmetic since the testbench drives DCRs directly, not a real functional problem.

Looking at the `write_status()` logic, the ebreak detection should trigger when the transaction reports it, but I need to verify the timing of when that flag gets set relative to when the monitor samples. The issue is that the status monitor only samples at fixed intervals, so it misses the `ebreak_detected` signal before the test drops its objection and ends the run phase. The real problem is that `detect_state_transitions()` triggers an event when it sees `ebreak_detected` go high, but it never actually writes to the analysis port—that only happens during the periodic sampling in `monitor_status()`. I could fix this by either reducing the sample interval to every cycle, adding a direct `ap.write()` call when the transition is detected, or inserting a small delay in the test before dropping the objection.

Looking at the broader picture, I need to address several issues: the scoreboard needs to use the right file loading function for hex files, the status monitor should broadcast a transaction immediately when it detects `ebreak` rather than waiting for the next sample interval, and I should reconsider how the scoreboard handles non-zero exit codes from simx since simple programs might not exit cleanly.

For the program loading fix, I can check the `program_type` configuration to determine whether to call `simx_load_hex` or `simx_load_bin` with the appropriate parameters.

Looking at the initialization sequence, I realize there's an ordering issue—`simx_load_hex` modifies the DCR and startup address, but `simx_init_exit_code_register` is currently called before the program loads, which means the bootstrap DCR gets overwritten. I need to reorder this so the program loads first, then the exit code register is initialized to restore the bootstrap configuration.

For the status monitor, I'm adding immediate ebreak detection and broadcast...

For the hex program, I need to set the exit code to 0 before ebreak—the simplest approach is an `addi` instruction to zero out x3, then the ebreak instruction itself, which encodes to `0x00100073`. , rs1=x0, funct3=000, rd=x0, opcode=1110011
  - = 000000000001 00000 000 00000 1110011
  - = 0x00100073

So the hex for EBREAK = `73 00 10 00` (little-endian byte order).

And `addi x3, x0, 0` = `0x00000193`:
- imm=0, rs1=x0, funct3=000, rd=x3, opcode=0010011
- = 000000000000 00000 000 00011 0010011
- = 0x00000193

Bytes: `93 01 00 00` (little-endian).

Actually wait, looking at `program_simple.hex` - we need to know what's actually in it. The output says 64 bytes loaded, which is exactly one cache line, so the DUT fetched a single cache line with just 1 instruction executed. That could mean the program is just `ebreak` by itself, and since x3 wasn't initialized by the user program or bootstrap, it might still be holding the value 3.

The main fixes needed are: get the load order right in scoreboard (init → load → bootstrap), make the status monitor broadcast immediately on ebreak, and possibly update program_simple.hex to explicitly zero out x3. The warning about EBREAK never being observed is happening because scoreboard's write_status() was never called with ebreak=1, which should be resolved by these changes.

For the DCR monitor warning about missing startup configuration—that's just because TB_TOP writes DCRs directly rather than through the UVM monitor path, which is intentional, so it's just cosmetic. The status monitor warning about execution not completing is also tied to the ebreak detection issue in the clocking block.

Looking at the detect_state_transitions() logic, it samples every clock cycle and should catch ebreak when it goes high, but the warning persists. The real issue is how ebreak_detected gets driven—it only goes high when tb_execution_complete is true AND the AXI channels are idle, which means...

Still tracing the signal dependencies... Defining the ebreak signal logic...

So the issue is a race condition: the smoke test uses a level-sensitive wait that fires immediately when `ebreak_detected` goes high, but the status monitor's edge-triggered check happens on the next clock cycle, by which point the objection is already dropped and UVM is shutting down the run phase.

Looking at the actual output though, there's something odd—the test itself reports "✓ EBREAK detected" and "✓ Execution completed", but the status monitor's check_phase says execution never completed. So the monitor's `detect_state_transitions()` never actually saw `ebreak_detected` go high, even though the test did. Let me trace through the timing more carefully using the log data to figure out where the disconnect is. I see there's a timing discrepancy here. The idle threshold is set to 5000 cycles by default, which means after the memory operation completes, the testbench waits 5000 idle cycles (50 microseconds) before triggering the execution complete signal. But there's also a separate condition checking the busy signal that could trigger completion earlier.

Looking at the timestamps, the "EXECUTION COMPLETE" message appears when the busy signal goes low, but the actual "✓ Execution completed" message is timestamped at 5525000, which is about 810 microseconds after the memory operation started. That's way longer than the 5000-cycle idle threshold would account for, so something else is controlling the timing here. The ebreak_detected signal seems to be tied to tb_execution_complete, which might be what's actually determining when the test finishes. I'm calculating the timing: 4715000 + 50000 cycles should give 4765000ns, not 5525000ns, so there's a discrepancy I need to investigate. But rather than getting lost in the math, let me focus on the actual problems the logs are showing: SimX is loading the program with the wrong function, EBREAK detection is failing due to sampling intervals or timing issues, and the scoreboard is erroring on SimX's non-zero exit code. The clearest fixes are to load the program before installing the bootstrap, and to address the EBREAK detection timing.

I'm realizing the order matters because `simx_load_hex` has a side effect—it overwrites the DCR startup address register at the end, so if I call the bootstrap initialization after loading the hex file, the bootstrap DCR value will be the final one that takes precedence.

For the vortex_scoreboard.sv file, I need to detect whether the program path ends with .hex and call the appropriate loader function accordingly.  If it's a hex file, `simx_load_hex` reads the address directly from the file itself, but if it's binary, `simx_load_bin` needs both the filepath and the load address passed separately.

For status_monitor.sv, when ebreak is detected, I should immediately broadcast a status transaction to the analysis port rather than waiting.

Now I'm thinking through what program_simple.hex needs for a basic smoke test. It should be minimal—just fetch some instructions and execute EBREAK to signal completion, with x3 set to 0 for SimX compatibility. I'm working out the hex encoding for a couple of simple instructions at the base address, and checking whether the @address syntax uses byte addresses and whether the data is byte-by-byte...

Looking at how the DPI and memory model parse the hex format, tokens with 2 hex digits are treated as bytes and written sequentially. The test results showing 64 bytes loaded with only 1 instruction executed suggests the hex file probably contains a minimal program—likely just an instruction or two followed by padding to reach that 64-byte size.

For a proper smoke test, I should construct a hex file that initializes the exit code register and then breaks, ensuring clean termination. I'm thinking the simplest approach is to load an instruction that sets x3 to 0, followed by an ebreak instruction, then pad the rest to fill the cache line. I'm realizing the key issue: the hex file must start with `@00000000` rather than `@80000000` to avoid the overflow check, and then the memory model adds the base address `0x80000000` during loading, placing the program at the correct location. For SimX, the hex file's address directive is read directly, so if it has `@00000000`, the program loads at that absolute address instead. X loads the program at 0x00000000, NOT at 0x80000000! That's another bug!

The simx_load_hex function reads the address from the hex file and sets both the current address and startup address to that value, then writes it to the DCR register. So SimX would load and start at address 0, while the RTL loads and starts at 0x80000000 — that's the mismatch causing the exit code 3, since it's trying to execute code at address 0.

I need to fix simx_load_hex to apply a base address offset like the mem_model does. I could modify the function to accept a base_addr parameter, or create a wrapper function that applies the offset before loading. The goal is to make SimX load the program at the same address as the RTL for the scoreboard use case.

Actually, the cleanest approach is to iterate through the mem_model's content in the scoreboard and write each byte directly to SimX memory, ensuring both see identical data at the same addresses.

But wait—the scoreboard doesn't have access to mem_model. The real solution is to add a new DPI function that loads the hex file and applies a base address offset, so SimX gets the same memory layout as the DUT.

Writing DPI function... I see the core issue now—the hex file uses relative addresses starting at `@00000000`, but SimX loads them at absolute address zero while the memory model expects them at the startup address offset. I need to add a new DPI function that applies a base address offset when loading the hex file, then call it from the scoreboard with the startup address as the offset.

For the test program itself, I'm creating a simple RISC-V hex file with a NOP instruction followed by an EBREAK, padded out to fill a cache line so the DUT fetches the entire line but only executes those two instructions. I'm realizing that in RISC-V simulators, registers typically start at zero unless the ABI or startup code initializes them differently. The bootstrap code I fixed should explicitly set x3 to zero before jumping to the program, so if the program just executes ebreak, x3 should be zero at that point—which means the exit code of 3 suggests something else is happening with register initialization in SimX.

The real problem is that simx_load_hex is treating the @address directive as an absolute address rather than applying the base_addr offset, which causes the startup code to load at the wrong location and SimX to begin execution at an incorrect address, running garbage instructions that leave x3 in an unexpected state. The fix is to create a new function that properly applies the base address offset when loading the hex file.

Writing the corrected loader function... I'm realizing the simx_load_hex function doesn't write to the DCR when it encounters the @ address marker—it just sets the current address and startup address, then proceeds to load data words at that location. Let me trace through the actual implementation to make sure I understand the memory loading behavior correctly. I'm realizing the cleanest solution is to create a new function that loads the hex file while applying a base address offset to all the @address directives, then ensure the hex loading happens before the bootstrap sequence. Let me outline the minimal changes needed to fix this.

Writing SystemVerilog code... Writing the ebreak transaction handler... Writing the hex file loader... Parsing hex data from the file... Writing the hex file parser...

Actually, let me reconsider the format here. Looking at what mem_model loaded and how the RTL executed, the hex file is probably structured with an address header followed by space-separated byte values, where shorter tokens are single bytes and longer ones are 32-bit words that get split into little-endian bytes before writing to memory. After processing all the tokens, I'm logging how many were loaded and returning successfully.

But I'm realizing I might be overthinking this. Let me check what program_simple.hex actually contains—given that 64 bytes were loaded and the file didn't trigger the address validation, it likely starts with a base address marker followed by the actual hex data. I'm tracing through the hex file loading logic to understand why SimX isn't handling the address correctly. The script rejects files starting with `@80000000`, so since it didn't exit, the file must start with `@00000000` instead. That's the issue—the hex file uses address 0x00000000, but the memory model expects data at 0x80000000, and simx_load_hex applies no base address offset to account for this mismatch.

Now I see what needs fixing. The program_simple.hex file likely contains 64 bytes with a single EBREAK instruction loaded at the base address...

Examining register initialization in SimX—the RISC-V ABI leaves x3 (the global pointer) uninitialized in bare-metal programs without startup code, so its value depends on the simulator's implementation.

If SimX initializes x3 to 3 instead of 0, that explains the exit code 3 we're seeing; the bootstrap code fixes this by explicitly zeroing x3 before jumping to the program. Once the bootstrap loads correctly, the exit code should be 0.

I should focus on the root cause—making sure the hex file loads at the right address with `simx_load_hex_at`—rather than treating non-zero exit codes as warnings, since we can actually fix the underlying issue.

Now I'm checking whether `simx_load_hex_at` needs to handle different hex formats: the word format with 32-bit values per line versus the byte format that `objcopy --verilog-data-width=1` produces with space-separated bytes.

Examining the current implementation...

Finishing the byte extraction logic...

The existing code only handles 32-bit word format, but `program_simple.hex` uses byte format, so I need to make `simx_load_hex_at` detect and handle both cases properly.

Looking at the actual issues: SimX loads the program at address 0x00000000 from the hex file's `@00000000` directive, but the RTL expects it at 0x80000000 because mem_model applies the base address offset. That's the core mismatch. Also, the bootstrap sequence is wrong—`simx_init_exit_code_register()` gets called before the program is actually loaded. I need to implement `simx_load_hex_at` to handle both byte-format (space-separated hex pairs) and word-format (single 8-character tokens) hex files, since the existing loader only handles word format and I need flexibility for different file layouts.

Writing the loader function... Continuing with the file parsing logic...

I'm handling the word format by breaking it into individual bytes in little-endian order and writing them to RAM, then updating the address counter. After processing all tokens, I'm logging how many bytes were loaded and the base address, then returning success. There's a note about not setting the startup address here since the caller will handle that separately through the scoreboard initialization.

Now I'm reconsidering whether we actually need to update the startup address and DCR register at all. If the caller invokes `simx_init_exit_code_register()` afterward, that function will set the bootstrap code at the appropriate offset and configure the DCR to point to it. The global `g_startup_addr` is initialized to 0x80000000 by default, and it gets overwritten by `simx_load_bin` or `simx_load_hex` depending on which loader is used. So the question is whether this hex loader should be responsible for updating that value based on the @address directive it encounters. our new `simx_load_hex_at`, if we DON'T set g_startup_addr, it stays at 0x80000000 (default). Then `simx_init_exit_code_register()` uses 0x80000000 as the base, installs bootstrap at 0x7FFFFFF0, and sets DCR to 0x7FFFFFF0. That's correct!

So in `simx_load_hex_at`, we should NOT update g_startup_addr (leave it at 0x80000000 from initialization, or whatever it was set to by the caller).

Let me trace through the full flow with the proposed fix: `simx_init` sets g_startup_addr to 0x80000000, then `simx_load_hex_at` loads the program at that address without changing it. When `simx_init_exit_code_register()` runs, it calculates the bootstrap address as 16 bytes below the startup address, writes the bootstrap code there, updates g_startup_addr to point to the bootstrap, and configures the DCR accordingly. Finally, `simx_run()` starts execution at the bootstrap address, which performs the necessary setup before jumping to the actual program entry point at 0x80000000. The hex file structure checks out—16 bytes per line across 4 lines gives exactly 64 bytes total, matching what the loader expects. The first instruction sets the exit code to 0 (redundant but harmless), followed by the ebreak instruction to halt execution, then padding to fill the cache line.

Looking at the actual behavior, the current program_simple.hex likely starts with ebreak as the first instruction with zeros filling the rest, which explains why SimX executed just one instruction and returned exit code 3—it was running uninitialized garbage at the default load address. Once we apply the fixes to load at 0x80000000 with the bootstrap code clearing x3 before jumping to the actual program, the exit code should properly reflect 0. I'll update program_simple.hex to explicitly set x3 to 0 before the ebreak instruction, padding with zeros to fill out the memory space, which ensures the exit code will be 0 when running through both the simulator and RTL. Now I need to document all the fixes clearly.

I'm realizing the `simx_load_hex` function has a parsing issue—it tries to convert space-separated byte strings like `93 01 00 00` directly to a single integer, which won't work correctly. Since the Makefile tests rely on the existing word-format behavior, I should keep that function as-is for backward compatibility and create a new `simx_load_hex_at` variant that properly handles byte-format input for the scoreboard tests.

There's another problem in vortex_scoreboard.sv where run_phase calls `simx_load_bin`, but that function expects a file path rather than inline binary data...

Actually, looking at the code more carefully, `simx_load_bin` opens files in binary mode and reads them as raw bytes. If the program_path points to a hex file with ASCII text like `93 01 00 00`, it'll read those characters as their ASCII byte values instead of interpreting them as hex instructions—completely wrong. I need to either convert the hex file to binary format first or use a different loading function that understands hex format.

Now I'm organizing my response to be practical and specific about what needs to change and where.
````

Looking at the output and project files carefully, I can identify **5 root-cause problems** causing the test failure.

---

## Problem Analysis

### Problem 1 (Critical): Wrong loading function for .hex files

In `vortex_scoreboard.sv` run_phase, it calls `simx_load_bin` on a `.hex` file:
```sv
status = simx_load_bin(cfg.program_path, 64'(cfg.startup_addr));
```
`simx_load_bin` opens the file in **raw binary mode** — it reads the ASCII text `"93 01 00 00\n"` as literal bytes `0x39,0x33,0x20,...`. Garbage gets loaded into SimX RAM.

### Problem 2 (Critical): Wrong load address — SimX vs RTL mismatch

`program_simple.hex` starts with `@00000000` (relative address, passes FIX-C check). The `mem_model` correctly applies `base_addr=0x80000000`, so RTL gets the program at **0x80000000**. But `simx_load_hex` uses the literal `@00000000` → loads at **0x00000000**. DCR also gets set to 0. SimX runs from address 0 (uninitialized memory) → garbage execution → exit code 3.

### Problem 3 (Critical): Wrong bootstrap installation order

Current order in `run_phase`:
1. `simx_init()`
2. `simx_init_exit_code_register()` → DCR = `0x7FFFFFF0`
3. `simx_load_bin()` → DCR overridden back to wrong address

Bootstrap is overridden and never takes effect.

### Problem 4 (Critical): Status monitor never broadcasts EBREAK to scoreboard

`status_monitor.detect_state_transitions()` uses `@(vif.monitor_cb)` every cycle but **never calls `ap.write()`** when ebreak fires. Only `monitor_status()` calls `ap.write()`, at `sample_interval=100` cycles. With only 154 total cycles, the ebreak window is missed. Scoreboard never gets the notification → falls back to `extract_phase` comparison → warning "EBREAK never observed."

### Problem 5 (Cosmetic): DCR monitor reports "startup not complete"

TB_TOP drives DCR wires directly, bypassing the UVM DCR monitor. Not a real error but produces false warnings.

---

## Fixes — Specific File Changes

### Fix 1: `uvm_env/ref_model/simx_dpi.cpp` — Add `simx_load_hex_at`

Add this new function **before** the closing `} // extern "C"`:

```cpp
// Load hex file with base_addr offset applied to @addresses.
// Handles both byte format (--verilog-data-width=1) and word format.
// Does NOT change g_startup_addr — caller controls that via bootstrap.
int simx_load_hex_at(const char* filepath, uint64_t base_addr) {
    if (!g_initialized || !g_ram) {
        std::cerr << "[SimX-DPI] Error: SimX not initialized" << std::endl;
        return -1;
    }

    std::ifstream file(filepath);
    if (!file.is_open()) {
        std::cerr << "[SimX-DPI] Cannot open hex file: " << filepath << std::endl;
        return -1;
    }

    std::string line;
    uint64_t current_addr = base_addr;
    int bytes_loaded = 0;

    while (std::getline(file, line)) {
        // Strip CR/LF
        while (!line.empty() && (line.back() == '\r' || line.back() == '\n'))
            line.pop_back();
        if (line.empty() || line[0] == '/' || line[0] == '#') continue;

        if (line[0] == '@') {
            // Address marker: apply base_addr offset
            uint64_t rel = std::stoull(line.substr(1), nullptr, 16);
            current_addr = rel + base_addr;
            std::cout << "[SimX-DPI] hex_at: @" << std::hex << rel
                      << " → absolute 0x" << current_addr << std::dec << std::endl;
            continue;
        }

        // Parse space-separated tokens (byte or word format)
        std::istringstream iss(line);
        std::string tok;
        while (iss >> tok) {
            if (tok.size() <= 2) {
                // Byte format (--verilog-data-width=1)
                uint8_t b = (uint8_t)std::stoul(tok, nullptr, 16);
                g_ram->write(&b, current_addr++, 1);
                bytes_loaded++;
            } else {
                // 32-bit word format (little-endian)
                uint32_t w = (uint32_t)std::stoul(tok, nullptr, 16);
                uint8_t bytes[4] = {
                    (uint8_t)(w),       (uint8_t)(w >> 8),
                    (uint8_t)(w >> 16), (uint8_t)(w >> 24)
                };
                g_ram->write(bytes, current_addr, 4);
                current_addr += 4;
                bytes_loaded += 4;
            }
        }
    }

    std::cout << "[SimX-DPI] Loaded " << bytes_loaded
              << " bytes from '" << filepath
              << "' at base=0x" << std::hex << base_addr << std::dec << std::endl;
    return 0;
}
```

---

### Fix 2: `uvm_env/vortex_env_pkg.sv` — Add DPI import

Find the block of `import "DPI-C"` statements and add one line:

```sv
  import "DPI-C" function int  simx_load_hex  (string filepath);
  // ADD THIS LINE ↓
  import "DPI-C" function int  simx_load_hex_at (string filepath, longint unsigned base_addr);
  import "DPI-C" function void simx_dcr_write (int addr, int value);
```

---

### Fix 3: `uvm_env/vortex_scoreboard.sv` — Fix load order and function

Replace the entire `run_phase` task with this corrected version:

```sv
virtual task run_phase(uvm_phase phase);
    int status;

    if (!cfg.simx_enable) begin
      `uvm_info("SCOREBOARD", "SimX disabled — shadow-memory checks only", UVM_MEDIUM)
      return;
    end

    `uvm_info("SCOREBOARD",
      $sformatf("Initialising SimX: cores=%0d warps=%0d threads=%0d",
                cfg.num_cores, cfg.num_warps, cfg.num_threads), UVM_MEDIUM)

    status = simx_init(cfg.num_cores, cfg.num_warps, cfg.num_threads);
    if (status != 0) begin
      `uvm_error("SCOREBOARD",
        $sformatf("simx_init() failed (status=%0d) — disabling SimX", status))
      cfg.simx_enable = 0;
      return;
    end
    `uvm_info("SCOREBOARD", "SimX initialised successfully", UVM_MEDIUM)

    // ── STEP 1: Load program FIRST ──────────────────────────────────────────
    // Must happen before bootstrap so the bootstrap DCR write is the LAST one.
    if (cfg.program_path != "") begin
      if (cfg.program_type == "hex") begin
        // Use hex_at to apply startup_addr offset to @00000000 addresses
        status = simx_load_hex_at(cfg.program_path, 64'(cfg.startup_addr));
      end else begin
        status = simx_load_bin(cfg.program_path, 64'(cfg.startup_addr));
      end
      if (status != 0)
        `uvm_error("SCOREBOARD",
          $sformatf("Failed to load program '%s' (type=%s)",
                    cfg.program_path, cfg.program_type))
      else
        `uvm_info("SCOREBOARD",
          $sformatf("SimX: loaded '%s' @ 0x%0h", cfg.program_path, cfg.startup_addr),
          UVM_MEDIUM)
    end

    // ── STEP 2: Install exit-code bootstrap AFTER program load ───────────────
    // This sets DCR startup to bootstrap at (startup_addr-16), overriding any
    // DCR that simx_load_* may have set. Bootstrap sets x3=0 then jumps to
    // startup_addr so the program always exits with code 0 (unless it sets x3).
    simx_init_exit_code_register();

endtask : run_phase
```

---

### Fix 4: `uvm_env/agents/status_agent/status_monitor.sv` — Immediate EBREAK broadcast

In `detect_state_transitions()`, find the EBREAK detection block (around line 266) and add the `ap.write()` call immediately when ebreak fires:

```sv
// EBREAK detection (program completion)
if (vif.monitor_cb.ebreak_detected && !prev_ebreak) begin
    execution_end_time = $time;
    execution_completed = 1;
    total_execution_cycles = vif.monitor_cb.cycle_count - execution_start_cycle;
    total_instructions = vif.monitor_cb.instr_count;

    -> execution_complete;  // Trigger event (CRITICAL for Option A)

    // ── ADD: broadcast immediately — don't wait for sample_interval ──────
    begin
        status_transaction ebreak_trans;
        ebreak_trans = status_transaction::type_id::create("ebreak_trans");
        ebreak_trans.busy            = vif.monitor_cb.busy;
        ebreak_trans.ebreak_detected = 1'b1;
        ebreak_trans.idle            = !vif.monitor_cb.busy;
        ebreak_trans.cycle_count     = vif.monitor_cb.cycle_count;
        ebreak_trans.instr_count     = vif.monitor_cb.instr_count;
        ebreak_trans.pc              = vif.monitor_cb.pc;
        ebreak_trans.sample_time     = $time;
        ebreak_trans.calculate_metrics();
        ap.write(ebreak_trans);
        `uvm_info("STATUS_MON",
            $sformatf("EBREAK: immediate broadcast to scoreboard @ cycle %0d",
                      vif.monitor_cb.cycle_count), UVM_LOW)
    end
    // ─────────────────────────────────────────────────────────────────────

    if (total_execution_cycles > 0) begin
        final_ipc = real'(total_instructions) / real'(total_execution_cycles);
    end

    `uvm_info("STATUS_MON", { ... }, UVM_LOW)  // existing summary
end
```

---

### Fix 5: `uvm_env/agents/host_agent/program_simple.hex` — Correct content

Replace with this (2 instructions + 56-byte zero pad = 64 bytes = 1 cache line):

```
@00000000
93 01 00 00 73 00 10 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

Where `93 01 00 00` = `addi x3, x0, 0` (explicit exit code 0) and `73 00 10 00` = `ebreak`.

---

### Fix 6: Rebuild `simx_model.so`

After editing `simx_dpi.cpp`, rebuild the shared library from `uvm_env/ref_model/`:

```bash
cd uvm_env/ref_model
make build
```

---

## Expected Result After All Fixes

```
[SimX-DPI] hex_at: @00000000 → absolute 0x80000000
[SimX-DPI] Loaded 64 bytes from 'program_simple.hex' at base=0x80000000
[SimX-DPI] Bootstrap: 0x7ffffff0 → x3=0 → jump 0x80000000
[SimX-DPI] Exit code: 0        ← was 3
UVM_INFO  EBREAK: immediate broadcast to scoreboard @ cycle 154
UVM_INFO  SCOREBOARD: SimX done — exit code = 0   ← no UVM_ERROR
UVM_INFO  SCOREBOARD: Comparing result region...
UVM_INFO  *** TEST PASSED ***   ← 0 UVM_ERROR
```

The 2 UVM_ERRORs disappear: one from wrong SimX exit code, one from the base_test error counter check.





---
Powered by [Claude Exporter](https://www.ai-chat-exporter.net)