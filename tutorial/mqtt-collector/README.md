# MQTT Collector Tutorial - Collecting Real-World Data

This tutorial will guide you through configuring the Timebase MQTT Collector to connect to a public MQTT broker, subscribe to a topic, and store the data in your Historian.

## What You'll Learn

- How to configure the Timebase MQTT Collector plugin
- Connecting to public MQTT brokers for testing
- Finding and subscribing to MQTT topics
- Viewing collected data in Timebase Explorer
- Understanding different payload formats

**Time Required:** 15-20 minutes

**Prerequisites:**
- Timebase stack running (see [Getting Started](../getting-started.md))
- Basic understanding of MQTT concepts (optional but helpful)

---

## Table of Contents

1. [Understanding MQTT and Timebase](#understanding-mqtt-and-timebase)
2. [Finding a Public MQTT Broker](#finding-a-public-mqtt-broker)
3. [Discovering Available Topics](#discovering-available-topics)
4. [Configuring the Collector](#configuring-the-collector)
5. [Verifying Data Collection](#verifying-data-collection)
6. [Viewing Data in Explorer](#viewing-data-in-explorer)
7. [Troubleshooting](#troubleshooting)
8. [Next Steps](#next-steps)

---

## Understanding MQTT and Timebase

### What is MQTT?

MQTT (Message Queuing Telemetry Transport) is a lightweight messaging protocol commonly used in IoT applications. It works on a publish-subscribe model:

- **Broker**: The central server that routes messages
- **Publisher**: Devices/services that send data to topics
- **Subscriber**: Applications that receive data from topics
- **Topic**: A hierarchical path like `home/sensors/temperature`

### How Timebase Uses MQTT

The Timebase Collector acts as an **MQTT subscriber**:
1. Connects to an MQTT broker
2. Subscribes to one or more topics
3. Receives messages published to those topics
4. Parses the message payload
5. Stores the data in the Historian with timestamps

---

## Finding a Public MQTT Broker

For this tutorial, we'll use a public MQTT broker for testing. Here are some popular options:

### Option 1: Eclipse Mosquitto Test Server

```
Host: test.mosquitto.org
Port: 1883 (no authentication)
Topics: Various test topics available
```

### Option 2: HiveMQ Public Broker

```
Host: broker.hivemq.com
Port: 1883 (no authentication)
Topics: Many active topics from IoT devices
```

### Option 3: EMQX Public Broker

```
Host: broker.emqx.io
Port: 1883 (no authentication)
Topics: Public test environment
```

**For this tutorial**, we'll use **test.mosquitto.org** as it's reliable and well-documented.

⚠️ **Security Note:** Public brokers should only be used for testing. Never send sensitive data to public brokers!

---

## Discovering Available Topics

Before configuring the collector, let's find an active topic with data.

### Method 1: Using MQTT Explorer (Recommended)

If you have MQTT Explorer installed:

1. Download from: http://mqtt-explorer.com/
2. Connect to `test.mosquitto.org:1883`
3. Browse the topic tree to find active topics
4. Common topics to try:
   - `test/#` - Test messages
   - `$SYS/#` - Broker statistics
   - `sensors/#` - Sensor data (if available)

### Method 2: Using Mosquitto CLI

If you have `mosquitto_sub` installed:

```bash
# Subscribe to all topics (use with caution!)
mosquitto_sub -h test.mosquitto.org -t '#' -v

# Subscribe to specific topics
mosquitto_sub -h test.mosquitto.org -t 'test/#' -v
mosquitto_sub -h test.mosquitto.org -t '$SYS/broker/clients/connected' -v
```

### Method 3: Using Docker

```bash
# Run mosquitto_sub in a container
docker run -it --rm eclipse-mosquitto mosquitto_sub -h test.mosquitto.org -t '$SYS/broker/load/messages/received/1min' -v
```

**For this tutorial**, we'll use the broker statistics topic:
```
$SYS/broker/load/messages/received/1min
```

This topic publishes the number of messages the broker receives per minute - perfect for testing!

---

## Configuring the Collector

Now that we have a broker and topic, let's configure the Timebase Collector.

### Step 1: Access the Collector Configuration

1. Open your web browser
2. Navigate to the Collector interface:
   ```
   http://localhost:4521
   ```
   (Or `http://your-server-ip:4521` if running remotely)

### Step 2: Enable the Collector

Before adding configurations, make sure the collector is active:

1. Look for the **"Active"** setting or toggle
2. Set it to **`true`** or enable it
3. The collector needs to be active to collect data

**Alternatively**, update your `.env` file:
```bash
COLLECTOR_01_ACTIVE=true
```

Then restart the stack:
```bash
docker-compose restart timebase-collector-01
```

### Step 3: Create MQTT Plugin Configuration

Navigate to the **Plugins** or **Configuration** section and create a new MQTT plugin.

**Configuration JSON:**

```json
{
  "Name": "Mosquitto-Test-Broker",
  "Plugin": "MQTT",
  "Enabled": true,
  "Settings": {
    "Host": "test.mosquitto.org",
    "Port": 1883,
    "UseTls": false,
    "Subscriptions": [
      {
        "Type": 1,
        "Topics": [
          "$SYS/broker/load/messages/received/1min"
        ],
        "TagnameFields": {
          "Default": "mosquitto.messages_per_minute"
        },
        "TagnameIncludesTopic": false,
        "TagnameDelimiter": ".",
        "QoS": 2
      }
    ]
  }
}
```

**Configuration Breakdown:**

| Field | Value | Description |
|-------|-------|-------------|
| `Name` | `Mosquitto-Test-Broker` | Friendly name for this configuration |
| `Plugin` | `MQTT` | Plugin type (always "MQTT" for MQTT) |
| `Enabled` | `true` | Enable this configuration |
| `Host` | `test.mosquitto.org` | MQTT broker address |
| `Port` | `1883` | Standard MQTT port (non-TLS) |
| `UseTls` | `false` | No encryption (public broker) |

**Subscription Details:**

| Field | Value | Description |
|-------|-------|-------------|
| `Type` | `1` | Payload type: 1=Simple Primitives, 2=Simple JSON, 3=Structured JSON |
| `Topics` | `["$SYS/broker/load/..."]` | Array of MQTT topics to subscribe to |
| `TagnameFields.Default` | `mosquitto.messages_per_minute` | Tag name in Historian |
| `TagnameIncludesTopic` | `false` | Don't include topic name in tag |
| `TagnameDelimiter` | `.` | Delimiter for hierarchical tag names |
| `QoS` | `2` | MQTT Quality of Service (0, 1, or 2) |

### Step 4: Save and Apply Configuration

1. Save the configuration
2. The collector should automatically connect to the broker
3. Data collection begins immediately

---

## Verifying Data Collection

### Check Collector Logs

1. View the collector logs to verify connection:
   ```bash
   docker logs collector-01 -f
   ```

2. Look for messages like:
   ```
   Connected to MQTT broker: test.mosquitto.org:1883
   Subscribed to topic: $SYS/broker/load/messages/received/1min
   Received message on topic: $SYS/broker/load/messages/received/1min
   ```

3. Check for any errors (connection failures, authentication issues, etc.)

### Verify in Collector UI

1. Go back to the Collector interface: `http://localhost:4521`
2. Look for **Status**, **Statistics**, or **Tags** section
3. You should see:
   - Connection status: **Connected**
   - Messages received count increasing
   - Tag `mosquitto.messages_per_minute` listed

---

## Viewing Data in Explorer

Now let's visualize the collected data!

### Step 1: Open Timebase Explorer

Navigate to:
```
http://localhost:4531
```
(Or `http://your-server-ip:4531` if running remotely)

### Step 2: Connect to Historian

If this is your first time opening Explorer:

1. Click **"Add Connection"** or **"Configure"**
2. Enter Historian connection details:
   - **Host**: `historian` (or `http://historian:4511`)
   - **Port**: `4511`
   - Click **"Connect"** or **"Save"**

### Step 3: Find Your Tag

1. In the tag browser or tree view, look for:
   ```
   mosquitto.messages_per_minute
   ```

2. The tag should appear in the list with:
   - Tag name
   - Last value
   - Last update timestamp

### Step 4: Create a Trend Chart

1. Click on the tag or drag it to a chart
2. Select **"Trend"** or **"Time Series"** view
3. Set the time range (e.g., "Last 15 minutes")
4. You should see:
   - A line graph showing message rate over time
   - Values updating in real-time
   - Timestamps for each data point

### Step 5: Explore the Data

Try these Explorer features:

**Zooming:**
- Drag to select a time range
- Zoom in/out to see details

**Data Table:**
- Switch to table view to see raw values
- View exact timestamps and values

**Statistics:**
- View min, max, average values
- See data quality metrics

---

## Troubleshooting

### Problem: Collector Not Connecting

**Error:** "Failed to connect to MQTT broker"

**Solutions:**
1. Verify the broker address and port:
   ```bash
   ping test.mosquitto.org
   telnet test.mosquitto.org 1883
   ```

2. Check firewall rules:
   ```bash
   # On Linux
   sudo ufw status
   sudo ufw allow 1883/tcp
   ```

3. Verify collector is active:
   ```bash
   docker logs collector-01 | grep -i "active"
   ```

### Problem: No Data Appearing

**Symptoms:** Connection successful but no data in Historian

**Solutions:**

1. **Verify topic is active:**
   ```bash
   mosquitto_sub -h test.mosquitto.org -t '$SYS/broker/load/messages/received/1min' -v
   ```
   You should see periodic messages

2. **Check subscription configuration:**
   - Ensure `Type` is correct (1, 2, or 3)
   - Verify `TagnameFields.Default` format (no special characters except `.` and `_`)
   - Confirm `Topics` is an array, not a string

3. **Review collector logs:**
   ```bash
   docker logs collector-01 --tail 50
   ```
   Look for parsing errors or exceptions

### Problem: Data Not Showing in Explorer

**Symptoms:** Data in Historian but not visible in Explorer

**Solutions:**

1. **Verify tag name in Historian API:**
   ```bash
   curl http://localhost:4511/api/tags
   ```
   Look for your tag in the JSON response

2. **Refresh Explorer:**
   - Clear browser cache
   - Refresh the tag list
   - Reconnect to Historian

3. **Check time range:**
   - Ensure the time range includes recent data
   - Try "Last 15 minutes" or "Last Hour"

### Problem: Connection Timeout

**Error:** "Connection timeout to test.mosquitto.org"

**Solutions:**

1. **Check Docker network:**
   ```bash
   docker exec collector-01 ping -c 3 test.mosquitto.org
   ```

2. **Verify DNS resolution:**
   ```bash
   docker exec collector-01 nslookup test.mosquitto.org
   ```

3. **Test with different broker:**
   Try `broker.hivemq.com` or `broker.emqx.io`

---

## Understanding Payload Types

Timebase MQTT Collector supports three different payload types, each designed for different data structures. **The payload format is determined by the data source** - your job is to identify which type matches the data you're receiving and configure the collector accordingly.

---

## How to Identify Which Type to Use

**Important:** You don't choose the payload format - the MQTT publisher does. Your task is to:
1. **Inspect** what the actual MQTT payload looks like
2. **Identify** which type matches that format
3. **Configure** the collector to parse it correctly

---

## Payload Type Comparison

| Feature | Type 1: Simple Primitives | Type 2: Simple JSON | Type 3: Structured JSON |
|---------|---------------------------|---------------------|-------------------------|
| **Data Format** | Single value (number, string, boolean) | Flat JSON object with key-value pairs | Nested JSON with arrays and metadata |
| **Topics per Tag** | One topic = one tag | One topic = multiple tags | One topic = multiple tags with metadata |
| **Timestamp Source** | Message arrival time | Message arrival time | Embedded in payload |
| **Quality/Status** | Not supported | Not supported | Supported (optional field) |
| **Complexity** | Simplest | Moderate | Advanced |
| **Performance** | Fastest | Fast | Slower (parsing overhead) |
| **Use Case** | Single sensor per topic | Multi-sensor device | Industrial systems, Sparkplug B |

### Identifying Type 1: Simple Primitives

**How to identify:**
1. Subscribe to the topic using `mosquitto_sub` or MQTT Explorer
2. Look at the raw payload

**The payload looks like Type 1 if you see:**
```bash
# Just a number
42.5

# Or a string
"online"

# Or a boolean
true
```

**You'll see this from:**
- Home automation sensors (Zigbee, Z-Wave bridges)
- Simple IoT devices (ESP8266, Arduino)
- Legacy industrial sensors
- Tasmota, ESPHome devices with single values
- Basic MQTT-enabled hardware

**Real example:**
```bash
$ mosquitto_sub -h broker.example.com -t 'home/livingroom/temperature' -v
home/livingroom/temperature 22.3
home/livingroom/temperature 22.4
home/livingroom/temperature 22.3
```

**Each line is a separate message with just the value.**

**Why this format exists:**
- Simplest to implement on embedded devices
- Lowest memory footprint
- No JSON parsing needed on publisher side
- Smallest network bandwidth

---

### Identifying Type 2: Simple JSON

**How to identify:**
1. Subscribe to the topic
2. Payload is valid JSON
3. JSON is a flat object (no nested objects or arrays)

**The payload looks like Type 2 if you see:**
```json
{"temperature": 22.3, "humidity": 45, "pressure": 1013}
```

**You'll see this from:**
- Modern IoT devices with multiple sensors
- Weather stations
- Custom applications using MQTT
- Node-RED flows
- Home Assistant MQTT integrations
- Devices with enough resources to format JSON

**Real example:**
```bash
$ mosquitto_sub -h broker.example.com -t 'weather/station1' -v
weather/station1 {"temp":22.3,"hum":45.2,"press":1013,"wind":5.2}
weather/station1 {"temp":22.4,"hum":45.1,"press":1013,"wind":5.3}
```

**Key characteristics:**
- It's valid JSON (starts with `{`, ends with `}`)
- All values at the top level (no nesting)
- All values are primitives (numbers, strings, booleans)
- No timestamp field (or you don't care about it)

**Why this format exists:**
- Atomicity: All measurements from the same instant
- Efficiency: One message for related data
- Self-documenting: Field names show what each value means
- Extensible: Easy to add new fields

---

### Identifying Type 3: Structured JSON

**How to identify:**
1. Subscribe to the topic
2. Payload is valid JSON
3. JSON has nested objects, arrays, or embedded timestamps

**The payload looks like Type 3 if you see:**
```json
{
  "timestamp": 1678901234000,
  "deviceId": "sensor-001",
  "metrics": [
    {"name": "temperature", "value": 22.3, "unit": "C"},
    {"name": "humidity", "value": 45.2, "unit": "%"}
  ]
}
```

**Or Sparkplug B format:**
```json
{
  "timestamp": 1678901234000,
  "metrics": [
    {"name": "Temp", "value": 22.3, "timestamp": 1678901234000},
    {"name": "Pressure", "value": 1013, "timestamp": 1678901234000}
  ],
  "seq": 123
}
```

**You'll see this from:**
- Industrial SCADA systems
- Ignition SCADA with Sparkplug B
- PLCs publishing through MQTT gateways
- Custom industrial applications
- Systems with store-and-forward (buffering)
- High-reliability systems needing quality flags

**Real example:**
```bash
$ mosquitto_sub -h broker.example.com -t 'spBv1.0/factory/DDATA/line1/plc001' -v
spBv1.0/factory/DDATA/line1/plc001 {"timestamp":1678901234000,"metrics":[{"name":"Speed","value":1500,"timestamp":1678901234000,"datatype":"Int32"},{"name":"Temp","value":75.5,"timestamp":1678901234000,"datatype":"Float"}],"seq":42}
```

**Key characteristics:**
- Contains a timestamp field (usually Unix milliseconds)
- Has arrays of objects (like `metrics[]`)
- May include quality/status indicators
- Often has metadata (units, datatypes, device IDs)
- Message structure is complex and nested

**Why this format exists:**
- Timestamp precision: Data timestamped at source, not arrival
- Quality tracking: Know if data is good/bad/uncertain
- Industrial standards: Compatible with Sparkplug B, ISA-95
- Reliability: Handles network delays and buffering
- Batch uploads: Multiple timestamped values in one message

---

## Step-by-Step: Identifying Your Payload Type

### Step 1: Subscribe and Inspect

Use `mosquitto_sub` to see raw messages:

```bash
# Subscribe to your topic
mosquitto_sub -h YOUR_BROKER -t 'YOUR/TOPIC' -v

# Or use wildcards to explore
mosquitto_sub -h YOUR_BROKER -t 'sensors/#' -v
```

### Step 2: Examine the Output

**If you see:**
```
sensors/temp 23.5
```
→ **Type 1**: Just a value

**If you see:**
```
sensors/data {"temp":23.5,"hum":45}
```
→ **Type 2**: Flat JSON, no timestamp

**If you see:**
```
sensors/data {"timestamp":1678901234000,"temp":23.5}
```
→ **Type 3**: JSON with timestamp field

**If you see:**
```
sensors/data {"metrics":[{"name":"temp","value":23.5}]}
```
→ **Type 3**: Nested JSON with arrays

### Step 3: Quick Decision Guide

```
Look at the payload:

Is it JSON?
├─ NO → Type 1
└─ YES → Does it have a "timestamp" or "ts" field?
    ├─ YES → Type 3
    └─ NO → Does it have arrays or nested objects?
        ├─ YES → Type 3
        └─ NO → Type 2 (simple flat JSON)
```

### Step 4: Test with a Simple Configuration

Start with what you think it is, then check the Historian:

**Try Type 1 first if unsure:**
```json
{
  "Type": 1,
  "Topics": ["your/topic"],
  "TagnameFields": {"Default": "test.tag"}
}
```

**If no data appears**, try Type 2:
```json
{
  "Type": 2,
  "Topics": ["your/topic"],
  "TagnameFields": {"Default": "test"}
}
```

**Still no data?** Check collector logs for parsing errors - you likely need Type 3.

---

## Detailed Examples

Let's explore each type in detail with complete configurations.

---

### Type 1: Simple Primitives

**What it is:** Single values (numbers, strings, booleans) published directly to a topic.

**Use case:** Simple sensors that publish a single value per message.

**Example MQTT Message:**
```
Topic: sensors/room1/temperature
Payload: 23.5
```

**Collector Configuration:**

```json
{
  "Name": "Simple-Temperature-Sensor",
  "Plugin": "MQTT",
  "Enabled": true,
  "Settings": {
    "Host": "broker.example.com",
    "Port": 1883,
    "UseTls": false,
    "Subscriptions": [
      {
        "Type": 1,
        "Topics": [
          "sensors/room1/temperature"
        ],
        "TagnameFields": {
          "Default": "building.room1.temperature"
        },
        "TagnameIncludesTopic": false,
        "TagnameDelimiter": ".",
        "QoS": 2
      }
    ]
  }
}
```

**Result in Historian:**
- Tag: `building.room1.temperature`
- Value: `23.5`
- Timestamp: Time the message was received

**With Topic Inclusion:**

If you set `"TagnameIncludesTopic": true`, the topic segments can be included in the tag name:

```json
{
  "Type": 1,
  "Topics": [
    "sensors/+/temperature"
  ],
  "TagnameFields": {
    "Default": "building"
  },
  "TagnameIncludesTopic": true,
  "TagnameDelimiter": ".",
  "QoS": 2
}
```

This would create tags like:
- Topic: `sensors/room1/temperature` → Tag: `building.sensors.room1.temperature`
- Topic: `sensors/room2/temperature` → Tag: `building.sensors.room2.temperature`

---

### Type 2: Simple JSON

**What it is:** Flat JSON objects with key-value pairs where each key becomes a separate tag.

**Use case:** Sensors that publish multiple related measurements in one message.

**Example MQTT Message:**
```
Topic: sensors/room1
Payload: {"temperature": 23.5, "humidity": 45.2, "pressure": 1013.25}
```

**Collector Configuration:**

```json
{
  "Name": "Multi-Sensor-JSON",
  "Plugin": "MQTT",
  "Enabled": true,
  "Settings": {
    "Host": "broker.example.com",
    "Port": 1883,
    "UseTls": false,
    "Subscriptions": [
      {
        "Type": 2,
        "Topics": [
          "sensors/room1"
        ],
        "TagnameFields": {
          "Default": "building.room1",
          "temperature": "temp",
          "humidity": "hum",
          "pressure": "press"
        },
        "TagnameIncludesTopic": false,
        "TagnameDelimiter": ".",
        "QoS": 2
      }
    ]
  }
}
```

**Result in Historian:**
- Tag: `building.room1.temp` → Value: `23.5`
- Tag: `building.room1.hum` → Value: `45.2`
- Tag: `building.room1.press` → Value: `1013.25`

**How TagnameFields Works:**

The `TagnameFields` object maps JSON keys to tag name suffixes:

```json
"TagnameFields": {
  "Default": "building.room1",      // Base tag name
  "temperature": "temp",            // JSON key "temperature" → tag suffix "temp"
  "humidity": "hum",                // JSON key "humidity" → tag suffix "hum"
  "pressure": "press"               // JSON key "pressure" → tag suffix "press"
}
```

Final tags are constructed as: `{Default}.{mapped_suffix}`

**Wildcards with Type 2:**

```json
{
  "Type": 2,
  "Topics": [
    "sensors/+/data"
  ],
  "TagnameFields": {
    "Default": "building",
    "temperature": "temp",
    "humidity": "hum"
  },
  "TagnameIncludesTopic": true,
  "TagnameDelimiter": ".",
  "QoS": 2
}
```

This would create:
- Topic: `sensors/room1/data` → Tags: `building.sensors.room1.data.temp`, `building.sensors.room1.data.hum`
- Topic: `sensors/room2/data` → Tags: `building.sensors.room2.data.temp`, `building.sensors.room2.data.hum`

---

### Type 3: Structured JSON

**What it is:** Complex nested JSON objects with timestamps, metadata, and multiple measurements.

**Use case:** Industrial protocols like Sparkplug B, or custom structured data with specific timestamp and quality fields.

**Example MQTT Message:**
```json
{
  "timestamp": 1678901234000,
  "metrics": [
    {
      "name": "temperature",
      "value": 23.5,
      "unit": "C",
      "quality": "good"
    },
    {
      "name": "humidity",
      "value": 45.2,
      "unit": "%",
      "quality": "good"
    }
  ],
  "device": "sensor-001"
}
```

**Collector Configuration:**

```json
{
  "Name": "Structured-Data-Sensor",
  "Plugin": "MQTT",
  "Enabled": true,
  "Settings": {
    "Host": "broker.example.com",
    "Port": 1883,
    "UseTls": false,
    "Subscriptions": [
      {
        "Type": 3,
        "Topics": [
          "sensors/structured/+"
        ],
        "TagnameFields": {
          "Default": "building",
          "device": "device",
          "name": "metric"
        },
        "TimestampFields": ["timestamp"],
        "TimestampType": 2,
        "ValueField": "value",
        "QualityField": "quality",
        "TagnameIncludesTopic": true,
        "TagnameDelimiter": ".",
        "QoS": 2
      }
    ]
  }
}
```

**Result in Historian:**
- Tag: `building.sensors.structured.sensor-001.temperature`
  - Value: `23.5`
  - Timestamp: `2023-03-15 10:20:34` (from the timestamp field)
  - Quality: `good`

- Tag: `building.sensors.structured.sensor-001.humidity`
  - Value: `45.2`
  - Timestamp: `2023-03-15 10:20:34`
  - Quality: `good`

**Type 3 Configuration Breakdown:**

| Field | Description |
|-------|-------------|
| `TimestampFields` | Array of JSON paths to timestamp fields (e.g., `["timestamp"]`) |
| `TimestampType` | Timestamp format: `1` = Unix seconds, `2` = Unix milliseconds, `3` = ISO8601/string |
| `TagnameFields` | Maps JSON keys to tag name components |
| `ValueField` | JSON path to the value field |
| `QualityField` | Optional JSON path to quality/status field |

---

## Next Steps

Congratulations! You've successfully configured MQTT data collection. Here's what to explore next:

### 1. Test Different Payload Types

Try creating test publishers for each type:

**Type 1 Test (using mosquitto_pub):**
```bash
mosquitto_pub -h broker.example.com -t "test/simple" -m "42.5"
```

**Type 2 Test:**
```bash
mosquitto_pub -h broker.example.com -t "test/json" -m '{"temp":23.5,"hum":45.2}'
```

**Type 3 Test:**
```bash
mosquitto_pub -h broker.example.com -t "test/structured" -m '{"timestamp":1678901234000,"metrics":[{"name":"temp","value":23.5}]}'
```

### 2. Use Wildcards for Multiple Sensors

Subscribe to multiple topics at once using MQTT wildcards:

- `+` matches a single level: `sensors/+/temperature` matches `sensors/room1/temperature` but not `sensors/floor1/room1/temperature`
- `#` matches multiple levels: `sensors/#` matches all topics under `sensors/`

**Example with wildcard:**
```json
{
  "Type": 1,
  "Topics": [
    "building/+/+/temperature",
    "building/+/+/humidity"
  ],
  "TagnameFields": {
    "Default": "facility"
  },
  "TagnameIncludesTopic": true,
  "TagnameDelimiter": ".",
  "QoS": 2
}
```

### 3. Connect to Your Own Broker

Deploy the included Mosquitto broker:

1. Uncomment `mqtt-broker` in `docker-compose.yml`
2. Run: `docker-compose up -d`
3. Configure collector to use `mqtt-broker:1883`

### 4. Add More Collectors

Scale to multiple data sources:

1. Uncomment `timebase-collector-02` in `docker-compose.yml`
2. Configure it to connect to a different broker
3. Run: `docker-compose up -d`

### 5. Advanced Configuration

Explore Timebase documentation for:
- **Structured JSON** payloads
- **SparkplugB** support
- **Custom timestamp** parsing
- **Metadata extraction**
- **QoS settings**

---

## Additional Resources

- **Timebase MQTT Documentation**: https://timebase.flow-software.com/en/knowledge-base/mqtt-collector
- **MQTT.org**: https://mqtt.org/
- **HiveMQ MQTT Essentials**: https://www.hivemq.com/mqtt-essentials/
- **Public MQTT Brokers**: https://github.com/mqtt/mqtt.org/wiki/public_brokers

---

## Summary

You've completed the MQTT Collector tutorial! You can now:

- ✅ Connect Timebase to MQTT brokers
- ✅ Subscribe to topics and collect data
- ✅ Configure payload parsing
- ✅ Store time-series data in the Historian
- ✅ Visualize data in Explorer
- ✅ Troubleshoot common issues

**Your working configuration:**
- Broker: `test.mosquitto.org`
- Topic: `$SYS/broker/load/messages/received/1min`
- Tag: `mosquitto.messages_per_minute`
- Collector: Active and collecting
- Explorer: Displaying real-time data

Happy data collecting! 📊
