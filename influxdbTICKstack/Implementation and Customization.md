
My journey into using a reletavily new and evolving timeseries database to gather, process and visualize numerous sensor and system metrics. It is also a journey to further improve my usage and knowledge of containers for agility in configuration and deployment and ulitmitly robustness


# Functional Requirement.

Take time series data from a number of diverse sensors and systems, normalize, downsample, analyse, alert, vizualize and administer.

## Ideals
- containered
- broad platform support, 
    * cloud, 
    * synology NAS, 
    * on site Servers
    * emmbedded platforms
- scalable
- security
- opensource

## Options

- Traditional SQL - Mysql / PostgresSQL not ideally suited to Timeseries data
- Prometheus - Limited support for strings, proved to be the my disqualifier
- InfluxDb - Strings are supported as values

## Options
InfluxDb was chosen on the basis of strings being supported as equal to numeric metrics, a seeminlgy good ecosystem of modules in what is refered to as the TICKstack. The updated (Nov17) Sandbox worked straight away on my Synology NAS DS916+ im a virtual DSM, Docker 1.11, Grafana would be used as a comparison with the influx Chronograf for visualization


## Implementation
The [InfluxDB sandbox](https://github.com/influxdata/sandbox) is the fast way to spin up 5 containers hosting all the influxDb modules, refered to as the TICKstack. It worked straightaway however there was a lot to learn and about environement access, what tools to use to etc. The Chronograph interface is well advanced for much of the configuration and adjusting, but you soon want to get into the CLI and logs. 

In the sandbox tearing down and recreating is quick and straightforward, if set correctly data is retained. See Gotchas (Below)




### References:
> https://www.influxdata.com/blog/why-build-a-time-series-data-platform/

> https://github.com/influxdata/sandbox

> https://www.influxdata.com/time-series-platform/influxdb/

> Raspberry Pi InfluxDB: The solution for IoT Data storage
Raspberry Pi is costeffect linux computer very commonly used for IoT home automation projects. https://gist.github.com/boseji/bb71910d43283a1b84ab200bcce43c26




# Requirement : Edge Detection (Stage Change)
Reduce records to stage changes, Where a device is polled for its data, amongst the numeric records are also state records often as strings.
### **Problem**
 In the timeseries data being gathered I have an ADSL router that I want to detect when its state changes from being online 'SHOWTIME' to offline 'TRAINING'. Every few minutes it is polled via an SSH session and the `show adsl status` result is returned for scrapping all line quality metrics and the connections status `SHOWTIME | TRAINING`
This data is passed by a python influxdb client to influxdb. The connection data is numerous records of `SHOWTIME` with sporadic `TRAINING` records

THis Table shows what I would like to have

|Metric,Tag | Value | TIMESTAMP |
|-----------|----------| --- |
| adsl_connection,host=router1 | value="SHOWTIME" | 1514979051065000000 |
| adsl_connection,host=router1 | value="TRAINING" | 1514979141070000000 |
| adsl_connection,host=router1 | value="SHOWTIME" | 1514979231065000000 |
| adsl_connection,host=router1 | value="TRAINING" | 1514979681040000000 |
| adsl_connection,host=router1 | value="SHOWTIME" | 1514980041020000000 |
| adsl_connection,host=router1 | value="TRAINING" | 1514980221010000000 |

This however is a mockup of the actual data being logged to influxdb, THe bolded timestamps are the changed records

|Metric,Tag | Value | TIMESTAMP |
|-----------|----------| --- |
| adsl_connection,host=router1 | value="SHOWTIME" | **1514979051065000000** |
| adsl_connection,host=router1 | value="TRAINING" | **1514979141070000000** |
| adsl_connection,host=router1 | value="SHOWTIME" | **1514979231065000000** |
| adsl_connection,host=router1 | value="SHOWTIME" | ~~~1514979321060000000~~~ |
| adsl_connection,host=router1 | value="SHOWTIME" | ~~~1514979411055000000~~~|
| adsl_connection,host=router1 | value="SHOWTIME" | ~~~1514979501050000000~~~ |
| adsl_connection,host=router1 | value="SHOWTIME" | ~~~1514979591045000000~~~ |
| adsl_connection,host=router1 | value="TRAINING" | **1514979681040000000** |
| adsl_connection,host=router1 | value="TRAINING" | ~~~1514979771035000000~~~ |
| adsl_connection,host=router1 | value="TRAINING" | ~~~1514979861030000000~~~ |
| adsl_connection,host=router1 | value="TRAINING" | ~~~1514979951025000000~~~ |
| adsl_connection,host=router1 | value="SHOWTIME" | **1514980041020000000** |
| adsl_connection,host=router1 | value="SHOWTIME" | ~~~1514980131015000000~~~ |
| adsl_connection,host=router1 | value="TRAINING" | **1514980221010000000** |
| adsl_connection,host=router1 | value="TRAINING" | ~~~1514980311005000000~~~ |
| adsl_connection,host=router1 | value="TRAINING" | ~~~1514980401000000000~~~ |
| adsl_connection,host=router1 | value="TRAINING" | ~~~1514980491000000000~~~ |

So how do we get there. We need to our short changes only table. We require some form of edge/state detection function, compare the current state to the last know state.

A bit of research shows this is an identified list of functions to be incorporated.


### ISSUES/ References
[FunctionReferences]:References 
| Issue | Details |Notes |  
|---|---|---|
|#5930 | [[[feature collection]] requested Functions and query operators ](https://github.com/influxdata/influxdb/issues/5930) | *Wishlist of downsampling and analysis functions* |
|#643 | [Mechanism for edge detection in series #643](https://github.com/influxdata/kapacitor/issues/643) | 8 Dec17 [Marked for Kapacitor 1.5]

|  **Filling in the Gaps - USER DEFINED FUNCTIONS** |
| --|
| Either add directly to the source code of Kapacitor/Influx or extend via the UDF options. I am presently unfamiliar with GO and the architechture of Kapacitor source so it will be quickest to realize a solution utilizing UDF's |

Looking for alternatives till the influxdb team, Kapacitors lead developer #nathanielc or someone skilled in GO submitts an PR? I have investigate and succeeded in doing a UDF, The UDF documentation is pretty good with support for python

As the TICKstack is running in DockerContainers, I did not want to alter the Docker builds to include Python so I have created a custom DockerContainer for Python based Kapacitor UDF's using unix sockets. 
Looks like I will need an individual container for each function I create, although it looks like it would be possible to run a multithreaded UDF service handling multiple UDF's. 

Using the Mirror example and the Moving AVG example I have cludged together a first pass at an EdgeDetect UDF
extending via udf's
    As docker, so udf's python needs to run in its own container

### ISSUES/ References
[FunctionReferences]:References 
| Issue | Details |Notes |  
|---|---|---|
| | Kapacitor - [Custom Anomaly Detection](https://docs.influxdata.com/kapacitor/v1.4/guides/anomaly_detection/) | 3D printing temperature monitoring  |
| | Kapacitor [Writing a socket based UDF]
||(https://docs.influxdata.com/kapacitor/v1.4/guides/socket_udf/)|
||https://github.com/influxdata/kapacitor/tree/master/udf/agent/examples/mirror | Mirror Stream UDF example |
||https://github.com/influxdata/kapacitor/tree/master/udf/agent/examples/moving_avg| Moving Averege UDF example |



# UDF Requirement : Edge Detection (Stage Change)
Reduce records to stage changes, Where a device is polled for its data, amongst the numeric records are also state records often as strings. These records are all passed as one timestamp entry into influxdb through telegraph or by direct interfaces, http  POST or specific language client. Python has a module influxdb 

### Implementation

My implementation is on a Synology NAS in a virtual DSM  testbed

```
ssh {username}@{SynlogoyNAS} -p 22
cd /volume1/{Dev Directory}

git clone https://github.com/auphofBSF/TimeSeriesData-with-influxDB.git

sudo docker-com

```






# DATAfeeder to InfluxDB

Written in python, running python 2.7 in a virtual environemnt (I use Annaconda on Windows, with PyCharm and or VScode) 

    pip install influxdb

in python

    from influxdb import InfluxDBClient



# Environment for developing


[Kapacitor API Documentation, - Recordings](https://github.com/influxdata/kapacitor/blob/778d644b824dfd33d880ac8244ed93775065f274/client/API.md#recordings)

[Requesting easier method to test scripts than recordings #1341
](
    https://github.com/influxdata/kapacitor/issues/1341)  see Pestana's interesting work on unit Tests, Something to tryout

# Gotcha's

- Kapacitor wont start if UDF's not present
identify by 

ISSUE: ?

    docker start -i sanbox_kapacitor1

- UDF's wont start if unix socket file present
    rm .....{udf}.sock
    
Maybe related [Recover from socket-based UDF connection failure? #1174](https://github.com/influxdata/kapacitor/issues/1174)

# Interesting work to watch
| | |
|-|-|
|Gonçalo​ ​Pestana's Blog entry [kapacitor-unit: unit tests framework for TICKscripts](https://www.gpestana.com/blog/kapacitor-unit/) | github.com/gpestana/kapacitor-unit|

