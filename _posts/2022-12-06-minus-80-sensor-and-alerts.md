---
layout: post
title: "Monitor -80 freezer temperatures and send email alerts with a Raspberry Pi"
description:
author: Cedar Warman
date: 2022-12-06
hero_image: /img/blog/2022-12-06_cover.jpg
image: /img/blog/2022-12-06_cover.jpg
tags: raspberry-pi sensors thermocouple minus-80-freezer led-matrix
canonical_url: https://www.cedarwarman.com/2022/12/06/minus-80-sensor-and-alerts.html
---

Do you trust your -80 freezer? I got tired of wondering when mine would fail, so I built a temperature sensor that emails me when there’s trouble. I made an R Shiny app to go along with it for easy visualization of freezer temperatures from any device. Here’s how!

## Part list.
<div class="container is-max-desktop has-text-centered">
<table class="table is-hoverable">
<thead>
<tr>
<th>Part</th>
<th>Cost</th>
<th>Note</th>
</tr>
</thead>
<tbody>
	<tr>
		<td><a href ="https://www.adafruit.com/product/3708">Raspberry Pi Zero W</a></td>
		<td>$14.00</td>
		<td>With pre-soldered header</td>
	</tr>
	<tr>
		<td><a href ="https://evosensors.com/products/t1x-wbwx-24g-ex-0-25-pfxx-40-stwl">Thermocouple</a></td>
		<td>$11.50</td>
		<td>24-gauge Type T</td>
	</tr>
	<tr>
		<td><a href ="https://www.adafruit.com/product/3263">Adafruit MAX31856</a></td>
		<td>$17.50</td>
		<td>Thermocouple amplifier breakout board</td>
	</tr>
	<tr>
		<td><a href ="https://www.amazon.com/gp/product/B07FFV537V/">MAX7219 LED matrix display</a></td>
		<td>$9.00</td>
		<td>HiLetgo, many similar options</td>
	</tr>
	<tr>
		<td><a href ="https://www.amazon.com/Charger-X-Universal-Adapter-Samsung/dp/B0794WT57Y/">USB wall charger</a></td>
		<td>$3.50</td>
		<td></td>
	</tr>
	<tr>
		<td><a href ="https://www.amazon.com/10ft3Pack-Charging-Smartphone-Connection-Blackwhite/dp/B06XYH75NQ/">USB cable</a></td>
		<td>$3.30</td>
		<td></td>
	</tr>
	<tr>
		<td><a href ="https://www.amazon.com/PNY-Elite-microSDHC-Memory-3-Pack/dp/B07YXJM282/">Micro SD card</a></td>
		<td>$6.00</td>
		<td></td>
	</tr>
	<tr>
		<td><a href ="https://www.adafruit.com/product/3471">Case</a></td>
		<td>$6.00</td>
		<td>Optional</td>
	</tr>
	<tr>
		<td><strong>Total</strong></td>
		<td><strong>$70.80</strong></td>
		<td></td>
	</tr>
</tbody>
</table>
m
</div>

## Thermocouple and display
At the heart of this system is a Type T thermocouple driven by an Adafruit MAX31856 thermocouple amplifier breakout board. Thermocouples come in lots of different types, but a Type T is commonly used for this application because of its ability to measure low temperatures and resist moisture. For this project, I used an SLE Type T, which should have an accuracy of 0.5°C or 0.4%, whichever is greater. This is a little more accurate than the more common Type K, but a Type K would also work fine. The Adafruit MAX31856 breakout board can drive several types of thermocouples; if you go with a Type K, you can use the slightly cheaper MAX31855K breakout board.

## Preparing the Raspberry Pi
I used a Raspberry Pi Zero W for this build, but any of the Raspberry Pi single-board computers should work fine. We’ll run it headless, so set up the Pi by <a href ="https://www.raspberrypi.com/documentation/computers/getting-started.html">installing Raspberry Pi OS</a> on an SD card, enabling ssh, and configuring your network. I wrote about these steps in more detail, including connecting to the troublesome WiFi at University of Arizona, in <a href ="https://www.cedarwarman.com/2021/08/09/connecting-to-campus-wifi.html">this blog post</a>.

## Connecting the thermocouple and display

<div class="container is-max-desktop">
    <div class="columns">
        <div class="column is-12">
            <img src="/img/blog/2022-12-06_device.jpeg">
        </div>
    </div>
</div>

