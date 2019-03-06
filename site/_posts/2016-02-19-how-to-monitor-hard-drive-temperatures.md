---
layout: post
title:  "How to Monitor Hard Drive Temperatures"
date:   2016-02-19 12:16:54 +0000
categories: sensors python fans harddrives temperatures
---
Monitoring hard drives is pretty similar to the work I touched on for onboard sensors. First we need the right tool for the job. In this case it’s smartctl.

smartctl is available in the `smartmontools` package for Ubuntu smartmontools. To install:

```bash
sudo apt-get install smartmontools
```
Once installed you will need to use sudo smartctl to test it.

By default the command outputs a lot of very useful information. Checkout a disk for example using the following:

```bash
$ sudo smartctl --all /dev/sda -s on
smartctl 6.2 2013-07-26 r3841 [x86_64-linux-3.16.0-60-generic] (local build)
Copyright (C) 2002-13, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Model Family:     Western Digital AV SATA
Device Model:     WDC WD2500AVJS-63B6A0
Serial Number:    WD-WCAT10290663
LU WWN Device Id: 5 0014ee 1563ee704
Firmware Version: 01.03A01
User Capacity:    250,059,350,016 bytes [250 GB]
Sector Size:      512 bytes logical/physical
Device is:        In smartctl database [for details use: -P show]
ATA Version is:   ATA8-ACS (minor revision not indicated)
SATA Version is:  SATA 2.5, 3.0 Gb/s
Local Time is:    Thu Feb 18 12:45:29 2016 GMT
SMART support is: Available - device has SMART capability.
SMART support is: Enabled

=== START OF ENABLE/DISABLE COMMANDS SECTION ===
SMART Enabled.

=== START OF READ SMART DATA SECTION ===
SMART overall-health self-assessment test result: PASSED

General SMART Values:
Offline data collection status:  (0x00)	Offline data collection activity
					was never started.
					Auto Offline Data Collection: Disabled.
Self-test execution status:      (   0)	The previous self-test routine completed
					without error or no self-test has ever
					been run.
Total time to complete Offline
data collection: 		( 5400) seconds.
Offline data collection
capabilities: 			 (0x7b) SMART execute Offline immediate.
					Auto Offline data collection on/off support.
					Suspend Offline collection upon new
					command.
					Offline surface scan supported.
					Self-test supported.
					Conveyance Self-test supported.
					Selective Self-test supported.
SMART capabilities:            (0x0003)	Saves SMART data before entering
					power-saving mode.
					Supports SMART auto save timer.
Error logging capability:        (0x01)	Error logging supported.
					General Purpose Logging supported.
Short self-test routine
recommended polling time: 	 (   2) minutes.
Extended self-test routine
recommended polling time: 	 (  66) minutes.
Conveyance self-test routine
recommended polling time: 	 (   5) minutes.
SCT capabilities: 	       (0x303f)	SCT Status supported.
					SCT Error Recovery Control supported.
					SCT Feature Control supported.
					SCT Data Table supported.

SMART Attributes Data Structure revision number: 16
Vendor Specific SMART Attributes with Thresholds:
ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
  1 Raw_Read_Error_Rate     0x002f   200   200   051    Pre-fail  Always       -       0
  3 Spin_Up_Time            0x0027   155   151   021    Pre-fail  Always       -       3241
  4 Start_Stop_Count        0x0032   099   099   000    Old_age   Always       -       1925
  5 Reallocated_Sector_Ct   0x0033   200   200   140    Pre-fail  Always       -       0
  7 Seek_Error_Rate         0x002e   200   200   000    Old_age   Always       -       0
  9 Power_On_Hours          0x0032   071   071   000    Old_age   Always       -       21172
 10 Spin_Retry_Count        0x0032   100   100   000    Old_age   Always       -       0
 11 Calibration_Retry_Count 0x0032   100   100   000    Old_age   Always       -       0
 12 Power_Cycle_Count       0x0032   099   099   000    Old_age   Always       -       1923
192 Power-Off_Retract_Count 0x0032   199   199   000    Old_age   Always       -       798
193 Load_Cycle_Count        0x0032   200   200   000    Old_age   Always       -       1925
194 Temperature_Celsius     0x0022   127   081   000    Old_age   Always       -       16
196 Reallocated_Event_Count 0x0032   199   199   000    Old_age   Always       -       1
197 Current_Pending_Sector  0x0032   200   200   000    Old_age   Always       -       0
198 Offline_Uncorrectable   0x0030   100   253   000    Old_age   Offline      -       0
199 UDMA_CRC_Error_Count    0x0032   200   200   000    Old_age   Always       -       0
200 Multi_Zone_Error_Rate   0x0008   100   253   000    Old_age   Offline      -       0

SMART Error Log Version: 1
No Errors Logged

SMART Self-test log structure revision number 1
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Short offline       Completed without error       00%     11791         -

SMART Selective self-test log data structure revision number 1
 SPAN  MIN_LBA  MAX_LBA  CURRENT_TEST_STATUS
    1        0        0  Not_testing
    2        0        0  Not_testing
    3        0        0  Not_testing
    4        0        0  Not_testing
    5        0        0  Not_testing
Selective self-test flags (0x0):
  After scanning selected spans, do NOT read-scan remainder of disk.
If Selective self-test is pending on power-up, resume after 0 minute delay.
```
Lots of info will appear, have a search for “Temperature_Celsius” to find the temperature in Celsius.

To make life simpler for collecting this information I use pySMART. This Python library makes processing the output from the command so much simpler. The script below extracts all temperatures for the disks stored in the dict DISKS. The pattern is device name (just the disk not the full path) and whatever you want to know the disk as.

I run the script from root’s cron as the smartctl command requires root privileges. Do the following to add it to root’s crontab. The example below will run the script every minute.

```bash
$ sudo -i
$ crontab -e
```

```
# add the following using your editor
* * * * * python /PATH/TO/SCRIPT/bin/read_disk_temps.py
```

```python
#!/usr/bin/python

import collections
from datetime import datetime
import json
from pySMART import Device
from traceback import print_exc

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

# map devices to mount points
DISKS = {
    'sda': 'root',
    'sdb': 'films',
    'sdc': 'backup',
    'sdd': 'tv'
}

DATA_FILE_LOCATION = '/tmp/disk_data.json'

USE_INFLUX = True
USE_REDIS = True


def fetch_sensor_data():

    try:
        data = {}
        for device, name in DISKS.iteritems():
            disk = Device('/dev/{}'.format(device))
            for attribute in disk.attributes:
                if not attribute:
                    continue

                if attribute.name == "Temperature_Celsius":
                    data[name] = float(attribute.raw)

        sorted_data = collections.OrderedDict(sorted(data.items()))

        write_data_file(sorted_data)

        if USE_REDIS:
            write_data_redis(data)

        if USE_INFLUX:
            write_data_influx(data)

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
        print("Failed to write to Redis. Error {}".format(exception.message))


def write_data_file(data):
    try:
        data['time'] = datetime.now().strftime("%Y-%m-%dT%H:%M:%S")
        with open(DATA_FILE_LOCATION, 'w') as data_file:
            data_file.write(json.dumps(data))
            data_file.close()

        return True
    except Exception as exception:
        print_exc()
        print("Failed to write to data file. Error {}".format(
            exception.message))


if __name__ == "__main__":
    fetch_sensor_data()
```