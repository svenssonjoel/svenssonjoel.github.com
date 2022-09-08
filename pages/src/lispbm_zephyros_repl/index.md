

# Example Implementation of a REPL Running on Zephyr OS

An earlier text [here](../lispbm_chibios_repl/index.html) showed an
example of how to implement a lispBM REPL in a ChibiOS thread. In this
text we instead take a look at
[Zephyr](https://www.zephyrproject.org/). I have only used Zephyr with
the [NRF52
Microcontroller](https://www.nordicsemi.com/Products/Low-power-short-range-wireless/nRF52840/GetStarted),
the nrf52840 to be precise. This is quite a powerful microcontroller
featuring an ARM Cortex m4 at 64MHz, 1MB of flash and 256K of RAM. The
nrf52840 also comes with Bluetooth. Some background on what lispBM is
can be found [here](../lispbm_current_status/index.html). 

When I first tried Zephyr OS, I felt that setting it up and getting to
the point where you can build some programs with it was quite
complicated. There is a [video here](https://youtu.be/HulT3RVHoKk)
about the steps involved in getting to a point where you can just edit
your code and run make.

The code in the listings here below, is all available within the
[lispBM repository](https://www.github.com/svenssonjoel/lispBM) on github.


## Introduction to the Code

I will start by going over the code involved and later in the text go
into the configuration files and make scripts.

All IO will be done over USB and when hooking the nrf52 based
development board up to a Linux computer it will appear as a CDC ACM
device and will be accessible through `/dev/ttyACMX` where X is a
number. Which `ttyACM` it gets attached as can be seen by running the
command `dmesg` after attaching the device (note that it has to run
firmware that configures the USB to appear as a device when connected
in this way). As I understand it CDC ACM stands for "Communication
Device Class" and "Abstract Control Model" but the exact implications
of those words are a bit beyond me. So, that is how the device will
appear as viewed from the Linux machine connecting to it, on the
firmware side commucation will look like a UART connection and we will
interact with that UART using interrupts.

```
#include <device.h>
#include <drivers/uart.h>
#include <zephyr.h>
#include <sys/ring_buffer.h>

#include "heap.h"
#include "symrepr.h"
#include "eval_cps.h"
#include "print.h"
#include "tokpar.h"
#include "prelude.h"


#define LISPBM_HEAP_SIZE 2048
#define LISPBM_OUTPUT_BUFFER_SIZE 4096
#define LISPBM_INPUT_BUFFER_SIZE 1024
```

The code starts out including the parts needed form Zephyr
OS. `device` and `uart` are involved in the low level aspects of
communication, the `ring_buffer` is used to store data received and
buffered to be sent over the uart. 

The the header files from lispBM are also included and some sizes that
are going to be used further down are defined. The heap size is
measured in number of cons-cells (so size in bytes is `8 *
LISPBM_HEAP_SIZE`) while the input and output (used for text
interaction with the user) buffer sizes are in bytes.

## Input and Output over USB on the NRF52

The interaction with the uart will be interrupt based. The interrupt
will occur when data is arriving on the uart or when we trigger an
interrupt because we have added data to be sent. A couple of slightly
higher level functions, `get_char`, `put_char` and `usb_printf` are
implemented as well. The interface between the low-level interrupt
routine and the higher level functions are a pair of ringbuffers that
are defined below. For example, when the interrupt routine goes off
because there is data coming in, the data is added to the
`in_ringbuf`.

```
#define RING_BUF_SIZE 1024
u8_t in_ring_buffer[RING_BUF_SIZE];
u8_t out_ring_buffer[RING_BUF_SIZE];

struct device *dev;

struct ring_buf in_ringbuf;
struct ring_buf out_ringbuf;
```

The `in_ring_buffer` and `out_ring_buffer` is the data storage arrays
for input and output data and the rest of the state that is needed for
ringbuffer functionality is in the struct `ring_buf`. This ringbuffer
implementation is a part of the Zephyr OS. 

Above, a device is also declared. These `device` structs are how Zephyr
handles perpiherals (even down to a single GPIO pin seems to need one
of these device structs declared). This device will represent the USB
peripheral.


The `interrupt_handler` defined below runs for as long as there are
uart interrupts pending and for each pending interrupt there is a
check to see if there is data to read or to send.

If data is comming in it is added to the `in_ringbuf`. If more data is
arriving than there is space in the buffer, those bytes are just
ignored.

The case where there is data to send, data is read from the out_ringbuf and
sent over the uart. 

```
static void interrupt_handler(struct device *dev)
{
  while (uart_irq_update(dev) && uart_irq_is_pending(dev)) {
    if (uart_irq_rx_ready(dev)) {
      int recv_len, rb_len;
      u8_t buffer[64];
      size_t len = MIN(ring_buf_space_get(&in_ringbuf),
                       sizeof(buffer));

      recv_len = uart_fifo_read(dev, buffer, len);

      rb_len = ring_buf_put(&in_ringbuf, buffer, recv_len);
      if (rb_len < recv_len) {
        //silently dropping bytes
      }
    }

    if (uart_irq_tx_ready(dev)) {
      u8_t buffer[64];
      int rb_len, send_len;

      rb_len = ring_buf_get(&out_ringbuf, buffer, sizeof(buffer));
      if (!rb_len) {
        uart_irq_tx_disable(dev);
        continue;
      }

      send_len = uart_fifo_fill(dev, buffer, rb_len);
    }
  }
}
```

If I remember correcly, the interrupt routine above is just a
tiny tweak of code from the CDC ACM example provided with Zephyr. 


Now, `get_char`. This function returns an integer which is `-1` on
failure to get a character, otherwise the result is the character
read. In this function interrupts are postponed while interacting with
the ringbuffer, I'm not sure this is actually required (since we are
operating at a single byte at a time). The function tries to read one
byte from the input ringbuffer and if that is successful, the byte is
returned as result.

```
int get_char() {

  int n;
  u8_t c;
  unsigned int key = irq_lock();
  n = ring_buf_get(&in_ringbuf, &c, 1);
  irq_unlock(key);
  if (n == 1) {
    return c;
  }
  return -1;
}
```

The `put_char` function takes an integer as input (not a `char`) this
is so that it can be directly hooked up to `get_char` if one would
like (`put_char(get_char())`). So `putchar` checks if the int
represents a character and in that case adds it to the output
ringbuffer. The same postponing of interrupts occur here. 


```
void put_char(int i) {
  if (i >= 0 && i < 256) {

    u8_t c = (u8_t)i;
    unsigned int key = irq_lock();
    ring_buf_put(&out_ringbuf, &c, 1);
    uart_irq_tx_enable(dev);
    irq_unlock(key);
  }
}
```

A `printf`-like function is always useful! Here it is called
`usb_printf`. One idea would be to implement this using `put_char`,
but it feels more efficient to implement it by writing a whole bunch
of bytes into the output ringbuffer at a time instead. The
`print_buffer` used internally in this function can hold up to at 4096
bytes, which is more than the rinbuffers can hold at once. Because of
this there is a loop that in each iteration adds chunks from the
`print_buffer` to the ringbuffer (as many bytes as it can), and then
triggers the interrupt that should free up new space in the
ringbuffer.


```
void usb_printf(char *format, ...) {

  va_list arg;
  va_start(arg, format);
  int len;
  static char print_buffer[4096];

  len = vsnprintf(print_buffer, 4096,format, arg);
  va_end(arg);

  int num_written = 0;
  while (len - num_written > 0) {
    unsigned int key = irq_lock();
    num_written +=
      ring_buf_put(&out_ringbuf,
                   (print_buffer + num_written),
                   (len - num_written));
    irq_unlock(key);
    uart_irq_tx_enable(dev);
  }
}
```

To read lines of text from the user an `inputline` function is used.
This function should be more or less identical to the same function shown
in [the ChibiOS REPL](../lispbm_chibios_repl/index.html). So I wont go into any
detail of this here.

```
int inputline(char *buffer, int size) {
  int n = 0;
  int c;
  for (n = 0; n < size - 1; n++) {

    c = get_char();
    switch (c) {
    case 127: /* fall through to below */
    case '\b': /* backspace character received */
      if (n > 0)
        n--;
      buffer[n] = 0;
      put_char('\b'); /* output backspace character */
      n--; /* set up next iteration to deal with preceding char location */
      break;
    case '\n': /* fall through to \r */
    case '\r':
      buffer[n] = 0;
      return n;
    default:
      if (c != -1 && c < 256) {
        put_char(c);
        buffer[n] = c;
      } else {
        n --;
      }

      break;
    }
  }
  buffer[size - 1] = 0;
  return 0; // Filled up buffer without reading a linebreak
}
``` 

## Setting up Communication and Implementing the REPL

The `main` function starts out exactly like the CDC ACM example that
comes with Zephyr. This example code can be found within your Zephyr
directory `zephyr/samples/subsys/usb/cdc_acm/src/main.c`

Here the device is created, ringbuffers initialized, the uart configured and
the `interrupt_handler` function is set to be called upon an uart interrupt.


``` 
void main(void)
{

  u32_t baudrate, dtr = 0U;

  dev = device_get_binding("CDC_ACM_0");
  if (!dev) {
    return;
  }

  ring_buf_init(&in_ringbuf, sizeof(in_ring_buffer), in_ring_buffer);
  ring_buf_init(&out_ringbuf, sizeof(out_ring_buffer), out_ring_buffer);

  while (true) {
    uart_line_ctrl_get(dev, LINE_CTRL_DTR, &dtr);
    if (dtr) {
      break;
    } else {
      k_sleep(100);
    }
  }

  uart_line_ctrl_set(dev, LINE_CTRL_DCD, 1);
  uart_line_ctrl_set(dev, LINE_CTRL_DSR, 1); 

  k_busy_wait(1000000);

  uart_line_ctrl_get(dev, LINE_CTRL_BAUD_RATE, &baudrate);
  
  uart_irq_callback_set(dev, interrupt_handler);
  
  uart_irq_rx_enable(dev);
```

In this example the REPL will run in the main thread, unlike the
ChibiOS example where a second thread was created.

After allocating space for input and output of text exchanged with the
user, the lispBM subsystems are started up. 

```
  usb_printf("Allocating input/output buffers\n\r");
  char *str = malloc(LISPBM_INPUT_BUFFER_SIZE);
  char *outbuf = malloc(LISPBM_OUTPUT_BUFFER_SIZE);
  int res = 0;

  heap_state_t heap_state;

  res = symrepr_init();
  if (res)
    usb_printf("Symrepr initialized.\n\r");
  else {
    usb_printf("Error initializing symrepr!\n\r");
    return;
  }

  res = heap_init(LISPBM_HEAP_SIZE);
  if (res)
    usb_printf("Heap initialized. Free cons cells: %u\n\r", heap_num_free());
  else {
    usb_printf("Error initializing heap!\n\r");
    return;
  }

  res = eval_cps_init(false);
  if (res)
    usb_printf("Evaluator initialized.\n\r");
  else {
    usb_printf("Error initializing evaluator.\n\r");
  }
	
  VALUE prelude = prelude_load();
  eval_cps_program(prelude);
```

At the end here a small library is loaded, called the prelude, and
evaluated. The prelude consists of a series of definitions of
functions and evaluating this prelude primes the environment with this
set of functions.

After this setup, the REPL loop begins. It prints a prompt. Clears
input and output buffers and then reads a line from the user.  If the
line read is a command for the REPL, `:info` or `quit` those are
processed otherwise the input is parsed by the `tokpar_parse` function
and the result of that parsing (a heap structure) is evaluated.



``` 
  usb_printf("Lisp REPL started (ZephyrOS)!\n\r");
	
  while (1) {
    k_sleep(100);
    usb_printf("# ");
    memset(str,0,LISPBM_INPUT_BUFFER_SIZE);
    memset(outbuf,0, LISPBM_OUTPUT_BUFFER_SIZE);
    inputline(str, LISPBM_INPUT_BUFFER_SIZE);
    usb_printf("\n\r");

    if (strncmp(str, ":info", 5) == 0) {
      usb_printf("##(REPL - ZephyrOS)#########################################\n\r");
      usb_printf("Used cons cells: %lu \n\r", LISPBM_HEAP_SIZE - heap_num_free());
      usb_printf("ENV: "); simple_snprint(outbuf, LISPBM_OUTPUT_BUFFER_SIZE, eval_cps_get_env()); usb_printf("%s \n\r", outbuf);
      heap_get_state(&heap_state);
      usb_printf("GC counter: %lu\n\r", heap_state.gc_num);
      usb_printf("Recovered: %lu\n\r", heap_state.gc_recovered);
      usb_printf("Marked: %lu\n\r", heap_state.gc_marked);
      usb_printf("Free cons cells: %lu\n\r", heap_num_free());
      usb_printf("############################################################\n\r");
      memset(outbuf,0, LISPBM_OUTPUT_BUFFER_SIZE);
    } else if (strncmp(str, ":quit", 5) == 0) {
      break;
    } else {

      VALUE t;
      t = tokpar_parse(str);

      t = eval_cps_program(t);

      if (dec_sym(t) == symrepr_eerror()) {
        usb_printf("Error\n");
      } else {
        usb_printf("> "); simple_snprint(outbuf, LISPBM_OUTPUT_BUFFER_SIZE, t); usb_printf("%s \n\r", outbuf);
      }
    }
  }

  symrepr_del();
  heap_del();
}
```

If the evaluation is successful, the result is printed and the loop is
executed again.

## Building Zephyr OS

The code and *other files* are stored in a directory structure that
looks as follows. 

```
repl-zephyr
├── CMakeLists.txt
├── Kconfig
├── prj.conf
└── src
    └── main.c
```

We will now look at those files that have been altered for this
example. The Kconfig file is unchanged from the CDC ACM example from
Zephyr.  

Starting with the `CMakeLists.txt` file that has been altered to also
pull in and build the source files from lispBM. The `repl-zephyr`
directory is located inside of the lispBM source tree directly under
its root so when the `CMakeLists.txt` file refers to lispBM those paths
start out with `../`. The `CMakeLists.txt` is used by cmake to create a
traditional `Makefile` that can be processed with the command `make`.
The contents of the `CMakeLists.txt` file is shown below.

```
cmake_minimum_required(VERSION 3.13.1)
include($ENV{ZEPHYR_BASE}/cmake/app/boilerplate.cmake NO_POLICY_SCOPE)
project(repl)

add_definitions(-D_32_BIT_ -D_PRELUDE -DTINY_SYMTAB)

add_custom_command(OUTPUT ../src/prelude.xxd 
                   COMMAND xxd
                   ARGS -i < ../src/prelude.lisp > ../src/prelude.xxd
                   DEPENDS ../src/prelude.lisp
                  )

FILE(GLOB app_sources src/*.c)
FILE(GLOB lisp_sources ../src/*.c)
target_sources(app PRIVATE ${app_sources}
                   PRIVATE ${lisp_sources}
                   PRIVATE ../src/prelude.xxd)
target_include_directories(app PRIVATE ../include
                               PRIVATE ../src)
``` 

The `add_definitions` command means the generated `Makefile` should
when building c source use the flags `-D_32_BIT_`, `-D_PRELUDE` and
`-DTINY_SYMTAB`.  This needs some polishing I see as the naming
convention here is not that uniform. Also, the `_32_BIT_` flag is
pretty pointless as I have now decided that I will not put any effort
into making it compatible with architectures that are not
32bit. However, the remaining defines mean that the prelude library
will be built into the binary and that the tiny version of the symbol
table will be used. See this [blog
post](../lispbm_current_status/index.html) for more information on the
symbol representation table. 

To build the prelude into the binary, the lispBM source code of the
library is "compiled" into a C array of bytes in the file
`prelude.xxd` that will be baked into the executable and later parsed
when starting up the REPL. This slightly special compilation step is
added in the `CMakeLists.txt` file as a custom command that will be
executed as part of the make procedure.

The `CMakeLists.txt` file is also told where the additional source and
header files can be found from the lispBM source tree. This is done
with the `target_include_directories` command for headers and the
`target_sources` for source code.


Next up is the `prj.conf` file that allows us to configure various
aspects of the Zephyr system. This is also derived from the CDC ACM
example but with some additions.

For one, the `CONFIG_NEWLIB_LIBC` flag is set to `y` to build with
newlib functionality. Newlib is C library (like the standard library)
intended for embedded platforms and provides many of the functions one
may be familiar with. I use it mainly to have access to a malloc function. 

```
CONFIG_GPIO=y
CONFIG_USB=y
CONFIG_USB_DEVICE_STACK=y
CONFIG_USB_DEVICE_PRODUCT="Zephyr CDC ACM sample"
CONFIG_USB_CDC_ACM=y
CONFIG_SERIAL=y
CONFIG_UART_INTERRUPT_DRIVEN=y
CONFIG_UART_LINE_CTRL=y
CONFIG_NEWLIB_LIBC=y
CONFIG_MAIN_STACK_SIZE=8192


CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE=2048
CONFIG_FLASH=y
CONFIG_FLASH_PAGE_LAYOUT=y
CONFIG_FLASH_MAP=y
CONFIG_FCB=y
CONFIG_SETTINGS=y
CONFIG_SETTINGS_FCB=y


# Settings needed for custom PCB based on RIGADO BMD-340
CONFIG_CLOCK_CONTROL_NRF_K32SRC_XTAL=n
CONFIG_CLOCK_CONTROL_NRF_K32SRC_RC=y
CONFIG_GPIO_AS_PINRESET=n
```

Towards the last lines of the `prj.conf` file contains some clock
configuration information. If using a development board like the
nrf52480-PCA10056 these can be commented out. If you use a custom
board, perhaps based on the BMD340 module, these are needed. The
difference is that on the latter boards there wont be any external
clock source and an internal clock source should be used. The
`GPIO_AS_PINRESET` option is related to how I flash these custom
boards using SWD and openocd and don't have access to any on board
j-link programmer as on the vendor provided development boards.


To help with building of Zephyr I use some scripts. The first one,
below, I call `set_fw_build.sh`. This is from the lispBM directory.
Running the script creates a build directory and runs `cmake` on the
source tree (that contains the `CMakeLists.txt` file.  This run of
`cmake` populates the build directory with (among other things) a
Makefile.


```
#!/bin/bash

if [ -d "repl-zephyr_build" ]; then
    echo "Build directory exists!"
else
    mkdir repl-zephyr_build
    cd repl-zephyr_build
    cmake -G "Eclipse CDT4 - Unix Makefiles" -DBOARD=nrf52840_pca10056 ../repl-zephyr
fi    
```

Before jumping into the build directory and running `make`, some
additional setup may be needed (depending on how you have configured
your system for Zephyr). Zephyr needs to now a bit more about our
intentions.  For example, we are going to use the arm-none-eabi gcc
compiler suite for cross-compilation to out target platform.

There is another file, called `zephyr-source-me.sh`, that is meant to
be sourced (`source zephyr-source-me`) to include it's settings into
the currently running bash shells environment. So, in this file, the
`ZEPHYR_TOOLCHAIN_VARIANT` is set to `cross-compile` and the
`CROSS_COMPILE` variable is set to the prefix that should be used for
all calls to different tools (such as `gcc`).

It also sources another file of settings from the Zephyr directory
tree called `zephyr-env.sh`

```
#!/bin/bash

# Tweak this line to reflect your setup
export ZEPHYR_TOOLCHAIN_VARIANT=cross-compile
export CROSS_COMPILE=$HOME/opt/gcc-arm-none-eabi-9-2019-q4-major/bin/arm-none-eabi-

source ../zephyrproject/zephyr/zephyr-env.sh 
```

After these last bits of setup, it is time to go into the build
directory and run make, hopefully successfully.


If the build is successful, the resulting binary can be flashed onto a
custom board using an ST-Link and OpenOCD by issuing the following command. 

```
openocd -f interface/stlink.cfg -f target/nrf52.cfg  -c "init" -c "program repl-zephyr_build/zephyr/zephyr.hex verify reset exit"
```

If instead you are using the nrf52480-PCA10056 I think it should be as easy as
running `make flash` after successfully running `make`.

___

[HOME](https://svenssonjoel.github.io)
