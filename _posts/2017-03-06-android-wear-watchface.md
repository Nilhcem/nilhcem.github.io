---
layout: post
title:  Android Wear - Improve your Android skills with style, building watch faces
permalink: android-wear/watchfaces-design
date: 2017-03-06
comments: true
---

Constantly wearing a watch that matches with your clothing can quickly turn out to be an expensive idea.

Hopefully, we, android developer, do not have that kind of problems, as we only need an Android Wear watch and some free time, so we can develop our own watch faces.

In this blog post, I will tell you how I created a watch face from a hoodie, during an initially boring evening at home.<br>
Please note this is not a step-by-step tutorial, but more likely a "making-of". If you're interested in knowing how to build watch faces from scratch, follow the official [watchface codelab][watchface-codelab].

![Watch face preview][pic1_final]
<br>

It all started a week ago, when my eyes landed on a sweatshirt that I immediately decided to buy.

![Hoodie preview][pic2_hoodie]

I actually liked it so much (*don't judge me, taste is subjective*) that I started creating a watch face inspired from it.


## Where to start?

Sadly, I am a very bad graphic designer (*objectively, this time*) and drawing the shape by hand would take me years. The fastest and easiest solution for me was to take a picture of the hoodie and use GIMP's fuzzy select (magic wand) to cut it, then resize it, color it to white, and save it.

![GIMP preview][pic3_gimp]

Once the shape is exported to a file with transparent background, displaying the bitmap on the watch screen was easy:

{% highlight java %}
// First, you need a bitmap reference
public void onCreate() {
  Bitmap lionShape = BitmapFactory.decodeResource(res, R.drawable.lion_shape);
}

// Then, in the onDraw, you clear the background, and draw the bitmap
public void onDraw(Canvas canvas, Rect bounds) {
  canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR);
  canvas.drawBitmap(lionShape, 0, 0, null);
}
{% endhighlight %}
<br>

## Ok, now how to display the time?

I came across multiple ways to display the time, but all those made the watch face ugly:

First, the *"I-have-no-imagination-at-all"* solution, which consists in placing watch hands over the background image.

{% highlight java %}
// For that, we first create our paint object
public void onCreate() {
  handPaint = new Paint();
  handPaint.setColor(Color.WHITE);
  handPaint.setStrokeWidth(8f);
  handPaint.setShadowLayer(4f, 2f, 2f, Color.GRAY);
  handPaint.setAntiAlias(true);
  handPaint.setStrokeCap(Paint.Cap.ROUND);
}

// Then, we draw some lines for the hour and minute hands,
// rotating the canvas before drawing the lines.
public void onDraw(Canvas canvas, Rect bounds) {
  canvas.save();
  canvas.rotate(angleHours, centerX, centerY);
  canvas.drawLine(centerX, centerY, centerX, centerY - hourHandLength, handPaint);
  canvas.rotate(angleMinutes - angleHours, centerX, centerY);
  canvas.drawLine(centerX, centerY, centerX, centerY - mnHandLength, handPaint);
  canvas.restore();
}
{% endhighlight %}

![Ugly result][pic4_ugly]
*#ObjectivelyUgly*

I tried experimenting with other solutions, as seen below:

![Experimentations][pic5_experiments]
<br>


## Finally, a single arc, to show the time

The previous solutions were not respecting the original hoodie design, which only has a single outer circle.<br>

So I was thinking how we could display the time in a readable manner, using a single outer circle.<br>
A way to achieve that is to draw a **clockwise arc**, starting from the usual hours-hand rotation to the minutes-hand rotation.

