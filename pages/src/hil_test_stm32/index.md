# Automated hardware in the loop testing with STM3F4-Discovery, OpenOCD and Expect

[Cortex M. McGee](http://www.github.com/svenssonjoel/cortex_m_mcgee)
is a collection of c functions for generation of 16 and 32bit thumb
instructions. The goal is to provide a c function for each of the
thumb1 and thum2 opcodes. As an example there is a function called
`m0_add_imm8` that adds a 8bit immediate value to a register. 

```
thumb_opcode_t m0_add_imm8(reg_t rdn, uint8_t imm8) {...}
```
Sequences of opcodes are created using a function call `emit_opcode`:

```
int emit_opcode(instr_seq_t *seq, thumb_opcode_t op) {...}
```

This function appends the opcode given as the second argument to the
sequence of opcodes given as the first.

I don't know if there is much value to be able to programatically
generate cortex-m machine code sequences in memory. JIT compilation
comes to mind and that is fun and all, but how applicable and useful
that is on machines that in general are quite restricted on resources
is something to experiment with later on. 

This is the first time I am attempting to genereate machine-code in
memory and I am quite insecure about getting the opcode encodings
right, so testing will be important. 

In general, it would be cool to be able to automate testing against
hardware.  Manually flashing the board, connecting a debugger and
stepping through instructions is a bit tedious and time
consuming. Doing this manually is fun and educational the first few
times though, but quite quickly becomes booring.

When it comes to the testing in this specific case of machine-code
generation, testing against the hardware is maybe not that crucial. It
could just as well be done against a disassembler that you run on your
cross-development platform. So in this particular case, setting up an
automated HIL testing system may be overkill, so that aspect of it is
just for the general case benefit and fun. Next time we may be working
on something where automated testing against the hardware is much more
necessary and directly valuable ;)

# Manual "testing" with OpenOCD and Telnet

# Automating the interaction with expect

# Multi-staged programming and test-generation

___

[HOME](https://svenssonjoel.github.io)
