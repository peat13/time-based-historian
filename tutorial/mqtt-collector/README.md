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
8. [Understanding Payload Types](#understanding-payload-types)
9. [Advanced: Secure Broker Authentication](#advanced-secure-broker-authentication)
10. [Advanced: Metadata Extraction with Fields](#advanced-metadata-extraction-with-fields)
11. [Advanced: Tag Filtering with Regex](#advanced-tag-filtering-with-regex)
12. [Next Steps](#next-steps)

---

<details open>
<summary><b>📖 Understanding MQTT and Timebase</b></summary>

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

</details>

---

<details open>
<summary><b>🌐 Finding a Public MQTT Broker</b></summary>

## Finding a Public MQTT Broker

For this tutorial, we'll use **test.mosquitto.org** as it's reliable and well-documented.

<details>
<summary><b>📋 Click to see other public broker options</b></summary>

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

</details>

⚠️ **Security Note:** Public brokers should only be used for testing. Never send sensitive data to public brokers!

</details>

---

<details open>
<summary><b>🔎 Discovering Available Topics</b></summary>

## Discovering Available Topics

**For this tutorial**, we'll use the broker statistics topic: `$SYS/broker/load/messages/received/1min`

This topic publishes the number of messages the broker receives per minute - perfect for testing!

<details>
<summary><b>🔍 Click to see methods for discovering topics</b></summary>

Before configuring the collector, you can explore available topics using these methods:

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

</details>

</details>

---

<details open>
<summary><b>⚙️ Configuring the Collector</b></summary>

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

Before adding configurations, make sure the collector container is active.

**IMPORTANT:** The `Active` setting is a **container-level environment variable**, not a setting in the Collector web UI or MQTT plugin JSON configuration.

#### What the Active Setting Controls

The `Active` environment variable controls whether the **entire collector service** runs and collects data. Each MQTT plugin configuration has its own `"Enabled": true` setting in the JSON, but the collector itself must be active for any data collection to occur.

**You won't see an "Active" toggle in the Collector web UI** at port 4521.

#### To Enable via Environment Variable

**In Docker Compose** - Edit your `.env` file:

```bash
# Collector must be active to collect data
COLLECTOR_01_ACTIVE=true
```

Then restart the collector:

```bash
docker-compose restart timebase-collector-01
```

**In docker-compose.yml** - The Active setting is already configured in the collector template:

```yaml
environment:
  Active: "false"  # Change to "true" in your .env file
```

#### Configuration File Locations

**Docker:** The configuration files are inside the container at:
- Settings: `/collector/settings` (mapped from volume)
- Config: `/collector/config` (mapped from volume)
- Data: `/collector/data` (mapped from volume)
- Logs: `/collector/logs` (mapped from volume)

These paths correspond to the environment variables in your `docker-compose.yml`.

**Windows:**
- Config: `C:\ProgramData\Flow Software\Timebase\Collector\Config`
- Settings: `C:\ProgramData\Flow Software\Timebase\Collector\Settings`
- Data: `C:\ProgramData\Flow Software\Timebase\Collector\Data`

### Step 3: Create MQTT Plugin Configuration

You have two options for creating MQTT plugin configurations:

#### Option A: Using the Collector Config UI (Recommended for Beginners)

The Collector provides a built-in web interface for managing configurations:

1. Navigate to `http://localhost:4521` in your browser
2. Click the **Config** tab
3. Use the visual editor to:
   - Select "MQTT" as the Data Source Type
   - Configure Target Historian connection
   - Edit MQTT Settings JSON

**Configuration files modified by the UI:**
- `collector.config` - Data source and historian settings
- `historians.config` - Target historian list
- `settings.config` - Plugin-specific settings (MQTT, OPC, etc.)

#### Option B: Manually Editing JSON Files

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
| `QoS` | `2` | MQTT Quality of Service: `0`=At most once, `1`=At least once, `2`=Exactly once (**default** - recommended for industrial data) |

---

### 🐳 Understanding Docker Networking (IMPORTANT!)

**This is critical for Timebase users running in Docker:**

When configuring connections between Timebase services (Collector → Historian, Explorer → Historian, Collector → MQTT Broker), you **must** understand how Docker networks work.

#### Why `localhost` and `127.0.0.1` Don't Work in Docker

Each Docker container has its **own isolated network namespace**. Inside a container:
- `localhost` and `127.0.0.1` refer to **that container only**, not your host machine
- `0.0.0.0` listens on all interfaces but doesn't help with connecting **between** containers

**❌ Common mistakes:**
```json
{
  "Host": "localhost",        // WRONG - points to the container itself
  "Host": "127.0.0.1",        // WRONG - same issue
  "Host": "0.0.0.0"           // WRONG - not a valid connection address
}
```

#### ✅ Use Container Hostnames Instead

Docker Compose creates a **bridge network** where containers can reach each other using **service names as hostnames**.

**In `docker-compose.yml`, the service names are:**
- `timebase-historian` → Use hostname `historian`
- `timebase-collector-01` → Use hostname `collector-01`
- `timebase-explorer` → Use hostname `explorer`

**Correct configuration examples:**

**Collector connecting to Historian:**
```json
{
  "Name": "Local-Historian",
  "Url": "http://historian:4511"    // ✅ Use service hostname
}
```

**Explorer connecting to Historian:**
```
Host: historian                     // ✅ Use service hostname
Port: 4511
```

**Collector connecting to MQTT broker in Docker:**
```json
{
  "Host": "mqtt-broker",            // ✅ If broker is in docker-compose
  "Port": 1883
}
```

#### When to Use Different Addresses

| Scenario | Address to Use | Example |
|----------|----------------|---------|
| **Container → Container** (same Docker network) | Service hostname | `historian`, `collector-01` |
| **Container → External Service** (internet/LAN) | Domain name or IP | `test.mosquitto.org`, `192.168.1.100` |
| **Your Browser → Container** (accessing web UI) | `localhost` or host IP | `http://localhost:4531` |
| **Container → Host Machine** (special case) | `host.docker.internal` | `host.docker.internal:1883` (Windows/Mac) |

#### Real-World Example

**Scenario:** You have an MQTT broker running on your host machine (not in Docker), and want the Timebase Collector (in Docker) to connect to it.

**❌ Wrong:**
```json
{
  "Host": "localhost",      // Won't work - refers to the container
  "Port": 1883
}
```

**✅ Correct (Windows/Mac):**
```json
{
  "Host": "host.docker.internal",   // Special hostname for host machine
  "Port": 1883
}
```

**✅ Correct (Linux):**
```json
{
  "Host": "172.17.0.1",            // Docker bridge gateway IP
  "Port": 1883
}
```
Or use `--network host` mode (advanced).

#### Quick Reference

- **Public internet broker:** Use the domain name (`test.mosquitto.org`)
- **Broker in same Docker Compose stack:** Use service name (`mqtt-broker`)
- **Broker on host machine (Windows/Mac):** Use `host.docker.internal`
- **Broker on LAN:** Use LAN IP address (`192.168.1.50`)
- **Accessing web UI from browser:** Use `localhost:PORT` or `HOST-IP:PORT`

**💡 Tip:** If you see connection errors, check whether you're using the correct hostname for your setup. This is the #1 cause of "connection refused" errors in Docker!

---

### Step 4: Save and Apply Configuration

1. Save the configuration
2. The collector should automatically connect to the broker
3. Data collection begins immediately

</details>

---

<details open>
<summary><b>✅ Verifying Data Collection</b></summary>

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

</details>

---

<details open>
<summary><b>📊 Viewing Data in Explorer</b></summary>

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

</details>

---

<details>
<summary><b>🔧 Troubleshooting</b></summary>

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

### Problem: "Connection Refused" with localhost/127.0.0.1

**Symptoms:**
- Collector shows "Connection refused" to `localhost:4511` or `127.0.0.1:4511`
- Explorer can't connect to Historian at `localhost:4511`
- Collector can't connect to MQTT broker at `localhost:1883`

**Root Cause:** You're using `localhost` or `127.0.0.1` inside Docker container configuration, which refers to the **container itself**, not your host machine or other containers.

**Solutions:**

#### 1. Connecting Between Containers (Collector → Historian, Explorer → Historian)

**❌ Wrong Configuration:**
```json
{
  "Url": "http://localhost:4511"
}
```

**✅ Correct Configuration:**
```json
{
  "Url": "http://historian:4511"    // Use service hostname from docker-compose.yml
}
```

**For Explorer:** Use `historian` as the hostname, port `4511`

#### 2. Connecting to Broker on Host Machine

**If your MQTT broker is running on your host** (not in Docker):

**❌ Wrong:**
```json
{
  "Host": "localhost",
  "Port": 1883
}
```

**✅ Correct (Windows/Mac):**
```json
{
  "Host": "host.docker.internal",   // Special Docker hostname for host
  "Port": 1883
}
```

**✅ Correct (Linux):**
```json
{
  "Host": "172.17.0.1",            // Default Docker bridge gateway
  "Port": 1883
}
```

Alternatively, find your actual LAN IP and use that:
```bash
# Windows
ipconfig

# Linux/Mac
ip addr show
ifconfig
```

Then use your LAN IP like `192.168.1.100`.

#### 3. Connecting to Broker in Same Docker Compose Stack

If you've added an MQTT broker to your `docker-compose.yml`:

```json
{
  "Host": "mqtt-broker",           // Use the service name from docker-compose.yml
  "Port": 1883
}
```

#### 4. Verify Docker Network

Check that containers are on the same network:

```bash
# List networks
docker network ls

# Inspect the timebase network
docker network inspect timebase-network

# Verify containers can see each other
docker exec collector-01 ping historian
docker exec collector-01 nslookup historian
```

#### 5. Quick Test

Test connectivity from inside the collector container:

```bash
# Test Historian connectivity
docker exec collector-01 curl http://historian:4511/api/status

# Test MQTT broker connectivity (if using telnet)
docker exec collector-01 telnet mqtt-broker 1883
```

**Key Takeaway:** When configuring services **inside Docker containers**, always use:
- **Container hostnames** for services in the same Docker Compose stack
- **host.docker.internal** (or LAN IP) for services on the host machine
- **Domain names or IPs** for external services

**See also:** [Understanding Docker Networking](#-understanding-docker-networking-important) in the Configuration section.

</details>

---

<details open>
<summary><b>📦 Understanding Payload Types</b></summary>

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
| **Data Format** | Single value (number, string, boolean) | JSON with path-based tagnames (flat or nested) | JSON with dedicated name/value fields |
| **Tagname Source** | Topic path | JSON structure/path | Dedicated field (e.g., `"name"`) |
| **Topics per Tag** | One topic = one tag | One topic = multiple tags | One topic = multiple tags |
| **Timestamp Source** | Message arrival time | Message arrival time | Embedded in payload (optional) |
| **Quality/Status** | Not supported | Not supported | Supported (via QualityField) |
| **Nesting** | N/A | Fully supported | Fully supported |
| **Complexity** | Simplest | Moderate | Advanced |
| **Performance** | Fastest | Fast | Slower (more parsing) |
| **Use Case** | Single sensor per topic | Multi-sensor devices, APIs | Industrial SCADA, Sparkplug B |

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
3. JSON uses **path-based tagnames** - the JSON structure itself defines tag names

**The payload looks like Type 2 if you see:**
```json
{"temperature": 22.3, "humidity": 45, "pressure": 1013}
```

**Or even nested JSON:**
```json
{
  "motor1": {
    "rpm": 23.6,
    "location": {
      "long": 12.3,
      "lat": 34.7
    }
  }
}
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
- Uses JSON **paths** as tagnames (e.g., `motor1.rpm`, `motor1.location.long`)
- Can be flat OR nested - **nesting is supported**
- Values are extracted from the JSON structure
- No dedicated "name" or "value" fields (unlike Type 3)

**Why this format exists:**
- Self-documenting: JSON structure defines tag hierarchy
- Flexible nesting: Can represent complex device structures
- Efficient: One message for multiple related measurements
- Natural fit for modern APIs and devices

---

### Identifying Type 3: Structured JSON

**How to identify:**
1. Subscribe to the topic
2. Payload is valid JSON
3. JSON uses **dedicated fields** for tagnames, values, and metadata (like `"name"`, `"value"`, `"timestamp"`)
4. Often has an array of measurements (`metrics[]`) rather than direct key-value pairs

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
├─ NO → Type 1 (Simple Primitives)
└─ YES → Does it have dedicated "name" and "value" fields for metrics?
    ├─ YES → Type 3 (Structured JSON - data-based tagnames)
    └─ NO → Type 2 (Simple JSON - path-based tagnames)
```

**Key distinction:**
- **Type 2**: The JSON **structure** defines tag names (`motor1.rpm` from `{"motor1": {"rpm": 23.6}}`)
- **Type 3**: A specific **field** defines tag names (`"name": "temperature"` in a metrics array)

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

</details>

---

<details>
<summary><b>📚 Click to see detailed configuration examples</b></summary>

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

**How Type 1 Tagnames Work:**

By default, Type 1 uses the **topic path as the tagname**. The `TagnameFields.Default` parameter allows you to override this base name, and `TagnameIncludesTopic` controls whether topic segments are appended.

In the example above:
- `TagnameIncludesTopic: false` → Uses only the `Default` value → Tag: `building.room1.temperature`
- `TagnameIncludesTopic: true` → Appends topic segments to `Default` → Tag: `building.room1.temperature.sensors.room1.temperature`

**With Topic Inclusion:**

If you set `"TagnameIncludesTopic": true`, the topic segments are appended to the base tag name:

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
  "Default": "building.room1",
  "temperature": "temp",
  "humidity": "hum",
  "pressure": "press"
}
```

**Field mapping explanation:**
- `"Default": "building.room1"` - Base tag name
- `"temperature": "temp"` - JSON key "temperature" → tag suffix "temp"
- `"humidity": "hum"` - JSON key "humidity" → tag suffix "hum"
- `"pressure": "press"` - JSON key "pressure" → tag suffix "press"

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

**Advanced: Nested JSON with Type 2:**

Type 2 supports **deeply nested JSON** structures. The JSON path becomes the tag name:

**Example Payload:**
```json
{
  "payload": {
    "timestamp": "2024-07-12T12:15"
  },
  "motor1": {
    "rpm": 23.6,
    "location": {
      "long": 12.3,
      "lat": 34.7
    }
  },
  "motor2": {
    "rpm": 28.6,
    "location": {
      "long": 22.4,
      "lat": 36.8
    }
  }
}
```

**Configuration:**
```json
{
  "Type": 2,
  "Topics": ["factory/motors"],
  "TagnameFields": {
    "Default": "facility"
  },
  "TagnameDelimiter": ".",
  "QoS": 2
}
```

**Result in Historian:**
- `facility.motor1.rpm` → `23.6`
- `facility.motor1.location.long` → `12.3`
- `facility.motor1.location.lat` → `34.7`
- `facility.motor2.rpm` → `28.6`
- `facility.motor2.location.long` → `22.4`
- `facility.motor2.location.lat` → `36.8`

Notice how the **nested structure** automatically creates hierarchical tag names!

#### Important: Array Handling in Type 2 (Observed Behavior)

**Note:** Based on testing, Type 2 appears to skip array element keys when constructing tagnames.

**Example with arrays:**
```json
{
  "data": [
    {
      "motor1": {
        "rpm": 23.6
      }
    }
  ]
}
```

**Observed result:** Tag name is `motor1.rpm`, **NOT** `data.motor1.rpm`

The array element name `"data"` appears to be skipped when constructing the tagname. This is different from Type 3, where you explicitly define which fields contain names using `TagnameFields`.

⚠️ **This behavior is not explicitly documented in official Timebase docs. Please verify with your specific use case.**

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

#### Timestamp Types Reference

| Type | Format | Example | Description |
|------|--------|---------|-------------|
| `1` | Unix seconds | `1689526200.234456` | Seconds since epoch (can include fractional seconds) |
| `2` | Unix milliseconds | `1689526200234.456` | Milliseconds since epoch (**default** for Type 3) |
| `3` | ISO 8601 string | `2024-07-28T14:30:03Z` or `2024-07-28T12:30:03+02:00` | ISO 8601 formatted date-time strings with timezone |

**Note:** If your payload includes a timestamp field, always specify the correct `TimestampType` to ensure accurate data timestamping. Using the wrong type will result in incorrect timestamps in the Historian.

</details>

---

<details>
<summary><b>🔐 Advanced: Secure Broker Authentication</b></summary>

## Advanced: Secure Broker Authentication

For production environments, you'll need to connect to private brokers requiring authentication.

### Basic Authentication

**Configuration with Username and Password:**

```json
{
  "Name": "Private-Broker",
  "Plugin": "MQTT",
  "Enabled": true,
  "Settings": {
    "Host": "private-broker.example.com",
    "Port": 8883,
    "UseTls": true,
    "Username": "your-username",
    "Password": "your-password",
    "ClientId": "timebase-collector-01",
    "Subscriptions": [
      {
        "Type": 1,
        "Topics": ["sensors/#"],
        "TagnameFields": {"Default": "production"},
        "QoS": 2
      }
    ]
  }
}
```

### Configuration Fields:

| Field | Required | Description |
|-------|----------|-------------|
| `Username` | Optional | MQTT broker username |
| `Password` | Optional* | MQTT broker password |
| `ClientId` | Optional | Custom MQTT client identifier (defaults to auto-generated) |
| `UseTls` | Optional | Enable TLS/SSL encryption (default: `false`) |
| `Port` | Optional | Broker port (default: 1883 for non-TLS, 8883 for TLS) |

**Important:** If `Username` is provided, `Password` must also be provided.

### TLS/SSL Configuration

For secure connections:

```json
{
  "Host": "secure-broker.example.com",
  "Port": 8883,
  "UseTls": true,
  "Username": "collector",
  "Password": "secure-password",
  "ClientId": "timebase-collector-01"
}
```

**Ports:**
- **1883**: Standard MQTT (no encryption)
- **8883**: MQTT over TLS/SSL (encrypted)

### Client ID Best Practices

The `ClientId` uniquely identifies your collector to the broker:

- **Auto-generated** (default): Timebase creates a unique ID
- **Custom**: Use descriptive names like `timebase-collector-01`, `factory-floor-collector`
- **Multiple collectors**: Each must have a unique Client ID

**Example with multiple collectors:**
```json
{
  "ClientId": "timebase-collector-factory-floor-1"
}
```

</details>

---

<details>
<summary><b>🏷️ Advanced: Metadata Extraction with Fields</b></summary>

## Advanced: Metadata Extraction with Fields

Timebase can extract metadata from MQTT topics and payloads to enrich your tags with contextual information. This is powerful for organizing industrial data hierarchically.

### Static Metadata

Apply the same metadata to all tags in a subscription:

```json
{
  "Type": 1,
  "Topics": ["factory/line1/#"],
  "TagnameFields": {"Default": "production"},
  "Fields": {
    "Enterprise": "ACME Corp",
    "Site": "Austin",
    "Area": "Packaging"
  },
  "QoS": 2
}
```

**Result:** Every tag from `factory/line1/#` will have these metadata fields attached in the Historian.

### Dynamic Metadata from Topics

Extract values from the topic structure using `TopicDefinition`:

```json
{
  "Type": 1,
  "Topics": ["ACME/Austin/Packaging/+"],
  "TopicDefinition": "enterprise/site/area/machine",
  "TagnameFields": {"Default": "production"},
  "Fields": {
    "Enterprise": "[Topic:enterprise]",
    "Site": "[Topic:site]",
    "Area": "[Topic:area]",
    "Machine": "[Topic:machine]"
  },
  "QoS": 2
}
```

**How it works:**
- `TopicDefinition` creates a semantic map of the topic structure
- `[Topic:segment]` syntax extracts values from specific topic segments
- Topic `ACME/Austin/Packaging/Capper1` produces:
  - `Enterprise` = "ACME"
  - `Site` = "Austin"
  - `Area` = "Packaging"
  - `Machine` = "Capper1"

### Dynamic Metadata from Payload

Extract metadata from the JSON payload:

```json
{
  "Type": 2,
  "Topics": ["sensors/data"],
  "TagnameFields": {"Default": "facility"},
  "Fields": {
    "DeviceID": "[Payload:deviceId]",
    "Location": "[Payload:location.building]"
  },
  "QoS": 2
}
```

**Example payload:**
```json
{
  "deviceId": "sensor-042",
  "location": {
    "building": "Building-A",
    "floor": 2
  },
  "temp": 23.5
}
```

**Result:**
- Tags will have metadata:
  - `DeviceID` = "sensor-042"
  - `Location` = "Building-A"

### Complete Example: ISA-95 Hierarchy

```json
{
  "Type": 2,
  "Topics": ["+/+/+/+/data"],
  "TopicDefinition": "enterprise/site/area/line/data",
  "TagnameFields": {"Default": "manufacturing"},
  "Fields": {
    "Enterprise": "[Topic:enterprise]",
    "Site": "[Topic:site]",
    "Area": "[Topic:area]",
    "Line": "[Topic:line]",
    "DeviceType": "[Payload:metadata.type]"
  },
  "TagnameIncludesTopic": true,
  "TagnameDelimiter": ".",
  "QoS": 2
}
```

**Topic:** `ACME/Austin/Packaging/Line3/data`
**Payload:**
```json
{
  "metadata": {"type": "PLC"},
  "motor1_rpm": 1500,
  "motor1_temp": 65.4
}
```

**Result:**
- Tag: `manufacturing.ACME.Austin.Packaging.Line3.data.motor1_rpm`
  - Value: `1500`
  - Metadata:
    - `Enterprise`: "ACME"
    - `Site`: "Austin"
    - `Area`: "Packaging"
    - `Line`: "Line3"
    - `DeviceType`: "PLC"

</details>

---

<details>
<summary><b>🎯 Advanced: Tag Filtering with Regex</b></summary>

## Advanced: Tag Filtering with Regex

Exclude unwanted tags using the `Filter` field with regex patterns.

### Basic Filtering

```json
{
  "Type": 2,
  "Topics": ["sensors/#"],
  "TagnameFields": {"Default": "facility"},
  "Filter": "test|debug",
  "QoS": 2
}
```

**Effect:** Excludes any tagname containing "test" OR "debug"

### Pattern Examples

```json
{
  "Filter": "Line 1"
}
```
Excludes tagnames containing "Line 1"

```json
{
  "Filter": ".*RPM$"
}
```
Excludes tagnames ending with "RPM"

```json
{
  "Filter": "^temp_.*|.*_debug$"
}
```
Excludes tagnames starting with "temp_" OR ending with "_debug"

### Regex Escape Characters

Use `\\` for backslashes in regex:

```json
{
  "Filter": "\\\\authors"
}
```

**Note:** The `Filter` field defines what to **exclude**, not what to include.

</details>

---

<details open>
<summary><b>🚀 Next Steps</b></summary>

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

You've learned the basics! Now explore advanced features:

- **Metadata extraction**: Use `Fields` and `TopicDefinition` (see [Metadata section](#advanced-metadata-extraction-with-fields))
- **Tag filtering**: Use `Filter` with regex to exclude unwanted tags (see [Filtering section](#advanced-tag-filtering-with-regex))
- **Authentication**: Connect to secure brokers (see [Authentication section](#advanced-secure-broker-authentication))
- **Unit of Measure**: Extract UOM from Type 3 payloads using `UOMFields` (see below)
- **Sparkplug B**: Full support for Sparkplug B protocol (Type 3)

#### Extracting Unit of Measure (Type 3)

The `UOMFields` parameter is an **array** of JSON paths to extract units of measure from your payload.

**Configuration:**
```json
{
  "Type": 3,
  "Topics": ["sensors/+"],
  "TagnameFields": {
    "Default": "facility",
    "device": "device",
    "name": "metric"
  },
  "ValueField": "value",
  "UOMFields": ["uom"],
  "TimestampFields": ["timestamp"],
  "TimestampType": 2,
  "QoS": 2
}
```

**Example Payload:**
```json
{
  "timestamp": 1678901234000,
  "device": "sensor-042",
  "metrics": [
    {
      "name": "temperature",
      "value": 23.5,
      "uom": "°C"
    },
    {
      "name": "pressure",
      "value": 1013,
      "uom": "hPa"
    }
  ]
}
```

**Result in Historian:**
- Tag: `facility.sensors.sensor-042.temperature`
  - Value: `23.5`
  - Unit: `°C` ← Extracted from `uom` field

- Tag: `facility.sensors.sensor-042.pressure`
  - Value: `1013`
  - Unit: `hPa`

**Note:** `UOMFields` is an array because Timebase tries each path in order. If multiple paths are specified, the last one found is used, or they can be concatenated depending on configuration.

</details>

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
