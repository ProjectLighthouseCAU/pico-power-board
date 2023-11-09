# Schematic, PCB Layout and BOM for the new Power-and-Distribution Board
This PCB is meant to be used with the Raspberry Pi Pico with Ethernet e.g. Wiznet W5100S-EVB-Pico or W5500-EVB-Pico or optionally WiFi with the Pi Pico W.

## EDA Software and Manufacturer
The Design was made with KiCAD, a free and open-source program for electronic design automation (EDA).
It was chosen because a proprietary solution would be both very costly (or only available as a student) and also not free. Furthermore, we don't need the features that a commercial solution like Altium offers.
The PCBs will be manufactured and assembled at JLCPCB, a large chinese PCB manufacturing company based in Shenzhen that offers cheap services, good quality, fast assembly and shipping and also has a lot of parts ready in their own storage or source them directly from LCSC (big component distributor in China).

## Purpose and features of the PCB
The PCB acts as a kind of motherboard for the Pi Pico (+ e.g. W5100S or W5500) which can be mounted on top using two rows of 20 pin header sockets.
Further, it has a barrel jack for a 12V power supply and a step down converter from 12V to 3.3V to power the Pico and lamp microcontrollers.
The board features six 20-pin IDC connectors where Lighthouse lamps can be connected via flat cables.  
The connectors (and cables) contain an SPI bus (MISO, MOSI, SCK, CS), 3.3V, Reset, 12V (on six wires) and GND (on 8 wires).  
Additional fearures are:
- polyfuses for each lamp with a detection (via voltage divider) if the fuse has tripped
- current measurement IC (TI INA219) with a shunt resistor to monitor power draw
- thermal sensor (TI TMP112A) to monitor temperature
- I2C and UART breakout pins for testing, adding devices and hacking stuff together
- signal termination resistors for SPI data lines and RESET to reduce ringing and over-/undershoot (series termination for SCK, MOSI, CS and RESET, thevenin termination for MOSI)

## Design Decisions

### Microcontroller
The Raspberry Pi Pico together with the Wiznet W5500 (or W5100S) provides a performant, reliable and cheap alternative to other ethernet capable microcontrollers.  
The previously used Atmel/Microchip ATTiny841 with the ENC28J60 was too unreliable and had insufficient performance and memory.
Ethernet capable STM32 chips would be a good alternative but those that feature Ethernet support are too expensive and also require a complete board design (instead of the now chosen "motherboard" solution).
Teensy boards have great performance but are way too expensive. Same holds for Raspberry Pis and other single board computers.  
In fact, after a lot of research we found many wireless capable microcontrollers (e.g. the official Pi Pico W and ESP32) because of the increasing popularity of the Internet of Things (IoT) but finding good and cheap Ethernet capable MCUs was difficult.  
Therefore we were glad that Wiznet released their Pi Pico based boards.
The C/C++ Pico SDK is well documented and easy to work with and the Wiznet library is also okay and features the Berkeley socket API and some examples.
Nevertheless, we decided to rewrite the firmware in Rust for the Pi Pico with the W5500.

### Motherboard instead of complete "bare" chip design
We chose to design a simple motherboard for the finished Pi Pico board because it was easier and faster to design with less possibilities for mistakes (at least on the MCU side).  
We simply added a DC-Jack, six IDC connectors and a step-down-converter together with some sensors, fuses and connected everything accordingly.

### Fuses
We need fuses for each lamp because if we have one 10A fuse for all lamps and one lamp has a short circuit, then up to 10A may flow through that one lamp which is too much and could cause a fire.  
We measured the maximum current that a lamp can draw which is about 1.6A and selected a 2A fuse. The selected fuse is a polyfuse (or resettable fuse) which automatically resets itself (conducts again) after the current returns to a normal < 2A.  
We wanted to monitor if a fuse has tripped, so we took the 12V after the fuse, added a voltage divider to 3V and hooked that back up to the Pi Pico and also the reset line to that lamp.  
Therefore the lamp automatically resets (because reset is active low) and we don't need extra GPIOs on the Pico. If the Pico wants to manually reset a lamp, it just has to change the pin to output and pull it low. The current that then flows through the voltage divider and the GPIO is low enough for the maximum rating of the GPIO.

### Signal termination resistors

After looking at the SPI signal with an oscilloscope, we found out that the fast switching of voltage levels causes reflections on the end of the longer cables. These reflections interfere with each other constructively and cause ringing (voltage swings around the desired voltage at a rising/falling edge) and therefore over-/undershoot (voltage goes above/below the GPIO pins rated maximum/minimum voltage).  
While the SPI communication works and nothing broke yet, this over-/undervoltage could damage the GPIO pins or the whole microcontroller (both RP2040 and ATtiny841) in the long run.  
To fix this issue, we need to terminate the transmission line (cable) with properly sized terminating resistors with a suitable termination technique.  
Because we only change the hardware on one side of the transmission line, we need to choose the termination technique accordingly.  
There are three suitable termination techniques:
- Series termination (series resistor at the transmitter)
- Parallel termination (parallel resistor at the receiver, pull down to GND)
- Thevenin termination (voltage divider at the receiver)