### SPI
Both the thermocouple and the display use the Serial Peripheral Interface (SPI) to connect to the Raspberry Pi. You can read more about SPI <a href ="https://learn.sparkfun.com/tutorials/serial-peripheral-interface-spi/all">here</a>. By default, Raspberry Pi supports two SPI devices (more can be enabled with software). The devices share most of the SPI pins, but have one chip select (CS) pin per device. This pin allows the Pi to designate which device is sending/receiving data. Fortunately all the specifics of the SPI protocol are handled for us by the Pi’s software. To use SPI devices, you must enable the SPI0 interface using the system config. Start by entering this command:

```bash
sudo raspi-config
```

Next, navigate through the config menus to `Interfacing Options` and then `SPI`. Click `Yes` when asked to enable the SPI interface, then reboot the Pi. 

### Wiring

<div class="container is-max-desktop">
    <div class="columns">
        <div class="column is-12">
            <img src="/img/blog/2022-12-06_thermocouple_pinout.jpg">
        </div>
    </div>
</div>

Writing two SPI devices gets a little convoluted. Two of the <a href ="https://pinout.xyz/pinout/spi">Raspberry Pi GPIO pins</a> are shared by the two devices (SPI clock and peripheral in). For these two pins, I made “Y” wires by soldering two wires to a single wire. The two ends go to the two devices, while the single end goes to the Raspberry Pi. The rest of the wires are not shared. The thermocouple breakout board is run at 3.3v and the LED matrix is run at 5v. The two devices can be wired to two separate ground pins on the GPIO. The two SPI chip select (CS) pins go to <a href ="https://pinout.xyz/pinout/spi">GPIO 8 and 7</a> by default. However, for whatever reason I was not able to get the thermocouple breakout board to run on any pin but the default adafruit library pin GPIO 5, even after changing the designated pin in the software with `cs = digitalio.DigitalInOut(board.D5)`. GPIO 5 seems to work fine though, so I left it. Finally, the thermocouple board has an additional SPI peripheral out pin because it sends data back to the Raspberry Pi (the LED matrix only receives data).

<div class="container is-max-desktop">
    <div class="columns">
        <div class="column is-8">
            <img src="/img/blog/2022-12-06_thermocouple_pinout_detail.jpg">
        </div>
    </div>
</div>

## Software
### <a href ="https://github.com/cedarwarman/Raspberry_Pi_freezer_sensor_blog_post">Here is a repo with all the code used in this project</a>
### Library descriptions and installation
First, the <a href ="https://github.com/adafruit/Adafruit_CircuitPython_MAX31856">Adafruit CircuitPython MAX31856</a> library drives the thermocouple board. This library is well maintained and easy to use. To install the library:

```bash
pip3 install adafruit-circuitpython-max31856
```

I used <a href ="https://github.com/rm-hull/luma.led_matrix">Luma.LED_Matrix</a> to control the display. This library can control several types of common LED displays and has functionality way beyond what I use here. Installation includes some dependencies and the authors recommend updating pip on the Raspberry Pi:

```bash
sudo usermod -a -G spi,gpio pi
sudo apt install build-essential python3-dev python3-pip libfreetype6-dev libjpeg-dev libopenjp2-7 libtiff5
sudo -H pip install --upgrade --ignore-installed pip setuptools
sudo python3 -m pip install --upgrade luma.led_matrix
```

I used <a href ="https://github.com/burnash/gspread">gspread</a> to upload the data to Google Sheets. Make sure you set up a Google service account and add the API key to your Raspberry Pi as described in a <a href ="https://www.cedarwarman.com/2022/03/08/raspberry-pi-dht22-sensor.html">previous post</a> and in the <a href ="https://docs.gspread.org/en/latest/oauth2.html">gspread documentation</a>. I’ve written the script to read Google sheet URLs located in the `url` directory of the repo, you can see an example there. To install gspread:

```bash
pip install gspread
```

Finally, the freezer alarm system uses <a href ="https://gspread-pandas.readthedocs.io/en/latest/index.html">gspread_pandas</a> to read the sensor data from Google Sheets and <a href ="https://github.com/kootenpv/yagmail">yagmail</a> to send the email alerts. Both gspread_pandas and yagmail need API keys to work with Google, so maybe sure to read their documentation and set them up as the authors recommend.

