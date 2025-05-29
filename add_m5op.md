# Adding Custom m5op in gem5 (Xiangshan RISC-V)

## 1. Overview

m5op is a special mechanism provided by gem5 that allows simulated programs to interact with the simulator. Through m5op, user programs can trigger specific simulator functions, such as enabling/disabling prefetchers, collecting statistics, creating checkpoints, etc. This document, based on the successfully implemented Prodigy prefetcher m5op, details how to add custom m5op for RISC-V architecture in gem5.

## 2. m5op Working Principle and Call Flow

### 2.1 Basic Principle

m5op implements communication between user programs and the simulator through special pseudo instructions:
- User program calls m5op function (e.g., `m5_prodigy_enable()`)
- This function actually executes a special RISC-V instruction
- gem5's CPU model recognizes and intercepts this special instruction
- Triggers the corresponding handler function inside the simulator

### 2.2 Complete Call Chain

When a user program calls `m5_prodigy_enable()`, it goes through the following flow:

#### Step 1: User Space Call
```c
// User program
m5_prodigy_enable();
```

#### Step 2: Assembly Instruction Execution
```assembly
// util/m5/src/abi/riscv/m5op.S
m5_prodigy_enable:
    .long 0x0000007b | (0x71 << 25)  // 0x71 is M5OP_PRODIGY_ENABLE
    ret
```

This generates a special RISC-V instruction:
- Base encoding: `0x0000007b`
- Function code (bit 31:25): `0x71`
- Final instruction: `0x8E00007B`

#### Step 3: CPU Execution and Decoding

When the CPU encounters this instruction:

1. **Instruction Decoding**: RISC-V decoder recognizes opcode `0x1e` (bit 6:2 = 30)
   ```
   // src/arch/riscv/isa/decoder.isa
   0x1e: M5Op::M5Op();
   ```

2. **M5Op Format Processing**: Decoder calls M5Op format definition
   ```
   // src/arch/riscv/isa/formats/m5ops.isa
   def format M5Op() {{
       iop = InstObjParams(name, Name, 'PseudoOp', '''
               uint64_t result;
               pseudo_inst::pseudoInst<RegABI64>(xc->tcBase(), M5FUNC, result);
               a0 = result''',
               ['IsNonSpeculative', 'IsSerializeAfter'])
   ```

#### Step 4: PseudoInst Dispatch
```cpp
// src/sim/pseudo_inst.hh
template <typename ABI, bool store_ret>
bool
pseudoInst(ThreadContext *tc, uint64_t &ret_val)
{
    uint64_t func;
    PseudoInstABI::decodeArg<uint64_t>(tc, func); // func = 0x71
    
    switch (func) {
        case M5OP_PRODIGY_ENABLE:  // 0x71
            invokeSimcall<ABI>(tc, prodigyEnable);
            return true;
        // ...
    }
}
```

#### Step 5: Execute Specific Handler Function
```cpp
// src/sim/pseudo_inst.cc
void
prodigyEnable(ThreadContext *tc)
{
    DPRINTF(PseudoInst, "PseudoInst::prodigyEnable()\n");
    warn("Prodigy: Prefetcher enabled\n");
    
    // Actual processing logic
    System *sys = tc->getSystemPtr();
    // ... Code to find and enable prefetcher
}
```

### 2.3 Data Flow Diagram

```
User Program                Assembly Layer              gem5 Simulator
------------                --------------              --------------
m5_prodigy_enable() -----> Special Inst 0x8E00007B ----> CPU Decoder
                                                            |
                                                            v
                                                        Recognize opcode 0x1e
                                                            |
                                                            v
                                                        M5Op Format Processing
                                                            |
                                                            v
                                                        Extract M5FUNC=0x71
                                                            |
                                                            v
                                                        pseudoInst() Dispatch
                                                            |
                                                            v
                                                        prodigyEnable(tc)
                                                            |
                                                            v
                                                        Execute Actual Operation
```

### 2.4 RISC-V Architecture Specific Implementation

RISC-V uses special instruction encoding to implement m5op:
- **Base instruction**: `0x0000007b`
- **Function code position**: bit 31:25 (M5FUNC)
- **OPCODE**: bit 6:2 = 0x1e (decimal 30)

## 3. Core File Structure

Before implementing custom m5op, you need to understand the core files involved and their roles:

### 3.1 User Space Library Files

