# Gem5
The clonned repository for the gem5 computer-system architecture simulator with new assembly instructions added for RISCV


## Added Instructions
- mod
- factx
- fact (alias)
- comb

| Parameter | Length | mod instr | factx instr | comb instr |
| - | - | - | - | - |
| MASK | 32 | 0xfe00707f | 0x707f | 0xfe00707f |
| MATCH | 32 | 0x6b | 0x106b | 0x200006b |
| OPCODE | 7 | 0x6b | 0x6b | 0x6b |
| QUARDRANT | 2 | 0x3 | 0x3 | 0x3 |
| OPCODE5 | 5 | 0x1a | 0x1a | 0x1a |
| FUNC3 | 3 | 0x0 | 0x1 | 0x0 |
| FUNC7 | 7 | 0x0 | - | 0x1 |

- The values for `QUARDRANT`, `OPCODE5`, `FUNC3` and `FUNC7` were decided after determining the empty slots in `gem5/src/arch/riscv/isa/decoder.isa` file.
- `OPCODE` = `OPCODE5` + `QUARDRANT`
- `MATCH` = `<instruction>` & `MASK`


## RISCV Instructions Format
| Type | Format | MASK |
| - | - | - |
| R | FUNC7 + RS2 + RS1 + FUNC3 + RD + OPCODE | 0xfe00707f |
| I | IMMI + RS1 + FUNC3 + RD + OPCODE | 0x707f |


## Building riscv-gnu-toolchain
### Installing dependencies
```
sudo apt install autoconf automake autotools-dev curl python3 python3-pip libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev ninja-build git cmake libglib2.0-dev
```

### Clone
```
git clone https://github.com/jameelkaisar/riscv-gnu-toolchain.git --recursive
```

### Compilation
```
cd riscv-gnu-toolchain
./configure --prefix=/opt/riscv --host=riscv64-unknown-elf
sudo make clean
sudo make -j `nproc`
```


## Building gem5
### Installing dependencies
```
sudo apt install build-essential git m4 scons zlib1g zlib1g-dev libprotobuf-dev protobuf-compiler libprotoc-dev libgoogle-perftools-dev python3-dev python-is-python3 libboost-all-dev pkg-config
```

### Clone
```
git clone https://github.com/jameelkaisar/gem5.git
```

### Compilation
```
cd gem5
pip install -r requirements.txt
scons build/RISCV/gem5.opt -j `nproc`
```


## Generating results
### Compilation
```
// hello.c

#include <stdio.h>

int mod(int a, int b) {
    int c;
    asm volatile(
        "mod %[z], %[x], %[y]\n"
        : [z] "=r" (c)
        : [x] "r" (a), [y] "r" (b)
    );
    return c;
}

int fact(int n) {
    int n_f;
    asm volatile(
        "fact %[z], %[x]\n"
        : [z] "=r" (n_f)
        : [x] "r" (n)
    );
    return n_f;
}

int comb(int n, int r) {
    int result;
    asm volatile(
        "comb %[z], %[x], %[y]\n"
        : [z] "=r" (result)
        : [x] "r" (n), [y] "r" (r)
    );
    return result;
}

int main() {
    printf("mod(10, 3) = %d", mod(10, 3));
    printf("fact(12) = %d", fact(12));
    printf("comb(10, 3) = %d", comb(10, 3));
    return 0;
}
```

```
# Testing

/opt/riscv/bin/riscv64-unknown-elf-gcc hello.c -c -o hello.o
/opt/riscv/bin/riscv64-unknown-elf-objdump -D hello.o
```

```
# Compilation

/opt/riscv/bin/riscv64-unknown-elf-gcc hello.c -static -o hello.out
```

### Simulation
```
# simulate.py

import m5
from m5.objects import *
import os

thispath = os.path.dirname(os.path.realpath(__file__))
binary = os.path.join(thispath, "hello.out")

system = System()
system.clk_domain = SrcClockDomain()
system.clk_domain.clock = "1GHz"
system.clk_domain.voltage_domain = VoltageDomain()
system.mem_mode = "timing"
system.mem_ranges = [AddrRange("512MB")]
system.cpu = RiscvTimingSimpleCPU()
system.membus = SystemXBar()
system.cpu.icache_port = system.membus.cpu_side_ports
system.cpu.dcache_port = system.membus.cpu_side_ports
system.cpu.createInterruptController()
system.mem_ctrl = MemCtrl()
system.mem_ctrl.dram = DDR3_1600_8x8()
system.mem_ctrl.dram.range = system.mem_ranges[0]
system.mem_ctrl.port = system.membus.mem_side_ports
system.system_port = system.membus.cpu_side_ports
system.workload = SEWorkload.init_compatible(binary)

process = Process()
process.cmd = [binary]
system.cpu.workload = process
system.cpu.createThreads()

root = Root(full_system=False, system=system)
m5.instantiate()
m5.simulate()
```

```
# Simulation

/path-to-gem5-folder/build/RISCV/gem5.opt simulate.py
```


## Results
TEST: Finding all terms in the binomial expansion of `(A+X)^n`

| Parameter | dafault | fact instr | comb instr |
| - | - | - | - |
| simSeconds | 0.001475 | 0.001279 | 0.001262 |
| simTicks | 1475213000 | 1279116000 | 1262437000 |
| simInsts | 18669 | 16549 | 16379 |
| system.cpu.cpi | 78.968631 | 77.236640 | 77.020133 |
