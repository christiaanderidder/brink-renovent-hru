# Brink Renovent Excellent Home Automation
This repository contains collection of information and configuration files needed for automating the Brink Renovent Excellent 400 heat recuperation unit (HRU / WTW).
The technniques here will likely work for other Brink devices as well.

**I am not resposible for any damage you might do to your device, nor will I provide any support besides the information given below**

The guide below was tested on the following devices, configured for [demand ventilation](manuals/demand-controlled-ventilation-installation-instructions.pdf)
- Brink Renovent Excellent 400 4/0 R
- Brink Zone Valve
- 3 Brink CO2 sensors
- Brink Air Control
- A RJ12 splitter
  - RJ11 3-way switch
  - RJ12 RF receiver and remote with 4 settings

Broadly speaking there are two methods that are commonly used to automate the system:
- The X1 (RJ12) connector used for 3/4-way physical switches and RF switches
- The X2 connector used for eBUS or OpenTherm communication

The [Brink Renovent Excellent installation guide](manuals/installation-manual-300-400.pdf) describes the wiring of these connections. Below I will focus on how to use them for automating your system.

You might be wondering which method you can use best. The answer is that *it depends*. For simple setups (filter indicator, fan speed) the X1 (RJ12) connector will be sufficient. If you are looking to automate all aspects of your HRU you will need an eBUS or OpenTherm integration.
In some cases (see [known issues](#known-issues)) you might need both.

## X1 (RJ12)
This is the simpler of the two, and allows for setting the fan mode to 0, 2 or 3 as well as indicating whether the filter needs to be replaced.
There are different sources describing how this can be wired up with a (smart) relay:
- [Dutch Tweakers forum (nl)](https://gathering.tweakers.net/forum/list_messages/1979992)
- [Lets control it forum (en)](https://www.letscontrolit.com/forum/viewtopic.php?t=5702#p49500)

I opted to using a board which combines most of the needed components (2 relays, a voltage regulator and an ESP12F) in [a single board](https://devices.esphome.io/devices/ESP-12F-Relay-X2).

This board does not come with any detailed instructions, but they are pretty straightforward:
1. Solder the included header pins into the through-holes
2. Use one of the included jumpers to connect IO0 and GND (marked blue in wiring schematic below)
3. Download the ESPHome configuration file in the [esphome](esphome) folder of this repository, adjust the settings (especially IP) according to your home setup.
3. Connect a development board to TXD, RXD, GND and 5V and flash ESPHome using the configuration file
4. Cut the power to the board, remove the jumper between IO2 and GND
5. Use two of the included jumpers to connect RY1 to IO5 and RY2 to IO4 (marked red in the wiring schematic below)
6. Connect power to the board and make sure ESPHome starts and can be reached on the configured IP.
7. Connect the board to the RJ12 connector as show in the diagram below.

![relay-schematic](https://github.com/user-attachments/assets/9feee3e0-4223-4b85-9936-f51c4b8538ad)

**This README will be updated with wiring diagrams and configuration files once I have this up and running**

## X2 (eBUS or OpenTherm)
The X2 connector can be used for either the eBUS or OpenTherm protocols. Which protocol you want to used is determined by a setting (08) on the HRU.

### Brink Home eModule
The easiest way to interact with the Brink HRU over eBUS would be to install a Brink Home eModule. This device connects the Brink HRU to a cloud-based service allowing you to control your device. A Home Assistant [integration](https://github.com/samuolis/brink) exists, but requires cloud connectivity.
In this guide we will focus on a fully local solution that does not require internet connectivity, therefore the Brink Home eModule will not be regarded.

### OpenTherm
While there are some efforts to control the device using OpenTherm ([here](https://github.com/raf1000/brink_openhab) and [here](https://github.com/tijsverkoyen/Home-Assistant-BrinkRenoventHR)), this seems to be a bit of a dead end when the HRU system has other brink products (e.g. Zone Valve, Air Control, Brink Home eModule) which all rely on eBUS. Therefore this guide will focus only on the eBUS based integration.

### Custom eBUS integration
The eBUS protocol is a little more complicated, but can easily be read and written to with the right tools. This protocol allows for automating almost any setting and sensor the HRU offers.
To make this work you need a way to connect to the bus. In my cases I got myself an [eBUS Adapter Shield](https://adapter.ebusd.eu/v5/index.en.html). This board can easily be connected directly to the HRU's eBUS connector, or by wiring it in parallel to something that's already plugged into it.

This board does some of the low level parsing of the eBUS messages and exposes them over the network. 
The board and a case can be ordered directly from Elecrow.
- [Board](https://www.elecrow.com/ebus-adapter-shield-v5.html)
- [Case](https://www.elecrow.com/enclosure-for-ebus-adapter-shield-v5.html)

The [getting started](https://adapter.ebusd.eu/v5/steps.en.html) guide on the eBUSd adapter site provide step by step instructions on how to configure the board.

### eBUSd
To parse the messages from eBUS and forward them to your home automation software, you will need to configure and run an instance of [eBUSd](https://ebusd.eu/).  I chose to run eBUSd in docker, using [the configuration](ebusd/docker-compose.yaml) in this repository. More information on how eBUSd configurations work can be found

While eBUSd is an open standard, the messages sent are often proprietary. This means that specific message parsing configuration is needed for your device. The configuration provided in the [eBUSd configuration repo](https://github.com/john30/ebusd-configuration) only applies to heating systems and does not have any Brink HRUs listed.

However, different people managed to reverse engineer the messages used by Brink, the following sources helped me a lot:
- [pvyleta](https://github.com/pvyleta/ebusd-brink-hru) who decompiled the .NET-based Brink Service tool to extract the different eBUS messages
- [tinus5](https://gathering.tweakers.net/forum/list_message/63666318#63666318) who provided the configuration for the Brink zone-valve and CO2 sensors

Based on the above, and some of my own reverse engineering ([here](https://github.com/pvyleta/ebusd-brink-hru/issues/3#issuecomment-2212419678) and [here](https://github.com/pvyleta/ebusd-brink-hru/issues/5#issuecomment-2221160210)), I created my own configuration files, which can be found under the [ebusd config folder](ebusd/config) in this repository.

The eBUSd wiki gives more information on the eBUSd [message definition](https://github.com/john30/ebusd/wiki/4.1.-Message-definition) and [how to create](https://github.com/john30/ebusd/wiki/HowTos) configuration files.

## Home Assistant
If MQTT is correctly configured in home assistant all entities will automatically be picked up and can be used on dashboards accordingly. 

After seeing the work people did for other HRU systems ([here](https://github.com/mweimerskirch/lovelace-comfoair) and [here](https://github.com/mweimerskirch/lovelace-hacomfoairmqtt)), I decided to create a custom card for to display the most important information in a single place: [Brink Renovent HRU card](https://github.com/christiaanderidder/lovelace-brink-renovent-hru-card/)

## Known Issues

### Fan speed 
Setting the fan speed (FanMode) using eBUS is known to cause issues for multiple people (e.g. [here](https://github.com/dstrigl/ebusd-config-brink-renovent-excellent-300/issues/7) and [here](https://github.com/pvyleta/ebusd-brink-hru/issues/2)).

For me, this was caused by the Air Control panel sending continuous fan speed updates to the HRU, which will reset the speed to the value Air Control wants it to be. There are a couple of ways to address this problem:

#### Disconnecting Air Control
Disconnecting the Air Control panel from the eBUS will stop the update messages and allow you to set the fan speed. However, doing this will lose any functionality that Air Control provides, including demand based (CO2 sensor based) ventilation.

#### Sending periodic updates
Making Home Assitant or other automation software send periodic fan speed updates just like Air Control does. This seems to work for some people ([here](https://github.com/dstrigl/ebusd-config-brink-renovent-excellent-300/issues/7#issuecomment-1336465329) and [here](https://github.com/pvyleta/ebusd-brink-hru/issues/2#issuecomment-2135583648)).

#### Combining RJ12 and eBUS
In my case neither solution works.
I can't disconnect Air Control, because I want to keep demand (CO2 sensor based) ventilation.

When using demand ventilation, sending periodic updates won't work because the Air Control panel sends a [special command](https://github.com/christiaanderidder/brink-renovent-hru/blob/9aec5e7ed4ab0a13d1d2ccd2a620caf2364d0f61/ebusd/config/7c.Excellent400.csv#L32). This command resets the fan speed back to Auto when trying to override it using eBUS. This happens so quickly that even periodic updates every second are not enough to keep the fan speed.

However, overriding the fan speed using the 3/4-way switch and RF remote do work because these directly override the HRU which then ignores the commands over eBUS.

For this reason I had to use a combination of the RJ12 method (for overriding the fan speed) and eBUS (for all other information coming from the HRU).
