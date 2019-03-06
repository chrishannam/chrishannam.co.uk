---
layout: post
title:  "Analog Dials"
date:   2014-12-01 12:16:54 +0000
categories: analog arduino
---
I have a deep love of old style analog dials. Recently I found a place selling old voltage meters. There were unusable by todays standards but they looked great. With a bit customising I set about converting them to use hobby servos and make them fully controlled by an Arduino.

The analog dials:

I have a deep love of old style analog dials. Recently I found a place selling old voltage meters. There were unusable by todays standards but they looked great. With a bit customising I set about converting them to use hobby servos and make them fully controlled by an Arduino.

The analog dials:

<img src="/assets/2014/12/3_dials.jpg" alt="3 Dials">

Below is the fairly quick transformation. Gutted the insides of the unit and used a glue gun to mount the servo and blu tac to attach the needle.

<img src="/assets/2014/12/monitor-in-bits.jpg" alt="Dial in bits">
<img src="/assets/2014/12/work-of-art.jpg" alt="Work of Art">

After the servos were mounted, I added a tri colour LED.

<img src="/assets/2014/12/lights.jpg" alt="Monitors">

It took a bit effort to get the casing back on the dials especially with the LED bent over the top the dial. Both the dials are mounted on an old shelf.

<img src="/assets/2014/12/wiring.png" alt="Monitors">

Above is the basic wiring, two servo controls and 6 LED controls. I kept the breadboard to make sure I was able to change things later.

Below is the simple code I used to control them:

```cpp
/*
I started with the sample from this guy

Sweep
 by BARRAGAN <http://barraganstudio.com>
 This example code is in the public domain.

 modified 8 Nov 2013
 by Scott Fitzgerald
 http://arduino.cc/en/Tutorial/Sweep
*/

#include <Servo.h>

Servo output_dial;  // create servo object to control a servo
Servo input_dial;   // twelve servo objects can be created on most boards

boolean stringComplete = false;
String inputString = "";
int led_green_input = 2;
int led_blue_input = 3;
int led_red_input = 4;
int led_green_output = 6;
int led_blue_output = 7;
int led_red_output = 8;

void setup()
{
  Serial.begin(9600);
  input_dial.attach(9);  // left
  output_dial.attach(10); // right
  pinMode(led_green_input, OUTPUT);
  pinMode(led_blue_input, OUTPUT);
  pinMode(led_red_input, OUTPUT);
  pinMode(led_green_output, OUTPUT);
  pinMode(led_blue_output, OUTPUT);
  pinMode(led_red_output, OUTPUT);
  output_dial.write(135); // reset to left side of the dial
  input_dial.write(145); // reset to left side of the dial

  // set green as LED colour default
  digitalWrite(led_red_input, LOW);
  digitalWrite(led_blue_input, LOW);
  digitalWrite(led_green_input, HIGH);
  digitalWrite(led_red_output, LOW);
  digitalWrite(led_blue_output, LOW);
  digitalWrite(led_green_output, HIGH);
}

void loop()
{

  if (stringComplete) {
    inputString.replace("\n","");
    int len = inputString.length();
    String servo = inputString.substring(0,1);
    String colour = inputString.substring(2,3);
    String movement = inputString.substring(4,len);
    Serial.println(stringComplete);

    // limit how far we move the meter to make suer we don't
    // damage the needle on the side of the casing.

    // right dial
    if (servo.toInt() == 1 && movement.toInt() <= 135 && movement.toInt() >= 30){
      output_dial.write(movement.toInt());
    }

    // left dial
    if (servo.toInt() == 0 && movement.toInt() <= 145 && movement.toInt() >= 55){
      input_dial.write(movement.toInt());
    }

    if (servo == "1"){
      if (colour.toInt() == 0){
        digitalWrite(led_red_output, LOW);
        digitalWrite(led_blue_output, LOW);
        digitalWrite(led_green_output, HIGH);
      }
      if (colour.toInt() == 1){
        digitalWrite(led_red_output, LOW);
        digitalWrite(led_blue_output, HIGH);
        digitalWrite(led_green_output, LOW);
      }
      if (colour.toInt() == 2){
        digitalWrite(led_blue_output, LOW);
        digitalWrite(led_red_output, HIGH);
        digitalWrite(led_green_output, LOW);
      }
    }
    if (servo == "0"){
      if (colour.toInt() == 0){
        digitalWrite(led_red_input, LOW);
        digitalWrite(led_blue_input, LOW);
        digitalWrite(led_green_input, HIGH);
      }
      if (colour.toInt() == 1){
        digitalWrite(led_red_input, LOW);
        digitalWrite(led_blue_input, HIGH);
        digitalWrite(led_green_input, LOW);
      }
      if (colour.toInt() == 2){
        digitalWrite(led_blue_input, LOW);
        digitalWrite(led_red_input, HIGH);
        digitalWrite(led_green_input, LOW);
      }
    }
    // clear the string:
    inputString = "";
    stringComplete = false;
  }

  delay(1000);

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

[gist](https://gist.github.com/bassdread/952e9df9d820c16841f3)

The above code is enough to move the needle the full range of the dial without hitting the side and damaging the needle. It also controls the colour of the LED. Currently it does a basic solid colour without any mixing.

To make it standalone I connected the Arduino to a Raspberry Pi and ran the following simple Flask app to control the dials over serial.

```
from flask import Flask
import serial
from datetime import datetime, timedelta
from time import sleep

serial_port = '/dev/tty.usbserial-A501S1GX'
ser = serial.Serial(serial_port, 9600)
sleep(1.5)
app = Flask(__name__)

# input is the left side dial

@app.route("/input/<colour>/<movement>")
def input(colour, movement):
    ser.write('0,{0},{1}\n' .format(colour, movement))
    return "Done!"

@app.route("/output/<colour>/<movement>")
def output(colour, movement):
    ser.write('1,{0},{1}\n' .format(colour, movement))
    return "Done!"

if __name__ == "__main__":
    app.run()
```

[gist](https://gist.github.com/bassdread/1a7d49adebf288c4684b)

Below is a better quality selection of pictures of it in action.

<blockquote class="imgur-embed-pub" lang="en" data-id="a/kc8Tq"><a href="//imgur.com/kc8Tq">Analog Monitors with an Arduino Twist</a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>