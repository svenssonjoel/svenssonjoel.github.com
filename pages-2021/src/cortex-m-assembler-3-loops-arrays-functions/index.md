

# Assembler programming of ARM Cortex-M microcontrollers - Part 3 - Loops, arrays and functions 

This coding example cycles through some patterns stored in an array
and display these on 4 LEDs. The program will loop over the array of
patterns and for each iteration change what is output onto PA0 -
PA3. To slow down the switching between patterns a basic delay
function is implemented and called in each iteration of the loop.

The code written here will have a lot in common with the
[previous](https://svenssonjoel.github.io/pages-2021/cortex-m-assembler-2-gpio/index.html)
post. There wont be any input from the user though, no buttons.

A video of the result of the code shown here can be found on [youtube](https://youtu.be/U5KB64owiP0).


Let's start by looking at the delay function which is implemented as
a loop that just spins around for some number of iterations. 

```
delay:
        ldr r0,=1000000         @ Duration of delay in iterations
delay_loop:     
        cmp r0, #0              @ Are we done yet?
        beq delay_done          @ If so, jump out
        sub r0,r0, #1           @ Otherwise, dec counter
        b delay_loop            @ and do another iteration
delay_done:     
        bx lr                   @ Return from function call	
```

A "large" number is loaded into `r0` giving the number of iterations
to perform.  We then enter into the body of the loop which checks if
`r0` is 0. If `r0` is 0 we should break out of the loop and return. If
`r0` is larger than 0 then subtract 1 from `r0` and run the loop body
one more time.

When the control jumps to `delay_done`, when `r0` hits 0, `bx lr`
returns from the function call.

The value to use as a delay was decided by trial and error. I really
don't know how fast the MCU is executing right now (what MHz it is
running at). I understand that there is some really tricky "clock
configuration" stuff one should do. I don't know anything about how
that works yet! We should definitely look at it in the future.


So that's how to implement a function. A simple function. To call this
function you do:

```
        bl delay
```

`bl` is the branch and link operation. This instruction jumps to the
delay function but also stores away the return address into the `lr`
register.  So, the `bx lr` instruction at the end of the function,
restores the address from `lr` into the program counter.  This
approach to function calls does not use any stack so how to do
functions that call functions in a nested way is something to look at
in the future.


Now, let's look at the array of patterns.

```
        .section .data 

led_states:     .byte 0x1, 0x2, 0x4, 0x8, 0xF, 0x0
        @ led_states could have been stored in flash as it is constant. 
led_states_end:
```
In this case the array is placed in the `.data` section. This is the section
that we wrote some startup code for, to get it copied from flash into RAM.
The program will not try to modify this array, so it could just as well have
been a constant array remaining in flash only. To get that effect, just define 
the array in the `.text` section instead.

The `led_states:` symbol refers to the address where the array starts
and `led_states_end` refers to the first byte after any of the defined bytes
that are part of the array.

The data itself is just listed after the `.byte` directive. 


Now we can jump to the `main:` label and look at that part of the code piece by piece.

```
main:
        ldr r1, =0x40023830     @ Load address of AHB1ENR
        ldr r0, [r1]            @ Load contents of AHB1ENR
        orr r0, 0x1             @ Turn on GPIO A        
        str r0, [r1]            @ Store back modified AHB1ENR

        ldr r1, =0x40020000     @ Pointer to PA MODER
        ldr r0, [r1]            @ Value of PA MODER
        ldr r2, =0xFFFFFF00
        and r0, r0, r2
        orr r0, r0, 0x55        @ PA0 - PA 3 output, 
        str r0, [r1]            @ Write back PA MODER   
```
The code starts out similarly to previous texts. The GPIO A (PA) is turned on and
then 4 of the PA pins are set to be output pins. 

Next we should go into an infinite loop that updates the PA0 - PA3 pins.
A few preparatory instructions before looping though: 

```        
        ldr r2,=led_states      @ Load led_states array address
        ldr r6,=led_states_end  @ Load led_states array end address
        ldr r3,=0x40020014      @ PA output data register
        ldr r5,=0xFFFFFF00      @ Clear-mask for bits of interest
```
Here the registers `r2`,`r6`,`r3` and `r5` are set to their initial values for the loop.
Actually, only `r2` will be changed over the course of the loop and the other will be kept
constant. `r2` will initially point at the start of the patterns array.
`r6` will always point at the "end" of the patterns array. Lastly `r3` and `r5` will
hold the PA output data register address and the mask we use for clearing the outputs
that we are toggling.


```
forever:
        ldrb r1, [r2], #1       @ Load a byte from array and increment address
        ldr r4, [r3]            @ Load state of PA output data register
        and r4, r4, r5          @ Clear some bits
        orr r4, r4, r1          @ Turn some bits on based on led_states value
        str r4, [r3]            @ Write to PA output data register
```

The loop starts out by loading a byte from the `led_states` array from address stored in `r2`.
The address in `r2` is incremented in the same instruction there, `ldrb r1, [r2], #1`.
The rest of that code chunk is familiar, it is the code that sets some of the PA0-PA3 pins high
and thus turns on LEDs. 

Next we need to check if we have cycled through all the patterns and need to restart
from the beginning.

``` 
        cmp r2, r6              @ Did we reach end of array?
        blt forever_cont
        ldr r2,=led_states      @ Reset r2 to start of led_states
forever_cont:
        
        bl delay
        b forever
```

If `r2` which keeps track of where we are in the array becomes equal
to (or greater than) the value in `r6` (`let_states_end`), then we
should reset `r2` back to pointing at the start of the `led_states`
array. The logic is a bit backwards there as it rather checks if `r2` is less than `r6`
and if it is less it skips over the instructions that performs the reset of `r2`.

In either case, the code end ups at `bl delay` which calls the delay routine. Then
the `forever` loop is run once more. 

## The complete assembly listing

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
delay:
        ldr r0,=1000000         @ Duration of delay in iterations
delay_loop:     
        cmp r0, #0              @ Are we done yet?
        beq delay_done          @ If so, jump out
        sub r0,r0, #1           @ Otherwise, dec counter
        b delay_loop            @ and do another iteration
delay_done:     
        bx lr                   @ Return from function call	
        
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
        ldr r1, =0x40023830     @ Load address of AHB1ENR
        ldr r0, [r1]            @ Load contents of AHB1ENR
        orr r0, 0x1             @ Turn on GPIO A        
        str r0, [r1]            @ Store back modified AHB1ENR

        ldr r1, =0x40020000     @ Pointer to PA MODER
        ldr r0, [r1]            @ Value of PA MODER
        ldr r2, =0xFFFFFF00
        and r0, r0, r2
        orr r0, r0, 0x55        @ PA0 - PA 3 output, 
        str r0, [r1]            @ Write back PA MODER   
                

        
        ldr r2,=led_states      @ Load led_states array address
        ldr r6,=led_states_end  @ Load led_states array end address
        ldr r3,=0x40020014      @ PA output data register
        ldr r5,=0xFFFFFF00      @ Clear-mask for bits of interest
forever:
        ldrb r1, [r2], #1       @ Load a byte from array and increment address
        ldr r4, [r3]            @ Load state of PA output data register
        and r4, r4, r5          @ Clear some bits
        orr r4, r4, r1          @ Turn some bits on based on led_states value
        str r4, [r3]            @ Write to PA output data register

        cmp r2, r6              @ Did we reach end of array?
        blt forever_cont
        ldr r2,=led_states      @ Reset r2 to start of led_states
forever_cont:    
        
        bl delay
        b forever

        .section .data 

led_states:     .byte 0x1, 0x2, 0x4, 0x8, 0xF, 0x0
        @ led_states could have been stored in flash as it is constant. 
led_states_end: 
```

## Conclusion

Ah, that was one more GPIO intensive text but at least it was getting
into some interesting concepts... loops, arrays, simple functions. I
hope to dig deeper into that as time goes by.

Some other interesting things to append to the todo-list are:
1. Nested function calls. 
2. Clock configuration.
3. More ways of writing more "principled" or "structured" assembler. 

I would love feedback if you spot bugs or things that you would do
differently. Any kind of hints and tips are very much appreciated. So
do not hesitate to poke.

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
