

# Setup the DAC on STM32 with ChibiOS

This is a follow-up on on the previous text about [reading the
ADCs](../adc_stm32/index.html) for the [sound-generating
PCB](http://www.github.com/svenssonjoel/SYNTH).

Some good news since the earlier text, is that the device actually
seems to be able to produce some noise! So far I have only tried the
DACs that are internal to the STM32F4 and not the external 8Bit DAC
that is added mostly as an experiment. If you are interesting in the
background story behind this PCB and all my doubts during the design
process, take a look at this playlist on
[YouTube](https://www.youtube.com/playlist?list=PLtf_3TaqZoDMNIfHCWMzmyu7kqccrfyPw).

So far I have only made very initial experimentation with the DAC on
the STM32F4 and will write more about this in the future as I try to
have them generate more interesting sounds. But for this text, it will all be about 
how to set up ChibiOS for DAC output on the STM32F4. 

## Some about the hardware 

The code I present in this text is not specific to my "custom" audio
experiment PCB but should be applicable to any STM32F4 board (for
example the Discovery or Nucleo). 

There are two DAC channels on the STM32F4 and these are associated
with the `GPIO A` pins 4 and 5. That is `PA4` and `PA5`. This means we
can output two independent analog signals. I am interested in using both
of these DAC channels, so that is what this text covers.

Another important bit of information is that the internal DACs are
12Bit which essentially means that they translate a value between 0
and 4095 to a voltage between 0 and VDDA (3V for example). 


## Configuring ChibiOS for DAC usage 

Just like with the ADC there are some "switches" to activate in `mcuconf.h` to 
configure the DAC system. 

```
/*
 * DAC driver system settings.
 */
#define STM32_DAC_DUAL_MODE                 FALSE
#define STM32_DAC_USE_DAC1_CH1              TRUE
#define STM32_DAC_USE_DAC1_CH2              TRUE
#define STM32_DAC_DAC1_CH1_IRQ_PRIORITY     10
#define STM32_DAC_DAC1_CH2_IRQ_PRIORITY     10
#define STM32_DAC_DAC1_CH1_DMA_PRIORITY     2
#define STM32_DAC_DAC1_CH2_DMA_PRIORITY     2
#define STM32_DAC_DAC1_CH1_DMA_STREAM       STM32_DMA_STREAM_ID(1, 5)
#define STM32_DAC_DAC1_CH2_DMA_STREAM       STM32_DMA_STREAM_ID(1, 6)
```

Since I am interested in using both channels I set `STM32_DAC_USE_DAC1_CH1` and
`STM32_DAC_USE_DAC1_CH2` to `TRUE`. 

In `halconf.h` there is also a switch to turn the DAC system on. 


```
/**
 * @brief   Enables the DAC subsystem.
 */
#if !defined(HAL_USE_DAC) || defined(__DOXYGEN__)
#define HAL_USE_DAC                         TRUE
#endif
``` 

With these settings ChibiOS will provide us with driver "handles" for the two 
DAC channels called `DACD1` and `DACD2`.

ChibiOS will now build with DAC support and we can get working on the application 
end of things. 


## The code 

I put the code concerning the DACs into files named `dac.c` and
`dac.h`.  The header file `dac.h`, just contains a few definitions and
some function prototypes. 

```
#include "hal.h"
#include "hal_pal.h"

#define DAC_GPIO GPIOA
#define DAC1_PIN 4
#define DAC2_PIN 5

#define DAC1 1
#define DAC2 2

extern void dac_init(void);
extern void dac_write(int dac, int16_t val);
```

The defines are just there to hide those otherwise magical constants
behind some new names that kind of makes more sense. I call the two DAC channels `DAC1` and `DAC2` 
and we can also see that the GPIOs that are of interest here are `PA4` and `PA5`. 


The `dac.c` file implements the two functions `dac_init` and `dac_write`. 
As part of initializing the DAC subsystem a `DACConfig` object is needed. 

```
static const DACConfig dac_config = {
  .init         = 2047u, 
  .datamode     = DAC_DHRM_12BIT_RIGHT,
  .cr           = 0
};

```

This sets up an initial value for the data register in the DAC. The
next field `datamode` that is set to `DAC_DHRM_12BIT_RIGHT`, my
feeling is that we are here specifying that the will be 12Bits and the
unused bits in a full 32bit register will be to the left and the first
12 bits from the right are what is being used. The `cr` field probably
allow us to set the DAC Control register to anything we like. There
seems to be a lot of special functions we can have the DAC perform by
setting up the `cr` in different ways. Take a look at the [reference
manual](https://www.st.com/resource/en/reference_manual/dm00031020-stm32f405-415-stm32f407-417-stm32f427-437-and-stm32f429-439-advanced-arm-based-32-bit-mcus-stmicroelectronics.pdf).

Now this `DACConfig` object can be used in the implementation of
`dac_init`. First, this function sets the GPIO pins associated with
the DAC channels to `PAL_MODE_INPUT_ANALOG`. That it is called "INPUT"
is odd but just some odd naming we need to accept. 

```
void dac_init(void) {

  palSetPadMode(DAC_GPIO, DAC1_PIN, PAL_MODE_INPUT_ANALOG);
  palSetPadMode(DAC_GPIO, DAC2_PIN, PAL_MODE_INPUT_ANALOG);

  dacStart(&DACD1, &dac_config);
  dacStart(&DACD2, &dac_config);
}
```
Then the both DAC channels are started and this is where that `DACConfig`is used. 
Here those two "handles" that ChibiOS provides `DACD1` and `DACD2` are referenced.


Finally then! A `dac_write` function that lets us output a value to either 
channel 1 or 2 by giving `DAC1` or `DAC2` as argument. The second argument is 
the value to output on the chosen DAC. 

```
void dac_write(int dac, int16_t val) {
  switch(dac){
  case DAC1:
    dacPutChannelX(&DACD1, 0, val);
    break;
  case DAC2:
    dacPutChannelX(&DACD2, 0, val);
    break;
  default:
    break;
  }  
}
``` 

All unknown values for `dac` are just silently ignored. 


## Concluding remarks 

So! I was actually quite surprised that the custom PCB worked at
all. Happy turn of events. The first test of it just used the above
code to initialize the DACs and then I set up a GPT (general purpose
timer) that periodically fired off a callback function. This callback
function outputs values on the DACs that it gets from a sine wave
table. That part of the code is, however, quite unpolished and I want
to do something more of that before writing about it.

The next text in this series on "audio" stuff, will then most likely
be about outputting waves of some kind on the DACS, using GPTs and
look-up tables perhaps. We'll see.

Oh! and of course, I need to see if the external 8Bit DAC experiment
works as well!

Thanks for reading, and as always all constructive feedback is much
appreciated!

___

[HOME](https://svenssonjoel.github.io)
