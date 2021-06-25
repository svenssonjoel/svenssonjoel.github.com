

# Absolute time interrupts, mailboxes and memory-pools in ChibiOS

In this text I share the results of a recent experiment with some
ChibiOS concepts (mailbox and memory-pool) as well as some "custom"
configuration of a TIM timer. ChibiOs provides GPT drivers and as I
understand it you can use those to set relative time interrupt either
recurring or one-shot.  However, I wanted one-shot absolute time
interrupt, that is an interrupt that fires at a specific count value
of the counter.

I am told there may be benefits to absolute count interrupts compared
to relative. Of course, in an application where you have interrupts
that recur every n ticks of the ticks, setting a recurring GPT should
be just fine. So the use-case for this setup is slightly different,
imagine having a set of things to do at specific times relative the
starting the time of the application. If you can only set an interrupt
relative to the current time (such as 500ticks into the future from
now), then the computing and setting of these relative offsets may
introduce some jitter and potentially an accumulating drift of your
schedule. This is because the timer keeps counting while you are
handling the interrupt, so the time you compute for the next interrupt
becomes dependent on when you read out the counter value.

So, the plan is to implement an absolute time interrupt that sends
a message (allocated on a memory-pool) to a mailbox. A thread will
wait (block) on the mailbox until there is a message. 

## Configure TIM5 for absolute time interrupts

To begin with, let's configure TIM5 (one of the 32bit counters of the
STM32) into compare mode. My guess is that the ChibiOS GPT drivers
will do pretty much the same, but then only (as far as I can tell)
expose an API for setting relative interrupts. Since we are not using
the GPTs, we will do this configuration using a lower level API supplied
by ChibiOS.

We need to include two files. 

```
#include "stm32_tim.h" 
#include "stm32_rcc.h"
``` 

One thing that the `stm32_tim.h` header provides is a struct that
maps onto the register structure for configuration of the TIM, very helpful!
I show this `struct` below augmented with some comments.

```
typedef struct {
volatile uint32_t     CR1;      // Control register 1
volatile uint32_t     CR2;      // Control register 2
volatile uint32_t     SMCR;     // Slave mode control register
volatile uint32_t     DIER;     // DMA/Interrupt enable register
volatile uint32_t     SR;       // Status register
volatile uint32_t     EGR;      // Event generation register
volatile uint32_t     CCMR1;    // Capture/Compare mode register 1
volatile uint32_t     CCMR2;    // Capture/Compare mode register 2
volatile uint32_t     CCER;     // Capture/Compare enable register
volatile uint32_t     CNT;      // Count register.
volatile uint32_t     PSC;      // Prescaler (1 - 65535).
volatile uint32_t     ARR;      // Auto reload register.
volatile uint32_t     RCR;      
volatile uint32_t     CCR[4];   // Compare/Capture registers.
volatile uint32_t     BDTR;
volatile uint32_t     DCR;      // DMA control register
volatile uint32_t     DMAR;     // DMA Address for full transfer
volatile uint32_t     OR;       // Option register.
volatile uint32_t     CCMR3;    // Capture/compare mode register 3 
volatile uint32_t     CCXR[2];
} stm32_tim_t;

```

The `RCC` stands for "Reset and Clock Control". There are two
registers in the `RCC` group that are involved in the TIM
configuration, these are called `RCC_APB1ENR` and `RCC_APB1RSTR`.  The
`RCC_APB1ENR` register has enable bits for the TIMs and the
`RCC_APB1RSTR` has reset bits.  We wont deal with these registers
immediately as ChibiOS provides the functions `rccEnableTIM5` and
`rccResetTIM5` that performs the appropriate interaction with the RCC.


The TIM5 counter configuration will follow this plan:
1. Enable and reset TIM5.
2. Enable the interrupt vector associated with TIM5 in the nvic.
3. Get a handle to the TIM5 registers.
4. Configure the TIM5 Prescaler (a factor to divide down the input clock by).
5. Configure the TIM5 reload registers (the maximum number).
6. Activate "compare"
7. Activate interrupt
8. Init, update and start. 

There are 4 different `CCR`, compare/capture registers where a value can be set
and used to trigger an interrupt if interrupts are activated on that channel.
This application will only use of those. 

