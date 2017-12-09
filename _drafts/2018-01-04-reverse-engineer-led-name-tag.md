---
layout: post
title:  IoT - Reverse engineering a Bluetooth LED name badge
permalink: iot/reverse-engineering-bluetooth-led-name-badge
date: 2018-01-04
comments: true
---

Security in Bluetooth LE devices is optional, and many cheap products you can find on the market are not secured at all.  
While this could lead to some privacy issues, this can sometimes also be a source of fun, especially when trying to understand how a device works, so we can use it in a different way, or on a totally different platform.

I recently bought a Bluetooth LED name badge thinking it could be a nice external screen for some use cases for an Android Things project _(such as displaying the device IP, or some short text)_.

![pic01_device]

_Unlike what can be seen on the box, I would not recommend using this badge for your hotel employees, unless you have fun customers._
<br><br>


## The official application

The device is shipped with a very simple iOS / Android application that can send up to 8 different messages to the LED badge.  

![pic02_app]

For each message, you can set some options, such as the display mode or the scrolling speed.  
When pressing the send button, the app scans for compatible devices and sends data directly. Surprisingly, you don't need to pair the device to your phone to change the text.
<br><br>


## Sniffing Bluetooth packets

It's now time to understand how to control the device manually.  
First, let's enable the "Bluetooth HCI snoop log" Developer option ([more info on this article][reverse-engineering-article]) to see what happens under the hood when we send the `Hello` message:

![pic03_wireshark]

Using Wireshark, we can notice multiple write requests to the characteristic: `0xfee1` of the service `0xfee0`.

When we send the `Hello` message, we can see the 8 following write requests:

<pre>
77616E67000000000000000000000000
00050000000000000000000000000000
000000000000E10C06172D2300000000
00000000000000000000000000000000
00C6C6C6C6FEC6C6C6C600000000007C
C6FEC0C67C000038181818181818183C
000038181818181818183C0000000000
7CC6C6C6C67C00000000000000000000
</pre>

At that point, it is almost impossible to understand anything.  
We need to send some more messages to detect a common pattern, so let's try to send a different message, this time `Hello, World!` _(differences are highlighted)_:

<pre>
<span style="color: #777777">77616E67000000000000000000000000</span>
<span style="color: #777777">00</span><b>0E</b><span style="color: #777777">0000000000000000000000000000</span>
<span style="color: #777777">000000000000E10C0617</span><b>3633</b><span style="color: #777777">00000000</span>
<span style="color: #777777">00000000000000000000000000000000</span>
<span style="color: #777777">00C6C6C6C6FEC6C6C6C600000000007C</span>
<span style="color: #777777">C6FEC0C67C000038181818181818183C</span>
<span style="color: #777777">000038181818181818183C0000000000</span>
<span style="color: #777777">7CC6C6C6C67C0000000000000000</span><b>3030</b>
<b>1020000000000000000000000000C6C6</b>
<b>C6C6D6FEEEC68200000000007CC6C6C6</b>
<b>C67C0000000000DE76606060F0000038</b>
<b>181818181818183C00001C0C0C7CCCCC</b>
<b>CCCC760000183C3C3C18180018180000</b>
<b>00000000000000000000000000000000</b>
</pre>

There are lots of interesting information there:

* This time, 14 write requests were sent _(instead of 8 previously)_. The longer the text is, the more data is sent
* The first write request: `77616E67000000000000000000000000` is identical for the 2 examples.
* The second write request is almost similar, only one byte differs: previously `0x05` for the `Hello` message, now `0x0E` for the `Hello, World!` message.
* The third write request: `000000000000E10C0617????00000000` is again very similar, except for 2 bytes.
* Then, we have a write request filled with 0: `00000000000000000000000000000000`.
* Finally, the next four write requests are identical: `00C6C6C6C6FEC6C6C6C600000000007C`, `C6FEC0C67C000038181818181818183C`, `000038181818181818183C0000000000`, `7CC6C6C6C67C0000000000000000????`. Those probably mean "**Hello**", somehow.
* And the last write requests are specific to the second example, this is probably the value for the string: **, World!**
<br><br>

## Understanding the metadata

Now, instead of sending a single `Hello, World!` message. Let's try to send 2 messages at the same time: `Hello`, and `, World!`, and let's highlight the differences once again:

![pic04_app]

