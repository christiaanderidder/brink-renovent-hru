# Brink Renovent Excellent Home Automation
This repository contains collection of information and configuration files needed for automating the Brink Renovent Excellent 400 heat recuperation unit (HRU / WTW)
The technniques here will likely work for other Brink devices as well.

**I am not resposible for any damage you might do to your device, nor will I provide any support besides the information given below**

The guide below was tested on the following devices, configured for [demand ventilation](manuals/demand-controlled-ventilation-installation-instructions.pdf)
- Brink Renovent Excellent 400 4/0 R
- Brink Zone Valve
- 3 Brink CO2 sensors
- Brink Air Control

Broadly speaking there are two methods that are commonly used to automate the system:
- The RJ12 connector used for 3/4-way physical switches and RF switches
- The eBUS connector used for more complex setups

The [Brink Renovent Excellent installation guide](manuals/installation-manual-300-400.pdf) describes the wiring of these connections. Below I will focus on how to use them for automating your system.

You might be wondering which method you can use best. The answer is that *it depends*. For simple setups (filter indicator, fan speed) the RJ12 connector will be sufficient. If you are looking to automate all aspects of your HRU you will need the eBUS integration. In some cases (see [known issues](#known-issues)) you might need both.

## RJ12
This is the simpler of the two, and allows for setting the fan mode to 0, 2 or 3 as well as indicating whether the filter needs to be replaced.
There are different sources describing how this can be wired up with a (smart) relay:
- [Dutch Tweakers forum (nl)](https://gathering.tweakers.net/forum/list_messages/1979992)
- [Lets control it forum (en)](https://www.letscontrolit.com/forum/viewtopic.php?t=5702#p49500)

I opted to using a board which combines most of the needed components (2 relays, a voltage regulator and an ESP12F) in [a single board](https://templates.blakadder.com/ESP12F_Relay_X2.html)

**This README will be updated with wiring diagrams and configuration files once I have this up and running**

## eBUS
The eBUS protocol is a little more complicted, but can easily be read and written to with the right tools. This protocol allows for automating almost any setting and sensor the HRU offers.
To make this work you need a way to connect to the bus. In my cases I got myself an [eBUS Adapter Shield](https://adapter.ebusd.eu/v5/index.en.html). This board runs of a micro USB charger and can easily be connected directly to the HRU's eBUS connector, or by wiring it in parallel to something that's already plugged into it.

This board does some of the low level parsing of the eBUS messages and exposes them over the network. 
The board and a case can be ordered directly from Elecrow.
- [Board](https://www.elecrow.com/ebus-adapter-shield-v5.html)
- [Case](https://www.elecrow.com/enclosure-for-ebus-adapter-shield-v5.html)

## Configuring eBUSd
To parse the messages from eBUS and forward them to your home automation software, you will need to configure and run an instance of [eBUSd](https://ebusd.eu/).  I chose to run eBUSd in docker, using [the configuration](ebusd/docker-compose.yaml) in this repository.
While eBUSd is an open standard, the messages sent are often proprietary. This means that specific message parsing configuration is needed for your device. The configuration provided in the [eBUSd configuration repo](https://github.com/john30/ebusd-configuration) only applies to heating systems and does not have any Brink HRUs listed.

However, different people managed to reverse engineer the messages used by Brink, the following sources helped me a lot:
- [pvyleta](https://github.com/pvyleta/ebusd-brink-hru) who decompiled the .NET-based Brink Service tool to extract the different eBUS messages
- [tinus5](https://gathering.tweakers.net/forum/list_message/63666318#63666318) who provided the configuration for the Brink zone-valve and CO2 sensors

Based on the above and [some of my own reverse engineering](https://github.com/pvyleta/ebusd-brink-hru/issues/5), I created my own configuration files, which can be found under the [ebusd config folder](ebusd/config) in this repository.

## Known issues
