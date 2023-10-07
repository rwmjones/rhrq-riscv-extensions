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

One unique aspect of RISC-V is that it is designed from the ground up
for **ISA extensions** — as the name suggests these add extra
instructions to the ISA, for implementing features like vector
operations or accelerated encryption.

In this article we'll look at how extensions work at the lowest
levels, how they are ratified and eventually standardized through
RISC-V International (RVI), what extensions are out there, how
extensions are grouped together into profiles, and how you can
discover what extensions are supported in your hardware.  We will also
cover QEMU's support for extensions.

As this article is about extending the ISA, we will not cover non-ISA
extensions in any detail.


### Why Extensions?

Broadly you can extend and accelerate the capabilities of a CPU in two
ways: add new instructions or implement a hardware accelerator.  On
other ISAs accelerator peripherals exist for TCP offloading, AI,
digital signal processing, encryption and so on.  Why would an
extension be better — or when is an extension better?

A way to think about this is that extensions are part of the stream of
instructions.  They are therefore extremely low latency — there may be
little to no overhead to using the instruction.  This is in contrast
to making an I/O request where you might have to form a request packet
and batch requests to get the best performance, and of course the
request goes off the CPU and over a PCIe network.

The flip side to this is that extending the CPU is far less easy than
plugging in a peripheral.


### How Extensions are Encoded

RISC-V has a very regular instruction encoding.  Because arbitrary
extensions are allowed RISC-V uses a variable-length encoding, with
all instructions (currently) being encoded in either 32 bits, or 16
bits for a compressed subset.  For this article I won't talk much
about compressed instructions — see the section below for more on
that.  And I won't discuss the variable-length encodings which are 48
bits are wider, since they are not used by any extensions today.  With
those assumptions in mind we can assume the least significant 2 bits
of each 32 bit instruction are always `1 1`.  The next 5 bits are the
**major opcode** which control how the instruction is decoded:

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
Major extensions like Vector have their own major opcode (`OP-V`),
also have instructions in `LOAD-FP` and elsewhere.  I have heard
extensions which have their own major opcode being called "green-field
extensions".  Smaller extensions use existing major opcodes and fit
into the gaps, known as "brown-field extensions".  All else being
equal, brown-field extensions would have a higher chance of being
accepted.  This means that when proposing a new extension you will
need to consider existing extensions, which is good practice anyway.


#### Byte Ordering of Instructions

Instructions are always stored little endian, with the least
significant byte first in memory, and this applies even on the
(theoretical) big endian RISC-V machine.  Note that disassembly tools
like `objdump` display the bytes as big endian.  Thus an instruction
like `auipc` appears in documentation and objdump output as:

```
      0001b397                auipc   t2,0x1b
        |   |_______________
        |                   |
  imm[31:12]      rd      001 0111
  MSB                          LSB
```

but in memory as:

