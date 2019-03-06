---
layout: post
title:  "Physical Website Monitor based on Server Density’s Monitoring"
date:   2016-02-20 12:16:54 +0000
categories: leds arudino
---

I wanted to make something to make monitoring more tangible. So I made a board to display the current status of this website chrishannam.co.uk as monitored from a number of remote “actors” provided by Server Density.

Below is a snapshot of the monitoring setup from Server Density’s service page.

<img src="/assets/2016/02/world-map.png" alt="World Map">
Remote monitoring actors.

The build was pretty basic and luckily I had the parts lying around from previous projects. Rather than explain the setup I’ll give you the link I used as it covers everything better than I could explain. Adafruit Shift Register is an excellent guide on wiring and programming 8 bit shift registers. The only difference is I used tri colour LEDs. The LEDs I used were almost identical to these Tri Colour LEDs from eBay. They do red, green and blue light. I just removed the blue leg as I didn’t need it.

The LEDs are mounted in a 6mm thick panel of MDF. The map was just a simple one printed off Wikipedia.

I used an Arduino Uno but any basic Arduino is up to the job. The code to control the board is listed below. It’s based on the Adafruit example from their excellent guide.

```c
/*
Adafruit Arduino - Lesson 4. 8 LEDs and a Shift Register - Brightness
*/

int latchPin = 5;
int clockPin = 6;
int dataPin = 4;
int outputEnablePin = 3;

byte leds = 0;

String inputString = ""; // a string to hold incoming data
boolean stringComplete = false; // for serial message complete
String location = "";
boolean california = false;
boolean uk = false;
boolean moscow = false;
boolean sydney = false;

void setup()
{
  Serial.begin(9600);
  pinMode(latchPin, OUTPUT);
  pinMode(dataPin, OUTPUT);
  pinMode(clockPin, OUTPUT);
  pinMode(outputEnablePin, OUTPUT);
  setBrightness(255);

  // set all to red
  Serial.println("All Red");
  bitSet(leds, 0);
  bitSet(leds, 2);
  bitSet(leds, 4);
  bitSet(leds, 6);
  updateShiftRegister();
}

void loop()
{
  if (stringComplete) {
    inputString.replace("\n","");
    location = inputString;
    Serial.println(location);
    changeLEDState();
    // clear the string:
    inputString = "";
    stringComplete = false;
  }
  delay(500);
}

void changeLEDState()
{
  if (location == "california") {
    // green for ok or true
    if (california == true) {
      Serial.println("updating California to down");
      bitClear(leds, 1);
      updateShiftRegister();
      bitSet(leds, 0);
      updateShiftRegister();
      california = !california;
    } else {
      Serial.println("updating California to up");
      bitClear(leds, 0);
      updateShiftRegister();
      bitSet(leds, 1);
      updateShiftRegister();
      california = !california;
    }
  }
  else if (location == "uk") {
    // green for ok or true
    if (uk == true) {
      Serial.println("updating uk to down");
      bitClear(leds, 3);
      updateShiftRegister();
      bitSet(leds, 2);
      updateShiftRegister();
      uk = !uk;
    } else {
      Serial.println("updating uk to up");
      bitClear(leds, 2);
      updateShiftRegister();
      bitSet(leds, 3);
      updateShiftRegister();
      uk = !uk;
    }
  } else if (location == "moscow") {
    // green for ok or true
    if (moscow == true) {
      Serial.println("updating moscow to down");
      bitClear(leds, 5);
      updateShiftRegister();
      bitSet(leds, 4);
      updateShiftRegister();
      moscow = !moscow;
    } else {
      Serial.println("updating moscow to up");
      bitClear(leds, 4);
      updateShiftRegister();
      bitSet(leds, 5);
      updateShiftRegister();
      moscow = !moscow;
    }
  } else if (location == "sydney") {
    // green for ok or true
    if (sydney == true) {
      Serial.println("updating sydney to down");
      bitClear(leds, 7);
      updateShiftRegister();
      bitSet(leds, 6);
      updateShiftRegister();
      sydney = !sydney;
    } else {
      Serial.println("updating sydney to up");
      bitClear(leds, 6);
      updateShiftRegister;
      bitSet(leds, 7);
      updateShiftRegister();
      sydney = !sydney;
    }
  }
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

void updateShiftRegister()
{
   digitalWrite(latchPin, LOW);
   shiftOut(dataPin, clockPin, LSBFIRST, leds);
   digitalWrite(latchPin, HIGH);
}

void setBrightness(byte brightness) // 0 to 255
{
  analogWrite(outputEnablePin, 255-brightness);
}
```
View from behind.

<img src="/assets/2016/02/wiring-from-behind.png" alt="World Map">

Here it is in action.
<iframe width="560" height="315" src="https://www.youtube.com/embed/QvG5ow2t0T4" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


Next I needed to talk to Server Density’s API which luckily is pretty simple. I get the last time from each actor, and test to see if it’s below 0.4 seconds.
0.4 ensures at least 1 server usually Sydney will be down, so it makes for a better display. The Arduino code flips the colour of the LED to make updates a binary change with a message over serial.

```python
import json
import requests
from datetime import datetime, timedelta
import serial
from time import sleep

DEVICE_PATH = "/dev/YOUR_DEVICE"

token = "YOUR TOKEN"
subject_id = "SERVICE ID FROM Server Density"

# get start and end date via datetime
start = datetime.now() - timedelta(seconds=600)
end = datetime.now()

# build a filter
filter = {
    'time':"all"
}

ser = serial.Serial(DEVICE_PATH, 9600, timeout=1)
sleep(5)
ser.write('california\n')
sleep(1)
ser.write('uk\n')
sleep(1)
ser.write('moscow\n')
sleep(1)
ser.write('sydney\n')
sleep(1)


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

    for stat in api_response.json()[0]['tree']:
        name = stat['name']
        timing = stat['data'][0]['y']

        if timing > 0.4:
            print "Slow!"
            if name == "USA: North California":
                ser.write('california\n')
                sleep(1)
            if name == "UK: London (Vultr)":
                ser.write('uk\n')
                sleep(1)
            if name == "Russia: Moscow":
                ser.write('moscow\n')
                sleep(1)
            if name == "Australia: Sydney (Vultr)":
                ser.write('sydney\n')
                sleep(1)
        print ("{0} -> {1}".format(name, timing))
    sleep(600)
```
One interesting thing about the code is the number of sleeps.

*OPENING A SERIAL CONNECTION TO AN ARDUINO CAUSES IT TO RESTART!*

Bear that in mind. You need to allow time for the Arduino to setup the connection over the serial connection is initiated. This also applies to sending data backwards and forwards as well. Adding the sleeps ensures everything runs smoothly and nothing gets lost.


This was a basic prototype, I am hoping to expand it to an A3 sized map with more locations.