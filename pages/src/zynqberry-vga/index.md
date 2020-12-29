

# Adding VGA output to the Zynqberry 

This text is about an experiment that maybe does not make very much
sense, but I did have lots of fun with all of it. VGA output on the Zynqberry!

The Zynqberry is a Xilinx Zynq-7010 based single board computer in
Raspberry Pi form-factor by
[trenz-electronic](https://shop.trenz-electronic.de/en/TE0726-03M-ZynqBerry-Module-with-Xilinx-Zynq-7010-in-Raspberry-Pi-Form-Faktor).
As the Zynqberry Mimics the Raspberry Pi, it already has an HDMI
connector which is what makes it perhaps a bit odd to want to add VGA
to it as well. But it is all for learning and fun, so who cares about
it making sense. Also HDMI seems a bit scary to me at the
moment. Maybe should take a look at HDMI output at some point in the
future.

What I describe in this post will build heavily upon a number of
earlier posts.  These are, [Nexys A7 VGA - part
1](https://svenssonjoel.github.io/pages/nexys-a7-vga/index.html),
[Nexys A7 VGA - part
2](https://svenssonjoel.github.io/pages/nexys-a7-vga-2/index.html) and also the 
[Get started with VHDL on the
Zynqberry](https://svenssonjoel.github.io/pages/vivado-zynq-7010-blinky-vhdl/index.html) post. 

The VHDL Implementation of the graphics memory and the VGA output is identical here to what 
is described in the part 2 post on VGA on the Nexys A7. 

VGA is analog and the Zynqberry GPIOs digital, so some kind of DAC is
needed. The Nexys A7 have a 4 bit DAC per color on the board for VGA
display but the Zynqberry does not have that. To get VGA output we
will need to build a DAC of our own.

One kind of DAC that can be built at home easily with just a bunch of
resistors is an R2R DAC. This is a network of resistors of only two
values `R` and `2*R`.  For more details check out for example this
tutorial: [Electronics Tutorials R2R
DAC](https://www.electronics-tutorials.ws/combination/r-2r-dac.html).

Check out this
[Illustration](http://www.falstad.com/circuit/circuitjs.html?ctz=CQAgjCAMB0l3AWAnC1b0DYQeggrPEgMyQYIAceeA7OSAffZAFBhn0IIhF4BMI1SFx79IILniiMxAMwCGAGwDOAUyjMATiHJwQvXlx1je1UUxZbqfbtcHDrYghYHWi5Lne7upTzS-4IbAJC4kGOcH5WARTBXJx04c5GHIa6yFhiCPDOSGDRdOTs8VJWzADu2Fk2-J4ieuWVYm4eIc31FWSZQZ6BWLwNneIxPTH9FVFDBWkxLBXJxYVxM8wA5tq6eJzrjtQZ6mtIGJnsh5mHUixrpsYGArzGeHuz23q3yWBIog3vn3diH19xvdwL9rq8uM9cvkQKdJuoKrDioj2CwAEopar0PKYiD8MC6XjQSSEyRiGB4Vb0Xh0CiOalYszPTZLOh8GnLCpsuFc2nw-yhLDzMLfaZTM4ZBoTNrJNrPKXWGUOEXGUwvOpJXTSzUOPQEPxCwWa7wPDWZGIGqS8PUAB20RXN7DqEDJlKh3GBbvVfMWmJ91IS33Y-ph2ODkND5DonuBLCAA).
It uses 5V (I could not find a way to set it to 3V3) and as such it
hits somewhere over 1V maximum while VGA wants 0.7V. The 75 Ohm
resistor seen there represents the load of the VGA display and cable
(as I understand it). The picture below shows the same DAC drawn up in
KiCad for simulation. I used the KiCad simulator to find the value of
`R` that makes the output signal at most roughly 0.7V.

![R2R DAC](./media/sim_vga.png)


A complete VGA DAC can now be constructed by replicating the circuit seen above 
once for each color channel (3 in total).

<!--
``` 
$ 1 0.0000049999999999999996 6.450009306485578 50 5 50
164 544 352 704 352 0 4 0 5 0 5 false 0
r 800 224 800 272 0 500
r 752 352 704 352 0 500
r 752 384 704 384 0 500
r 752 416 704 416 0 500
r 752 448 704 448 0 500
r 800 544 800 496 0 40000
r 912 448 864 448 0 75
w 640 352 704 352 2
w 640 384 704 384 2
w 640 416 704 416 2
w 640 448 704 448 2
w 752 448 800 448 0
w 800 448 864 448 0
g 800 544 800 576 0 0
g 960 464 960 496 0 0
g 720 224 720 256 0 0
w 800 224 800 192 0
w 800 192 720 192 0
w 720 192 720 224 0
w 912 448 960 448 0
w 960 448 960 464 0
R 544 352 512 352 1 2 100 2.5 2.5 0 0.5
g 528 480 528 512 0 0
w 544 448 528 448 0
w 528 448 528 480 0
w 752 416 800 416 0
w 800 448 800 496 0
w 752 384 800 384 0
w 752 352 800 352 0
w 800 272 800 352 0
r 800 384 800 352 0 250
r 800 416 800 384 0 250
r 800 448 800 416 0 250
p 864 448 864 352 1 0 0
g 912 320 912 352 0 0
w 864 352 864 288 0
w 864 288 912 288 0
w 912 288 912 320 0
```
-->

I also made an attempt at a VGA DAC on an experiment board (The
soldering kind). The result of this is shown below.

 Experiment board VGA DAC  | Hat-like connection
| :---:|:---:|
![Experiment board VGA DAC](./media/dac1.jpg)| ![Hat-like connection](./media/dac0.jpg)

This attempt actually worked quite well but the circuit it implements 
does not look exactly like the one used in the simulation above. It is missing 
that bigger termination resistor and at some point I kind of flipped it
upside down. Oddly enough it produces an image that looks just fine! Another 
difference is that I don't have resistors in my boxes with resistances 
that make up good `R` and `2*R` pairs, so it was very roughly an R2R DAC. 

The next attempt was more faithful to what was simulated in KiCad. I
still don't have access to exactly `R` and `2*R` resistors, but they
are somewhere close. This experiment was executed on breadboard and
looks like the picture below.

 Bread board VGA DAC  | Connection to Zynqberry
| :---:|:---:|
![Bread board VGA DAC](./media/dac3.jpg)| ![Connection to Zynqberry](./media/dac2.jpg)

Ok, all of this experimentation seems to point out that for a 4 bit
(per color) VGA DAC, you don't need to be very strict. If you are
"close enough" there will some graphics on the screen. Maybe the
colors are slightly too dim (or too bright) or the steps in intensity
not entirely linear. The picture below shows what the VGA output 
looks like through the breadboard DAC. 

![VGA Output from Zynqberry](./media/screen_out.jpg) 


## Vivado, constraints and Vitis

The VHDL code used for this first VGA experiment on the Zynqberry is
identical to that used in [Nexys A7 VGA - part
2](https://svenssonjoel.github.io/pages/nexys-a7-vga-2/index.html).
So wont go into any of the details of that here again. There are some
differences in how to get it all to work on the Zynq though. The main
difference being the Zynq processing system and how to clock the
thing. In the Nexys case getting a clock to the design was done
through the constraints file but here I did it using the block design
GUI.

I am not sure that what I describe here is the one and true way to do it. 
If you have feedback, please just send an email. There may be better ways. 

To set up this on the Zynq one needs to go through roughly these steps: 

1. Create a new project with a Zynqberry target. 
2. Add the VHDL files developed in the [Nexys A7 VGA - part
2](https://svenssonjoel.github.io/pages/nexys-a7-vga-2/index.html).
3. Create a block design.
4. Add Zynq processing system to block design (apply defaults, disable unused AXI Master etc). 
5. You will need to create HDL wrappers (and let Vivado manage those). 
6. Add the VGA module. Right click in the block design window and
   select add module and select the "vga module".
7. Run block automation to hook up clk and reset. 
8. Add an inverter on the reset line (since the vga module uses positive reset). 
9. Double click on the "processing system" and locate the clock configuration settings. Then 
 change the frequency of FCLK_CLK0 to 100MHz. 
10. Create ports called RED, GREEN and BLUE. these should be 4 bit port vector outputs
11. Create single bit output ports HSYNC and VSYNC. 
12. Connect the ports up to the vga block. 

I'm thinking the result of doing all this should look roughly 
like the picture below. 

![Vivado block design Zynqberry VGA](./media/vivado_screen.png)

One thing to perhaps double check is that the HDL wrappers generated 
by Vivado are now the top level design source. If it is not you should 
make it so. 


Now, create a constraints file and add the following constraints to that file. 

```
#RED
set_property -dict {PACKAGE_PIN K15 IOSTANDARD LVCMOS33} [get_ports {RED[3]}]
set_property -dict {PACKAGE_PIN J14 IOSTANDARD LVCMOS33} [get_ports {RED[2]}]
set_property -dict {PACKAGE_PIN H12 IOSTANDARD LVCMOS33} [get_ports {RED[1]}]
set_property -dict {PACKAGE_PIN G11 IOSTANDARD LVCMOS33} [get_ports {RED[0]}]

#GREEN
set_property -dict {PACKAGE_PIN G12 IOSTANDARD LVCMOS33} [get_ports {GREEN[3]}]
set_property -dict {PACKAGE_PIN H13 IOSTANDARD LVCMOS33} [get_ports {GREEN[2]}]
set_property -dict {PACKAGE_PIN H14 IOSTANDARD LVCMOS33} [get_ports {GREEN[1]}]
set_property -dict {PACKAGE_PIN J13 IOSTANDARD LVCMOS33} [get_ports {GREEN[0]}]

#BLUE
set_property -dict {PACKAGE_PIN J15 IOSTANDARD LVCMOS33} [get_ports {BLUE[3]}]
set_property -dict {PACKAGE_PIN N14 IOSTANDARD LVCMOS33} [get_ports {BLUE[2]}]
set_property -dict {PACKAGE_PIN R15 IOSTANDARD LVCMOS33} [get_ports {BLUE[1]}]
set_property -dict {PACKAGE_PIN R13 IOSTANDARD LVCMOS33} [get_ports {BLUE[0]}]

# VSYNC
set_property -dict {PACKAGE_PIN R12 IOSTANDARD LVCMOS33} [get_ports {VSYNC}]
#HSYNC
set_property -dict {PACKAGE_PIN L12 IOSTANDARD LVCMOS33} [get_ports {HSYNC}]
```

At this stage it should be time to synthesize, implement and generate
bitstream. And if all of that completes successfully the next step 
is to export hardware (include the bitstream) and then launch Vitis. 

In Vitis create a hello world project using the exported hardware 
and run it through the debugger. 





## Conclusion 

Most of what is described in this text builds upon the previous posts
on
[Zynqberry](https://svenssonjoel.github.io/pages/vivado-zynq-7010-blinky-vhdl/index.html),
[Nexys A7 VGA - part
1](https://svenssonjoel.github.io/pages/nexys-a7-vga/index.html) and
[part
2](https://svenssonjoel.github.io/pages/nexys-a7-vga-2/index.html). But
still, lots of fun.

The Zynq platform opens up for some fun next steps. One thing that
would be nice is to pull that image memory hardware out of the VGA box
and then replace it with a memory with an AXI interface so that it
could be written to from the ARM cores. Maybe this memory would need
to be dual ported so that the VGA module and the ARM core can
read/write to it at the same time? I don't know. I noticed that there
is a "Block Memory Generator" (BMG) IP in Vivado that can be used to
generate memories.  Would be nice to figure out how to use that.  The
memories generated by the BMG all seems to be 32 bit word read and
write though while the VGA as it is now reads a byte and then performs
3x color nibble lookups. Some major changes would be needed but it
sounds like fun and I want to attempt it if there is ever some time.

Building some DACs was also fun. So to make it easier to experiment
more with VGA on ZynqBerry I decided to make a PCB. The breadboard
setup wont last very long, not with all the cats that like to walk
around on my desk.

Below are some pics of the Zynqberry VGA PCB and the schematics. 

 PCB VGA DAC front | PCB VGA DAC back
| :---:|:---:|
![PCB VGA DAC front](./media/hat_front.png)| ![PCB VGA DAC back](./media/hat_back.png)


![Schematics for Zynqberry VGA](./media/schematics.png)

Thanks a lot for reading. I hope you are having a good day. Please get
in touch if you have questions, hints, tips or any other kind of
constructive feedback. 

---
[HOME](https://svenssonjoel.github.io)