The TIM `CR1` register configures if the counter is up- or down-counting and
is also used to start the counter. There are 9 configurable bits in the CR1
register and we are going to set all of these to zero (to start with). At the
end, bit 0 of `CR1` is set to one to start enable the TIM. 

The TIM `EGR` register has an "update generation" bit. It seems that after making
changes to the other TIM registers, this "update generation" bit should be set to
make the updates stick immediately.  

Below you find the `setup_timer` function that I use with comments. I
also define a global `tim5` pointer as the low level interface to set
new "alarms" (set new interrupt time).

```
stm32_tim_t *tim5 = NULL;

void setup_timer(void) {

  
  rccEnableTIM5(true);
  rccResetTIM5();
  
  nvicEnableVector(STM32_TIM5_NUMBER, STM32_GPT_TIM5_IRQ_PRIORITY); /* use GPT level prio */

  
  tim5 = STM32_TIM5;  /* gives direct access to the tim5 registers */

  tim5->PSC = 0xFFFF;     /* counter rate is input_clock / (0xFFFF+1) */
  tim5->ARR = 0xFFFFFFFF; /* Value when counter should flip to zero. */

  tim5->CCR[0] = 0xFFFFFFFF; /* init compare values */ 
  tim5->CCR[1] = 0xFFFFFFFF; /* CCR[1] - CCR[3] are irrelevant */
  tim5->CCR[2] = 0xFFFFFFFF;
  tim5->CCR[3] = 0xFFFFFFFF;

  tim5->CCER |= 0x1; /* activate compare on ccr channel 1 */
  tim5->DIER |= 0x2; /* interrupt enable register */
  
  tim5->CR1 &= ~0x1FF; /* zero out the low 9 bits */
  tim5->CNT = 0;       /* initialize counter value to 0 */
  tim5->EGR |= 0x1; /* Update event (Makes all the configurations stick) */
  tim5->CR1 |= 0x1; /* enable */

}
```

Implementation of the interrupt service routine for the TIM will come later.
There is a bit of plumbing to sort out first (mailbox and memory-pool).  

## Set an absolute time interrupt

To set a new absolute time for an interrupt use `set_ccr_tim5`. This
function just sets a new value in the `tim5->CCR` register.

```
void set_ccr_tim5(int ix, uint32_t c_value) {
  if (ix >= 0 && ix <= 3) {
    tim5->CCR[ix] = c_value;
  }
}
```

To handle the case where no alarm at all is set better, maybe one should
disable interrupts in the `tim5->DIER` in those cases. 

## Configure mailbox and memory-pool

The mailbox in ChibiOS can receive messages that are as large as a
pointer.  So, we can send any 32 bit value without the need of a
memory-pool to hold messages if that is what we want. You can also send
a message to a mailbox which is a pointer to a chunk of memory and
this is the case that will be implemented here.

The messages we send to the mailbox will just be the current value
of the counter as a starting point. I imagine that in the future we
may want to send additional information in this message to tell what
kind of interrupt was fired. Maybe we will want to use all 4 `CCR`
registers for example and then need to be able to encode into the
message which one went off. 

The messages sent will be of the following type: 
```
typedef struct {  /* message type */ 
  uint32_t tick;
}timer_tick_t;
```

Below I use the ChibiOS macros for defining a static mailbox and memory-pool.
There are more dynamic interfaces as well. 

```
#define MAX_MESSAGES 64

msg_t box_contents[MAX_MESSAGES]; /* mailbox storage */
MAILBOX_DECL(mb, box_contents, MAX_MESSAGES);


timer_tick_t tick_storage[MAX_MESSAGES];
MEMORYPOOL_DECL(tick_pool,MAX_MESSAGES, PORT_NATURAL_ALIGN, NULL);
``` 

The mailbox can hold 64 messages that are all of type `msg_t` (which
is large enough to hold a pointer).

The memory-pool, called `tick_pool` is backed up by an array of 64 `timer_tick_t`.
`MEMORYPOOL_DECL` takes the following arguments:
1. Memory-pool name.
2. Number of chunks of memory available to the pool.
3. Alignment of chunks of memory.
4. A memory allocation function in case the pool can grow (as I understand it). `NULL` in this case. 


## Sending and receiving mail

