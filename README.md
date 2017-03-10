# My Nintendo Switch reverse engineering attempts

I'm just going to dump all my discoveries here, and hopefully they would be useful to the Nintendo Switch community.

If you think something is wrong, have some observations or hypothesis that I might have missed, stuff you want to contribute, or just general questions please feel free to ask here.

If you want to use the information here somewhere else, feel free to do so, but please credit me (dekuNukem) if possible.

## Left Joycon PCB Layout and test points

![Alt text](http://i.imgur.com/7Ui8lFv.jpg)

[Full-size PDF](./joycon_left_pcb.pdf).

The "JC" is for the 10-pin Joycon connector, see below for details.

### Remarks

* Joycon runs at 1.8V

* There is no silkscreen marking component and test point numbers, maybe Nintendo is trying to discourage people from doing funky things to the Joycon?

* Also, in a bizarre move, Nintendo didn't use the traditional "one side pulled-up and other side to ground" way of reading buttons, instead they used a keypad configuration where buttons are arranged to rows and columns. They used the keypad scanner built-in inside the BCM20734 with 128KHz clock for reading the buttons. That means it would be extremely hard to spoof button presses for TAS and twitch-plays. Maybe the Pro controller is different, need to buy one though.

* The only button that's not part of the keypad is the joystick button, which is still activated by pulling it down to ground.

## Left Joycon SPI flash dump

![Alt text](https://i.imgur.com/2c3tmyd.png)

Well it's not actually a dump, just a capture of the SPI lines when the Joycon is powered up (battery connected). I don't have time to go through it right now but of course you can if you want.

The SPI clock runs at 3MHz.

[Raw capture data](./logic_captures/leftjoyconspiflashpoweron.logicdata).

## Joycon to Console Communication

When attached to the console, the Joycon talks to it through a physical connection instead of Bluetooth. There are 10 pins on the connector, I'm just going to arbitrarily name it like this:

![Alt text](https://i.imgur.com/52xjlRb.jpg)

Looking at the pins on the left Joycon, the left most one is Pin 1, and the right most one is Pin 10. I simply removed the rumble motor, burned a hole on the back cover, and routed all the wires out through that.

[And here](./logic_captures/leftjoycon_docking.logicdata) is a capture of the docking of the left Joycon.

![Alt text](https://i.imgur.com/iUq5RNG.png)

## Joycon Connector Pinout

| Logic analyzer channel | Joycon Connector Pin |            Function           |                                                              Remarks                                                              |
|:----------------------:|:--------------------:|:-----------------------------:|:---------------------------------------------------------------------------------------------------------------------------------:|
|            -           |           1          |              GND              |                                                                 -                                                                 |
|            -           |           2          |              GND              |                                                                 -                                                                 |
|            0           |           3          |           BT status           | Only high when connected to console via bluetooth, low when console is powered off, unpaired, or attached to the console directly |
|            1           |           4          |               5V              |                                                     Joycon power and charging                                                     |
|            2           |           5          | Serial data console to Joycon |                                                    Inverted level (idle at GND)                                                   |
|            3           |           6          |               ?               |                                        1.8V open, changes during handshake, GND afterwards                                        |
|            -           |           7          |              GND              |                                                                 -                                                                 |
|            4           |           8          | Serial data Joycon to console |                                                   Standard level (idle at 1.8V)                                                   |
|            5           |           9          |               ?               |                                                           Always at GND                                                           |
|            6           |          10          |       Data packet sync?       |                                        Always goes low before the next data packet on Pin 8                                       |

* When first connected the baud rate is at 1000000bps(!), after the initial handshake the speed is then switched to 3125000bps(!!). The handshake probably exchanges information about the side of the Joycon, the color, and bluetooth address etc.

### Protocol

In normal operation the console asks Joycon for an update every 15ms (66.6fps), the command for requesting update is:


```
19 01 03 08 00 92 00 01 00 00 69 2d 1f
```

Around 4ms later, Joycon respond with a 61 bytes long answer, grouped in 15 4-byte packets, with 1 stop byte that is always 0xff.

One sample:

```
19 81 03 38 
00 92 00 31 
00 00 e9 2e 
30 7f 40 00 
00 00 65 f7 
81 00 00 00 
c0 23 01 e2 
ff 3e 10 0a 
00 d6 ff d0 
ff 23 01 e1 
ff 37 10 0a 
00 d6 ff cf 
ff 29 01 dd 
ff 34 10 0a 
00 d7 ff ce 
ff 
```

The first 8 byte is always ` 19 81 03 38 00 92 00 31 `, I'm not sure if this differs in different Joycons because I only took apart one.

### Button status

The 16th and 17th byte (on line 5, before `65 f7`) are the button status, when a button is pressed the corresponding bit is set to 1.

![Alt text](http://i.imgur.com/H7DUmCx.png)

### Joystick value

Byte 19 and 20 (`f7 81` between 5th and 6th line) are the Joystick values, most likely the raw 8-bit ADC data. Byte 19 is X while byte 20 is Y. Again, bizarrely, the X value is reversed, as in the `f7` should actually be `7f` (127 at neutral position). The Y value is correct though (`0x81` is 129).

### The rest of them

Still working on decoding those... It has to contain battery level, button status, joystick position, accelerometer and gyroscope data, and maybe more.

## Ending remarks

Right now I only took apart one left Joycon and yet to tear into the console itself, because I want to finish the Zelda first. But so far it looks like Nintendo make some really weird design decisions both in software and in hardware, probably to make what I'm doing more difficult. Anyway, I'll update this from time to time when I have new discoveries. Please share if you find this useful.

