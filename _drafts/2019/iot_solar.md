---
layout: post
title: Solar Powered Raspberry Pi Zero
tags:
- iot
series: Smart Home
---

Some areas of my apartment lack convenient outlets to power smart home devices.  But living in a sunny climate makes solar power a viable option.  I ended up using the "adafruit build" (see projects [1](https://learn.adafruit.com/solar-boost-bag?view=all), [2](http://juliansarokin.com/how-to-build-a-solar-powered-raspberry-pi/), [3](https://www.instructables.com/lesson/Solar-USB-Charger-2/)):

- [6V 6W solar panel](https://www.adafruit.com/product/1525)
- [3.7V 6600mAh Li-Ion battery](https://www.adafruit.com/product/353)
- [USB/DC/Solar Li-Ion/Poly charger](https://www.adafruit.com/product/390): charges battery via solar panel
- [PowerBoost 500 converter](https://www.adafruit.com/product/1903): converts 3.7V to ~5V (for USB-powered devices)

Given that the [PiZero consumes around 0.5W](https://www.jeffgeerling.com/blogs/jeff-geerling/raspberry-pi-zero-power), 6W solar and a 6600mAh battery is overkill.  Over-specing now means I don't need to optimize for power and I can repurpose it with some of my other devices.

## Some Assembly Required

https://learn.adafruit.com/usb-dc-and-solar-lipoly-charger/overview

1. Solder:
    - Capacitor to solar charger (stripe is negative lead)
    - USB-A connector to PowerBoost
1. Connect:
    - Solar panel to charger (via [1.3mm to 2.1mm DC jack adapter](https://www.adafruit.com/product/2788))
    - Battery to charger (`BATT` connector)
    - Charger `LOAD` to PowerBoost converter (via spliced [JST PH 2-pin cables](https://www.adafruit.com/product/261))

![](/assets/solar_usb.jpg)

Assuming everything is done correctly, the orange `CHRG` LED on the solar charger flashes and the green `PWR` LED on the converter is lit.  If you're foolish like me and do this at night in a home with "energy efficient" LED lighting, a 1000 lumen torch does the trick.


## Alternatives

There's numerous other projects:

- Projects:
    - [Squirrel Kam](https://www.hackster.io/reichley/solar-powered-squirrel-kam-pi-zero-w-updated-797db4)
    - [Apple tree camera](https://kaspars.net/blog/solar-raspberry-pi-camera)
- All-in-one Solutions:
    - PiJuice: [1](https://www.electromaker.io/tutorial/blog/go-off-the-grid-with-your-raspberry-pi-zero-w), [2](https://howchoo.com/g/mmfkn2rhoth/raspberry-pi-solar-power)
    - [Solar Pi Platter](http://www.danjuliodesigns.com/products/solar_pi_platter.html)
    - [Witty Pi 2](http://www.uugear.com/portfolio/use-witty-pi-2-to-build-solar-powered-time-lapse-camera/)
    - [PiSolMan](http://www.pisolman.com/)