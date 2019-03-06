---
layout: post
title:  "Server Fan Monitoring"
date:   2016-02-01 12:16:54 +0000
categories: sensors python fans
---
My biggest concern relocating my server to the garage was dust clogging up the fans. The server has two fans, one CPU and one mounted at the rear of the case.

The best guide for reading sensor data I have found for Ubuntu is Sensors How To. This guides you through setting up the command sensors to access the hardware sensors located on the motherboard.

I am using an ASUS main board and by default sensors wont pick up the fans attached to the board. A bit of Googling found the solution installing the correct kernel module.

```
sudo modprobe nct6775
```
Make sure you add `nct6775` to `/etc/modules` to ensure it’s loaded at boot time.

sensors command now gives the following output:
```
$ sensors
acpitz-virtual-0
Adapter: Virtual device
temp1:        +27.8°C  (crit = +105.0°C)
temp2:        +29.8°C  (crit = +105.0°C)

coretemp-isa-0000
Adapter: ISA adapter
Physical id 0:  +16.0°C  (high = +80.0°C, crit = +100.0°C)
Core 0:         +15.0°C  (high = +80.0°C, crit = +100.0°C)
Core 1:         +16.0°C  (high = +80.0°C, crit = +100.0°C)

nct6791-isa-0290
Adapter: ISA adapter
in0:                    +0.88 V  (min =  +0.00 V, max =  +1.74 V)
in1:                    +1.01 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
in2:                    +3.31 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
in3:                    +3.30 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
in4:                    +1.01 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
in5:                    +2.02 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
in6:                    +0.64 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
in7:                    +3.44 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
in8:                    +3.25 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
in9:                    +1.01 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
in10:                   +0.21 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
in11:                   +0.16 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
in12:                   +1.01 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
in13:                   +1.01 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
in14:                   +0.20 V  (min =  +0.00 V, max =  +0.00 V)  ALARM
fan1:                     0 RPM  (min =    0 RPM)
fan2:                  1008 RPM  (min =    0 RPM)
fan3:                     0 RPM  (min =    0 RPM)
fan4:                     0 RPM  (min =    0 RPM)
fan5:                     0 RPM  (min =    0 RPM)
SYSTIN:                  +9.0°C  (high =  +0.0°C, hyst =  +0.0°C)  ALARM  sensor = thermistor
CPUTIN:                 +11.5°C  (high = +80.0°C, hyst = +75.0°C)  sensor = thermistor
AUXTIN0:                +47.0°C    sensor = thermistor
AUXTIN1:               +111.0°C    sensor = thermistor
AUXTIN2:               +109.0°C    sensor = thermistor
AUXTIN3:               +110.0°C    sensor = thermistor
PECI Agent 0:           +15.5°C
PCH_CHIP_CPU_MAX_TEMP:   +0.0°C
PCH_CHIP_TEMP:           +0.0°C
PCH_CPU_TEMP:            +0.0°C
intrusion0:            ALARM
intrusion1:            ALARM
beep_enable:           disabled
```

Only the case fan is reporting, it’s fan2 on the list. I’m not sure why only one fan is reported, still better than none…

Using the script at the bottom I add the data into an InfluxDB database and use Grafana to view it. Below is a sample graph:

<img src="/assets/2016/02/fan-speed.png" alt="Fan Speed">

This is some basic graphing that allows me to track any changes that might indicate a problem with the fan.


Below is a script to output the fan data from the above output [gist](https://gist.github.com/chrishannam/aa47f263a1193d08138fc94a56a3fb96):
```python
""" Takes output from "sensors" command and writes it to
a file by default. Can also connect to Redis and Influxdb
"""

#!/usr/bin/python

import subprocess
import json
from datetime import datetime
from traceback import print_exc

# you can install these modules with pip if needed
try:
  import redis
  REDIS_CONNECTION = redis.StrictRedis(host='localhost', port=6379, db=0)
except ImportError:
  pass

try:
  from influxdb import InfluxDBClient
  INFLUX_CLIENT = InfluxDBClient('localhost', 8086, 'USERNAME', 'PASSWORD', 'DATABASE')
except ImportError:
  pass

IGNORE_FANS = ['fan1', 'fan3', 'fan4', 'fan5']
DATA_FILE_LOCATION = '/tmp/fan_speeds.json'

# make True to update a influxdb server
USE_INFLUX = False

# make True to update a redis server
USE_REDIS = False


def fetch_sensor_data():

    try:
        proc = subprocess.Popen(['sensors'], stdout=subprocess.PIPE,
                                close_fds=True)
        results = proc.communicate()[0]
        fans = {}

        for line in results.split("\n"):
            fan = fetch_fan(line)
            if fan:
                fans[fan.get('name', 'NO_NAME')] = float(fan.get('value', 0))

        for fan, speed in fans.iteritems():
            print("{0} currently at {1}RPM".format(fan, speed))

        write_data_file(fans)

        if USE_REDIS:
            write_data_redis(fans)

        if USE_INFLUX:
            write_data_influx(fans)

    except Exception as exception:
        print_exc()
        print("Failed to get sensor data: {0}".format(exception.message))


def write_data_influx(fans):
    json_payload = []
    time = datetime.now().strftime("%Y-%m-%dT%H:%M:%S")
    try:
        for fan, speed in fans.iteritems():
            payload = {
                "measurement": fan,
                "tags": {
                    "host": "jarvis",
                    "location": "garage"
                },
                "time": time,
                "fields": {
                    "value": speed
                }
            }
            json_payload.append(payload)
        INFLUX_CLIENT.write_points(json_payload)
    except Exception as exception:
        print_exc()
        print("Failed to write to Influx. Error {}".format(exception.message))


def write_data_redis(fans):
    try:
        for fan, speed in fans.iteritems():
            REDIS_CONNECTION.set(fan, speed)
    except Exception as exception:
        print_exc()
        print("Failed to write to Redis. Error {}".format(exception.message))


def write_data_file(fans):
    try:
        fans['time'] = datetime.now().strftime("%Y-%m-%dT%H:%M:%S")
        with open(DATA_FILE_LOCATION, 'w') as fan_speed_file:
            fan_speed_json = fan_speed_file.write(json.dumps(fans))
            fan_speed_file.close()
    except Exception as exception:
        print_exc()
        print("Failed to write to data file. Error {}".format(
            exception.message))


def fetch_fan(line):
    if not line.startswith('fan'):
        return False

    result = {}

    try:
        name, speed = line.split(':')

        if name not in IGNORE_FANS:
            result['name'] = name
            result['value'] = speed.split('RPM')[0].strip()
    except Exception as exception:
        print_exc()
        print ("Failed to get fan data from sensor output")
        return False
    return result

if __name__ == "__main__":
    fetch_sensor_data()
```