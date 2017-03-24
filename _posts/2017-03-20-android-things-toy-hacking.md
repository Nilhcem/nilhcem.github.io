---
layout: post
title:  Android Things - Hack your toys to make them smarter
permalink: android-things/toy-hacking
date: 2017-03-20
comments: true
---

One of the advantages of Android Things is its simplicity to implement complex features to small devices, using a JVM compatible language and the Android framework.

So far, we have been creating new projects from scratch using a breadboard and a few components. Today we will "refactor" an existing project: an electronic toy.<br><br>


## Follow me Poli

![Original toy][pic1_original-toy]

This electronic toy is simple, it consists in an autonomous car that automatically follows the red light emitted by a stick.<br>

This could be fun for children, but there's one problem though: **it does not really work as expected**.<br>
The car is moving too randomly, the lightstick is useless, children are frustrated and customers are disappointed.

![Amazon reviews][pic2_amazon-reviews]

Look at those Amazon reviews! It's so sad, because the toy could be fun and the hardware looks Ok.<br>
It is time to use some Android Things magic here: replacing the software, and making the toy fun to use.
<br><br>


## Let's take a look inside

First, we need to unscrew the toy case to see what's inside:

![Inside the toy][pic3_inside]

We can see that we have 2 DC motors, some sensors, LEDs, and other components we can reuse.
Let's focus on the motor part first.


## Making the car move

Instead of having a car that moves randomly, we will transform this toy into a remote controlled car.<br>
We won't need to buy a car chassis, 2 DC motors and a battery holder, as we will reuse all those components from the original toy directly.

When taking a closer look at the toy, we can find some red and black wires from the battery box (_power and ground_). Also, each motor has 2 wires (_to simplify: one to make it move forward, and one to make it move backward_).

![DC motors][pic4_dcmotors]
<br>

We are going to bare all those wires, and plug them into an **L298N Dual Motor Controller**.<br>
For more info on that part, I wrote [an article here][self-promotion] that explains why the L298N is useful, and how to connect everything to the Android Things board.<br>

![L298N][pic5_l298n]
<br>

Now, let's figure out which GPIO wire is for which direction. For that, there is no secret: we will give current to each GPIO, and observe which wheel is moving to which direction.

{% highlight java %}
PeripheralManagerService manager = new PeripheralManagerService();
Gpio gpio = manager.openGpio("BCM22")
gpio.setValue(true)
{% endhighlight %}

<iframe width="560" height="315" src="https://www.youtube.com/embed/cA9LqPu7ui0" frameborder="0" allowfullscreen></iframe>
<br>

It seems that this *BCM22* GPIO here is for actually moving the left wheel forward. Now we can repeat the same operation with the 3 others GPIOs, and we will be able to figure out how making the car move.

{% highlight java %}
public void move(Direction direction) {
  boolean valueLeftForward = false;
  boolean valueRightBackward = false;
  boolean valueLeftBackward = false;
  boolean valueRightForward = false;

  switch (direction) {
    case UP:
      valueLeftForward = true;
      valueRightForward = true;
      break;
    case DOWN:
      valueLeftBackward = true;
      valueRightBackward = true;
      break;
    case LEFT:
      valueRightForward = true;
      break;
    case RIGHT:
      valueLeftForward = true;
      break;
  }

  gpioLeftForward.setValue(valueLeftForward);
  gpioRightBackward.setValue(valueRightBackward);
  gpioLeftBackward.setValue(valueLeftBackward);
  gpioRightForward.setValue(valueRightForward);
}
{% endhighlight %}
<br>


## Using other components

We can focus on reusing other components if we want.<br>
For example, it could be a good idea to add a light sensor, and automatically turn on the car's headlights when the luminosity is low.

The toy already has 2 LEDs for its headlights, we will use a resistor and a multimeter to test those LEDs first and determine the positive/negative leads.

![Multimeter][pic6_multimeter]
<br>


## Wireless control

To wirelessly control the car, I initially thought using the Nearby Connections API over Wi-Fi, as I [already did some time ago][self-promotion].<br>
It works flawlessly, and is easy to integrate.

Problem: this blog post would not be very different from the original one, so I decided to use an even more generic solution: **embedding an HTTP server** inside the Android Things project


### NanoHTTPD to the rescue

We will use NanoHTTPD, a tiny web server in Java.<br>
You only need to add one dependency to your `build.gradle` file:

{% highlight groovy %}
compile 'org.nanohttpd:nanohttpd:${version}'
{% endhighlight %}

And then, you can start and stop your server in your lifecycle methods (_e.g. onCreate/onDestroy_).<br>
To serve pages, you need to override a `serve` method.

Simplified code:

{% highlight java %}
public class HttpdServer extends NanoHTTPD {

  private static final int PORT = 8888;

  public interface OnDirectionChangeListener {
    void onDirectionChanged(String direction);
  }

  private OnDirectionChangeListener listener;

  public HttpdServer(OnDirectionChangeListener listener) {
    super(PORT);
    this.listener = listener;
    start(NanoHTTPD.SOCKET_READ_TIMEOUT, false);
  }

  @Override
  public Response serve(IHTTPSession session) {
    Map<String, List<String>> parameters = session.getParameters();
      if (parameters.get("direction") != null) {
        listener.onDirectionChanged(parameters.get("direction").get(0));
      }

    String html =
      "<html><head><script type=\"text/javascript\">" +
      "  function move(direction) { window.location = '?direction='+direction; }" +
      "</script></head>" +
      "<body>" +
      "  <button onclick=\"move('UP');\">UP</button>" +
      "  <button onclick=\"move('DOWN');\">DOWN</button>" +
      "  <button onclick=\"move('LEFT');\">LEFT</button>" +
      "  <button onclick=\"move('RIGHT');\">RIGHT</button>" +
      "  <button onclick=\"move('STOP');\">STOP</button>" +
      "</body></html>";

      return newFixedLengthResponse(html);
    }
}
{% endhighlight %}

That way, the Android Things device will embed an HTTP server. To control it, simply open the device URL on any browser and click on the direction buttons.<br>
When a button is pressed, an observer is notified so it can move the car to the specified direction.


## Conclusion

Sometimes, you don't need to buy components (_e.g. a car chassis, DC motors, LEDs..._) to create some new projects. Reusing existing components can be a good alternative.<br>
The car toy is now able to serve web pages, and can be controlled manually. Implementing those features was pretty quick.

To finish, we can place the Raspberry Pi inside the toy, and use a USB portable battery pack to power it.<br>
I only had a huge Power Bank, but a smaller one could easily be placed inside the box, so it won't look too experimental 😂.

<iframe width="560" height="315" src="https://www.youtube.com/embed/Q4ukWClPJQE" frameborder="0" allowfullscreen></iframe>

[self-promotion]: http://nilhcem.com/android-things/discovering-the-GPIO-api-building-a-remote-car
[pic1_original-toy]: /public/images/20170320/01_followme-poli.jpg
[pic2_amazon-reviews]: /public/images/20170320/02_amazon-reviews.png
[pic3_inside]: /public/images/20170320/03_inside.jpg
[pic4_dcmotors]: /public/images/20170320/04_dcmotors.jpg
[pic5_l298n]: /public/images/20170320/05_l298n.jpg
[pic6_multimeter]: /public/images/20170320/06_multimeter.jpg