Let's say it's 10:30, so I draw a clockwise arc (via [Canvas#drawArc][canvas-drawarc-doc]) from 10'o clock to 30 minutes

![10:30][pic6_ten-thirty]

{% highlight java %}
public void onDraw(Canvas canvas, Rect bounds) {
  // Get the clock-hands rotations
  Calendar calendar = timezoneHelper.getCalendar();
  float secondsRotation = calendar.get(Calendar.SECOND) + calendar.get(Calendar.MILLISECOND) / 1000f;
  float minutesRotation = calendar.get(Calendar.MINUTE) + secondsRotation / 60f;
  float hoursRotation = calendar.get(Calendar.HOUR) + minutesRotation / 60f;

  // Draw the time arc
  float sweepAngle = (360f - hoursRotation + minutesRotation) % 360f;
  canvas.drawArc(arcBounds, hoursRotation - 90, sweepAngle, false, paintObject);
}
{% endhighlight %}

It gives the watch a unique look (as people don't usually know how to tell the time there), and respects the hoodie design.<br>
Also, once you are used to, you can tell the time very quickly.
<br><br>


## Put some gold in your eyes

The hoodie is gold on black, so I'll need a gold texture:

![Gold texture][pic7_gold-texture]

Then, I can use apply this texture on my bitmap programmatically via [PorterDuff XferMode][porterduff]

{% highlight java %}
Bitmap background = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888);
Canvas canvas = new Canvas(result);
canvas.drawBitmap(goldTexture, 0, 0, null);

Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST_IN));
canvas.drawBitmap(lionShape, 0, 0, paint);
{% endhighlight %}

I will also apply this texture on my Paint object (*so the time-arc will be in gold too*) via a Shader
{% highlight java %}
BitmapShader goldShader = new BitmapShader(goldTexture, TileMode.CLAMP, CLAMP);
paint.setShader(goldShader);
{% endhighlight %}
<br>


## Ambient mode

When the watch is not used, the watch face goes into an ambient mode, a "battery-saver" mode where most of the pixels are black (so they are turned off, on an AMOLED screen).

There are many ways to create an ambient mode. To me, a watch face does not have to be especially pretty in ambient mode (as you won't need it to look at it), but it should obviously still display the time.

So I decided not to draw the lion shape, and use a different Paint object on ambient mode (white color, no shaders, thinner stroke width)

{% highlight java %}
public void onAmbientModeChanged(boolean inAmbientMode) {
  paintObject = inAmbientMode ? ambientPaint : interactivePaint;
}
{% endhighlight %}

You can see the ambient mode is very minimalist, that way, it does not consume that much battery, and we can still tell the time

![Ambient mode][pic8_ambient]
<br><br>


## Conclusion

I created this watch face only for myself.<br>
I am not planning to release it publicly, as I am not the author of the original design.<br>
The complete source code *(using a different bitmap)* is available at [github.com/Nilhcem/hoodie-androidwear][hoodie-androidwear]

Learning how to develop watch faces for Android wear is fun and rewarding.<br>
It gives us a fast feeling of accomplishment, as we'll only need to spend a few hours/days for a nice watch face, while creating a nice mobile app takes usually longer than that.

Also, learning how to develop watch faces for Android Wear can also make you become a better Android developer, as you'll play with some concepts you can reuse in your apps.
After a few days, you'll probably become a Path / Canvas / Custom Views expert.

![Final design][pic9_final]

[watchface-codelab]: https://codelabs.developers.google.com/codelabs/watchface/index.html
[canvas-drawarc-doc]: https://developer.android.com/reference/android/graphics/Canvas.html#drawArc(android.graphics.RectF,%20float,%20float,%20boolean,%20android.graphics.Paint)
[porterduff]: http://ssp.impulsetrain.com/porterduff.html
[hoodie-androidwear]: https://github.com/Nilhcem/hoodie-androidwear
[pic1_final]: /public/images/20170306/01_final.jpg
[pic2_hoodie]: /public/images/20170306/02_hoodie.jpg
[pic3_gimp]: /public/images/20170306/03_gimp.jpg
[pic4_ugly]: /public/images/20170306/04_ugly.jpg
[pic5_experiments]: /public/images/20170306/05_experiments.jpg
[pic6_ten-thirty]: /public/images/20170306/06_ten-thirty.png
[pic7_gold-texture]: /public/images/20170306/07_gold-texture.jpg
[pic8_ambient]: /public/images/20170306/08_ambient.jpg
[pic9_final]: /public/images/20170306/09_final.jpg
