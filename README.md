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

The foreground gets buffered, either in SRAM, DRAM or the FPGA's internal block RAM.
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

### DRAM

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
A small issue here might be the number of I2C ports on the MCU: 2.
For the VGA Display Data Channel, our board will be a slave device.
Since we have two VGA inputs, they might both want to drive the clock signal. 
The LCD screen probably uses I2C, so our MCU also needs to be a master.

### Power circuit
We base it on the PCB lecture. We need to power up the 1.0V, 1.8V and 3.3V in order.
The MCU and SD card use 3.3V.

Remember the jumpers between the power circuit and rest of the board.

### Clock circuit
By using an external crystal, as recommended, we can run the MCU at 48MHz.
The FPGA should take the same clock as input, and use phase-locked loops
to internally run faster clocks at rational multiples of the input.

The clock circuit itself needs components.

### SD-card & SD-card slot
We want a slot to be able to move the SD-card to and from a computer.
When connected, the MCU communicates via SPI to the card.
In addition to the 4 SPI pins, The SD-card takes a 3.3V VDD and ground.

### LCD text output

### Input panel
We need to design a button layout that fits our functionality,
but also allows us to experiment after the PCB design is finalized.

### VGA ports

#### VGA Display Data Channel
The [DDC2B](https://en.wikipedia.org/wiki/Display_Data_Channel#DDC2) standard is an I2C protocol to communicate information about the "monitor" to the video source.
Our device should respond to I2C read commands, to return 128-256 bytes of read only [Extended Display Identification Data](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data). 
The VGA DDC uses three pins: I2C data, I2C clock, and DC +5V delivered by the video source.
We should probably ignore that last one.

We can let the MCU handle the I2C communication. It has several I2C ports built in.

### ADCs
The analog color inputs of each VGA input needs to become digital 6 or 5 bit values.
We can use ADCs with more bit depth than this, and then ignore the least significant bits.
When configuring the gain on the ADC, I2C is used, which should be the MCU's task (right?? TODO).

#### ADC support components
It seems people expect the cable to be 75 Ohms, in
addition to 75 Ohm pull-down resistors on both the DAC output and the ADC input.

### DAC
When outputing analog color data, we don't want to use resistors, as they take space,
and the FPGA outputs are not the best at delivering DC current.
We have several contenders, both have 3 10-bit inputs named R, G and B.
Both can be driven from 3.3V and/or 1.8V, which we already have.
For 800x600 @ 60Hz, a throughput of at least 40 MSPS is enough.
Both DACs we look at have data registers.

One possible DAC is the 10 bit [THS8136](https://www.ti.com/product/THS8136).
It seems to be a bit difficult to purchase.

A guy on the EEVblog [forums](https://www.eevblog.com/forum/beginners/impedance-matching-on-custom-vga-dac/)
have used the **ADV7123** for VGA purposes, and said it worked.

It also has a "TTL input interface, and a high impedance, analog output current source".
Digikey.no has lots of versions of it [in stock](https://www.digikey.no/en/products/base-product/analog-devices-inc/505/ADV7123/24791).
Note: They need to be bought in bulk, and cost 100-200NOK per unit.

TODO: What is TTL, how much do we need to configure it. Can the MCU do the configuration?

## Development components
While developing, we want to use some components that might not be part of the finished PCB.

### SD-card reader breakout board
A breakout SD card reader would be great to get the MCU dev-board to do SD-card reading, when developing.

### VGA breakout board
To test both FPGA video output and Display Data Chanel info
