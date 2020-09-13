# Automated hardware in the loop testing with STM32F4-Discovery, OpenOCD and Expect

[Cortex M. McGee](http://www.github.com/svenssonjoel/cortex_m_mcgee)
is a collection of c functions for generation of 16 and 32bit thumb
instructions. The goal is to provide a c function for each of the
thumb1 and thumb2 opcodes. As an example there is a function called
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

I don't know if there is much value to be able to programmatically
generate cortex-m machine code sequences in memory. JIT compilation
comes to mind and that is fun and all, but how applicable and useful
that is on machines that, in general, are quite restricted on resources
is something to experiment with later on. 

This is the first time I am attempting to generate machine-code
opcodes and I am quite insecure about getting the opcode encodings
right, so testing will be important. The encoding functions are based
on the information found in [ARMv7-M Architecture Reference
Manual](https://developer.arm.com/documentation/ddi0403/ed/) as well
as [The definitive guide..](https://www.goodreads.com/book/show/17070271-the-definitive-guide-to-arm-cortex-m3-and-cortex-m4-processors)
by Joseph Yiu.

In general, it would be cool to be able to automate testing against
hardware.  Manually flashing the board, connecting a debugger and
stepping through instructions is a bit tedious and time
consuming. Doing this manually is fun and educational the first few
times though, but quite quickly becomes boring.

When it comes to the testing in this specific case of machine-code
generation, testing against the hardware is maybe not that crucial. It
could just as well be done against a disassembler that you run on your
cross-development platform. So in this particular case, setting up an
automated HIL testing system may be overkill, so that aspect of it is
just for the general case benefit and fun. Next time we may be working
on something where automated testing against the hardware is much more
necessary and directly valuable ;)

# An example of machine code generation


The code below generates a sequence of 4 operations into an array of
`uint16_t`. The opcodes all come from the instruction set that the
cortex-m0 can execute and they are all 16 bits wide.

The four calls to `emit_opcode` creates instructions for putting the
value 2 in register 0 then the value 3 in register 1. Following,
register 1 and register 0 are added and the result stored in
register 2.  Lastly register 0 and register 2 are added and the result
stored in register 0.


```
uint16_t instrs[4];
instr_seq_t seq;
seq.size = 4;
seq.pos  = 0;
seq.mc = instrs;
  
emit_opcode(&seq, m0_mov_imm(r0, 2));
emit_opcode(&seq, m0_mov_imm(r1, 3));
emit_opcode(&seq, m0_add_low(r2, r1, r0));
emit_opcode(&seq, m0_add_any(r0, r2));
```

The instruction sequence can be stored into a file, here called `example0.bin`.


```
FILE *fp = fopen("example0.bin", "w");
if (!fp)  {
  printf("Error opening file %s\n", fn);
  return 0;
}

if (fwrite(seq.mc,sizeof(uint16_t),4,fp) < 4) {
  printf("Error writing file\n");
  return 0;
}
```

After generating `example0.bin` the contents can be inspected using `xxd`: 


```
> xxd example0.bin
00000000: 0220 0321 0a18 1044                      . .!...D
> xxd -b example0.bin
00000000: 00000010 00100000 00000011 00100001 00001010 00011000  . .!..
00000006: 00010000 01000100                                      .D
``` 

The `example0.bin` file can be disassembled using
`arm-none-eabi-objdump` together with some magical incantations:


``` 
> arm-none-eabi-objdump -D -b binary -m armv6-m example0.bin -Mforce-thumb
Disassembly of section .data:

00000000 <.data>:
   0:	2002      	movs	r0, #2
   2:	2103      	movs	r1, #3
   4:	180a      	adds	r2, r1, r0
   6:	4410      	add	r0, r2
``` 

The reason all that extra "magic" (all those parameters) is needed is
because the `example0.bin" file is not a proper elf file, it just a
bunch of opcodes thrown into a binary file. For elf files Objdump can
figure out some of the things by itself from the elf header. Objdump
also states that it has disassembled the "data" section, but this is
of course irrelevant as there are no sections at all.

However, pretty neat that the disassembly seems to be what we wanted.
Two immediate mov, one three register add and one two register
add. Great!

# Manual "testing" with OpenOCD and Telnet

It is always fun to involve the development board and we can get the same 
kind of information with the STM32F4-Discovery board in the loop. 

Testing this on the board does not really give that much information
it is however fun to see if we can write a sequence of opcodes that we
generated into memory and then execute the operations. 

OpenOCD will be used to "debug" against the board. To do this OpenOCD 
should be started with a command such as this: 

``` 
openocd -f stm32f407g.cfg
```

The file `stm32f407g.cfg` has the following contents: 

```
# This is an STM32F4 discovery board with a single STM32F407VGT6 chip.
# http://www.st.com/internet/evalboard/product/252419.jsp

#source [find interface/stlink.cfg]
source [find interface/stlink-v2-1.cfg]

transport select hla_swd

# increase working area to 64KB
set WORKAREASIZE 0x10000

source [find target/stm32f4x.cfg]

reset_config srst_only
``` 

Now OpenOCD should have started and output something that looks like below to the terminal: 

``` 
Open On-Chip Debugger 0.10.0
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
debug_level: 1
adapter speed: 2000 kHz
adapter_nsrst_delay: 100
none separate
srst_only separate srst_nogate srst_open_drain connect_deassert_srst
```

When OpenOCD is running like this it is possible to connect to it with either gdb or using telnet. 
I will be using telnet here. 

In a new terminal type `telnet localhost 4444`. this should bring up the following: 

``` 
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Open On-Chip Debugger
> 
```

We now have an interface to talk to OpenOCD and thus also the STM32F4-Discovery board. 


Typing `reset init`, resets the board: 

``` 
> reset init
adapter speed: 1800 kHz
target halted due to debug-request, current mode: Thread 
xPSR: 0x01000000 pc: 0x08000300 msp: 0x20000400
adapter speed: 4000 kHz
> 
``` 

Another fun command is `reg` that will display information about the values in all registers. 

``` 
> reg
===== arm v7m registers
(0) r0 (/32): 0x00000000
(1) r1 (/32): 0x00000000
(2) r2 (/32): 0x00000000
(3) r3 (/32): 0x00000000
(4) r4 (/32): 0x00000000
(5) r5 (/32): 0x00000000
(6) r6 (/32): 0x00000000
(7) r7 (/32): 0x00000000
(8) r8 (/32): 0x00000000
(9) r9 (/32): 0x00000000
(10) r10 (/32): 0x00000000
(11) r11 (/32): 0x00000000
(12) r12 (/32): 0x00000000
(13) sp (/32): 0x20000400
(14) lr (/32): 0xFFFFFFFF
(15) pc (/32): 0x08000300
(16) xPSR (/32): 0x01000000
(17) msp (/32): 0x20000400
(18) psp (/32): 0x00000000
(19) primask (/1): 0x00
(20) basepri (/8): 0x00
(21) faultmask (/1): 0x00
(22) control (/2): 0x00
(23) d0 (/64): 0x0000000000000000
(24) d1 (/64): 0x0000000000000000
(25) d2 (/64): 0x0000000000000000
(26) d3 (/64): 0x0000000000000000
(27) d4 (/64): 0x0000000000000000
(28) d5 (/64): 0x0000000000000000
(29) d6 (/64): 0x0000000000000000
(30) d7 (/64): 0x0000000000000000
(31) d8 (/64): 0x0000000000000000
(32) d9 (/64): 0x0000000000000000
(33) d10 (/64): 0x0000000000000000
(34) d11 (/64): 0x0000000000000000
(35) d12 (/64): 0x0000000000000000
(36) d13 (/64): 0x0000000000000000
(37) d14 (/64): 0x0000000000000000
(38) d15 (/64): 0x0000000000000000
(39) fpscr (/32): 0x00000000
===== Cortex-M DWT registers
(40) dwt_ctrl (/32)
(41) dwt_cyccnt (/32)
(42) dwt_0_comp (/32)
(43) dwt_0_mask (/4)
(44) dwt_0_function (/32)
(45) dwt_1_comp (/32)
(46) dwt_1_mask (/4)
(47) dwt_1_function (/32)
(48) dwt_2_comp (/32)
(49) dwt_2_mask (/4)
(50) dwt_2_function (/32)
(51) dwt_3_comp (/32)
(52) dwt_3_mask (/4)
(53) dwt_3_function (/32)
> 
``` 

We can change the contents of a register by issuing `reg reg_name
value`, for example `reg pc 0x20000000`.

```
> reg pc 0x20000000
pc (/32): 0x20000000
> reg
===== arm v7m registers
(0) r0 (/32): 0x00000000
(1) r1 (/32): 0x00000000
(2) r2 (/32): 0x00000000
(3) r3 (/32): 0x00000000
(4) r4 (/32): 0x00000000
(5) r5 (/32): 0x00000000
(6) r6 (/32): 0x00000000
(7) r7 (/32): 0x00000000
(8) r8 (/32): 0x00000000
(9) r9 (/32): 0x00000000
(10) r10 (/32): 0x00000000
(11) r11 (/32): 0x00000000
(12) r12 (/32): 0x00000000
(13) sp (/32): 0x20000400
(14) lr (/32): 0xFFFFFFFF
(15) pc (/32): 0x20000000 (dirty)
(16) xPSR (/32): 0x01000000
(17) msp (/32): 0x20000400
(18) psp (/32): 0x00000000
(19) primask (/1): 0x00
(20) basepri (/8): 0x00
(21) faultmask (/1): 0x00
(22) control (/2): 0x00
(23) d0 (/64): 0x0000000000000000
(24) d1 (/64): 0x0000000000000000
(25) d2 (/64): 0x0000000000000000
(26) d3 (/64): 0x0000000000000000
(27) d4 (/64): 0x0000000000000000
(28) d5 (/64): 0x0000000000000000
(29) d6 (/64): 0x0000000000000000
(30) d7 (/64): 0x0000000000000000
(31) d8 (/64): 0x0000000000000000
(32) d9 (/64): 0x0000000000000000
(33) d10 (/64): 0x0000000000000000
(34) d11 (/64): 0x0000000000000000
(35) d12 (/64): 0x0000000000000000
(36) d13 (/64): 0x0000000000000000
(37) d14 (/64): 0x0000000000000000
(38) d15 (/64): 0x0000000000000000
(39) fpscr (/32): 0x00000000
===== Cortex-M DWT registers
(40) dwt_ctrl (/32)
(41) dwt_cyccnt (/32)
(42) dwt_0_comp (/32)
(43) dwt_0_mask (/4)
(44) dwt_0_function (/32)
(45) dwt_1_comp (/32)
(46) dwt_1_mask (/4)
(47) dwt_1_function (/32)
(48) dwt_2_comp (/32)
(49) dwt_2_mask (/4)
(50) dwt_2_function (/32)
(51) dwt_3_comp (/32)
(52) dwt_3_mask (/4)
(53) dwt_3_function (/32)
> 
```

Now, let's load the `example0.bin` into memory on the discovery board.
This is done with the command `load_image example0.bin 0x20000000
bin`.  This places the contents of the file at address `0x20000000`
and onwards in the ram on the STM32.

```
> load_image example0.bin 0x20000000 bin
8 bytes written at address 0x20000000
downloaded 8 bytes in 0.000639s (12.226 KiB/s)
```

To make sure that the instructions have been loaded correctly we can
issue the command `arm disassemble 0x20000000 4`, which means "please
show me 4 disassembled instructions starting from address `0x20000000`
". 

```
> arm disassemble 0x20000000 4
0x20000000  0x2002    	MOVS	r0, #0x02
0x20000002  0x2103    	MOVS	r1, #0x03
0x20000004  0x180a    	ADDS	r2, r1, r0
0x20000006  0x4410    	ADD	r0, r2
```

Now when it seems that the code is in memory correctly we can step
through it and see if the operations have the expected results on the
registers.

The `step` command executes one instruction and then halts.

```
> step
target halted due to single-step, current mode: Thread 
xPSR: 0x01000000 pc: 0x20000002 msp: 0x20000400
``` 

The program counter has been moved forward to address `0x20000002`. Looks right. 
To inspect the the register that should have been altered by this instruction, issue 
the command `reg r0`. 

``` 
> reg r0
r0 (/32): 0x00000002
``` 

This looks quite correct after execution on the first instructions. 
Now I will step through the rest of the program and inspect the suitable registers. 

```
> step 
target halted due to single-step, current mode: Thread 
xPSR: 0x01000000 pc: 0x20000004 msp: 0x20000400
> reg r1
r1 (/32): 0x00000003
> step
target halted due to single-step, current mode: Thread 
xPSR: 0x01000000 pc: 0x20000006 msp: 0x20000400
> reg r2
r2 (/32): 0x00000005
> step
target halted due to single-step, current mode: Thread 
xPSR: 0x01000000 pc: 0x20000008 msp: 0x20000400
> reg r0
r0 (/32): 0x00000007
```

That seems fine! I really enjoy these low-level hands-on
exercises. Interacting with the discovery board like this is pretty
time consuming though and it would be nice to automate it.

# Automating the interaction with expect

One way to automate processes which involves interaction with a
terminal is to use the `expect` program. `expect` let's you write
scripts that play out an interaction for you. It seems like a quite
finicky system though and writing out an `expect` script by hand for a
multitude of interactions is a bit frustrating and boring.

I just briefly show some small examples of `expect` script interaction
with OpenOCD and the STM32F4. 

Remember how we earlier by hand launched `telnet`, then did a reset of
the board and set the program counter to `0x20000000`. Following this
we loaded a binary image to address `0x20000000`. With expect this can
be done as shown below. 


``` 
#!/usr/bin/expect -f
set timeout 60
spawn telnet -e ~ localhost 4444
expect -re "> ?$"
send -- "reset init\n"
expect -re "> ?$"
send -- "reg pc 0x20000000\n"
expect {
    "pc*0x20000000*\n" {
        puts "PC set successfully\n"
    }
    "pc*0x*\n" {
        puts "Error setting pc\n"
        exit 1
    }
}
expect -re "> ?$"
send -- "load_image test0.bin 0x20000000\n"
expect -re "> ?$"
send -- "~" 
expect "telnet>"
send -- "close\n"
expect eof
wait
``` 

The line `spawn telnet -e ~ localhost 4444` starts telnet and tells it to use `~` as the escape character. 
This means that a `~` can be sent to break out and exit from telnet. 

After launching telnet, expect will wait for the reception of a prompt `expect -re "> ?$`.
After reception of the prompt, we send the `reset init` command. 


# Multi-staged programming and test-generation

So, to automate this a bit further, it would be very fun to be able to
express tests that create an instruction sequence and where we at the
same time could express what results we should get during interaction
with the STM32 with that sequence of instructions loaded into memory. 

This testing system should consist of a C program that creates both an 
instruction sequence binary file and an expect script that can be launched. 
The expect script should automatically load the binary into memory and perform a sequence 
of tests upon it that are also expressed in the C program.

Two files are involved in the generation of expect test scripts, `test_expect.h` and `test_expect.c`. 
These files contains the definitions of a small number of functions. 

```
extern void test_step(void);
extern void test_assert_reg(char *reg, uint32_t value);
extern int test_expect_init(const char *testname);
extern void test_expect_shutdown(void);
``` 


`test_expect_init` expects a test-name which will correspond to the
generated expect script (with .expect appended to it).  The test
system also assumes that the binary file is also has filename
`testname` (but with .bin appended to it). 

`test_step` issues a step command into the expect file to instruct OpenOCD to step an instructions. 
`test_assert_reg` inserts a command into the expect file that inspects a register and compares it's value 
to the argument called `value`. 

Given these tools we can augment the code from earlier that generated
the binary file we have been using.

```
uint16_t instrs[4];
instr_seq_t seq;
seq.size = 4;
seq.pos  = 0;
seq.mc = instrs;

if (!test_expect_init("example0")) {
  printf("error initializing test_expect\n");
  return 0;
}
 
emit_opcode(&seq, m0_mov_imm(r0, 2));
test_step();
test_assert_reg("r0", 2);
emit_opcode(&seq, m0_mov_imm(r1, 3));
test_step();
test_assert_reg("r1", 3);
emit_opcode(&seq, m0_add_low(r2, r1, r0));
test_step();
test_assert_reg("r2", 5);
emit_opcode(&seq, m0_add_any(r0, r2));
test_step();
test_assert_reg("r0", 7);

test_expect_shutdown();

```

`test_expect_shutdown` outputs the file `example0.expect` with the
following contents.

``` 
#!/usr/bin/expect -f
set timeout 60
spawn telnet -e ~ localhost 4444
expect -re "> ?$"
send -- "reset init\n"
expect -re "> ?$"
send -- "reg pc 0x20000000\n"
expect {
    "pc*0x20000000*\n" {
        puts "PC set successfully\n"
    }
    "pc*0x*\n" {
        puts "Error setting pc\n"
        exit 1
    }
}
expect -re "> ?$"
send -- "load_image example0.bin 0x20000000 bin\n"
expect -re "> ?$"
send -- "step\n"
expect -re "> ?$"
send -- "reg r0\n"
expect {
    "r0*0x00000002*\n" {
        puts "r0 set successfully\n"
    }
    "r0*0x*\n" {
        puts "Error setting r0\n"
        exit 1
    }
}
expect -re "> ?$"
send -- "step\n"
expect -re "> ?$"
send -- "reg r1\n"
expect {
    "r1*0x00000003*\n" {
        puts "r1 set successfully\n"
    }
    "r1*0x*\n" {
        puts "Error setting r1\n"
        exit 1
    }
}
expect -re "> ?$"
send -- "step\n"
expect -re "> ?$"
send -- "reg r2\n"
expect {
    "r2*0x00000005*\n" {
        puts "r2 set successfully\n"
    }
    "r2*0x*\n" {
        puts "Error setting r2\n"
        exit 1
    }
}
expect -re "> ?$"
send -- "step\n"
expect -re "> ?$"
send -- "reg r0\n"
expect {
    "r0*0x00000007*\n" {
        puts "r0 set successfully\n"
    }
    "r0*0x*\n" {
        puts "Error setting r0\n"
        exit 1
    }
}
expect -re "> ?$"
send -- "~" 
expect "telnet>"
send -- "close\n"
expect eof
wait
``` 

If you also output the `example0.bin`, like this. 

```
FILE *fp = fopen("example0.bin", "w");
if (!fp)  {
  printf("Error opening file %s\n", fn);
  return 0;
}

if (fwrite(seq.mc,sizeof(uint16_t),4,fp) < 4) {
  printf("Error writing file\n");
  return 0;
}
```

You now have a pair of files that can be auto-tested on the STM32. 
I run my tests from a script that opens OpenOCD in the background 
and then runs all expect scripts in the directory. 

Output from this automated interactions looks something like this. 

```
Open On-Chip Debugger 0.10.0
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
adapter speed: 2000 kHz
adapter_nsrst_delay: 100
none separate
srst_only separate srst_nogate srst_open_drain connect_deassert_srst
Info : Unable to match requested speed 2000 kHz, using 1800 kHz
Info : Unable to match requested speed 2000 kHz, using 1800 kHz
Info : clock speed 1800 kHz
spawn telnet -e ~ localhost 4444
Telnet escape character is '~'.
Trying 127.0.0.1...
Connected to localhost.
Escape character is '~'.
Info : STLINK v2 JTAG v25 API v2 SWIM v14 VID 0x0483 PID 0x374B
Info : using stlink api v2
Info : Target voltage: 2.878307
Info : stm32f4x.cpu: hardware has 6 breakpoints, 4 watchpoints
Info : accepting 'telnet' connection on tcp/4444
Open On-Chip Debugger
> reset init
Info : Unable to match requested speed 2000 kHz, using 1800 kHz
Unable to match requested speed 2000 kHz, using 1800 kHz
Info : Unable to match requested speed 2000 kHz, using 1800 kHz
adapter speed: 1800 kHz
Unable to match requested speed 2000 kHz, using 1800 kHz
adapter speed: 1800 kHz
target halted due to debug-request, current mode: Thread 
xPSR: 0x01000000 pc: 0x08000300 msp: 0x20000400
target halted due to debug-request, current mode: Thread 
xPSR: 0x01000000 pc: 0x08000300 msp: 0x20000400
Info : Unable to match requested speed 8000 kHz, using 4000 kHz
Info : Unable to match requested speed 8000 kHz, using 4000 kHz
Unable to match requested speed 8000 kHz, using 4000 kHz
adapter speed: 4000 kHz
Unable to match requested speed 8000 kHz, using 4000 kHz
adapter speed: 4000 kHz
> reg pc 0x20000000
PC set successfully

pc (/32): 0x20000000
pc (/32): 0x20000000
> load_image example0.bin 0x20000000 bin
8 bytes written at address 0x20000000
downloaded 8 bytes in 0.000532s (14.685 KiB/s)
8 bytes written at address 0x20000000
downloaded 8 bytes in 0.000532s (14.685 KiB/s)
> step
target halted due to single-step, current mode: Thread 
xPSR: 0x01000000 pc: 0x20000002 msp: 0x20000400
Info : halted: PC: 0x20000002
target halted due to single-step, current mode: Thread 
xPSR: 0x01000000 pc: 0x20000002 msp: 0x20000400
halted: PC: 0x20000002
> reg r0
r0 (/32): 0x00000002
r0 (/32): 0x00000002
> r0 set successfully

step
target halted due to single-step, current mode: Thread 
xPSR: 0x01000000 pc: 0x20000004 msp: 0x20000400
Info : halted: PC: 0x20000004
target halted due to single-step, current mode: Thread 
xPSR: 0x01000000 pc: 0x20000004 msp: 0x20000400
halted: PC: 0x20000004
> reg r1
r1 (/32): 0x00000003
r1 (/32): 0x00000003
> r1 set successfully

step
target halted due to single-step, current mode: Thread 
xPSR: 0x01000000 pc: 0x20000006 msp: 0x20000400
Info : halted: PC: 0x20000006
target halted due to single-step, current mode: Thread 
xPSR: 0x01000000 pc: 0x20000006 msp: 0x20000400
halted: PC: 0x20000006
> reg r2
r2 (/32): 0x00000005
r2 (/32): 0x00000005
> r2 set successfully

step
target halted due to single-step, current mode: Thread 
xPSR: 0x01000000 pc: 0x20000008 msp: 0x20000400
Info : halted: PC: 0x20000008
target halted due to single-step, current mode: Thread 
xPSR: 0x01000000 pc: 0x20000008 msp: 0x20000400
halted: PC: 0x20000008
> reg r0
r0 (/32): 0x00000007
r0 (/32): 0x00000007
> r0 set successfully


telnet> close
Connection closed.
Info : dropped 'telnet' connection
``` 

# Conclusion 

There are a lot of instructions in the ARM Thumb1 and Thumb2
instruction sets. Well not a lot compared to X86, but still a lot if
manual interaction is needed for every test. Still, this automation
still leaves a lot of repetitive work if everything is to be tested. 
Will look for further automation as I progress. 

I hope you enjoyed reading and do not hesitate to contact me with questions 
if you think any of what I do is fun or interesting in any way :)

Thanks!

___

[HOME](https://svenssonjoel.github.io)
