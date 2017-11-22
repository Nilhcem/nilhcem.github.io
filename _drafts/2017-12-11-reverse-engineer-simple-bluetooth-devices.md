---
layout: post
title:  IoT - Reverse engineering simple Bluetooth LE devices
permalink: iot/reverse-engineering-simple-bluetooth-devices
date: 2017-12-11
comments: true
---

In a [previous post][ir-rgb-bulb-post], we reverse-engineered an infrared light bulb, so we can control it using the Google Assistant, but we quickly encountered some limitations due to the infrared technology.

Our goal this time is to replace infrared lights with Bluetooth bulbs instead, but before that let's practice on a very simple device to reverse engineer: a smart candle.<br><br>


## MiPow PlayBulb candle

The PlayBulb candle is a Bluetooth 4.0 LED light controlled via your smartphone.

![pic01_smartcandle]

The smartphone app allows us to choose the candle color and optionally apply a flickering candle effect

![pic02_smartcandle_app]
<br>

Our goal is to understand how the smartphone application communicates with the smart candle, so we can later control the candle color programmatically, for instance setting it from  green to red when our continuous integration server notifies us that the build is broken.
<br><br>


## Scanning the Bluetooth device

A few months ago, when we built our own [Android Things Bluetooth LE device][ble-devices-post], we used the [nRF Connect][nrf-app] Android application to test our Bluetooth implementation easily.

We will use the same application, this time to get the address of the Bluetooth smart candle, and see which services/characteristics it is advertising.

![pic03_nrfcandle]

* On the first screenshot, we can get our device Bluetooth address: `F1:5A:4B:16:AC:E6`
* On the second screenshot, we can see that the smart candle is advertising multiple services:
  * "Generic Access" (`0x1800`) and "Generic Attribute" (`0x1801`)
  * "Battery Service" (`0x180F`) and "Device Information" (`0x180A`)
  * And 2 unknown services: `0xFF02` and `0x1016`
* On the third screenshot, we can see that each service has many characteristics, for instance the service `0x1800` has a "Device Name" characteristic (`0x2A00`) that allows us to get the name of the device (here "PLAYBULB CANDLE") when pressing the read icon _(arrow down)_. We can also set a custom name pressing the write icon _(arrow up)_.

If we want to set the color of the bulb programmatically, we can guess that we will have to write some data to a characteristic of either of the 2 unknown services.  
Now the question is: _"What should I write, to which characteristic, from which service?"_.  
There are many unknowns!
<br><br>


## Intercepting Bluetooth packets

To answer that question, we will intercept Bluetooth packets, and use the official Android application to control the candle colors.

Android, since 4.4 (KitKat) has a really convenient Developer Option, name "Bluetooth HCI snoop log", which will sniff Bluetooth HCI (Host Controller Interface) packets, and save those in the `/sdcard/Android/data/btsnoop_hci.log` file of the device.

![pic04_devoption]

Let's enable this option, and start the official candle app to change the color to red, green, and blue.  
At that moment, the `btsnoop_hci.log` file should contain the packets that were sent from the phone to the candle to change its colors.

We can now use adb to pull the file to our computer, and open it with Wireshark _(clicking on File > Open, and selecting our .log file)_.

```
adb pull /sdcard/Android/data/btsnoop_hci.log
```

![pic05_wireshark]

Since we are only interested in analyzing Bluetooth packets related to our smart candle, we can add a filter, specifying our Bluetooth device address we got from the nRF Connect app, adding the following filter:

```
bluetooth.addr==F1:5A:4B:16:AC:E6
```

![pic06_filter]

There are 5 important details in this screenshot:  
1. We have added a filter to see only Bluetooth packets related to our smart candle
2. We can see 3 write requests from `localhost()` (our smartphone) to the candle
3. When selecting the first request, we can see that it's a write request targeting the service `0xff02`
4. The characteristic UUID=`0xfffc` is concerned
5. The value `0x00ff0000` is written for the first request (red).

