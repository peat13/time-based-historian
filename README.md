# Time Based Local Historian Stack

```plaintext
████████╗ ██╗ ███╗   ███╗ ███████╗    ██████╗   █████╗  ███████╗ ███████╗ ██████╗ 
╚══██╔══╝ ██║ ████╗ ████║ ██╔════╝    ██╔══██╗ ██╔══██╗ ██╔════╝ ██╔════╝ ██╔══██╗
   ██║    ██║ ██╔████╔██║ █████╗      ██████╔╝ ███████║ ███████╗ █████╗   ██║  ██║
   ██║    ██║ ██║╚██╔╝██║ ██╔══╝      ██╔══██╗ ██╔══██║ ╚════██║ ██╔══╝   ██║  ██║
   ██║    ██║ ██║ ╚═╝ ██║ ███████╗    ██████╔╝ ██║  ██║ ███████║ ███████╗ ██████╔
   ╚═╝    ╚═╝ ╚═╝     ╚═╝ ╚══════╝    ╚═════╝  ╚═╝  ╚═╝ ╚══════╝ ╚══════╝ ╚═════╝ 
```

A complete, beginner-friendly Docker Compose stack for deploying a local Industrial Historian infrastructure. This stack provides everything you need to start collecting, storing, and visualizing time-series data.

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

## What's Included

- **Timebase Historian** (port 4511) - Time-series data storage engine
- **Timebase Explorer** (port 4531) - Web UI for data visualization
- **Timebase Collector** (port 4521) - Data collection service

## Quick Start

### 🎓 New to Docker or Portainer?

**Start here:** [Our Complete Beginner's Tutorial](tutorial/getting-started.md)

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

> **Note**: For managing files and configurations, we recommend deploying the [File-Browser](https://github.com/DMDuFresne/File-Browser) stack alongside this historian infrastructure.

## Adding Multiple Collectors

One of the most powerful features of this stack is the ability to easily add multiple collectors. Each collector can gather data from different sources.

**To add a second collector:**

1. Open `docker-compose.yml`
2. Find and uncomment the `timebase-collector-02` service in the Services section
3. Find and uncomment the `collector-02-data` volume in the Volumes section
4. Run: `docker-compose up -d`

That's it! The second collector will be available at **http://localhost:4522**

You can add as many collectors as you need - just follow the same pattern for `collector-03`, `collector-04`, etc.

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

## Data Management

All data is stored in Docker named volumes for persistence and portability:

- `timebase-historian-data` - Time-series database storage
- `timebase-collector-01-data` - Collector 01 configurations and cache
- `timebase-explorer-data` - Explorer settings and dashboards

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

### Backing Up Your Data

**To backup your data:**
```bash
docker volume ls                              # List all volumes
docker volume inspect timebase-historian-data # View volume details

# Backup a volume (example)
docker run --rm -v timebase-historian-data:/data -v $(pwd):/backup ubuntu tar czf /backup/historian-backup.tar.gz /data
```

**To restore from backup:**
```bash
docker run --rm -v timebase-historian-data:/data -v $(pwd):/backup ubuntu tar xzf /backup/historian-backup.tar.gz -C /data --strip 1
```

## Additional Resources

### Timebase
- **Timebase Documentation**: <https://timebase.flow-software.com/en/knowledge-base>

### Docker & Portainer
- **Docker Installation**: <https://docs.docker.com/engine/install/>
- **Docker Compose**: <https://docs.docker.com/compose/>
- **Portainer Installation**: <https://docs.portainer.io/start/install-ce/server/docker/linux>

### Tutorials
- **Getting Started**: [Complete setup guide](tutorial/getting-started.md) - Install Docker, Portainer, and deploy this stack
- **MQTT Collector**: [Collecting real-world data](tutorial/mqtt-collector/) - Connect to MQTT brokers and collect data

## License

This project is licensed under the [MIT License](LICENSE).
