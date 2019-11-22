---
layout: post
title: Solar Powered USB
tags:
- smarthome
- iot
- homeautomation
series: Smart Home
---

Some areas of my apartment lack convenient power outlets for smart home devices.  Running cables everywhere will decimate the fengshui of my abode, but living in a sunny climate makes solar power a palatable option.  I ended up trying the "adafruit build" (see projects [1](https://learn.adafruit.com/solar-boost-bag?view=all), [2](http://juliansarokin.com/how-to-build-a-solar-powered-raspberry-pi/), [3](https://www.instructables.com/lesson/Solar-USB-Charger-2/))- aptly named because all components are available from [Adafruit](https://www.adafruit.com):

- [6V 6W solar panel](https://www.adafruit.com/product/1525)
- [3.7V 6600mAh Li-Ion battery](https://www.adafruit.com/product/353)
- [USB/DC/Solar Li-Ion/Poly charger](https://www.adafruit.com/product/390): charges battery via solar panel
- [PowerBoost 500 converter](https://www.adafruit.com/product/1903): converts 3.7V from charger/battery to ~5V for USB-powered devices (like Raspberry Pi 3/4/Zero)

Given that the [PiZero consumes around 0.5W](https://www.jeffgeerling.com/blogs/jeff-geerling/raspberry-pi-zero-power), 6W solar and a 6600mAh battery is overkill for many devices.  Overspeccing now means:  
1. Don't need to optimize software for power consumption
1. Can repurpose it with some of my other devices

## Some Assembly Required

The small number of components really only fit together one way and some basic soldering is required.  [Adafruit has a tutorial](https://learn.adafruit.com/usb-dc-and-solar-lipoly-charger/overview) or you can follow one of the related projects mentioned above, but the instructions boil down to:

1. Solder:
    - Capacitor to solar charger (stripe is negative lead)
    - USB-A connector to PowerBoost
1. Connect:
    - Solar panel to charger (via [1.3mm to 2.1mm DC jack adapter](https://www.adafruit.com/product/2788))
    - Battery to charger's `BATT` connector
    - PowerBoost converter to charger's `LOAD` connector (via spliced [JST PH 2-pin cables](https://www.adafruit.com/product/261))

![](/assets/solar_usb.jpg)

I didn't have any shrink wrap on-hand so I used a bit of electrical tape for exposed wire bits.

Assuming everything is done correctly, expose the solar panel to sunlight and the orange `CHRG` LED on the solar charger flashes and the green `PWR` LED on the converter is lit.  If you're foolish like me and do this at night in a home with "energy efficient" LED lighting, a 1000 lumen torch does the trick.
 
 ## A Long Straw

 Should you need to place the solar panel some distance from the rest of the hardware (e.g. outside):

 - Shortest, cheapest 3.2/1.3mm to 5.5/2.1mm adapter
 - Appropriate length of amply available, inexpensive 5.5/2.1mm DC extension cables

 ## Winning

This was a fun and incredibly rewarding project.

On occasion I share these side projects with my family, but this was the first that garnered a reaction and aroused interest.  Few people can appreciate the subtle nuances of software development, but in 2019 *everyone* can grok the covetousness of portable USB power!

The modular nature of it also deserves attention.  You can disconnect the JST from the PowerBoost converter and hook the 3.7V from the charger/battery directly up to 3.3V devices.  Plan to use an ultra-low-power Arduino?  Swap out the 6W solar panel and 6600mAh battery for something more modest.

I'm tempted to revisit my [air-quality monitor({% post_url /2019/2019-03-25-pi3_pm25 %}), but no longer living in Shanghai I have little use for it.

## Alternatives

There's numerous other potential solutions if you've got different requirements (or hold a grudge against Adafruit).  A by no means exhaustive list in no particular order:

- Projects:
    - [Squirrel Kam](https://www.hackster.io/reichley/solar-powered-squirrel-kam-pi-zero-w-updated-797db4)
    - [Apple tree camera](https://kaspars.net/blog/solar-raspberry-pi-camera)
- All-in-one Solutions:
    - PiJuice: [1](https://www.electromaker.io/tutorial/blog/go-off-the-grid-with-your-raspberry-pi-zero-w), [2](https://howchoo.com/g/mmfkn2rhoth/raspberry-pi-solar-power)
    - [Solar Pi Platter](http://www.danjuliodesigns.com/products/solar_pi_platter.html)
    - [Witty Pi 2](http://www.uugear.com/portfolio/use-witty-pi-2-to-build-solar-powered-time-lapse-camera/)
    - [PiSolMan](http://www.pisolman.com/)