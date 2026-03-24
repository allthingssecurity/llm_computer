# Simplified Guide

This file explains the repo without assuming you already think in transformer jargon.

## Attribution

This codebase is based on and explicitly credits:

- Original repo: [AbdelStark/llm-provable-computer](https://github.com/AbdelStark/llm-provable-computer)
- Original concept article: [Percepta, "Can LLMs Be Computers?"](https://www.percepta.ai/blog/can-llms-be-computers)

This version extends the ISA and example set, but the core idea comes from that original work.

If you want diagrams instead of prose, see:

- [ARCHITECTURE.md](ARCHITECTURE.md)

## 30-Second Mental Model

This project is a small virtual machine presented in transformer-shaped form.

Normal VM:

1. Read current state.
2. Look at current instruction.
3. Maybe read memory.
4. Compute next state.

This repo:

1. Encode current state as numbers.
2. Use attention to read memory if the instruction needs it.
3. Use baked FFN weights for that instruction to compute the next state.
4. Repeat until `HALT`.

So the key idea is:

- attention = retrieval
- FFN = state transition

It is not a trained chatbot. It is a deterministic computer built from transformer-like pieces.

## Two Different Inputs: Program vs State

The biggest source of confusion is mixing up the program with the runtime state.

### 1. Program

The program is the fixed `.tvm` file.

Example:

```asm
.memory 4

LOADI 5
ADD 3
HALT
```

This is compile-time input.

### 2. State

The state is the current machine snapshot while the program is running.

```text
state = {
  pc,
  acc,
  sp,
  zero_flag,
  carry_flag,
  halted,
  memory
}
```

This is runtime input.

The most important field is `pc`:

```text
instruction = program[pc]
```

That is how the runtime knows which instruction to execute next.

## What The ISA Supports

The VM now has several instruction families.

### Basic arithmetic and memory

- `LOADI`, `LOAD`, `STORE`
- `ADD`, `ADDM`
- `SUB`, `SUBM`
- `MUL`, `MULM`
- `AND`, `ANDM`
- `OR`, `ORM`
- `XOR`, `XORM`
- `CMP`, `CMPM`

### Pointer-style memory

- `LOADP addr`
- `STOREP addr`
- `ADDP addr`
- `SUBP addr`
- `MULP addr`
- `CMPP addr`

These use a pointer cell:

```text
LOADP 1  => ACC = MEM[ MEM[1] ]
```

### Stack and subroutines

- `PUSH`, `POP`
- `CALL`, `RET`

### Stack-frame locals

- `ADJSP imm`
- `LOADR offset`
- `STORER offset`
- `ADDR offset`
- `SUBR offset`
- `MULR offset`
- `CMPR offset`

These let a subroutine treat memory near `SP` as local variables.

Example:

```text
LOADR 0  => ACC = MEM[SP + 0]
STORER 1 => MEM[SP + 1] = ACC
```

### Control flow

- `JMP`
- `JZ`
- `JNZ`
- `JC`
- `JNC`
- `HALT`

## What Gets Compiled Into Weights

The `.tvm` file is parsed into a `Program`.

Then each instruction is compiled into a small deterministic block with:

- one memory-read mode
- one baked FFN weight set
- one set of flag-update rules

So for:

```asm
0: LOADI 5
1: ADD 3
2: HALT
```

the compiler effectively creates:

- weights for instruction 0 = behavior of `LOADI 5`
- weights for instruction 1 = behavior of `ADD 3`
- weights for instruction 2 = behavior of `HALT`

No gradient descent.
No training dataset.
No learned model.

This is closer to generating a circuit than training a neural network.

## How Weights Are Baked

The baking logic lives mainly in:

- [src/model.rs](../src/model.rs)

The compiler fills output slots such as:

- `next_pc`
- `raw_acc`
- `next_sp`
- `mem_write_enable`
- `mem_write_addr`
- `mem_write_value`

### Example: `ADD 3`

The compiler bakes weights so that one FFN pass behaves like:

```text
next_pc = pc + 1
next_acc = acc + 3
next_sp = sp
```

Conceptually:

```text
OUT_NEXT_PC  <- IN_PC_NEXT
OUT_RAW_ACC  <- IN_ACC + 3
OUT_NEXT_SP  <- IN_SP
```

### Example: `LOADP 1`

The compiler marks this instruction as:

```text
memory_read = Indirect(1)
```

Then the FFN logic is baked to do:

```text
next_pc = pc + 1
next_acc = operand
next_sp = sp
```

where `operand` is whatever the attention system fetched.

### Example: `SWAP 1`

The compiler bakes:

```text
next_pc         = pc + 1
next_acc        = operand
mem_write_on    = 1
mem_write_addr  = 1
mem_write_value = current_acc
next_sp         = sp
```

So one forward pass can both:

- load a memory value into `ACC`
- write the old `ACC` back to memory

## What Attention Is Actually Doing

Attention is not processing text here.

It is only being used as a memory retrieval system.

Each memory cell keeps a write history as 2D points:

```text
(step, value)
```

If a memory address was written three times:

```text
(2, 10)
(5, 17)
(9, 23)
```

and the query is:

```text
q = [1, 0]
```

then the score is:

```text
q dot key = 1 * step + 0 * value = step
```

So the latest write gets the highest score.

That is why people say in this repo:

- keys = memory-write records
- query = "give me the latest value"
- output = selected memory value

### Memory read modes in practice

- `Direct(addr)` means read `MEM[addr]`
- `Indirect(addr)` means read `MEM[ MEM[addr] ]`
- `Pointer(addr)` means read the pointer cell value itself
- `StackTop` means read `MEM[SP]`
- `StackRelative(offset)` means read `MEM[SP + offset]`

## How Inference Works

One runtime step looks like this:

1. Start with current state.
2. Read `pc`.
3. Select the compiled instruction block for `program[pc]`.
4. Use attention to fetch the operand if needed.
5. Build the numeric input vector from state plus operand.
6. Run one FFN pass.
7. Decode outputs into the next state.
8. Apply any memory write.
9. Repeat.

In pseudocode:

```text
while not halted:
  instruction = program[state.pc]
  operand = attention_read_if_needed(state, instruction)
  output = baked_ffn_for(instruction)(state, operand)
  state = decode_transition(output, state)
```

## One Full Tiny Example

Program:

```asm
.memory 4

LOADI 5
ADD 3
HALT
```

Initial state:

```text
pc=0, acc=0, sp=4, halted=false, memory=[0,0,0,0]
```

### Step 1: `LOADI 5`

- `pc=0` selects the compiled block for `LOADI 5`
- no attention read is needed
- FFN outputs:
  - `next_pc = 1`
  - `next_acc = 5`
  - `next_sp = 4`

New state:

```text
pc=1, acc=5
```

### Step 2: `ADD 3`

- `pc=1` selects the compiled block for `ADD 3`
- no attention read is needed
- FFN outputs:
  - `next_pc = 2`
  - `next_acc = 8`
  - `next_sp = 4`

New state:

```text
pc=2, acc=8
```

### Step 3: `HALT`

- `pc=2` selects the `HALT` block
- FFN preserves `pc`, `acc`, `sp`
- halted flag becomes true

Done.

## A Memory Example

Program:

```asm
.memory 2
.init 1 7

LOAD 1
ADD 3
HALT
```

Flow:

1. `pc=0` selects `LOAD 1`
2. attention reads `MEM[1] = 7`
3. FFN sets `ACC = 7`
4. next step runs `ADD 3`
5. FFN sets `ACC = 10`
6. `HALT`

## A More Complex Example: Stack-Framed Array Sum

See:

- [programs/sum_array_subroutine.tvm](../programs/sum_array_subroutine.tvm)

That program:

- pushes arguments on the stack
- calls a subroutine
- allocates locals with `ADJSP -2`
- uses `LOADR` / `STORER` to manage frame-local variables
- uses `ADDP` to read array elements through a pointer

This is the kind of program that becomes much easier once stack-relative locals exist.

## Another Complex Example: Sorting

See:

- [programs/bubble_sort_four.tvm](../programs/bubble_sort_four.tvm)

That program uses:

- pointer cells
- indirect reads
- indirect stores
- compare-and-branch control flow

to sort four values in memory.

## File-Level Architecture

The most important files are:

- [src/assembly.rs](../src/assembly.rs): parses `.tvm` source into `Program`
- [src/instruction.rs](../src/instruction.rs): instruction enum and program representation
- [src/interpreter.rs](../src/interpreter.rs): native reference semantics
- [src/model.rs](../src/model.rs): transformer-style compiler and runtime pieces
- [src/runtime.rs](../src/runtime.rs): execution loop
- [src/memory.rs](../src/memory.rs): write-history memory model
- [src/proof.rs](../src/proof.rs): vanilla STARK proving path
- [tests/runtime.rs](../tests/runtime.rs): runtime behavior tests
- [tests/e2e.rs](../tests/e2e.rs): full parse -> compile -> execute -> verify tests

## Important Limits

This repo is powerful enough to express much better programs now, but it is still intentionally small.

### 1. It is not trained

Weights are constructed directly from instruction semantics.

### 2. It is not formally Turing complete yet

The current encoded machine state uses:

- `pc: u8`
- `sp: u8`
- `acc: i16`

So the machine is still finite-state.

### 3. Proof support is narrower than execution support

The transformer/native/Burn/ONNX paths can execute the newer instructions, but the vanilla STARK AIR still does not prove all of them yet.

## Bottom Line

The cleanest way to think about the repo is:

- write a `.tvm` program
- compile each instruction into a deterministic transformer block
- use attention as memory lookup
- use FFN weights as the instruction transition
- run state-by-state until `HALT`
- optionally prove the trace

That is the whole system.
