

# Programming of ARM Cortex-M microcontrollers - Part 0 - Hard-fault on reset and some GPIO

In the
[previous](https://svenssonjoel.github.io/pages-2021/cortex-m-assembler-0/index.html)
post I tried to get some code written in assembly to run on my
STM32F4-Discovery. Now, this confuses me a lot, but I was able to run
that code in a GDB session against the board but after a reset the MCU
kept ending up in a "hard-fault".

The fix of this problem is what is really confusing me. To make it work
the `reset_handler` should have a `.thumb_func` directive before it. Like this:

```
.thumb_func	
reset_handler:
	ldr r0, =_estack
	mov sp, r0
		
	ldr r0, =_dstart
	ldr r1, =_dend

	sub r2,r1,r0
	cmp r2, #0
	beq main

	ldr r1, =_flash_dstart
```

*Note* In this version of the reset handler I am checking if there are
more than 0 bytes to copy to ram, if not it jumps to label
`main`. This check was not present in the
[previous](https://svenssonjoel.github.io/pages-2021/cortex-m-assembler-0/index.html)
post.


The `.thumb_func` directive before `reset_handler` has a very subtle effect. To see
the effect let's look at some `objdump` output.


**Without `.thumb_func`**

```
08000000 <vtable>:
 8000000:	20020000 	.word	0x20020000
 8000004:	08000012 	.word	0x08000012
 8000008:	00000000 	.word	0x00000000
 800000c:	08000010 	.word	0x08000010

08000010 <hard_fault_handler>:
 8000010:	e7fe      	b.n	8000010 <hard_fault_handler>

08000012 <reset_handler>:
 8000012:	4811      	ldr	r0, [pc, #68]	; (8000058 <done+0x2>)
```

**With `.thumb_func`**

```
Disassembly of section .text:

08000000 <vtable>:
 8000000:	20020000 	.word	0x20020000
 8000004:	08000013 	.word	0x08000013
 8000008:	00000000 	.word	0x00000000
 800000c:	08000011 	.word	0x08000011

08000010 <hard_fault_handler>:
 8000010:	e7fe      	b.n	8000010 <hard_fault_handler>

08000012 <reset_handler>:
 8000012:	4811      	ldr	r0, [pc, #68]	; (8000058 <done+0x2>)
```

Now if you look at the second line of the `vtable` it says
` 8000004:      08000012        .word   0x08000012` without `.thumb_func` and
` 8000004:      08000013        .word   0x08000013` with it. What this line is
supposed to be is a pointer to the `reset_handler` function. But then in both cases
the actual address of the `reset_handler` is `8000012`. As I understand it, the least
significant bit is actually discarded when reset interrupt invokes the pointer (this may
be what happens for all function calls) and the least significant bit has the role
of being a flag indicating ARM-mode or Thumb-mode.

Now! This is something I would love some feedback or pointers on
because it confuses me extremely. Aren't the Cortex-M MCUs THUMB-only architectures?
I mean there is the larger THUMB instruction set for M3 and M4 and a smaller one for M0, and M1.
I think these are referred to as Thumb and Thumb2. Those larger Thumb instructions on
the M4 for example are 32bit instructions, but they are not the ARM instruction set.

If the paragraph above is true, why would we even need a flag to
differentiate between thumb and arm?

Ah, that was a long addendum to the
[previous](https://svenssonjoel.github.io/pages-2021/cortex-m-assembler-0/index.html)
post in the series! Let's take a look at some GPIO


## Enable GPIO and turn on some LEDs

Now that we can really assemble programs that "work" it would be nice to add
some visual feedback to what we are doing. So let's turn on some LEDs!

The complete assembly code listing will be shown further down in the text. First
we can take a look at what is added/changed in comparison to the code in the earlier post.


```
main:
	ldr r1, =0x40023830
	ldr r0, [r1]
	orr r0, 0x1
	str r0, [r1]
	
	ldr r1, =0x40020000     @ Pointer to PA MODER
	ldr r0, [r1]            @ Value of PA MODER
	orr r0, r0, 0x55        @ PA0 - PA 3 output set 
        str r0, [r1]            @ Write back PA MODER

	ldr r1, =0x40020014     @ PA output data register
	ldr r0, [r1]
	orr r0, r0, 0xF         @ Set all PA0 - PA3 to 1
	str r0, [r1]            @ Write back data register
	
``` 

The code above is split into 3 chunks of related instructions. The first chunk
sets a bit in the `RCC_AHB1ENR` register. This register is located with an offset of
`0x30` from the RCC base address which is `0x40023800`. This info can be found in
the reference manual (linked at the end of this text).

Bit 0 of this `RCC_AHB1EN` register is called `GPIOAEN` in the reference manual and
setting this bit activates the GPIO A port (the PA pins on the Discovery board).

The second chunk, sets a bit pattern into the `MODER` register associated with GPIO A.
The GPIO A peripheral base address is `0x40020000` and the `MODER` register is at offset 0.
The `MODER` register associates two bits with each of the GPIO pins of the port.
The decimal value `5` is `0101` in binary and thus `0x55` is `01010101` in binary.
This pattern sets the 4 GPIO pins PA0 - PA3 into output mode.

The last chunk of code sets the 4 least significant bits of the data
output register associated with GPIO A. This register is at offset
`0x14` from the GPIO A peripheral base address. This sets PA0 - PA3
high and the LEDs connected there should light up. Note that the
discovery board does not have any LEDs on these pins by default so a
breadboard and some LEDs will be needed. 

Below is the full assembly listing. 

```

	.syntax unified
	.cpu cortex-m4
	.thumb
	
	.global vtable
	.global reset_handler

	.section .text
	
vtable:
	.word _estack
	.word reset_handler
	.word 0
	.word hard_fault_handler

.thumb_func	
hard_fault_handler:
	b hard_fault_handler

.thumb_func	
reset_handler:
	ldr r0, =_estack
	mov sp, r0
		
	ldr r0, =_dstart
	ldr r1, =_dend

	sub r2,r1,r0
	cmp r2, #0
	beq main

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
	ldr r1, =0x40023830
	ldr r0, [r1]
	orr r0, 0x1
	str r0, [r1]
	
	ldr r1, =0x40020000     @ Pointer to PA MODER
	ldr r0, [r1]            @ Value of PA MODER

	orr r0, r0, 0x55        @ PA0 - PA 3 output set 
        str r0, [r1]            @ Write back PA MODER

	ldr r1, =0x40020014     @ PA output data register
	ldr r0, [r1]
	orr r0, r0, 0xF         @ Set all PA0 - PA3 to 1
	str r0, [r1]            @ Write back data register
	
	
	
done:	
	b done

	.section .data 
``` 


## Conclusion

Wow! That hard-fault issue was very subtle. Not the easiest thing to google an answer
for either as it seems a hard-fault can be just about anything at this stage.

Stay in touch! Have a great day


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
