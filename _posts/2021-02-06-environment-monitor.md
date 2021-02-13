---
layout: post
title: "IoT Home Environmental Alerting"
author: "Antti"
tags: Twilio
---

# IoT Home Environmental Alerting

Internet of Things (IoT) is an ever expanding field and one of the most common things to do is environmental alerting.  To do that, this tutorial uses the [Twilio Developer Kit for Broadband IoT](https://www.twilio.com/docs/iot/wireless/get-started-twilio-developer-kit-broadband-iot) to monitor for temperature, humidity, and liquid and sends an SMS alerting you to conditions that are outside your desired parameters. The developer kit allows you to be connected to the cellular network, allowing for monitoring at remote locations such as a cabin where internet access may not be readily available.  

If you are setting this up in a location where internet is available, data via a cellular network is not needed to complete setup.

## Requirements
To complete this tutorial you'll need the following
- [Twilio Developer Kit for Broadband IoT](https://www.twilio.com/docs/iot/wireless/get-started-twilio-developer-kit-broadband-iot) (Instructions on how to purchase are at this link)
- [Parallax 28090 Liquid Level Sensor](https://www.parallax.com/product/mini-liquid-level-sensor/)
- [Adafruit ADS1115 or ADS1015 ADC](https://cdn-learn.adafruit.com/downloads/pdf/adafruit-4-channel-adc-breakouts.pdf)
- Wire to connect sensor to ADC and ADC to LTE Cat 1 Pi HAT (included in Twilio Developer Kit for Broadband IoT)    
    - [Seed Studio 110990210](https://www.seeedstudio.com/Grove-4-pin-Male-Jumper-to-Grove-4-pin-Conversion-Cable-5-PCs-per-Pack.html) 
    - [Jumper Wires](https://www.digikey.ca/en/products/detail/adafruit-industries-llc/153/7241430)
- [Latest Raspberry Pi OS](https://www.raspberrypi.org/software/)
- python libraries
    - python venv 
    - pip3
    - [groove](https://github.com/Seeed-Studio/grove.py#installation)
    - [luma.core](https://pypi.org/project/luma.core/)
    - [luma.oled](https://pypi.org/project/luma.oled/)
    - [paho.mqtt](https://pypi.org/project/paho-mqtt/)
    - [adafruit-circuitpython-ads1x15](https://pypi.org/project/adafruit-circuitpython-ads1x15/)
    - [twilio](https://pypi.org/project/twilio/)
- (Optional) Breadboard

## Installing the latest Raspberry Pi OS
The Raspberry Pi Foundation has a Raspberry Pi Imager that can be used to get the latest Raspberry Pi OS and image it on the SD card.  You can dowload the imager [here](https://www.raspberrypi.org/software/).  Select the latest Raspberry Pi OS Lite (that has no desktop application).  We'll connect to it via SSH to do all of our work.  

![Imaging Card](../assets/image_card.gif)

## Enable SSH

We'll be connecting to our Raspberry Pi via SSH. We can enable SSH by adding a file to SD Card named `ssh` with no extension.  This file should be added to the boot partition of the SD card ([further instructions here](https://www.raspberrypi.org/documentation/remote-access/ssh/)).


##  Connect to Local Wi-Fi if Available
We also will want to connect the Raspberry Pi to Wi-Fi if available to reduce data costs on our Twilio SIM card.  

There are many ways of doing this but the "headless" way is to create a file named `wpa_supplicant.conf` in the boot directory updating the `country`, `ssid`, and `psk`. The Raspberry Pi foundation has more detailed instructions [here](https://www.raspberrypi.org/documentation/configuration/wireless/headless.md).

```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=<Insert 2 letter ISO 3166-1 country code here>

network={
 ssid="<Name of your wireless LAN>"
 psk="<Password for your wireless LAN>"
}
```
[source](https://www.raspberrypi.org/documentation/configuration/wireless/headless.md)

To confirm that you're connected to Wi-Fi, login to your router and confirm that a device named `raspberrypi` or equivalent. Note the IP Address and while you're at it setup a permanent IP Address to make connecting to your Raspberry Pi easier.

## Setting up SIM Card

Twilio has a [guide](https://www.twilio.com/docs/iot/wireless/how-to-order-and-register-your-first-sim) for registering your SIM card and selecting a rate plan. We will only be using data to send sms messages.

## Twilio Developer Kit Setup
Twilio has a [Quick Start Guide](https://www.twilio.com/docs/iot/wireless/get-started-twilio-developer-kit-broadband-iot) that walks through how to setup and get going with the developer kit that is worth.  Follow the [hardware setup section](https://www.twilio.com/docs/iot/wireless/get-started-twilio-developer-kit-broadband-iot#the-hardware), don't plug in the screen, and this guide will take it from there.


## Connecting the Liquid Level Sensor to the Raspberry Pi
If you connected the screen to the I2C port disconnect it now as that is where we will connect our liquid level sensor to. See the diagram and image below on how to wire it up. Also read [Adafruit's guide](https://cdn-learn.adafruit.com/downloads/pdf/adafruit-4-channel-adc-breakouts.pdf) for further details and warnings when wiring.

![Wiring Diagram](../assets/wiring_diagram.png)

![Wiring Image](../assets/wiring_image.jpg)

## Power up!

Once the above is complete we can power it up. Once powered up you should see some lights on the Raspberry Pi, the LTE Cat 1 Pi HAT, and the liquid level sensor.

## Installation of Libraries

For development we'll be installing the following libraries but first will run `sudo apt update` to update

- git (`sudo apt install git`)
- python 3 venv (`sudo apt-get install python3-venv`)
- python 3 pip (`sudo apt install python3-pip`)
- groove.py ([Install Instructions](https://github.com/Seeed-Studio/grove.py#installation))

Let's now make a directory called `environment_alert/` for our project and make it the working directory, create a virtual environment and activate it 

```
cd environment_alert/
python3 -m venv venv
source venv/bin/activate
```

Once activated install the following packages 

`pip install grove.py adafruit-circuitpython-ads1x15 luma.core luma.oled paho.mqtt twilio`


## Create Environment Alert Script

``` Python
import time
import sys
import subprocess
import json
import urllib
from grove.i2c import Bus
from twilio.rest import Client

import board
import busio
import adafruit_ads1x15.ads1115 as ADS
from adafruit_ads1x15.analog_in import AnalogIn


# Global Variables
UNITS = 'fahrenheit'  # Default temperature unit
DISPLAY_TYPE = 'temperature' # Show temperature
DAILY_MESSAGE = False

# Liquid Level Sensor

##############################################
# Grove Temperature / Humidity Sensor SHT35  #
##############################################
class GroveTemperatureHumiditySensorSHT3x(object):
    def __init__(self, address=0x45, bus=None):
        self.address = address
        self.bus = Bus(bus)

    def CRC(self, data):
        crc = 0xff
        for s in data:
            crc ^= s
            for i in range(8):
                if crc & 0x80:
                    crc <<= 1
                    crc ^= 0x131
                else:
                    crc <<= 1
        return crc

    def read(self):
        # High repeatability, clock stretching disabled
        self.bus.write_i2c_block_data(self.address, 0x24, [0x00])

        # Measurement duration < 16 ms
        time.sleep(0.016)

        # Read 6 bytes back:
        # Temp MSB, Temp LSB, Temp CRC,
        # Humididty MSB, Humidity LSB, Humidity CRC
        data = self.bus.read_i2c_block_data(0x45, 0x00, 6)
        temperature = data[0] * 256 + data[1]
        celsius = -45 + (175 * temperature / 65535.0)
        humidity = 100 * (data[3] * 256 + data[4]) / 65535.0
        if data[2] != self.CRC(data[:2]):
            raise RuntimeError("Temperature CRC mismatch")
        if data[5] != self.CRC(data[3:5]):
            raise RuntimeError("Humidity CRC mismatch")
        return celsius, humidity


##############################################
#     Parallax 28090 Liquid Level Sensor     #
##############################################
class ParallaxLiquidLevelSensor(object):

    def __init__(self, address=0x48):
        self.address = address
        self.i2c = busio.I2C(board.SCL, board.SDA)
        self.ads = ADS.ADS1115(self.i2c, address=0x48)
        self.chan = AnalogIn(self.ads, ADS.P0)

    def read(self):
        print('liquid value: ' + str(self.chan.value) + 'liquid voltage: ' + str(self.chan.voltage))
        return self.chan.value
    
    def water_detected(self):
        return self.chan.value > 3000


def read_sensors_and_display():
    try:
        sensor = GroveTemperatureHumiditySensorSHT3x()
        liquid_sensor = ParallaxLiquidLevelSensor()

        celsius_temperature, humidity = sensor.read()
        fahrenheit_temperature = (celsius_temperature * 1.8) + 32
        celsius_temperature = float("{0:.2f}".format(celsius_temperature))
        fahrenheit_temperature = float("{0:.2f}"
                                        .format(fahrenheit_temperature))
        humidity = float("{0:.2f}".format(humidity))

        # Format a temperature reading message to the cloud
        if UNITS == "fahrenheit":
            long_temp = ('Temperature: {:.2f} F'
                            .format(fahrenheit_temperature))
        else:
            long_temp = ('Temperature: {:.2f} C'
                            .format(celsius_temperature))

        # Format humidity
        long_humid = ('Relative Humidity: {:.2f}%'
                        .format(humidity))

        print(long_temp)
        print(long_humid)
        # liquid_sensor_msg = 'liquid sensor value: ' + str(chan.value) + '\nliquid sensor voltage: ' + str(chan.voltage)
        # print(liquid_sensor_msg)

        if (liquid_sensor.water_detected() or DAILY_MESSAGE):
            text_details(long_temp + '\n ' + long_humid + '\nWater found: ' + str(liquid_sensor.water_detected()) )
    except KeyboardInterrupt:
        print ("IoTHubClient sample stopped")

def text_details(details):
    account_sid = 'ACcaa9583ac20e99e1946de9579af43b39' 
    auth_token = '1582f36a05a93357acec0c608ba2612b' 
    client = Client(account_sid, auth_token) 
    message = client.messages.create( 
        from_='+17198385172',  
        body=details,      
        to='+13062276751' 
    ) 

if __name__ == '__main__':
    DAILY_MESSAGE = len(sys.argv) > 1 and sys.argv[1] == '1'
    print(DAILY_MESSAGE)
    read_sensors_and_display()



```

## Create bash file for crontab to use

``` Python
#!/bin/bash
source /home/pi/environment_monitor/venv/bin/activate
python /home/pi/environment_monitor/environment_monitor.py $1
```

## Setup crontab

crontab -e

```
0 17 * * * bash /home/pi/environment_alert.bash 1
1-59/2 * * * * bash /home/pi/environment_alert.bash
```