### Code overview
There are two main scripts that make up the system. The first script, `read_freezer_thermocouple.py`, controls the thermocouple and display. The script reads the thermocouple, updates the display to show the reading, then uploads the reading to a Google sheet. The script updates the displayed temperature every three seconds, but only uploads data to the Google sheet every three minutes. I set up the script to run automatically when the Raspberry Pi is turned on using `systemd`. I’ve written more about `systemd` in <a href ="https://www.cedarwarman.com/2022/03/08/raspberry-pi-dht22-sensor.html">this post</a>, but in short, to set up the service run these commands:

```bash
sudo cp /home/pi/git/Raspberry_Pi_freezer_sensor_blog_post/service/freezer.service /etc/systemd/system/
sudo systemctl enable freezer.service
```

The second script, `freezer_alarm.py` watches the Google sheet containing the freezer temperature readings. I run this script from a separate computer in case I lose power to the freezer, which is on the same circuit as the Raspberry Pi that monitors it. This script can be run from anywhere with an internet connection because it uses the Google sheet created by `read_freezer_thermocouple.py`. The script downloads the sheet and calculates the average temperature over the most recent 10 minute interval. If the temperature is above -65 ºC and it has been more than 30 minutes since the last alarm, the script emails an alarm to every email address on a second Google sheet. This allows alarm recipients to be easily edited by anyone in the lab. The script will also send out a separate alarm email if no new data has been sent to the Google sheet in the past 30 minutes, indicating a sensor failure or power outage. I run the script every 2 minutes with cron using the following crontab:

```bash
*/2 * * * * python /home/cedar/git/shiny_freezer/python/freezer_alarm.py >> ~/.cron_log.txt 2>&1
```

An optional third script, `trim_google_sheets.py`, keeps the Google sheet at a manageable length for the R Shiny app described below. If the sheet is not trimmed, it continues to grow in size as the sensor runs, eventually slowing down the app. I made an optional workaround solution that uploads the data to three Google sheets, one with all the data, one with the past 30 days, and one with the past 7 days. The Shiny app only reads the data from the past 7 day sheet, keeping things relatively fast. `trim_google_sheets.py` trims the sheets so that they are the proper length. The sheet urls are read in by both `read_freezer_thermocouple.py` and `trim_google_sheets.py` from the `url` directory in the following tab-separated format, where the strings of characters are each sheet’s unique id (found in the sheet’s url between the last two slashes):

```bash
all	1PTW1-y7jl3w4oL8Hdp_pLoetTxZ0kBCYqSxjU1W_Hm8
week	1KpIEUuMpRD8q3DDNNUeJ1BqSztl_nAzA8DWtdTHFnVY
month	1_HbyPLVbzkS-GPMJ0nYEyBJogZneVJrzzAQgygPMeoY
pin	4
name	freezer_1
```

I run `trim_google_sheets.py` once a day with cron using the following crontab line:

```bash
0 23 * * * cd /home/pi/Documents/pi_sensor/python/ && python3 /home/pi/git/pi_sensor/python/trim_google_sheets.py > ~/cron_log.txt
```

### Hacky display fix
I’ve noticed a small problem with my MAX7219 LED matrix display: once every few days, some of the pixels with become stuck in the on or off position. While there is probably some way to clear the screen with software, I’ve decided to take the classic approach of turning it off and back on again a couple times each day with the following crontab line:

```bash
0 */12 * * * /sbin/shutdown -r now
```

## Monitor remotely with R Shiny app
I like to check in on the freezer, so I made a simply <a href ="https://shiny.rstudio.com/">R Shiny</a> app to visualize the data remotely. The app imports the data from the Google sheet and makes an interactive plot with <a href ="https://plotly.com/r/">Plotly</a>. You can try out the app here, and I’ve included all the code I used to make it in the repo linked above.

<div class="container is-max-desktop">
    <div class="columns">
        <div class="column is-10">
            <img src="/img/blog/2022-12-06_shiny_interface.jpg">
        </div>
    </div>
</div>

## Conclusion
Hopefully this post inspires you to use cheap hardware and freely available software tools to monitor your lab equipment. It might also convince you of the value in paying a company to do it for you. Doing it yourself can be a labor of love, but is also be pretty fun. Please feel free to reach out if you actually attempt this!

### A note about Raspberry Pi shortages
As of this writing, it is very hard to buy a Raspberry Pi. I was lucky to have a several before the shortages started, but the ongoing problems are making me start to seriously consider alternative single-board computers. Right now the best way to get a Raspberry Pi is to use <a href ="https://rpilocator.com/">this website</a> and <a href ="https://twitter.com/rpilocator/">Twitter page</a>, where restocks are automatically tracked and posted.

