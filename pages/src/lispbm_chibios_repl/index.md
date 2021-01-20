

# Example Implementation of a REPL Running in a ChibiOs Thread

This text will show an example of how to implement a lispBM REPL
based on ChibiOS for a microcontroller such as the STM32F4. I have
tried this code on a stm32f407G-discovery board. The version of
ChibiOS used for that test is `19.1.2`.


## Input and Output on a ChibiOs BaseSequentialStream

ChibiOs has an abstraction for byte streams called
`BaseSequentialStream` that you can operate on using the functions
`streamRead`, `streamWrite`, `streamGet` and `streamPut`. 

The `streamGet` functions read one byte of data from the stream,
blocking until data is available. `streamPut` instead writes one byte
of data onto the stream. These two functions can be used to implement
an `inputline` function where for each byte we read into a buffer we
check if it is the newline character and in that case return a number
indicating how many bytes have been read.


Arguments to the `inputline` function are an abstract stream, a target
character buffer and the size of that buffer.


```
int inputline(BaseSequentialStream *chp, char *buffer, int size) {
  int n = 0;
  unsigned char c;
  for (n = 0; n < size - 1; n++) {

    c = streamGet(chp);
    switch (c) {
    case 127: /* fall through to below */
    case '\b': /* backspace character received */
      if (n > 0)
        n--;
      buffer[n] = 0;
      streamPut(chp,0x8); /* output backspace character */
      n--; /* set up next iteration to deal with preceding char location */
      break;
    case '\n': /* fall through to \r */
    case '\r':
      buffer[n] = 0;
      return n;
    default:
      if (isprint(c)) { /* ignore non-printable characters */
        streamPut(chp,c);
        buffer[n] = c;
      } else {
        n -= 1;
      }
      break;
    }
  }
  buffer[size - 1] = 0;
  return 0; // Filled up buffer without reading a linebreak
}
```

The function tries to fill up the buffer, by looping from `0` to
`size-1` and in each iteration getting a character from the
stream.

There are a few special cases here. For example, the character read
could be a backspace character. If that is the case a character should
be removed from the buffer and the loop counter be decremented, unless
of course it is zero.

Then in the case of a newline character, the number of read characters
are returned.

In most cases the character read from the stream is also *echoed* back
onto the stream, so they will appear on the terminal on the other end
of the stream as well (where the user sits).

## The REPL as a ChibiOs Thread

With the help of the `inputline` function defined above we can now
start to write a REPL. First, though, I would like to show how to
implement a lispBM extension for printing strings. 

### Example of a LispBM Extension

The previous text, on [Fundamentals and
Extensions](../lispBM/fundamentals_and_extensions/index.html),
outlined how fundamentals and extensions work *internally* but didn't
provide any example extension. So this example may make the extensions
concept a bit more concrete.


The extension `ext_print` takes a pointer to the first argument and the number of
arguments as input. This is the interface used for all extensions. All arguments are
lispBM `VALUE`s so the extension function itself has to decode these into things that make
sense in a C program.

```
VALUE ext_print(VALUE *args, int argn) {
 
  for (int i = 0; i < argn; i ++) {
    VALUE t = args[i];

    if (is_ptr(t) && ptr_type(t) == PTR_TYPE_ARRAY) {
      array_t *array = (array_t *)car(t);
      switch (array->elt_type){
      case VAL_TYPE_CHAR:
	chprintf(chp,"%s", array->data.c);
	break;
      default:
	return enc_sym(symrepr_nil());
	break;
      }
    } else if (val_type(t) == VAL_TYPE_CHAR) {
      chprintf(chp,"%c", dec_char(t));
    } else {
      return enc_sym(symrepr_nil());
    }
 
  }
  return enc_sym(symrepr_true());
}
```

The extension can print strings and characters. It can be seen in the
conditionals that either an array, `PTR_TYPE_ARRAY`, is expected or a
character, `VAL_TYPE_CHAR`.  Any other type is not accepted and the
extension just returns nil. Another choice would have been to return
the type error symbol in this case, or to ignore that argument and
proceed with printing all printable arguments. 

The print extension is also an example of a function that can take 0
or more arguments.

```
# (print)
> t 
# (print "apa")
apa> t 
# (print "apa" "kurt")
apakurt> t 
# (print "apa" "kurt" "bertil")
apakurtbertil> t 
```

### Initialization and Reset of the Runtime System

The following function resets and reinitializes the lispBM runtime
system. If there is an existing symbol representation table, heap or
extensions these are freed. Then all of the subsystems are restarted.

```
int reset_repl(int heap_size) {
  symrepr_del();
  heap_del();
  extensions_del();

  int res = 0;

  res = symrepr_init();
  if (res)
    chprintf(chp,"Symrepr initialized.\n\r");
  else {
    chprintf(chp,"Error initializing symrepr!\n\r");
    return res;
  }
  
  res = heap_init(heap_size);
  if (res)
    chprintf(chp,"Heap initialized. Free cons cells: %u\n\r", heap_num_free());
  else {
    chprintf(chp,"Error initializing heap!\n\r");
    return res;
  }

  res = eval_cps_init(false);
  if (res)
    chprintf(chp,"Evaluator initialized.\n\r");
  else {
    chprintf(chp,"Error initializing evaluator.\n\r");
    return res;
  }
  
  res = extensions_add("print", ext_print);
  if (res)
    chprintf(chp,"Extension added.\n\r");
  else
    chprintf(chp,"Error adding extension.\n\r");

  VALUE prelude = prelude_load();
  eval_cps_program(prelude);

  chprintf(chp,"Lisp REPL started (ChibiOS)!\n\r");
  
  return res;
}
```

