---
layout: post
title: Solar Powered USB
tags:
- smarthome
- iot
- homeautomation
- hass
series: Smart Home
---

https://www.home-assistant.io/integrations/mqtt_room
https://www.home-assistant.io/getting-started/presence-detection/
https://www.home-assistant.io/integrations/person/

## VS Code

1. Install `Remote - SSH` extension ([`ms-vscode-remote.remote-ssh`](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh))
1. __Activity Bar > Remote Explorer__ (or `Cmd+Shift+p` __> Remote Explorer: Focus on SSH Targets View__)
1. __SSH Targets > `+`__ to "Add New" and input the ssh command; e.g. `ssh pi@pi3.local` will add the following to `~/.ssh/config`:
    ```
    Host pi3.local
        HostName pi3.local
        User pi
    ```
1. Setup ssh key-based authentication. `pi@pi3.local:~/.ssh/authorized_keys`:
    ```
    ssh-rsa <host:~/.ssh/id_rsa.pub>
    ```

1. Select __pi3.local > `+`__ to "Connect to Host in New Window" opens:  

![](/assets/vscode_remote_ssh_pi3.png)



https://rust-embedded.github.io/book/intro/install.html

brew install armmbed/formulae/arm-none-eabi-gcc openocd qemu

```
-----------------------------------
esptool.py wrapper for MATRIX Voice
-----------------------------------
/usr/bin/voice_esp32_enable: line 8: /sys/class/gpio/gpio25/direction: Permission denied
/usr/bin/voice_esp32_enable: line 9: /sys/class/gpio/gpio24/direction: Permission denied
/usr/bin/voice_esp32_enable: line 11: /sys/class/gpio/gpio25/value: Permission denied
/usr/bin/voice_esp32_enable: line 13: /sys/class/gpio/gpio24/value: Permission denied
/usr/bin/voice_esp32_enable: line 14: /sys/class/gpio/gpio25/value: Permission denied
/usr/bin/voice_esp32_enable: line 15: /sys/class/gpio/gpio24/value: Permission denied
/usr/bin/voice_esp32_enable: line 16: /sys/class/gpio/gpio25/value: Permission denied
/usr/bin/voice_esp32_enable: line 17: /sys/class/gpio/gpio24/value: Permission denied
esptool.py v2.8
Serial port /dev/ttyS0
Traceback (most recent call last):
  File "/usr/local/bin/esptool.py", line 3201, in <module>
    _main()
  File "/usr/local/bin/esptool.py", line 3194, in _main
    main()
  File "/usr/local/bin/esptool.py", line 2889, in main
    esp = chip_class(each_port, initial_baud, args.trace)
  File "/usr/local/bin/esptool.py", line 237, in __init__
    self._port = serial.serial_for_url(port)
  File "/usr/lib/python2.7/dist-packages/serial/__init__.py", line 88, in serial_for_url
    instance.open()
  File "/usr/lib/python2.7/dist-packages/serial/serialposix.py", line 268, in open
    raise SerialException(msg.errno, "could not open port {}: {}".format(self._port, msg))
serial.serialutil.SerialException: [Errno 2] could not open port /dev/ttyS0: [Errno 2] No such file or directory: '/dev/ttyS0'
done

[SUCCESS] Please disconnect your MatrixVoice from the RaspberryPi and reconnect it alone for future OTA updates.
```