| File Path | Function |
|-----------|----------|
| `include/gem5/m5ops.h` | Defines m5op function prototypes for user program calls |
| `include/gem5/asm/generic/m5ops.h` | Defines m5op opcode constants |
| `util/m5/src/abi/riscv/m5op.S` | RISC-V architecture m5op assembly implementation |

### 3.2 gem5 Simulator Side Files

| File Path | Function |
|-----------|----------|
| `src/sim/pseudo_inst.hh` | Declares pseudo instruction handler functions |
| `src/sim/pseudo_inst.cc` | Implements pseudo instruction handling logic |
| `src/arch/riscv/isa/decoder.isa` | RISC-V instruction decoder configuration |
| `src/arch/riscv/isa/formats/m5ops.isa` | M5Op instruction format definition |

## 4. Implementation Steps

### 4.1 Step 1: Define Opcodes

Add new opcodes in `include/gem5/asm/generic/m5ops.h`:

```cpp
// Prodigy prefetcher operations
#define M5OP_PRODIGY_ENABLE          0x71
#define M5OP_PRODIGY_DISABLE         0x72
#define M5OP_PRODIGY_REGISTER_NODE   0x73
#define M5OP_PRODIGY_REGISTER_EDGE   0x74
#define M5OP_PRODIGY_REGISTER_TRIGGER 0x75
#define M5OP_PRODIGY_PRINT_DIG       0x76

// Add to M5OP_FOREACH macro
#define M5OP_FOREACH                                            \
    // ... existing ops ...                                    \
    M5OP(m5_prodigy_enable, M5OP_PRODIGY_ENABLE)                \
    M5OP(m5_prodigy_disable, M5OP_PRODIGY_DISABLE)              \
    M5OP(m5_prodigy_register_node, M5OP_PRODIGY_REGISTER_NODE)  \
    M5OP(m5_prodigy_register_edge, M5OP_PRODIGY_REGISTER_EDGE)  \
    M5OP(m5_prodigy_register_trigger, M5OP_PRODIGY_REGISTER_TRIGGER)  \
    M5OP(m5_prodigy_print_dig, M5OP_PRODIGY_PRINT_DIG)
```

### 4.2 Step 2: Declare User Space Functions

Add function declarations in `include/gem5/m5ops.h`:

```cpp
// Prodigy prefetcher operations
void m5_prodigy_enable(void);
void m5_prodigy_disable(void);
void m5_prodigy_register_node(uint64_t params_addr);
void m5_prodigy_register_edge(uint64_t params_addr);
void m5_prodigy_register_trigger(uint64_t params_addr);
void m5_prodigy_print_dig(void);
```

### 4.3 Step 3: Implement gem5 Side Handler Functions

Declare handler functions in `src/sim/pseudo_inst.hh`:

```cpp
// Prodigy pseudo instructions
void prodigyRegisterNode(ThreadContext *tc, uint64_t params_addr);
void prodigyRegisterEdge(ThreadContext *tc, uint64_t params_addr);
void prodigyRegisterTrigger(ThreadContext *tc, uint64_t params_addr);
void prodigyEnable(ThreadContext *tc);
void prodigyDisable(ThreadContext *tc);
void prodigyPrintDIG(ThreadContext *tc);
```

Implement specific logic in `src/sim/pseudo_inst.cc`:

```cpp
void
prodigyEnable(ThreadContext *tc)
{
    DPRINTF(PseudoInst, "PseudoInst::prodigyEnable()\n");
    warn("Prodigy: Prefetcher enabled\n");
    
    // Actual enable logic
    // e.g., Find Prodigy prefetcher and call its enable method
}

void
prodigyRegisterNode(ThreadContext *tc, uint64_t params_addr)
{
    DPRINTF(PseudoInst, "PseudoInst::prodigyRegisterNode(0x%x)\n", params_addr);
    
    // Read parameters from memory
    auto proxy = tc->getVirtProxy();
    ProdigyNodeParams params;
    proxy.readBlob(params_addr, &params, sizeof(params));
    
    // Process parameters...
}
```

### 4.4 Step 4: Register to Dispatcher

Add cases in the `pseudoInst` template function in `src/sim/pseudo_inst.hh`:

