# Working with InfluxDB 3.x

In this workshop we will learn how to use the InfluxDB NoSQL database.

We assume that the platform described [here](../01-environment) is running and accessible. 

For this workshop we will be using an IoT device/sensor simulator available via the [MQTTX CLI](https://mqttx.app/docs/cli) command line client.

![](./images/architecture.png)

## Table of Contents

- [What you will learn](#what-you-will-learn)
- [Prerequisites](#prerequisites)
- [Running the Simulator and publish to MQTT](#running-the-simulator-and-publish-to-mqtt)
- [Using an MQTT Client to view messages](#using-an-mqtt-client-to-view-messages)
- [Using HiveMQ Web UI](#using-hivemq-web-ui)
- [Configure Telegraf with an access token](#configure-telegraf-with-an-access-token)
- [Using Telegraf to retrieve values from MQTT and store in InfluxDB](#using-telegraf-to-retrieve-values-from-mqtt-and-store-in-influxdb)
- [Querying Data with the InfluxDB 3.x CLI](#querying-data-with-the-influxdb-3x-cli)
- [Querying Data with the InfluxDB Explorer UI](#querying-data-with-the-influxdb-explorer-ui)
- [Working with InfluxDB from Python](#working-with-influxdb-from-python)

## What you will learn

- How to simulate IoT sensor data using the MQTTX CLI `smart_home` scenario
- How to view MQTT messages using a dockerized CLI client and the HiveMQ Web UI
- How to create an operator authentication token for InfluxDB 3.x
- How to configure Telegraf to consume JSON messages from an MQTT broker
- How to store time-series data from MQTT into InfluxDB 3.x using Telegraf's `influxdb_v3` output plugin
- How to query time-series data using the `influxdb3` CLI with SQL
- How to explore and query data visually using the InfluxDB Explorer UI
- How to connect Grafana to InfluxDB 3.x and build time-series dashboards

## Prerequisites

- The **Data Platform** described [here](../01-environment) is running and accessible

## Running the Simulator and publish to MQTT

The MQTT CLI is part of the platform we have started using docker compose. We can use it via `docker exec` command. 

The simulator comes with a few built-in scenarios. To list the available scenarios, in a terminal window execute the following:

```
docker exec -ti mqttx-cli mqttx ls --scenarios
```

and you should see a result similar to

```
~/w/platys-datahub>docker exec -ti mqttx-cli mqttx ls --scenarios                                                               1.303s 23:12
You can use any of the above scenario names as a parameter to run the scenario.
┌───────────────┬──────────────────────────────────────────────────────────────────────────────────────────────┐
│ Scenario Name │ Description                                                                                  │
├───────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
│ IEM           │ Simulation to generate Industrial Energy Monitoring data.                                    │
├───────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
│ smart_home    │ Simulation to generate Smart Home data.                                                      │
├───────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
│ tesla         │ Simulation to generate Tesla's data, reference form https://github.com/adriankumpf/teslamate │
├───────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
│ weather       │ Simulation to generate advanced weather station's data.                                      │
└───────────────┴──────────────────────────────────────────────────────────────────────────────────────────────┘
~/w/platys-datahub>
```

> **What you should see:** a table of four built-in scenarios including `smart_home`

we will be using the `smart_home` scenario. 

To run it, use the `simulate` option and specify with `conn` the MQTT broker to connect to. We are running `mosquitto` as part of the platform and this is the one we are connecting to.

```bash
docker exec -ti mqttx-cli mqttx simulate -sc smart_home -c 100 conn  -h 'mosquitto-1' -p 1883
```

> **What you should see:** the simulator runs silently; messages are being published to the `mqttx/simulate/#` topic at regular intervals

## Using an MQTT Client to view messages

For viewing the messages in MQTT, there are many options available.

In this workshop we will present two alternative options for consuming from MQTT

 * use dockerized MQTT client in the terminal
 * use browser-based HiveMQ Web UI

Using dockerized MQTT Client

To start consuming using through a command line, perform the following docker command from another terminal window:

```bash
docker run -it --network nosql-platform --rm efrecon/mqtt-client mosquitto_sub -h mosquitto-1 -p 1883 -t mqttx/simulate/#
```

The consumed messages will show up on the terminal window as shown below.

![](./images/mosquitto-sub.png)

> **What you should see:** a continuous stream of single-line JSON messages appearing in the terminal, one per simulated home per interval

## Using HiveMQ Web UI

To start consuming using the MQTT UI (HiveMQ Web UI), navigate to <http://dataplatform:28136> and connect using `dataplatform` for the Host field, `9101` for the Port field

![](./images/hiveMQ-1.png)

> **What you should see:** the HiveMQ Web Client login form

and click on Connect to connect to the broker.

When successfully connected, click on Add New Topic Subscription and enter `mqttx/simulate/#` into Topic field

![](./images/hiveMQ-2.png)

> **What you should see:** a topic subscription dialog with `mqttx/simulate/#` entered

and click Subscribe:

![](./images/hiveMQ-3.png)

> **What you should see:** messages flowing in the subscriptions panel in real time

You should again see the messages as they are being sent to MQTT.

Alternatively you can also use the [MQTTX Desktop](https://mqttx.app/downloads) version, available for installation on Mac or Windows.

In the subscription pattern of we have used `mqttx/simulate/#`, where  the `#` symbol is a wildcard used in topic subscriptions to match multiple levels in the topic hierarchy. It's known as the multi-level wildcard. It's important to note that # can only be used as the last character in a topic string, and only one # can be used in a single subscription.

If we check one of the messages, we can see that they are in JSON format, although all one one single line:

```json
{"home_id":"88a76b99-6e22-4771-90cd-aba57deb1015","owner_name":"Erik O'Connell","address":"518 Ullrich Mall","rooms":[{"room_type":"living room","temperature":20,"humidity":47,"lights_on":true,"window_open":false},{"room_type":"bedroom","temperature":22,"humidity":39,"lights_on":true,"window_open":false,"bed_occupancy":false},{"room_type":"kitchen","temperature":19,"humidity":34,"lights_on":true,"window_open":true,"fridge_temperature":7,"oven_on":true},{"room_type":"bathroom","temperature":24,"humidity":50,"lights_on":true,"window_open":true,"water_tap_running":true,"bath_water_level":91}],"timestamp":1714807284732}
```

if we "pretty-print" it then it is more visible

```json
{
   "home_id":"88a76b99-6e22-4771-90cd-aba57deb1015",
   "owner_name":"Erik O'Connell",
   "address":"518 Ullrich Mall",
   "rooms":[
      {
         "room_type":"living room",
         "temperature":20,
         "humidity":47,
         "lights_on":true,
         "window_open":false
      },
      {
         "room_type":"bedroom",
         "temperature":22,
         "humidity":39,
         "lights_on":true,
         "window_open":false,
         "bed_occupancy":false
      },
      {
         "room_type":"kitchen",
         "temperature":19,
         "humidity":34,
         "lights_on":true,
         "window_open":true,
         "fridge_temperature":7,
         "oven_on":true
      },
      {
         "room_type":"bathroom",
         "temperature":24,
         "humidity":50,
         "lights_on":true,
         "window_open":true,
         "water_tap_running":true,
         "bath_water_level":91
      }
   ],
   "timestamp":1714807284732
}
```

> **What you should see:** one message per home containing a `rooms` array with per-room sensor readings.

> **What just happened?** the `smart_home` simulator generates one JSON message per home per interval, with nested room-level sensor data for temperature, humidity, and appliance states

We can see that one message of the `smart_home` simulator contains messages for one home with multiple rooms, each providing `temperature`, `humidity` and some other metrics. 

Let's use Telegraf to retrieve them from MQTT and store it in InfluxDB. 

## Configure Telegraf with an access token

Telegraf, a server-based agent, collects and sends metrics and events from databases, systems, and IoT sensors. Written in Go, Telegraf compiles into a single binary with no external dependencies–requiring very minimal memory.

Telegraf is running as part of the dataplatform, but if you check the logs using `docker logs -f telegraf` you will see that it does not work because it is not authenticated:

```bash
2026-04-12T11:54:06Z D! [outputs.influxdb_v3] Buffer fullness: 12 / 10000 metrics
2026-04-12T11:54:06Z E! [agent] Error writing to outputs.influxdb_v3: failed to write metric to demo-db (401 Unauthorized): the request was not authenticated
2026-04-12T11:54:11Z E! [outputs.influxdb_v3] Writing to "http://influxdb3:8181" failed: failed to write metric to demo-db (401 Unauthorized): the request was not authenticated
2026-04-12T11:54:11Z D! [outputs.influxdb_v3] Buffer fullness: 13 / 10000 metrics
2026-04-12T11:54:11Z E! [agent] Error writing to outputs.influxdb_v3: failed to write metric to demo-db (401 Unauthorized): the request was not authenticated
```

> **What you should see:** repeated `401 Unauthorized` errors confirming Telegraf cannot write without a valid token

Let's use the influxdb3 CLI with the `--admin` option to create an operator token:

```bash
docker exec -ti influxdb3 influxdb3 create token --admin
```

The command returns a token string for authenticating CLI commands and API requests.

```bash
New token created successfully!

Token: apiv3_FBiA8QmpreTRyfkSwjfnI07NfmbNyEXvbc7tlsTtW2NQMQFm1Fi9MC-Clp7VlYapEeNF030nH8PIlzwyz0O60Q
HTTP Requests Header: Authorization: Bearer apiv3_FBiA8QmpreTRyfkSwjfnI07NfmbNyEXvbc7tlsTtW2NQMQFm1Fi9MC-Clp7VlYapEeNF030nH8PIlzwyz0O60Q

IMPORTANT: Store this token securely, as it will not be shown again.
```

> **What you should see:** a newly generated token string starting with `apiv3_`.

> **What just happened?** an operator-level token was created in InfluxDB 3.x — this token has full admin access and must be stored securely as it is shown only once

Navigate to the docker folder

```bash
cd $DATAPLATFORM_HOME
```

and create an `.env` file with the token using the following command (replace the token with the one you got in the previous step)

```bash
echo "PLATYS_INFLUXDB_TOKEN=apiv3_FBiA8QmpreTRyfkSwjfnI07NfmbNyEXvbc7tlsTtW2NQMQFm1Fi9MC-Clp7VlYapEeNF030nH8PIlzwyz0O60Q" >> .env
```

Stop and remove the existing `telegraf` container and recreate it

```bash
docker stop telegraf && docker rm telegraf && docker compose up -d telegraf
```

> **What you should see:** Docker pulls nothing (image is cached), recreates the container, and the new telegraf starts

A `docker logs -f telegraf` should show no more errors:

```bash
2026-04-12T12:13:39Z W! Strict environment variable handling is the new default starting with v1.38.0! If your configuration does not work with strict handling please explicitly add the --non-strict-env-handling flag to switch to the previous behavior!
2026-04-12T12:13:39Z I! Loading config: /etc/telegraf/telegraf.conf
2026-04-12T12:13:39Z I! Starting Telegraf 1.38.2 brought to you by InfluxData the makers of InfluxDB
2026-04-12T12:13:39Z I! Available plugins: 244 inputs, 9 aggregators, 35 processors, 26 parsers, 68 outputs, 8 secret-stores
2026-04-12T12:13:39Z I! Loaded inputs: mock
2026-04-12T12:13:39Z I! Loaded aggregators:
2026-04-12T12:13:39Z I! Loaded processors:
2026-04-12T12:13:39Z I! Loaded secretstores:
2026-04-12T12:13:39Z I! Loaded outputs: influxdb_v3
2026-04-12T12:13:39Z I! Tags enabled: host=telegraf
2026-04-12T12:13:39Z I! [agent] Config: Interval:5s, Quiet:false, Hostname:"telegraf", Flush Interval:5s
2026-04-12T12:13:39Z W! [agent] The default value of 'skip_processors_after_aggregators' will change to 'true' with Telegraf v1.40.0! If you need the current default behavior, please explicitly set the option to 'false'!
2026-04-12T12:13:39Z D! [agent] Initializing plugins
2026-04-12T12:13:39Z D! [agent] Connecting outputs
2026-04-12T12:13:39Z D! [agent] Attempting connection to [outputs.influxdb_v3]
2026-04-12T12:13:39Z D! [agent] Successfully connected to outputs.influxdb_v3
2026-04-12T12:13:39Z D! [agent] Starting service inputs
2026-04-12T12:13:46Z D! [outputs.influxdb_v3] Wrote batch of 1 metrics in 1.170064513s
2026-04-12T12:13:46Z D! [outputs.influxdb_v3] Buffer fullness: 1 / 10000 metrics
2026-04-12T12:13:50Z D! [outputs.influxdb_v3] Wrote batch of 1 metrics in 91.645953ms
2026-04-12T12:13:50Z D! [outputs.influxdb_v3] Buffer fullness: 1 / 10000 metrics
```

> **What you should see:** log lines showing `Successfully connected to outputs.influxdb_v3` and periodic `Wrote batch of 1 metrics` lines with no errors.

> **What just happened?** Telegraf re-read the `.env` file containing the token and can now authenticate with InfluxDB 3.x

## Using Telegraf to retrieve values from MQTT and store in InfluxDB

For that we define which plugins Telegraf will use in the `telegraf.conf` configuration file. Each configuration file needs at least one enabled input plugin (where the metrics come from) and at least one enabled output plugin (where the metrics go).

We will use the [MQTT consumer](https://docs.influxdata.com/telegraf/v1/plugins/#input-mqtt_consumer) input plugin and the [InfluxDB_v3](https://docs.influxdata.com/telegraf/v1/plugins/#output-influxdb_v3) output plugin. 

Navigate to the `data-transfer` folder and create a file named `telegraf.conf`. 

```bash
cd $DATAPLATFORM_HOME/data-transfer
nano telegraf.conf
```

Add the following configuration to the empty file

```conf
# Configuration for telegraf agent
[agent]
  ## Default data collection interval for all inputs
  interval = "10s"
  ## Rounds collection interval to 'interval'
  ## ie, if interval="10s" then always collect on :00, :10, :20, etc.
  round_interval = true

  ## Telegraf will send metrics to outputs in batches of at most
  ## metric_batch_size metrics.
  ## This controls the size of writes that Telegraf sends to output plugins.
  metric_batch_size = 1000

  ## Maximum number of unwritten metrics per output.  Increasing this value
  ## allows for longer periods of output downtime without dropping metrics at the
  ## cost of higher maximum memory usage.
  metric_buffer_limit = 10000

  ## Collection jitter is used to jitter the collection by a random amount.
  ## Each plugin will sleep for a random time within jitter before collecting.
  ## This can be used to avoid many plugins querying things like sysfs at the
  ## same time, which can have a measurable effect on the system.
  collection_jitter = "0s"

  ## Default flushing interval for all outputs. Maximum flush_interval will be
  ## flush_interval + flush_jitter
  flush_interval = "10s"
  ## Jitter the flush interval by a random amount. This is primarily to avoid
  ## large write spikes for users running a large number of telegraf instances.
  ## ie, a jitter of 5s and interval 10s means flushes will happen every 10-15s
  flush_jitter = "0s"

  ## By default or when set to "0s", precision will be set to the same
  ## timestamp order as the collection interval, with the maximum being 1s.
  ##   ie, when interval = "10s", precision will be "1s"
  ##       when interval = "250ms", precision will be "1ms"
  ## Precision will NOT be used for service inputs. It is up to each individual
  ## service input to set the timestamp at the appropriate precision.
  ## Valid time units are "ns", "us" (or "µs"), "ms", "s".
  precision = ""

  ## Log at debug level.
  debug = true
  ## Log only error level messages.
  # quiet = false

  ## Log target controls the destination for logs and can be one of "file",
  ## "stderr" or, on Windows, "eventlog".  When set to "file", the output file
  ## is determined by the "logfile" setting.
  # logtarget = "file"

  ## Name of the file to be logged to when using the "file" logtarget.  If set to
  ## the empty string then logs are written to stderr.
  # logfile = ""

  ## The logfile will be rotated after the time interval specified.  When set
  ## to 0 no time based rotation is performed.  Logs are rotated only when
  ## written to, if there is no log activity rotation may be delayed.
  # logfile_rotation_interval = "0d"

  ## The logfile will be rotated when it becomes larger than the specified
  ## size.  When set to 0 no size based rotation is performed.
  # logfile_rotation_max_size = "0MB"

  ## Maximum number of rotated archives to keep, any older logs are deleted.
  ## If set to -1, no archives are removed.
  # logfile_rotation_max_archives = 5

  ## Pick a timezone to use when logging or type 'local' for local time.
  ## Example: America/Chicago
  # log_with_timezone = ""

  ## Override default hostname, if empty use os.Hostname()
  hostname = ""
  ## If set to true, do no set the "host" tag in the telegraf agent.
  omit_hostname = false

[[outputs.influxdb_v3]]
  ## The URLs of the InfluxDB cluster nodes.
  ##
  ## Multiple URLs can be specified for a single cluster, only ONE of the
  ## urls will be written to each interval.
  ##   ex: urls = ["https://us-west-2-1.aws.cloud2.influxdata.com"]
  urls = ["http://influxdb3:8181"]

  ## Token for authentication.
  token = "$INFLUXDB_TOKEN"

  ## Organization is the name of the organization you wish to write to; must exist.
  database = "$INFLUXDB_DATABASE"

  ## The value of this tag will be used to determine the database. If this
  ## tag is not set the 'database' option is used as the default.
  # database_tag = ""

  ## If true, the database tag will not be added to the metric
  # exclude_database_tag = false

  ## Wait for WAL persistence to complete synchronization
  ## Setting this to false reduces latency but increases the risk of data loss.
  ## See https://docs.influxdata.com/influxdb3/enterprise/write-data/http-api/v3-write-lp/#use-no_sync-for-immediate-write-responses
  # sync = true

  ## Enable or disable conversion of unsigned integer fields to signed integers
  ## This is useful if existing data exist as signed integers e.g. from previous
  ## versions of InfluxDB.
  # convert_uint_to_int = false

  ## Omit the timestamp of the metrics when sending to allow InfluxDB to set the
  ## timestamp of the data during ingestion. You likely want this to be false
  ## to submit the metric timestamp
  # omit_timestamp = false

  ## HTTP User-Agent
  # user_agent = "telegraf"
  
  ## Timeout for HTTP messages.
  # timeout = "5s"

# Read metrics from MQTT topic(s)
[[inputs.mqtt_consumer]]
  ## Broker URLs for the MQTT server or cluster.  To connect to multiple
  ## clusters or standalone servers, use a separate plugin instance.
  ##   example: servers = ["tcp://localhost:1883"]
  ##            servers = ["ssl://localhost:1883"]
  ##            servers = ["ws://localhost:1883"]
  servers = ["tcp://mosquitto-1:1883"]

  ## Topics that will be subscribed to.
  topics = [
    "mqttx/simulate/#",
  ]

  ## The message topic will be stored in a tag specified by this value.  If set
  ## to the empty string no topic tag will be created.
  topic_tag = ""

  ## Username and password to connect MQTT server.
  # username = "telegraf"
  # password = "metricsmetricsmetricsmetrics"

  ## Data format to consume.
  ## Each data format has its own unique set of configuration options, read
  ## more about them here:
  ## https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_INPUT.md
  data_format = "json_v2"

  [[inputs.mqtt_consumer.json_v2]]
      measurement_name = "smart_home" # A string that will become the new measurement name
      timestamp_path = "timestamp" # A string with valid GJSON path syntax to a valid timestamp (single value)
      timestamp_format = "unix_ms" # A string with a valid timestamp format (see below for possible values)

    [[inputs.mqtt_consumer.json_v2.tag]]
      path = "home_id" # A string with valid GJSON path syntax
      rename = "id" # A string with a new name for the tag key
      type = "string" # A string specifying the type (int,uint,float,string,bool)
      optional = false # true: suppress errors if configured path does not exist

    [[inputs.mqtt_consumer.json_v2.tag]]
      path = "owner_name" # A string with valid GJSON path syntax
      rename = "owner" # A string with a new name for the tag key
      type = "string" # A string specifying the type (int,uint,float,string,bool)
      optional = false # true: suppress errors if configured path does not exist

    [[inputs.mqtt_consumer.json_v2.object]]
      path = "rooms"
```

With that in place, let's start the telegraf service. We can use the `telegraf` docker container running as part of the platform. It is currently running with a default configuration file, which retrieves metrics from docker and sends them to InfluxDB. We could use Platys (the docker compose generator tool) to change the `docker-compose.yml` to use our new `telegraf.conf` configuration file. But instead we just use the container to start a 2nd telegraf job. In a terminal window, execute the following `docker exec ...` command:

```bash
docker exec -ti telegraf telegraf --config ./data-transfer/telegraf.conf
```

You should see an output similar to the one below

```
~/w/platys-datahub>docker exec -ti telegraf telegraf --config ./data-transfer/telegraf.conf                                      9.22s 20:04
2026-04-12T12:17:57Z I! Loading config: ./data-transfer/telegraf.conf
2026-04-12T12:17:57Z I! Starting Telegraf 1.38.2 brought to you by InfluxData the makers of InfluxDB
2026-04-12T12:17:57Z I! Available plugins: 244 inputs, 9 aggregators, 35 processors, 26 parsers, 68 outputs, 8 secret-stores
2026-04-12T12:17:57Z I! Loaded inputs: mqtt_consumer
2026-04-12T12:17:57Z I! Loaded aggregators:
2026-04-12T12:17:57Z I! Loaded processors:
2026-04-12T12:17:57Z I! Loaded secretstores:
2026-04-12T12:17:57Z I! Loaded outputs: influxdb_v3
2026-04-12T12:17:57Z I! Tags enabled: host=telegraf
2026-04-12T12:17:57Z I! [agent] Config: Interval:10s, Quiet:false, Hostname:"telegraf", Flush Interval:10s
2026-04-12T12:17:57Z W! [agent] The default value of 'skip_processors_after_aggregators' will change to 'true' with Telegraf v1.40.0! If you need the current default behavior, please explicitly set the option to 'false'!
2026-04-12T12:17:57Z D! [agent] Initializing plugins
2026-04-12T12:17:57Z D! [agent] Connecting outputs
2026-04-12T12:17:57Z D! [agent] Attempting connection to [outputs.influxdb_v3]
2026-04-12T12:17:57Z D! [agent] Successfully connected to outputs.influxdb_v3
2026-04-12T12:17:57Z D! [agent] Starting service inputs
2026-04-12T12:17:57Z I! [inputs.mqtt_consumer] Connected [tcp://mosquitto-1:1883]
2026-04-12T12:18:01Z D! [outputs.influxdb_v3] Wrote batch of 1000 metrics in 449.718227ms
2026-04-12T12:18:01Z D! [outputs.influxdb_v3] Buffer fullness: 200 / 10000 metrics
2026-04-12T12:18:03Z D! [outputs.influxdb_v3] Wrote batch of 1000 metrics in 442.349105ms
2026-04-12T12:18:03Z D! [outputs.influxdb_v3] Buffer fullness: 0 / 10000 metrics
2026-04-12T12:18:06Z D! [outputs.influxdb_v3] Wrote batch of 1000 metrics in 446.073468ms
```

> **What you should see:** Telegraf loads the new config, connects to `mosquitto-1:1883`, and immediately starts writing batches of 1000 metrics.

> **What just happened?** Telegraf's MQTT consumer subscribed to `mqttx/simulate/#`, parsed the JSON payload using the `json_v2` format, extracted the room objects as individual metrics named `smart_home`, and forwarded them to InfluxDB 3.x using the `influxdb_v3` output plugin

Because debug is enabled in the `telegraf.conf` we see additional output whenever data is flushed to InfluxDB.

## Querying Data with the InfluxDB 3.x CLI

With data flowing from MQTT through Telegraf into InfluxDB, we can now query it using the `influxdb3` CLI. The CLI is available inside the `influxdb3` container and uses standard SQL.

For convenience, export your token into the current terminal session (replace the token value with the one you created earlier):

```bash
export INFLUXDB_TOKEN=apiv3_FBiA8QmpreTRyfkSwjfnI07NfmbNyEXvbc7tlsTtW2NQMQFm1Fi9MC-Clp7VlYapEeNF030nH8PIlzwyz0O60Q
```

### List available databases

To see which databases exist on the InfluxDB 3.x instance:

```bash
docker exec -ti influxdb3 influxdb3 show databases \
  --token $INFLUXDB_TOKEN
```

You should see `demo-db` listed, which is the database Telegraf has been writing into.

```bash
+---------------+
| iox::database |
+---------------+
| _internal     |
| demo-db       |
+---------------+
```

> **What you should see:** a table listing `demo-db` and `_internal`

### List tables in the database

Each measurement written by Telegraf becomes a table. To list all tables in `demo-db`:

```bash
docker exec -ti influxdb3 influxdb3 query \
  --token $INFLUXDB_TOKEN \
  --database demo-db "SHOW TABLES"
```

You should see `smart_home` listed — the measurement name we configured in `telegraf.conf`.

```
+---------------+--------------------+-------------------------------------+------------+
| table_catalog | table_schema       | table_name                          | table_type |
+---------------+--------------------+-------------------------------------+------------+
| public        | iox                | mock                                | BASE TABLE |
| public        | iox                | smart_home                          | BASE TABLE |
| public        | system             | distinct_caches                     | BASE TABLE |
| public        | system             | influxdb_schema                     | BASE TABLE |
| public        | system             | last_caches                         | BASE TABLE |
| public        | system             | parquet_files                       | BASE TABLE |
| public        | system             | processing_engine_logs              | BASE TABLE |
| public        | system             | processing_engine_trigger_arguments | BASE TABLE |
| public        | system             | processing_engine_triggers          | BASE TABLE |
| public        | system             | queries                             | BASE TABLE |
| public        | information_schema | tables                              | VIEW       |
| public        | information_schema | views                               | VIEW       |
| public        | information_schema | columns                             | VIEW       |
| public        | information_schema | df_settings                         | VIEW       |
| public        | information_schema | schemata                            | VIEW       |
| public        | information_schema | routines                            | VIEW       |
| public        | information_schema | parameters                          | VIEW       |
+---------------+--------------------+-------------------------------------+------------+
````

> **What you should see:** a table listing `mock` and `smart_home` under `iox` schema, plus system tables

### Query the raw data

Retrieve the 10 most recent rows from the `smart_home` measurement:

```bash
docker exec -ti influxdb3 influxdb3 query \
  --token $INFLUXDB_TOKEN \
  --database demo-db \
  "SELECT time, id, owner, room_type, temperature, humidity FROM smart_home ORDER BY time DESC LIMIT 10"
```

The output will look similar to this:

```
+-------------------------+--------------------------------------+----------------------+-------------+-------------+----------+
| time                    | id                                   | owner                | room_type   | temperature | humidity |
+-------------------------+--------------------------------------+----------------------+-------------+-------------+----------+
| 2026-04-12T13:03:12.903 | 94169b47-7c9b-4c4f-832a-d257082fc928 | Tina Collins IV      | kitchen     | 22.0        | 33.0     |
| 2026-04-12T13:03:12.903 | c0f5a3fe-ba16-4fc3-ae63-8cc470648b35 | Shaun Stiedemann V   | living room | 26.0        | 40.0     |
| 2026-04-12T13:03:12.903 | bfa22b56-8dc0-4739-b113-a0f922267094 | Margie Barrows       | kitchen     | 25.0        | 37.0     |
| 2026-04-12T13:03:12.902 | 94944779-1801-4a71-863e-a185c647a44a | Kerry Schaefer       | bedroom     | 21.0        | 32.0     |
| 2026-04-12T13:03:12.902 | db27e579-9f18-49cc-80c3-c063ff954b99 | Lola Thompson        | living room | 24.0        | 44.0     |
| 2026-04-12T13:03:12.902 | 7b8f1772-ec8a-44dd-bb06-67c528170f04 | Lee Maggio           | bathroom    | 18.0        | 37.0     |
| 2026-04-12T13:03:12.902 | 3d99d6cd-9fd4-4d4b-abf5-8023d24b77e3 | Mr. Gustavo Strosin  | bedroom     | 26.0        | 47.0     |
| 2026-04-12T13:03:12.902 | b598928d-9bcd-4a44-9aaa-c5fd89d56ebc | Shannon Heller       | bedroom     | 18.0        | 42.0     |
| 2026-04-12T13:03:12.902 | dceccdf0-4efa-4cde-9687-96a5af5741cf | Gilberto Bednar      | bathroom    | 21.0        | 38.0     |
| 2026-04-12T13:03:12.902 | 946b43e5-f70a-4e2a-8a4a-09d097bb262e | Mr. Alexander Heaney | living room | 25.0        | 30.0     |
+-------------------------+--------------------------------------+----------------------+-------------+-------------+----------+
```

> **What you should see:** rows with columns `time`, `id`, `owner`, `room_type`, `temperature`, `humidity` — one row per room per home per interval, most recent first.

> **What just happened?** InfluxDB 3.x stored each Telegraf metric as a row in the `smart_home` table, with tags (`id`, `owner`, `room_type`) as indexed columns and fields (`temperature`, `humidity`, etc.) as regular columns

### Filter by room type

Use a `WHERE` clause to restrict results to a specific room type:

```bash
docker exec -ti influxdb3 influxdb3 query \
  --token $INFLUXDB_TOKEN \
  --database demo-db \
  "SELECT time, id, owner, temperature, humidity FROM smart_home WHERE room_type = 'living room' ORDER BY time DESC LIMIT 10"
```

```
+-------------------------+--------------------------------------+----------------------+-------------+----------+
| time                    | id                                   | owner                | temperature | humidity |
+-------------------------+--------------------------------------+----------------------+-------------+----------+
| 2026-04-12T13:03:35.036 | 35ecbc8b-648b-47b9-88e0-0595eb72e0d0 | Ms. Sue Purdy        | 19.0        | 47.0     |
| 2026-04-12T13:03:35.036 | 00f1015b-b548-46e4-b5cd-a002d3496bfc | Eduardo Kiehn        | 22.0        | 45.0     |
| 2026-04-12T13:03:35.036 | 44330b51-a6f0-4eec-a054-b34cb93d78e3 | Janet Altenwerth Jr. | 23.0        | 38.0     |
| 2026-04-12T13:03:35.036 | 24c65087-69ba-458f-8a47-21806554ff73 | Gertrude Crona III   | 18.0        | 33.0     |
| 2026-04-12T13:03:35.004 | 962faa7c-273f-44b3-910a-016ce12c78e8 | Sherman Bednar       | 24.0        | 37.0     |
| 2026-04-12T13:03:35.003 | 91c484ac-508a-4bcb-9ee9-7fba69eecac3 | Brandy Reichel       | 22.0        | 30.0     |
| 2026-04-12T13:03:34.928 | 7b8f1772-ec8a-44dd-bb06-67c528170f04 | Lee Maggio           | 18.0        | 35.0     |
| 2026-04-12T13:03:34.928 | dceccdf0-4efa-4cde-9687-96a5af5741cf | Gilberto Bednar      | 21.0        | 40.0     |
| 2026-04-12T13:03:34.928 | c0f5a3fe-ba16-4fc3-ae63-8cc470648b35 | Shaun Stiedemann V   | 19.0        | 50.0     |
| 2026-04-12T13:03:34.928 | 946b43e5-f70a-4e2a-8a4a-09d097bb262e | Mr. Alexander Heaney | 21.0        | 41.0     |
+-------------------------+--------------------------------------+----------------------+-------------+----------+
```

> **What you should see:** only rows where `room_type = 'living room'`

### Aggregate by room type

Calculate the average temperature and humidity across all homes, grouped by room type:

```bash
docker exec -ti influxdb3 influxdb3 query \
  --token $INFLUXDB_TOKEN \
  --database demo-db \
  "SELECT room_type, AVG(temperature) AS avg_temp, AVG(humidity) AS avg_humidity FROM smart_home GROUP BY room_type ORDER BY room_type"
```

You should see output similar to:

```
+-------------+--------------------+--------------------+
| room_type   | avg_temp           | avg_humidity       |
+-------------+--------------------+--------------------+
| bathroom    | 21.993545028630923 | 40.041841633408524 |
| bedroom     | 22.00419140761439  | 39.98399457582544  |
| kitchen     | 22.00609735269001  | 39.98896669513237  |
| living room | 21.989200549336218 | 39.99074035540389  |
+-------------+--------------------+--------------------+
```

> **What you should see:** one row per room type with averaged temperature and humidity.

> **What just happened?** InfluxDB 3.x executed the SQL aggregation natively — it supports standard SQL GROUP BY with aggregate functions like AVG, COUNT, MIN, MAX directly on the time-series data

### Filter by time range

InfluxDB 3.x supports standard SQL interval expressions for time-based filtering. To retrieve data from the last 5 minutes:

```bash
docker exec -ti influxdb3 influxdb3 query \
  --token $INFLUXDB_TOKEN \
  --database demo-db \
  "SELECT time, id, room_type, temperature, humidity FROM smart_home WHERE time >= now() - interval '5 minutes' ORDER BY time DESC"
```

> **What you should see:** only rows from the last 5 minutes, sorted most-recent first

### Count records per home

To see how many room readings have been collected per home:

```bash
docker exec -ti influxdb3 influxdb3 query \
  --token $INFLUXDB_TOKEN \
  --database demo-db \
  "SELECT id, owner, COUNT(*) AS readings FROM smart_home GROUP BY id, owner ORDER BY readings DESC LIMIT 10"
```

## Querying Data with the InfluxDB Explorer UI

InfluxDB 3 Explorer is the standalone web application designed for visualizing, querying, and managing your data stored in InfluxDB 3 Core and Enterprise. Explorer provides an intuitive interface for interacting with your time series data, streamlining database operations and enhancing data insights.

### Open the Explorer UI

In a browser window, navigate to <http://dataplatform:28264>.

![](./images/influxdb-explorer-1.png)

> **What you should see:** the InfluxDB Explorer home screen with a "Connect Your First Server" button

### Create a server connection

Click on the **Connect Your First Server** button and enter the following values (make sure to replace the token with the one you have generated before):

 * **Server Name**: `demo`
 * **Server URL**: `influxdb3:8181`
 * **Token**: `apiv3_FBiA8QmpreTRyfkSwjfnI07NfmbNyEXvbc7tlsTtW2NQMQFm1Fi9MC-Clp7VlYapEeNF030nH8PIlzwyz0O60Q`

![](./images/influxdb-explorer-2.png)

> **What you should see:** the server connection form with Name, URL, and Token fields

and then click on **Add Server**. The new server will show up on the main screen. 

![](./images/influxdb-explorer-3.png)

> **What you should see:** the new `demo` server listed on the home screen.

> **What just happened?** Explorer saved the server connection including the token; it will now route all queries to `http://influxdb3:8181` authenticated with that token

### Select the database

In the menu on the left navigate to **Query Data** | **Data Explorer** and select `demo-db` in the **Schema** drop-down. 

![](./images/influxdb-explorer-4.png)

> **What you should see:** the Data Explorer view with `demo-db` selected in the schema dropdown

This is the database that Telegraf has been writing the `smart_home` data into.

### Browse tables

On the left-hand panel you will see all tables available in the selected database. Click on **smart_home** to expand it and inspect its columns — you should see the tags (`id`, `owner`, `room_type`) and fields (`temperature`, `humidity`, `lights_on`, etc.) that were mapped from the MQTT JSON payload by Telegraf.

![](./images/influxdb-explorer-5.png)

> **What you should see:** the `smart_home` table expanded showing its columns with their data types — tags and fields distinguished

### Run a query

Click in the SQL editor area and enter a query. For example, to retrieve the 10 most recent readings:

```sql
SELECT time, id, owner, room_type, temperature, humidity
FROM smart_home
ORDER BY time DESC
LIMIT 10
```

Click **Run** (or press **Ctrl+Enter**) to execute. The results appear in a table below the editor.

![](./images/influxdb-explorer-6.png)

> **What you should see:** a results table with time-ordered rows of sensor readings

### Aggregate by room type

Try an aggregation query to see average temperature and humidity per room type:

```sql
SELECT room_type,
       AVG(temperature) AS avg_temp,
       AVG(humidity)    AS avg_humidity
FROM smart_home
GROUP BY room_type
ORDER BY room_type
```

### Filter by time range

Use a time filter to scope the query to recent data:

```sql
SELECT time, id, room_type, temperature, humidity
FROM smart_home
WHERE time >= now() - interval '5 minutes'
ORDER BY time DESC
```

### Visualise results as a chart

After running a query that returns numeric columns over time, switch from the **Table** view to the **Line** view using the toggle above the results. This renders the data as a time-series line chart, which is especially useful for spotting trends in temperature or humidity over time.

![](./images/influxdb-explorer-7.png)

> **What you should see:** a line chart with one line per `room_type`, showing temperature or the selected metric over time

---

## Working with InfluxDB from Python

The `influxdb3-python` library is the official Python client for InfluxDB 3.x. It uses the Apache Arrow Flight SQL protocol under the hood for fast columnar query results and returns data as a **pandas DataFrame**, making it easy to combine time-series queries with data analysis in a notebook.

Open a browser and navigate to <http://dataplatform:28888> and log in with token `abc123!`.

Create a new Python 3 notebook by clicking on the **Python 3 (ipykernel)** widget, then work through the cells below in order.

### Cell 1 — Install the library

```python
import sys
!{sys.executable} -m pip install influxdb3-python pandas
```

> **What you should see:** pip output ending with `Successfully installed influxdb3-python-...`.

### Cell 2 — Connect to InfluxDB

Replace the token value with the operator token you created in the [Configure Telegraf with a access token](#configure-telegraf-with-a-access-token) section.

```python
from influxdb_client_3 import InfluxDBClient3, Point

TOKEN    = "apiv3_FBiA8QmpreTRyfkSwjfnI07NfmbNyEXvbc7tlsTtW2NQMQFm1Fi9MC-Clp7VlYapEeNF030nH8PIlzwyz0O60Q"
HOST     = "http://influxdb3:8181"
DATABASE = "demo-db"

client = InfluxDBClient3(host=HOST, token=TOKEN, database=DATABASE)
print("Connected to InfluxDB 3.x")
```

> **What you should see:** `Connected to InfluxDB 3.x` — the client is initialised. The actual connection to the server is lazy; it is established on the first write or query.

### Cell 3 — Write data points

The `Point` builder constructs a single time-series data point in Line Protocol format. Tags are indexed string columns (good for filtering); fields are the measured values.

```python
from datetime import datetime, timezone

# Write a few simulated living-room readings
points = [
    Point("smart_home")
        .tag("id",        "python-home-001")
        .tag("owner",     "Python User")
        .tag("room_type", "living room")
        .field("temperature", 21.5)
        .field("humidity",    44.0)
        .time(datetime.now(timezone.utc)),
    Point("smart_home")
        .tag("id",        "python-home-001")
        .tag("owner",     "Python User")
        .tag("room_type", "kitchen")
        .field("temperature", 19.0)
        .field("humidity",    38.5)
        .time(datetime.now(timezone.utc)),
]

client.write(record=points)
print(f"Wrote {len(points)} points")
```

```
Wrote 2 points
```

> **What you should see:** `Wrote 2 points` with no errors.

> **What just happened?** Each `Point` maps to one row in the `smart_home` table. Tags become indexed columns (used in `WHERE` clauses), fields become regular columns (used in aggregations), and `.time()` sets the `time` column. Without `.time()`, InfluxDB uses the server's current time.

### Cell 4 — Query with SQL

`query` executes standard SQL against the database and returns an Apache Arrow `Table`. Call `.to_pandas()` to get a familiar DataFrame:

```python
df = client.query("""
    SELECT time, id, owner, room_type, temperature, humidity
    FROM smart_home
    WHERE id = 'python-home-001'
    ORDER BY time DESC
    LIMIT 10
""").to_pandas()

print(df.to_string(index=False))
```

```
                      time              id        owner     room_type  temperature  humidity
 2026-04-30 09:14:22+00:00  python-home-001  Python User       kitchen         19.0      38.5
 2026-04-30 09:14:22+00:00  python-home-001  Python User  living room         21.5      44.0
```

> **What you should see:** the two rows you just wrote, most-recent first.

### Cell 5 — Aggregate query

Use SQL aggregation to compute average temperature and humidity per room type across the entire dataset:

```python
df = client.query("""
    SELECT   room_type,
             ROUND(AVG(temperature), 2) AS avg_temp,
             ROUND(AVG(humidity), 2)    AS avg_humidity,
             COUNT(*)                   AS readings
    FROM     smart_home
    GROUP BY room_type
    ORDER BY room_type
""").to_pandas()

print(df.to_string(index=False))
```

```
   room_type  avg_temp  avg_humidity  readings
    bathroom     22.01         40.04     18432
     bedroom     22.00         39.98     18450
     kitchen     22.01         39.99     18448
 living room     21.99         39.99     18441
```

> **What you should see:** one row per room type with average temperature, average humidity, and the total number of readings collected since Telegraf started writing.

### Cell 6 — Time-range query

Filter to the last 5 minutes using an `interval` expression — the same syntax supported by the CLI:

```python
df = client.query("""
    SELECT   time, room_type, temperature, humidity
    FROM     smart_home
    WHERE    time >= now() - interval '5 minutes'
    ORDER BY time DESC
    LIMIT    20
""").to_pandas()

print(f"{len(df)} rows in the last 5 minutes")
print(df[["time", "room_type", "temperature", "humidity"]].head(5).to_string(index=False))
```

> **What you should see:** rows timestamped within the last 5 minutes. The count reflects how many MQTT messages arrived in that window.

### Cell 7 — Close the client

```python
client.close()
print("Client closed")
```

