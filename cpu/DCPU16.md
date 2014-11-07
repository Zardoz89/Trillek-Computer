---
layout : default
title : DCPU-16 CPU
cat : CPU
---
# DCPU-16 Specification
#### Version 1.8.0



=== SUMMARY ====================================================================

* 16 bit CISC CPU
* 16 bit addressing
* 16 bit I/O address bus
* 0x10000 words of ram (64KiW <-> 128KiB )
* 8 registers (**A**, **B**, **C**, **X**, **Y**, **Z**, **I**, **J**)
* program counter (**PC**)
* stack pointer (**SP**)
* ALU high word - extra/excess (**EX**)
* interrupt address (**IA**)

In this document, anything within [brackets] is shorthand for "the value of the
RAM at the location of the value inside the brackets". For example, SP means
stack pointer, but [SP] means the value of the RAM at the location the stack
pointer is pointing at.

Whenever the CPU needs to read a word, it reads two octets [PC], then increases PC by one.
Shorthand for this is [PC++]. In some cases, the CPU will modify a value before
reading it, in this case the shorthand is [++PC].

For stability and to reduce bugs, it's strongly suggested all multi-word
operations use little endian in all DCPU-16 programs, wherever possible.

NOTATION : 
 * Byte = octect, i.e. 8 bit values
 * Word = two octect, i.e. 16 bit values 

=== INSTRUCTIONS ===============================================================

Instructions are 1-3 words long and are fully defined by the first word.
In a basic instruction, the lower five bits of the first word of the instruction
are the opcode, and the remaining eleven bits are split into a five bit value b
and a six bit value a.
b is always handled by the processor after a, and is the lower five bits.
In bits (in LSB-0 format), a basic instruction has the format: **aaaaaabbbbbooooo**

In the tables below, C is the time required in cycles to look up the value, or
perform the opcode, VALUE is the numerical value, NAME is the mnemonic, and
DESCRIPTION is a short text that describes the opcode or value.


```
--- Values: (5/6 bits) ---------------------------------------------------------
 C | VALUE     | DESCRIPTION
---+-----------+----------------------------------------------------------------
 0 | 0x00-0x07 | register (A, B, C, X, Y, Z, I or J, in that order)
 0 | 0x08-0x0f | [register]
 1 | 0x10-0x17 | [register + next word]
 0 |      0x18 | (PUSH / [--SP]) if in b, or (POP / [SP++]) if in a
 0 |      0x19 | [SP] / PEEK
 1 |      0x1a | [SP + next word] / PICK n
 0 |      0x1b | SP
 0 |      0x1c | PC
 0 |      0x1d | EX
 1 |      0x1e | [next word]
 1 |      0x1f | next word (literal)
 0 | 0x20-0x3f | literal value 0xffff-0x1e (-1..30) (literal) (only for a)
 --+-----------+----------------------------------------------------------------
```  
* "next word" means "[PC++]". Increases the word length of the instruction by 1.
* By using 0x18, 0x19, 0x1a as PEEK, POP/PUSH, and PICK there's a reverse stack
  starting at memory location 0xffff. Example: "SET PUSH, 10", "SET X, POP"
* Attempting to write to a literal value fails silently