<pre>
<span style="color: #777777">77616E67000000000000000000000000</span>
<b>00050008</b><span style="color: #777777">000000000000000000000000</span>
<span style="color: #777777">000000000000</span><b>E10C07001C03</b><span style="color: #777777">00000000</span>
<span style="color: #777777">00000000000000000000000000000000</span>
<span style="color: #777777">00C6C6C6C6FEC6C6C6C600000000007C</span>
<span style="color: #777777">C6FEC0C67C000038181818181818183C</span>
<span style="color: #777777">000038181818181818183C0000000000</span>
<span style="color: #777777">7CC6C6C6C67C00000000000000003030</span>
<span style="color: #777777">1020000000000000000000000000C6C6</span>
<span style="color: #777777">C6C6D6FEEEC68200000000007CC6C6C6</span>
<span style="color: #777777">C67C0000000000DE76606060F0000038</span>
<span style="color: #777777">181818181818183C00001C0C0C7CCCCC</span>
<span style="color: #777777">CCCC760000183C3C3C18180018180000</span>
<span style="color: #777777">00000000000000000000000000000000</span>
</pre>

This starts to be interesting. Again:

* The first write request is always the same. This looks like a header.
* The second write request now starts with `00050008`. We can now understand that this second write request is indicating the length of the 8 messages. It starts with `0x0005` which is the length of the first message ("Hello"), and `0x0008` which is the length of the second message (", World!"). The next 12 Bytes are indicating the length of the 6 other messages (`0x0000`).
* When inspecting the third write request, we can realize that it is a value that seems to be incremented over the time. Probably a timestamp.  
Indeed, converting `E1:0C:07:00:1C:03` to decimal gives us: `225 12 07 00 28 03`.  
The request was sent on 2017/12/07 at 00:28:03, (and 2017 in hexadecimal is `0x07E1`, so `0xE1` is actually representing the last byte of the year).
* The fourth request is always composed of zeros. As if it is a separator between metadata and actual content
* Finally, the other requests are representing the "Hello, World!" string, somehow.
<br><br>

## Understanding the text data format

Everything is getting clearer.  
The last thing we need to understand is how the "Hello" string can be translated to `00C6C6C6C6FEC6C6C6C600000000007CC6FEC0C67C000038181818181818183C000038181818181818183C00000000007CC6C6C6C67C00000000000000000000`.

Instead of spending too much time understanding so many Bytes, let's send a shorter message: `A`, instead of `Hello`.

![pic05_app_a]

<pre>
<span style="color: #777777">77616E67000000000000000000000000</span>
<span style="color: #777777">00050000000000000000000000000000</span>
<span style="color: #777777">000000000000E10C0700203100000000</span>
<span style="color: #777777">00000000000000000000000000000000</span>
<b>00386CC6C6FEC6C6C6C600</b><span style="color: #777777">0000000000</span>
</pre>

Looks like the character A is represented by the `00:38:6C:C6:C6:FE:C6:C6:C6:C6:00` hex value.  
If we convert each byte to binary, this will be:

<pre>
0x00: 00000000
0x38: 00111000
0x6C: 01101100
0xC6: 11000110
0xC6: 11000110
0xFE: 11111110
0xC6: 11000110
0xC6: 11000110
0xC6: 11000110
0xC6: 11000110
0x00: 00000000
</pre>

Have you noticed anything special? Let's highlight the "1s":

<pre>
<span style="color: #aaaaaa">00000000</span>
<span style="color: #aaaaaa">00</span><b>111</b><span style="color: #aaaaaa">000</span>
<span style="color: #aaaaaa">0</span><b>11</b><span style="color: #aaaaaa">0</span><b>11</b><span style="color: #aaaaaa">00</span>
<b>11</b><span style="color: #aaaaaa">000</span><b>11</b><span style="color: #aaaaaa">0</span>
<b>11</b><span style="color: #aaaaaa">000</span><b>11</b><span style="color: #aaaaaa">0</span>
<b>1111111</b><span style="color: #aaaaaa">0</span>
<b>11</b><span style="color: #aaaaaa">000</span><b>11</b><span style="color: #aaaaaa">0</span>
<b>11</b><span style="color: #aaaaaa">000</span><b>11</b><span style="color: #aaaaaa">0</span>
<b>11</b><span style="color: #aaaaaa">000</span><b>11</b><span style="color: #aaaaaa">0</span>
<b>11</b><span style="color: #aaaaaa">000</span><b>11</b><span style="color: #aaaaaa">0</span>
<span style="color: #aaaaaa">00000000</span>
</pre>

Each bit of the hex data `00:38:6C:C6:C6:FE:C6:C6:C6:C6:00` is representing an LED. 1 to turn it on, and 0 to turn it off

![pic06_badge_a]
<br><br>

## Writing an Android app to control the LED badge

