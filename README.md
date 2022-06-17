# The Things Network: iC880a based Semtech LoRa Basics™ Station
This is a reference setup for [The Things Network](https://www.thethingsnetwork.org/) gateways based on the IMST iC880a SPI concentrator with a Raspberry Pi as host platform. As LoRaWAN gateway software the official Semtech [Basic Station](https://github.com/lorabasics/basicstation) implementation is used. The setup works with both the The Things Network (TTN) and The Things Stack (TTS) Private Server at the current version 3 of the Network Server implementation, informally known as V3.

## Basics
[The Things Stack (TTS)](https://github.com/TheThingsNetwork/lorawan-stack) is an open-source LoRaWAN Network Server stack. The Things Network (TTN) is a global collaborative Internet of Things ecosystem that creates networks, devices and solutions using LoRaWAN. TTN runs nothing else than the TTS Community Edition

## Setup
If you want to assemble the hardware yourself, go to the best howto from our friends at [TTN-Zürich](https://github.com/ttn-zh/ic880a-gateway/wiki). Stop before "Setting up the software" and come back here.
Alternatively, you can use an IMST LoRa® Lite Gateway, which is nothing else than a iC880a concentrator board pre assembled on a Raspberry Pi.

**Warning**: never power up your hardware without the antenna attached!

## Installation
Install a functional operating system on your RPi with network (and ideally SSH) enabled. We prefer to use Raspberry Pi OS Lite without a desktop, which you can download directly within the [Raspberry Pi Imager](https://www.raspberrypi.com/software/) when flashing your SD card.

<img src="https://github.com/iot-basel/ic880a-basicstation/blob/main/images/pi-imager.jpg?raw=true" alt="PI Imager" width="400"/>

Before flashing, make sure to modify your settings in order to activate SSH. You can also set the hostname, username and password. Note: Since the Raspberry Pi OS Bullsey April 2022 update, the account creation process no longer features a default username "pi" due to security reasons. So the best thing to do is to define your own user before flashing your SD card.

Another note: Using the WiFi for network connection is not recommended due to noise interference with the LoRaWAN. On the other hand, if you choose WiFi instead of Ethernet, then try to use a dongle with an external antenna and move the antenna outside the enclosure to have less noise inside the box.

<img src="https://github.com/iot-basel/ic880a-basicstation/blob/main/images/pi-imager-settings.jpg?raw=true" alt="PI Imager settings" width="400"/>

Insert the newly flashed SD card into your RPi, make sure the LoRa antenna is attached and plug the power supply. This will also power the concentrator board, so make sure your power supply can draw at least 2.5 A.

From a computer in the same LAN, ssh into the RPi using either your defined hostname or the assigned IP address:
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

Next, clone the latest version of the Basics Station and build the executable on your target:
```
$ git clone https://github.com/lorabasics/basicstation.git
$ cd basicstation
$ make platform=rpi variant=std
```
This will take a while. During the build process, dependencies such as [mbed TLS](https://github.com/ARMmbed/mbedtls/) and the libloragw [SX1301](https://github.com/Lora-net/lora_gateway)/[SX1302](https://github.com/Lora-net/sx1302_hal) Driver/HAL are downloaded and compiled.

## Testing the installation
You will find some examples on how to use the basic station in the examples folder. Assuming SPI is enabled and wired correctly, you can test the station by passing the LoRaWAN concentrator (SPI device) as an environment variable using `RADIODEV`:
```
$ cd examples/live-s2.sm.tc
$ RADIODEV=/dev/spidev0.0 ../../build-rpi-std/bin/station
```
Note: We will later add the SPI definition to the config.
The example configuration connects to a public test server s2.sm.tc through which the basic station fetches all required credentials and a channel plan matching the region as determined from the IP address of the gateway. Provided there are active LoRa devices in proximity, received LoRa frames are printed in the log output on `stderr`.

If you see an error like `Concentrator start failed: lgw_start`, this is likely due to the missing reset of the SX1301 (see next chapter).

## Concentrator reset
The SX1301 digital baseband chip on the iC880a concentrator board should be reset after power-up. However, this initial reset of the SX1301 is not performed by the basic station. We have to do this manually by controlling the correct GPIO pin: GPIO 5 for the IMST Light Gateway or GPIO 25 (corresponds to RPi Pin 22) when using the DIY wiki of TTN ZH)

We want to keep the configuration files as well as the reset script together, so lets create a new folder inside the user directory.
```
$ cd ~
$ mkdir TTN
$ cd TTN
```

Next, create a reset_concentrator.sh file:
```
$ nano reset_concentrator.sh
```
and populate it with the following commands:
```
#! /bin/bash

SX1301_RESET_PIN=5
echo "$SX1301_RESET_PIN"  > /sys/class/gpio/export
sleep 1
echo "out" > /sys/class/gpio/gpio$SX1301_RESET_PIN/direction
echo "0"   > /sys/class/gpio/gpio$SX1301_RESET_PIN/value
sleep 0.5
echo "1"   > /sys/class/gpio/gpio$SX1301_RESET_PIN/value
sleep 1
echo "0"   > /sys/class/gpio/gpio$SX1301_RESET_PIN/value
sleep 0.5
echo "$SX1301_RESET_PIN"  > /sys/class/gpio/unexport
echo "Done\n"
```
Then, press Ctrl+X to exit, press Y to save and Enter.
Note: replace the number 5 on the 3rd line with 25 when using the DIY version instead of the IMST Light Gateway.

Make this file executable by changing its permission:
```
$ chmod +x reset_concentrator.sh
```


## Configuration
Create the station configuration file station.conf 
```
$ nano station.conf
```

and populate it with:
```
{
   "radio_conf": {                  /* Actual channel plan is controlled by the server */
       "lorawan_public": true,      /* is default */
       "clksrc": 1,                 /* radio_1 provides clock to concentrator */
       /* path to the SPI device, un-comment if not specified on the command line e.g., RADIODEV=/dev/spidev0.0 */
       "device": "/dev/spidev0.0",  /* default SPI device is platform specific */
       "pps": true,
       "radio_0": {
           /* freq/enable provided by LNS - only hardware-specific settings are listed here */
           "type": "SX1257",
           "rssi_offset": -166.0,
           "tx_enable": true,
           "antenna_gain": 0
       },
       "radio_1": {
           "type": "SX1257",
           "rssi_offset": -166.0,
           "tx_enable": false
       }
       /* chan_multiSF_X, chan_Lora_std, chan_FSK provided by LNS */
   },
   "station_conf": {
     "RADIO_INIT_WAIT": "2s",
     "radio_init": "/home/[username]/TTN/reset_concentrator.sh",
     "log_file":    "stderr",
     "log_level":   "DEBUG",
     "log_size":    10e6,
     "log_rotate":  3
   }
}
```
Then, press Ctrl+X to exit, press Y to save and Enter. Maker sure to replace ```[username]``` with your actual username

Further details of the configuration file can be found in the official [Semtech LoRa Basics™ Station documentation](https://lora-developers.semtech.com/build/software/lora-basics/lora-basics-for-gateways/?url=conf.html).

The Basics Station software includes two sub-protocols for connecting the gateway to the network server, the LoRaWAN Network Server (LNS), and the Configuration and Update Server (CUPS) protocol.

The LNS protocol establishes a data connection between the Basics Station compatible gateway and the web server. LoRa upstream and downstream messages are exchanged over a data connection via secure WebSockets.

The CUPS protocol enables credential management, as well as remote configuration of gateways and firmware updates. 

In this reference setup we will use the LNS method. Create a file called tc.uri 
```
$ nano tc.uri
```
and populate it with the server name. Packet transport with the The Things Network V3 LNS happens on port 8887.
```
wss://{eu1|nam1|au1}.cloud.thethings.network:8887
```
press Ctrl+X to exit, press Y to save and Enter. Note: Choose your closest geographically located server to minimize network latency (i.e. eu1 for Europe).

Next, we need to establish a trust relationship with the TTN network server. TTN uses the Let's Encrypt ISRG Root X1 Trust (expires June 2035). Download this root certificate and save it as tc.trust file:
```
$ curl https://letsencrypt.org/certs/isrgrootx1.pem.txt -o tc.trust
```
Note: You may also use a self signed certificate, if you need to connect to your private TTS network server (more information [here](https://www.thethingsindustries.com/docs/reference/root-certificates/)).

Last, we also need to create a tc.key file to authorise the gateway, but more on this later, since first we need to register the gateway.

## Register gateway
To register your gateway with TTN, you will need the gateway’s EUI, which is made up from the Raspberry Pi’s MAC address. It is shown in the first couple of lines when starting the station, i.e.
```
2022-05-06 08:15:40.039 [SYS:INFO] Station EUI : b827:ebff:fe5d:51f8
```
Log into the TTN [console](https://console.cloud.thethings.network/) and click "Go to gateways"

Then click "Add Gateway". The following dialog should appear:

<img src="https://github.com/iot-basel/ic880a-basicstation/blob/main/images/add-gateway.jpg?raw=true" alt="Add Gateway"/>

Enter your Gateway EUI into the Gateway EUI field, give your gateway a unique Gateway ID and a suitable name. Make sure the correct Gateway Server address is set (i.e. eu1.cloud.thethigns.network for Europe). Select the correct frequency plan for your country or location (i.e. Europe 863-870 MHz (SF9 for RX2 - recommended)) and click "Create gateway". You should now have an enrolled gateway.

The important thing now is to create the LNS API key in order to authorise your gateway. Click API keys and then + Add API key (in the top right hand corner).

<img src="https://github.com/iot-basel/ic880a-basicstation/blob/main/images/gateway-overview.jpg?raw=true" alt="Gateway Dashboard"/>

Give your API key a name (e.g. lns-key) and change the rights to grant individual rights to "link as Gateway to a Gateway Server for traffic exchange, i.e. write uplink and read downlink" as per the example below. Then click Create API key to create it.

<img src="https://github.com/iot-basel/ic880a-basicstation/blob/main/images/add-api-key.jpg?raw=true" alt="Add API LNS Key" width="600"/>

You will be given a one time opportunity to copy the key. Click the copy button. 

<img src="https://github.com/iot-basel/ic880a-basicstation/blob/main/images/copy-api-key.jpg?raw=true" alt="Copy API LNS Key" width="600"/>

You can now create our tc.key file back in the RPi console
```
$ nano tc.key
```
and past your key right after the following text:
```
Authorization: Bearer NNSXS.XXXXXX....
```
where NNSXS.XXXXX.... is the copied API key. Press Ctrl+X to exit, press Y to save and Enter.

There are some situations where this tc.key file cannot be read due to wrong line endings. If so, you can also create the file by the following cammands:
```
export LNS_KEY="xxxx"
echo "Authorization: Bearer $LNS_KEY" | perl -p -e 's/\r\n|\n|\r/\r\n/g' > tc.key
```
where xxxx is the string copied when creating the lns-key in the create gateway process.

Start/restart your basic station inside your TTN folder and your gateway should successfully register:
```
$ ~/basicstation/build-rpi-std/bin/station
```
Note: the Basic Station automatically takes the config files inside the folder, where you start the station. It will also add the files tc-bak.done, tc-bak.key, tc-bak.trust and tc-bak.uri once connected

More information on the Basic Station can be found in the [Offical LoRa Basics™ Station documentation](https://lora-developers.semtech.com/build/software/lora-basics/lora-basics-for-gateways/) from Semtech.

## Create System Service for automatic boot (optional)
In case you didn't reflash your SD card and still have Semtech's UDP packet forwarder on your system, make sure the old ttn-gatway service is disbaled or deinstall the packet forwarder.

Go to /lib/systemd/system directory and create the service file
```
$ cd /lib/systemd/system
$ sudo nano basicstation.service
```
with the following content:
```
[Unit]
Description=Basic Sation TTN V3 service

[Service]
WorkingDirectory=/home/[username]/basicstation/bin
ExecStart=/home/[username]/basicstation/build-rpi-std/bin/statio -h /home/[username]/TTN
SyslogIdentifier=ttn-gateway
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
again, press Ctrl+X to exit, press Y to save and Enter.
Note: Make sure to replace [username] with your actual username created when flashing your SD card.

Enable the basicstation service
```
$ sudo systemctl enable basicstation.service
```

Make sure you have stopped the basic station started in the chapter before and then start it via the service:
```
$ sudo systemctl start basicstation.service
```

If you want to check the status of your service, use the following command:
```
$ systemctl status basicstation.service
```
You should see some green output saying `active (running)`