```
  LSB                                                       MSB
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
- `H`, `U` and `S` pseudo-extensions but used in
  the `misa` CSR (Machine ISA Control and Status Register, more below)
  to refer to support for hypervisor, user and supervisor modes
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
instructions.

Current RISC-V implementations support a "compressed", 16 bit encoding
for common instructions, of course limited in the range of registers
that may be accessed and the opcodes available.  This is similar in
spirit to armv7 Thumb instructions.  The compressed extension was
added very early on by the original RISC-V designers to help save
instruction cache and fetch bandwidth.  A little later, Linux
distributions started to assume that the compressed extension is
always available.

The downside to this is that compressed instructions consume ¾ of the
available opcode space (non-compressed, 32 bit instructions must have
both least significant bits `1 1`).  Also instructions are no longer
automatically 4 byte aligned, and may also cross cache lines and page
boundaries, making decoding harder, albeit still much easier than
complex architectures like x86.

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
impractical.  Shipping two distro variants with and without compressed
instructions is not attractive.  Thus we must decide whether to
require it in hardware or ban it in software.


### The New Extensions

The classic extensions, in hindsight, don't cover much of what is
needed for in a modern server.  Since then dozens of extensions have
been proposed and ratified.  I chose to arrange this section to
highlight the most important extensions first (for servers).  You can
get a complete list of extensions and their status at this link:
https://wiki.riscv.org/display/HOME/Specification+Status

#### Vector, Encryption, Math

The most important extensions to the classic set are Vector (`V`,
`Zv*`), Bit manipulation (`Zb*`), and Packed SIMD (`P`, `Zbpno`,
`Zp*`).  These very roughly correspond to MMX/SSE/AVX on x86, but
RISC-V adds more flexibility and a different (and simpler) programming
paradigm.  It is expected that when hardware with these instructions
appear, they will be widely used in binaries (as happens on x86).

A whole article could be written about the vector extension (which is
in fact a large collection of extensions), here are a couple:
[Adventures with RISC-V Vectors and LLVM](https://llvm.org/devmtg/2019-04/slides/TechTalk-Kruppe-Espasa-RISC-V_Vectors_and_LLVM.pdf),
[Programming with RISC-V Vector Instructions](https://gms.tf/riscv-vector.html).

RISC-V also has two important sets of extensions for cryptography,
called Scalar Crypto and Vector Crypto.  Scalar Crypto was folded into
the Bit manipulation extensions (`Zbkb`) — for example the `zip`
instruction in `Zbkb` is called out for being useful to implement
SHA3.  Vector Crypto (`Zvknhb`, `Zvbc`, `Zvkn`, `Zvks` and more!)
contains extended Vector instructions useful for Elliptic curve
cryptography, various Message Authentication Codes, AES, AES-GCM, and
many more.

Another important group of extensions extend floating point support,
adding Bfloat16 (`Zfbfmin`, `Zvfbfmin`, `Zvfbfwma`); adding common
floating point constants and many useful floating point operations
that are not present in the classic set (`Zfa`); and the "f-in-x"
extensions (`Zfinx`, `Zdinx`, `Zhinx`, `Zhinxmin`) which allow
floating point and integer registers to be shared.


#### Virtualization

RISC-V support for running virtual machines (the Hypervisor extension)
was demonstrated as far back as 2017, was ratified in 2021, but is
only expected to appear in hardware in 2024.  This is expected to be
vital for RISC-V adoption on servers.  `H` (hypervisor) is mostly a
complicated addition to the [privileged
spec](https://riscv.org/technical/specifications/privileged-isa/)
involving new modes and CSRs rather than new instructions, but some
new instructions were added.  In particular there are instructions to
access memory while translating guest virtual addresses (useful for
emulating I/O), and extra fencing instructions.


#### Interrupts, Cache and Memory

Many extensions have been proposed and ratified which are beneficial
for server-class operating systems.

The most important are probably the ones which fix the interrupt
architecture of the original design (which was notably inefficient).
In particular the Advanced Interrupt Architecture (`Smaia`, `Ssaia`)
and the older Fast Interrupt specification (`S*clic*`).  (Recall that
extension names prefixed with `S` apply to supervisor mode).  Worth a
mention is `Smrnmi` which fixes another issue with the base standard,
that after a Non-Maskable Interrupt, the interrupted program could not
resume running.  This adds a new `mnret` instruction to resume after
NMI.

The original RISC-V design assumed a relaxed memory consistency
similar to Arm, but some machines would prefer the stricter ordering
found on x86 (for example, because you need to emulate or port code
from x86).  The `Ztso` extension changes load and store operations to
use total store ordering.

Cache management operations (CMOs) are important in modern operating
systems and RISC-V defines a family of extensions for cache block
operations.  `Zicbom` are cache block management instructions for
things like invalidating blocks of cache.  `Zicbop` instructions are
prefetch hints.  `Zicboz` instructions store zeros over blocks of
cache.


#### Safety and Security

Security is a key issue for servers, and one area that is being
actively developed on all architectures is control flow integrity
(CFI).  RISC-V is ratifying two extensions for CFI.  `Zicfiss` defines
a shadow stack and provides new instructions to push and pop values
there.  `Zicfilp` defines places where code is allowed to branch to
(especially through "computed gotos"), known as "landing pads".  These
techniques are designed to prevent ROP attacks after stack smashing
exploits.

Landing pads themselves are defined by further extensions — `Zimop`,
`Zcmop` — that reserve some opcode space for ["may be
operations"](https://github.com/riscv/riscv-isa-manual/blob/main/src/zimop.adoc).


#### Miscellaneous

Other extensions relevant to servers:

- `Svinval` can be used for selective TLB invalidation.

- `Zawrs` adds instructions that make polling memory locations more
  efficient, typically used in spinlocks or when polling on a lockless
  queue.  `Zihintpause` adds a new `pause` instruction which can also be
  used to reduce power consumption and memory traffic in spinlocks.

- `Zihintntl` may be used to hint that memory accesses are non-temporal
  (ie. do not need to be cached).

- `Zacas` adds atomic compare and swap, omitted from the original Atomic
  instructions.

- `Zicond` adds conditional instruction prefixes, similar to armv7
  conditional operations.


### Profiles

Software generally needs some kind of baseline target to run.  While
some extensions can be detected at runtime and different code paths
chosen, much software will be written that expects a basic set of
extensions to exist.

For this reason RISC-V International defines a set of profiles.
Currently they are named after the year they were defined, and grouped
into two families.  Thus, at time of writing, the latest profile is
RVA22 and RVA23 is in development.  `A` stands for the "Application
processors running rich operating systems" family (ie. servers),
`22`/`23` are the year code.

RVA22 includes all the classic extensions, and a scattering of older
system extensions.  More notable is what it omits.  It predates the
ratification of the Vector extension, so this is only optional, and
vector crypto and Packed SIMD are also missing.  The full RVA22
profile is described here:
https://github.com/riscv/riscv-profiles/blob/main/profiles.adoc

It's expected in future this will change as the system doesn't reflect
the many branches of the RISC-V ecosystem.  We expect in future that
there will be stricter requirements for backwards compatibility, that
only fully ratified extensions will be allowed, and that unused opcode
space will be forced to trap (allowing some forward compatibility
through trap and emulate).


### Discovering What Extensions Are Available — a.k.a. where's my CPUID?

x86 has CPUID, a comprehensive method to detect at runtime what
features the processor supports, and many other aspects of the CPU
(like cache sizes and so forth).  There is nothing this comprehensive
available in RISC-V at the moment.

For RISC-V, there are three ways to determine what extensions are
available in the hardware.  The oldest mechanism, now mostly
deprecated, is to read the `misa` CSR.  This register lets you read
the machine `XLEN` (ie. the base ISA: RV32I, RV64I or RV128I), but
your code won't run unless it uses the right instructions in the first
place so you must know this already.  It also contains 26 bits
corresponding to the 26 letters of the alphabet, anticipating up to 26
extensions (minus reserved letters).  As discussed before it was naive
to believe there would be only 26 extensions.

Another problem with `misa` is that you cannot read versions of
extensions, but the two other methods do allow you to get all
extensions and (in theory) their versions.

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

If you know the machine is using device tree, then in Linux the
`riscv,isa` field can be read out directly from `/sys`.  Here is an
example from a QEMU guest:

```
$ cat '/sys/firmware/devicetree/base/cpus/cpu@0/riscv,isa'
rv64imafdch_zicbom_zicboz_zicsr_zifencei_zihintntl_zihintpause_zawrs_zfa_zca_zcd_zba_zbb_zbc_zbs_sstc_svadu
```


### How Extensions are Implemented by QEMU

QEMU, a virtual machine emulator and hypervisor, can emulate RISC-V in
software.  This is useful when you don't have RISC-V hardware or don't
have hardware that supports a particular extension.  The software
emulation in QEMU is called the **Tiny Code Generator ("TCG")** and so
when you see TCG used here, it is a shorthand for software emulation
of the RISC-V architecture.

TCG works by translating [basic
blocks](https://en.wikipedia.org/wiki/Basic_block) as they are
encountered.  The translated code is stored in a QEMU
`TranslationBlock` (TB) structure, and referenced through a hash table
of (CPU state, physical address).  TBs persist so that code doesn't
need to be retranslated, but can be invalidated by various things such
as writes happening to the same code page.

TCG defines a set of basic operations, like integer adds, loads,
stores, labels, branches and so on, and you can recognize these when
you see `tcg_gen_*` called in QEMU code.  For example
`tcg_gen_qemu_ld_i64` would be called when translating a block of
code, and would generate a TCG instruction to do a 64 bit load and
append it to the list of translated instructions.  However anything
complicated (CSRs, vector instructions, etc) is translated into a call
to a helper function.  You will see helper functions defined in QEMU
using the macro `HELPER(<name>)`, and when translating a call to the
helper would be generated using `gen_helper_<name>`.

Since most RISC-V extensions are "complicated" they are almost always
implemented as a set of helpers.

There is a file in the QEMU source `target/riscv/insn32.decode` which
describes how instruction bit patterns are decoded.  Extensions must
list their new instructions here.

The file `target/riscv/cpu.c` contains two tables listing ISA
extensions, their names and versions.  This is a very useful reference
for finding out what extensions have been implemented in QEMU.

At the time of writing (2023H2), QEMU supports these extensions,
making it probably the most capable RISC-V platform:

 - The base RV32I and RV64I ISAs
 - The classic extensions: M A F D
 - Compressed instructions: C, Zca, Zcb, Zcf, Zcd, Zce, Zcmp, Zcmt
 - The embedded extension: E
 - The hypervisor extension: H
 - User and Supervisor modes (but note, not Machine mode): U S
 - Dynamic languages: J
 - Cache management (partial): Zicbom, Zicboz
 - Conditional ops: Zicond
 - Read and write CSRs: Zicsr
 - FENCE.I instruction: Zifencei
 - Pause hint: Zihintpause
 - Wait on reservation set: Zawrs
 - Additional scalar FP: Zfa
 - Bfloat16 (partial): Zfbfmin
 - Half-width FP: Zfh, Zfhmin
 - FP using integer regs: Zfinx, Zdinx, Zhinx
 - Bit manipulation: Zba, Zbb, Zbc, Zbs
 - Crypto scalar: Zbkb, Zbkc, Zbkx, Zk*
 - Vector (mostly complete): V, Zv*
 - Advanced Interupt Architecture: Smaia, Ssaia
 - State enable: Smstateen
 - Count overflow & filtering: Sscofpmf
 - Time compare: Sstc
 - Hardware update of PTE A/D bits: Svadu
 - Fast TLB invalidation: Svinval
 - NAPOT pages: Svnapot
 - Page-based memory types: Svpbmt
 - T-HEAD multiple custom extensions
 - Ventana custom extensions for conditional ops

### Emulation of RISC-V Extensions on RISC-V

RISC-V extensions may also be emulated on RISC-V hardware using trap
and emulate.  There are two broad approaches taken:

1. Modify OpenSBI illegal instruction handler
   (`lib/sbi/sbi_illegal_insn.c`) to catch the illegal instruction and
   emulate it.

2. Modify the Linux kernel illegal instruction handler
   (`arch/riscv/kernel/traps.c`).

The first method was used to implement a mostly complete emulation of
the Hypervisor extension: https://github.com/dramforever/opensbi-h

The second method was used to implement the Vector extension for
machines that lack it.

Modifying OpenSBI has some downsides which you should be aware of:

- On some machines SBI is part of the platform firmware and might not
  be open source or user-replaceable.

- M-mode does not use paging, so the emulation must do its own page
  table walk if the extension uses virtual addresses.

- M-mode traps to SBI have extra overhead in hardware.

- There are also security and operational concerns as M-mode has
  complete access to the hardware, but a bug in the operating system
  might be limited and recoverable (eg. by a software watchdog).

Modifying Linux has the downside that the emulation is only available
for Linux, and won't work for other operating systems nor for the code
that runs before Linux, such as SBL, SBI, u-boot and EDK2.
