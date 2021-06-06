

# Hardware Design Languages (Work in progress)

## History of hardware design languages 

### Early, haven't found date yet
1. PALASM

### 1970s
1. (1974) AHPL (A Hardware Programming Language) Frederick j. Hill

### 1980s
1. (1983) ABEL (Advanced Boolean Expression Language) - Data IO Corporation.
2. (1983) CUPL (Compiler for Universal Programmable Logic)
3. (1984) Verilog.
4. (1986) Ruby (hardware description language) - Mary Sheeran.
5. (1986) Ella - Royal Signals and Radar Establishment
6. (1987) VHDL (VHSIC Hardware Description Language). Standardized 1987  (IEEE Std 1076).

### 1990s
1. AHDL (Altera Hardware Description Language).
2. (1993) HML (Hardware ML) - JohnO'Leary, MarkLinderman, MiriamLeeser, MarkAagaard
3. (1993) KARL (KAiserslautern Register Language) (check KARL AND ABL)
4. (1995) Lola - ETH Zurich
5. (1996) Handel-C - Oxford University Computing Laboratory

### 2000s
1. Bluespec - Lennart Augustsson.
2. Bluespec SystemVerilog.
3. Clash.
4. Confluence - Tom Hawkins.
5. (1999,2000) SystemC - IEEE 1666-2011.
6. (2001) Lava.
7. (2002) Hardware Join Java.
8. (2002) SystemVerilog - IEEE 1800-2017.
9. (2003) MyHDL. 
10. (2003) Impulse C - Impulse Accelerated Technologies.
11. (2005) ROCCC (Riverside Optimizing Compiler for Configurable Computing) (maybe earlier than 2005).
12. (2006) RHDL (maybe earlier than 2005).
13. (2006) JHDL (Just Another Hardware Description Language).
14. (2009) ESys.net. Port of SystemC framework for C++ to .net.

### 2010s
1. Chisel.
2. (2011) HHDL (Haskell HDL) - Serguey Zefirov.
3. (2011) THDL++.
4. (2014) PyMTL - Cornell, computer systems laboratory.
5. (2014) PyRTL - University of California, Santa Barbara.
6. (2015) SpinalHDL.
7. (2018) Hardcaml - janestreet.
8. (2018) nMigen - m-labs.
9. (2018) TL-Verilog.

### Hard to find understandable information about

- C-to-Verilog.
- COLAMO.
- CoWareC. 
- Hydra. 
- ISPS.
- ParC (Parallel C++).
- M  (Mentor graphics HDL).
- SystemTCL 	HDL based on Tcl.

### Intermediate Representations 

- EDIF: Electronic Design Interchange Format.
- (2007) AHIR "A Hardware Intermediate Representation for Hardware Generation from High-level Programs".
- (2019) LNAST "A Language Neutral Intermediate Representationfor Hardware Description Languages"
- (2020) LLHD "a multi-level intermediate representation for hardware description languages".
- IROHA "Intermediate Representation Of Hardware Abstraction"
- A Compiler Intermediate Representationfor Reconfigurable Fabrics - Zhi Guo et'al

### To look into
- KARUTA hdl


## Why So Many Languages?

What is wrong with VHDL and Verilog?
1. VHDL is poorly suited for formal verification [citation](https://dl.acm.org/doi/pdf/10.1145/291251.289440).
2. VHDL is a simulation language retrofitted for synthesis [citation](https://dl.acm.org/doi/pdf/10.1145/291251.289440).
3. VHDL lacks a clearly defined synthesis semantics [citation](https://books.google.se/books?hl=en&lr=&id=rJ_wBwAAQBAJ&oi=fnd&pg=PA116&ots=-N_8j52lXl&sig=JBYmXr08Xkmpb42CxbuyUvvgEpQ&redir_esc=y#v=onepage&q&f=false)
   The synthesis language is a subset of the simulation language yet one would
   want the synthesized hardware to be an exact implementation of what was synthesised.
   There may be details to this:
   1. You can simulate function, energy consumption, timing.
      But these take place on different stages of elaboration of the hardware
      design (which makes sense). 
   
4. Poor "help" with concurrent programming [citation](https://www.cl.cam.ac.uk/teaching/0910/P35/obj3.2/page14.html)
   (TODO: Look into Handel-C) 
5. Hard to write behavioural style code in Verilog [citation](https://www.cl.cam.ac.uk/teaching/0910/P35/obj3.2/page14.html)
6. Difficult to understand dataflow model. Lots of concurrent statements and signals.
   Very low abstraction level. This gives rise to higher level programming styles
   within VHDL, such as the Gaisler 2-process method [citation](https://www.gaisler.com/doc/vhdl2proc.pdf).
   However, programmer discipline is required to reap the benefits.
   You can draw a parallel to writing object oriented or functional code in C,
   sure you can, but the C compiler wont help you do it correctly in any way.


**Thoughts**

Should one even expect that the moving towards higher level
abstractions within hardware design should be similar to the one of
software. Many of the *alternative* hardware description languages are
based on functional programming. Why is that so? Does that feel natural?

Functional programming languages are often very dynamic in terms of
their memory use. Closures are made up on the fly and passed around as
objects (higher order functions). Hardware is very static (more
dynamic in the future perhaps). So what aspect of functional programming
is it that make (so many) people thing, "hardware design should be functional 
programming".

The paragraph directly above is not entirely true. Some functional
hardware description system I know of use a functional language as a
hardware modeling language, so you use a functional language to
describe the hardware while the hardware described is not
functional. A very static hardware netlist is generated from the
description written in the functional language. Also there is hardware
that is purely functional, these are called combinational blocks and
these are very efficiently expressed in something like Lava.

Combinational hardware blocks are not clocked and this may be why it seems
like clocks are nowhere to be found in functional approaches.
(TODO: Check some functional languages for hardware design and see how
many of them totally abstract out the clock). 






### High-level and low-level programming in hardware and software

Hardware designs are becoming increasingly complex so it is natural
to want higher level abstractions. This mirrors what has been
happening in software:  Assembler -> C -> C ++ -> C#/Java. See for
example
[citation](https://dl.acm.org/doi/pdf/10.1145/1363686.1364037).

On the software side there were incredibly abstract programming
models popping up very early, Lisp for example. Also in hardware,
people were experimenting with formal and highly abstracted models
for hardware design at the same time as Verilog and VHDL
appeared. Something drove the popularity
of the C -> C++ and VHDL/Verilog from the beginning.

In the 70s, from the Intel 4004 to the Motorola 68000, the number of
transistors went from 2250 to 68000. this was before VHDL and Verilog
and maybe we can consider the 70s the Assembler era of hardware
design?

In the 80s the number of transistors lept from 68000 on the
motorola 68K in 1979 to 1180235 (over a million) with the Intel 486.
On the 486 die there was room for over 500 Intel 4004s, wow.  



## A Closer Look At

### Ruby

### Bluespec

### Clash

### Chisel

### PyMTL

### PyRTL




___

[HOME](https://svenssonjoel.github.io)
