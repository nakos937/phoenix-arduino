# phoenix-arduino

Convert a Gravis Phoenix Joystick to USB using an Arduino

The Gravis Phoenix is a 4-axis programmable Joystick with 24 buttons. Up to 4 buttons could be programmed to behave as standard joystick buttons, while the remainder could behave as if a key was pressed on the keyboard. To accomplish this, the joystick connected via both the 15 pin joystick port and a AT or PS/2 keyboard passthrough.
![Gravis Phoenix](images/gravis_phoenix.jpg "Gravis Phoenix")

It's no longer practical to use this joystick with modern computers due to the lack of a joystick port. Even USB joystick adapters do not make it possible to use the DOS boot disk configuration utility in order to reprogram the buttons.

This project will investigate removing the main logic board of the phoenix and replacing it with an Arduino which will also act as a USB input device. The [UnoJoy](https://github.com/AlanChatham/UnoJoy) project includes both Arduino code and host drivers in order to send axes positions and button states to a computer.

The initial focus will be on understanding the Phoenix's circuitry in order to make the proper connections and readings with the Arduino. To do this, I disassembled the joystick in order to investigate its components.

## Components

![Gravis Phoenix Exposed Bottom View](images/phoenix_bottom_open.jpg)

The Phoenix contains 5 Printed Circuit Boards (PCBs)
1. Micro PCB: Connects to the computer via joystick and keyboard connectors. Also connects to the Slider PCB via 6 pins and Wing PCB via 10 pins. This is the part we will remove and replace with an Arduino to modernize the joystick.

2. Slider PCB: Contains 2 potentiometers (X and Y axes) and a RJ11 connector which connects to the Handle PCB.
![Slider PCB](images/slider_pcb.png)

3. Handle PCB: Contains 8 buttons and connects to the Slider PCB via a RJ11 cable. Uses a HC165 8-bit parallel load shift register to make 8 button states available over the 4 wire connection.
![Handle Top](images/handle_disassembly1.png)
![Handle PCB w/ Buttons](images/handle_disassembly2.png)
![Handle PCB](images/handle_disassembly3.png)
![Handle PCB Diagram](images/handle_pcb.png)

4. Wing PCB: Contains 8 buttons and 2 potentiometers to read the throttle and rudder axes. Connects to the Throttle PCB via a 4 wire ribbon cable. Uses a HC165 8-bit parallel load shift register to allow the Micro PCB to read the button states.
![Wing PCB](images/wing_pcb.png)

5. Throttle PCB: Contains 8 buttons. I did not this component to map out the traces, since it's essentially the same as the Handle PCB with a different layout. It includes another HC165 8-bit parallel load shift register. I tested both combinations of SH/!LD and CLK/RD pins in order to label the connections to the Throttle PCB on the Wing PCB.
![Wing and Throttle](images/wing_throttle.png)

## HC165

Since the Phoenix uses 3 HC165 chips, it will be important to understand how it operates. The [SN74HC165 Data Sheet](images/SN74HC165.pdf) shows the pinout and describes the operation. There is also an [HC165 Arduino code sample](https://playground.arduino.cc/Code/ShiftRegSN74HC165N) which is available for experimentation. However, note in the Handle PCB that the CLK and Q_H(read) pins are actually connected to the same wire. Presumably this is a cost saving measure.

Briefly setting the SH/!LD pin LOW will cause the HC165 to load 8 bits in parallel from pins A-H. When SH/!LD is set HIGH, values on A-H is ignored and the value from H is immediately available on Q_H. At this point, each LOW to HIGH transition on CLK/Q_H is seen by the CLK pin and causes the HC165 to make the next data bit available (G,F,...). If the button is not pressed, the signal will remain HIGH. If the button is pressed, the voltage will be forced LOW, which is ok since it won't be interpreted as a clock transition. Here is a sample oscilloscope trace of the CLK/READ wire of the Handle/Slider PCBs while connected to the Micro PCB and powered via the joystick port. In this case the 5th button was pressed, showing substantially longer LOW voltage during that time.
![HC165 Clock/Read Oscilloscope Trace](images/clock_read_oscilloscope.png)

I used a Hantek HT6022BE Oscilloscope with the [OpenHantek Software](https://github.com/OpenHantek/openhantek) in order to collect the above traces. The oscilloscope is available for $65-70 USD and is an excellent troubleshooting tool.

## Prototyping with an Arduino Nano

The [Arduino Nano](https://store.arduino.cc/usa/arduino-nano) is a small device which should fit inside the Phoenix with plenty of clearance. Despite its small size, it uses an ATmega328P and exposes 8 analog inputs and 12 digital pins. Mine is a clone which enumerates with USB VID:1A86 PID:7523, identifying it as a CH340 uart. This requires a [driver from the manufacturer's website](http://www.wch.cn/download/CH341SER_EXE.html) in order to use the serial monitor in the Arduino IDE.

The image below shows how the nano is wired to work with the [phoenix-arduino-test.ino](examples/phoenix-arduino-test/phoenix-arduino-test.ino) test program.

![Arduino Nano Bread Board Prototype](images/arduino_nano_bread_board.png)

As long as no buttons are pressed, phoenix-arduino-test.ino continuously dumps the state of the potentiometers for each axis:
```
axes: X: adc:548 res:87 out:127 clmp:127
      Y: adc:553 res:85 out:127 clmp:127
      R: adc:697 res:47 out:63 clmp:63
      T: adc:702 res:46 out:63 clmp:63
```
* adc: 0-1023 reading from the Analog to Digital Converter
* res: resistance computed using Ohm's law from the ADC reading and the reference resistance. In the image above, note the reference resistors to the left of the Arduino near the analog input pins.
* out: resistance mapped linearly to a specified output range, currently set to 0-127 for each potentiometer
* clmp: the output value clamped to the output range, to ensure it is within spec

Each time one of the buttons' state changes, the test program dumps the button state. Below is sample output when the trigger and one of the wing buttons are pressed.
```
 group 01234567
handle .....0..
  wing .......0
thrttl ........
```

## Testing UnoJoy
