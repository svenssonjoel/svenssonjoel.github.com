

# Example Implementation of a REPL running on the ESP32 microcontroller


I just got an ESP32 based development board, the [ESP32 DevKitC
v4](https://docs.espressif.com/projects/esp-idf/en/latest/hw-reference/get-started-devkitc.html),
that I ordered some week back. This seems to be quite a powerful
platform, dual-core as I understand it and lots of MHz and memory.

Espressif supplies a development toolchain setup,
[ESP-IDF](https://docs.espressif.com/projects/esp-idf/en/latest/get-started/index.html#step-2-get-esp-idf)
for building software for this MCU and this is what I am using
here. This setup is comes with a ESP32 variant of the GNU toolchain
for cross-compilation as well as FREERTOS with ESP32 additions. I've
had the ESP32 only for a few days now, so this is just a first impression.

The ESP-IDF system hides a lot of what is going on behind python
scripts. I don't know how I feel about this so far. It is in a way
nice that there are scripts that take care of the building and
flashing procedure, but it also obscures what is going on. When trying
to do something that is just a tiny bit outside the "normal flow"
things can get very confusing in a setup like that. It is the same
with the Zephyr OS that hides the details behind the `west`
tool. Currently, I prefer the approach of ChibiOS, where things are
more traditional and Makefile based.

The approach taken here is to tweak the hello-world example that comes
with ESP-IDF so that it creates a lispBM repl task. To do this one has
to figure out how on earth ESP-IDF can be coerced to also build
lispBM. While it is not a perfect fit, lispBM is added as a so-called
*component*. What I am missing for it to be pretty good is a way to
make the build process launch a custom build step that "compiles" the
lispBM prelude using `xxd`.

If you are interested in more background about lispBM look [here](../lispbm_current_status/index.html).

## Input and Output using the UART to USB bridge

The hello-world example uses `printf` to output text. This text can be
seen if connecting to the ESP32 (which turns up as `/dev/ttyUSB0` for
me) with a serial terminal, for example `screen /dev/ttyUSB0`. There
is a USB-UART bridge on the development board, and currently I have
not figured out where there is the configuration to select what
`printf` is hooked up to but by default it seems to be whichever UART
is connected to the USB-UART bridge.

Since we need text input to communicate with the REPL, step one is
figuring out which UART to use and how to configure it.

After a bit of trial-and-error, googling and reading of examples that
come with ESP-IDF the following setup came to be and seems to work for
my dev-board.

```
  const uart_config_t uart_config = {
    .baud_rate = 115200,
    .data_bits = UART_DATA_8_BITS,
    .parity = UART_PARITY_DISABLE,
    .stop_bits = UART_STOP_BITS_1,
    .source_clk = UART_SCLK_REF_TICK,
  };

  uart_driver_install(UART_NUM_0, 1024 * 2, 0, 0, NULL, 0);
  uart_param_config(UART_NUM_0, &uart_config);
  uart_set_pin(UART_NUM_0, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);
``` 

From the data-sheet it seems that the ESP32 has three UARTs that can
all be configured to use any set of 4 GPIO pins. It seems that
`UART_NUM_0` is configured by default to use the pins that are
connected to the USB-UART bridge, so the `uart_set_pin` function is applied with
arguments that tells it not to change any pin assignment here. 


After some more digging, I found that the application of
`uart_set_pin` shown below has the same effect as the one with
`UART_PIN_NO_CHANGE` above. I take this to mean that `GPIO_NUM_34` and
`GPIO_NUM_35` are the pins connected to the USB-UART bridge. In the
ESP32 data-sheet, the *pin layout* section, these pins are referred to
as `GPIO5` and `GPIO18` on physical pins 34 and 35 on the package,
which makes it a bit confusing to me as to why the code should refer
to package pin number rather than that other identity as `GPIOX`. If
this makes total sense to you please share your insight with me.

```
uart_set_pin(UART_NUM_0, GPIO_NUM_34, GPIO_NUM_35, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);
``` 

With the UART configured we can implement `get_char`, `put_char` and `inputline`.

The `get_char` functions uses `uart_read_bytes` to try to read a byte from the
UART. If this does not deliver any character within a certain time-frame, the function
returns the integer `-1` otherwise it returns the character value stored in an int.

```
int get_char() {
  unsigned char c;
  if (uart_read_bytes(UART_NUM_0, (unsigned char *)&c, 1, portMAX_DELAY) == 1) {
    return c;
  } 
  return -1;
}
```

The `put_char` function takes an integer and if the value stored in that integer
represents a character code it is sent over the UART using `uart_write_bytes`.

```
void put_char(int i) {
  if (i >= 0 && i < 256) {
    char c = (char)i;
    uart_write_bytes(UART_NUM_0, &c, 1);
  }
}
```

The `get_char` and `put_char` functions operate on the `int` type so
that, even though `get_char` can fail, they they can be hooked up end
to end as `put_char(get_char())` giving a compact implementation of
"echo".


The `inputline` function is very similar to the same function in
previous posts and explained a bit
[here](../lispbm_chibios_repl/index.html). In essence it reads
characters as long as a newline is not encountered and also tries to deal
with backspaces (removing characters from the buffer).

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


## Tweak hello-world to spawn a REPL task

The `main` function of this attempt to implement a lispBM REPL is just
a small tweak of the hello-world example. I have left the code that
outputs some info about the MCU in there simply because I think such
things are really cool. Other than printing that info, the only thing that is new
is that a task for the repl is created.


```
void app_main(void)
{

  printf("Hello world!\n");

  /* Print chip information */
  esp_chip_info_t chip_info;
  esp_chip_info(&chip_info);
  printf("This is %s chip with %d CPU cores, WiFi%s%s, ",
	 CONFIG_IDF_TARGET,
	 chip_info.cores,
	 (chip_info.features & CHIP_FEATURE_BT) ? "/BT" : "",
	 (chip_info.features & CHIP_FEATURE_BLE) ? "/BLE" : "");

  printf("silicon revision %d, ", chip_info.revision);

  printf("%dMB %s flash\n", spi_flash_get_chip_size() / (1024 * 1024),
	 (chip_info.features & CHIP_FEATURE_EMB_FLASH) ? "embedded" : "external");


  xTaskCreate(repl_task, "repl_task", 8192, NULL, 10, NULL); 
}

```

The `xTaskCreate` function creates a thread that runs a function
called `repl_task`. The `repl_task` will be shown below. It is given a
string name that I have noticed is sometimes visible in error reports
from the RTOS. 8192 is the depth of the stack (in words) that will be
allocated for the task. The number 10, is the priority of the task, a
lower value here means lower priority. The first of the `NULL`s could
be a pointer to some argument that should be passed to the task
function and the last of the `NULL`s is a bit of a mystery to me at
this point (from the documentation it sounds as if it passes a handle
to some task description structure to the created task).

I have noticed that the stack size of 8192 words is perhaps a bit
small. Many functions within lispBM are recursive and use a lot of
stack! So it is quite easy to get the REPL to crash as it is now by
writing a program that generates a large heap structure (for example
`(iota 1000)`) when the repl tries to (recursively) print this heap
structure to show it to the user the stack can deplete, resulting in a
crash and a reboot. It would be nice to make lispBM more friendly to
small stacks by rewriting these internal recursive functions in some
way that is more stack efficient. I'm appending this to the todo-list.

## The REPL task

The REPL tasks start out by configuring the UART in the way shown earlier.
It then proceeds to allocate a buffer for input strings and sets up
each of the lispBM subsystems. 

```
static void repl_task(void *arg) {
  const uart_config_t uart_config = {
    .baud_rate = 115200,
    .data_bits = UART_DATA_8_BITS,
    .parity = UART_PARITY_DISABLE,
    .stop_bits = UART_STOP_BITS_1,
    .source_clk = UART_SCLK_REF_TICK,
  };

  uart_driver_install(UART_NUM_0, 1024 * 2, 0, 0, NULL, 0);
  uart_param_config(UART_NUM_0, &uart_config);
  uart_set_pin(UART_NUM_0, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);

  char *str = malloc(1024);
  size_t len = 1024;
  int res = 0;

  heap_state_t heap_state;

  res = symrepr_init();
  if (res)
    printf("Symrepr initialized.\n");
  else {
    printf("Error initializing symrepr!\n");
  }
  
  unsigned int heap_size = 2048;
  res = heap_init(heap_size);
  if (res)
    printf("Heap initialized. Heap size: %f MiB. Free cons cells: %d\n", heap_size_bytes() / 1024.0 / 1024.0, heap_num_free());
  else {
    printf("Error initializing heap!\n");
  }

  res = eval_cps_init(false); // dont grow stack 
  if (res)
    printf("Evaluator initialized.\n");
  else {
    printf("Error initializing evaluator.\n");
  }

  VALUE prelude = prelude_load();
  eval_cps_program(prelude);

  printf("Lisp REPL started!\n");
  printf("Type :quit to exit.\n");
  printf("     :info for statistics.\n");

  
  while (1) {
    printf("# ");
    fflush(stdout);

    ssize_t n = inputline(str,len);
    printf("\n");

    if (n >= 5 && strncmp(str, ":info", 5) == 0) {
      printf("############################################################\n");
      printf("Used cons cells: %d\n", heap_size - heap_num_free());
      printf("ENV: "); simple_print(eval_cps_get_env()); printf("\n");
      //symrepr_print();
      //heap_perform_gc(eval_cps_get_env());
      heap_get_state(&heap_state);
      printf("Allocated arrays: %u\n", heap_state.num_alloc_arrays);
      printf("GC counter: %d\n", heap_state.gc_num);
      printf("Recovered: %d\n", heap_state.gc_recovered);
      printf("Recovered arrays: %u\n", heap_state.gc_recovered_arrays);
      printf("Marked: %d\n", heap_state.gc_marked);
      printf("Free cons cells: %d\n", heap_num_free());
      printf("############################################################\n");
    } else  if (n >= 5 && strncmp(str, ":quit", 5) == 0) {
      break;
    } else {

      VALUE t;
      t = tokpar_parse(str);

      t = eval_cps_program(t);

      if (dec_sym(t) == symrepr_eerror()) {
        printf("Eval error\n");
      } else {
        printf("> "); simple_print(t); printf("\n");
      }
    }
  }

  symrepr_del();
  heap_del();
}
```

The while loop, that potentially runs forever, starts out by printing
a prompt `#` and waits for a line of input from the user. If the input
line is not a command to the REPL, such as `:info` or `:quit`, the
expectation is that it is a lisp expression. This string is then
parsed and the result of parsing is evaluated.

If the `:quit` command is issued, heap and symbol representations are freed
and the task quits. In this setup this results in the RTOS rebooting and we end up
right back in the REPL, but fresh. 


## Included headers

Just to make the source listed here complete, below are the headers included.

```
#include <stdio.h>
#include <stdlib.h>
#include "sdkconfig.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_system.h"
#include "esp_spi_flash.h"
#include "driver/uart.h"
#include "driver/gpio.h"

#define _32_BIT_

#include "heap.h"
#include "symrepr.h"
#include "extensions.h"
#include "eval_cps.h"
#include "print.h"
#include "tokpar.h"
#include "prelude.h"
```

## Project directory structure

The directory structure of this REPL example is shown below. The starting
point here was the hello-world example from ESP-IDF and I don't yet know
what all files are about. 

LispBM is added as a "component" called `lispbm` that has its own
`CMakeLists.txt` file (shown further below) that describes what files
to build into a `liblispbm.a` file that is linked with the main app
code located in the `main` directory. The `components` directory is an
addition compared to what is present in the hello-world example.

```
repl-esp32/
├── build
├── CMakeLists.txt
├── components
│   └── lispbm
│       ├── CMakeLists.txt
│       ├── component.mk
│       └── lispbm
├── loadable_elf_example_test.py
├── main
│   ├── CMakeLists.txt
│   ├── component.mk
│   └── hello_world_main.c
├── Makefile
├── README.md
├── sdkconfig
├── sdkconfig.ci
└── sdkconfig.old
```

inside of the directory for the `components/lispbm` there is a clone of the
lispBM repository. The `CMakeLists.txt` file points to the files within that clone.


The entire `CMakeLists.txt` is shown below. It lists the source files
and also points out where header files are located. A few compiler options
are also specified that are needed when building lispBM. These specify that
the small library of functions should be built into the executable, `-D_PRELUDE` and that the
tiny version of the symbol table should be used, `-DTINY_SYMTAB`. 

```
idf_component_register(SRCS "./lispbm/src/compression.c"
  "./lispbm/src/env.c"
  "./lispbm/src/eval_cps.c"
  "./lispbm/src/extensions.c"
  "./lispbm/src/fundamental.c"
  "./lispbm/src/heap.c"
  "./lispbm/src/prelude.c"
  "./lispbm/src/print.c"
  "./lispbm/src/stack.c"
  "./lispbm/src/symrepr.c"
  "./lispbm/src/tokpar.c"
                       INCLUDE_DIRS ./lispbm/include)

target_compile_options(${COMPONENT_LIB} PRIVATE -D_32_BIT_ -DTINY_SYMTAB -D_PRELUDE)

``` 

One thing I would like to add to this `CMakeLists.txt` file is a
custom command that applies `xxd` to a file called `prelude.lisp` to
generate a C array representing the contents of the file. Currently I
create this file manually by going into the lispBM source tree and
running make once manually which creates this file. 


## Building the REPL

In the root directory of the source tree, the following command builds everything.

```
idf.py build
```

issuing this command for the first time seems to take some time as all sorts of files are
pulled in. Highly mysterious, but also convenient. 

Then the dev-board can flashed with the binary (if build is successful). 

```
idf.py -p /dev/ttyUSB0 flash
```


And finally we can interact with the REPL over a serial link. 
```
screen /dev/ttyUSB0 115200
```

___

[HOME](https://svenssonjoel.github.io)
