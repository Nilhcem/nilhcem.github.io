---
layout: post
title:  Talk - Organizing an Android Things workshop
permalink: talk/organizing-an-android-things-workshop
date: 2017-04-06
comments: true
---

Eyal ([@Eyal_Lezmy][eyal-twitter]), Romain ([@romemore][romain-twitter]), and I organized an Android Things workshop during Devoxx France and droidcon Berlin.

This article is a post-mortem explaining what worked, what did not, and how we organized ourselves.

![pic01_full-room]

_<center>❤ Full room (40 attendees) ❤</center>_


## The idea

The workshop is titled "Android Things, from Developer to Maker".  
The maker word is important here. Not only we wanted attendees to discover Android Things, but also we wanted them to make a physical connected object, using Android Things. And if possible, a fun one.

To sum up, the object had to be fun, cheap, easy to make, and the Android Things integration should be simple enough to be done in a few hours by somebody who is discovering the platform.

Challenging! Then Eyal came up with the connected catapult idea:

<iframe width="560" height="315" src="https://www.youtube.com/embed/7kLAbW8O37M" frameborder="0" allowfullscreen></iframe>
<br>

We needed attendees to:

- Create a catapult using paper sheets, rubber bands, scotch tape, and glue (how-to video [here][howto-catapult])
- Use a servo motor as a locking mechanism for the catapult
- Use a button to toggle the servo motor state (locked / released)
- Use Wi-Fi to wirelessly control the catapult


## Gathering the hardware

We needed 20 Raspberry Pi + USB cables + micro SD cards + breadboards + servo motors + jumpers + resistors + buttons.

That's a lot of expensive stuff. Hopefully, we were very lucky, as Google decided to sponsor the workshop, lending us 20 Pimoroni Raspberry Pi 3 [Starter Kit][pimoroni-kit] for Android Things.  
We only had to contribute buying the servo motors and all the necessary stuff to create the catapults.

We had 1 Raspberry Pi for every 2 attendees. They had to pair and help each other. It was not something we initially planned, but it turned out to be a good idea. We had more interactions between everyone, and they could follow and complete the workshop easily together.


## Preparing the hardware

The workshop lasts 3 hours.  
We could not afford to ask attendees to flash Android Things on the SD cards, configure the Wi-Fi, and create the catapult.  
It would have been a massive waste of time _(around 80 minutes lost, at least)_, so we had to prepare everything a few days before the workshop starts.


**Preparing the catapults**

Not complicated, just time consuming, and painful when you burn your fingers using the glue gun.  

![pic02_catapults]
<br>

**Preparing the Android Things SD cards**

This part is tricky, and interesting.

At the time of the workshop, the latest version of Android Things was Developer Preview 0.5.1.  
In this version, to deploy applications, you need to connect to the board first, via adb over TCP (using the `adb connect <IP>` command).  
This means that you first need to know the IP of the board before accessing it.

Android Things, once booted, shows its IP on the external HDMI output. If you don't have an HDMI screen, you can use [nmap][nmap] to scan your network and guess the board's IP.  
This works fine... but when you have 20 boards connected at the same time, this is almost impossible to guess which IP is corresponding to which board.

That's why we decided to use static MAC address binding, so each board has its own IP.

To know the mac address, connect each board 1 by 1, and run:

```
$ adb shell cat /sys/class/net/wlan0/address
```

Then, label the board. Below is board #10. Thanks to static mac address binding, we know that its unique IP on the network will be `10.0.0.10`.

![pic03_labelled]

Of course, we don't have admin access to the conference Wi-Fi routers, so we had to use our own routers. This is great, because when the conference starts, each attendee can connect directly to the board easily. The only thing they have to do is to connect their computers to our own local network instead of the conference network.

To configure the Wi-Fi on 20 boards, we configured only one, then used `dd` to dump the OS + Wi-Fi configuration and flash all the other sd cards with our custom image.

By the way: flashing 20 SD cards is long... very long... and I'm lucky not to have done it personally.

Our custom Android Things image also included a sample app we created that moves a servo and turns on some LEDs. This was a pretty smart idea as we could ensure when the workshop started that every board was working well.

