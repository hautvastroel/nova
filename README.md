# Challenge 0009 Analysis

## Challenge Overview

**Title:** RV32IMCK Processor Design  
**Difficulty:** Medium  
**Estimated Time:** 336 hours

## Critical Requirements

### 1. Functionality (50% of score)
- **CRITICAL:** Functionality score must be **at least 80%** otherwise overall score is set to **ZERO**
- Must correctly execute all validator-side functional tests
- Covers diverse instruction sequences
- This is the most important metric - without 80% functionality, the solution gets zero points

### 2. Area (25% of score)
- Target: **200,000 μm²**
- Optimization needed but secondary to functionality

### 3. Delay (25% of score)
- Target: **100,000,000 instructions per second** (100 MIPS)
- Performance optimization needed but secondary to functionality

### 4. Power (0% of score)
- Not evaluated in this challenge

## Submission Requirements

- **File Format:** SystemVerilog/Verilog
- **Filelist Name:** `rtl.f`
- **Max File Size:** 50,000 KB per file
- **Coding Style:** Synthesizable Verilog 2005

## Processor Requirements

The processor must support:
- **RV32I:** Base Integer instruction set
- **M:** Multiply/Divide extension
- **C:** Compressed instruction extension (16-bit instructions)
- **K:** Cryptography extension

## Key Issues to Address

Based on previous analysis, the following critical issues prevent reaching 80% functionality:

### 1. Division/Remainder Algorithm (HIGH PRIORITY)
- **Problem:** Quotient calculation is incorrect
- **Impact:** Affects DIV, DIVU, REM, REMU instructions
- **Status:** Remainder calculation fixed, but quotient still wrong
- **Root Cause:** Bit extraction logic in `divu_int` module

### 2. PC Misalignment (HIGH PRIORITY)
- **Problem:** Core PC is 4 bytes ahead of Spike reference
- **Impact:** Affects instruction tracing and comparison
- **Status:** Previous fix attempt decreased score (reverted)
- **Root Cause:** Compressed instruction handling or PC update logic

### 3. Instruction Execution Mismatches (MEDIUM PRIORITY)
- Various instruction types showing mismatches
- Need systematic analysis of failing instructions

## Strategy for Improvement

1. **Fix Division Algorithm First** - This affects multiple instruction types
2. **Investigate PC Logic Carefully** - Don't break what's working
3. **Systematic Instruction Testing** - Identify patterns in failures
4. **Verify Against Reference** - Use Spike as ground truth

## Success Criteria

- Functionality score ≥ 80%
- All test suites passing
- Correct instruction execution matching Spike reference
- Synthesizable code meeting Verilog 2005 standards

