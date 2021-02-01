
#  programming of ARM Cortex-M microcontrollers - Part 0 - Tools and basics

I think it may be pretty hard to justify picking up assembly language
at this time unless if for learning or for fun. I'm going to have a go
at it for both of those reasons. As usual, I will share the process
with you here. Do not hesitate to interact over email or via the google
group. Together we can learn more!

Assembler programming should give opportunity to learn more about some
(for a high-level programmer) perhaps mysterious things. In this first text 
we will look at some assembler code, tools (such as assembler, linker, gdb and OpenOCD)
and the linker script.

The platform I will target is ARM Cortex-M4. There are many different
development boards with MCUs of this architecture. Mine are from
[ST](https://www.st.com). So while experimenting with this I will
target stm32F4 of the various kinds I have access to
(STM32F411E-DISC0, STM32F407G-DISC1, Various home-made STM32F405RGT6
based boards).

There are great tutorials on Cortex-M programming. One that is
definitely worth looking at can be found here:
[https://vivonomicon.com](https://vivonomicon.com/2018/04/02/bare-metal-stm32-programming-part-1-hello-arm/).

## The tools 

I like open source software, so of course I am doing this work on a
Linux machine. If you are not using Linux, then I think that you should. 
Go here and get [Ubuntu](https://ubuntu.com/) for example. 

The development tools needed for ARM Cortex-M cross compilation (and
assembling) can be found here: [arm-none-eabi- cross compilation
tools](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads).
Unpack this `tar.bz2` archive somewhere and add the `bin` directory to
the PATH variable. 

OpenOCD, the open on-chip debugger, that can be used to "flash"
(program) the boards and for debugging can be found here: [OpenOCD](https://sourceforge.net/p/openocd/code/ci/master/tree/).

That should be more or less what is needed to get started. Some other
tools will be useful, like the "make" tool. This is something that
should be available on most Linux systems more or less per default. On
a fresh Ubuntu install one may need to `sudo apt-get install
build-essential` to get `make`.


## Baremetal microcontroller programming

The purpose here is to get up and running with some kind of first 
little program and get it to run on a development board. Even for such 
a humble goal there are a lot of details to look at though. 

The microcontroller has flash memory (1024KiB), think of this as
read-only memory once it has been flashed. In the flash memory we will
store the program. The program will not be loaded from the flash into
ram at startup, rather it will be executed directly from flash. The
MCU of course has RAM as well. 192KiB in the case of the
microcontrollers I use. The ram will be used for storage of variables
that the program use. The ram is split into two different
categories. There is a 128K RAM and a 64K core-coupled ram. What this
core-coupled ram is about is left as thing to look at in the future.

The [reference
manual](https://www.st.com/content/ccc/resource/technical/document/reference_manual/3d/6d/5a/66/b4/99/40/d4/DM00031020.pdf/files/DM00031020.pdf/jcr:content/translations/en.DM00031020.pdf)
contains information about the memory map of the system. Here you can
find information on where in the 32bit address space you find RAM,
FLASH and memory mapped peripheral devices and so on. What we need 
to know right now is that flash starts at address `0x08000000` 
and that RAM starts at address `0x20000000`.

## The linker script

Using the information above, we can construct a linker script. The
linker script is used during the linking phase that combines some
number of object files into a single `elf` file. The linker script
holds information about the memory of the system and of where in
memory to place different things. The code should be placed in flash,
variables that are initialized should have their initial values in
flash but be copied into ram at startup. So, that is the kind of info
that goes in the linker script.

We can also define various "symbols" in the linker script that we 
can later access from out assembly program. Let's talk about those when 
we get to them. 

First of all, the linker script holds a description of the memory
of the system

``` 
MEMORY
{
FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 1024K
CCRAM (xrw) : ORIGIN = 0x10000000, LENGTH = 64K
RAM (xrw)   : ORIGIN = 0x20000000, LENGTH = 128K
}
```

This just names some memory areas, here "FLASH", "CCRAM" and
"RAM". "CCRAM" is something we wont use and it could have been left
out from the script. 

Each memory is followed by some attributes that specify if they are
`r` read, `x` executable and `w` writable. Then there is an starting
address of the memory range, declared using the `ORIGIN =` syntax
followed by the length of the memory (size in bytes).


Next the linker script specifies sections. Two sections so far. 
The first is called `.text` and will hold the code and another called `.data`
for variables. 

``` 
SECTIONS
{

.text : {	
      *(.text)
}>FLASH

_flash_dstart = .;

.data :  {
      _dstart = .;
      *(.data)
      _dend = .; 
}>RAM AT> FLASH  /* Load into FLASH, but live in RAM */

} /* SECTIONS END */ 
``` 

The syntax below (as I understand it) says that there should be a text section in 
the output binary after linking and that it will contain `*(.text)` everything 
from all text sections in the files that are linked together. 

``` 
.text : {	
      *(.text)
}>FLASH
```

The `>FLASH` part means that this should be located in the flash memory. 

For the `.data` section this part looks like this: 

```
>RAM AT> FLASH  /* Load into FLASH, but live in RAM */
```

and as I understand it that means that it should be written to flash,
but at runtime the address to access these "data" will be in RAM. So, 
that means that we have to move this data from the flash to the RAM as 
part of starting up the system later. 

There are also a couple of symbols defined in the linker script
`_estack`, `_flash_dstart`, `_dstart` and `_dend`.  Those are all
defined using the `.` which means "current location". The linker will
keep a location counted while processing the script and the `.` is the
way to access the current value of that counter. It is essentially an
address into the binary where we are currently at.

The symbols that are defined in the linker script can be accessed in
the assembly code.  The `_dstart` and `_dend` values will be used to
compute how many bytes to copy from flash to ram at startup in order
to move the `.data` section to ram. `_flash_dstart` is the address in
flash where the `.data` section is initially stored and `_dstart` is
the address in RAM where it will end up after copy.

Below is the linker script in full. These files are usually given the
`.ld` file extension. 

``` 
_estack = 0x20020000;

MEMORY
{
FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 1024K
CCRAM (xrw) : ORIGIN = 0x10000000, LENGTH = 64K
RAM (xrw)   : ORIGIN = 0x20000000, LENGTH = 128K
}


SECTIONS
{

.text : {	
      *(.text)
}>FLASH

_flash_dstart = .;

.data :  {
      _dstart = .;
      *(.data)
      _dend = .; 
}>RAM AT> FLASH  /* Load into FLASH, but live in RAM */

} /* SECTIONS END */ 
``` 



## Assembler code

There are a number of different syntactic constructs in an assembler
source file. Directives for the assembler start with a `.`, for
example `.syntax unified`. `syntax unified` seems to be about ARM vs
Thumb instruction syntax, and "unified" fits both into one style. I am
not sure about the details about this yet. 

Thumb vs ARM is interesting in general. As I understand it the
Cortex-M4 only runs Thumb (Thumb2 to be precise) while other
non-cortex-M architectures can run both Thumb and ARM
instructions. This whole Thumb, Thumb2 and ARM instruction set
business is quite confusing! 

In the assembler source you will also find labels that are strings
ending in a `:` and left justified. These provide a name to the address 
of the directly following instruction. 


The assembler file starts out with some directives followed by the
definition of a vector table (part of a vector table).

```
	.syntax unified

	.section .text 
	
	.global vtable
	.global reset_handler

vtable:
	.word _estack
	.word reset_handler
	.size vtable, .-vtable
```

The vector table (or interrupt vector table of ISR table) is a
collection of function pointers which are invoked by the MCU on
interrupts. My knowledge about this is very thin and the `vtable` is
definitely something to investigate in more detail later.  The [STM32
Cortex®-M4 MCUs and MPUs programming
manual](https://www.st.com/content/ccc/resource/technical/document/programming_manual/6c/3a/cb/e7/e4/ea/44/9b/DM00046982.pdf/files/DM00046982.pdf/jcr:content/translations/en.DM00046982.pdf)
seems like a good source of information about this table. 

For now I think it is enough to assume that when the MCU is reset, the
first thing it does is jump to the `reset_handler` stored in the
`vtable`. 

The reset handler loads the `_estack` value into the `sp` register
(stack pointer register). It then loads the start and end of the data
section into registers to calculate the number of bytes of variable
data. Then a loop copies data from flash to RAM. 


```
reset_handler:
	ldr r0, =_estack
	mov sp, r0

	ldr r0, =_dstart
	ldr r1, =_dend

	sub r2,r1,r0

	ldr r1, =_flash_dstart
	
cpy_loop:
	ldrb r3, [r1]
	strb r3, [r0] 

	add r1, r1, #1
	add r0, r0, #1
	
	sub r2, r2, #1
	cmp r2, #0
	bne cpy_loop
```

After completing the loop execution falls through into the code below. 


```
main:
	ldr r9, =apa
	ldr r9, [r9]

	ldr r7, =0xF00DF00D
	ldr r8, =0x1337BEEF
```
The code above, loads some easily recognizable values into registers so that 
it is easy to see what is going on when later running the program. 
The `=apa` that is loaded into `r9` is interesting. This is a variable 
in the data section that we will see below. 

After setting those registers the program goes into an infinite loop. 

```
done:	
	b done
```

Next comes the data section with the declaration of the `apa`. 

```
	.section .data 

apa:	
	.word 0xFEEBDAED

``` 

This text is not really about understanding the details of that
assembler program. Rather it is about getting something to run and
laying a foundation that we can build further experiments and
understanding on top of.

Below you see he complete assembly listing

```
	.syntax unified

	.section .text 
	
	.global vtable
	.global reset_handler

vtable:
	.word _estack
	.word reset_handler
	.size vtable, .-vtable

reset_handler:
	ldr r0, =_estack
	mov sp, r0

	ldr r0, =_dstart
	ldr r1, =_dend

	sub r2,r1,r0

	ldr r1, =_flash_dstart
	
cpy_loop:
	ldrb r3, [r1]
	strb r3, [r0] 

	add r1, r1, #1
	add r0, r0, #1
	
	sub r2, r2, #1
	cmp r2, #0
	bne cpy_loop
	

main:
	ldr r9, =apa
	ldr r9, [r9]

	ldr r7, =0xF00DF00D
	ldr r8, =0x1337BEEF

done:	
	b done

	.section .data 

apa:	.word 0xFEEBDAED
```

## Assembling 

I stored the assembler code in a file called `session0.s` and to
assemble that file and produce an object file one issues the following
command.

```
arm-none-eabi-as -g -mcpu=cortex-m4 -mthumb session0.s -o session0.o
```

This uses the gnu assembler with the flags `-g` for debug info,
`-mcpu=cortex-m4` for the target architecture and `-mthumb`.... I
wonder if `-mthumb` is really needed as the cortex-m4 don't execute
anything but thumb. oh well. Then the input file is specified and
after the `-o` the output file.

## Linking

Linking takes a linker script and some number of `.o` files (in this case 
only one `.o` file) and outputs an elf file. 

Now, there is a bit of mystique in this area as well. The `.o` files
are really also ELF files and what the linker does is combining some
number such into a target ELF. But it also takes the information from
within the linker script in consideration and performs relocation. 

Let's look at the command for linking and then take a look at how `.o` file 
is different from the `.elf` file. 

```
arm-none-eabi-ld session0.o -T ./session0.ld -o session0.elf
``` 

After the assembly step and the linking step we should have a file called 
`session0.o` and one called `session0.elf`. 

To checkout the contents of these files in a format that makes any sense at 
all, we can use the `objdump` program. For example like this. 

``` 
arm-none-eabi-objdump -s -d  session0.o
``` 
The `-s` means "display the full contents of all sections requested" and 
the `-d` means "perform disassembly". 

**session0.o** 

```
session0.o:     file format elf32-littlearm

Contents of section .text:
 0000 00000000 00000000 0c488546 0c480d49  .........H.F.H.I
 0010 a1eb0002 0c490b78 037001f1 010100f1  .....I.x.p......
 0020 0100a2f1 0102002a f5d1dff8 2090d9f8  .......*.... ...
 0030 0090074f dff81c80 fee70000 00000000  ...O............
 0040 00000000 00000000 00000000 00000000  ................
 0050 0df00df0 efbe3713                    ......7.        
Contents of section .data:
 0000 eddaebfe                             ....            
Contents of section .ARM.attributes:
 0000 41200000 00616561 62690001 16000000  A ...aeabi......
 0010 05436f72 7465782d 4d340006 0d074d09  .Cortex-M4....M.
 0020 02                                   .               

Disassembly of section .text:

00000000 <vtable>:
	...

00000008 <reset_handler>:
   8:	480c      	ldr	r0, [pc, #48]	; (3c <done+0x4>)
   a:	4685      	mov	sp, r0
   c:	480c      	ldr	r0, [pc, #48]	; (40 <done+0x8>)
   e:	490d      	ldr	r1, [pc, #52]	; (44 <done+0xc>)
  10:	eba1 0200 	sub.w	r2, r1, r0
  14:	490c      	ldr	r1, [pc, #48]	; (48 <done+0x10>)

00000016 <cpy_loop>:
  16:	780b      	ldrb	r3, [r1, #0]
  18:	7003      	strb	r3, [r0, #0]
  1a:	f101 0101 	add.w	r1, r1, #1
  1e:	f100 0001 	add.w	r0, r0, #1
  22:	f1a2 0201 	sub.w	r2, r2, #1
  26:	2a00      	cmp	r2, #0
  28:	d1f5      	bne.n	16 <cpy_loop>

0000002a <main>:
  2a:	f8df 9020 	ldr.w	r9, [pc, #32]	; 4c <done+0x14>
  2e:	f8d9 9000 	ldr.w	r9, [r9]
  32:	4f07      	ldr	r7, [pc, #28]	; (50 <done+0x18>)
  34:	f8df 801c 	ldr.w	r8, [pc, #28]	; 54 <done+0x1c>

00000038 <done>:
  38:	e7fe      	b.n	38 <done>
	...
  4e:	0000      	.short	0x0000
  50:	f00df00d 	.word	0xf00df00d
  54:	1337beef 	.word	0x1337beef
``` 


**session0.elf**

```
session0.elf:     file format elf32-littlearm

Contents of section .text:
 8000000 00000220 08000008 0c488546 0c480d49  ... .....H.F.H.I
 8000010 a1eb0002 0c490b78 037001f1 010100f1  .....I.x.p......
 8000020 0100a2f1 0102002a f5d1dff8 2090d9f8  .......*.... ...
 8000030 0090074f dff81c80 fee70000 00000220  ...O........... 
 8000040 00000020 04000020 58000008 00000020  ... ... X...... 
 8000050 0df00df0 efbe3713                    ......7.        
Contents of section .data:
 20000000 eddaebfe                             ....            
Contents of section .ARM.attributes:
 0000 41200000 00616561 62690001 16000000  A ...aeabi......
 0010 05436f72 7465782d 4d340006 0d074d09  .Cortex-M4....M.
 0020 02                                   .               

Disassembly of section .text:

08000000 <vtable>:
 8000000:	20020000 	.word	0x20020000
 8000004:	08000008 	.word	0x08000008

08000008 <reset_handler>:
 8000008:	480c      	ldr	r0, [pc, #48]	; (800003c <done+0x4>)
 800000a:	4685      	mov	sp, r0
 800000c:	480c      	ldr	r0, [pc, #48]	; (8000040 <done+0x8>)
 800000e:	490d      	ldr	r1, [pc, #52]	; (8000044 <done+0xc>)
 8000010:	eba1 0200 	sub.w	r2, r1, r0
 8000014:	490c      	ldr	r1, [pc, #48]	; (8000048 <done+0x10>)

08000016 <cpy_loop>:
 8000016:	780b      	ldrb	r3, [r1, #0]
 8000018:	7003      	strb	r3, [r0, #0]
 800001a:	f101 0101 	add.w	r1, r1, #1
 800001e:	f100 0001 	add.w	r0, r0, #1
 8000022:	f1a2 0201 	sub.w	r2, r2, #1
 8000026:	2a00      	cmp	r2, #0
 8000028:	d1f5      	bne.n	8000016 <cpy_loop>

0800002a <main>:
 800002a:	f8df 9020 	ldr.w	r9, [pc, #32]	; 800004c <done+0x14>
 800002e:	f8d9 9000 	ldr.w	r9, [r9]
 8000032:	4f07      	ldr	r7, [pc, #28]	; (8000050 <done+0x18>)
 8000034:	f8df 801c 	ldr.w	r8, [pc, #28]	; 8000054 <done+0x1c>

08000038 <done>:
 8000038:	e7fe      	b.n	8000038 <done>
 800003a:	0000      	.short	0x0000
 800003c:	20020000 	.word	0x20020000
 8000040:	20000000 	.word	0x20000000
 8000044:	20000004 	.word	0x20000004
```

The two files are quite similar. The addresses along the left hand
side of the disassembly are different though. In the `.elf` file these
addresses are changed to match the "constraints" (or what to call it)
that we imposed via the linker script. 



## Creating a `.hex` file for flashing

There is one more file we can create, a `.hex` file for flashing of the 
MCU. As I understand these, they are just a compact representation of the `.elf`
file generated above. The `.hex` file is created using the following command. 

``` 
arm-none-eabi-objcopy -O ihex session0.elf session0.hex
```
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg


## Flashing and debugging using OpenOCD 

To flash the mcu with the `.hex` file, issue this command. 

``` 
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg -c "init" -c "program session0.hex verify reset exit"
```

This call to `openocd` performs a number of commands against the MCU, the commands 
are what follows the  `-c`. Init the board and then program. 

We can also launch `openocd` without these "commands", like this. 

``` 
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg
```
What happens now is that OpenOCD connects to the boards and stays running. 
OpenOCD is running a telnet server that we can log onto and then manually 
enter the commands for programming, for example. To connect to OpenOCD 
using telnet enter. 

```
telnet localhost 4444
``` 

## Stepping through the program using GDB 

When launching OpenOCD like this: 

``` 
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg
```

it is also running a GDB server that we can connect to and interact with 
the program. 

So, run the `openocd` command above in one terminal and then in another 
start `gdb` like this. 

``` 
arm-none-eabi-gdb
``` 

When GDB has started it will give you a `(gdb)` prompt. Then enter:

```
target extended-remote :3333
``` 

GDB may then output something like this.

``` 
0x490d480c in ?? ()
(gdb) 
```
Enter: 

``` 
load session0.elf
```

GDB replies: 

``` 
Loading section .text, size 0x58 lma 0x8000000
Loading section .data, size 0x4 lma 0x8000058
Start address 0x8000000, load size 92
Transfer rate: 201 bytes/sec, 46 bytes/write.
(gdb) 
```

I am not sure why this is, but it seems that I also must run the
command `file session.elf` for things to behave properly, such as
stepping through programs. I wish to understand this better so if you 
have info, please share it with me! 


Now you can look at the contents of the registers by typing the `info
register` command. It should print something like this: 

```
r0             0x0                 0
r1             0x0                 0
r2             0x0                 0
r3             0x0                 0
r4             0x0                 0
r5             0x0                 0
r6             0x0                 0
r7             0x0                 0
r8             0x0                 0
r9             0x0                 0
r10            0x0                 0
r11            0x0                 0
r12            0x0                 0
sp             0x2001ffe0          0x2001ffe0
lr             0xfffffff9          -7
pc             0x8000000           0x8000000
xPSR           0x1000003           16777219
fpscr          0x0                 0
msp            0x20020000          0x20020000
psp            0x0                 0x0
primask        0x0                 0
basepri        0x0                 0
faultmask      0x0                 0
control        0x0                 0
```

Ok. At this point it works to step through the program using `step`. 
and after going through all the operations I end up with the 
following register state (which seems to make sense). 

``` 
(gdb) info register
r0             0x20000004          536870916
r1             0x800005c           134217820
r2             0x0                 0
r3             0xfe                254
r4             0x0                 0
r5             0x0                 0
r6             0x0                 0
r7             0xf00df00d          -267522035
r8             0x1337beef          322420463
r9             0xfeebdaed          -18097427
r10            0x0                 0
r11            0x0                 0
r12            0x0                 0
sp             0x20020000          0x20020000
lr             0xffffffff          -1
pc             0x8000038           0x8000038 <done>
xPSR           0x61000003          1627389955
fpscr          0x0                 0
msp            0x20020000          0x20020000
psp            0x0                 0x0
primask        0x0                 0
basepri        0x0                 0
faultmask      0x0                 0
control        0x0                 0
```

The key registers are `r7`, `r8` and `f9` containing "FOODFOOD", "LEETBEEF" 
and backwards "DEADBEEF", just as planned.


## Putting it all together in a Makefile

``` 
all: session0.hex

session0.o: session0.s
	arm-none-eabi-as -g -mcpu=cortex-m4 -mthumb session0.s -o session0.o

session0.elf: session0.o session0.ld
	arm-none-eabi-ld session0.o -T ./session0.ld -o session0.elf

session0.hex: session0.elf
	arm-none-eabi-objcopy -O ihex session0.elf session0.hex

clean:
	rm *.hex
	rm *.o
	rm *.elf

flash: session0.hex
	openocd -f interface/stlink.cfg -f target/stm32f4x.cfg -c "init" -c "program session0.hex verify reset exit"

debug:
	openocd -f interface/stlink.cfg -f target/stm32f4x.cfg

disas: session0.elf
	arm-none-eabi-objdump -s -d session0.elf
``` 


## Conclusion 

Lots of learning to do here and as you can tell I am quite confused at some 
parts. I hope to improve in understanding about these concepts over time 
and issue corrections here as needed. Any help, feedback, hints, tips and so 
on are very much welcome.

Have a good day!


## Additional resources 

1. [STM32F405/415 (407/417) (427/437) (429/439) reference
   manual](https://www.st.com/content/ccc/resource/technical/document/reference_manual/3d/6d/5a/66/b4/99/40/d4/DM00031020.pdf/files/DM00031020.pdf/jcr:content/translations/en.DM00031020.pdf)
2. [STM32F412xG/E reference manual](https://www.st.com/content/ccc/resource/technical/document/reference_manual/9b/53/39/1c/f7/01/4a/79/DM00119316.pdf/files/DM00119316.pdf/jcr:content/translations/en.DM00119316.pdf)
3. [STM32F405xx STM32F407xx datasheet](https://www.st.com/content/ccc/resource/technical/document/datasheet/ef/92/76/6d/bb/c2/4f/f7/DM00037051.pdf/files/DM00037051.pdf/jcr:content/translations/en.DM00037051.pdf)
4. [STM32F411xC STM32F411xE](https://www.st.com/content/ccc/resource/technical/document/datasheet/b3/a5/46/3b/b4/e5/4c/85/DM00115249.pdf/files/DM00115249.pdf/jcr:content/translations/en.DM00115249.pdf)
5. [STM32 Cortex®-M4 MCUs and MPUs programming manual](https://www.st.com/content/ccc/resource/technical/document/programming_manual/6c/3a/cb/e7/e4/ea/44/9b/DM00046982.pdf/files/DM00046982.pdf/jcr:content/translations/en.DM00046982.pdf)
6. [vivonomicon](https://vivonomicon.com/2018/04/02/bare-metal-stm32-programming-part-1-hello-arm/)






___

[HOME](https://svenssonjoel.github.io)
