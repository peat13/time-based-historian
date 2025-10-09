# Free Industrial Historian Docker Stack | Timebase Historian for IoT & SCADA

![Hero Image](images/Hero-image.png)

A complete, beginner-friendly Docker Compose stack for deploying a **free industrial historian** infrastructure. Perfect for **manufacturing**, **IoT data logging**, **SCADA systems**, and **OT/IT convergence**. This open-source historian stack provides everything you need to collect, store, and visualize industrial time-series data from PLCs, sensors, MQTT brokers, OPC UA servers, and Modbus devices.

---

## ⚠️ Disclaimer

**This is an unofficial, community-maintained guide** and is **not affiliated with or endorsed by Flow Software** (the creators of Timebase). While I strive to keep this documentation accurate and up-to-date, discrepancies may exist.

**For official documentation and authoritative guidance, always refer to:**
- **[Official Timebase Knowledge Base](https://timebase.flow-software.com/en/knowledge-base)** - The definitive source for Timebase documentation

If you find any conflicts between this guide and the official documentation, please:
1. Trust the official documentation first
2. Open an issue on this repository so it can be corrected

This guide is provided as-is, without warranty, to help users get started with Timebase in Docker environments.

---

## Why Choose This Industrial Historian?

✨ **Completely FREE Industrial Data Historian** - No licensing fees, no tag limits, no user restrictions
- **Unlimited tags, users, and queries** - Scale without licensing costs
- **Free forever guarantee** from Flow Software
- **Zero hidden costs** or premium tiers for core historian functionality
- **Store and Forward reliability** - No data loss during network outages
- **High-performance ingestion** - 25,000 changes/second data collection
- **REST API** for custom industrial integrations
- **Docker-ready** for easy deployment in industrial environments

### 🛡️ Industrial-Grade Store and Forward

Enterprise reliability for manufacturing and process control:
- **Automatic buffering** to local disk during network outages
- **Continuous data collection** from PLCs, sensors, and industrial devices
- **Auto-sync** when historian connection restores
- **Zero data loss** guarantee for critical industrial processes

**Best Practice:** Deploy data collectors near your industrial equipment (separate from historian server) to maximize Store and Forward benefits.

## What's Included in This Historian Stack

- **Timebase Historian** (port 4511) - Industrial time-series database engine
- **Timebase Explorer** (port 4531) - Web-based SCADA visualization and HMI dashboard
- **Timebase Collector** (port 4521) - OPC UA, Modbus, MQTT data collection service

## ⚠️ Important: Activating Collectors

**Collectors are disabled by default.** To start collecting data:

1. Edit your `.env` file:
   ```bash
   COLLECTOR_01_ACTIVE=true
   ```

2. Restart the collector:
   ```bash
   docker-compose restart timebase-collector-01
   ```

Without this, your collector will run but **will not collect any data**.

## Quick Start

### 🎓 New to Docker or Portainer?

**Start here:** [Our Complete Beginner's Tutorial](tutorial/README.md)

Our step-by-step guide walks you through:
- Installing Docker on Ubuntu Server
- Installing Portainer for easy management
- Deploying this stack via Portainer's web interface
- Troubleshooting common issues

**Want to learn more?** Check out the official documentation:
- [Docker Installation Guide](https://docs.docker.com/engine/install/) - Install Docker on various platforms
- [Portainer Installation Guide](https://docs.portainer.io/start/install-ce/server/docker/linux) - Install Portainer Community Edition
- [Timebase Knowledgebase](https://timebase.flow-software.com/en/knowledge-base) - Get to know Timebase

### 🚀 Already Have Docker?

Get up and running in 3 simple steps:

```bash
# 1. Clone the repository
git clone https://github.com/DMDuFresne/time-based-historian.git
cd time-based-historian

# 2. (Optional) Customize your configuration
cp example.env .env
# Edit .env if you need to change ports or other settings

# 3. Start the stack
docker-compose up -d
```

Access Timebase Explorer at **http://localhost:4531** and start exploring your data!

## Multi-Source Industrial Data Collection

One of the most powerful features of this historian stack is the ability to easily add multiple data collectors for different industrial protocols and data sources.

**Supported Industrial Protocols:**
- **OPC UA** - Modern industrial automation standard
- **Modbus TCP/RTU** - Legacy PLC and RTU communication
- **MQTT** - IoT sensors and edge devices
- **REST APIs** - Custom industrial integrations
- **And more** - Extensible plugin architecture

**To add a second collector for different data sources:**

1. Open `docker-compose.yml`
2. Find and uncomment the `timebase-collector-02` service in the Services section
3. Find and uncomment the `collector-02-data` volume in the Volumes section
4. Run: `docker-compose up -d`

That's it! The second collector will be available at **http://localhost:4522**

You can add as many collectors as you need for distributed industrial data collection - just follow the same pattern for `collector-03`, `collector-04`, etc.

## Configuration

The stack works out of the box with sensible defaults, but you can customize everything via environment variables.

**Common customizations:**

```bash
# Use specific versions instead of 'latest'
HISTORIAN_TAG=v2.1.0
COLLECTOR_TAG=v2.1.0
EXPLORER_TAG=v2.1.0

# Change ports if defaults are taken
HISTORIAN_PORT=5511
EXPLORER_PORT=5531
COLLECTOR_PORT=5521

# Set timezone
TZ=America/New_York

# Start collector 01 immediately
COLLECTOR_01_ACTIVE=true
```

See `example.env` for all available options with detailed comments.

## Deploying in Portainer

This stack works great with Portainer for easy management via a web interface.

1. **Log in to Portainer** and navigate to **Stacks**

2. **Add a new stack** with these settings:
   - Name: `time-based-historian`
   - Build method: **Git Repository**
   - Repository URL: `https://github.com/DMDuFresne/time-based-historian.git`
   - Branch: `main`

3. **Customize environment variables** (optional):
   - Use the **Environment variables** section to override any defaults
   - See `example.env` for available options

4. **Deploy the stack** and access Timebase Explorer at **http://your-server:4531**

## Useful Commands

```bash
# View status of all services
docker-compose ps

# View logs (all services)
docker-compose logs -f

# View logs (specific service)
docker-compose logs -f timebase-historian

# Stop the stack (containers removed, data preserved)
docker-compose down

# Stop and remove data (⚠️ deletes everything!)
docker-compose down -v

# Update to latest images
docker-compose pull
docker-compose up -d

# Restart a specific service
docker-compose restart timebase-collector
```

## Industrial Data Management & Historian Database

All industrial time-series data is stored in Docker named volumes for persistence and portability:

- `timebase-historian-data` - Industrial historian time-series database storage
- `timebase-collector-01-data` - Data collector configurations and local cache
- `timebase-explorer-data` - SCADA/HMI explorer settings and dashboards

### ⚠️ Data Persistence Warning

**IMPORTANT:** All your historical data is stored in Docker named volumes. If you run:

```bash
docker-compose down -v  # The -v flag DELETES ALL VOLUMES
```

**You will lose ALL historical data permanently.** This cannot be undone.

**For safe shutdown:**
```bash
docker-compose down  # Stops containers but preserves volumes
```

### Backing Up Industrial Historian Data

**To backup your historian time-series data:**
```bash
docker volume ls                              # List all volumes
docker volume inspect timebase-historian-data # View volume details

# Backup a volume (example)
docker run --rm -v timebase-historian-data:/data -v $(pwd):/backup ubuntu tar czf /backup/historian-backup.tar.gz /data
```

**To restore historian data from backup:**
```bash
docker run --rm -v timebase-historian-data:/data -v $(pwd):/backup ubuntu tar xzf /backup/historian-backup.tar.gz -C /data --strip 1
```

## Use Cases for This Industrial Historian

This free historian stack is perfect for:
- **Manufacturing Data Logging** - Track production metrics, OEE, and equipment performance
- **SCADA Historian** - Long-term storage for SCADA systems and process control data
- **IoT Industrial Data** - Collect and analyze sensor data from industrial IoT devices
- **OPC UA Data Archival** - Store historical PLC and automation data
- **Energy Management** - Monitor power consumption and environmental metrics
- **Predictive Maintenance** - Analyze equipment trends and predict failures
- **Quality Tracking** - Store quality metrics and traceability data

## Additional Resources

### Industrial Historian Documentation
- **Timebase Documentation**: <https://timebase.flow-software.com/en/knowledge-base>

### Docker & Container Deployment
- **Docker Installation**: <https://docs.docker.com/engine/install/>
- **Docker Compose**: <https://docs.docker.com/compose/>
- **Portainer Installation**: <https://docs.portainer.io/start/install-ce/server/docker/linux>

### Getting Started Tutorials
- **Complete Setup Guide**: [Install Docker, Portainer, and deploy this historian stack](tutorial/README.md)
- **MQTT Data Collection**: [Collect IoT and industrial data from MQTT brokers](tutorial/mqtt-collector/)
- **OPC UA Integration**: Configure OPC UA data collection from PLCs and automation systems

## License

This project is licensed under the [MIT License](LICENSE).

---

<div align="center">

### Ready to Take Your Industrial Architecture to the Next Level?

<br>
<br>

<a href="https://abelara.com" target="_blank">
  <img src="images/Abelara_Logo_Primary_FullColor_W_RGB.png" alt="Abelara - Industrial Automation & Controls" width="300">
</a>

[Visit Abelara.com →](https://abelara.com)

</div>
