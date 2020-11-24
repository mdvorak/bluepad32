# Bluepad32 for AirLift

## What is AirLift

AirLift is a family of [ESP32 modules][airlift-esp32] created by [Adafruit][adafruit].
These modules are co-processors, usually used only to bring WiFi or BLE to the main processor.

The AirLift module and the main processor talk to each other using the SPI protocol.
AirLift modules come with the [official nina-fw firmware][nina-fw].

Bluepad32 replaces the [official ESP32 firmware][nina-fw] that comes with AirLift
modules. Bluepad32 is "compatible-enough" with Nina-fw:

* Uses SPI, and the same GPIOs to talk to the main processor
* Uses the same Nina-fw protocol that runs on top of SPI
* But not all Nina-fw messages are implemented. Only the ones that are needed
  to have gamepad support working.

[adafruit]: https://www.adafruit.com
[airlift-esp32]: https://www.adafruit.com/product/4201

## Getting Bluepad32 for Airlift firwmare

### 1. Get a pre-compiled firmware

You can grab a precompiled firmware from here (choose latest version):

* https://github.com/ricardoquesada/bluepad32/tags

And download the `bluepad32-airlift.zip`. It includes a README with instructions.

### 2. Or compile it yourself

Read [README.md][readme] for the ESP-IDF requirements. And choose `airlift` as
the target platform:

```sh
$ export PLATFORM=airlift

# And then compile it

$ make -j
```

[readme]: https://gitlab.com/ricardoquesada/bluepad32/-/blob/master/README.md

## Install Bluepad32 for AirLift

In theory, flashing Bluepad32 is as easy as flahsing any firmware to any ESP32 module.
The problem is that AirLift modules were designed to be minimal, and not all
GPIOs are exposed, and might not be possible to flash the firmware directly using
`make flash`.

It depends from AirLift module to AirLift module. But to give a concrete example,
let's see how to install it on a [MatrixPortal M4][matrix_portal_m4]
(which has an AirLift module):

* Put MatrixPortal M4 in "boot" mode
* Flash the MatrixPortal-M4 [passthrough firmware]

And only now, you can do `make flash`, with just one exception: by default
`make flash` will try to reset ESP32, but it just won't work. Just use
`--before no_reset`. E.g:

```sh
# Double check port. Might be different on differente OSs
$ esptool.py --port /dev/ttyACM0 --baud 115200 --before no_reset write_flash 0x1000 ./build/bootloader/bootloader.bin 0x10000 ./build/bluepad32-airlift.bin 0x8000 ./build/partitions_singleapp.bin
```

[matrix_portal_m4]: https://learn.adafruit.com/adafruit-matrixportal-m4
[passthrough firmware]: https://learn.adafruit.com/adafruit-airlift-breakout/upgrade-external-esp32-airlift-firmware

## CircuitPython example

### Patching adafruit_esp32spi

The first thing that you need to do is to patch `adafruit_esp32spi`:

* https://github.com/adafruit/Adafruit_CircuitPython_ESP32SPI/pull/118

This will add the `get_gamepads_data` function to CircuitPython.

### Complete example

And this is how you would use it:

```python
import time

import board
import busio
from digitalio import DigitalInOut
from adafruit_esp32spi import adafruit_esp32spi

# If you are using a board with pre-defined ESP32 Pins:
#esp32_cs = DigitalInOut(board.ESP_CS)
#esp32_ready = DigitalInOut(board.ESP_BUSY)
#esp32_reset = DigitalInOut(board.ESP_RESET)

# If you have an AirLift Shield:
# esp32_cs = DigitalInOut(board.D10)
# esp32_ready = DigitalInOut(board.D7)
# esp32_reset = DigitalInOut(board.D5)

# If you have an AirLift Featherwing or ItsyBitsy Airlift:
# esp32_cs = DigitalInOut(board.D13)
# esp32_ready = DigitalInOut(board.D11)
# esp32_reset = DigitalInOut(board.D12)

# If you have an externally connected ESP32:
# NOTE: You may need to change the pins to reflect your wiring
esp32_cs = DigitalInOut(board.D10)
esp32_ready = DigitalInOut(board.D9)
esp32_reset = DigitalInOut(board.D6)

spi = busio.SPI(board.SCK, board.MOSI, board.MISO)
esp = adafruit_esp32spi.ESP_SPIcontrol(spi, esp32_cs, esp32_ready, esp32_reset, debug=0)

# Optionally, to enable UART logging in the ESP32
# esp.set_esp_debug(1)

# Should display "Bluepad32 for Airlift"
print('Firmware vers:', esp.firmware_version)

while True:
    gp = esp.get_gamepads_data()
    print('Gamepad: ', gp)
    time.sleep(0.032)
```

