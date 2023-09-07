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

One unique aspect of RISC-V is the ISA is designed for **extensions**.

In this article we'll look at how extensions work at the lowest
levels, how they are ratified and eventually standardized through
RISC-V International (RVI), what extensions are out there, how
extensions are grouped together into profiles, and how you can
discover what extensions are supported in your hardware.  We will also
cover what hardware today supports particular extensions as well as
looking at QEMU's support for extensions if you need to develop for an
extension but it is not yet available.

