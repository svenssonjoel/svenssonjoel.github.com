

# Control an 8x8 LED Matrix from an STM32 and ChibiOs

I recently got a kit of various electronics and one of the things 
in there was an 8x8 LED matrix and decided to try to get it running. 
As usual with these, it does not seem to come with any kind of indication 
of pin-out (where pin one is and so on). Mine only had "788BS" printed 
on one of the sides. 

788BS data-sheets are available though! Look
[here](http://www.ledtoplite.com/uploadfile/2017/TOP-CA-788BS.pdf) for
example. The mapping of pins to rows and columns on my LED matrix seem 
to match up with the data-sheet. Figuring out which of the rows of pins 
on the package are pins 1 - 8 and which are pins 9 - 16 can take 
some trial and error though. 

To turn on a specific LED you need to put a zero on the corresponding
column pin (ground that pin) and a one on the corresponding row pin
(3V in the case of the Discovery board). 

The picture below shows how I wired mine up. 
![8x8 LED MATRIX](./media/8x8_LED_2.png)

As indicated in the picture, I have hooked up all the row pins to GPIO pin PA0 - 7
and all the column pins to PD0 - 7. The wiring is such that row 1 is PA0, row 2 is PA1 and 
so on, and likewise for columns. 



What we will implement here is scrolling text in the LED matrix. To do so 
we will need characters (a font). I found [this](https://github.com/dhepper/font8x8) 
github repository containing exactly what is needed, a set of characters 
expressed in C code. I use the `font8x8_basic.h` file from that repository 
to generate the text. 

In `font8x8_basic.h` each character is represented by 8 `char` values,
one for each row of the symbol. Each bit in the `char` value then
represent a point or absence of a point for each column. You can head
over to [github](https://github.com/dhepper/font8x8) and check out the
README file. 


To see what the text-scroller looks like, jump over to
[YouTube](https://youtu.be/hPiWrrbekuc) and take a look. 



## The Code 

Let's start by writing a bit of code for drawing a character in an 8x8 buffer at an 
arbitrary position. 


``` 
void render_character(char *buff, int c, int x, int y) {

  if (x <= -8 || y <= -8 || y >= 8 || x >= 8) return;

  int cx = 0;
  int cy = 0;
  
  for (int i = x; i < 8; i ++) {
    for (int j = y; j < 8; j ++) {

      if (font8x8_basic[c][cy] & (1<<cx)) {
        buff[j] |= 1 << i;
      }
      cy++;
    }
    cy = 0;
    cx++;
  }
} 
```

This is done by looping over the area of the target buffer (which will
map directly to the 8x8 LED matrix) that will be covered by the 8x8
character we are printing. So, if the position we want to draw the 
character at is (0,0), then `i` and `j` will range over 0 to, and including, 7. 

The font data (in `font8x8_basic.h`) is stored in an array such that the ASCII code 
for a character corresponds to the location in the array. So we can directly index 
into the `font8x8_basic` array at position `c` and find the bit pattern for the 
corresponding ASCII character. 

While `i` and `j` are the rows and columns of the buffer, `cy` and
`cx` are the row and column of the character to print. If we want to
paint a character at position (4,4) for example, then `i` and `j` will
both start at value 4 the rows and columns of the character should
however start at (0,0) (or we would draw out the bottom right corner of the character 
instead of the top left one).



Ok! That renders one character. Now, let's implement a text-scroller. 

```
void render_string(char *buf, char *str) {
  
  size_t len = strlen(str);
  int len_bits = len * 8;

  int pos = 0;

  while (pos > -len_bits) {

    /* Clear buffer */
    for (unsigned int i = 0; i < 8; i ++) buf[i] = 0;

    /* render all characters */
    int char_pos = pos;
    for (unsigned int a = 0; a < len; a ++) {
      
      render_character(buf, str[a], char_pos, 0);
      char_pos += 8; 
    }
    chThdSleepMilliseconds(50);
    pos--;
  }
}
```
The `render_string` draws the string, `str`, repeatedly into the buffer each time 
at an updated position. Between each time the string is drawn the thread goes to sleep. 
this is to set the speed with which the text scrolls and to allow that other threads 
can execute. One other thread that must be given time to execute is the thread 
that copies the contents of the buffer that we print into onto the LED Matrix. 

First this function computes the length of the string in "pixels", here call `len_bits`. 
A position variable `pos` is set to 0. This is the position in the buffer where 
the first character in the string will be painted. The `pos` variable will then decrease 
for every iteration of the while loop, this essentially moves the starting point 
of the string outside of the buffer to "the left". If you go back and look at `render_character`
you can see that it will not try to paint any characters that are entirely outside 
of the buffer. 

``` 
if (x <= -8 || y <= -8 || y >= 8 || x >= 8) return;
``` 

So, anyway, the string will be drawn as many times as there are pixels in the string. 

``` 
  while (pos > -len_bits) {
```
We start at position 0 and decrease pos until it is `-len_bits` which means that 
the entire string has scrolled by. 

Each iteration in the while loop clears the buffer and then loops through 
the entire string and performs `render_character` for each character. The 
position for each individual character is called `char_pos` and it starts being 
the same as `pos` for the first character of the string. `char_pos` then increases 
by 8 for each iteration of the loop. 

That takes care of the scrolling text. 

The buffer is stored in a global variable of 8 by 8 bits. 

``` 
char buffer[8]; 
``` 


A ChibiOs thread will be used to update the 8x8 LED matrix. This
thread will be called `update`.

```
static THD_WORKING_AREA(update_area, 2048);

static THD_FUNCTION(update, arg) {

  (void) arg;

  while(true) { 
    /* plot buffer */
    for (int i = 0; i < 8; i ++) {
      palWriteGroup(GPIOA, PAL_GROUP_MASK(8),0,(1 << i));
      palWriteGroup(GPIOD, PAL_GROUP_MASK(8),0,~buffer[i]);
      chThdSleepMicroseconds(100);
    }
  }
}

```

This thread just loops forever and keeps copying the values from the 
buffer to the GPIOs. `palWriteGroup` allows you to set or clear a bunch 
of GPIOs in just one call. 

We loop over the buffer and for each row we use `palWriteGroup` to first 
set a 1 on the row output of interest. Then `palWriteGroup` is used again 
to copy a row from the buffer onto the PD0-7 pins. The value from the buffer 
is negated as these pins will be used to sink a current in the case an LED 
should turn on, not source one. 

Then sleep for a short short while to let some current flow and LEDs on that 
row light up. 



Ok, that is just about all. Just the `main` function left to look at. 
In `main` the GPIO pins are configured and the `update` thread is started. 

```
int main(void) {
  halInit();
  chSysInit();
    
  palSetGroupMode(GPIOA, PAL_GROUP_MASK(8), 0, PAL_MODE_OUTPUT_PUSHPULL);
  palSetGroupMode(GPIOD, PAL_GROUP_MASK(8), 0, PAL_MODE_OUTPUT_PUSHPULL);

  palWriteGroup(GPIOA, PAL_GROUP_MASK(8), 0, 0);
  palWriteGroup(GPIOD, PAL_GROUP_MASK(8), 0, 0);
  
  (void)chThdCreateStatic(update_area,
			  sizeof(update_area),
			  NORMALPRIO,
			  update, NULL);
  
  while(true) {
    render_string(buffer, "My scrolling text on an 8x8 LED matrix!!!");
    
  }
  return 0; //unreachable
}

```

The `main` "thread" also runs forever, but in this case all it does is
to repeatedly render a string into the buffer.


I paste the complete code listing here: 

```
#include "ch.h"
#include "hal.h"
#include "string.h"

#include "font8x8_basic.h"

void render_character(char *buff, int c, int x, int y) {

  if (x <= -8 || y <= -8 || y >= 8 || x >= 8) return;

  int cx = 0;
  int cy = 0;
  
  for (int i = x; i < 8; i ++) {
    for (int j = y; j < 8; j ++) {

      if (font8x8_basic[c][cy] & (1<<cx)) {
        buff[j] |= 1 << i;
      }
      cy++;
    }
    cy = 0;
    cx++;
  }
} 

char buffer[8];

void render_string(char *buf, char *str) {
  
  size_t len = strlen(str);
  int len_bits = len * 8;

  int pos = 0;

  while (pos > -len_bits) {

    /* Clear buffer */
    for (unsigned int i = 0; i < 8; i ++) buf[i] = 0;

    /* render all characters */
    int char_pos = pos;
    for (unsigned int a = 0; a < len; a ++) {
      
      render_character(buf, str[a], char_pos, 0);
      char_pos += 8; 
    }
    chThdSleepMilliseconds(50);
    pos--;
  }
}

static THD_WORKING_AREA(update_area, 2048);

static THD_FUNCTION(update, arg) {

  (void) arg;

  while(true) { 
    /* plot buffer */
    for (int i = 0; i < 8; i ++) {
      palWriteGroup(GPIOA, PAL_GROUP_MASK(8),0,(1 << i));
      palWriteGroup(GPIOD, PAL_GROUP_MASK(8),0,~buffer[i]);
      chThdSleepMicroseconds(100);
    }
  }
}




int main(void) {
  halInit();
  chSysInit();
    
  palSetGroupMode(GPIOA, PAL_GROUP_MASK(8), 0, PAL_MODE_OUTPUT_PUSHPULL);
  palSetGroupMode(GPIOD, PAL_GROUP_MASK(8), 0, PAL_MODE_OUTPUT_PUSHPULL);

  palWriteGroup(GPIOA, PAL_GROUP_MASK(8), 0, 0);
  palWriteGroup(GPIOD, PAL_GROUP_MASK(8), 0, 0);
  

  (void)chThdCreateStatic(update_area,
              sizeof(update_area),
              NORMALPRIO,
              update, NULL);
  
  while(true) {
    render_string(buffer, "8x8 LED MATRIX CONTROLLED BY STM32F407G-DISC1 ___ SVENSSONJOEL.GITHUB.COM ___ ");
    
  }

  return 0; //unreachable
}
```



## Conclusion

Thank you for reading. If you have any questions or feedback you can
always send an email or join the google group.

Have a good day!



___

[HOME](https://svenssonjoel.github.io)