A more complete working example, is this "Paint for CircuitPython":

* https://gitlab.com/ricardoquesada/bluepad32/-/blob/master/tools/circuitpython_paint.py

## How to debug Bluepad32 for AirLift

Assuming that the ESP32 UART pins are not exposed (true for all AirLift modules),
the recommended way to debug the Bluepad32 firmware is:

* Get any ESP32 breakout module. It could be an [Adafruit HUZZAH32][esp32-adafruit] or [any other ESP32 breakout][amazon-esp32].
  * JUST DON'T GET A ESP32-S2 (doesn't have Bluetooth!). JUST "ESP32"
* Get a SAMD51 module, like the [Adafruit Feather M4 Express][feather_m4]
* Wire it like this:

|       | ESP32 | SAMD51 |
|-------|-------|--------|
| MOSI  | 14    | MOSI   |
| MISO  | 23    | MISO   |
| SCK   | 18    | SCK    |
| CS    | 5     | D10    |
| READY | 33    | D9     |
| RESET | EN    | D6     |

Something like this (left: Feather M4 express, right: HUZZAH32)

![fritzing](https://lh3.googleusercontent.com/pw/ACtC-3fNxNMUdaoBg7DGB6OPPDDnu_DQ15fmJS_I3crWjFKg7k3DA4HDeI8I_SUicSFamGuIVsHpM-myo5h-v1YOOFUU7lz6mU5tyExXDWZXedaYbUxhgf-GXfeZhMCdJCt1nZ04zFb1nyH86-pvZqc8yG9Y4A=-no)

NOTE: If you use a HUZZAH32, pay attention to the GPIO labels. The MOSI/MISO/SCK lables that are in the front of board, are not the ones that you should use. See the back of the board for the correct GPIO numbers.

In real life it might look more messy:

![wiring](https://lh3.googleusercontent.com/pw/ACtC-3dutrQXEj9I5zicNFW3K3PBbfge7MdwgB8dyi-wPSrtSp8zku3Y4c9WtBqQ9Bfa92xOjgSkZncAuzAZyc5F392tFkzkqWUl4YkfrKrM4e8TGP-B_7I7G_fRvFbIYbEQQIi-LlOnPU5SdGYYeW6hxxpJ_w=-no)

The benefits of using two separate modules (SAMD51 + ESP32) are:

* You can see the console of both the ESP32 and SAM51 at the same time
* Faster flashing: it is > 2x faster to flash Bluepad32 directly to the ESP32 than
  to flash it with "PassThrough" method required for AirLift modules.
* If needed, you can inspect the SPI protocol with a logic analizer

[esp32-adafruit]: https://www.adafruit.com/product/4172?gclid=EAIaIQobChMI-eeixraV7QIVED2tBh2qywzJEAQYASABEgLsTfD_BwE
[amazon-esp32]: https://www.amazon.com/s?k=esp32+module+breakout
[feather_m4]: https://www.adafruit.com/product/3857

## FAQ

**Does Bluepad32 only run on AirLift modules?**

**Bluepad32** can run on any ESP32 module and it is very easy to adapt it to
your own needs.

But in particular, **Bluepad32 for AirLift** can be used without custom changes
in any module that runs the [Nina-fw][nina-fw] firmware, including but not limited to:

* All Adafruit boards that include an AirLift module
* All Arduino boards that include a [NINA-W10 module][nina-w10] (although not tested)


[nina-fw]: https://github.com/adafruit/nina-fw
[nina-w10]: https://www.u-blox.com/en/product/nina-w10-series-open-cpu