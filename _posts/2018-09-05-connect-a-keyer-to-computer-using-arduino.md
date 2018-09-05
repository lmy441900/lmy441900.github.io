---
layout: post
title:  "Connect A CW Keyer To A Computer Using Arduino Leonardo"
date:   2018-09-05
categories: ham
---

Recently I take an interest in the amateur radio sports and I have been planning to get a license from [CRAC] for a month. Few days ago I bought a CW keyer (although I will not have the chance to operate on [MF] before I got the B license), but had no idea how to connect it to a computer.

[CRAC]: http://www.crac.org.cn/
[MF]:   https://en.wikipedia.org/wiki/Short_wave

![My keyer](/assets/keyer.jpg)

_Figure 1. My DJG-K4 keyer purchased from Taobao_

There are resolutions that disassembling an old mouse, soldering the two contacts to the pads of left button, turning the keyer to be a mouse's left button. However I have no old mouses so I just came out an idea: use an Arduino device that can simulate HID devices to turn the keyer into a mouse's left button. In my case I am using a CJMCU Beetle board, which is Arduino Leonardo compatible, only with less GPIOs.

![CJMCU Beetle](/assets/cjmcu-beetle.jpg)

_Figure 2. CJMCU Beetle development board, looks like a USB dongle_

It is quite simple to program the logic. Basically speaking, we set one of the digital pins to be an input, and connect one of the keyer's plug contact to it; we also connect GND to another plug contact (see the code below). When we detect a closed circuit on that pin, we use Arduino's Mouse library to press the mouse. When we detect an open circuit, we release the mouse. That's it!

```arduino
#include <Arduino.h>
#include <Mouse.h>

#define PIN_KEYER 9

void setup() {
    // Refer to https://www.arduino.cc/en/Tutorial/DigitalPins
    // for the meaning of INPUT_PULLUP
    pinMode(PIN_KEYER, INPUT_PULLUP);
    Mouse.begin();
}

void loop() {
    if (digitalRead(PIN_KEYER) == LOW) // low active due to pulled up
        Mouse.press();
    else
        Mouse.release();
}
```

There is another problem on how to gracefully connect the plug of keyer to the board. So far I randomly pick two wires to connect them. Later I may show my soldered, dedicated 6.5mm-to-USB converter here, if I ~~had~~have time :)

_Figure 3. A finely crafted converter should be shown here_

I use Lakey to practice morse code. It seems to be a (semi-)open-source[^1] software written by a Chinese HAM. It has a CW button on the user interface, and I can put the cursor on the CW button, then use my keyer to practice morse code sending. Lakey can also collect audio from the microphone, sample sounds from a frequency range, and turn CW signal into texts. It integrates the Koch training mechanism so that a user can train his morse code copying with a scientific method. HAMs recommend it, and so do I (for now, I am just a "sausage" (pre-HAM)).

![Lakey about](/assets/lakey.png)

_Figure 4. Lakey_

If you are also a HAM, I wish a QSO with you! 73!

## Notes

[^1]: On several Chinese download websites there says "... buddy needing the source code can visit the home page of Lakey to download it, but because the server is located in my home and is accessed using DDNS, a 100% 7x24 uptime cannot be guaranteed. If you can't download it, please send me an email or contact me over MSN." For reference see https://www.crsky.com/soft/20392.html.