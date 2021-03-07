---
layout: post
title: Heterogeneous Smart Home Lighting (Part 1)
tags:
- smarthome
- iot
- homeautomation
- hass
- hue
- ikea
series: Smart Home
---

I've been messing around with my ["smart home" setup for a while now]({% post_url /2019/2019-11-22-home_assistant %}).  I started with [Philips Hue](https://www.philips-hue.com/en-us) for no other reason than it seemed to be the leader or de-facto standard for smart lighting.  But, let's be honest- Hue isn't cheap, even for the white (non-colored) bulbs; they're 12-18 EUR or 12+ USD a piece depending on the bundle.

The other day I came across [this post](https://www.the-ambient.com/how-to/ikea-smart-bulbs-on-philips-hue-app-255) about using Ikea brand "Trådfri" smart lights (which start at 10 EUR/USD) alongside Hue.  On a whim I picked up a few different models and following the guide:

1. Hue App > ⋯ > Light Setup > Add light
1. Power-cycle the light 6 times

They all showed up like a charm and worked like the rest of my Hue lights.  Even the [LED Power Supply](https://www.ikea.com/us/en/p/tradfri-driver-for-wireless-control-gray-10356189/).  I was able to add them to existing rooms, and they even went straight into Home Assistant.

However, complications arose when I tried to use non-Zigbee devices (e.g. bluetooth/Wifi devices by [Wiz](https://www.wizconnected.com/en/consumer/), etc.), or non-lights like [Ikea power control](https://www.ikea.com/us/en/p/tradfri-wireless-control-outlet-30356169/) or Xiaomi motion sensor.

The [Ikea Trådfri motion sensor](https://www.ikea.com/us/en/p/tradfri-wireless-motion-sensor-white-60377655/) is a bit odd because while you __can__ pair it with 1 or more lights that it simply turns on, that hardly qualifies as "smart".  I'd like to use them like more generalized sensors.  I'm particularly excited by the Wiz motion sensor because while larger than some of the other options it can use standard AA rechargeable batteries instead of obnoxious batteries like CR2032:

![](/assets/wiz_motion_sensor_eneloop.jpg)

Turns out a lot of these failings can be solved with software (hurray!), but I leave that for my next post.