```
--- Basic opcodes (5 bits) ----------------------------------------------------
 C | VAL  | NAME     | DESCRIPTION
---+------+----------+---------------------------------------------------------
 - | 0x00 | n/a      | special instruction - see below
 1 | 0x01 | SET b, a | sets b to a
 2 | 0x02 | ADD b, a | sets b to b+a, sets EX to 0x0001 if there's an overflow, 
   |      |          | 0x0 otherwise
 2 | 0x03 | SUB b, a | sets b to b-a, sets EX to 0xffff if there's an underflow,
   |      |          | 0x0 otherwise
 2 | 0x04 | MUL b, a | sets b to b*a, sets EX to ((b*a)>>16)&0xffff (treats b,
   |      |          | a as unsigned)
 2 | 0x05 | MLI b, a | like MUL, but treat b, a as signed
 3 | 0x06 | DIV b, a | sets b to b/a, sets EX to ((b<<16)/a)&0xffff. if a==0,
   |      |          | sets b and EX to 0 instead. (treats b, a as unsigned)
 3 | 0x07 | DVI b, a | like DIV, but treat b, a as signed. Rounds towards 0
 3 | 0x08 | MOD b, a | sets b to b%a. if a==0, sets b to 0 instead.
 3 | 0x09 | MDI b, a | like MOD, but treat b, a as signed. (MDI -7, 16 == -7)
 1 | 0x0a | AND b, a | sets b to b&a
 1 | 0x0b | BOR b, a | sets b to b|a
 1 | 0x0c | XOR b, a | sets b to b^a
 1 | 0x0d | SHR b, a | sets b to b>>>a, sets EX to ((b<<16)>>a)&0xffff 
   |      |          | (logical shift)
 1 | 0x0e | ASR b, a | sets b to b>>a, sets EX to ((b<<16)>>>a)&0xffff 
   |      |          | (arithmetic shift) (treats b as signed)
 1 | 0x0f | SHL b, a | sets b to b<<a, sets EX to ((b<<a)>>16)&0xffff
 2+| 0x10 | IFB b, a | performs next instruction only if (b&a)!=0
 2+| 0x11 | IFC b, a | performs next instruction only if (b&a)==0
 2+| 0x12 | IFE b, a | performs next instruction only if b==a 
 2+| 0x13 | IFN b, a | performs next instruction only if b!=a 
 2+| 0x14 | IFG b, a | performs next instruction only if b>a 
 2+| 0x15 | IFA b, a | performs next instruction only if b>a (signed)
 2+| 0x16 | IFL b, a | performs next instruction only if b<a 
 2+| 0x17 | IFU b, a | performs next instruction only if b<a (signed)
 - | 0x18 | -        |
 - | 0x19 | -        |
 3 | 0x1a | ADX b, a | sets b to b+a+EX, sets EX to 0x0001 if there is an over-
   |      |          | flow, 0x0 otherwise
 3 | 0x1b | SBX b, a | sets b to b-a+EX, sets EX to 0xFFFF if there is an under-
   |      |          | flow, 0x0 otherwise
 - | 0x1c | INP b, a | sets b to the value read on I/O port a 
 - | 0x1d | OUT b, a | writes b to the I/O port b 
 2 | 0x1e | STI b, a | sets b to a, then increases I and J by 1
 2 | 0x1f | STD b, a | sets b to a, then decreases I and J by 1
---+------+----------+----------------------------------------------------------
```
* The branching opcodes take one cycle longer to perform if the test fails
  When they skip an if instruction, they will skip an additional instruction
  at the cost of one extra cycle. This lets you easily chain conditionals.  
* Signed numbers are represented using two's complement.

    
Special opcodes always have their lower five bits unset, have one value and a
five bit opcode. In binary, they have the format: **aaaaaaooooo00000**
The value (a) is in the same six bit format as defined earlier.

```
--- Special opcodes: (5 bits) --------------------------------------------------
 C | VAL  | NAME  | DESCRIPTION
---+------+-------+-------------------------------------------------------------
 - | 0x00 | n/a   | compact instruction - see below
 3 | 0x01 | JSR a | pushes the address of the next instruction to the stack,
   |      |       | then sets PC to a
 - | 0x02 | -     |
 - | 0x03 | -     |
 - | 0x04 | -     |
 - | 0x05 | -     |
 - | 0x06 | -     |
 - | 0x07 | -     | 
 4 | 0x08 | INT a | triggers a software interrupt with message a
 1 | 0x09 | IAG a | sets a to IA 
 1 | 0x0a | IAS a | sets IA to a
 3 | 0x0b | RFI a | disables interrupt queueing, pops A from the stack, then 
   |      |       | pops PC from the stack
 2 | 0x0c | IAQ a | if a is nonzero, interrupts will be added to the queue
   |      |       | instead of triggered. if a is zero, interrupts will be
   |      |       | triggered as normal again
 - | 0x0d | -     |
 - | 0x0e | -     |
 - | 0x0f | -     |
 - | 0x10 | -     |
 - | 0x11 | -     |
 - | 0x12 | -     |
 - | 0x13 | -     |
 - | 0x14 | -     |
 - | 0x15 | -     |
 - | 0x16 | -     |
 - | 0x17 | -     |
 - | 0x18 | -     |
 - | 0x19 | -     |
 - | 0x1a | -     |
 - | 0x1b | -     |
 - | 0x1c | -     |
 - | 0x1d | -     |
 - | 0x1e | -     |
 - | 0x1f | -     |
---+------+-------+-------------------------------------------------------------
```

