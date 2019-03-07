---
layout: post
title:  "Maplin Arm"
date:   2016-07-10 12:16:54 +0000
categories: arm maplin
---
Decided to power up my Maplin arm and was having issues getting it going on OS X El Capitan.

I kept seeing the error “usb.core.NoBackendError: No backend available” looks like at some point while upgrading I no longer had the correct usb libraries. Installing libusb-compat via brew fixed the issue.

```
>>> s = MaplinRobot()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "maplinrobot.py", line 26, in __init__
    self.rctl = usb.core.find(idVendor=self.usb_vendor_id, idProduct=self.usb_prod_id) #Object to talk to the robot
  File "/Library/Python/2.7/site-packages/usb/core.py", line 1263, in find
    raise NoBackendError('No backend available')
usb.core.NoBackendError: No backend available

# fix
$ brew install libusb-compat
```