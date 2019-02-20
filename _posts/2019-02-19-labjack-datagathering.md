---
layout: post
title:  "LabJack LJMs and automated data collection"
date:   2019-02-19
excerpt: "Determining torsion constants is easy if you work really hard for it."
project: true
tag:
- python 
- labjack
- parallel
---

## Introduction
For the last few months, I've been working with one of my former professors to try to develop a method of collecting data from a [magnetically dampened torsion pendulum][0] in some kind of reliable and scalable way. We've been using [Labjack T7 Pros][1] due to their versatility, scalability, and open-source libraries; this arrangement has only been hampered by the fact that developing scalable data collection functionality requires a lot of work from the ground up.

Effectively, our needs were as follows:

1. Easily or automatically connect to a T7 and handle stream opening/closing nuances
2. Given the Labjack device, network connection, and hosting computer, determine the maximum frequency of data scans, and how many scans should be fit into a packet.
3. Catch and recover from any kind of interruption or error.
4. Allow the recorded data to be accessible for realtime backup and analysis.

These requirements led to [labjack-controller][2], a Python streaming API that we will be presenting at the [March 2019 APS meeting][3].

## Implementation Details

### Issues of the Base Library

Labjack provides a base [LJM library][4], and a Python wrapper. These tools are fairly powerful, which is fantastic, but they require a significant amount of work to set up basic functionality. For instance, here is an abbreviated demo file that Labjack gives out to show how to do basic data streaming:

```python
from datetime import datetime
import sys

from labjack import ljm


MAX_REQUESTS = 25  # The number of eStreamRead calls that will be performed.

# Open first found LabJack
handle = ljm.openS("T7", "ETHERNET", "ANY")  # T7 device, Ethernet connection, Any identifier

# Get device type, connection type, serial number, IP, port, max bytes / MB
info = ljm.getHandleInfo(handle)
deviceType = info[0]

# Stream Configuration
aScanListNames = ["AIN0", "AIN1"]  # Scan list names to stream
numAddresses = len(aScanListNames)
aScanList = ljm.namesToAddresses(numAddresses, aScanListNames)[0]
scanRate = 1000
scansPerRead = int(scanRate / 2)

try:
    # When streaming, negative channels and ranges can be configured for
    # individual analog inputs, but the stream has only one settling time and
    # resolution.

    # LabJack T7 and other devices configuration

    # Ensure triggered stream is disabled.
    ljm.eWriteName(handle, "STREAM_TRIGGER_INDEX", 0)

    # Enabling internally-clocked stream.
    ljm.eWriteName(handle, "STREAM_CLOCK_SOURCE", 0)

    # All negative channels are single-ended, AIN0 and AIN1 ranges are
    # +/-10 V, stream settling is 0 (default) and stream resolution index
    # is 0 (default).
    aNames = ["AIN_ALL_NEGATIVE_CH", "AIN0_RANGE", "AIN1_RANGE",
                "STREAM_SETTLING_US", "STREAM_RESOLUTION_INDEX"]
    aValues = [ljm.constants.GND, 10.0, 10.0, 0, 0]
    # Write the analog inputs' negative channels (when applicable), ranges,
    # stream settling time and stream resolution configuration.
    numFrames = len(aNames)
    ljm.eWriteNames(handle, numFrames, aNames, aValues)

    # Configure and start stream
    scanRate = ljm.eStreamStart(handle, scansPerRead, numAddresses, aScanList, scanRate)
    print("\nStream started with a scan rate of %0.0f Hz." % scanRate)

    start = datetime.now()
    totScans = 0
    totSkip = 0  # Total skipped samples

    i = 1
    while i <= MAX_REQUESTS:
        ret = ljm.eStreamRead(handle)

        aData = ret[0]
        scans = len(aData) / numAddresses
        totScans += scans

        # Count the skipped samples which are indicated by -9999 values. Missed
        # samples occur after a device's stream buffer overflows and are
        # reported after auto-recover mode ends.
        curSkip = aData.count(-9999.0)
        totSkip += curSkip

        i += 1

    end = datetime.now()

    print("\nTotal scans = %i" % (totScans))
    tt = (end - start).seconds + float((end - start).microseconds) / 1000000
    print("Time taken = %f seconds" % (tt))
    print("LJM Scan Rate = %f scans/second" % (scanRate))
    print("Timed Scan Rate = %f scans/second" % (totScans / tt))
    print("Timed Sample Rate = %f samples/second" % (totScans * numAddresses / tt))
    print("Skipped scans = %0.0f" % (totSkip / numAddresses))
except ljm.LJMError:
    ljme = sys.exc_info()[1]
    print(ljme)
except Exception:
    e = sys.exc_info()[1]
    print(e)

try:
    print("\nStop Stream")
    ljm.eStreamStop(handle)
except ljm.LJMError:
    ljme = sys.exc_info()[1]
    print(ljme)
except Exception:
    e = sys.exc_info()[1]
    print(e)

# Close handle
ljm.close(handle)

```

