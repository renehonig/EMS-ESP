# EMS-ESP

EMS-ESP is a project to build a controller circuit running with an ESP8266 to communicate with EMS (Energy Management System) based Boilers and Thermostats from the Bosch range and compatibles such as Buderus, Nefit, Junkers etc.

There are 3 parts to this project, first the design of the circuit, second the code for the ESP8266 microcontroller firmware and lastly an example configuration for Home Assistant to monitor the data and issue direct commands via MQTT.

[![Codacy Badge](https://api.codacy.com/project/badge/Grade/b8880625bdf841d4adb2829732030887)](https://app.codacy.com/app/proddy/EMS-ESP?utm_source=github.com&utm_medium=referral&utm_content=proddy/EMS-ESP&utm_campaign=Badge_Grade_Settings)
[![version](https://img.shields.io/badge/version-1.3.0-brightgreen.svg)](CHANGELOG.md)

- [EMS-ESP](#ems-esp)
  - [Introduction](#introduction)
  - [Supported Boilers Types](#supported-boilers-types)
  - [Supported ESP8266 devices](#supported-esp8266-devices)
  - [Getting Started](#getting-started)
  - [Monitoring The Output](#monitoring-the-output)
  - [Building The Circuit](#building-the-circuit)
    - [Powering The EMS Circuit](#powering-the-ems-circuit)
  - [How The EMS Bus Works](#how-the-ems-bus-works)
    - [EMS IDs](#ems-ids)
    - [EMS Polling](#ems-polling)
    - [EMS Broadcasting](#ems-broadcasting)
    - [EMS Reading and Writing](#ems-reading-and-writing)
  - [The ESP8266 Source Code](#the-esp8266-source-code)
    - [Supported EMS Types](#supported-ems-types)
    - [Which thermostats are supported?](#which-thermostats-are-supported)
    - [Customizing The Code](#customizing-the-code)
    - [Using MQTT](#using-mqtt)
    - [The Basic Shower Logic](#the-basic-shower-logic)
  - [Home Assistant Configuration](#home-assistant-configuration)
  - [Building The Firmware](#building-the-firmware)
    - [Using PlatformIO Standalone](#using-platformio-standalone)
    - [Building Using Arduino IDE](#building-using-arduino-ide)
  - [Using the Pre-built Firmware](#using-the-pre-built-firmware)
  - [Troubleshooting](#troubleshooting)
  - [Known Issues](#known-issues)
  - [Wish List](#wish-list)
  - [Your Comments and Feedback](#your-comments-and-feedback)
  - [DISCLAIMER](#disclaimer)

## Introduction

My original intention for this home project was to build a custom smart thermostat that interfaces with my Nefit Trendline HRC30 boiler and have it controlled via a mobile app using [Home Assistant](https://www.home-assistant.io/). I had a few cheap ESP32s and ESP8266s microcontrollers lying around from previous IoT projects and learning how to build a circuit to decode the EMS messages seemed like a nice challenge.

Firstly, some acknowledgments and kudos to the following people who have open-sourced their projects which have helped me tremendously:

 **susisstrolch** - Probably the first working version of the EMS bridge circuit I found designed for the ESP8266. I borrowed Juergen's [schematic](https://github.com/susisstrolch/EMS-ESP12) and parts of his code logic for reading telegrams.

 **bbqkees** - Kees built a working [circuit](https://shop.hotgoodies.nl/ems/) and some sample Arduino code to read from the EMS and push messages to Domoticz. His SMD board is also available for purchase.

 **EMS Wiki** - A comprehensive [reference](https://emswiki.thefischer.net/doku.php?id=wiki:ems:telegramme) (in German) for the EMS bus which is a little outdated, not 100% accurate and unfortunately no longer maintained.

## Supported Boilers Types

Most Bosch branded boilers that support the Logamatic EMS (and EMS+) bus protocols work with this design. This includes Nefit, Buderus, Worcester and Junkers (all copyrighted). Please make sure you read the **Disclaimer** carefully before sending ambiguous messages to your EMS bus as you could cause serious damage to your boiler or thermostat.

## Supported ESP8266 devices

I've tested the code and circuit with a few ESP8266 development boards such as the Wemos D1 Mini, Wemos D1 Mini Pro, Nodemcu0.9 and Nodemcu2 dev boards. It will also work on bare ESP8266 chips such as the ESP-12E but do make sure you disabled the LED support and wire the UART correctly as the code doesn't use the normal Rx and Tx pins. Why is explained later.

## Getting Started

1. Either build the circuit described below or purchase a ready built board from bbqkees via [website](https://shop.hotgoodies.nl/ems/).
2. Get an ESP8266 dev board and connect the 2 EMS output lines from the boiler to the circuit and the Rx and Tx out to ESP pins D7 and D8 respectively. The EMS connection can either be the 12-15V AC direct from the thermostat bus line or from the 3.5" Service Jack at the front.
3. Optionally connect an external LED or decide to use the onboard ESP8266 LED. This will flash when there is an error on the EMS bus line or stay solid when it's connected.
4. Modify `my_custom.h`
5. Build and upload the firmware to the ESP8266 device. I used PlatformIO with Visual Studio Code but using Atom or a command-line is just as easy if you don't plan to make code changes. Do make sure you set the MQTT and WiFi credentials correctly in the build flags and if you're not using MQTT leave the MQTT_IP blank. The firmware supports OTA too with the default hostname as 'boiler'.
6. Power the ESP either via USB or direct into the 5v vin pin from an external power 5V volts supply with min 400mA.
7. Attach the 3v3 out on the ESP8266 to the DC power line on the EMS circuit as indicated in the schematics.
8. The WiFi connects via DHCP by default. Find the IP by from your router and then telnet (port 23) to it. If a connection can't be made it will go into Access Point mode. Tip: to enable Telnet on Windows run `dism /online /Enable-Feature /FeatureName:TelnetClient` or install something like [putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html). If everything is working you should see the messages appear in the window as shown in the next section. However if you're unable to locate the IP of the ESP then something went wrong. Re-compile with the -DDEBUG_SUPPORT and connect via USB to a PC and check the Serial log for errors.

## Monitoring The Output

Use the telnet client to inform you of all activity and errors real-time. This is an example of the telnet output:

![Telnet](doc/telnet/telnet_example.jpg)

Type 'log v' and Enter and you'll be seeing verbose logging messages. ANSI colors with white text for info messages, green are for broadcast telegrams, yellow are the ones sent to us and red are for unknown data or telegrans which have failed the CRC check.

![Telnet](doc/telnet/telnet_verbose.PNG)

To see the current stats and collected values type 'info'. Watch out for unsuccessful telegram packets in the #CrcErrors line.

![Telnet](doc/telnet/telnet_stats.PNG)

**Disclaimer: be careful when sending values to the boiler. If in doubt you can always reset the boiler to its original factory settings by following the instructions in the user guide. On my **Nefit Trendline HRC30** that is done by holding down the Home and Menu buttons simultaneously for a few seconds, selecting factory settings from the scroll menu followed by pressing the Reset button.**

## Building The Circuit

The EMS circuit is really all credit to the hard work many people have done before me, noticeably *susisstrolch* with his ESP8266 [version](https://github.com/susisstrolch/EMS-ESP8266_12-PCB/tree/newmaster/Schematics/EMS-ESP8266-12).

I've included a prototype boards you can build yourself on a breadboard. One part for only reading values from the Boiler and an extension with the write logic so you can also send commands.

We need the Rx/Tx of the ESP8266 for flashing, so the code in ``emsuart.cpp`` switches the UART pins to use RX1 and TX1 (GPIO13/D7 and GPIO15/D8 respectively). This also prevents any bogus stack data being sent to EMS bus when the ESP8266 decides to crash with a Watch Dog (WD) Reset.

The breadboard layout was done using [DIY Layout Creator](https://github.com/bancika/diy-layout-creator) and sources files are included in this repo.

![Breadboard Circuit](doc/schematics/breadboard.png)

The schematic used (as designed by [susisstrolch](https://github.com/susisstrolch/EMS-ESP8266_12-PCB)):

![Schematic](doc/schematics/circuit.png)

*Optionally I've also added 2 0.5A/72V polyfuses between the EMS and the two inductors L1 and L2 for extra protection.*

And here's a version using bbqkees' circuit

![WemosD1](doc/schematics/wemos_kees.png)

### Powering The EMS Circuit

The EMS circuit will work with both 3.3V and 5V. It's easiest though to power directly from the ESP8266's 3V3 line and run a steady 5V into the microcontroller. Powering the ESP8266 microcontroller can be either:

- via the USB if your dev board has one
- using an external 5V power supply into the 5V vin on the board
- powering from the 3.5" service jack on the boiler. This will give you 8V so you need a buck converter (like a [Pololu D24C22F5](https://www.pololu.com/product/2858)) to step this down to 5V to provide enough power to the ESP8266 (250mA at least)
- powering from the EMS line, which is 15V A/C and using a buck converter as described above. Note the current design has stability issues when sending packages in this configuration so this is not recommended yet if you plan to many send commands to the thermostat or boiler.

| With Power Circuit                              |
| ------------------------------------------ |
| ![Power circuit](doc/schematics/Schematic_EMS-ESP-supercap.png) |

## How The EMS Bus Works

Packages are sent to the EMS "bus" from the Boiler and any other compatible connected devices via serial TTL transmission. The protocol is 9600 baud, 8N1 (8 bytes, no parity, 1 stop bit). Each package is terminated with a break signal `<BRK>`, a 11-bit long low signal of zeros.

A package can be a single byte (see Polling below) or a string of 6 or more bytes making up an actual data telegram. A telegram is always in the format:

``[src] [dest] [type] [offset] [data] [crc] <BRK>``

I reference the first 4 bytes as the *header* in this document.

### EMS IDs

Each device has a unique ID.

The Boiler has an ID of 0x08 (type MC10) and also referred to as the Bus Master or UBA.

My thermostat, which is a* Moduline 300* uses the RC30 protocol and has an ID 0x17. If you're using a RC35 type thermostat such as the newer Moduline 300s or 400s use 0x10 and make adjustments in the code as appropriate. bbqkees did a nice write-up on his github page [here](https://github.com/bbqkees/Nefit-Buderus-EMS-bus-Arduino-Domoticz/blob/master/README.md).

Our circuit acts as a service key and thus uses an ID 0x0B. This ID is reserved for special devices intended for installation engineers for maintenance work.

### EMS Polling

The bus master (boiler) sends out a poll request every second by sending out a sequential list of all possible IDs as a single byte followed by the break signal. The ID always has its high 8th bit (MSB) set so in the code we're looking for 1 byte messages matching the format `[dest|0x80] <BRK>`.

Any connected device can respond to a Polling call with an acknowledging by sending back a single byte with its own ID. In our case we would listen for a `[0x8B] <BRK>` (meaning us) and then send back `[0x0B] <BRK>` to say we're alive and ready. Although I found this is not needed for normal operation so it's disabled as default in the code.

Polling is also the trigger to start transmitting any packages queued for sending. It must be done within 200ms or the bus master will time out.

### EMS Broadcasting

When a device is broadcasting to everyone there is no specific destination needed. `[dest]` is always 0x00.

The tables below shows which types are broadcasted regularly by the boiler (ID 0x08) and thermostat (ID 0x17). The **data length** is excluding the 4 byte header and CRC and the **Name** references those in the [ems wiki](https://emswiki.thefischer.net/doku.php?id=wiki:ems:telegramme).

| Source (ID)   | Type ID | Name                | Description                            | Data length | Frequency  |
| ------------- | ------- | ------------------- | -------------------------------------- | ----------- | ---------- |
| Boiler (0x08) | 0x34    | UBAMonitorWWMessage | warm water temperature                 | 19 bytes    | 10 seconds |
| Boiler (0x08) | 0x18    | UBAMonitorFast      | boiler temps, power, gas/pump switches | 25 bytes    | 10 seconds |
| Boiler (0x08) | 0x19    | UBAMonitorSlow      | boiler temp and timings                | 22 bytes    | 60 seconds |
| Boiler (0x08) | 0x1C    | UBAWartungsmelding  | maintenance messages                   | 27 bytes    | 60 seconds |
| Boiler (0x08) | 0x2A    | n/a                 | status, specific to boiler type        | 21 bytes    | 10 seconds |
| Boiler (0x08) | 0x07    | n/a                 | ?                                      | 21 bytes    | 30 seconds |

| Source (ID)       | Type ID | Name              | Description                                         | Frequency  |
| ----------------- | ------- | ----------------- | --------------------------------------------------- | ---------- |
| Thermostat (0x17) | 0x06    | RCTime            | returns time and date on the thermostat             | 60 seconds |
| Thermostat (0x17) | 0x91    | RC30StatusMessage | returns current and set temperatures                | 60 seconds |
| Thermostat (0x17) | 0xA3    | RCTempMessage     | returns temp values from external (outdoor) sensors | 60 seconds |

Refer to the code in `ems.cpp` for further explanation on how to parse these message types and also reference the EMS Wiki.

### EMS Reading and Writing

Telegram packets can only be sent after the Boiler sends a poll to the sending device. The response can be a read command to request data or a write command to send data. At the end of the transmission a poll response is sent from the client (`<ID> <BRK>`) to say we're all done and free up the bus for other clients.

When doing a request to read data the `[src]` is our device `(0x0B)` and the `[dest]` must have has it's MSB (8th bit) set. Say we were requesting data from the thermostat we would use `[dest] = 0x97` since RC30 has an ID of 0x17.

Following a write request, the `[dest]` doesn't have the 8th bit set and after this write request the destination device will send either a single byte 0x01 for success or 0x04 for failure.

Every telegram sent is echo'd back to Rx.

## The ESP8266 Source Code

*Disclaimer*: This code here is really for reference only, I don't expect anyone to use "as is" since it's highly tailored to my environment and my needs. Most of the code however is self explanatory with comments here and there in the code.

`emsuart.cpp` handles the low level UART read and write logic. You shouldn't need to touch this. All receive commands from the EMS bus are handled asynchronously using a circular buffer via an interrupt. A separate function processes the buffer and extracts the telegrams. Since we don't send too many write commands this is done sequentially. I couldn't use the standard Arduino Serial implementation because of the 11-bit break signal causes a frame-error which gets ignored.

`ems.cpp` is the logic to read the EMS packets (telegrams), validates them and process them based on the type.

`ems-esp.ino` is the Arduino code for the ESP8266 that kicks it all off. This is where we have specific logic such as the code to monitor and alert on the Shower timer and light up the LEDs. LED support is enabled by default and can be switched off at compile time using the -DNO_LED build flag.

`my_config.h` all the custom settings

`ems_devices.h` has all the configuration for the known EMS devices this supports

`MyESP.cpp` is my custom library to handle WiFi, MQTT, MDNS and Telnet. Uses a modified version of TelnetSpy (https://github.com/yasheena/telnetspy)

### Supported EMS Types

`ems.cpp` defines callback functions that handle all the broadcast types listed above (e.g. 0x34, 0x18, 0x19 etc) plus these extra types:

| Source (ID)       | Type ID          | Name                          | Description                              |
| ----------------- | ---------------- | ----------------------------- | ---------------------------------------- |
| Boiler (0x08)     | 0x33             | UBAParameterWW                | reads selected & desired warm water temp |
| Boiler (0x08)     | 0x14             | UBATotalUptimeMessage         |                                          |
| Boiler (0x08)     | 0x15             | UBAMaintenanceSettingsMessage |                                          |
| Boiler (0x08)     | 0x16             | UBAParametersMessage          |                                          |
| Thermostat (0x17) | 0xA8             | RC20Set                       | sets operating modes for an RC20         |
| Thermostat (0x10) | 0xA7             | RC30Set                       | sets operating modes for an RC30         |
| Thermostat        | 0x02             | Version                       | reads Version major/minor                |
| Thermostat        | 0x91, 0x41, 0x0A | Status Message                | read monitor values                      |

In `ems-esp.ino` you can make calls to automatically request these types in the function *regularUpdates()*.

### Which thermostats are supported?

I am still working on adding more support to known thermostats. Any contributions here are welcome. The know types are listed in `ems_devices.h`.

- RC20 and RC30 are fully supported
- RC35 only supports the 1st heating circuit (HC1)
- TC100/TC200/Easy has only support for reading the temperatures. There seems to be no way to set settings using EMS bus messages. One option is to send XMPP messages but a client is needed.

### Customizing The Code

- To configure for your thermostat and specific boiler settings, modify `my_config.h`. Here you can
  - set flags for enabled/disabling functionality such as `BOILER_SHOWER_ENABLED` and `BOILER_SHOWER_TIMER`.
  - Set WIFI and MQTT settings, instead of doing this in `platformio.ini`. The values can be set from the telnet command menu using the **set** command.
- To add new handlers for EMS data types, first create a callback function and add to the `EMS_Types` array at the top of the file `ems.cpp` and modify `ems.h`
- To add new devices modify `ems_devices.h` 

### Using MQTT

The boiler data is collected and sent as a single JSON object to MQTT TOPIC `home/ems-esp/boiler_data`. The `home` preifx is the MQTT topic prefix and can be customized in `my_config.h`. A hash is generated (CRC32 based) to determine if the payload has changed, otherwise don't send it. An example payload looks roughly like:

`{"wWSelTemp":"60","selFlowTemp":"5.0","outdoorTemp":"?","wWActivated":"on","wWComfort":"Comfort","wWCurTmp":"46.0","wWCurFlow":"0.0","wWHeat":"on","curFlowTemp":"54.2","retTemp":"51.5","burnGas":"off","heatPmp":"off","fanWork":"off","ignWork":"off","wWCirc":"off","selBurnPow":"0","curBurnPow":"0","sysPress":"1.2","boilTemp":"56.7","pumpMod":"0","ServiceCode":"0H"}`

Similarly the thermostat values are sent as a json package under a topic named `home/ems-esp/thermostat_data` with the current mode, room temperature and set temperature like

`{"thermostat_currtemp":"19.8","thermostat_seltemp":"16.0","thermostat_mode":"manual"}`

### The Basic Shower Logic

Checking whether the shower is running is tricky. We know when the warm water is on and being heated but need to distinguish between the central heating, shower, hot tap and even a bath tap. So this code is a little experimental.

There is other logic in the code to compensate for water heating up to shower temperature and whether the shower is turned off and back on again quickly within a 10 second window.

## Home Assistant Configuration

Within Home Assistant it renders as:

![Home Assistant panel)](doc/home_assistant/ha.png)

and the alerts on an iOS/Android device using PushBullet, PushOver or any notification service would look like:

![Home Assistant iPhone notify)](doc/home_assistant/ha_notify.jpg)

You can find the .yaml configuration files under `doc/ha`. See also https://community.home-assistant.io/t/thermostat-and-boiler-controller-for-ems-based-boilers-nefit-buderus-bosch-using-esp/53382

## Building The Firmware

### Using PlatformIO Standalone

**On Windows:**

- Download [Git](https://git-scm.com/download/win) (install using the default settings)
- Download and install [Visual Studio Code](https://code.visualstudio.com/docs/?dv=win) (VSC). It's like 40MB so don't confuse with the commercial Microsoft Visual Studio.
- Restart the PC (if using Windows) to apply the new PATH settings. It should now detect Git
- Install the VSC extensions "PlatformIO IDE" then click reload to activate it
- Git clone this repo, eith using `git clone` from PlatformIO's terminal or the Git GUI interface
- Create a `platformio.ini` based on the `platformio.ini-example` making the necessary changes for your board type

**On Linux (e.g. Ubuntu under Windows 10):**

- make sure Python 2.7 is installed
- make sure you have a Linux distro installed (https://docs.microsoft.com/en-us/windows/wsl/install-win10)
- Then...
```python
% pip install -U platformio
% sudo platformio upgrade
% platformio platform update

% git clone https://github.com/proddy/EMS-ESP.git
% cd EMS-ESP
% cp platformio.ini-example platformio.ini
```
- edit `platformio.ini` to set `env_default` to your board type
```c
% platformio run -t upload
```

### Building Using Arduino IDE

Porting to the Arduino IDE can be a little tricky but it is possible.

- Add the ESP8266 boards (from Preferences add Additional Board URL `http://arduino.esp8266.com/stable/package_esp8266com_index.json`)
- Go to Boards Manager and install ESP8266 2.4.x platform
- Select your ESP8266 from Tools->Boards and the correct port with Tools->Port
- From the Library Manager install the needed libraries from platformio.ini
- Put all the files in a single sketch folder
- cross your fingers and hit CTRL-R to compile...

## Using the Pre-built Firmware

pre-baked firmwares for some ESP8266 devices are available in the directory `/firmware` which you can upload yourself using [esptool](https://github.com/espressif/esptool) bootloader. On Windows, follow these instructions:

1. Check if you have **python 2.7** installed. If not [download it](https://www.python.org/downloads/) and make sure you select the option to add Python to the windows PATH
2. Install the ESPTool by running `pip install esptool` from a command prompt
3. Connect the ESP via USB, figure out the COM port
4. run `esptool.py -p <com> write_flash 0x00000 <firmware>` where firmware is the `.bin` file and \<com\> is the COM port, e.g. `COM3`

The ESP8266 will start in Access Point (AP) mode, so connect via WiFi to the SSID **EMS-ESP** and telnet to 192.168.4.1. Then use the set command to configure your own network settings.

## Troubleshooting

If the WiFi, MQTT, MDNS or something else fails to connect, re-build the firmware using the `-DDEBUG_SUPPORT` option, connect the ESP8266 to a USB in your computer and monitor the Serial output. A lot of detailed logging will be printed to help you pinpoint the cause of the error.

The onboard LED will flash if there is no connection with the EMS bus. You can disable LED support by adding -DNO_LED to the build options.

## Known Issues

Some annoying issues that need fixing:

- Very infrequently an EMS write command is not sent, probably due to a collision somewhere in the UART between an incoming Rx and a Poll. The retries in the code fix this for now.

## Wish List

- Measure amount of gas in m3 per day for the hot water vs the central heating, and convert this into cost in Home Assistant
- Support changing temperatures on an Nefit Easy. To do this you must send XMPP messages directly to the thermostat. There is already a TCP stack and a Wifi and Telnet server running in the code, so the building blocks are there to extend with another XMPP client. Here are a number of Python based projects that show how to do this:
  - https://github.com/patvdleer/nefit-client-python
  - https://github.com/marconfus/ha-nefit
  - https://github.com/robertklep/nefit-easy-core
- Add support for a temperature sensor on the circuit (DS18B20)
- Improve detection of Heating Off without checking for selFlowTemp (selected flow temperature)
- Split MQTT into smaller chunks. Now the messages can be up to 600 bytes which may cause issues.

## Your Comments and Feedback

Any comments, suggestions or code contributions are very welcome. Please post a GitHub issue.

## DISCLAIMER

This code and libraries were developed from information gathered on the internet and many hours of reverse engineering the communications between the EMS bus and thermostats. It is **not** based on any official documentation or supported libraries from Buderus/Junkers/Nefit (and associated companies) and therefore there are no guarantees whatsoever regarding the safety of your devices and/or their settings, or the accuracy of the information provided.