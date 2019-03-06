---
layout: post
title:  "Improving the Server Density Service Map"
date:   2016-04-17 12:16:54 +0000
categories: leds arudino
---
In my first attempt at Physical Website Monitoring I used some 8 bit shift registers and tri-colour LEDs. This while fun, was hard work. Each green, red and blue along with the earth legs of the LED had to be soldered and wired up. That’s OK for a small project but I wanted to expand to 10+ monitoring locations.

So plan B was to use Neopixels, as usual Adafruit has a great guide for them. I had some no brand ones lying around from a while back. You can buy the “branded” ones from any of the major online retailers or you can go cheap from eBay.

<img src="/assets/2016/04/neo-pixel-closeup.jpg" alt="Neo pixel">
Close up of Neopixel

Wiring them up is nice and simple. Just solder them together in serial following the arrows on their backs. They only have 3 pins: 5v, earth and data.

I roughly measured the distance between locations I am monitoring from  to measure the length of cable, and then solder the Neopixels together. I ran out of black wire pretty quick, which is why there is more red, sorry for any confusion. Below shows them attached to the back of the map.

<img src="/assets/2016/04/rear-view-of-map.jpg" alt="Rear view of map">

The code for controlling the lights is listed below:

```c
// Based on NeoPixel Ring simple sketch (c) 2013 Shae Erisson
// released under the GPLv3 license to match the rest of the AdaFruit NeoPixel library

#include <Adafruit_NeoPixel.h>
#ifdef __AVR__
  #include <avr/power.h>
#endif

// Which pin on the Arduino is connected to the NeoPixels?
// On a Trinket or Gemma we suggest changing this to 1
#define PIN            6

// How many NeoPixels are attached to the Arduino?
#define NUMPIXELS      10

// When we setup the NeoPixel library, we tell it how many pixels, and which pin to use to send signals.
// Note that for older NeoPixel strips you might need to change the third parameter--see the strandtest
// example for more information on possible values.
Adafruit_NeoPixel pixels = Adafruit_NeoPixel(NUMPIXELS, PIN, NEO_GRB + NEO_KHZ800);

int delayval = 500; // delay for half a second

String inputString = ""; // a string to hold incoming data
boolean stringComplete = false; // for serial message complete
String update = "";

void setup() {
  Serial.begin(9600);
  pixels.begin(); // This initializes the NeoPixel library.
}

boolean firstRun = true;

void loop() {

  if (firstRun){
    pixels.setPixelColor(0, pixels.Color(0,150,0));
    pixels.setPixelColor(1, pixels.Color(0,150,0));
    pixels.setPixelColor(2, pixels.Color(0,150,0));
    pixels.setPixelColor(3, pixels.Color(0,150,0));
    pixels.setPixelColor(4, pixels.Color(0,150,0));
    pixels.setPixelColor(5, pixels.Color(0,150,0));
    pixels.setPixelColor(6, pixels.Color(0,150,0));
    pixels.setPixelColor(7, pixels.Color(0,150,0));
    pixels.setPixelColor(8, pixels.Color(0,150,0));
    pixels.setPixelColor(9, pixels.Color(0,150,0));
    pixels.show();
    firstRun = false;
  }

  if (stringComplete) {
    inputString.replace("\n","");
    update = inputString;

    int lengthOfUpdate = update.length();
    int space = update.indexOf(' ');
    String location = update.substring(0, space);
    String locationStatus = update.substring(space + 1, lengthOfUpdate);

    changeLEDState(location, locationStatus);
    // clear the string:
    inputString = "";
    stringComplete = false;
  }

  delay(delayval); // Delay for a period of time (in milliseconds).
}

void changeLEDState(String location, String locationStatus){
  int pixel = 0;
  if (location == "northcalifornia") {
    pixel = 0;
  }
  else if (location == "vinadelmar") {
    pixel = 1;
  }
  else if (location == "johannesburg") {
    pixel = 2;
  }
  else if (location == "milan") {
    pixel = 3;
  }
  else if (location == "frankfurt") {
    pixel = 4;
  }
  else if (location == "london") {
    pixel = 5;
  }
  else if (location == "stockholm") {
    pixel = 6;
  }
  else if (location == "moscow") {
    pixel = 7;
  }
  else if (location == "toyko") {
    pixel = 8;
  }
  else if (location == "sydney") {
    pixel = 9;
  }

  if (locationStatus == "down") {
    Serial.print("updating ");
    Serial.print(location);
    Serial.println(" to down");
    pixels.setPixelColor(pixel, pixels.Color(150,0,0));
   } else if (locationStatus == "up") {
    Serial.print("updating ");
    Serial.print(location);
    Serial.println(" to up");
    pixels.setPixelColor(pixel, pixels.Color(0,150,0));
   } else if (locationStatus == "slow") {
    Serial.print("updating ");
    Serial.print(location);
    Serial.println(" to slow");
    pixels.setPixelColor(pixel, pixels.Color(255,140,0));
   }
  pixels.show();
}

void serialEvent() {
  while (Serial.available()) {
    // get the new byte:
    char inChar = (char)Serial.read();
    // add it to the inputString:
    inputString += inChar;
    // if the incoming character is a newline, set a flag
    // so the main loop can do something about it:
    if (inChar == '\n') {
      stringComplete = true;
    }
  }
}
```

