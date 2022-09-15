# VGA mixer PCB design
This repository contains the research and design work that went into the PCB
for our VGA mixer project in TDT4295 - computer project at NTNU, autumn 2022.

## Initial feasibility study
The [pdf](InitialFeasibilityStudy.pdf) contains the first research, before
the capabilities and requirements were fully fleshed out. In this repo,
we are working on the actual design.

## Project features
The VGA mixer is a device that takes two VGA inputs,
called "background" and "foreground", and outputs a single VGA video stream.

It should support the SVGA resolution 800x600 @ 60Hz and 16-bit color depth.
The colors are stored as (5,6,5)-bit RGB values internally.

On the board there are several buttons and an LCD screen,
allowing you to pick between different **modes**:
 - Showing only the foreground
 - Showing only the background
 - Overlaying with chroma keying appied to the foreground
 - Showing the foreground at 25%, 50% or 75% transparancy
 - Adding the colors (double exposure)

In addition to modes, the foreground can also be **scaled** and **translated**:
 - A button lets you pick between 100%, 50% and 25% size of the foreground.
 - Directional buttons allow you to move the foreground around

Finally, the foreground video input can be replaced by a still image loaded
from an SD card. Chroma keying can still be applied, to create borders or overlays.

TODO: Should we be able to save images, or is having a bank of overlays enough?

The LCD allows the user to orchestrate the modes and options to create quick
transitions, and saving states.

## Implementation overview
Both the background and foreground VGA signals get converted into digial pixel color values.
ADC chips do the conversion, and an FPGA reads the digital data.

The background video signal is never scaled or translated, and can be sent
almost directly out to the output DAC.
The output signal is just a few pixels behind the background.

The foreground gets buffered, either in SRAM or the FPGA's internal block RAM.
When the background signal is passing through the FPGA, it knows which pixel it is processing.
By checking its settings, it will know if and where the foreground framebuffer should be sampled.
Translations and scaling affect how the foreground overlays the background.

Scaling is done by doubling or quadrupling the current pixel coordinate (left shifting).
Translations are done by adding offsets. If the resulting pixel position
is still within the framebuffer bounds, it gets sampled.

Chroma key is handled by looking at the color chanels.
Transparancy is done by averaging the foreground and background layers.
To get 25% foreground, pixel = (fg+(fg<<1) + bg) >> 2, or something similar.
We don't want to do actual division.

User input is handled by an MCU, which is connected to the FPGA via SPI.
The MCU also handles the VGA input data pins, to communicate to our video sources
what resolution we support.