```cpp
template <typename ABI, bool store_ret>
bool
pseudoInst(ThreadContext *tc, uint64_t &ret_val)
{
    uint64_t func;
    PseudoInstABI::decodeArg<uint64_t>(tc, func);

    switch (func) {
        // Prodigy pseudo instructions
        case M5OP_PRODIGY_ENABLE:
            invokeSimcall<ABI>(tc, prodigyEnable);
            return true;

        case M5OP_PRODIGY_DISABLE:
            invokeSimcall<ABI>(tc, prodigyDisable);
            return true;

        case M5OP_PRODIGY_PRINT_DIG:
            invokeSimcall<ABI>(tc, prodigyPrintDIG);
            return true;
        
        // ... other cases ...
    }
}
```

## 5. Build Process

### 5.1 Build m5 Tool Library

```bash
cd util/m5
scons build/riscv/out/m5
```

### 5.2 Build gem5

```bash
cd ../..
scons build/RISCV/gem5.opt -j8
```

## 6. Usage Example

### 6.1 Write User Program

```c
#include <stdio.h>
#include <stdint.h>
#include <gem5/m5ops.h>

// Prodigy parameter structure
typedef struct {
    uint64_t node_base;
    uint64_t node_size;
    uint64_t node_type_size;
    uint64_t node_id;
} ProdigyNodeParams;

int main() {
    printf("=== Prodigy Simple DIG Test ===\n");
    
    // Create test data
    int data_array[100];
    for (int i = 0; i < 100; i++) {
        data_array[i] = i * 10;
    }
    
    // Enable Prodigy
    printf("Enabling Prodigy...\n");
    m5_prodigy_enable();
    
    // Register node
    ProdigyNodeParams params = {
        .node_base = (uint64_t)data_array,
        .node_size = 100 * sizeof(int),
        .node_type_size = sizeof(int),
        .node_id = 1
    };
    m5_prodigy_register_node((uint64_t)&params);
    
    // Perform computation...
    
    // Print DIG state
    m5_prodigy_print_dig();
    
    // Disable Prodigy
    m5_prodigy_disable();
    
    return 0;
}
```

### 6.2 Compile User Program

Create Makefile:

```makefile
GEM5_PATH=/path/to/gem5
TARGET_ISA=riscv

CC=riscv64-unknown-linux-gnu-gcc
CFLAGS=-I$(GEM5_PATH)/include
LDFLAGS=-L$(GEM5_PATH)/util/m5/build/$(TARGET_ISA)/out -lm5

test_prodigy: test_prodigy.c
	$(CC) $(CFLAGS) -o $@ $< $(LDFLAGS)
```

### 6.3 Run Test

```bash
./build/RISCV/gem5.opt configs/example/se.py \
    --cpu-type=DerivO3CPU \
    -c /path/to/test_prodigy
```

**Note**: This uses SE (System-call Emulation) mode

## 7. Creating m5op Test Programs in nexus-am