The boolean argument to `eval_cps_init` indicates if the continuation
stack is allowed to grow or not. In this case it is not.

Later a command will be added to the REPL called `:reset` that will run the
function above. 

```
# :reset
Symrepr initialized.
Heap initialized. Free cons cells: 2048
Evaluator initialized.
Extension added.
Lisp REPL started (ChibiOS)!
# 
```

### The REPL Thread

In ChibiOs thread functions are defined using the `THD_FUNCTION` macro
that takes two arguments, the function name and the name of an
argument. For the REPL thread no argument of value is passed over, so
we ignore this.

This thread function will run forever and in parallel with the main
exection thread. Before entering into an eternal loop some storage
space for input and output strings are allocated and the REPL is reset
using the function defined above.

The value of type `heap_state_t` is used as storage when reading out
statistics from the heap subsystem. The REPL will present these
statistics if the command `:info` is given. No valid lispBM construct
begins with (or even contains) `:` ,thats why this character is chosen
for commands directed at the runtime system rather than the evaluator.
Three of the commands are implemented, `:info`, `:quit` and `:reset`.

```
static THD_FUNCTION(repl, arg) {

  (void) arg;
  
  size_t len = 1024;
  char *str = malloc(1024);
  char *outbuf = malloc(2048);

  heap_state_t heap_state;

  int heap_size = 2048;

  reset_repl(heap_size);

  while (1) {
    chprintf(chp,"# ");
    memset(str,0,len);
    memset(outbuf,0, 2048);
    inputline(chp,str, len);
    chprintf(chp,"\n\r");

    if (strncmp(str, ":reset", 6) == 0) {
      reset_repl(heap_size);
      continue;
    } else if (strncmp(str, ":info", 5) == 0) {
      chprintf(chp,"##(ChibiOS)#################################################\n\r");
      chprintf(chp,"Used cons cells: %lu \n\r", heap_size - heap_num_free());
      chprintf(chp,"ENV: "); simple_snprint(outbuf,2048, eval_cps_get_env()); chprintf(chp, "%s \n\r", outbuf);
      heap_get_state(&heap_state);
      chprintf(chp,"GC counter: %lu\n\r", heap_state.gc_num);
      chprintf(chp,"Recovered: %lu\n\r", heap_state.gc_recovered);
      chprintf(chp,"Marked: %lu\n\r", heap_state.gc_marked);
      chprintf(chp,"Free cons cells: %lu\n\r", heap_num_free());
      chprintf(chp,"############################################################\n\r");
      memset(outbuf,0, 2048);
    } else if (strncmp(str, ":quit", 5) == 0) {
      break;
    } else {

      VALUE t;
      t = tokpar_parse(str);

      t = eval_cps_program(t);

      if (dec_sym(t) == symrepr_eerror()) {
	chprintf(chp,"Error\n");
      } else {
	chprintf(chp,"> "); simple_snprint(outbuf, 2048, t); chprintf(chp,"%s \n\r", outbuf);
      }
    }
  }

  symrepr_del();
  heap_del();
}
```

The thread then goes into an infinite loop and most of the code in
there is about handling the `:` commands. But first it prints a
prompts, clear the input buffer and reads a line of text from the
user. If the text that is read in is not one of the `:` commands it
will be fed through the parser and the result of that is fed to the
evaluator. This will either result in an error symbol and an error
message is printed. Otherwise it is output as a result. The REPL then
does it all again.


## The Main Function

The `main` function doesn't do very much. It does what ChibiOs main
function usually do, `halInit` and `chSysInit`. Then it sets up the
byte stream over USB (this is code directly from one of the examples
that come with ChibiOs).

After setting up the byte stream and USB, the REPL thread is launched
and the main thread goes into an infinite loop.

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

	chp = (BaseSequentialStream*)&SDU1;
	
	chThdCreateFromHeap(NULL, REPL_WA_SIZE,
			    "repl", NORMALPRIO + 1,
			    repl, (void *)NULL);
	

	
	while(1) { 
	  chThdSleepMilliseconds(500);
	}

}
```

The `REPL_WA_SIZE`, is a value that sets up a thread working area
size. This is set to 40kb in this case. With a heap of 2048 cells, the
heap alone should be using 16kb. So picking 40kb for working area
should leave some room for the symbol tables and such. I am actually
entirely sure that when malloc is called from a thread it is allocated
within the working area, but experiments seem to indiciate this as
setting the working area too small will cause a failure. Must learn
more about how ChibiOs threads and working areas work.

If you want to get a hold of all of the code involved, that is a
source tree where all you need to do is run make, this is present on
[Github](www.github.com/svenssonjoel/lispBM) under the directory
`repl-ChibiOs`.

___

[HOME](https://svenssonjoel.github.io)
