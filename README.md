# Schematic, PCB Layout and BOM for the new Power-and-Distribution Board
This PCB is meant to be used with the Raspberry Pi Pico with Ethernet e.g. Wiznet W5100S-EVB-Pico or W5500-EVB-Pico or optionally WiFi with the Pi Pico W.

## EDA Software and Manufacturer
The Design was made with KiCAD, a free and open-source program for electronic design automation (EDA).
It was chosen because a proprietary solution would be both very costly (or only available as a student) and also not free. Furthermore, we don't need the features that a commercial solution like Altium offers.
The PCBs will be manufactured and assembled at JLCPCB, a large chinese PCB manufacturing company based in Shenzhen that offers cheap services, good quality, fast assembly and shipping and also have a lot of parts ready in their own storage or source them directly from LCSC (big component distributor in China).

## Purpose and features of the PCB
The PCB acts as a kind of motherboard for the Pi Pico (+ e.g. W5100S) which can be mounted on top using two rows of 20 pin header sockets.
Further, it has a barrel jack for a 12V power supply and a step down converter from 12V to 3.3V to power the Pico and lamp microcontrollers.
The board features six 20-pin IDC connectors where Lighthouse lamps can be connected via flat cables.  
The connectors (and cables) contain an SPI bus (MISO, MOSI, SCK, CS), 3.3V, Reset, 12V (on six wires) and GND (on 8 wires).  
Additional fearures are:
- polyfuses for each lamp with a detection (via voltage divider) if the fuse has tripped
- current measurement IC (TI INA219) with a shunt resistor to monitor power draw
- thermal sensor (TMP102AIDRLR) to monitor temperature
- I2C, UART and ADC breakout pins for testing, adding devices and hacking stuff together
- line buffer / driver for time critical SPI output lines (MOSI, CS (x6), SCK)

## Design Decisions

### Microcontroller
The Raspberry Pi Pico together with the Wiznet W5100S (or W5500) provides a performant, reliable and cheap alternative to other ethernet capable microcontrollers.  
The previously used Atmel/Microchip ATTiny841 with the ENC28J60 was too unreliable and had unsufficient performance and memory.
Ethernet capable STM32 chips would be a good alternative but those that feature Ethernet support are too expensive and also require a complete board design (instead of the now chosen "motherboard" solution).
Teensy boards have great performance but are way too expensive. Same holds for Raspberry Pis and other single board computers.  
In fact, after a lot of research we found many wireless capable microcontrollers (e.g. the official Pi Pico W and ESP32) because of the increasing popularity of the Internet of Things (IoT) but finding good and cheap Ethernet capable MCUs was difficult.  
Therefore we were glad that Wiznet released their Pi Pico based boards.
The Pico SDK is well documented and easy to work with and the Wiznet library is also okay and features the Berkeley socket API and some examples.

### Motherboard instead of complete "bare" chip design
We chose to design a simple motherboard for the finished Pi Pico board because it was easier and faster to design with less possibilities for mistakes (at least on the MCU side).  
We simply added a DC-Jack, six IDC connectors and a step-down-converter together with some sensors, a line-driver, fuses and connected everything accordingly.

### Fuses
We need fuses for each lamp because if we have one 8A fuse for all lamps and one lamp has a short circuit, then up to 8A may flow through that one lamp which is too much and could cause a fire.  
We measured the maximum current that a lamp can draw which is about 1.2A and selected a 1.5A fuse. The selected fuse is a polyfuse (or resettable fuse) which automatically resets itself (conducts again) after the current returns to a normal 1.2A.  
We wanted to monitor if a fuse has tripped, so we took the 12V after the fuse, added a voltage divider to 3V and hooked that back up to the Pi Pico and also the reset line to that lamp.  
Therefore the lamp automatically resets (because reset is active low) and we don't need extra GPIOs on the Pico. If the Pico wants to manually reset a lamp, it just has to change the pin to output and pull it low. The current that then flows through the voltage divider and the GPIO is low enough for the maximum rating of the GPIO.