[nexus-am](https://github.com/OpenXiangShan/nexus-am) is an embedded system abstraction layer framework that can compile bare-metal programs, making it ideal for system-level testing on gem5. Below is the complete process for creating and running m5op test programs in nexus-am.

### 7.1 nexus-am Environment Setup

nexus-am has integrated m5 library support. You can see the relevant configuration in `nexus-am/Makefile.app`:

```makefile
# Define the m5 library path
M5_LIB = ${M5_PATH}/build/riscv/out/libm5.a

# Link files include M5_LIB
LINK_FILES = \
  $(OBJS) \
  $(AM_HOME)/am/build/am-$(ARCH).a \
  ... \
  $(M5_LIB)
```

**Note**: You need to set the following environment variables:
```bash
export GEM5_PATH=/path/to/gem5
export M5_PATH=$GEM5_PATH/util/m5
export AM_HOME=/path/to/nexus-am
```

### 7.2 Create New Test Program

#### Step 1: Create Program Directory
```bash
cd nexus-am/apps
mkdir my_m5op_test
cd my_m5op_test
```

#### Step 2: Write Main Program (main.c)
```c
#include <klib.h>
#include <stdio.h>
#include <stdint.h>
#include <gem5/m5ops.h>

// Define test data structure
typedef struct {
    uint64_t node_base;
    uint64_t node_size;
    uint64_t node_type_size;
    uint64_t node_id;
} ProdigyNodeParams;

int main() {
    printf("=== My M5op Test Program ===\n");
    
    // Create test arrays
    int index_array[100];
    int data_array[100];
    
    // Initialize arrays
    for (int i = 0; i < 100; i++) {
        index_array[i] = (i * 7) % 100;  // Create indirect access pattern
        data_array[i] = i * 10;
    }
    
    printf("Arrays initialized:\n");
    printf("  index_array at %p, size=%d\n", index_array, 100);
    printf("  data_array at %p, size=%d\n", data_array, 100);
    
    // Enable Prodigy prefetcher
    printf("\nEnabling Prodigy prefetcher...\n");
    m5_prodigy_enable();
    
    // Register data structure node
    ProdigyNodeParams params = {
        .node_base = (uint64_t)data_array,
        .node_size = 100 * sizeof(int),
        .node_type_size = sizeof(int),
        .node_id = 1
    };
    m5_prodigy_register_node((uint64_t)&params);
    
    // Perform memory accesses
    printf("\nPerforming memory accesses:\n");
    for (int i = 0; i < 5; i++) {
        int idx = index_array[i];
        int val = data_array[idx];
        printf("  data_array[%d] = %d\n", idx, val);
    }
    
    // Print DIG state
    printf("\nPrinting DIG state:\n");
    m5_prodigy_print_dig();
    
    // Disable Prodigy
    printf("\nDisabling Prodigy prefetcher...\n");
    m5_prodigy_disable();
    
    // Use m5_exit to exit simulation normally
    printf("\n=== Test Complete ===\n");
    m5_exit(0);
    
    return 0;
}
```

#### Step 3: Create Makefile
```makefile
NAME = my_m5op_test
SRCS = main.c

# Add gem5 include directory <gem5/m5ops.h> here
INC_DIR += $(GEM5_PATH)/include/

include $(AM_HOME)/Makefile.app
```

### 7.3 Compile Program

```bash
# Set environment variables
export AM_HOME=/path/to/nexus-am
export GEM5_PATH=/path/to/gem5
export M5_PATH=$GEM5_PATH/util/m5

# Compile RISC-V 64-bit version
make ARCH=riscv64-xs

# Generated binary file is located at
# build/my_m5op_test-riscv64-xs.bin
```

### 7.4 Run on gem5

Use the following command to run the compiled program on gem5:

```bash
./build/RISCV/gem5.opt ./configs/example/fs.py \
    --num-cpus=1 --caches --l2cache --cpu-type=DerivO3CPU \
    --mem-type=DRAMsim3 --dramsim3-ini=xiangshan_DDR4_8Gb_x8_2400.ini --mem-size=8GB \
    --cacheline_size=64 --l1i_size=32kB --l1i_assoc=8 \
    --l1d_size=128kB --l1d_assoc=8 \
    --l2_size=512kB --l2_assoc=8 \
    --l3cache --l3_size=2MB --l3_assoc=16 \
    --generic-rv-cpt /path/to/nexus-am/apps/my_m5op_test/build/my_m5op_test-riscv64-xs.bin \
    --xiangshan-system --raw-cpt
```

**Note**: nexus-am generates bare-metal programs that need to run in FS (Full System) mode.

### 7.5 Expected Output

After successful execution, you should see in the gem5 output:

```
**** REAL SIMULATION ****
build/RISCV/sim/simulate.cc:194: info: Entering event queue @ 0.  Starting simulation...
=== My M5op Test Program ===
Arrays initialized:
  index_array at 0x80001930, size=100
  data_array at 0x80001AC0, size=100

Enabling Prodigy prefetcher...
build/RISCV/sim/pseudo_inst.cc:636: warn: Prodigy: Prefetcher enabled

Performing memory accesses:
  data_array[0] = 0
  data_array[7] = 70
  data_array[14] = 140
  data_array[21] = 210
  data_array[28] = 280

Printing DIG state:
build/RISCV/sim/pseudo_inst.cc:650: warn: Prodigy: Print DIG requested

Disabling Prodigy prefetcher...
build/RISCV/sim/pseudo_inst.cc:643: warn: Prodigy: Prefetcher disabled

=== Test Complete ===
Exiting @ tick XXX because m5_exit instruction encountered
```

## 8. References

- [gem5 Official M5ops Documentation](https://www.gem5.org/documentation/general_docs/m5ops/)
- [gem5 Memory System Documentation](https://www.gem5.org/documentation/general_docs/memory_system/)
- [nexus-am Project Homepage](https://github.com/NJU-ProjectN/nexus-am)
- [NPB (NAS Parallel Benchmarks) on gem5](https://gem5.googlesource.com/public/gem5-resources/+/refs/heads/stable/src/npb) - Another example using m5op 
