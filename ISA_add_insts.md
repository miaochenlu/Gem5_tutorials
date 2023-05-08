# Add a new instruction in Gem5

[slides](attachments/SW_Prefetch.pptx)

## 1. Overall Steps

The tutorial https://nitish2112.github.io/post/adding-instruction-riscv/ teaches us how to add a simple `modulo` instruction

```
mod r1, r2, r3

Semantics:
R[r1] = R[r2] % R[r3]
```

### 1.1 Gem5's Support
modify `arch/riscv/decoder.isa`, add `mod` in format `ROp`
```
0x33: decode FUNCT3 {
	format ROp {
		0x0: decode FUNCT7 {
			0x0: add({{
				Rd = Rs1_sd + Rs2_sd;
			}});
			0x1: mul({{
				Rd = Rs1_sd * Rs2_sd;
			}}, IntMultOp);
			0x10: mod({{
				Rd = Rs1_sd % Rs2_sd;
			}});
			0x20: sub({{
				Rd = Rs1_sd - Rs2_sd;
			}});
		}
```

### 1.2 Compiler's Support
* install riscv-tools
```
$ git clone https://github.com/riscv/riscv-tools.git
$ git submodule update --init --recursive
$ export RISCV=/path/to/install/riscv/toolchain
$ ./build.sh
```
* modify `riscv-opcodes/opcodes`
```
sra     rd rs1 rs2 31..25=32 14..12=5 6..2=0x0C 1..0=3
or      rd rs1 rs2 31..25=0  14..12=6 6..2=0x0C 1..0=3
and     rd rs1 rs2 31..25=0  14..12=7 6..2=0x0C 1..0=3

mod     rd rs1 rs2 31..25=1  14..12=0 6..2=0x1A 1..0=3

addiw   rd rs1 imm12            14..12=0 6..2=0x06 1..0=3
slliw   rd rs1 31..25=0  shamtw 14..12=1 6..2=0x06 1..0=3
srliw   rd rs1 31..25=0  shamtw 14..12=5 6..2=0x06 1..0=3
sraiw   rd rs1 31..25=32 shamtw 14..12=5 6..2=0x06 1..0=3
```
* add MATCH and MASK to `riscv-gnu-toolchain/riscv-binutils-gdb/include/opcode/riscv-opc.h`
```cpp
#define MATCH_MOD 0x200006b                                                    
#define MASK_MOD 0xfe00707f
```

* modify `riscv-gnu-toolchain/riscv-binutils-gdb/opcodes/riscv-opc.c`
```cpp
const struct riscv_opcode riscv_opcodes[] =                                     
{                                                                               
/* name,      isa,   operands, match, mask, match_func, pinfo.  */              
{"unimp",     "C",   "",  0, 0xffffU,  match_opcode, 0 },                       
{"unimp",     "I",   "",  MATCH_CSRRW | (CSR_CYCLE << OP_SH_CSR), 0xffffffffU,  match_opcode, 0 }, /* csrw cycle, x0 */
{"ebreak",    "C",   "",  MATCH_C_EBREAK, MASK_C_EBREAK, match_opcode, INSN_ALIAS },
{"ebreak",    "I",   "",    MATCH_EBREAK, MASK_EBREAK, match_opcode, 0 },          
{"sbreak",    "C",   "",  MATCH_C_EBREAK, MASK_C_EBREAK, match_opcode, INSN_ALIAS },
{"sbreak",    "I",   "",    MATCH_EBREAK, MASK_EBREAK, match_opcode, INSN_ALIAS },
....
....
....
{"mod",       "I",   "d,s,t",  MATCH_MOD, MASK_MOD, match_opcode, 0 }
....
....
```

## 2. Example: Add RISC-V Prefetch Instructions

## 2.1 RISC-V Prefetch Instructions
* `prefetch.i` (omit in this example)
    * Synopsis: Provide a HINT to hardware that a cache block is likely to be accessed by an instruction fetch in the near future 
    * Mnemonic: `prefetch.i offset(base)`
