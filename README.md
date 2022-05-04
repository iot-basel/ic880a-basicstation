# The Things Network: iC880a based Semtech LoRa Basics™ Station
This is a reference setup for [The Things Network](https://www.thethingsnetwork.org/) gateways based on the IMST iC880a SPI concentrator with a Raspberry Pi as host platform. As LoRaWAB gateway software the official Semtech [Basic Station](https://github.com/lorabasics/basicstation) implementation is used. The setup works with both the The Things Network (TTN) and The Things Stack (TTS) Private Server at the current version 3 of the Network Server implementation, informally known as V3.

## Basics
[The Things Stack (TTS)](https://github.com/TheThingsNetwork/lorawan-stack) is an open-source LoRaWAN Network Server stack. The Things Network (TTN) is a global collaborative Internet of Things ecosystem that creates networks, devices and solutions using LoRaWAN. TTN runs nothing else than the TTS Community Edition

## Setup
If you want to assembly the hardware yourself, go to the best howto from our friends at [TTN-Zürich](https://github.com/ttn-zh/ic880a-gateway/wiki). Stop before "Setting up the software" and come back here.
Alternativly, you can use an IMST LoRa® Lite Gateway, which is nothing else than a iC880a concentator board preassembled on a Raspberry Pi.

**Warning**: never power up your hardware without the antenna attached!

### Installation
Install a functional operating system on your RPi with network (and ideally SSH) enabled. We prefer to use Raspberry Pi OS Lite without a desktop, which you can download directly within the [Raspberry Pi Imager](https://www.raspberrypi.com/software/) when flashing your SD card.

<img src="https://github.com/iot-basel/ic880a-basicstation/blob/main/images/pi-imager.jpg?raw=true" alt="PI Imager" width="400"/>

Before flashing, make sure to modify your settings in order to activate SSH. You can also set the hostname, username and password. Note: Since the Raspberry Pi OS Bullsey April 2022 update, the account creation process no longer features a default username "pi" due to security reasons. So best thing to do is defining your own user befor flashing your SD card.

Another note: Using the WiFi for network connection is not recommended due to noise interference with the LoRaWAN. On the other hand, if you choose WiFi instead of Ethernet, then try to use a dongle with external antenna and move the antenna outside the enclosure to have less noise inside the box.

<img src="https://github.com/iot-basel/ic880a-basicstation/blob/main/images/pi-imager-settings.jpg?raw=true" alt="PI Imager settings" width="400"/>

Insert the newly flashed SD card into your RPi, make sure the LoRa antenna is attached and plug the power supply. This will also power the concentrator board, so make sure your power supply can draw at least 2.5 A.

From a computer in the same LAN, ssh into the RPi using either your defined hostname or the assigend IP address:
```
$ ssh [username]@[hostname]
```

Use `raspi-config` utility to enable SPI (`[5] Interfacing options -> P4 SPI`), because it normally switched off by default:
```
$ sudo raspi-config
```

Make sure you have an updated installation and install git
```
$ sudo apt update
$ sudo apt upgrade
$ sudo apt install git
```

Now clone the latest version of the Basics Station and build the executable on your target:
```
git clone https://github.com/lorabasics/basicstation.git
cd basicstation
make platform=rpi variant=std
```
This will take a while. During the build process, dependencies such as [mbed TLS](https://github.com/ARMmbed/mbedtls/) and the libloragw [SX1301](https://github.com/Lora-net/lora_gateway)/[SX1302](https://github.com/Lora-net/sx1302_hal) Driver/HAL are downloaded and compiled.

## Configuration

[Offical LoRa Basics™ Station documentation](https://lora-developers.semtech.com/build/software/lora-basics/lora-basics-for-gateways/)