#### Series termination for outgoing data lines
For the transmitting data lines (SCK, MOSI, CS, RESET), we chose series termination since it is the only termination that is recommended at the source. It also is the most widely used technique, gives good results and has no extra power usage. The only downside is that the resistor limits the current and therefore increases the rise-/fall-times of the signal (which is negligable in our case).

#### Thevenin termination for incoming data line
For the one incoming data line (MISO) we tried to use parallel termination but this technique has the downside that it draws a lot of current from the transmitter (ATtiny841) which it cannot provide. So the signal couldn't reach the required voltage for a logic 1.  
Alternatively we chose Thevenin termination, which uses a voltage divider at the receiving end to pull the line to a voltage between Vcc and GND. This makes it "easier" for the driver to pull the line high and low and solved the insufficient voltage issue. We chose to use resistors of equal size to pull the signal exaclty at the half of Vcc.

#### Terminating resistor value
The value for the terminating resistors was determined experimentally.
Ideally one would consult the datasheet of the cable for its characteristic impedance, but unfortunately no information about the installed cables was found.
So, we tried out several different resistor values and measured the maximum and minimum voltage, rise and fall time and visually inspected the ringing with an oscilloscope.
The results showed that a 100 Ohm resistor is suitable for series termination and two 200 Ohm resistors for Thevenin termination (since they are considered to be in parallel from the signals perspective, the combined resistance is 100 Ohm).
So the cables characteristic impedance is somewhere around 100 Ohm (+/- measurement inaccuracies and + ouput driver impedance).

#### Problem: SPI over long distance unshielded cable
We are aware that transmitting SPI signals over long unshielded cables can and will cause interference problems.  
The old system was designed like this to save on cost.
Adding transceivers for some kind of differential transmission on both ends and using shielded cables would be the ideal solution.  
But designing, buying and installing new transceiver boards for the lamps, new cables and connectors would be very time consuming and expensive.  
Since the old system worked well enough, we decided to try out a lower frequency, a line driver IC (which we found out was not needed), signal termination resistors and add some error detection (checksum/CRC).
Unfortunately, the return channel (MISO / lamp -> controller) had the most problems especially with longer transmissions of 8 bytes. This is partly caused by the unbuffered SPI data register of the ATtiny841. When the lamp doesn't update the data register in time, the controller reads (part of) the previous value. The only solution to fix this (after all optimizations) is to reduce the frequency.  

### Step-Down-Converter
We decided to use a drop-in replacement for a linear regulator since it provides a finished and tested solution for efficient voltage regulation and we don't need to calculate inductances and capacitances.
After all we are computer scientists and not electrical engineers and therefore will probably make mistakes.

Note that the original Pi Pico has a Step-Up/Step-Down converter builtin (with a maximum input of about 5V).
But the Wiznet W5100S-EVB-Pico has a linear regulator.
Because two Step-Downs can cause interference and a linear regulator is not well suited for "converting" 3.3V to 3.3V (it needs at least 1V higher input voltage), we opted to power the Pico using the 3V3(OUT)-pin instead of the VSYS-pin which connects to the point after its internal linear regulator.

### DC-Barrel-Jack
The barrel jack should support our 12V power supply connectors and be rated for 10A. It should also have an inner pin diameter of 2.5mm and an outer barrel diameter of 5.5mm (the outer diameter can be larger if it has a spring mechanism that holds the plug).

### Debug and extension header
We added a 6-pin header as a breakout for some GPIOs of the pico.
They can be used for debugging or manually extending some functionality.  
These are:  
- UART (TX+RX)
- I2C (SDA+SCL)
- 3.3V
- GND

### Mounting Holes
The mounting holes are spaced to fit our old plastic cases.
(84mmx55.5mm)

### Current Measurement
We chose the INA219 from Texas Instruments together with a 10mOhm current shunt resistor. The INA219 features a current sensing amplifier, a precision ADC and an I2C interface.
We can configure the chip with the shunt resistance and then query voltage-drop, input-voltage, current and power.

### Temperature Measurement
The Pico has a builtin temperature sensor but it is not very accurate and also inside of the chip where any heat from the CPU is measured as well. We therefore added a small temperature sensor from Texas Instruments (TMP112A) which also has an I2C interface. This temperature sensor is installed near the barrel jack to measure the temperature increase from the current flowing through the 12V trace.

### Bypass capacitors
The step-down-converter has a 100nF and a 10uF capacitor on the 12V side and a 22uF capacitor on the 3.3V side (all ceramic as mentioned in the datasheet).
The temperature sensor has a 10nF and the current sensor has a 100nF capacitor (as mentioned in the datasheet).
The header for the Pi Pico has a 100nF and a 22pF capacitor at the 3.3V input.

## Insecurities / TODOs / To-Checks

### Trace width
Especially 12V high current traces (to the lamps).

### Trace spacing and routing
- enough spacing between tracks?
- too many vias?
- signal trace routing okay?

### Current return paths through GND plane
Is the current return path through the GND plane large and direct enough?
The signal traces on the bottom side were routed back to the top for short distances to keep a clear path from the GND of the connectors to the GND of the barrel jack.

### Double-check components
- check datasheets, bypass capacitors
- check footprints
- check placement and rotation files
- check BOM