We actually had an issue with 1 board, and we could understand this issue quickly thanks to that app automatically installed. It's a kind of factory-testing mode.
<br><br>


### A Wi-Fi router for 20 Raspberry Pi and 20 computers, what could go wrong?

**This was our biggest mistake.** We decided to plug the Raspberry Pi over Wi-Fi, as we didn't have a router with at least 40 ethernet ports, so we decided to use a Wi-Fi router (wrt54g).

When the workshop started. Everyone had issues with the Wi-Fi. The connection to the network was slow. For some, impossible. The network was completely unstable. Devices disconnected frequently.

This lasted for 30 minutes (*of hell and stress, both for us and the attendees*), until Romain saved us installing a second router.  
We had 2 routers: one for the 20 Raspberry Pi, and 1 for the 20 attendees' computers.

Once installed, it worked flawlessly. The workshop was saved, and we had no network issues anymore when we did the workshop again at droidcon Berlin.


## Preparing the workshop

We have been preparing the workshop for 3 months on our spare time.

The 3 of us communicated using Hangouts, we had a shared Google documents where we could write the plan, and all our ideas. Slides were on Google Slides, and the workshop code on GitHub.

We initially wanted to use Google Nearby Connections API to create a phone-to-raspberry wireless communication. While it worked great when we started working on the workshop, we found [a blocking issue][issue] a week before the workshop started. We decided to use NanoHTTPD to control the catapult wirelessly.

It turned out to be a good idea, as it was very easy to implement for everyone, even with a small experience developing for Android.


## During the workshop

We started with a 45-minute introduction to Android Things.

[Slides are available here][slides]

Then, we had 2 hours left for the workshop. Attendees had to clone a repository, and follow the instructions on the `README.md` file.

[Workshop is available here][workshop]

We were happy seeing attendees having fun, and all succeeding in building their wireless catapult during the time of the workshop.


## After the workshop

Unfortunately, we could not give Android Things boards to attendees. Google lent us some, so we can reuse those in multiple events.  
And it's quite complicated to keep a close eye on 40+ people when you are only 3 organizers, especially when the 3 of us are already helping other attendees.

During devoxx, the room was closed, so when people left at the end of the workshop, it was easy for us to take the boards back and thank everyone personally.  
But at another event, the room was always open, and people came and left whenever they wanted, which was really hard for us to watch everyone. We lost 2 boards there.

After some discussion, Romain came with a nice idea: at the beginning of the workshop, we should take attendees badges before we lend them a board, and associate the board's number to the badge, so that people can take their badges back in exchange of the board.


## Videos

### #devoxxFR

<iframe width="560" height="315" src="https://www.youtube.com/embed/8IJgy1IaCIQ" frameborder="0" allowfullscreen></iframe>
<br>

### #droidconDE

<iframe width="560" height="315" src="https://www.youtube.com/embed/AK3Mk4RnJk8" frameborder="0" allowfullscreen></iframe>
<br>

Thanks a lot to everyone involved. The talk received great feedbacks, this means a lot to us given the amount of work we had provided to prepare it.

[eyal-twitter]: https://twitter.com/Eyal_Lezmy
[romain-twitter]: https://twitter.com/romemore
[howto-catapult]: https://www.youtube.com/watch?v=9JvV8PWawLs
[pimoroni-kit]: https://shop.pimoroni.com/products/rainbow-hat-for-android-things
[nmap]: http://stackoverflow.com/questions/41500019/android-things-how-do-i-connect-to-my-raspberry-pi-when-i-dont-know-the-ip-ad
[issue]: https://code.google.com/p/android/issues/detail?id=316748
[slides]: https://docs.google.com/presentation/d/1Oep62Au2nd0h74q-Ps-DEjht9J_7Fh3AancTdYOV7sc/edit?usp=sharing
[workshop]: https://github.com/eyal-lezmy/android-things-workshop

[pic01_full-room]: /public/images/20170406/01_full-room.jpg
[pic02_catapults]: /public/images/20170406/02_catapults.jpg
[pic03_labelled]: /public/images/20170406/03_labelled.jpg