To send a mail to the mailbox, a `timer_tick_t` object is allocated from the
`tick_pool`. If this allocation succeeds, the address of that object is
sent to the mailbox using `chMBPostI`. If sending the mail fails, the `timer_tick_t` object
is returned to the memory-pool using `chPoolFree`. 

```
static bool send_mail(timer_tick_t t) {
 
  /* will be called from inside of the timer interrupt */
  
  bool r = true; /*success*/
  timer_tick_t *m = (timer_tick_t*)chPoolAllocI(&tick_pool);
  
  if (m) { 
    *m = t;
    
    msg_t msg_val = chMBPostI(&mb, (msg_t)m);
    if (msg_val != MSG_OK) {  /* failed to send */
      chPoolFree(&tick_pool, m);
      r = false;
    }
  }
 
  return r;
}
```

The `send_mail` function will later be called from within the
interrupt handler for the timer. So it is important to use variants of
the pool allocation function and mailbox post function that can be
called from an ISR. As I understand it the functions postfixed with an
`I` fall in this category.


The function that receives mail is called `block_mail` because it is a blocking
receive. To make a non-blocking variant the timeout on `chMBFetchTimeout` should
be something else than `TIME_INFINITE`.

```
static bool block_mail(timer_tick_t *t) {

  bool r = true;
  msg_t msg_val;

  int m = chMBFetchTimeout(&mb, &msg_val, TIME_INFINITE);

  if (m == MSG_OK) {
    *t = *(timer_tick_t*)msg_val;

    chPoolFree(&tick_pool, msg_val); /* free the pool allocated pointer */
  } else {
    r = false;
  }
  return r;
}
```

`block_mail` tries to fetch a mail from the box. If this is
successful, the message received is a pointer to a `timer_tick_t`
object. The contents of this `timer_tick_t` object is copied into a
local copy and then the message pointer is returned to the memory-pool.


## The interrupt service routine

Now all the plumbing is in place and we can implement the interrupt
service routine!

```
OSAL_IRQ_HANDLER(STM32_TIM5_HANDLER) {
  OSAL_IRQ_PROLOGUE();
  
  uint32_t sr = tim5->SR;
  sr &= tim5->DIER & STM32_TIM_DIER_IRQ_MASK;
  tim5->SR = ~sr;  

  timer_tick_t t; 
  t.tick = tim5->CNT;

  osalSysLockFromISR();
  send_mail(t);
  osalSysUnlockFromISR();
  
  OSAL_IRQ_EPILOGUE();
}
```

The interrupt service routine does three main things.  To start with
the routine isolates the bit in the `SR` (Status Register)
corresponding to the interrupt we enabled in the `DIER` and clears
this bit. My understanding of this is a bit vague, I assume that the
interrupt raises a bit in the SR to indicate which kind of interrupt
happened and that we can use this to discriminate between different
kinds. Clearing the bit that was set may be done simply so that we
wont confuse ourselves the next time we get an interrupt (that is if
we had more than one CCR activated for example). 

```
  uint32_t sr = tim5->SR;
  sr &= tim5->DIER & STM32_TIM_DIER_IRQ_MASK;
  tim5->SR = ~sr;  
```

Next the current count of the counter is read out and stored in a `timer_tick_t`

```
 timer_tick_t t; 
  t.tick = tim5->CNT;
```

This `timer_tick_t` is sent to the mailbox. 

```
  osalSysLockFromISR();
  send_mail(t);
  osalSysUnlockFromISR();
``` 

`send_mail` has to be wrapped in `osalSys(Un)LockFromIsr` even if the
`I` postfix is present!



## Mail receiving thread

The thread that receives mail just implements a little demo.
An array of 5 alarm-times is used to schedule absolute time interrupts. 

```
uint32_t alarms[5] = { 1357, 2596, 5679, 8000, 10101 };
``` 
The idea is that when the first mail arrives, we process it and then
we schedule the next timer interrupt.

In ChibiOS you create a thread by creating a "thread working area" and a `thread_t` object. 
``` 
static THD_WORKING_AREA(thread_wa, 1024); 
static thread_t *thread;
``` 

Then using the `THD_FUNCTION` macro to declare the function which is the entry
point to the thread. 

