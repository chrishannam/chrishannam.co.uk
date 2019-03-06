---
layout: post
title:  "Environment Monitoring From Scratch"
date:   2016-11-28 12:16:54 +0000
categories: monitoring arduino
---
Mointoring your immediate enviroment is fairly simple witha few simple tools.

# Requirements

# ds18s20 Temperature Sensor
Download [OneWire](http://www.pjrc.com/teensy/arduino_libraries/OneWire.zip) and copy it to your libs directory. On OSX its the following:

```bash
wget http://www.pjrc.com/teensy/arduino_libraries/OneWire.zip
unzip OneWire.zip
mv OneWire ~/Documents/arduino/libraries/
```

You can test if this is installed correctly by restarting the Arduino IDE and you should see the following from the Sketch -> Include Library from the top menu.

Next grab the drivers for the light sensor, I am using a TSL2561.
```bash
wget https://github.com/adafruit/Adafruit_Sensor/archive/1.0.2.zip
unzip Adafruit_Sensor-1.0.2.zip
mv Adafruit_Sensor-1.0.2 ~/Documents/arduino/libraries/Adafruit_Sensor
wget https://github.com/adafruit/Adafruit_TSL2561/archive/1.0.0.zip
unzip Adafruit_TSL2561-1.0.0.zip
mv Adafruit_TSL2561-1.0.0 ~/Documents/arduino/libraries/Adafruit_TSL2561
```
You can test these are installed correctly as they will appear in the “Recommended libraries” section in the screenshot above once you restart the Arduino IDE.

The following code is mostly from [OneWire Code](http://playground.arduino.cc/Learning/OneWire), I have had to make few really minor changes.

The code below is all will read from the two sensors and output JSON over the serial port.

```c
#include <OneWire.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_TSL2561_U.h>

int DS18S20_Pin = 3;

OneWire ds(DS18S20_Pin);
Adafruit_TSL2561_Unified tsl = Adafruit_TSL2561_Unified(TSL2561_ADDR_FLOAT, 12345);

void setup() {
  // put your setup code here, to run once:
  Serial.begin(9600);

  if(!tsl.begin())
  {
    /* There was a problem detecting the ADXL345 ... check your connections */
    Serial.print("Ooops, no TSL2561 detected ... Check your wiring or I2C ADDR!");
    while(1);
  }
}

void loop() {
  float temperature = getTemperature();

  /* Get a new sensor event */
  sensors_event_t event;
  tsl.getEvent(&event);

  Serial.print("{\"temperature\":");
  Serial.print(temperature);
  Serial.print(", \"light\":");
  Serial.print(event.light);
  Serial.print("}\n");
  delay(5000);
}



/**************************************************************************/
/*
    Configures the gain and integration time for the TSL2561
*/
/**************************************************************************/
void configureSensor(void)
{
  /* You can also manually set the gain or enable auto-gain support */
  // tsl.setGain(TSL2561_GAIN_1X);      /* No gain ... use in bright light to avoid sensor saturation */
  // tsl.setGain(TSL2561_GAIN_16X);     /* 16x gain ... use in low light to boost sensitivity */
  //tsl.enableAutoGain(true);          /* Auto-gain ... switches automatically between 1x and 16x */

  /* Changing the integration time gives you better sensor resolution (402ms = 16-bit data) */
  tsl.setIntegrationTime(TSL2561_INTEGRATIONTIME_13MS);      /* fast but low resolution */
  //tsl.setIntegrationTime(TSL2561_INTEGRATIONTIME_101MS);  /* medium resolution and speed   */
  //tsl.setIntegrationTime(TSL2561_INTEGRATIONTIME_402MS);  /* 16-bit data but slowest conversions */
}

float getTemperature() {
  //returns the temperature from one DS18S20 in DEG Celsius

  byte data[12];
  byte addr[8];

  if ( !ds.search(addr)) {
    //no more sensors on chain, reset search
    ds.reset_search();
    return -1000;
  }

  if ( OneWire::crc8( addr, 7) != addr[7]) {
    Serial.println("CRC is not valid!");
    return -1000;
  }

  if ( addr[0] != 0x10 && addr[0] != 0x28) {
    Serial.print("Device is not recognized");
    return -1000;
  }

  ds.reset();
  ds.select(addr);
  ds.write(0x44, 1); // start conversion, with parasite power on at the end

  byte present = ds.reset();
  ds.select(addr);
  ds.write(0xBE); // Read Scratchpad


  for (int i = 0; i < 9; i++) { // we need 9 bytes
    data[i] = ds.read();
  }

  ds.reset_search();

  byte MSB = data[1];
  byte LSB = data[0];

  float tempRead = ((MSB << 8) | LSB); //using two's compliment
  float TemperatureSum = tempRead / 16;

  return TemperatureSum;

}
```