I borrowed most of it from the example in Adafruit’s library for the Neopixel. I can’t stress enough how excellent Adafruit is a resource of how to do stuff!

You can test everything is working from the serial port of the Arduino. Send “sydney down\n” and Sydney will change to red, “slow” will change it to orange instead. “up” resets it to green.

The script below is some basic Python to fetch the last response time from Server Density and decided what if at all the colour of a location should be changed to. It’s all a bit rough around the edges but you get the idea.

```python
import json
import requests
from datetime import datetime, timedelta
import serial
from time import sleep
from copy import deepcopy

DEVICE_PATH = "/dev/tty.usbmodem1421"

token = "YOUR_TOKEN"
subject_id = "SERVICE_ID"

# get start and end date via datetime
start = datetime.now() - timedelta(seconds=600)
end = datetime.now()

# build a filter
filter = {
    'time':"all"
}

try:
    ser = serial.Serial(DEVICE_PATH, 9600, timeout=1)
except:
    ser = serial.Serial("/dev/ARUDINO_DEVICE", 9600, timeout=1)
sleep(5)

lookup_table = {
    "Australia: Sydney (Vultr)": "sydney ",
    "Chile: Vina del Mar": "vinadelmar ",
    "Germany: Frankfurt (Amazon)": "frankfurt ",
    "Italy: Milan": "milan ",
    "Japan: Tokyo (Amazon)": "toyko ",
    "UK: London (Vultr)": "london ",
    "South Africa: Johannesburg": "johannesburg ",
    "Russia: Moscow": "moscow ",
    "Sweden: Stockholm": "stockholm ",
    "USA: North California": "northcalifornia "
}


while (True):
    api_response = requests.get(
        'https://api.serverdensity.io/metrics/graphs/{0}'.format(subject_id),
        params={
            'token': token,
            'start' : start.isoformat(),
            'end': end.isoformat(),
            'filter': json.dumps(filter)
        }
    )

    actors = deepcopy(lookup_table)

    for stat in api_response.json()[0]['tree']:
        name = stat['name']
        timing = stat['data'][0]['y']

        print name
        serial_command = lookup_table[name]

        if timing > 0.4:
            print "Slow!"
            serial_command += "slow\n"

        else:
            serial_command += "up\n"

        print "writing " + serial_command
        ser.write(serial_command)
        sleep(1)
        del actors[name]
        print "{0} -> {1}".format(name, timing)


    for actor, name in actors.iteritems():
        serial_command = lookup_table[actor] + "down\n"
        ser.write(serial_command)
    sleep(600)
```

Below is it in action from the serial port:
<iframe width="560" height="315" src="https://www.youtube.com/embed/I7Nmzf8kif0" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>