![](attachments/Pasted%20image%2020230508164924.png)
* `prefetch.r`
	* Synopsis: Provide a HINT to hardware that a cache block is likely to be accessed by a data read in the near future
	* Mnemonic: `prefetch.r offset(base)`
![](attachments/Pasted%20image%2020230508165017.png)
* `prefetch.w`
	* Synopsis: Provide a HINT to hardware that a cache block is likely to be accessed by a data write in the near future
	* Mnemonic: `prefetch.w offset(base)`
![](attachments/Pasted%20image%2020230508165049.png)

## 2.2 Support Prefetch Instructions in Gem5
### a. add prefetch instruction format
1. add `src/arch/riscv/isa/formats/cbo.isa`: define prefetch instruction processing template
```
def template PrefetchDeclare {{
    /**
     * Static instruction class for "%(mnemonic)s".
     */
    class %(class_name)s : public %(base_class)s
    {
      private:
        %(reg_idx_arr_decl)s;

      public:
        /// Constructor.
        %(class_name)s(ExtMachInst machInst);

        Fault execute(ExecContext *, Trace::InstRecord *) const override;
        Fault initiateAcc(ExecContext *, Trace::InstRecord *) const override;
        Fault completeAcc(PacketPtr, ExecContext *,
                          Trace::InstRecord *) const override;
    };
}};

def template PrefetchConstructor {{
    %(class_name)s::%(class_name)s(ExtMachInst machInst):
        %(base_class)s("%(mnemonic)s", machInst, %(op_class)s)
    {
        %(set_reg_idx_arr)s;
        %(constructor)s;
        %(offset_code)s;
    }
}};


def template PrefetchExecute {{
    Fault
    %(class_name)s::execute(ExecContext *xc,
        Trace::InstRecord *traceData) const
    {
        Addr EA;

        %(op_decl)s;
        %(op_rd)s;
        %(ea_code)s;

        readMemAtomicLE(xc, traceData, EA, Mem, memAccessFlags);

        return NoFault;
    }
}};

def template PrefetchInitiateAcc {{
    Fault
    %(class_name)s::initiateAcc(ExecContext *xc,
        Trace::InstRecord *traceData) const
    {
        Addr EA;

        %(op_decl)s;
        %(op_rd)s;
        %(ea_code)s;

        return initiateMemRead(xc, traceData, EA, Mem, memAccessFlags);
    }
}};

def template PrefetchCompleteAcc {{
    Fault
    %(class_name)s::completeAcc(PacketPtr pkt, ExecContext *xc,
        Trace::InstRecord *traceData) const
    {
        return NoFault;
    }
}};


def format PrefetchOp(memacc_code, mem_flags=[], inst_flags=[]) {{
    mem_flags = makeList(mem_flags)
    inst_flags = makeList(inst_flags)
    offset_code = """offset = sext<12>(IMM5 | (IMM7 << 5));"""

    ea_code = """EA = Rs1 + offset;"""
    memacc_code = """Mem_sw"""

    iop = InstObjParams(name, Name, 'PrefetchOp',
        {'memacc_code': memacc_code, 'offset_code': offset_code,
         'ea_code': ea_code}, inst_flags)

    if mem_flags:
        mem_flags = [ 'Request::%s' % flag for flag in mem_flags ]
        s = '\n\tmemAccessFlags = ' + '|'.join(mem_flags) + ';'
        iop.constructor += s

    fullExecTemplate = eval('Prefetch' + 'Execute')
    initiateAccTemplate = eval('Prefetch' + 'InitiateAcc')
    completeAccTemplate = eval('Prefetch' + 'CompleteAcc')

    header_output = PrefetchDeclare.subst(iop)
    decoder_output = PrefetchConstructor.subst(iop)
    decode_block = BasicDecode.subst(iop)
    exec_output =  fullExecTemplate.subst(iop)+initiateAccTemplate.subst(iop)+completeAccTemplate.subst(iop)
}};

```
2. include "cbo.isa" in `src/arch/riscv/isa/formats/formats.isa`
```
##include "cbo.isa"
```

