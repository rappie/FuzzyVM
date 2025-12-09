# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

FuzzyVM is a fuzzing framework for differential testing of Ethereum Virtual Machine (EVM) implementations. It generates EVM state tests that can be executed against multiple EVM clients to find consensus bugs. Test execution is handled by the external `goevmlab` tool.

## Build and Test Commands

```bash
# Build
go build ./...

# Run all tests
go test -v ./...

# Run native Go fuzzer
go test --fuzz FuzzVMBasic --parallel N

# CLI commands (after building cmd/fuzzyvm)
./fuzzyvm corpus --count N      # Generate N corpus elements
./fuzzyvm run --threads N       # Run fuzzer with N threads
./fuzzyvm bench --count N       # Benchmark test generation
./fuzzyvm minCorpus             # Minimize corpus (remove duplicates)
```

## Architecture

### Data Flow
Input data (fuzzing corpus) → **Filler** → **Generator** → **Fuzzer** → GeneralStateTest JSON output

### Core Components

**Filler (`filler/`)**: Abstracts fuzzing input as a consumable byte stream with typed extraction methods (Byte, Uint16/32/64, BigInt256, GasInt, MemInt). Implements `io.Reader`.

**Generator (`generator/`)**: Creates EVM bytecode using a strategy pattern. Strategies are probabilistically selected based on importance weights (1-100). Three strategy groups:
- Basic strategies (13): Individual opcodes, memory/storage, hashing
- Call strategies (4): CREATE/CREATE2, CALL, precompile invocations
- Jump strategies: JUMP/JUMPI with valid destination management

**Fuzzer (`fuzzer/`)**: Orchestrates test generation. Entry points are `Fuzz()` and `FuzzStateless()` for go-fuzz. Handles program minimization via binary search and test deduplication via SHA3-256 hashing.

**Precompiles (`generator/precompiles/`)**: Handlers for EVM precompiled contracts (ecrecover, SHA256, RIPEMD160, identity, modexp, bn256 operations, blake2f).

### Key Constants
- Fork: "Cancun" (generator.go:32)
- Sender: 0xa94f5374fce5edbc8e2a8697c15331677e6ebf0b
- Target contract: 0x0000ca1100f022
- Max bytecode: 10,000 bytes
- Max recursion depth: 10 (CREATE/CALL nesting)
- Min corpus input: 32 bytes

### Output Structure
- `out/[hash_prefix]/`: Generated tests organized by first byte of SHA3 hash
- `corpus/`: Fuzzing corpus files
- `crashes/`: Error cases

## Debugging

- Set `debug = true` in generator/generator.go:59 for verbose generation output
- Set `shouldTrace = true` in fuzzer/fuzzer.go for execution tracing

## Notes

- BLOCKHASH opcode is intentionally excluded due to Nethermind's different implementation
- Recursion level decrement is currently disabled in call_strategies.go:77
