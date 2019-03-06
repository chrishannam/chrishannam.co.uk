---
layout: post
title:  "How to Monitor Hard Drive SMART Status on Linux"
date:   2016-02-20 12:16:54 +0000
categories: sensors python fans harddrives temperatures smart
---
Following on monitoring hard drive temperatures I thought I would add a check for the current SMART assessment on the status of the drive. The pySMART lib makes this trivial. The code below will output the status.

As a failure could seriously impact the life expectancy of the drive.

```python
#!/usr/bin/python

import collections
from datetime import datetime
import json
import sys

try:
    from pySMART import Device
except ImportError:
    print """You will need to install pySMART from
https://pypi.python.org/pypi/pySMART to access
smartctl stats."""
    sys.exit(1)

import requests
from traceback import print_exc


# map devices to mount points
DISKS = {
    'sda': 'root',
    'sdb': 'films',
    'sdc': 'backup',
    'sdd': 'tv'
}

DATA_FILE_LOCATION = '/tmp/disk_assessment.json'

def test_disks():

    try:
        data = {}
        failed_disks = []
        for device, name in DISKS.iteritems():
            disk = Device('/dev/{}'.format(device))

            data[name] = disk.assessment

            # log failed ones so we can alert
            if disk.assessment != 'PASS':
                failed_disks.append(device)

        sorted_data = collections.OrderedDict(sorted(data.items()))

        write_data_file(sorted_data)

        for name, reading in sorted_data.iteritems():
            print("{0}: {1}".format(name, reading))

    except Exception as exception:
        print_exc()
        print("Failed to get sensor data: {0}".format(exception.message))


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
    test_disks()
```