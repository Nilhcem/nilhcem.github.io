---
layout: post
title:  IoT - A beginner's guide to making your own devices from scratch
permalink: iot/make-your-own-devices-from-scratch
date: 2019-05-20
comments: true
---

![pic01_device]
<br>

Building a prototype from scratch used to be really expensive not so long ago.  
The overall cost has dropped significantly years after years, and everyone is now able to build their own prototypes at home with very little means.

This post will explain, from a beginner's perspective, one of the many different approaches to change your ideas into fully functional objects.
<br><br>

_Back in 2005, when building the now famous SoftBank Robotics NAO, each required prototype iteration used to cost tens of thousands of dollars to build. With a lot of will now, this could almost be a single person's hobby._
![pic02_nao]
<br>


## Choose the hardware

The hardware will depend on what you want to build.  
A good start is to use a famous prototyping board with a microcontroller unit embedded, such as an Arduino, an ESP8266 or an ESP32.  
You'll also need some sensors and actuators.

The [previous post][cloud-iot-core-with-the-esp32-and-arduino] was about how you could program an ESP32 to send temperature and humidity data to the Google Cloud platform. We'll turn this idea into an actual product and build everything in a single day.

Instead of an ESP32, we'll take a less efficient but cheaper ESP8266 (WeMos D1 Mini, $2), and we'll keep the great BME280 temperature and humidity sensor ($3).

![pic03_breadboard]
<br>


## Write the software

You'll choose your programming language depending on your microcontroller.  
The ESP8266 offers a large choice of programming languages you can use:
- C (FreeRTOS)
- Arduino C (Arduino)
- Python (MicroPython)
- Lua (NodeMCU)
- Javascript (Mongoose OS)
- Go (Gobot)
- _...and others_

Arduino C is good to start, as the language is easy to use for beginners.  
Later, you could still rewrite your code for FreeRTOS if you want the software to be better optimized.

We will keep the Arduino C code we wrote in the previous article, as the code is already written and both the BME280 and the [Google Cloud IoT JWT][google-cloud-iot-jwt] libraries are compatible with the ESP8266.

The program will send temperature and humidity data every minute to Google Cloud.
<br><br>


## Create the circuit board

Fritzing is a nice software that allows you to model your components wiring.

![pic04_fritzing]

There's also a PCB tab to let you easily create a PCB layout for manufacturing, in just a few clicks.  

![pic05_fritzing_pcb]

Once made, just click on the "Export for PCB" button to generate an Extended Gerber (RS274X) archive you can upload to various online PCB printing services websites such as [fab.fritzing.org](http://fab.fritzing.org) ([aisler.net](https://aisler.net)).
<br><br>


## Print the circuit board

It was convenient for me to use an online service to print the circuit board and ship it to me.  
Here's a screenshot of my aisler order:  

![pic06_aisler]
The PCB cost me around $5 (I ordered 3).
<br><br>

![pic07_pcb]
<br><br>


## Solder components

Easy part, you just need to take your soldering iron (_if you don't have any, the TS100 is a good choice_). I decided here to use pin headers so I can plug/unplug the esp8266 and bme280 easily whenever I want.

![pic08_pcb-headers]
<br><br>


## Design a custom case

To design a case, we can use Autodesk Fusion 360 which is an easy-to-use 3D Design & Modeling software that offers a free license for hobbyists, startups and makers.  

I started by designing four 6x6mm squares located on each side of the 58x28mm PCB. Their goal is to hold the PCB.

![pic09_fusion-start]

Then, you need to extrude the sketch to the desired height, and iterate creating and extruding other sketches until you reach your goal.

![pic10_fusion-end]

Step-by-step video:

<iframe width="740" height="416" src="https://www.youtube.com/embed/bQVoHQdDbxQ" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
<br><br>


## 3D-Print it

The 3D model is finished. We can export it to an STL file that can be open using Cura, a free software that will convert the STL file into a set of instructions for a 3D printer.

![pic11_cura]
<br><br>

You can use an online 3D printing service. However, I found them quite expensive, and you can now find good 3D printers for around $200.  
I am using the Elegoo NEPTUNE 3D printer with a transparent PLA filament:

![pic12_elegoo]
<br><br>

Time-lapse using a wood filament:

<iframe width="740" height="416" src="https://www.youtube.com/embed/iWcA3oAxggo" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
<br><br>


## Ready for mass production

Excluding the price of the existing hardware _(computer, 3D printer)_ and time spent, for around $12 I was able to create a fully functional electronic device that can be used to monitor the temperature in my home and send data to the Google Cloud Platform.  
Nowadays, creating your own electronic devices from scratch is a reality. It gives a rewarding feeling, the device is unique and does exactly what you expect it to do.

We could, of course, find cheaper and better temperature sensors in the market. However, finding one that does exactly what I wanted it to do _(sending specific data to GCP)_ is more difficult.

Also, since I've built the device, I can modify it anytime. Want to update the software to send data to a local server? Want to add an LED strip below the transparent cover so that it illuminates in a specific color when the temperature is too cold?  
Everything's possible!

![pic13_production]

[cloud-iot-core-with-the-esp32-and-arduino]: http://nilhcem.com/iot/cloud-iot-core-with-the-esp32-and-arduino
[google-cloud-iot-jwt]: https://github.com/GoogleCloudPlatform/google-cloud-iot-arduino/

[pic01_device]: /public/images/20190520/01_device.jpg
[pic02_nao]: /public/images/20190520/02_nao.jpg
[pic03_breadboard]: /public/images/20190520/03_breadboard.jpg
[pic04_fritzing]: /public/images/20190520/04_fritzing.png
[pic05_fritzing_pcb]: /public/images/20190520/05_fritzing_pcb.jpg
[pic06_aisler]: /public/images/20190520/06_aisler.png
[pic07_pcb]: /public/images/20190520/07_pcb.jpg
[pic08_pcb-headers]: /public/images/20190520/08_pcb-headers.jpg
[pic09_fusion-start]: /public/images/20190520/09_fusion-start.jpg
[pic10_fusion-end]: /public/images/20190520/10_fusion-end.jpg
[pic11_cura]: /public/images/20190520/11_cura.jpg
[pic12_elegoo]: /public/images/20190520/12_elegoo.jpg
[pic13_production]: /public/images/20190520/13_production.jpg
