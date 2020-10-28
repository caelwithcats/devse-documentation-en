<center>
<b> Warning! </b> <br> This article is in progress.
</center>

# Introduction
The COM port was commonly used as a hardware communication interface.
The COM port has been out of use for many years in favor of better alternitives such as USB.

Although they are obsolete, COM ports are still widely used for operating system development.
They are very simple to implement and are very useful for debugging because it allows you to log debug information to the terminal or file on the host machine.
They are also very useful because they can be initialized very early and therefore have debugging information efficiently.

For example, serial ports can send data and receive data, which would make it possible to make an external terminal using only that port.

RS-232 (which has been revised over and over again) is a standard that standardizes serial ports.
Initally created in 1981, it standardizes names (COM1, COM2, COM3, etc.) and limits the speed to 19200 Baud, which is more than enough for a small terminal (therefore potentially 19200 characters per second).

The limit being calculated in Baud, this being expressed in bit / s, 1 baud therefore corresponds to 1 bit per second.
The limit also depends on the distance of the connection with the wire, a long wire has a lower capacity than a short wire.

# Initialization
Each port needs to be initialized before use.

To start with, there are a few constant values ​​to know for each COM port.

| COM port    | Port Id       | IRQ           |
|-------------|---------------|---------------|
| COM1        | 0x3F8         | 4             |
| COM2        | 0x2F8         | 3             |
| COM3        | 0x3E8         | 4             |
| COM4        | 0x2E8         | 3             |

To initalize a COM port we must set some offsets. Each offset has a certain action.
(= PORT ID + OFFSET)

| offset | action |
| ------------- | ----------------------------------- -------------------------------------------------- -------------------------------------------------- -------------------------------------------------- ---- |
| 0 | The Data port of the COM, it is used to send and receive data, if the DLAB bit = 1 then it is to put the Baud divider (the lower bits) |
| 1 | The Interrupt port of the COM, it is used to activate the Interrupt of the port, if the DLAB bit = 1 then it is to put the value of the divider (of the Baud also but for the higher bits) |
| 2 | The interrupt identifier or the FIFO controller |
| 3 | the line control (The highest bit is the one for DLAB) |
| 4 | Modem control |
| 5 | The status of the line |
| 6 | The status of Modem |
| 7 | The scratch register |

To enable the DLAB you must put the port as indicated:
`PORT + 3 = 0x80 = 128 = 0b10000000`

```cpp
outb(COM_PORT + 3, 0x80);
```

To deactivate it, you just have to reset bit 8 to 0.

## Baud Rate
The COM port updates 115,200 times per second.
To control the speed, it is necessary to set up a divider, which can be used by activating the DLAB.

Then, the value must be passed through the offset 0 (the lower bits) and 1 (the upper bits).

Example allowing to put a divisor of 5 (then the port will have a 'rate' of 115200/5):
```cpp
outb (COM_PORT + 3, 0x80); // activate DLAB
outb (COM_PORT + 0, 5); // the smallest bits
outb (COM_PORT + 1, 0); // the highest bits
```

## Data size
You can set the size of the data sent to the COM port.
This can range from 5 bits to 8 bits

5bits = 0 0 (0x0)

6bits = 0 1 (0x1)

7bits = 1 0 (0x2)

8bits = 1 1 (0x3)

To define the size of the data, you must write it to the line control port (the smallest bits) having configured the port rate (and therefore having enabled the DLAB).
```cpp
outb(COM_PORT + 3, 0x3); // disable DLAB + put the data size to 8, the size of a char / unsigned char in c++
```

## Resources
- https://www.sci.muni.cz/docs/pc/serport.txt

### Written by @Supercip971, contribution by @busybox11
