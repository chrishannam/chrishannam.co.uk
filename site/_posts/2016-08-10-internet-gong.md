---
layout: post
title:  "Internet Gong"
date:   2016-08-10 12:16:54 +0000
categories: gong arduino
---
A while back I wanted something that made a noise to notify me of an event. The original plan was to replicate Andy Rubin’s gong doorbell. The large gong and mallet was a little bit out of my price range but a good idea is a good idea. I ordered one from Amazon and combined with a servo and a Arduino Nano clone I set about the same idea.

Gong
![alt text](/assets/2016/08/internet-gong.jpg "Gong.")

With a little bit of tape it was finished. The design is basic but functional. The beater is attached to a 9g servo which acts as the human arm. An Arduino Nano clone is used to to move the servo.

In action:
<iframe width="560" height="315" src="https://www.youtube.com/embed/-vFhrOvHxIs" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

# Triggering the Gong
Communication to the gong is done over serial. Sending “gong\n” over serial, triggers the beater/mallet to strike the gong and then move out the way to avoid a second gong strike. The code for the Arduino is below:

```c
#include <Servo.h>

Servo myservo;

String inputString = "";         // a string to hold incoming data
boolean stringComplete = false;  // whether the string is complete

void setup()
{

  myservo.attach(2);  // attaches the servo on pin 2 to the servo object
  myservo.write(90);
  Serial.begin(9600);
  // reserve 200 bytes for the inputString:
  inputString.reserve(200);
}


void loop()
{
  if (stringComplete && inputString == "gong\n") {
    Serial.println(inputString);
    myservo.write(140);
    delay(100);
    myservo.write(40);
    delay(5000);
    myservo.write(90);

    // clear the string ready for another go
    inputString = "";
    stringComplete = false;
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
```

# Conclusion
For a mini project this is perfect for getting servos and Arduinos to work together. Mine is currently connected to my Google calendar to alert me five minutes before
a meeting.