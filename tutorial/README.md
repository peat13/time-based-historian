# Complete Beginner's Guide to Timebase Historian

This step-by-step guide will take you from a fresh Ubuntu Server installation to a fully running Timebase Historian stack. No prior Docker or Linux experience required!

## Table of Contents

1. [What You'll Need](#what-youll-need)
2. [Installing Docker on Ubuntu Server](#installing-docker-on-ubuntu-server)
3. [Installing Portainer](#installing-portainer)
4. [Deploying Timebase Stack via Portainer](#deploying-timebase-stack-via-portainer)
5. [Accessing Your Services](#accessing-your-services)
6. [Troubleshooting](#troubleshooting)
7. [Next Steps](#next-steps)

---

## What You'll Need

- A server or PC running Ubuntu Server (20.04 or newer recommended)
- SSH access to your server (or direct terminal access)
- Internet connection
- At least 2GB RAM and 20GB free disk space
- Your server's IP address (find it with `ip addr` or `hostname -I`)

**Time Required:** About 20-30 minutes

---

## Installing Docker on Ubuntu Server

Docker is the container platform that will run all your Timebase services. Let's install it step by step.

### Step 1: Connect to Your Server

```bash
# If using SSH from another computer:
ssh username@your-server-ip

# If you're directly on the server, just open the terminal
```

### Step 2: Update Your System

First, make sure your system is up to date:

```bash
sudo apt update
sudo apt upgrade -y
```

**What this does:** Updates the list of available packages and upgrades your system to the latest versions.

### Step 3: Install Prerequisites

```bash
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
```

**What this does:** Installs tools needed to download Docker securely.

### Step 4: Add Docker's Official Repository

```bash
# Add Docker's GPG key (this verifies the download is authentic)
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add the Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

**What this does:** Adds Docker's official software source so you can install Docker.

### Step 5: Install Docker

```bash
# Update package list again to include Docker packages
sudo apt update

# Install Docker
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

**What this does:** Installs Docker Engine, Docker CLI, containerd (container runtime), and Docker Compose.

### Step 6: Verify Docker Installation

```bash
# Check Docker version
docker --version

# Test Docker with hello-world container
sudo docker run hello-world
```

**Expected output:** You should see a message that says "Hello from Docker!" confirming Docker is working.

### Step 7: Add Your User to Docker Group (Optional but Recommended)

This lets you run Docker commands without `sudo`:

```bash
# Add your user to the docker group
sudo usermod -aG docker $USER

# Apply the new group membership (log out and back in, or use this)
newgrp docker

# Test without sudo
docker run hello-world
```

**What this does:** Gives your user account permission to run Docker commands directly.

### Step 8: Enable Docker to Start on Boot

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

**What this does:** Ensures Docker starts automatically when your server boots up.

✅ **Docker is now installed!** Let's move on to Portainer.

---

## Installing Portainer

Portainer is a web-based interface that makes managing Docker containers easy. No more complex command-line operations!

### Step 1: Create a Volume for Portainer Data

```bash
docker volume create portainer_data
```

**What this does:** Creates persistent storage for Portainer's settings and data.

### Step 2: Install Portainer Community Edition

```bash
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

**What this does:**
- `-d` - Runs in the background
- `-p 9443:9443` - Makes Portainer accessible on port 9443 (HTTPS)
- `-p 8000:8000` - Exposes port for edge agents (optional)
- `--restart=always` - Automatically restarts Portainer if it stops
- `-v /var/run/docker.sock` - Gives Portainer access to Docker
- `-v portainer_data:/data` - Stores Portainer data persistently

### Step 3: Verify Portainer is Running

```bash
docker ps
```

**Expected output:** You should see the `portainer` container running.

### Step 4: Access Portainer Web Interface

1. Open your web browser
2. Navigate to: `https://your-server-ip:9443`
   - Replace `your-server-ip` with your actual server IP address
   - Example: `https://192.168.1.100:9443`

3. **Important:** You'll see a security warning because Portainer uses a self-signed certificate
   - This is normal and safe
   - Click "Advanced" → "Proceed to [your-server-ip]" (wording varies by browser)

### Step 5: Create Your Admin Account

On first visit, you'll be prompted to create an admin account:

1. **Username:** `admin` (or your preferred username)
2. **Password:** Choose a strong password (at least 12 characters)
3. Click **"Create user"**

⚠️ **Important:** You have 5 minutes to create this account. If you wait too long, restart Portainer:
```bash
docker restart portainer
```

### Step 6: Connect to Local Docker Environment

1. Select **"Get Started"** or **"Local"**
2. Click **"Connect"**
3. You'll see your local Docker environment dashboard

✅ **Portainer is now ready!** Let's deploy the Timebase stack.

---

## Deploying Timebase Stack via Portainer

Now comes the easy part - deploying the entire Timebase Historian stack with just a few clicks!

### Step 1: Navigate to Stacks

1. In Portainer's left sidebar, click **"Stacks"**
2. Click the **"+ Add stack"** button

### Step 2: Configure Your Stack

Fill in the following information:

**Name:**
```
timebase-historian
```

**Build method:** Select **"Git Repository"**

### Step 3: Git Repository Settings

**Repository URL:**
```
https://github.com/DMDuFresne/time-based-historian
```

**Repository reference:**
```
refs/heads/main
```

**Compose path:**
```
docker-compose.yml
```

**Authentication:** Leave unchecked (this is a public repository)

### Step 4: Environment Variables (Optional)

Click on **"Environment variables"** to expand the section.

**Basic setup (no changes needed):** The stack will work out of the box with default settings!

**To customize** (optional), add environment variables as `name` and `value` pairs:

| Name | Value | Description |
|------|-------|-------------|
| `HISTORIAN_PORT` | `4511` | Change if port 4511 is already in use |
| `EXPLORER_PORT` | `4531` | Change if port 4531 is already in use |
| `COLLECTOR_PORT` | `4521` | Change if port 4521 is already in use |
| `TZ` | `America/New_York` | Set your timezone |

**Example customization:**
- Name: `TZ`
- Value: `America/New_York`

### Step 5: Deploy the Stack

1. Scroll down and click **"Deploy the stack"**
2. Wait for the deployment to complete (usually 30-60 seconds)
3. You'll see a success message when it's done

### Step 6: Verify Deployment

1. Click on **"timebase-historian"** in your stacks list
2. You should see three services:
   - ✅ **timebase-historian** - Running
   - ✅ **timebase-explorer** - Running
   - ✅ **timebase-collector** - Running

If any service shows an error, check the [Troubleshooting](#troubleshooting) section below.

✅ **Your Timebase stack is now running!**

---

## Accessing Your Services

Your Timebase services are now accessible via your web browser.

### Timebase Explorer (Web UI)

**URL:** `http://your-server-ip:4531`

**What it does:** This is the main web interface where you'll visualize and analyze your time-series data.

**First-time setup:**
1. Open the URL in your browser
2. Follow the Timebase Explorer setup wizard
3. Connect to the Historian at: `http://historian:4511`
   - Note: Use `historian` as the hostname (not your server IP)
   - This is the internal Docker network name

### Timebase Historian API

**URL:** `http://your-server-ip:4511`

**What it does:** The REST API for the time-series database. You typically won't access this directly - Explorer and Collector connect to it automatically.

### Timebase Collector API

**URL:** `http://your-server-ip:4521`

**What it does:** The data collection service configuration interface.

---

## Managing Your Stack in Portainer

### Viewing Logs

1. Go to **Stacks** → **timebase-historian**
2. Click on any service name (e.g., `timebase-historian`)
3. Click **"Logs"** to see real-time logs
4. Use the **"Auto-refresh"** toggle to watch logs live

### Stopping the Stack

1. Go to **Stacks** → **timebase-historian**
2. Click **"Stop this stack"**
3. All services will shut down gracefully

### Starting the Stack

1. Go to **Stacks** → **timebase-historian**
2. Click **"Start this stack"**
3. All services will start back up

### Updating the Stack

To pull the latest version from GitHub:

1. Go to **Stacks** → **timebase-historian**
2. Click **"Editor"** at the top
3. Click **"Pull and redeploy"**
4. Wait for the update to complete

### Removing the Stack (Deleting Everything)

⚠️ **Warning:** This will delete all your data!

1. Go to **Stacks** → **timebase-historian**
2. Click **"Delete this stack"**
3. Check **"Remove associated volumes"** if you want to delete data too
4. Confirm deletion

---

## Troubleshooting

### Problem: Cannot Access Portainer (Connection Refused)

**Solution:**
```bash
# Check if Portainer is running
docker ps | grep portainer

# If not running, start it
docker start portainer

# Check the logs
docker logs portainer
```

### Problem: Port Already in Use

**Error message:** "Bind for 0.0.0.0:4511 failed: port is already allocated"

**Solution:** Change the port in Portainer:
1. Go to your stack
2. Click **"Editor"**
3. Add environment variable: `HISTORIAN_PORT=5511` (or any free port)
4. Click **"Update the stack"**

**Check what's using a port:**
```bash
sudo lsof -i :4511
```

### Problem: Service Won't Start

**Solution:**
1. Click on the failed service in Portainer
2. Click **"Logs"** to see error messages
3. Look for error messages at the end of the logs

**Common causes:**
- Not enough disk space (`df -h` to check)
- Not enough memory (`free -h` to check)
- Port conflicts (see above)

### Problem: Can't Access Services from Another Computer

**Solution:** Check your firewall:

```bash
# Ubuntu firewall (UFW)
sudo ufw allow 4511/tcp
sudo ufw allow 4521/tcp
sudo ufw allow 4531/tcp
sudo ufw allow 9443/tcp  # For Portainer

# Check firewall status
sudo ufw status
```

### Problem: Docker Permission Denied

**Error message:** "permission denied while trying to connect to the Docker daemon socket"

**Solution:**
```bash
# Add your user to docker group
sudo usermod -aG docker $USER

# Log out and back in, or use:
newgrp docker
```

### Problem: Stack Deployed but Services Not Running

**Solution:**
1. Check Docker daemon status:
```bash
sudo systemctl status docker
```

2. Restart Docker if needed:
```bash
sudo systemctl restart docker
```

3. Check service logs in Portainer for specific errors

### Getting Help

If you're still stuck:

1. **Check the logs** in Portainer - they usually tell you what's wrong
2. **Timebase Documentation:** https://timebase.flow-software.com/en/knowledge-base
3. **GitHub Issues:** https://github.com/DMDuFresne/time-based-historian/issues

---

## Next Steps

Congratulations! You now have a fully functional Timebase Historian stack running. Here's what to do next:

### 1. Add More Collectors (Optional)

To collect data from multiple sources:

1. In Portainer, go to **Stacks** → **timebase-historian** → **Editor**
2. Find the commented `# timebase-collector-02` section (look for "EXAMPLE: Adding a second collector")
3. Uncomment the entire `timebase-collector-02` service
4. Scroll down to the **Volumes** section
5. Uncomment the `collector-02-data` volume
6. Click **"Update the stack"**
7. Your second collector will be available at `http://your-server-ip:4522`

### 2. Configure Data Collection

1. Open Collector at `http://your-server-ip:4521`
2. Follow the Collector setup wizard to configure your data sources:
   - OPC UA servers
   - Modbus devices
   - MQTT brokers
   - REST APIs
   - And more!

### 3. Secure Your Installation

For production use:

**Enable HTTPS:**
- Use a reverse proxy like Nginx or Traefik
- Get free SSL certificates from Let's Encrypt

**Change default ports:**
- Especially if your server is internet-facing
- Use environment variables to customize

**Backup your data:**
```bash
# List volumes
docker volume ls

# Backup a volume
docker run --rm -v timebase-historian-data:/data -v $(pwd):/backup ubuntu tar czf /backup/historian-backup.tar.gz /data
```

### 4. Learn More

- **Timebase Documentation:** https://timebase.flow-software.com/en/knowledge-base
- **Docker Documentation:** https://docs.docker.com/
- **Portainer Documentation:** https://docs.portainer.io/

---

## Summary

You've successfully:

- ✅ Installed Docker on Ubuntu Server
- ✅ Installed Portainer for easy management
- ✅ Deployed the complete Timebase Historian stack
- ✅ Learned how to manage services via Portainer
- ✅ Configured your system for data collection and visualization

**Your running services:**
- Timebase Explorer: `http://your-server-ip:4531`
- Timebase Historian API: `http://your-server-ip:4511`
- Timebase Collector: `http://your-server-ip:4521`
- Portainer: `https://your-server-ip:9443`

**Need help?** Check the Troubleshooting section above or visit the Timebase documentation.

Happy data collecting! 📊
