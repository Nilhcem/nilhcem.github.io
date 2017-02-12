---
layout: post
title:  Android Things - Discovering the UART API
permalink: android-things/discovering-the-UART-api
date: 2017-02-15
---

![Serial port][pic1_serial]{: .center-image }

If you can easily recognize the Serial port on this picture, let me first remind you that you are old.<br>
We're going to talk about what's behind this interface today. Welcome to 2017.<br><br>


## Discovering the UART

A **UART** (Universal Asynchronous Receiver/Transmiter) is a small chip integrated on the board *(e.g. the Raspberry Pi)* that lets you perform a bi-directional communication between two devices.<br>
In ancient times, it was pretty useful, and seen under the form of a Serial port on a computer. Now it's still here and rocking, this time on nano-computers.

Over this blog post, we will discover the UART across two different Android Things use cases<br><br>


## #1: Communication between a Raspberry Pi and a computer

I always wanted to be a rock star, but can't play the piano. (*Do rockstars play piano?*)<br>
Yet, I have years practicing on a computer keyboard; so, why not use my laptop to play Europe's "The final countdown" song?

<iframe width="560" height="315" src="https://www.youtube.com/embed/OdsTbKnVXgY" frameborder="0" allowfullscreen></iframe>{: .center-image }
<br>

To achieve that, I bought a PL2303HX USB to TTL cable:

![UART cable][pic2_uartcable]{: .center-image }

We plug the USB cable into the computer, and the pins on the Raspberry Pi.