### Line driver / buffer
We added a line drive IC (SN74HC541NSR) to the high speed SPI output lines (MOSI, SCK, CS x6).  
A line driver helps to charge long transmission lines up to the desired voltage (e.g. 3.3V) within a short time.  
Because a microcontrollers GPIO pins only have a small supply current, charing a long transmission line could take more time and therefore limit the transmission speed.  
For the SPI input line (MISO) we would ideally add a line driver as well, but that would need to be on the lamp side, which would require a redesign of the lamp PCB.  
We decided that the reset lines are not time critical since they are held high or low for a long time and rise/fall times are not that important. Also a simple line driver is unidirectional and then the reset line could not double as the fuse detection.

#### Problem: SPI over long distance unshielded cable
We are aware that transmitting SPI signals over long unshielded cables can and will cause interference problems.  
The old system was designed like this to save on cost.
Adding transceivers for some kind of differential transmission on both ends and using shielded cables would be the ideal solution.  
But designing, buying and installing new transceiver boards for the lamps, new cables and connectors would be very time consuming and expensive.  
Since the old system worked well enough, we decided to try out a lower frequency, a line driver IC and add some error detection (checksum/CRC).
Unfortunately, the return channel (MISO / lamp -> controller) had the most problems especially with longer transmissions of 8 bytes.  
TODO: test this with the new prototype

### Step-Down-Converter
We decided to use a drop-in replacement for a linear regulator since it provides a finished and tested solution for efficient voltage regulation and we don't need to calculate inductances and capacitances.
After all we are computer scientists and not electrical engineers and therefore will probably make mistakes.

Note that the original Pi Pico has a Step-Up/Step-Down converter builtin (with a maximum input of about 5V).
But the Wiznet W5100S-EVB-Pico has a linear regulator.
Because two Step-Downs can cause interference and a linear regulator is not well suited for "converting" 3.3V to 3.3V (it needs at least 1V higher input voltage), we opted to power the Pico using the 3V3(OUT)-pin instead of the VSYS-pin which connects to the point after its internal regulator.

### DC-Barrel-Jack
The barrel jack should support our 12V power supply connectors and be rated for 10A.

### Pull-Down/Up-Resistors
We added a pull down resistor to the MISO line since it would otherwise be floating when no lamp is transmitting.
We also added pull up resistors to all CS lines since they would be floating when the Pico starts or restarts.

### Debug and extension header
We added a 7-pin header as a breakout for some GPIOs of the pico.
They can be used for debugging or manually extending some functionality.  
These are:  
- UART (TX+RX)
- I2C (SDA+SCL)
- ADC
- 3.3V
- GND
#### Jumpers
We added two jumpers to connect two of the GPIO pins of the pico to the MOSI and SCK lines.
The two GPIOs can be used as I2C (SDA and SCL) alternatively.
We added this option in case we find out that I2C communication is more stable than SPI. Then we can just add the jumpers and rewrite the firmware.

### Mounting Holes
The mounting holes are spaced to fit our old plastic cases.
(84mmx55.5mm)

### Current Measurement
We chose the INA219 from Texas Instruments together with a 10mOhm current shunt resistor. The INA219 features an current sensing amplifier, a precision ADC and an I2C interface.
We can configure the chip with the shunt resistance and then query voltage-drop, input-voltage, current and power.

### Temperature Measurement
The Pico has a builtin temperature sensor but it is not very accurate and also inside of the chip where any heat from the CPU is measured as well. We therefore added a small temperature sensor from Texas Instruments (TMP102AIDRLR) which also has an I2C interface.

### Bypass capacitors
The step-down-converter has a 100nF and a 10uF capacitor on the 12V side and a 22uF capacitor on the 3.3V side (all ceramic as mentioned in the datasheet).
The temperature sensor has a 10nF and the current sensor has a 100nF capacitor (as mentioned in the datasheet)
The header for the Pi Pico has a 100nF and a 22pF capacitor at 3.3V input.

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

### Fuses
Are the fuses rated too tightly?
Maybe use 2A fuses instead in case the lamps draw a little more current?

### Double-check components
- check datasheets, bypass capacitors
- check footprints
- check placement and rotation files
- check BOM

### Temperature sensor
Maybe place it further away from potentially warm power traces to get the room temperature.
Or nearer to get the board temperature.