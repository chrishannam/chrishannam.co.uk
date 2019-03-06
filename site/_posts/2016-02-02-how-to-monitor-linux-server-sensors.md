---
layout: post
title:  "How to Monitor Linux Server Sensors"
date:   2016-02-02 12:16:54 +0000
categories: sensors python fans
---
Following on from my last post on monitoring fan speed I found PySensors.This library providers a simple method for extracting data from the sensors command. Below shows the basic usage on my server:

```
$ python
Python 2.7.6 (default, Jun 22 2015, 17:58:13)
[GCC 4.8.2] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import sensors
>>> sensors.init()
>>> try:
...     for chip in sensors.iter_detected_chips():
...         print '%s at %s' % (chip, chip.adapter_name)
...         for feature in chip:
...             print '  %s: %.2f' % (feature.label, feature.get_value())
... finally:
...     sensors.cleanup()
...
acpitz-virtual-0 at Virtual device
  temp1: 27.80
  temp2: 29.80
coretemp-isa-0000 at ISA adapter
  Physical id 0: 16.00
  Core 0: 12.00
  Core 1: 15.00
nct6791-isa-0290 at ISA adapter
  in0: 0.88
  in1: 1.01
  in2: 3.30
  in3: 3.30
  in4: 1.01
  in5: 2.02
  in6: 0.64
  in7: 3.44
  in8: 3.25
  in9: 1.01
  in10: 0.21
  in11: 0.16
  in12: 1.01
  in13: 1.01
  in14: 0.19
  fan1: 0.00
  fan2: 1016.00
  fan3: 0.00
  fan4: 0.00
  fan5: 0.00
  SYSTIN: 7.00
  CPUTIN: 10.00
  AUXTIN0: 45.00
  AUXTIN1: 112.00
  AUXTIN2: 110.00
  AUXTIN3: 110.00
  PECI Agent 0: 15.50
  PCH_CHIP_CPU_MAX_TEMP: 0.00
  PCH_CHIP_TEMP: 0.00
  PCH_CPU_TEMP: 0.00
  intrusion0: 1.00
  intrusion1: 1.00
  beep_enable: 0.00
>>>
```
Extending the script from the last article, it’s now simple to record all the sensor data shown here on Grafana. I have updated the script and listed it below.

Temperature Graph from Main Board

<img src="/assets/2016/02/fan-speed-temps.png" alt="Temperature and Fan Speeds">

```python
#!/usr/bin/python

import collections
from datetime import datetime
import json
# https://pypi.python.org/pypi/PySensors/
# tl;dr: "sudo pip install pysensors"
import sensors
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

IGNORE_READINGS = ['fan1', 'fan3', 'fan4', 'fan5']
DATA_FILE_LOCATION = '/tmp/sensors_output.json'

USE_INFLUX = False
USE_REDIS = False


def fetch_sensor_data():

    try:
        sensors.init()
        data = {}

        for chip in sensors.iter_detected_chips():
            for feature in chip:
                # log stuff we care about
                if feature.label not in IGNORE_READINGS:
                    data[feature.label] = round(feature.get_value(), 3)

        sorted_data = collections.OrderedDict(sorted(data.items()))

        write_data_file(sorted_data)

        if USE_REDIS:
            write_data_redis(sorted_data)

        if USE_INFLUX:
            write_data_influx(sorted_data)

        for name, reading in sorted_data.iteritems():
            print "{0}: {1}".format(name, reading)

    except Exception as exception:
        print_exc()
        print("Failed to get sensor data: {0}".format(exception.message))


def write_data_influx(data):
    json_payload = []
    time = datetime.now().strftime("%Y-%m-%dT%H:%M:%S")
    try:
        for name, value in data.iteritems():
            payload = {
                "measurement": name,
                "tags": {
                    "host": "jarvis",
                    "location": "garage"
                },
                "time": time,
                "fields": {
                    "value": value
                }
            }
            json_payload.append(payload)
        INFLUX_CLIENT.write_points(json_payload)
    except Exception as exception:
        print_exc()
        print("Failed to write to Influx. Error {}".format(exception.message))


def write_data_redis(data):
    try:
        for name, value in data.iteritems():
            REDIS_CONNECTION.set(name, value)
    except Exception as exception:
        print_exc()
        print ("Failed to write to Redis. Error {}".format(exception.message)


def write_data_file(data):
    try:
        data['time'] = datetime.now().strftime("%Y-%m-%dT%H:%M:%S")
        with open(DATA_FILE_LOCATION, 'w') as data_file:
            data_file.write(json.dumps(data))
            data_file.close()

        return True
    except Exception as exception:
        print_exc()
        print ("Failed to write to data file. Error {}".format(
            exception.message))


if __name__ == "__main__":
    fetch_sensor_data()
```
