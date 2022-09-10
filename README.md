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
 - Showing the foreground at 33%, 50% or 67% transparancy
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
When the background signal is passing through the FPGA, it checks the settings and
foreground frame buffer to see what, if anything, should happen to the pixel before it is output.

User input is handled by an MCU, which is connected to the FPGA via SPI.
The MCU also handles the VGA input data pins, to communicate to our video sources
what resolution we support.
The SD card is also connected to the MCU over another SPI.
We use FAT32 formatted SD-cards, allowing the user to prepare overlays on their computer, saved as bmp.
See [this project here](https://create.arduino.cc/projecthub/SurtrTech/display-bmp-pictures-from-sd-card-on-tft-lcd-shield-f3074c).
The LCD can be used to pick images by file name.

In general, the MCU takes user input and uses SPI to send a state struct to the FPGA.
This means timed transitions, and switching between saved states, can be orchestrated from there.
Note that the MCU doesn't have enough RAM to store even a single frame.

## Parts
This section contains the components we will need to place on the PCB.

### FPGA
We are working with the Artix-7 series, see the [Series Overview](https://docs.xilinx.com/v/u/en-US/ds180_7Series_Overview).

#### Dev-board
The ARTY development board has an **ARTIX-7 XC7A35T**
 - 33280 logic cells
 - 50 blocks of 36Kib dual port block ram = 1 800Kib
 - Max 250 user I/O pins, but less than 100 accessible on dev-board
 
The actual dev-board itself contains:
 - DRAM (TODO)
 
#### On PCB

#### Required accessories
