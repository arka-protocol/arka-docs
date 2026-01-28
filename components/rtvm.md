# RTVM (Regulatory Technology Virtual Machine)

The RTVM is the deterministic execution engine at the heart of ARKA Engine. It executes compiled compliance rules with guaranteed reproducibility and generates cryptographic proofs of execution.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Bytecode Specification](#bytecode-specification)
- [Execution Model](#execution-model)
- [Determinism Guarantees](#determinism-guarantees)
- [Configuration](#configuration)
- [Performance](#performance)
- [Troubleshooting](#troubleshooting)

---

## Overview

### Purpose

The RTVM provides:

1. **Deterministic Execution**: Identical inputs always produce identical outputs
2. **Isolated Execution**: Each evaluation runs in complete isolation
3. **Verifiable Execution**: Complete execution traces for audit and verification
4. **Efficient Execution**: Optimized bytecode execution for high throughput

### Key Properties

| Property | Description |
|----------|-------------|
| **Determinism** | Same input + same rules = same output (always) |
| **Isolation** | No side effects between evaluations |
| **Reproducibility** | Can replay any historical evaluation |
| **Auditability** | Complete execution trace captured |
| **Security** | Sandboxed execution with resource limits |

---

## Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           RTVM ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                      BYTECODE LOADER                                │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │ │
│  │  │   Bytecode   │  │  Validator   │  │    Cache     │              │ │
│  │  │   Reader     │  │              │  │   Manager    │              │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                    │                                     │
│                                    ▼                                     │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                      EXECUTION ENGINE                               │ │
│  │                                                                     │ │
│  │  ┌──────────────────────────────────────────────────────────────┐  │ │
│  │  │                    EXECUTION CONTEXT                          │  │ │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │  │ │
│  │  │  │  Stack   │  │  Locals  │  │  Heap    │  │  Trace   │     │  │ │
│  │  │  │          │  │          │  │          │  │  Buffer  │     │  │ │
│  │  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │  │ │
│  │  └──────────────────────────────────────────────────────────────┘  │ │
│  │                                                                     │ │
│  │  ┌──────────────────────────────────────────────────────────────┐  │ │
│  │  │                    INSTRUCTION EXECUTOR                       │  │ │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │  │ │
│  │  │  │  Stack   │  │Arithmetic│  │ Control  │  │ Domain   │     │  │ │
│  │  │  │   Ops    │  │   Ops    │  │  Flow    │  │ Specific │     │  │ │
│  │  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │  │ │
│  │  └──────────────────────────────────────────────────────────────┘  │ │
│  │                                                                     │ │
│  │  ┌──────────────────────────────────────────────────────────────┐  │ │
│  │  │                    RESOURCE MONITOR                           │  │ │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │  │ │
│  │  │  │  Memory  │  │   CPU    │  │   Time   │  │  Stack   │     │  │ │
│  │  │  │  Limit   │  │  Limit   │  │  Limit   │  │  Depth   │     │  │ │
│  │  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │  │ │
│  │  └──────────────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                    │                                     │
│                                    ▼                                     │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                      PROOF GENERATOR                                │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │ │
│  │  │    Trace     │  │    Hash      │  │  Attestation │              │ │
│  │  │  Collector   │  │  Generator   │  │   Builder    │              │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Execution Pipeline

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  Load   │───▶│  Init   │───▶│ Execute │───▶│ Collect │───▶│  Return │
│Bytecode │    │ Context │    │  Loop   │    │  Trace  │    │ Result  │
└─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘
```

---

## Bytecode Specification

### Instruction Format

Each instruction is encoded as:

```
┌─────────────────────────────────────────────────────────────┐
│                    INSTRUCTION FORMAT                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Byte 0       Bytes 1-N (variable)                          │
│  ┌────────┐  ┌─────────────────────────────────────────┐    │
│  │ Opcode │  │            Operands                     │    │
│  │ (1 byte)│  │   (0-8 bytes depending on opcode)      │    │
│  └────────┘  └─────────────────────────────────────────┘    │
│                                                              │
│  Example: PUSH_CONST 42                                      │
│  ┌────────┬────────────────────┐                            │
│  │  0x01  │  0x00 0x00 0x00 0x2A│   (42 as i32)             │
│  └────────┴────────────────────┘                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Instruction Categories

#### Stack Operations (0x00-0x0F)

| Opcode | Mnemonic | Operands | Description |
|--------|----------|----------|-------------|
| 0x00 | NOP | - | No operation |
| 0x01 | PUSH_CONST | value: varies | Push constant onto stack |
| 0x02 | PUSH_LOCAL | index: u16 | Push local variable |
| 0x03 | POP | - | Remove top of stack |
| 0x04 | DUP | - | Duplicate top of stack |
| 0x05 | SWAP | - | Swap top two values |
| 0x06 | ROT | - | Rotate top three values |

#### Arithmetic Operations (0x10-0x2F)

| Opcode | Mnemonic | Operands | Description |
|--------|----------|----------|-------------|
| 0x10 | ADD_I64 | - | Add two i64 values |
| 0x11 | SUB_I64 | - | Subtract two i64 values |
| 0x12 | MUL_I64 | - | Multiply two i64 values |
| 0x13 | DIV_I64 | - | Divide two i64 values |
| 0x14 | MOD_I64 | - | Modulo two i64 values |
| 0x18 | ADD_DEC | precision: u8 | Add decimals with precision |
| 0x19 | SUB_DEC | precision: u8 | Subtract decimals |
| 0x1A | MUL_DEC | precision: u8 | Multiply decimals |
| 0x1B | DIV_DEC | precision: u8, mode: u8 | Divide with rounding mode |

#### Comparison Operations (0x30-0x3F)

| Opcode | Mnemonic | Operands | Description |
|--------|----------|----------|-------------|
| 0x30 | EQ | - | Equal |
| 0x31 | NE | - | Not equal |
| 0x32 | LT | - | Less than |
| 0x33 | LE | - | Less than or equal |
| 0x34 | GT | - | Greater than |
| 0x35 | GE | - | Greater than or equal |

#### Logical Operations (0x40-0x4F)

| Opcode | Mnemonic | Operands | Description |
|--------|----------|----------|-------------|
| 0x40 | AND | - | Logical AND |
| 0x41 | OR | - | Logical OR |
| 0x42 | NOT | - | Logical NOT |
| 0x43 | XOR | - | Logical XOR |

#### Control Flow (0x50-0x5F)

| Opcode | Mnemonic | Operands | Description |
|--------|----------|----------|-------------|
| 0x50 | JMP | offset: i32 | Unconditional jump |
| 0x51 | JMP_IF | offset: i32 | Jump if true |
| 0x52 | JMP_UNLESS | offset: i32 | Jump if false |
| 0x53 | CALL | func_id: u32 | Call function |
| 0x54 | RET | - | Return from function |
| 0x55 | HALT | status: u8 | Halt execution |

#### Temporal Operations (0x60-0x6F)

| Opcode | Mnemonic | Operands | Description |
|--------|----------|----------|-------------|
| 0x60 | DATE_NOW | - | Push current timestamp |
| 0x61 | DATE_CMP | - | Compare two dates |
| 0x62 | DATE_ADD | unit: u8 | Add duration to date |
| 0x63 | DATE_SUB | unit: u8 | Subtract duration |
| 0x64 | DATE_DIFF | unit: u8 | Difference between dates |
| 0x65 | DATE_PART | part: u8 | Extract date part |
| 0x66 | IN_RANGE | - | Check if date in range |

#### Data Access (0x70-0x7F)

| Opcode | Mnemonic | Operands | Description |
|--------|----------|----------|-------------|
| 0x70 | LOAD_FIELD | path: string | Load field from input |
| 0x71 | LOAD_ARRAY | - | Load array element |
| 0x72 | ARRAY_LEN | - | Get array length |
| 0x73 | OBJ_GET | key: string | Get object property |
| 0x74 | OBJ_HAS | key: string | Check property exists |

#### Domain-Specific (0x80-0xAF)

| Opcode | Mnemonic | Operands | Description |
|--------|----------|----------|-------------|
| 0x80 | JUR_LOAD | jur_id: string | Load jurisdiction context |
| 0x81 | JUR_CHECK | - | Check jurisdiction applies |
| 0x82 | RULE_REF | rule_id: string | Reference another rule |
| 0x83 | EVIDENCE_ADD | type: u8 | Add evidence item |
| 0x84 | FLAG | severity: u8 | Create compliance flag |
| 0x85 | ANNOTATE | key: string | Add annotation |

### Data Types

```
┌─────────────────────────────────────────────────────────────┐
│                    RTVM DATA TYPES                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Primitive Types:                                            │
│  ┌────────────┬──────────┬─────────────────────────────────┐│
│  │ Type       │ Size     │ Description                     ││
│  ├────────────┼──────────┼─────────────────────────────────┤│
│  │ bool       │ 1 byte   │ Boolean (0 or 1)                ││
│  │ i32        │ 4 bytes  │ Signed 32-bit integer           ││
│  │ i64        │ 8 bytes  │ Signed 64-bit integer           ││
│  │ u32        │ 4 bytes  │ Unsigned 32-bit integer         ││
│  │ u64        │ 8 bytes  │ Unsigned 64-bit integer         ││
│  │ f64        │ 8 bytes  │ 64-bit float (limited use)      ││
│  │ decimal    │ 16 bytes │ Fixed-precision decimal         ││
│  └────────────┴──────────┴─────────────────────────────────┘│
│                                                              │
│  Compound Types:                                             │
│  ┌────────────┬─────────────────────────────────────────────┐│
│  │ Type       │ Description                                 ││
│  ├────────────┼─────────────────────────────────────────────┤│
│  │ string     │ UTF-8 encoded, length-prefixed              ││
│  │ bytes      │ Raw byte array, length-prefixed             ││
│  │ array      │ Homogeneous array of any type               ││
│  │ object     │ Key-value map (string keys)                 ││
│  │ timestamp  │ ISO 8601 datetime (stored as i64)           ││
│  │ duration   │ Time duration (stored as i64 nanoseconds)   ││
│  └────────────┴─────────────────────────────────────────────┘│
│                                                              │
│  Decimal Representation:                                     │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  struct Decimal {                                        ││
│  │    mantissa: i128,    // Scaled integer value            ││
│  │    scale: u8,         // Decimal places (0-38)           ││
│  │  }                                                        ││
│  │                                                           ││
│  │  Example: 123.45 with scale 2                            ││
│  │  mantissa = 12345, scale = 2                             ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Execution Model

### Execution Context

Each evaluation creates an isolated execution context:

```rust
struct ExecutionContext {
    // Stack for computation
    stack: Vec<Value>,
    max_stack_depth: usize,

    // Local variables
    locals: Vec<Value>,

    // Heap for complex objects
    heap: Heap,
    max_heap_size: usize,

    // Execution state
    program_counter: usize,
    call_stack: Vec<CallFrame>,

    // Input data (read-only)
    input: Value,

    // Execution trace
    trace: ExecutionTrace,

    // Resource counters
    instructions_executed: u64,
    max_instructions: u64,
    start_time: Instant,
    max_duration: Duration,

    // Determinism seed
    determinism_seed: [u8; 32],
}
```

### Execution Loop

```rust
fn execute(bytecode: &[u8], input: Value, config: &Config) -> Result<ExecutionResult> {
    let mut ctx = ExecutionContext::new(input, config);

    loop {
        // Check resource limits
        ctx.check_limits()?;

        // Fetch instruction
        let opcode = bytecode[ctx.program_counter];
        ctx.program_counter += 1;

        // Decode and execute
        match opcode {
            0x00 => { /* NOP */ },
            0x01 => {
                let value = decode_value(&bytecode[ctx.program_counter..])?;
                ctx.stack.push(value);
                ctx.program_counter += value.encoded_size();
            },
            0x30 => {
                let b = ctx.stack.pop()?;
                let a = ctx.stack.pop()?;
                ctx.stack.push(Value::Bool(a == b));
            },
            0x55 => {
                let status = bytecode[ctx.program_counter];
                return Ok(ExecutionResult {
                    status,
                    output: ctx.stack.pop()?,
                    trace: ctx.trace,
                });
            },
            // ... other opcodes
        }

        // Record trace
        ctx.trace.record_step(opcode, &ctx.stack);
    }
}
```

### Call Stack Management

```rust
struct CallFrame {
    return_address: usize,
    base_pointer: usize,      // Stack base for this frame
    local_count: usize,       // Number of locals in frame
}

fn call_function(ctx: &mut ExecutionContext, func_id: u32) -> Result<()> {
    let func = ctx.get_function(func_id)?;

    // Save return address
    ctx.call_stack.push(CallFrame {
        return_address: ctx.program_counter,
        base_pointer: ctx.stack.len(),
        local_count: func.local_count,
    });

    // Allocate locals
    for _ in 0..func.local_count {
        ctx.locals.push(Value::Null);
    }

    // Jump to function
    ctx.program_counter = func.entry_point;

    Ok(())
}

fn return_from_function(ctx: &mut ExecutionContext) -> Result<()> {
    let frame = ctx.call_stack.pop()?;

    // Deallocate locals
    ctx.locals.truncate(ctx.locals.len() - frame.local_count);

    // Restore return address
    ctx.program_counter = frame.return_address;

    Ok(())
}
```

---

## Determinism Guarantees

### Sources of Non-Determinism (Eliminated)

| Source | How RTVM Eliminates It |
|--------|------------------------|
| Floating-point | Use fixed-precision decimals |
| Random numbers | Seeded PRNG with determinism_seed |
| Current time | Explicit timestamp in context |
| Hash iteration order | Canonical ordering |
| Memory addresses | Virtual addresses in heap |
| External I/O | Prohibited during execution |
| Threading | Single-threaded execution |

### Fixed-Precision Arithmetic

```rust
impl Decimal {
    /// Add two decimals with specified precision
    fn add(self, other: Decimal, precision: u8) -> Decimal {
        // Align scales
        let (a, b) = self.align_scales(other);

        // Perform addition
        let result_mantissa = a.mantissa + b.mantissa;

        // Round to precision
        Decimal::round(result_mantissa, a.scale, precision)
    }

    /// Round to specified precision using HALF_EVEN (banker's rounding)
    fn round(mantissa: i128, current_scale: u8, target_scale: u8) -> Decimal {
        if current_scale <= target_scale {
            return Decimal { mantissa, scale: current_scale };
        }

        let scale_diff = current_scale - target_scale;
        let divisor = 10i128.pow(scale_diff as u32);

        let quotient = mantissa / divisor;
        let remainder = mantissa % divisor;

        // HALF_EVEN rounding
        let half = divisor / 2;
        let rounded = if remainder > half {
            quotient + 1
        } else if remainder < half {
            quotient
        } else {
            // Exactly half: round to even
            if quotient % 2 == 0 { quotient } else { quotient + 1 }
        };

        Decimal { mantissa: rounded, scale: target_scale }
    }
}
```

### Canonical Ordering

```rust
/// Canonicalize object for hashing
fn canonicalize(value: &Value) -> Vec<u8> {
    match value {
        Value::Object(map) => {
            let mut entries: Vec<_> = map.iter().collect();
            // Sort keys lexicographically
            entries.sort_by(|a, b| a.0.cmp(b.0));

            let mut result = Vec::new();
            for (key, val) in entries {
                result.extend(canonicalize(&Value::String(key.clone())));
                result.extend(canonicalize(val));
            }
            result
        },
        Value::Array(arr) => {
            arr.iter().flat_map(canonicalize).collect()
        },
        // ... other types
    }
}
```

### Deterministic PRNG

```rust
/// Deterministic random number generator
struct DeterministicRng {
    state: [u64; 4],  // xoshiro256** state
}

impl DeterministicRng {
    fn from_seed(seed: [u8; 32]) -> Self {
        // Initialize state from seed
        let mut state = [0u64; 4];
        for i in 0..4 {
            state[i] = u64::from_le_bytes(
                seed[i*8..(i+1)*8].try_into().unwrap()
            );
        }
        Self { state }
    }

    fn next_u64(&mut self) -> u64 {
        // xoshiro256** algorithm
        let result = self.state[1].wrapping_mul(5).rotate_left(7).wrapping_mul(9);
        let t = self.state[1] << 17;

        self.state[2] ^= self.state[0];
        self.state[3] ^= self.state[1];
        self.state[1] ^= self.state[2];
        self.state[0] ^= self.state[3];
        self.state[2] ^= t;
        self.state[3] = self.state[3].rotate_left(45);

        result
    }
}
```

---

## Configuration

### RTVM Configuration File

```yaml
# rtvm.yaml
rtvm:
  # Resource limits
  limits:
    max_stack_depth: 1000
    max_heap_size: 100MB
    max_instructions: 10000000
    max_execution_time: 30s
    max_call_depth: 100

  # Memory settings
  memory:
    initial_heap: 10MB
    heap_growth_factor: 2.0
    gc_threshold: 0.75

  # Execution settings
  execution:
    trace_enabled: true
    trace_level: full  # none, basic, full
    proof_generation: true

  # Decimal arithmetic
  arithmetic:
    default_precision: 18
    max_precision: 38
    rounding_mode: half_even

  # Performance
  performance:
    bytecode_cache_size: 1000
    worker_threads: 8
    batch_size: 100

  # Security
  security:
    sandbox_enabled: true
    syscall_filter: strict
```

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `RTVM_MAX_STACK_DEPTH` | Maximum stack depth | 1000 |
| `RTVM_MAX_HEAP_SIZE` | Maximum heap size | 100MB |
| `RTVM_MAX_INSTRUCTIONS` | Max instructions per evaluation | 10M |
| `RTVM_MAX_EXECUTION_TIME` | Max execution time | 30s |
| `RTVM_TRACE_LEVEL` | Trace detail level | basic |
| `RTVM_WORKER_THREADS` | Number of worker threads | CPU count |

---

## Performance

### Benchmarks

| Metric | Value | Notes |
|--------|-------|-------|
| Simple rule evaluation | ~50 μs | Single condition |
| Complex rule (10 conditions) | ~200 μs | Multiple branches |
| Large input (10KB) | ~500 μs | With parsing |
| Throughput (single thread) | ~20,000 eval/s | Simple rules |
| Throughput (8 threads) | ~150,000 eval/s | Simple rules |

### Optimization Techniques

1. **Bytecode Caching**: Compiled bytecode cached in memory
2. **JIT Compilation**: Hot paths compiled to native code (optional)
3. **Batch Evaluation**: Multiple evaluations in single call
4. **Memory Pooling**: Reuse execution contexts
5. **Lazy Loading**: Load rule bytecode on demand

### Monitoring Metrics

```
# RTVM metrics (Prometheus format)

# Evaluation latency
rtvm_evaluation_duration_seconds_bucket{le="0.001"} 45000
rtvm_evaluation_duration_seconds_bucket{le="0.01"} 49000
rtvm_evaluation_duration_seconds_bucket{le="0.1"} 50000
rtvm_evaluation_duration_seconds_sum 250
rtvm_evaluation_duration_seconds_count 50000

# Resource usage
rtvm_stack_depth_max 45
rtvm_heap_bytes_used 1048576
rtvm_instructions_executed_total 500000000

# Errors
rtvm_errors_total{type="stack_overflow"} 0
rtvm_errors_total{type="timeout"} 5
rtvm_errors_total{type="invalid_bytecode"} 0
```

---

## Troubleshooting

### Common Issues

#### Stack Overflow

**Symptom**: `StackOverflowError: stack depth exceeded limit`

**Cause**: Rule has deeply nested conditions or recursive calls

**Solution**:
```yaml
# Increase stack limit if rule is legitimate
limits:
  max_stack_depth: 2000

# Or optimize rule to reduce nesting
```

#### Execution Timeout

**Symptom**: `TimeoutError: execution exceeded time limit`

**Cause**: Rule too complex or infinite loop

**Solution**:
```bash
# Analyze rule complexity
arka-cli rule analyze <rule_id> --complexity

# Increase timeout if justified
RTVM_MAX_EXECUTION_TIME=60s
```

#### Decimal Precision Loss

**Symptom**: Unexpected rounding in calculations

**Cause**: Operations with insufficient precision

**Solution**:
```yaml
# Increase default precision
arithmetic:
  default_precision: 28
```

### Debug Mode

Enable detailed debugging:

```bash
# Start RTVM with debug logging
RTVM_LOG_LEVEL=debug RTVM_TRACE_LEVEL=full arka-rtvm

# Analyze specific evaluation
arka-cli rtvm debug --evaluation-id <id> --show-trace
```

### Execution Trace Analysis

```json
{
  "evaluation_id": "eval_123",
  "trace": {
    "instructions": [
      {"pc": 0, "opcode": "PUSH_CONST", "stack": [42]},
      {"pc": 5, "opcode": "LOAD_FIELD", "stack": [42, 100]},
      {"pc": 12, "opcode": "GT", "stack": [true]},
      {"pc": 13, "opcode": "JMP_IF", "stack": []},
      {"pc": 20, "opcode": "FLAG", "stack": ["high"]}
    ],
    "total_instructions": 5,
    "max_stack_depth": 2,
    "heap_allocated": 0
  }
}
```

---

## Next Steps

- [Rule Authoring Guide](../guides/rule-authoring.md) - Write rules for the RTVM
- [Plugin Framework](./plugin-framework.md) - Extend RTVM with plugins
- [Distributed Agents](./distributed-agents.md) - Deploy RTVM at the edge