The SD card is also connected to the MCU over another SPI.
We use FAT32 formatted SD-cards, allowing the user to prepare overlays on their computer, saved as bmp.
See [this project here](https://create.arduino.cc/projecthub/SurtrTech/display-bmp-pictures-from-sd-card-on-tft-lcd-shield-f3074c).
The MCU does all file system work and image decoding. The LCD can be used to pick images by file name.
When selected, the foreground input gets ignored, and the FPGA fills the framebuffer with the image instead, sent over SPI.

In general, the MCU takes user input and uses SPI to send a state struct to the FPGA.
This means timed transitions, and switching between saved states, can be orchestrated from there.
Note that the MCU doesn't have enough RAM to store even a single frame.

## Components
This section contains the components we will need to place on the PCB.

### FPGA
We are working with the Artix-7 series, see the [Series Overview](https://docs.xilinx.com/v/u/en-US/ds180_7Series_Overview).

#### Dev-board
The [ARTY A7 development board](https://digilent.com/reference/programmable-logic/arty-a7/reference-manual) has an **ARTIX-7 XC7A35T** which has
 - 33280 logic cells
 - 50 blocks of 36Kib dual port block ram = 1 800Kib
 - 5 clock management tiles (CMTs), each with a phase-locked loop and mixed-mode clock manager
 - Programmable over JTAG or Quad-SPI Flash
 
The dev-board itself contains:
 - 256MB DDR3L with a 16-bit bus @ 667MHz
 - 16MB Quad-SPI Flash
 
#### On PCB
On the PCB we will use the **ARTIX-7 XC7A100T** in FTG256 package, which has already been procured.
 - 4,860 Kib of dual port block ram

#### Voltages
The FPGA requires, 1.0V, 1.8V and 3.3V, and they must become ready in order.
The 3.3V is the standard signal voltage we use.

#### Required FPGA support components
In the datasheet they define that certain sets of **decoupling capacitors** should be connected to certain pins.
This is already included in the example schematic from the lecture.
Try to get them close to the pins on the actual PCB.

We also need a **configuration flash chip** on the board,
to program the FPGA on startup.
TODO: Find supported flash chip (See UG908 Appendix C).

It must be programmable using **JTAG**, so we add a connector.
See the example setup from the lecture (e.g. the 10k pullups).

### SRAM
We need to calculate the required SRAM size.
With an 800x600=480 000 pixel grid, 16 bit color depth, it makes sense to
use a 512K x 16 bit SRAM. We do however also want to make the SRAM easy to address.

800 pixels of X-coordinate rounds up to 10 bits.
600 pixels of Y-coordinate also rounds up to 10 bits.
A 20 bit address means 2^20 = 1M x 16bit SRAM.

This size would also allow us to support 1024x768 (XGA),
but speed is a bigger issue. XGA has a pixel clock of 15.3ns / pixel.

To get good speed, we have looked at the syncronous 1M x 18b SRAM [CY7C1372KV33-200AXC](https://www.digikey.no/en/products/detail/cypress-semiconductor-corp/CY7C1372KV33-200AXC/5982740)
which supposedly costs 94kr per chip. It takes 3.3V.
With pipelining, the access time becomes 3ns,
with each operation delayed for two cycles.

The number of pins needed becomes:
 - 20 address pins
 - 18 data pins
 - write enable
 - clock
 - chip select
 - output enable

We use chip select to deselect, which when we don't need access.
The deselect is pipelined as well, allowing us to mix it into the operations.

We want to connect output enable to be sure that we never try to
write to th data line while it is being driven.
If we do everything perfectly, The data line is automatically tri-stated by writes,
but we want to be safe.

The rest, like output enable, clock enable, byte enable, advance etc. can all be pulled by the pcb.
This SRAM package does not expose JTAG pins.

### MCU
Available in the lab is the EFM32GG990F1024-BGA112
 - ARM Cortex M3
 - Flash: 1024kB
 - RAM: 128kB
 - USB support
 - ADC: 8 channels, 12 bit
 - DAC: 2 channels, 12 bit
 - SPI: 3
 - I2C: 2
 - GPIO: 87
 - 112 pin BGA package
 
### Connections
We use SPI between MCU and FPGA, and between MCU and SD-card. They can be fully independent.
For our SPI communication, the lecture tells us to add extra wires to the PCB.

[I2C](https://en.wikipedia.org/wiki/I%C2%B2C) is "open drain", which means we must use pull-up resistors on the lines.
Calculations can be found in this [EFM32GG application note](https://www.silabs.com/documents/public/application-notes/AN0011.pdf).

We use I2C for the LCD, and I2C for the ADCs.
They both seem to like 400kHz, which means a 2kOhm resistor is common.
They both also want to be pulled up to 3.3V.

### Power circuit
We base it on the PCB lecture. We need to power up the 1.0V, 1.8V and 3.3V in order.
The MCU and SD card use 3.3V, so does the SRAM.
The DRAM is a bit more complicated, see the DRAM section.

**Remember the jumpers** between the power circuit and rest of the board.

### Clock circuit
By using an external crystal, as recommended, we can run the MCU at 48MHz.
The FPGA can take the **clock out** signal from the EMF32GG as a clock input, and use phase-locked loops
to internally run faster clocks at rational multiples of the input.

We should also have a clock circuit on the FPGA side, so that we have two options

The clock circuit itself needs components.

### SD-card & SD-card slot
We want a slot to be able to move the SD-card to and from a computer.
When connected, the MCU communicates via SPI to the card.
In addition to the 4 SPI pins, The SD-card takes a 3.3V VDD and ground.

See for instance [this website](https://www.electroniccircuitsdesign.com/pinout/sd-microsd-card-pinout.html) for pinout on the physical SD and microSD cards.

Note that [this SO-post](https://electronics.stackexchange.com/questions/496357/sd-card-via-spi-pull-up-resistors-or-dedicated-ic)
talks about pull-up resistors on all the data pins, we can do that too.

People seem to disagree, but hooking up a 10k Ohm resistor to each of DAT0-DAT3 and CMD seems good.
Also a 0.1 uF decoupling capacitor between Vcc and GND.

To easily solder, we can buy the [10067847-001RLF](https://www.digikey.no/no/products/detail/amphenol-cs-fci/10067847-001RLF/2283478).
It is full size SD, with solder pins visible

### LCD text output
A cheap 2 row x 16 char width LCD screen is [DFR0555](https://www.digikey.no/no/products/detail/dfrobot/DFR0555/9356340).
It is controlled over I2C (with an address different to the EEPROMS, btw).
At 87mm x 32mm total, it is quite small. Each char is 2.96mm x 5.56mm.
Mouser doesn't have anything cheaper using I2C, so it's probably a good option.

### Input panel
We need to design a button layout that fits our functionality,
but also allows us to experiment after the PCB design is finalized.

We can use an off the shelf keyboard matrix, and re-label the keys.
Mouser stocks 81 of the 4x4 [3844](https://no.mouser.com/ProductDetail/Adafruit/3844?qs=qSfuJ%252Bfl%2Fd6WS5%252BJGim1hw%3D%3D).

### VGA ports
We need the physical VGA ports, as well as supporting components.
The speed we target is [SVGA 800x600@60Hz](http://tinyvga.com/vga-timing/800x600@60Hz).

The VGA connector itself is [described here](http://www.hardwarebook.info/VGA).
For VGA output we only really need to connect 5 pins, plus ground:
 - red, green, blue color (analog 0-0.7V, 75 Ohm)
 - horizontal sync, vertical sync (5V, but 3.3V makes do). See timing chart.

Note: While we are in the "porch" and sync pulse areas of the signal, the color values must all be 0V.
See for example the 2nd [Ben Eater VGA video](https://www.youtube.com/watch?v=uqY3FMuMuRo).

We also might want to connect the 3 DDC2B ports, in case we want to read info about the output monitor.
See the following section about DAC and VGA Display Data Channel.

For inputs, we need to read those 5 signals, including the analog color signals (using ADCs).
We should also let the video sources get info about our board over I2C. See next sections.

As for the physical VGA ports, there is [this](https://www.digikey.no/no/products/detail/amphenol-icc-commercial-products/L77HDE15SD1CH4FVGA/4888525) on digikey at 15kr a pop. Is has through-hole pins. Buy 9, 3 per PCB.

#### VGA Display Data Channel
The [DDC2B](https://en.wikipedia.org/wiki/Display_Data_Channel#DDC2) standard is an I2C protocol to communicate information about the "monitor" to the video source.

For the output port, we can connect the DDC pins to one of the MCU's I2C ports,
in case we want to read info about the screen.
NOTE: The I2C lines are open drain, and expect to be pulled up to 5V.
We will need a level shifter for the MCU to be able to use the I2C port.

##### Input VGA ports: I2C EEPROMs
On our VGA inputs, we should respond to I2C read commands, to return 128-256 bytes of read only [Extended Display Identification Data](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data).
See [this page](http://www.hardwarebook.info/VGA_(VESA_DDC)) for specifications.
The VGA DDC uses three pins: I2C data, I2C clock, and DC +5V delivered by the video source.
Our board must respond to the 7-bit I²C address 50h.

Remember that the I2C lines are open collector, and pulled to 5V. Our MCU would not handle that.

To save our MCU from being a slave over the VGA ports, we can buy dirt cheep I2C EEPROMSs.
For instance [M24C02-WMN6TP](https://www.digikey.no/no/products/detail/stmicroelectronics/M24C02-WMN6TP/1663568),
in stock at Digikey.no, at 1,7kr a piece. They work at 2.5V - 5.5V, and we use one per VGA input = 6 for 3 boards.

The I2C master will try to read from address 0xA1 = 0b1010 0001.
This is perfect! The EEPROM listens for address 0b1010 XXXX, where
we configure the first 3 X's ourselves. The last bit is 1 for read and 0 for write.

##### I2C EEPROM support components
 - Remember to add connectors for the chip to the board, to program it.
 - The Write Control should be pulled HIGH by default, and low only when being programmed.
 - When programming, the programmer is the master -> pull SDA and SCL up.
 - Programming can be done over 3.3V, while the VGA uses 5V
 - Vcc should be decoupled "usually of the order of 10 nF to 100 nF, close to the Vcc/Vss pins".
 - E0, E1 and E2 should be connected to ground, to listen the correct address.

The programming header thus becomes:
 - Vcc (3.3)
 - SCL
 - SDA
 - Write control (pulled low when programming)
 - Vss (ground)

#### Input VGA ADCs
The analog color inputs of each VGA input needs to become digital 6 or 5 bit values.
We can use ADCs with more bit depth than this, and then ignore the least significant bits.

The video ADC we looked at is the 3 channel x 10bit TI [TVP7002PZPR](https://no.mouser.com/ProductDetail/Texas-Instruments/TVP7002PZPR?qs=0OHPnBYoB%2FnBqXV%2Fq5aVXw%3D%3D)

It is configurable over I2C, things like gain, clamp and offset. Also programmable subpixel positioning.
It also has a PLL on the HSYNC, allowing it to generate a pixel clock for the FPGA VGA inputs.
The 40MHz on SVGA is well within the 12-165MHz range.

So: **we need to connect the ADCs to the MCU, to program them**.
Luckily, the chips have one bit of customizable address, so the options are:
 - 0xB8 = 0b10111000
 - 0xBA = 0b10111010
Remember that the last bit is 1 for read and 0 for write.
Notably, neither of these addresses collide with the 0xA0/0xA1 of the VGA DDC,
but you have to remember voltages.

The ADC needs 3.3V and **1.9V**! It says MIN 1.8V, but do we want to take any chances?

The I/O is however 3.3V, so we can connect it to a "normal" FPGA bank.

But: VGA HSYNC and VSYNC are 5V signals. We can **not** pass those into the ADC!
Can we simply create voltage dividers between ground and HSYNC/VSYNC?

##### ADC support components
The 3 unused Sync-on-green inputs should be connected to GND using in total:
 - 3x 1nF capacitors

The HSYNC PLL requires (see Figure 4)
 - Resistor 1.5 kOhm
 - Capacitor 4.7 nF
 - Capacitor 0.1 uF

To pull the address choice up or down they specify:
- 2.2 kOhm Resistor

We also need decoupling capacitors, and reference voltage resistors.
See the example setup in the datasheets.

#### Output VGA DAC
When outputing analog color data, we don't want to use resistors, as they take space,
and the FPGA outputs are not the best at delivering DC current.

A nice DAC that Mouser stocks is  [THS8135PHP](https://no.mouser.com/ProductDetail/Texas-Instruments/THS8135PHP?qs=wYV1UssyEL%2Fe7CEpgO3AeQ%3D%3D)
It has 3 10-bit inputs named R, G and B, and 240MS/s.
For 800x600 @ 60Hz, a throughput of at least 40 MSPS is enough.
It takes 3.3V analog, and 1.8V digital.
**This means the FPGA pins delivering VGA out must be 1.8V**.

 - 0.1 uF decoupling capacitors near each power pin (3 of them)
 - 0.1 uF capacitors between COMP-AVdd and between VREF-AVss
 - R_FS between FSADJ and AVss. TODO
 - 3x 75 Ohm resistors, for each analog output? TODO

It also has [PCB Layout Guidelines](https://www.ti.com/lit/an/slea077/slea077.pdf?ts=1663284093618).
10x trace width clearance between analog inputs.
Everything close! Resistors, capacitors, analog output port.
TODO: ESD protection on the analog outputs is recommended.

## Development components
While developing, we want to use some components that might not be part of the finished PCB.

### SD-card reader breakout board
A breakout SD card reader would be great to get the MCU dev-board to do SD-card reading, when developing.
We can make one ourselves by soldering onto the pads of a microSD-SD adapter Håvard has.

Håvard made a shitty one. Done!

### VGA breakout board
It costs a bit, but we can use the [DB15F-VGA-TERM](https://no.mouser.com/ProductDetail/Gravitech/DB15F-VGA-TERM?qs=WZeyYeqMOWcPAs%252BVmUTiLg%3D%3D) from Mouser.

### SRAM breakout board
The SRAM has a 14mm x 20mm 100-pin LQFP package.
We can buy [this board](https://www.digikey.no/no/products/detail/schmartboard,-inc./202-0010-02/9559360)

## Circuit board design
The maunals folder contains the PCB lecture.

For hooking up the FPGA, [this guy](https://www.youtube.com/watch?v=THLdycw9-Vs) from Altium Academy shows
the bare minimum of pins to run an ARTIX-7. Includes flash, voltages and and JTAG.

There is also the [7 Series FPGAs PCB Design Guide](https://docs.xilinx.com/v/u/en-US/ug483_7Series_PCB).