As you can imagine, this makes the Labjack LJM devices difficult to deploy. How do you know what rates you can safely scan at? What is the importance of `STREAM_TRIGGER_INDEX`, `AIN_ALL_NEGATIVE_CH`, or `STREAM_SETTLING_US`? Making every developer who needs to use this library learn the meaning of these kinds of parameters and configurations leads to universally high time investment - time that could be spent in development.

Fortunately, it is this thoroughness of the base library that makes it possible to write an API to streamline this process.

### C Bindings of the Base Libraries

Labjack's implementation of the Python wrapper was to take the native C functions that talk to the device and implement functions in one to one correspondence. Unsuprisingly, this maintains the relative complexity of the the original C functions, and introduces overhead when performing some data-related tasks, such as in `ljm.eStreamRead`; this function returns a triple containing a data array that was once a C array that is converted into a Python list every single time it is called. In the long run, this has some cost; each conversion is likely a $$\Theta(n)$$ conversion, where $$n$$ is the size of each packet sent from the scan.

### Our Approach

We have attempted to group tasks logically. For the task above, we can treat it as the grouped steps

* Set up device
    * Convert names ("AIN0") to addresses
    * Disable stream triggering and clocked sources, if desired and applicable
    * Write all this information to the device

* Run the stream
    * Start stream
    * Loop through the stream scans, handling them fast enough s.t. no buffers overflow
    * Handle any skips or other errors.
    * Close the stream when done

In the interest of performance, we eschew the Python wrapper and deal directly with the base C library in order to minimize datatype conversions and allow for more robust error recovery.

### `labjack-controller` Syntax

The result of our design is exhibited when we replicate the function from above.

```python
from labjackcontroller.labtools import LabjackReader

duration = 30 # seconds, because why not
channels = ["AIN0", "AIN1"]
voltages = [10.0, 10.0]


# Instantiate a LabjackReader
my_lj = LabjackReader("T7", connection="ETHERNET")

# Actually collect data
data_proc = my_lj.collect_data(channels, voltages, duration, 1000, sample_rate=500)

# Bonus, print out our collected data.
print(my_lj.to_dataframe())
```

This kind of abstraction makes it far easier to advance our project. You can see an example of realtime data processing with realtime backup in parallel [here][5].

## Benchmarks

Our methodology is as follows:

1. Stream data s.t. a large volume of data is sent; we arbitrarily chose the values in the table below.
2. Observe the time between the device time and the time when data is recieved on the host computer.
3. Stream for a sufficiently long time as to get one million samples of this time delta.
4. Streaming is done on the same computer, with the same cables, and the same Labjack, on the same day.

| # Channels | Frequency | Scans / Read | Resolution Level |
|:----------:|:---------:|:------------:|:----------------:|
|     4      | 12000 Hz  |      32      |         0        |
|     4      | 12000 Hz  |      32      |         2        |
|     4      | 12000 Hz  |      32      |         4        |
|     4      | 12000 Hz  |      32      |         6        |
|     4      | 12000 Hz  |      32      |         8        |
|     1      | 40000 Hz  |      32      |         0        |
|     1      | 40000 Hz  |      32      |         2        |
|     1      | 40000 Hz  |      32      |         4        |
|     1      | 40000 Hz  |      32      |         6        |
|     1      | 40000 Hz  |      32      |         8        |


## Conclusion

We were able to achieve both the quality of life and performance improvements we desired. At this point, our major functionality is complete; in the future, we hope to expand the degree to which we process data in parallel. We are also considering adding other modes besides streaming, such as command-response mode.

If you would like to contribute to the project, create an issue on Github or submit a pull request. I'd love for this library to be the best it possibly can be.


[0]: http://www.paulnakroshis.net/blog/
[1]: https://labjack.com/products/t7
[2]: https://github.com/Nyctanthous/labjack-controller
[3]: http://meetings.aps.org/Meeting/MAR19/Session/X23.4
[4]: https://labjack.com/ljm
[5]: https://github.com/Nyctanthous/labjack-controller/blob/master/demos/Parallel%20Backup.ipynb