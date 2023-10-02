## The Use of RISC-V Compressed Instructions in Linux Distributions

by Richard W.M. Jones


RISC-V uses a variable sized instruction length.  While the vast
majority of instructions are 32 bits long, 48 bit and longer
instructions are possible (though none are defined at the time of
writing).  The topic of this article is the various 16 bit compressed
instructions — the C major extension, as well as those defined by Zca,
Zcb, Zcf, Zcd, Zce, Zcmp, Zcmt and others.

In particular this article is about what packages, tools and
programming languages may generate compressed instructions and how the
feature might be detected, enabled or disabled in each case.

The background to this topic is that although in 2023 most Linux
distributions for RISC-V use compressed instructions — indeed it is
part of the canonical and now venerable RV64GC standard — we expect in
future that there may be high end server chips which ship without this
feature, since it makes high performance designs more complicated.
Therefore it is useful to have a survey of how compressed instructions
are currently used and enabled, and how they might be disabled in
future.


### Statically detecting compressed instructions in a binary

For a longer discussion of how instructions are encoded, please see
the RISC-V User Spec.

In brief the bottom two bits of the least significant byte of an
instruction tell you if it is not compressed (both bits are `1`), or
compressed (`0 0`, `0 1` or `1 0`).  To detect longer than 32 bit
instructions you also have to look at the major opcode in the same
byte:

```
x001 1111   48-bit instruction
x011 1111   64-bit instruction
x101 1111   48 bit instruction
x111 1111   80 bit and longer instructions
xxxx xx00   16-bit compressed instruction
xxxx xx01   16-bit compressed instruction
xxxx xx10   16-bit compressed instruction
xxxx xxxx   anything else is a normal 32 bit instruction
```

Note that in documentation and in disassembler output, instructions
are shown in big endian format, so the least significant byte is the
last one, but in the actual binary, bytes are always stored little
endian so the first byte in the instruction stream is the least
significant byte (and this is true even on the theoretical big endian
RISC-V machine).

In disassembly output the shorter instruction length is obvious:

```
4922:       e4e8                    sd      a0,200(s1)
4924:       479d                    li      a5,7
4926:       2af902e3                beq     s2,a5,53ca
```

Note the two compressed instructions have least significant byte `e8`
(`xxxx xx00`) and `9d` (`xxxx xx01`).


### Dynamically detecting compressed instructions

While statically examining executables for compressed instructions
gets you some way there, there are JIT code generators which could
generate compressed instructions dynamically.  How should we detect
them?

At least in theory it would be possible to reset the `C` bit in the
MISA CSR to disable compressed instructions and cause them to trap,
but I don't know of any actual hardware that can do this.

QEMU can emulate RISC-V in software (and the emulation is quite
complete, in fact in 2023 it is more complete than any hardware
available).  So a better plan is to modify QEMU so it traps on (or
reports) any compressed instructions when it translates them, a simple
modification.  This would allow distros to be booted and applications
run to check dynamically if compressed instructions are used.


### binutils

The binutils configure `--with-arch` option can be used to set the
default architecture and extensions, eg:

```
./configure --with-arch=rv64gv
```

would disable the compressed extension by default.  If not given then
it appears to be detected from the build machine somehow although the
exact mechanism is unclear.


### gas (GNU assembler) syntax

By default, gas determines if compressed instructions are supported by
using the binutils default, and then using the `-march` command line
flag.

If compressed instructions are supported, gas will "opportunistically
compress" instructions (meaning that if an instruction has a
compressed equivalent it will emit that).  Per-file you can enable or
disable this using:

```
  .option rvc  ; enable opportunistic compressed instructions
  .option arch, +c ; same as rvc
  .option norvc ; disable
  .option arch, -c ; same as norvc
```

ELF object files that *could* contain compressed instructions have the
`EF_RISCV_RVC == 0x0001` flag set.

This does *not* mean they do contain compressed instruction, only that
the gas configuration and command line flags meant that compressed
instructions could have been emitted.  Even then I found that simply
using assembler directives would not turn off the flag.  (The code
only ever sets the flag, so unless you use `gas -march=rv64g` on the
command line, the flag will be set despite your assembler file
containing the right directives to turn off compressed instructions
and no compressed instructions actually being generated.)

A file with this flag set looks like:

```
$ file test.o
test.o: ELF 64-bit LSB relocatable, UCB RISC-V, RVC, double-float ABI, version 1 (SYSV), not stripped
$ eu-readelf -h test.o
ELF Header:
...
  Flags:                             0x5
```

A file without this flag set looks like:

```
$ file test.o
test.o: ELF 64-bit LSB relocatable, UCB RISC-V, double-float ABI, version 1 (SYSV), not stripped
$ eu-readelf -h test.o
...
  Flags:                             0x4
```

Compressed instructions can also be generated by explicit opcode, eg.
this generates the compressed instruction explicitly:

```
  c.li    a5,0
```

However if the `-march` setting does not support compressed, gas will
correctly give an error instead of generating the instruction:

```
$ gcc -c -Wa,-march=rv64g test.s
test.s: Assembler messages:
test.s:19: Error: unrecognized opcode `c.li a5,0', extension `c' required
```

### GCC


### LLVM and Clang


### Linux kernel


#### eBPF JIT


### QEMU



### Golang


### Rust


### OCaml


### Javascript