3. define PrefetchOp
add `src/arch/riscv/insts/cbo.hh`
```cpp
#ifndef __ARCH_RISCV_INST_CBO_HH__
#define __ARCH_RISCV_INST_CBO_HH__

#include <string>

#include "arch/riscv/insts/mem.hh"
#include "arch/riscv/insts/static_inst.hh"
#include "cpu/exec_context.hh"
#include "cpu/static_inst.hh"

namespace gem5
{

namespace RiscvISA
{

class PrefetchOp : public MemInst
{
  protected:
    using MemInst::MemInst;

    std::string generateDisassembly(
        Addr pc, const loader::SymbolTable *symtab) const override;
};

} // namespace RiscvISA
} // namespace gem5

#endif // __ARCH_RISCV_INST_CBO_HH__
```

add `src/arch/riscv/insts/cbo.cc`
```cpp
#include "arch/riscv/insts/cbo.hh"

#include <sstream>
#include <string>

#include "arch/riscv/insts/bitfields.hh"
#include "arch/riscv/insts/static_inst.hh"
#include "arch/riscv/utility.hh"
#include "cpu/static_inst.hh"

namespace gem5
{

namespace RiscvISA
{

std::string
PrefetchOp::generateDisassembly(Addr pc,
        const loader::SymbolTable *symtab) const
{
    std::stringstream ss;
    ss << mnemonic << ' ' <<
        offset << '(' << registerName(srcRegIdx(0)) << ')';
    return ss.str();
}

} // namespace RiscvISA
} // namespace gem5
```

modify `src/arch/riscv/insts/SConscript`: add
```
Source('cbo.cc', tags='riscv isa')
```
### a. Decode
![](attachments/Pasted%20image%2020230508170246.png)
For simplicity, in `src/arch/riscv/isa/bitfields.isa`, add PFTYPE fields to figure out prefetch types: `prefetch.i` or `prefetch.r` or `prefetch.w`
```
// Prefetch
def bitfield PFTYPE <24:20>;
```

in `src/arch/riscv/isa/decoder.isa`
```
decode QUADRANT default Unknown::unknown() {
    0x03: decode OPCODE {
        …
        0x04: decode FUNCT3 {
            0x6: decode RD {
                0x00: decode PFTYPE {
                format PrefetchOp {
                    0x0: Prefetch_i({{
                    }}, inst_flags=IsInstPrefetch, mem_flags=PREFETCH);
                    0x1: Prefetch_r({{
                    }}, inst_flags=IsDataPrefetch, mem_flags=PREFETCH);
                    0x3: Prefetch_w({{
                    }}, inst_flags=IsDataPrefetch, mem_flags=PF_EXCLUSIVE);
                    }
                }
                default: IOp::ori({{
                    Rd = Rs1 | imm;
                }}, uint64_t);
            }
        }
```

## 2.3 Modify RISCV-GNU-Toolchain
in `riscv-binutils/include/opcode/riscv-opc.h`
```cpp
#define MATCH_PREFETCH_I 0x6013
#define MASK_PREFETCH_I 0x1f07fff
#define MATCH_PREFETCH_R 0x106013
#define MASK_PREFETCH_R 0x1f07fff
#define MATCH_PREFETCH_W 0x306013
#define MASK_PREFETCH_W 0x1f07fff

DECLARE_INSN(prefetch_i, MATCH_PREFETCH_I, MASK_PREFETCH_I)
DECLARE_INSN(prefetch_r, MATCH_PREFETCH_R, MASK_PREFETCH_R)
DECLARE_INSN(prefetch_w, MATCH_PREFETCH_W, MASK_PREFETCH_W)
```
in `riscv-binutils/include/opcode/riscv-opc.c`
```cpp
const struct riscv_opcode riscv_opcodes[] =
{
{"prefetch.i",64, INSN_CLASS_I, "q(s)",      MATCH_PREFETCH_I, MASK_PREFETCH_I, match_opcode, 0 },
{"prefetch.r",64, INSN_CLASS_I, "q(s)",      MATCH_PREFETCH_R, MASK_PREFETCH_R, match_opcode, 0 },
{"prefetch.w",64, INSN_CLASS_I, "q(s)",      MATCH_PREFETCH_W, MASK_PREFETCH_W, match_opcode, 0 },
...
}
```

