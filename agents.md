# Operational Playbook for Autonomous AI Agents

This document is written for autonomous AI coding agents (such as Antigravity, SWE-agent, etc.) that have system shell access, terminal execution tools, and the capability to make file edits. It details exact execution procedures, testing environments, diagnostic troubleshooting, and code modification guardrails.

> [!IMPORTANT]
> **SELF-UPDATE MANDATE**: If any changes you make to the codebase during your work affect existing patterns, add new modules, change core APIs, or introduce new paradigms, you **MUST** update this file (`agents.md`) and `llms.md` as part of your changes to keep future AI agents correctly instructed.

---

## 1. Prerequisites & Execution Environment

### 1.1 Python Dependencies
The randomizer runs entirely on Python 3 with the standard library. No external pip installations are required.

### 1.2 Target ROM File
To verify ROM modifications and successfully run a seed generation test, a valid Super Nintendo FF6 ROM file is required:
- **Filename**: ff3.smc (located in the workspace root).
- **Format**: Unheadered, 3,145,728 bytes (3MB).
- **Verify ROM Validity**: You can check the ROM's validity by running:
  ```powershell
  python3 -c "import valid_rom_file; print(valid_rom_file.valid_rom_file('ff3.smc'))"
  ```
  It should output `True`.

### 1.3 Test Data Isolation
> [!IMPORTANT]
> **TEST DATA DIRECTORY RULE**: All test output data (including modified test ROMs, debug log text files, or API metadata manifests generated during your test runs) **MUST** be isolated and placed inside a `tests` directory in the workspace root (e.g., `tests/test_output.smc`, `tests/test_output.txt`).
> - Do not write test output files directly to the root directory.
> - Ensure the `tests` directory is created before running commands (e.g. `mkdir -Force tests`).

---

## 2. Playbook Commands

### 2.1 Basic Verification
To test if the options interface parses arguments successfully without crashes:
```powershell
python3 wc.py -h
```

### 2.2 Seed Generation Verification
To run a complete randomizer build, executing all event scripting, memory parsing, and ROM compilations:
```powershell
# Ensure the tests folder exists
mkdir -Force tests

# Run seed build outputting directly to the isolated tests folder
python3 wc.py -i ff3.smc -o tests/test_output.smc
```
This generates the seed ROM and its log inside the `tests` directory.

### 2.3 Debug Seeding with Logs
To print full section reports (event rewards, character scaling, and shops mapping) to standard stdout:
```powershell
python3 wc.py -i ff3.smc -o tests/test_output.smc -debug -slog
```

---

## 3. Diagnostic & Troubleshooting Guide

During development, agents commonly trigger three specific errors. Here is how to diagnose and resolve them:

### 3.1 Memory Overflow / Bank Exhaustion
**Error Signature**:
```text
MemoryError: Not enough room in space "custom event toggle": Next (0xc0f124) > End (0xc0f100). Diff: 36
```
**Cause**: You have written more bytes of assembly instructions or static data than the target `Reserve` range fits, or have exceeded the size of a dynamically requested `Allocate` block in that bank.
**Resolution**:
1. Check the size parameters in your dynamic allocation: `Allocate(Bank.C0, size, "description")`. Increase `size` to support your assembly array length.
2. If using a `Reserve` block, you are locked to vanilla game boundaries. You must rewrite your routine to be more byte-efficient (e.g., using shared subroutines with `JSR`/`JSL` or tighter register usage).
3. Alternatively, place the bulk of your custom assembly payload inside a dynamic `Allocate` space in Bank `C0`, `C2`, or `F0`, and use the `Reserve` block only to write a `JMP` or `JSR` redirecting to your dynamically allocated address.

### 3.2 Dynamic Import / Startup Crash
**Error Signature**:
```text
AttributeError: module 'args' has no attribute 'my_new_flag'
```
**Cause**: The subsystem list inside [arguments.py](file:///c:/Projects/wrjones104_WorldsCollide/args/arguments.py) does not contain your new module, or a circular dependency was introduced.
**Resolution**:
1. Verify that your custom CLI option module is registered in `self.groups` inside the constructor of `Arguments` in [arguments.py](file:///c:/Projects/wrjones104_WorldsCollide/args/arguments.py#L4-L13).
2. Look at import structures. Ensure you did not import game subsystems globally at the top of your custom script file. Move imports inside functions (e.g. inside `mod()`) to resolve circular bindings.

### 3.3 Event Bit Validation Assertion
**Error Signature**:
```text
AssertionError: Number of char/esper only checks changed - Check usages of CHARACTER_ESPER_ONLY_REWARDS...
```
**Cause**: You modified the number of rewards in an event, causing the randomizer's gating logic to detect that the total available rewards no longer match the hardcoded pool sizes.
**Resolution**:
- If you deliberately added/removed a character/esper check to a quest event (e.g. `phoenix_cave.py`), you must update the global expected total `CHARACTER_ESPER_ONLY_REWARDS` constant in [constants/objectives/results.py](file:///c:/Projects/wrjones104_WorldsCollide/constants/objectives/results.py) (or related metadata files) so that the count validator passes.

---

## 4. Step-by-Step Code Modification Protocol

When tasked with implementing a new feature or fixing a bug, execute this cycle:

```mermaid
graph TD
    A[1. Identify Target Component] --> B[2. Check Memory Allocations & Offsets]
    B --> C[3. Implement Code Changes]
    C --> D[4. Run CLI and Seed Build Tests]
    D -->|Failure| E[5. Resolve Memory or Import Errors]
    E --> C
    D -->|Success| F[6. Run valid_rom_file Verification]
    F --> G[7. Update llms.md and agents.md]
```

1. **Locate Target Files**: Pinpoint if you need to modify options (`args/`), data modeling (`data/`), events (`event/`), or CPU logic (`bug_fixes/`, `battle/`).
2. **Review Offsets**: If doing assembly modifications, examine target ROM offsets to confirm no existing dynamic spaces overlap.
3. **Write Pythonic Logic**: Use standard code paradigms. Implement localized dynamic imports to keep module initialization clean.
4. **Compile & Generate**: Execute `python3 wc.py -i ff3.smc` to verify that standard seed compilation compiles cleanly.
5. **Verify Outputs**: Use `valid_rom_file` verification logic to ensure ROM consistency where applicable.
6. **Update Instruction Files**: Update both `llms.md` and `agents.md` immediately if any changes affect the structure or expectations documented inside these manuals.
