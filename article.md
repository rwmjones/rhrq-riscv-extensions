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
        __
          LSB 1 1 = not compressed
```

This is not a comprehensive guide to decoding RISC-V instructions, as
that is covered well in the [RISC-V ISA
spec](https://riscv.org/technical/specifications/) but some things are
worth pointing out:

1. Some instructions have further opcode fields, for example `OP`
(`0110011`) has two extra opcode fields totalling another 10 bits.
This major opcode is shared with the base ISA, the Multiply extension,
the Zicond extension, the Packed SIMD extension, and more.  There is
not a general way to tell if a particular opcode corresponds to the
base ISA or an extension (without a large table).

2. Some major opcodes do correspond entirely to specific extensions.
For example `AMO` contains Atomic instructions (the `A` extension) and
`OP-V` was originally reserved but is now used by the Vector
extension.

3. Large sections of the opcode space are used for some fairly obscure
features, like fused multiply-add.

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

```
  imm[31:12]      rd      001 0111
```

but in memory as:

```
  rd[0] 001 0111   imm[15:12] rd[4:1]   imm[23:16]   imm[31:24]
      major opcode
```

In other words when decoding RISC-V instructions you can tell from the
first byte much about how to decode the instruction and where the next
instruction begins.  This is in stark contrast to x86 where simply
determining the boundaries between instructions is a research project
in itself!


### The Classic Extensions

RISC-V was originally envisaged as the base ISA — RV32I, RV64I or
RV128I — surrounded by what I will call the "classic" extensions:

```
 I     (Base integer instructions, add, subtract, jump etc)
 M     Multiply and divide
 A     Atomic operations
 F     Single-precision floating point arithmetic
 D     Double-precision floating point arithmetic
 C     Compressed instructions
```

Notice how some of these map fairly well to the major opcodes, for
example Atomic operations are implemented in the `AMO` opcode space.

We might add:

- `G` an early attempt to define a basic profile, `G` = `IMAFD`
- `E` a still not ratified embedded subset with 16 base registers
  instead of the usual 32,
- `Q` quad-precision floating point,
- `H`, `U` and `S` which are not really extensions but used in
  the misa CSR (see below) to refer to support for hypervisor, user
  and supervisor modes, so cannot be used for extension names
- the `X`-prefix for custom extensions
- the `Z`-prefix

It was soon obvious that we were going to run out of single letters
quickly, so three prefixes were reserved for named extensions, those
are `S` (eg. `Smmtt`) for supervisor-mode extensions, `Z`
(eg. `Zimop`) for general extensions, and `X` for custom,
vendor-specific extensions.

`Zicsr` and `Zifencei` were retrospectively detached from the base ISA
after it was realised that CSRs might not be present on very low end
hardware, and the `FENCE.I` instruction didn't work very well.

One extension may also require another, eg. `D` requires `F`.

#### Naming extensions

There is a well-defined naming scheme describing what extensions are
supported by hardware, and later in the article I'll talk about how
you can find this out for your hardware (or QEMU).  At the time of
writing in 2023, the vast majority of hardware can be described as:

```
RV64IMAFDCZicsr_Zifencei
```

Extension versions can also be encoded here (eg.  `RV64I1p0` would be
base ISA version 1.0).

#### The Curious Case of Compressed Encoding

If you were paying close attention to how RISC-V extensions are
encoded you will see that I assumed 32 bit, non-compressed
instructions.  Current RISC-V implementations support a "compressed",
16 bit encoding for common instructions, of course limited in the
range of registers that may be accessed and the opcodes available.
This is similar in spirit to armv7 Thumb instructions.

The downside to this is that compressed instructions consume ¾ of the
available opcode space (non-compressed, 32 bit instructions must have
both least significant bits `1 1`).  Also instructions are no longer
automatically 4 byte aligned, and may also cross page boundaries,
making decoding harder, albeit still much easier than crazy
architectures like x86.

High end RISC-V server vendors are pushing back against supporting
compressed instructions, arguing that their machines will have huge
instruction caches (so code size is not critical), would prefer to use
the opcode space in other ways, and would like to have uniformly sized
instructions.

We have yet to see how this one will pan out, but don't be surprised
if servers appear that don't implement the `C` extension.

This is very much a concern for Red Hat.  `C` is a special extension
as these instructions appear frequently in binaries — as many as half
of all instructions can be compressed — trap and emulate would be
impossible.  Shipping two distro variants with and without compressed
instructions is not attractive.  Thus we must decide whether to
require it in hardware or ban it in software.


### The New Extensions







### Profiles





### Discovering What Extensions Are Available — a.k.a where's my CPUID?

x86 has CPUID, a comprehensive method to detect at runtime what
features the processor supports, and many other aspects of the CPU
(like cache sizes and so forth).  There is nothing this comprehensive
available in RISC-V at the moment.

For RISC-V, there are three ways to determine what extensions are
available in the hardware.  The oldest mechanism, now mostly
deprecated, is to read the `misa` CSR.  This register lets you read
the machine `XLEN` (ie. RV32I, RV64I or RV128I), but your code won't
run unless it uses the right instructions in the first place so you
must know this already.  It also contains 26 bits corresponding to the
26 letters of the alphabet anticipating up to 26 extensions (minus
reserved letters).  As discussed before it was naive to believe there
would be only 26 extensions.

Another problem with `misa` is that you cannot extension versions, but
the two other methods do allow you to get all extensions and (in
theory) their versions.

The second method is to use information from Device Tree (DT).  The
deprecated `riscv,isa` field contains a full extension string with
optional versions.  Linux ignores the versions and contains
workarounds for buggy strings in existing implementations.  The
replacement is `riscv,isa-base` and `riscv,isa-extensions` which [has
a cleaner
implementation](https://lore.kernel.org/all/20230702-eats-scorebook-c951f170d29f@spud/).

The third method is to use information from the ACPI RISC-V Hart
Capabilities Table (`RHCT`).  This encodes a full extension string
with optional versions.

Essentially all these methods are only available directly to code
running in Machine or Supervisor modes.  To pass the information up to
userspace, Linux provides `/proc/cpuinfo` and a new system call
[riscv_hwprobe](https://www.kernel.org/doc/html/v6.5-rc2/riscv/hwprobe.html).
However the information available through these is very sparse at the
moment, even relative to what is available from the hardware.


### Hardware Support for Extensions




### How Extensions are Implemented by QEMU











