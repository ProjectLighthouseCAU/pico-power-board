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
