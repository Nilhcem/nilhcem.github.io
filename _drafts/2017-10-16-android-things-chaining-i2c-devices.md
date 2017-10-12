---
layout: post
title:  Android Things - Discovering & chaining I²C devices
permalink: android-things/chaining-i2c-devices
date: 2017-10-16
comments: true
---

In a [previous post][previous-post], we created our own I²C device.  
This was actually a necessary first step before we could try a cool feature of I²C, which is supporting multiple slave devices connected along the same bus.  
Now that we've got some I²C devices, let's start.


## Our I²C devices

We will use 3 different devices:  

- SSD1306 OLED display ([driver][driver-ssd1306])
- LCD 1602 with PCF8574 I2C adapter ([driver][driver-pcf8574])
- Arduino IIC fan ([implementation][driver-fan])

![pic01_components]
<br>

The Android Things master device will start the fan at a lower speed, set the speed to medium, then high, and finally stop the fan.  
Each time the speed changes, the LCD and OLED screens will show the new state, either by displaying a Bitmap (for the SSD1306), or some text (for the LCD1602).
<br><br>


## Connecting the devices

You can connect multiple I²C devices on the bus in parallel.  
Here's how, from the [official I2C documentation][official-i2c-doc]:  

![pic02_connections]
<br>


## Determining I2C address without datasheet

With I²C, every slave device must have an address, even if the bus contains only a single slave.  
Thus, when you open an I2CDevice using the Things Support Library, you have to specify the I2C bus and the device address:

{% highlight kotlin %}
val pioService = PeripheralManagerService()
val device = pioService.openI2cDevice(I2C_BUS_NAME, I2C_ADDRESS)
{% endhighlight %}

As you have guessed, addresses are useful when multiple devices are connected to the same bus.  
Each device has its own I²C address, and it is usually mentioned in the device datasheet.

If you don't know a device's address, you can either:
- Use an Arduino to scan the bus using the [Arduino i2c scanner sketch][i2c-scanner]
- Add the following extension function to your Android Things project:  

{% highlight kotlin %}
fun PeripheralManagerService.scanI2cAvailableAddresses(i2cName: String): List<Int> {
    return (0..127).filter { address ->
        openI2cDevice(i2cName, address).use { device ->
            try {
                device.write(ByteArray(1), 1)
                true
            } catch (e: IOException) {
                false
            }
        }
    }
}
{% endhighlight %}

This function will loop for each address and try to write a "0" byte. If it succeeds, then it means a device is connected. The function will return a list of detected device addresses.

Here's how to call it and write device addresses to the logs:

{% highlight kotlin %}
Log.i(TAG, "Scanning I2C devices")
pioService.scanI2cAvailableAddresses(I2C_BUS_NAME)
    .map { String.format(Locale.US, "0x%02X", it) }
    .forEach { address -> Log.i(TAG, "Found: $address") }
}
{% endhighlight %}

In your Android logs, you'll get the following:

```
Scanning I2C devices
Found: 0x3C
Found: 0x3F
Found: 0x42
```

3 addresses were found: `0x3C` (OLED screen), `0x3F` (LCD screen), and `0x42` (Arduino fan).
<br><br>


## Displaying the current fan speed

We've got everything we need to use our I²C peripherals now.  
First, we define an enum for the fan speed

{% highlight kotlin %}
enum class Speed {
  LOW, MEDIUM, HIGH
}
{% endhighlight %}

And then, we can start initializing our I²C peripherals:

{% highlight kotlin %}
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    lcd = LcdPcf8574(I2C_PIN_NAME, I2C_ADDRESS_LCD).apply {
        begin(16, 2)
        setBacklight(true)
    }
    oled = Ssd1306(I2C_PIN_NAME, I2C_ADDRESS_OLED)
    fan = ArduinoFanI2C(I2C_PIN_NAME, I2C_ADDRESS_ARDUINO)
}
{% endhighlight %}


Now, we can write some functions to show the speed, for each display:

{% highlight kotlin %}
fun LcdPcf8574.showSpeed(speed: Speed) {
    clear()
    print("Speed is ${speed.name}")
}
{% endhighlight %}

and

{% highlight kotlin %}
fun Ssd1306.showSpeed(speed: Speed, resources: Resources) {
    val resId = when (speed) {
        LOW -> R.drawable.ssd1306_low
        MEDIUM -> R.drawable.ssd1306_medium
        HIGH -> R.drawable.ssd1306_high
    }

    clearPixels()
    val bmp = BitmapFactory.decodeResource(resources, resId)
    BitmapHelper.setBmpData(this, 0, 0, bmp, false)
    show()
}
{% endhighlight %}

We'll need the following assets:

![pic03_assets]
<br>

Finally, we can add the following code to start the fan, iterate through each speed and stop the fan

{% highlight kotlin %}
fan.start()

Speed.values().forEach { speed ->
    fan.speed = speed
    lcd.showSpeed(speed)
    oled.showSpeed(speed, resources)
    wait1sec()
}

fan.stop()
lcd.clear()
oled.clearPixels()
{% endhighlight %}
<br>

## Video

<iframe width="560" height="315" src="https://www.youtube.com/embed/Zh8MQJvvsh4?rel=0" frameborder="0" allowfullscreen></iframe>
<br><br>


## Conclusion

Communicating with multiple I²C slaves is really easy.  
You actually have nothing special to do, except connecting them on the bus in parallel.

You can find the complete source code on GitHub:  
[https://github.com/Nilhcem/i2cfun-androidthings/][i2cfun-sources]


[previous-post]: http://nilhcem.com/android-things/arduino-as-an-i2c-slave
[driver-ssd1306]: https://github.com/androidthings/contrib-drivers/tree/master/ssd1306
[driver-pcf8574]: https://github.com/Nilhcem/lcd-pcf8574-androidthings
[driver-fan]: https://github.com/Nilhcem/i2cfun-androidthings/blob/master/app/src/main/java/com/nilhcem/androidthings/i2cfun/device/components/ArduinoFanI2C.kt
[official-i2c-doc]: https://developer.android.com/things/sdk/pio/i2c.html
[i2c-scanner]: https://playground.arduino.cc/Main/I2cScanner
[i2c-doc]: https://developer.android.com/things/sdk/pio/i2c.html
[i2cfun-sources]: https://github.com/Nilhcem/i2cfun-androidthings/

[pic01_components]: /public/images/20171016/01_components.jpg
[pic02_connections]: /public/images/20171016/02_connections.png
[pic03_assets]: /public/images/20171016/03_assets.png
