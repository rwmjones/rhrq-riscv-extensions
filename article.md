## RISC-V Extensions

by Richard W.M. Jones

Article for Red Hat Research Quarterly


RISC-V is a new Instruction Set Architecture (ISA) which over the next
decade will compete with x86-64 and ARM in all areas from the lowest
end IoT devices all the way to huge servers.  Unlike existing ISAs it
is a true open source, permissionless architecture.  You can start
making chips using designs downloaded from github without signing any
legal agreement.  If this sounds like Linux versus proprietary
operating systems all over again, then you understand why Red Hat is
interested.  RHRQ has previously covered [RISC-V in
FPGAs](https://research.redhat.com/blog/article/risc-v-for-fpgas-benefits-and-opportunities/)
and [how RISC-V fosters open innovation in
hardware](https://research.redhat.com/blog/article/fostering-open-innovation-in-hardware/).

One unique aspect of RISC-V are **extensions** — as the name suggests
these add extra instructions to the ISA, for implementing features
like vector operations or accelerated encryption.

In this article we'll look at how extensions work at the lowest
levels, how they are ratified and eventually standardized through
RISC-V International (RVI), what extensions are out there, how
extensions are grouped together into profiles, and how you can
discover what extensions are supported in your hardware.  We will also
cover what hardware today supports particular extensions as well as
looking at QEMU's support for extensions if you need to write software
for an extension but it is not available.


### How Extensions are Encoded

RISC-V has a very regular instruction encoding.  For this article I
won't talk much about compressed instructions — see the section below
for more on that.  And I won't discuss the variable-length encodings
wider than 32 bits, since they are not yet used by any extensions.
With those assumptions in mind we can assume the least significant 2
bits of each 32 bit instruction are always `1 1`.  The next 5 bits are
the **major opcode** which control how the instruction is decoded:

```
  major opcode
  000 0011   LOAD
  000 0111   LOAD-FP
  000 1011   custom-0
  000 1111   MISC-MEM
  001 0011   OP-IMM
  001 0111   AUIPC
  001 1011   OP-IMM-32
  001 1111   (48-bit instruction)
  010 0011   STORE
  010 0111   STORE-FP
  010 1011   custom-1
  010 1111   AMO (Atomic extension)
  011 0011   OP
  011 0111   LUI
  011 1011   OP-32
  011 1111   (64-bit instruction)
  100 0011   MADD (fused multiply add)
  100 0111   MSUB ( "" )
  100 1011   NMSUB ( "" )
  100 1111   NMADD ( "" )
  101 0011   OP-FP
  101 0111   OP-V (Vector extension)
  101 1011   custom-2 / RV128I
  101 1111   (48 bit instruction)
  110 0011   BRANCH
  110 0111   JALR
  110 1011   reserved
  110 1111   JAL
  111 0011   SYSTEM
  111 0111   OP-P (Packed SIMD extension)
  111 1011   custom-3 / RV128I
  111 1111   (80 bit and longer instructions)
```

This is not a comprehensive guide to decoding RISC-V instructions, as
that is covered well in the [RISC-V ISA
spec](https://riscv.org/technical/specifications/) but some things are
worth pointing out:

1. Some instructions have further opcodes, for example `LOAD` has
another 3 bits in the middle of the instruction so you can
tell what kind of load it is.

2. Some major opcodes correspond entirely to specific extensions.  For
example `AMO` contains Atomic instructions (the `A` extension) and
`OP-V` was originally reserved but is now used by the Vector
extension.

3. Large sections of the opcode space are used for some fairly obscure
features, like fused-multiply.

4. `custom-0` through `custom-3` are for vendors to add their own
extensions which they don't intend to ratify.  Essentially `custom-*`
is a free-for-all (except that `custom-2` and `custom-3` will
eventually be used by the proposed 128 bit RV128I ISA).

In this article I want to concentrate only on extensions which are
non-custom, non-vendor-specific, and are on the path to ratification
by RISC-V International (RVI).

It is most important to note that (non-custom) extensions are not
distinct, separate parts of the decoding space.  RVI appears to want
to thread extensions into the gaps between existing base instructions.
Even major extensions like Vector which appear to have their own major
opcode (`OP-V`), also have instructions in `LOAD-FP` and elsewhere.
Smaller extensions use existing major opcodes and fit into the gaps.
This means that when proposing a new extension you will need to
consider existing extensions, which is good practice anyway.


#### Byte Ordering of Instructions

Instructions are always stored little endian, with the least
significant byte first in memory, and this applies even on the
(extremely rare) big endian RISC-V machine.  However disassembly tools
like `objdump` swap the bytes back again.  Thus an instruction like
AUIPC appears in documentation and objdump output as:

  imm[31:12]      rd      001 0111

but in memory as:

  rd[0] 001 0111   imm[15:12] rd[4:1]   imm[23:16]   imm[31:24]
      major opcode

In other words when decoding RISC-V instructions you can tell from the
first byte much about how to decode the instruction and where the next
instruction begins.  This is in stark contrast to x86 where simply
determining the boundaries between instructions is a research project
in itself!


### The Classic Extensions



#### The Curious Case of Compressed Encoding


### The New Extensions



### Profiles



### Hardware Support for Extensions




### How Extensions are Implemented by QEMU











