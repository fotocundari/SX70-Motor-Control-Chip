# SX70-Motor-Control-Chip Replacement
An open source Motor Control Chip Replacement for Polaroid SX70s.

## The Original Motor Driver Chips:
From about 1974 onward, Polaroid used a Texas Instruments motor control chip (MCC) labeled SN28648P in a DIP-8 package (with 3 pins cut off). Earlier Model 1 cameras used a similar motor control chip but integrated in a metal can Transistor Outline (TO) package with 6 pins.

The MCC was known to contain a "control circuit, an NPN drive transistor, and a PNP braking transistor" (https://e2e.ti.com/support/motor-drivers-group/motor-drivers/f/motor-drivers-forum/938479/sn28648p).

At a very basic level, we know that when the NPN line from the shutter's electronic control module (ECM) goes high, the NPN drive transistor is turned on, the PNP braking transistor is turned off, and the motor is driven forward. When the NPN line goes low, the NPN drive transistor turns off, the PNP braking transistor turns on, and the motor is actively braked to a stop. The braking works by the PNP transistor shorting the motor terminals together, causing the back EMF of the free spinning motor to create an opposing magnetic field to its forward direction, stopping the motor very quickly. It is interesting to note that while the majority of the shutters have a single "NPN" line controlling the motor (likely named by the fact that it turns on the NPN drive transistor), some rarer models also had a "PNP" line from the shutter, suggesting these earlier variants may have had independent drive and braking control.

Here is an oversimplified diagram of how the original MCC is believed to work with regards to the transistor switching:

![alt text](https://github.com/fotocundari/SX70-Motor-Control-Chip/blob/main/images/SCH_Schematic2_1-P1_2025-09-25.png)

We know from the documentation that the chip was much more complex than this oversimplification, and likely involved a comparator as well as some feedback and driver control circuits. However, given the era of the SX70s development and the fact the shutter ECM operated on I2L logic (a class of digital logic built on bipolar junction transistors), it is very possible the motor control chips developed during this time also operated using bipolar junction transistor logic, before CMOS (built with power efficient MOSFETs) dominated the integrated circuit logic in the late 70s. However until someone examines the silicon inside the MCC under a microscope, we can only speculate on its design. 

## Modern H-Bridge Drivers:
Many modern motor driver ICs include an H-Bridge. An H-Bridge is an arrangement of 4 switching elements (typically MOSFETs) arranged in an "H" pattern around the motor or load. The switching elements can be turned on/off in various combinations to achieve forward, reverse, coast, and braking of the motor. The below diagram from the Texas Instruments DRV8212 datasheet shows the configuration and current flow for the various H-Bridge states. 

![alt_text](https://www.ti.com/ods/images/SLVSFY9B/GUID-FF5A2ACA-6802-4E19-BB8D-7DBD46CAD9A6-low.gif)

## Selecting a Modern Driver:
In selecting a modern replacement there are many options to choose from. Many different motor driver chips can be used, but very often will require additional inputs/outputs to microcontrollers and modification of the connecting flex circuit to accommodate the differences from the original. There are some important considerations to make if we want to be able to use the driver safely and with the original flex circuit:

* Current/Voltage Rating - we want the chip to be able to handle at least a 6V motor with 2.5A of peak current and 1A continuous. 
* Simplicity - We are not driving the chip with a microcontroller and so we need a chip that can be controlled with only a single input allowing both forward drive and braking depending on the state of the input.
* Compatibility with the Original Flex (Independent Half-Bridge Control) - The original MCC controlled the motor by what is known as low side switching. This means the + terminal of the motor is always connected to +6V and the motor driver switches only the - side of the motor to Gnd. Therefore only the negative side of the motor is connected to 1/2 of the driver chip outputs. The ability to independently control each half of the H-bridge allows for such configuration without requiring both the + and - sides of the motor connected to the driver chip outputs. With independent 1/2 bridge control combined with low/high side switching, you can actually control two independent loads with one H-Bridge driver. However, independent half-bridge control also allows for the two half-bridges to be connected to the same side of the motor in parallel, doubling the current capacity and reducing RDS(on), which makes the driver more efficient for a single load.
* Size - The chip selected will also need to be small so that it can fit onto a daughter board with any other required components and have the inputs and outputs routed to where the original DIP-8 hole pattern (less 3 pins) was placed on the original flex.

Putting all of the above together, a search landed on the Texas Instruments DRV8212. (https://www.ti.com/product/DRV8212)

The DRV8212 can drive up to a 12V motor with a peak current of 4A (or up to 8A in parallel half bridge mode). The chip can be put in independent half-bridge mode by simply leaving the mode pin floating, and each half-bridge is controlled with a single input, requiring no extra inputs from the shutter ECM. This all fits into a tiny 2x2mm 8-WSON package allowing it to easily fit onto a daughter board that will fit within the original DIP-8 (less 3 pins) space. 

The below diagram from the DRV8212 data sheet shows the low side switching parallel half-bridge configuration best suited for our application:

![alt_text](https://www.ti.com/ods/images/SLVSFY9B/GUID-20200728-CA0I-C98H-SC96-TMC6ZJ5WBDBM-low.gif)

### Remaining Issues with the DRV8212:
* Logic Voltage Levels - The Vcc and Input voltages for the DRV8212's logic side have a max rating of 5.75V. Since the Polaroid SX70 uses a +6V power source, we have to reduce the logic side voltage by at least 0.25V to ensure we do not exceed this rating. Since a general purpose diode drops voltage by ~0.7V, a diode and current limiting resistor can be used to bring the logic side voltage safely below the maximum rating. Another option would be to use a linear voltage regulator but because the drop needed is so small, a simple diode is suitable. It is also important to note that the logic side Vcc is separate from the motor driving supply (Vm), which is directly connected to +6V and can handle up to 12V, and reducing the logic side Vcc will not affect motor power.
* Inverted Logic - The DRV8212 in low side parallel half-bridge configuration will drive the motor when the input is low, and brake the motor when the input is high. This is the inverse of the normal ECM signals outputted to the original driver through the NPN line. Therefore we need to invert the NPN signal. This can be done in various ways but simple schmitt-trigger inverter IC such as the Texas Instruments SN74LVC1G14DRLR works well and comes in a very small package. It too needs its supply voltage dropped below 5.5V. 
* Decoupling and Bulk Capacitors - Looking at the data sheet for the DRV8212 the chip also requires decoupling and bulk capacitors. Decoupling capacitors smooth out any high-frequency noise on the chips power supply lines. They are typically small with a recommended capacitance of 0.1uF. A bulk capacitor smooths the effects of low-frequency current fluctuations when the motor switches. The low frequency current fluctuations are a factor of the current required by the motor, supply voltage ripple, braking needs, and parasitic inductance between the motor and driver chip (a reason the driver chips are likely located close to the motor). The bulk capacitor maintains stable voltage and allows high current to be supplied quickly. Texas Instruments recommends at least a 10uF bulk capacitor on their motor drivers, but warns testing is required as your mileage may vary. The larger the capacitor the more space is required. For the current design, a 22uF was selected since it has 2X the minimum recommended capacitance and is the largest readily available capacitor found in a small 0402 SMD package. However, it is also likely 0603 capacitors up to 100uF would fit if issues are noticed on testing.

It should also be noted you will actually often find Polaroid added bulk capacitors directly to the motor terminals on some cameras sent in for repair due to mid cycle shutdowns. Therefore if you wanted to increase the bulk capacitance you could also add a larger capacitor directly to the motor terminals without sacrificing space on the motor control replacement pcb.  

# MCC Replacement Design:
Putting all of the above together, the following circuit was designed to replace the MCC chip:
![alt_text](https://github.com/fotocundari/SX70-Motor-Control-Chip/blob/main/images/SCH_Schematic1_1-P1_2025-09-26b.png)

Components were selected for the space and SMT assembly by JLCPCB, and a PCB was designed to fit the circuit onto a 10mmx10mm PCB with through hole patterns to match the DIP-8 hole configuration (less 3 pins) on the original body flex. 

To create the pins to mimic the DIP pins of the original chips, 22 gauge resistor legs were used. 22 gauge resistor legs can purchased as tin plated 0 Ohm through hole resistor jumpers manufactured by Stackpole Electronics with a part #	
JW60ZT0R00 (https://www.digikey.ca/en/products/detail/stackpole-electronics-inc/JW60ZT0R00/1742973).  

## Component List:
* Motor Driver - Texas Instruments DRV8212DSGR (8-WSON 2x2)
* Inverter IC - Texas Instruments SN74LVC1G14DRLR 1.65-5.5V Schmitt-Trigger Inverter
* Decoupling Capacitors x 3 (C1,C2,C3) - 0.1uF 10V ceramic X5R capacitors (0201 SMD size)
* Bulk Capacitor (C4) - 22uF 10V ceramic X5R (0402 SMD size)
* Diodes x 2 - 1N4448X-TP - 75V/250mA diode (SOD-523 size)
* Resistor R1 - Current limiting resistor for DRV8212 logic side VCC - 1kOhm 1/20W (0201 SMD size)
* Resistor R2 - Current limiting resistor for Inverter VCC - 200Ohm 1/20W (0201 SMD size)
* Pins x 5 - 22 gauge 0 Ohm axial jumpers - Stackpole JW60ZT0R00 (22 gauge axial resistor leads)

## PCB:
![alt_text](https://github.com/fotocundari/SX70-Motor-Control-Chip/blob/main/images/mccpcb.png)

## Finished Motor Control Chip Replacement:
![alt_text](https://github.com/fotocundari/SX70-Motor-Control-Chip/blob/main/images/20250926_111121.jpg)
![alt_text](https://github.com/fotocundari/SX70-Motor-Control-Chip/blob/main/images/20250926_111226.jpg)
![alt_text](https://github.com/fotocundari/SX70-Motor-Control-Chip/blob/main/images/20250926_111539.jpg)