```
static THD_FUNCTION(tick_thread, arg) {
 chRegSetThreadName("tick");
 (void) arg;

 int current_alarm = 0;

 chprintf((BaseSequentialStream *)&SDU1, "Entering tick_thread\r\n"); 

 setup_timer();  /* setup timer for continuous runing compare mode */
 set_ccr_tim5(0, alarms[current_alarm]); /* set the first alarm */
 current_alarm ++;
 
 while (1) {
   chprintf((BaseSequentialStream *)&SDU1, "Waiting for mail\r\n"); 
   timer_tick_t t;
   bool r = block_mail(&t);
   if (r) { 
     chprintf((BaseSequentialStream *)&SDU1, "Got mail: %u\r\n", t.tick);
   } else {
     chprintf((BaseSequentialStream *)&SDU1, "Got mail: Error\r\n");
   }

   if (current_alarm < 5) {
     set_ccr_tim5(0, alarms[current_alarm++]);
   }
 }
}

```

This thread starts out by giving itself a name (not relevant or
required). Then it runs `setup_timer` and `set_ccr_tim5` to schedule
the first alarm.  The thread then goes into an infinite loop that
waits for mail by the mailbox.  When mail arrives it prints a message
depending on the success or failure of the mail delivery and then
schedules the next alarm (if there are any more alarms to schedule).


## Main

The `main` function in this case just does some setup. and then goes
into an infinite loop that prints "hello world" twice per second.  It
also prints the current value of the counter each time it prints
"hello world", just for fun.

There are two relevant pieces of initialization in the `main` function:

```
  /* Initialize the memory-pool for tick messages */
  chPoolLoadArray(&tick_pool, tick_storage, MAX_MESSAGES);
  
  /* initialize mailbox */
  chMBObjectInit(&mb, box_contents, MAX_MESSAGES);
```

The `chPoolLoadArray` function, as I understand it, adds all the objects of the
`tick_storage` array to a kind of "free list" managed by the pool.

And then there's an initialization routine for the mailbox, `chMBObjectInit`.

There are comments within the function to deal with the rest. 

```
int main(void) {
  halInit();
  chSysInit();
  
  sduObjectInit(&SDU1);
  sduStart(&SDU1, &serusbcfg);

  /*
   * Activates the USB driver and then the USB bus pull-up on D+.
   * Note, a delay is inserted in order to not have to disconnect the cable
   * after a reset.
   */
  usbDisconnectBus(serusbcfg.usbp);
  chThdSleepMilliseconds(1500);
  usbStart(serusbcfg.usbp, &usbcfg);
  usbConnectBus(serusbcfg.usbp);
  chThdSleepMilliseconds(500);

  /* Initialize the memory-pool for tick messages */
  chPoolLoadArray(&tick_pool, tick_storage, MAX_MESSAGES);
  
  /* initialize mailbox */
  chMBObjectInit(&mb, box_contents, MAX_MESSAGES);


  /* Give the user some time to hook up a usb cable 
     or to connect a terminal */ 
  chThdSleepMilliseconds(2000);

  
  /* start the ssm tick thread */
  thread = chThdCreateStatic(&thread_wa, sizeof(thread_wa), /* working area */
  			     (tprio_t)(NORMALPRIO-20),     /* priority */
  			     tick_thread,                  /* thread function */
  			     NULL);                        /* argument */
  
  while(true) {

    chprintf((BaseSequentialStream *)&SDU1, "Hello world: %u \r\n", tim5->CNT); 
    chThdSleepMilliseconds(500);
  }

  return 0; //unreachable
}
```

## TODOs

Things to figure out as future work:

1. Is it also possible to set an interrupt when the counter wraps around?
   If that is possible, how can we tell the difference between a "normal" (the set alarm time)
   and this overflow interrupt?
2. Different capture/compare modes, can we compare for `>=` instead of just `=`. I think this is
   the case, but more reading of the STM32F4 reference manual is needed.

## The end

Thanks for reading, I hope you found something interesting.
The full source code is [available on github](https://github.com/svenssonjoel/HACKSTUGAN/tree/master/stm32/chibios-abs-timer).
If you have questions or comments I would be very happy to hear
them.

Have a great day!

___

[HOME](https://svenssonjoel.github.io)