As the 'RT' (**R**eceiver/**T**ransmitter) in UART suggests , I have to plug the Receiver (**Rx**) cable wire to the Raspberry Pi Transmitter (**Tx**) pin, and the Transmitter (**Tx**) cable wire to the Raspberry Pi Receiver (**Rx**) pin. Also, plug the Ground wire to the Ground pin *(this one was easy)*.

![UART on rainbowHat][pic4_uartrainbow]{: .center-image }<br>


#### Raspberry Pi configuration

By default, the Raspberry Pi UART port is mapped to the Linux console. Before accessing UART from your app, we first have to mount the Android Things micro-SD card on our computer, modify the `cmdline.txt` file to remove the following statement: `console=serial0,115200`.<br>
Consult the [official documentation][raspberrypi-doc] for detailed instructions.<br><br>


#### Sample UART Loopback

The best way to start playing with UART is to clone the official UART sample [(sample-uartloopback)][official-uart-sample] and import it on Android Studio.<br>
The sample is easy to understand. First things first, the UART device initialisation:

{% highlight java %}
// We receive an instance of the "UART0" device.
UartDevice uart = new PeripheralManagerService().openUartDevice("UART0");

// Some basic UART configuration.
// Make sure to configure the other device the exact same way.
uart.setBaudrate(115200);
uart.setDataSize(8);
uart.setParity(UartDevice.PARITY_NONE);
uart.setStopBits(1);

// Finally, we register a callback on a background thread that will be
// triggered when some data is received via UART.
uart.registerUartDeviceCallback(mCallback, mInputHandler);
{% endhighlight %}

One baud is equal to 1 bit per second. A 115200 baud rate is therefore equal to 14KBps.<br>
Pretty slow, so better transfer small payloads of data via UART.

In the last line, we are registering a callback on a background thread that will be called each time UART receives some data.<br>
Here is an implementation of this callback:

{% highlight java %}
private UartDeviceCallback mCallback = new UartDeviceCallback() {
    @Override
    public boolean onUartDeviceDataAvailable(UartDevice uart) {
        // Read the received data
        byte[] buffer = new byte[512];
        int read;
        while ((read = uart.read(buffer, buffer.length)) > 0) {
            // And send it back to the other device, as a loopback.
            uart.write(buffer, read);
        }

        // Continue listening for more interrupts.
        return true;
    }

    @Override
    public void onUartDeviceError(UartDevice uart, int error) {
        Log.w(TAG, uart + ": Error event " + error);
    }
};
{% endhighlight %}

Receiving and Sending data over UART is as easy as calling `uart.read()` and `uart.write()` methods.<br>

My initial requirement was to play music when data is received.
I will modify the content of the callback's while loop to get only the first received char, and play a music note according to the given character

{% highlight java %}
int key = (int) buffer[0];
char c = Character.toUpperCase((char) key);
switch (c) {
    case 'F': playNote("F#"); break;
    case 'G': playNote("G#"); break;
    case 'H': playNote("A"); break;
    case 'J': playNote("B"); break;
    case 'K': playNote("C#"); break;
    case 'L': playNote("D"); break;
}
{% endhighlight %}

All the magic here is simply a basic switch statement that plays a note in the piezo buzzer. No rockstar stuff here after all, just smoke and mirror.<br>

### Using a Serial Console

Now that the Android Things project is done, We'll need a client on the computer to communicate via UART to the Android Things application.

For that, we will use a Serial Console Terminal program, such as PuTTY **(Windows)**, Serial **(Mac OS)**, CuteCom or Minicom **(Linux)**, with the same configuration as the one specified in the Android Things project (Baud Rate: 115200, Data Bits: 8, Parity: None, Stop Bits: 1)

{% highlight bash %}
$ screen /dev/ttyUSB0 115200
{% endhighlight %}

I connect to the USB device (on Linux) with a 115200 baud rate. Now every time I will write a character, it will be transferred to the Raspberry Pi via a Serial communication.
I can now play some music on my computer keyboard.<br>

Source code for *The Final Countdown Keyboard* is available on this [GitHub repository][final-countdown-uart].<br><br>


## #2: Communication between a Raspberry Pi and an Arduino

Let's now see a different use case.

Being a keyboard rockstar was surely fun, but if I'm using Android Things to create a smart device, I don't want to plug it on my computer over USB all the time! Can't we find a better usage of UART?

Imagine that you are building a smart robot. At some point, you'll find out that the Raspberry Pi has not enough pins to make all your dreams come true (*only 2 PWMs may not be enough for big needs*).<br>
How about delegating some stuff to a separate nano computer? In this use case, I'll choose an Arduino.<br><br>


### Here comes the Bi-Directional Logic Level Converter

I'm using an Arduino here, instead of 2 Raspberry Pi to show you something tricky:
The Raspberry Pi pins use 3.3V, while the Arduino pins use 5V.<br>
Connecting a Raspberry Pi pin to a voltage higher than 3.3V will likely damage your board.<br>
To plug these 2 components together, we will use a small chip that will adapt the voltage: a Bi-Directional Logic Level Converter.

![Level shifter][pic5_levelshifter]{: .center-image }

HV on the chip means *"**H**igh **V**oltage"*, and LV *"**L**ow **V**oltage"*.<br>
Here, a low-voltage signal sent to LV will be shifted up to the higher voltage and sent out HV<br>
HV/LV 1 to 4 are for data *(GPIO/I2C/UART)*. HV/LV is for the voltage pin, And once again, GND, for the ground pin.

Since the Raspberry Pi's using 3.3V it will be plugged to the LV side. The Arduino (5V) will be on the HV side.<br><br>


### Blinking LEDs, once again

Making an LED blink is our new *Hello, World!*

I will connect a button to the Arduino, and an LED to the Android Things Raspberry Pi.<br>
When the button is pressed, the Arduino will delegate its work to the Raspberry Pi via UART, so that the latter can turn its LED on/off.

See below a demonstration video

<iframe width="560" height="315" src="https://www.youtube.com/embed/7ncfLEEFoCo" frameborder="0" allowfullscreen></iframe>{: .center-image }
*<center>#BlinkingAnLEDIsSometimesAHardStuff</center>*<br>

#### Schematic

![Arduino schematic][pic6_arduinofritzing_lq]{: .center-image }
[full size version here][pic7_arduinofritzing_hq]<br><br>


The only component linking the 2 nano-computers is the UART connection (Rx/Tx/Ground) passing through a Bi-Directional Logic Level Converter.

On the Android Things project, when receiving data, I will simply toggle the LED using the GPIO API:

{% highlight java %}
private UartDeviceCallback mCallback = new UartDeviceCallback() {
    @Override
    public boolean onUartDeviceDataAvailable(UartDevice uart) {
        ledGpio.setValue(!ledGpio.getValue());
        return true;
    }
{% endhighlight %}

<br>And here's the Arduino sketch:<br>
When pressing the button on pin #9, I send the "Pressed!" payload to the UART:

{% highlight c %}
int buttonPin = 9;

void setup() {
  pinMode(buttonPin, INPUT_PULLUP);
  Serial.begin(115200);
}

void loop() {
  if (digitalRead(buttonPin) == LOW) {
    Serial.println("Pressed!");
    delay(1000);
  }
}
{% endhighlight %}
<br>

## Conclusion

Communicating via UART on Android Things is pretty easy. It's all about `read()` and `write()`.<br>
At this point *(Android Things Developer Preview 2)* where USB communication is not supported yet, it can be an alternative, as long as you send small payloads of data.

You can find many different UART use cases, from live-debugging your application over USB to your computer (e.g. *create a Stetho's dumpapp clone*) to interacting with/delegating to different components/nano-computers.<br>
As an Android Developer, it's been years I've not written any code in C. Delegating stuff from an Arduino to an Android Things Java 8 / Kotlin project could be a faster way for me to prototype/achieve something complex.

Want to get started with UART right now? Clone the official [sample uart loopback][official-uart-sample], and have fun!


[raspberrypi-doc]: https://developer.android.com/things/hardware/raspberrypi.html
[official-uart-sample]: https://github.com/androidthings/sample-uartloopback
[final-countdown-uart]: https://github.com/Nilhcem/uartfun-androidthings
[pic1_serial]: /public/images/20170215/01_serial.jpg
[pic2_uartcable]: /public/images/20170215/02_uartcable.jpg
[pic3_uartscheme]: /public/images/20170215/03_uartscheme.png
[pic4_uartrainbow]: /public/images/20170215/04_uartrainbow.jpg
[pic5_levelshifter]: /public/images/20170215/05_levelshifter.jpg
[pic6_arduinofritzing_lq]: /public/images/20170215/06_arduinofritzing_lq.jpg
[pic7_arduinofritzing_hq]: /public/images/20170215/07_arduinofritzing_hq.png