We now understand how the Bluetooth LED badge works, converting a text to multiple byte arrays we can send using the Bluetooth LE APIs.

The implementation will consist of manipulating bits. That may be tricky. A single bit error and nothing will work, plus it will be hard to debug.  
For those reasons, and since the specs are now perfectly clear, it is strongly recommended to start writing unit tests before the code implementation.

This is an example of the unit tests that I wrote:

{% highlight kotlin %}
fun `result should start with 77616E670000`() {
    // Given
    val data = DataToSend(listOf(Message("A")))

    // When
    val result = DataToByteArrayConverter.convert(data).join()

    // Then
    result.slice(0..5) `should equal` listOf<Byte>(0x77, 0x61, 0x6E, 0x67, 0x00, 0x00)
}

fun `size should contain a 2 byte hexadecimal value for each message`() { ... }

fun `the 6 next bytes after the size should all be equal to 0x00`() { ... }

fun `timestamp should contain 6 bytes, 1 for the last 2 digits of the year, 1 for the month, the day, the hour, the minute and the second`() { ... }

fun `the 20 next bytes after the timestamp should all be equal to 0x00`() { ... }

fun `message should contain hex code for each character, skipping invalid characters`() { ... }

fun `each write request should contain 16 bytes`() { ... }
{% endhighlight %}

If you are interested, the tests implementation can be found [here][test-implementation].
<br><br>

## Using the Android Bluetooth LE APIs

Our app can translate a String message into byte arrays we can send via Bluetooth.  
We can now use the [Android Bluetooth APIs][android-ble-apis] to scan for the LED badge, and send the data once a device has been found.  
For more information, you can read the following blog post:  
[Communicating with Bluetooth Low Energy devices][ble-blog-post].

![pic07_itworks]
<br><br>


## Adding a new feature

The official app is lacking a fun feature: the possibility to send a Bitmap to the LED badge directly.  
Since we now have a full control over the device, we can create a 40x11px bitmap and write some code to convert it into byte arrays we can send to the LED badge.

![pic08_pixelart]

{% highlight kotlin %}
for (i in 0 until (40 / 8)) {
  for (row in 0 until 11) {
    var byte = 0
    for (col in 0 until 8) {
      val isOn = bitmap.getPixel((i * 8) + col, row) != Color.TRANSPARENT
      byte = byte or ((if (isOn) 1 else 0) shl (7 - col))
    }
    bytes.add(byte)
  }
}
{% endhighlight %}

![pic09_final]
<br><br>


## Conclusion

Reverse engineering this Bluetooth LE device was a lot of fun.  
It's rewarding to finally understand how things work after a few tries, and now the LED badge can be controlled by any devices, and not only the provided mobile app.

I can use the LED badge for different purposes such as an Android Things external wireless screen, and can even display any bitmap that I want.

The complete source code to control this device (you can buy on [aliexpress][aliexpress]) is available on this link:  
[https://github.com/Nilhcem/ble-led-name-badge-android/][github-code]


[reverse-engineering-article]: http://nilhcem.com/iot/reverse-engineering-simple-bluetooth-devices
[test-implementation]: https://github.com/Nilhcem/ble-led-name-badge-android/blob/master/app/src/test/java/com/nilhcem/blenamebadge/device/DataToByteArrayConverterTest.kt
[android-ble-apis]: https://developer.android.com/guide/topics/connectivity/bluetooth-le.html
[ble-blog-post]: http://nilhcem.com/android-things/bluetooth-low-energy
[github-code]: https://github.com/Nilhcem/ble-led-name-badge-android/
[bitmap-code]: https://github.com/Nilhcem/ble-led-name-badge-android/blob/master/app/src/main/java/com/nilhcem/blenamebadge/device/DataToByteArrayConverter.kt#L137
[aliexpress]: https://fr.aliexpress.com/item/Sign-Scrolling-advertising-business-card-show-digital-display-tag-LED-name-badge-Rechargeable-Programmable-White/32759781214.html

[pic01_device]: /public/images/20180104/01_device.jpg
[pic02_app]: /public/images/20180104/02_app.png
[pic03_wireshark]: /public/images/20180104/03_wireshark.png
[pic04_app]: /public/images/20180104/04_app.png
[pic05_app_a]: /public/images/20180104/05_app_a.png
[pic06_badge_a]: /public/images/20180104/06_badge_a.jpg
[pic07_itworks]: /public/images/20180104/07_works.jpg
[pic08_pixelart]: /public/images/20180104/08_pixelart.png
[pic09_final]: /public/images/20180104/09_final.jpg