Implied opcodes always have their lower ten bits unset, have one value and a    
six bit opcode. In binary, they have the format: **oooooo0000000000**

```
--- Special opcodes: (5 bits) --------------------------------------------------
 C | VAL  | NAME  | DESCRIPTION
---+------+-------+-------------------------------------------------------------
 4 | 0x00 | HLT   | if interrupts are enabled, generates an interrupt with 
   |      |       | message 0, otherwise halts CPU operation. 
 3 | 0x01 | SLP   | halts CPU operation, and puts the CPU in a low power state 
   |      |       | if interrupts are enabled, then the DCPU-16N will resume 
   |      |       | operation when the next interrupt is triggered.
 - | 0x02 | -     |
 - | 0x03 | -     |
 1 | 0x04 | BYT   | Next instruction operates only writes over the Lowest Byte
 - | 0x05 | -     |
 - | 0x06 | -     |
 - | 0x07 | -     | 
 - | 0x08 | -     |
 - | 0x09 | -     | 
 - | 0x09 | -     | 
 - | 0x0a | -     |
 - | 0x0b | -     |
 - | 0x0c | -     |
 - | 0x0d | -     |
 - | 0x0e | -     |
 - | 0x0f | -     |
 - | 0x10 | -     |
 - | 0x11 | -     |
 - | 0x12 | -     |
 - | 0x13 | -     |
 - | 0x14 | -     |
 - | 0x15 | -     |
 - | 0x16 | -     |
 - | 0x17 | -     |
 - | 0x18 | -     |
 - | 0x19 | -     |
 - | 0x1a | -     |
 - | 0x1b | -     |
 - | 0x1c | -     |
 - | 0x1d | -     |
 - | 0x1e | -     |
 - | 0x1f | -     |
---+------+-------+-------------------------------------------------------------

=== INTERRUPTS =================================================================    

The DCPU-16 will perform at most one interrupt between each instruction. If
multiple interrupts are triggered at the same time, they are added to a queue.
If the queue grows longer than 256 interrupts, the DCPU-16 will catch fire. 

When IA is set to something other than 0, interrupts triggered on the DCPU-16
will turn on interrupt queueing, push PC to the stack, followed by pushing A to
the stack, then set the PC to IA, and A to the interrupt message.
 
If IA is set to 0, a triggered interrupt does nothing. Software interrupts still
take up four clock cycles, but immediately return, incoming hardware interrupts
are ignored. Note that a queued interrupt is considered triggered when it leaves
the queue, not when it enters it.

Interrupt handlers should end with RFI, which will disable interrupt queueing
and pop A and PC from the stack as a single atomic instruction.
IAQ is normally not needed within an interrupt handler, but is useful for time
critical code.




=== I/O PORTS =============== =================================================    

The DCPU-16 have a 16 bit I/O address space that allows to comunicate with 
devices separated from the memory subsystem.

To access this I/O bus, the DCPU-16 have two special instructions, INP and OUT.

INP allows to read from an I/O Port and set an register, stack or an memory 
address to the value readed on these I/O Port

OUT allows to write a valute to an I/O Port.




=== INTERFACE TO THE COMPUTER =================================================    


The DCPU-16 I/O bus is mapped againsts the address 0x11000 to 0x12000, with it 
could access the Hardware Enumeration And Control registers, plus the embeded 
devices of the Trillek computer.




=== EXTENDED MEMORY ========= =================================================    


Additionally, on the 0x11FF00 (I/O Port 0x7F80), there is exposed an interface 
to the Extended Memory Unit (EMU). The EMU allows to the DCPU-16 to access a 24
bit address space, doing bank switching with banks of 8 KiB. The virtual memory
space of the DCPU-16 is split into 16 0x1000 byte blocks. The EMU maps each 
block to one of 4096 pages in the 24 bit physical address space, where each page
is also 0x1000 bytes.

Pages can be selected to blocks writing to the port 0x7F80, were is write a 
packed 12 bit page number and a 4 bit block number to map the page to. The pages
may be mapped to multiple blocks at once, meaning both blocks will refer to the
same memory or hardware at that physical address.

The initial layout of the pages at boot time maps the address 0x100000 to the 
first bank, mapping the first 8 KiB of the ROM on the page 0. The rest of the 
blocks are set sequentially starting at page 1. 