## 2.4 Test
### a. Test RISCV-GNU-Toolchain
write a sample code
```cpp
int main() {
    int a;
    asm volatile ("prefetch.r %[addr]": : [addr]"A"(a));
}
```
Compiler produces:
```
0000000000010146 <main>:
   10146:	1101                	addi	sp,sp,-32
   10148:	ec22                	sd	s0,24(sp)
   1014a:	1000                	addi	s0,sp,32
   1014c:	fec40793          	    addi	a5,s0,-20
   10150:	0017e013          	    prefetch.r	0(a5)
   10154:	4781                	li	a5,0
   10156:	853e                	mv	a0,a5
   10158:	6462                	ld	s0,24(sp)
   1015a:	6105                	addi	sp,sp,32
   1015c:	8082                	ret

```

Successfully generated prefetch instructions.

### b. Test Gem5
use gdb
add breakpoints
* b `Prefetch_r::initiateAcc`
* b `Prefetch_r::completeAcc`
![](attachments/Pasted%20image%2020230508165513.png)

### c. Matrix Addition Test
```cpp
#include <stdlib.h>
#include <gem5/m5ops.h>
const int MATRIX_SIZE = 100;
int main() {
        int** matA = (int**)malloc(MATRIX_SIZE * sizeof(int*));
        int** matB = (int**)malloc(MATRIX_SIZE * sizeof(int*));
        int** matC = (int**)malloc(MATRIX_SIZE * sizeof(int*));
        for(int i = 0; i < MATRIX_SIZE; i++) {
                matA[i] = (int*)malloc(MATRIX_SIZE * sizeof(int));
                matB[i] = (int*)malloc(MATRIX_SIZE * sizeof(int));
                matC[i] = (int*)malloc(MATRIX_SIZE * sizeof(int));
        }
m5_reset_stats(0, 0);
        for(int i = 0; i < MATRIX_SIZE; i++) {
                for(int j = 0; j < MATRIX_SIZE; j++) {
                    asm volatile ("prefetch.r %[addr]": 
		: [addr]"A"(matA[i][j + stride]));
                    asm volatile ("prefetch.r %[addr]": 
		: [addr]"A"(matB[i][j + stride]));
                    asm volatile ("prefetch.w %[addr]": 
		: [addr]"A"(matC[i][j + stride]));
                        matC[i][j] = matA[i][j] + matB[i][j];
                }
        }
m5_dump_stats(0, 0);
        ...
}
```
Test results is shown in the following table:
| Settings                         | Clock Cycles |
|----------------------------------|--------------|
| No Prefetch                      | 260038       |
| Stride 16 / 3 prefetch           | 195276       |
| Stride 16 / 2 prefetch (no pf.w) | 173609       |
| Stride 16/ 1 prefetch            | 214953       |
| Stride 8/3 prefetch              | 195229       |
| Stride 8/2 prefetch (no pf.w)    | 180388       |
| Stride 8/ 1 prefetch             | 220065       |
| Stride 1/3 prefetch              | 389572       |
| Stride 1/ 2 Prefetch (no pf.w)   | 292908       |
| Stride 1/ 1 Prefetch             | 389572       |


## References
* [使用Gem5自定义RISC-V指令集](https://junningwu.haawking.com/tech/2019/11/28/%E4%BD%BF%E7%94%A8Gem5%E8%87%AA%E5%AE%9A%E4%B9%89RISC-V%E6%8C%87%E4%BB%A4%E9%9B%86-%E6%8C%81%E7%BB%AD%E6%9B%B4%E6%96%B0/)
* [Adding custom instruction to RISCV ISA and running it on gem5 and spike | Nitish Srivastava](https://nitish2112.github.io/post/adding-instruction-riscv/)