The value from the second request _(green)_ is `0x0000ff00`, and the one from the third request _(blue)_ is `0x000000ff`.  
We can detect a pattern here: the value is actually the hexadecimal color code, which means that if we want to change the candle color to yellow (`#ffff00`), we will have to write the value `0x00ffff00` to the characteristic `0xfffc` of the service `0xff02`.
<br><br>

## Testing

Let's open the nRF Connect app again, select the `0xff02` service, click on the write button for the characteristic `0xfffc` and send the value `0x00ffff00`

![pic07_nrfsend]
<br>

We have sent the `#ffff00` hex color code to the smart candle, and the candle is now yellow!

![pic08_yellow]
<br>

Now that we know how to change colors, we can write some Android code using the [Bluetooth LE APIs][ble-apis] to control the candle programmatically.  
As you have seen, it is really easy with Android to capture Bluetooth packets, and analyse those using Wireshark.  

We can now do the exact process to understand the other features of the smart candle

* **Flickering candle effect**: Write `00RRGGBB0X000100` to characteristic=`0xfffb` from service=`0xff02`, where `RRGGBB` is the hex color code and `X` is either `4` to activate the effect, and `5` to deactivate the effect.
* **Flashing (on/off)**: Write `00RRGGBB0000SS00` to characteristic=`0xfffb` from service=`0xff02` where `RRGGBB` is the hex color code and `SS` the speed from `00` (very fast) to `FF` (very slow)
* **Pulse (smooth on/off)**: Write `00RRGGBB0100SS00` to characteristic=`0xfffb` from service `0xff02` where `RRGGBB` is the hex color code and `SS` the speed from `00` (very fast) to `FF` (very slow)
* **Rainbow (switch colors)**: Write `0000ff000200SS00` to characteristic=`0xfffb` from service `0xff02` where `SS` is the speed from `00` (very fast) to `FF` (very slow)
* **Smooth rainbow**: Write `0000ff000300SS00` to characteristic=`0xfffb` from service `0xff02` where `SS` is the speed from `00` (very fast) to `FF` (very slow)
* **No color**: Write `00000000` to characteristic=`0xfffc` from service=`0xff02`.
* **Default candle color (whitish)**: Write `ff000000` to characteristic=`0xfffc` from service=`0xff02`.
<br><br>


## Reverse engineering an RGB bulb

Since candles are not especially useful, let's buy a Magic Blue Bluetooth RGB bulb, for $10 on aliexpress

![pic09_magicblue]
<br>

After sniffing Bluetooth packets the same way, we can see that we need to write to the characteristic=`0xffe9` from the service=`0xffe5`.

This screenshot from wireshark is from a packet that sets the color to red:

![pic10_wireshark]

To control the bulb, you have to write the following value:
* **Turn on**: `cc2333`
* **Turn off**: `cc2433`
* **Custom color**: `56RRGGBB00f0aa` where `RRGGBB` is the hex color code
<br><br>


## Next step

Reverse engineering those minimalist Bluetooth devices was fairly easy.  
In a following article, we will try to reverse-engineer step by step a little more complex device, and write an Android application to interact with it easily.

[ir-rgb-bulb-post]: http://www.nilhcem.com/iot/reverse-engineering-ir-rgb-bulb
[ble-devices-post]: http://nilhcem.com/android-things/bluetooth-low-energy
[nrf-app]: https://play.google.com/store/apps/details?id=no.nordicsemi.android.mcp
[ble-apis]: https://developer.android.com/guide/topics/connectivity/bluetooth.html

[pic01_smartcandle]: /public/images/20171211/01_smartcandle.jpg
[pic02_smartcandle_app]: /public/images/20171211/02_smartcandle_app.png
[pic03_nrfcandle]: /public/images/20171211/03_nrfcandle.png
[pic04_devoption]: /public/images/20171211/04_devoption.png
[pic05_wireshark]: /public/images/20171211/05_wireshark.jpg
[pic06_filter]: /public/images/20171211/06_filter.jpg
[pic07_nrfsend]: /public/images/20171211/07_nrfsend.jpg
[pic08_yellow]: /public/images/20171211/08_yellow.jpg
[pic09_magicblue]: /public/images/20171211/09_magicblue.jpg
[pic10_wireshark]: /public/images/20171211/10_wireshark.jpg
