# My SOC Lab: A Hands-On Cybersecurity Journey

## 1. Project Overview

### 1.1. My Motivation

I've always been fascinated by the world of cybersecurity, especially the defensive side of things. Reading about threat actors, attack vectors, and defensive strategies is one thing, but I wanted to *really* get my hands dirty. To move beyond theory and into practice, I decided to build my own Security Operations Center (SOC) lab from the ground up. This project was my way of creating a personal digital playground where I could dissect attacks, master security tools, and truly understand the flow of a security incident from detection to resolution. This document is the log of my journey.

### 1.2. Design Philosophy

From the start, I knew I wanted a modular and easily reproducible setup. That's why I chose a **containerized, microservices-based architecture** using *Docker* and *Docker Compose*. This approach has some killer advantages:

*   ***Isolation:*** Every tool runs in its own self-contained environment, which keeps things clean and prevents a mess of conflicting dependencies.
*   ***Scalability:*** It's a breeze to add new tools to the mix or swap components out as I discover new things.
*   ***Reproducibility:*** The entire lab is defined in a single `docker-compose.yml` file. I can tear it down and spin it back up in minutes—a huge plus for experimentation.
*   ***Efficiency:*** Containers are much lighter on resources than full-blown virtual machines.

## 2. The Foundation: My Virtual Environment

### 2.1. The Host Machine

I built the lab on a dedicated virtual machine, which keeps it nicely sandboxed from my main machine. I went with **VirtualBox** because I'm familiar with it and it's incredibly easy to use.

> **VM Specs:**
> *   **CPUs:** `8` cores
> *   **RAM:** `16 GB`
> *   **Storage:** `150 GB` (Dynamically Allocated)
> *   **Networking:** `Bridged Adapter` (This gives the VM its own IP on my network, making it feel like a real, separate server.)

### 2.2. The Operating System

I chose **Ubuntu 22.04 LTS** as the base OS—it's stable, reliable, and has a massive community for support. After the initial install, I got right into the terminal to set things up.

First, a quick update and installation of some essential tools:
```bash
# Always good practice to get the latest packages
sudo apt update && sudo apt upgrade -y

# Installing some daily drivers
sudo apt install -y curl git vim
```

### 2.3. Docker and System Prep

With the OS ready, it was time to install the containerization engine.

I used the official convenience script to get Docker:
```bash
# Install Docker Engine
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add my user to the docker group so I don't have to type 'sudo' all the time
sudo usermod -aG docker $USER
newgrp docker # Applies the new group membership immediately

# Install Docker Compose plugin
sudo apt-get install -y docker-compose-plugin
```

To keep the system from crashing if my memory-hungry tools get out of hand, I configured an 8GB swap file. This acts as a safety net.
```bash
# Create and activate a swap file
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make the swap permanent so it survives reboots
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 2.4. Lab Networking

I created a dedicated Docker network to allow all the lab components to communicate with each other seamlessly.
```bash
docker network create soc-net
```

## 3. Assembling the Arsenal: Lab Deployment

### 3.1. Directory Setup

To keep things organized and ensure my data persists if a container is restarted, I created a directory structure for all the tools first.

```bash
# Create a home for all my persistent container data
mkdir -p wazuh/{manager,indexer,dashboard}
mkdir -p suricata/{logs,rules}
mkdir -p evebox
mkdir -p velociraptor/config
mkdir -p arkime/data
mkdir -p misp/db
mkdir -p shuffle/config
mkdir -p thehive/data
mkdir -p cortex/data
mkdir -p cyberchef
mkdir -p portainer/data
mkdir -p fleetdm/mysql
mkdir -p caldera/conf
```

### 3.2. The Master Blueprint: Docker Compose

This is where it all comes together. The `docker-compose.yml` file is the heart of the lab, defining every single service.

```yaml
version: '3.8'

networks:
  soc-net:
    external: true

services:
  # --- SIEM & MONITORING ---
  wazuh-manager:
    image: wazuh/wazuh-manager:4.3.10
    container_name: wazuh-manager
    hostname: wazuh-manager
    restart: always
    ports:
      - "1514:1514/udp"
      - "1515:1515/tcp"
      - "55000:55000/tcp"
    volumes:
      - ./wazuh/manager:/var/ossec/data
    networks:
      - soc-net

  wazuh-indexer:
    image: wazuh/wazuh-indexer:4.3.10
    container_name: wazuh-indexer
    hostname: wazuh-indexer
    restart: always
    ports:
      - "9200:9200"
    volumes:
      - ./wazuh/indexer:/usr/share/wazuh-indexer/data
    environment:
      - "INDEXER_USERNAME=admin"
      - "INDEXER_PASSWORD=SecretPassword"
    networks:
      - soc-net

  wazuh-dashboard:
    image: wazuh/wazuh-dashboard:4.3.10
    container_name: wazuh-dashboard
    hostname: wazuh-dashboard
    restart: always
    ports:
      - "7001:5601"
    environment:
      - "DASHBOARD_USERNAME=admin"
      - "DASHBOARD_PASSWORD=SecretPassword"
      - "INDEXER_HOST=wazuh-indexer"
    networks:
      - soc-net

  suricata:
    image: jasonish/suricata:latest
    container_name: suricata
    restart: always
    network_mode: "host"
    cap_add:
      - NET_ADMIN
      - SYS_NICE
    volumes:
      - ./suricata/logs:/var/log/suricata
      - ./suricata/rules:/etc/suricata/rules
    command: -i eth0 # IMPORTANT: Change to your VM's network interface

  evebox:
    image: jasonish/evebox:latest
    container_name: evebox
    restart: always
    ports:
      - "7015:5636"
    volumes:
      - ./suricata/logs:/var/log/suricata
    command: -e http://wazuh-indexer:9200 # Pointing to my Wazuh indexer
    networks:
      - soc-net

  # --- DFIR & FORENSICS ---
  velociraptor:
    image: wazuh/velociraptor:0.6.6
    container_name: velociraptor
    restart: always
    ports:
      - "7000:8000"
    volumes:
      - ./velociraptor/config:/etc/velociraptor
    networks:
      - soc-net

  arkime:
    image: aol/moloch:latest
    container_name: arkime
    restart: always
    ports:
      - "7008:8005"
    volumes:
      - ./arkime/data:/data/moloch
      - /pcap:/data/pcap # A dedicated directory for live packet captures
    cap_add:
      - NET_RAW
      - NET_ADMIN
    command: /data/moloch/bin/moloch_config_interfaces.sh

  # --- THREAT INTELLIGENCE ---
  misp-server:
    image: coolacid/misp-docker:latest
    container_name: misp-server
    restart: always
    ports:
      - "7003:443"
    volumes:
      - ./misp/db:/var/lib/mysql
    environment:
      - "MYSQL_USER=misp"
      - "MYSQL_PASSWORD=misp-password"
      - "MYSQL_DATABASE=misp"
    networks:
      - soc-net

  # --- SOAR & AUTOMATION ---
  thehive:
    image: thehiveproject/thehive4:latest
    container_name: thehive
    restart: always
    ports:
      - "7005:9000"
    volumes:
      - ./thehive/data:/opt/thp/thehive/data
    networks:
      - soc-net

  cortex:
    image: thehiveproject/cortex:latest
    container_name: cortex
    restart: always
    ports:
      - "7006:9001"
    volumes:
      - ./cortex/data:/opt/thp/cortex/data
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - soc-net
  
  shuffle:
    image: shuffler/shuffler:latest
    container_name: shuffle
    restart: always
    ports:
      - "7002:3001"
    volumes:
      - ./shuffle/config:/usr/src/app/shuffle-config
    networks:
      - soc-net

  # --- UTILITIES & MANAGEMENT ---
  cyberchef:
    image: mpepping/cyberchef:latest
    container_name: cyberchef
    restart: always
    ports:
      - "7004:8000"

  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    ports:
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./portainer/data:/data
    networks:
      - soc-net
      
  caldera:
    image: mitre/caldera:latest
    container_name: caldera
    restart: always
    ports:
      - "7009:8888"
    volumes:
      - ./caldera/conf:/caldera/conf
    networks:
      - soc-net
```

### 3.3. Bringing the Lab to Life

With the blueprint complete, all it took was one command to spin everything up in detached mode.

```bash
docker compose up -d
```

## 4. Putting the Lab to the Test: Practical Scenarios

A lab is only as good as the skills you build with it. Here are a few of the exercises I've run to simulate real-world SOC tasks.

### Scenario 1: The Phishing Investigation

*   **Objective:** Triage a suspicious email, hunt for indicators, and pivot to endpoint forensics.
*   **My Process:**
    1.  **The Bait:** I started with a sample phishing email (`.eml` file). My first step was to open it in a sandboxed environment to analyze its headers and body. I paid close attention to the `Received` headers to trace its path, the `Return-Path` for the true sender, and any URLs or attachments.
    2.  **Extracting IOCs:** I quickly identified a few juicy indicators: the sender's IP, a sketchy URL in the email body (hidden with a convincing hyperlink), and the hash of a zipped attachment.
    3.  **Intel Check:** I pivoted over to my **MISP** instance. A quick search for the URL showed a hit—it was part of a known campaign! This context is gold; it tells me I'm not dealing with a one-off attack.
    4.  **The Hunt:** Now I had something to hunt for. I jumped into the **Wazuh** dashboard and built a query for the malicious IP and domain. *Bingo*. One of my lab machines had attempted to connect to the C2 server.
    5.  **Going Deep with DFIR:** Knowing which machine was compromised, I turned to **Velociraptor**. I kicked off a hunt for the malware's file hash, which confirmed the endpoint was infected. From there, I used Velociraptor's incredible power to pull browser history, running processes, and registry keys to see what else the malware had done.
    6.  **Case Management:** The whole time, I was building a case in **TheHive**, adding my findings and observables as I went. This created a clean, chronological record of the entire investigation, perfect for a post-mortem.

### Scenario 2: Malware Mayhem

*   **Objective:** Detect and analyze a malware sample, then use SOAR to automate the initial response.
*   **My Process:**
    1.  **The Detection:** My scheduled **YARA** scan went off, flagging a file on a Windows machine. The rule it matched was for a known ransomware family—not good.
    2.  **Quick Triage:** I grabbed the file and threw it into **CyberChef**. A quick entropy check confirmed it was likely packed. I ran the `strings` recipe and found some interesting API calls and a long, suspicious base64 string.
    3.  **Unraveling the Secret:** I dropped the base64 string into a new CyberChef recipe, and out popped a C2 server address. Now I had another critical IOC.
    4.  **SOAR to the Rescue:** I created a case in **TheHive** and added the file hash and C2 address. This is where the magic happens. I had already built a **Shuffle** workflow for this exact situation. I triggered it from TheHive, and it automatically:
        *   Checked the C2 address against **MISP**.
        *   Blocked the C2 address on my lab's virtual firewall.
        *   Told **Velociraptor** to grab a memory dump of the affected machine for deeper analysis.
    5.  **Cleanup:** With the immediate threat neutralized, I could then focus on safely removing the malware and restoring the machine.

### Scenario 3: Playing the Bad Guy with Caldera

*   **Objective:** Use **Caldera** to simulate an adversary and see if my defenses would catch it.
*   **My Process:**
    1.  **The Attack Plan:** I decided to simulate a common lateral movement technique: `T1059.003 - Command and Scripting Interpreter: Windows Command Shell`. I configured a simple Caldera operation to run `whoami` on a target machine.
    2.  **Execution:** I deployed a Caldera agent on one of my lab's Windows VMs and launched the operation.
    3.  **The Catch:** I immediately saw alerts firing. The **Wazuh** agent on the VM caught the `cmd.exe` process execution and forwarded it to the SIEM. My custom rule for suspicious command-line activity lit up like a Christmas tree.
    4.  **Network View:** I also checked my **EveBox** dashboard. The **Suricata** logs showed alerts for the Caldera agent's C2 communication, which I had written a rule to detect.
    5.  **Validation:** This was a huge success. It proved that my detection rules were working as expected and gave me valuable insight into the forensic trail left by this specific attack. It's a great way to proactively test and improve my defenses.

## 5. Final Thoughts

This project wasn't just about learning new things from scratch; it was about solidifying and expanding the strong foundation I'd already built. Looking back, building this SOC lab has been one of the most rewarding experiences in my cybersecurity journey. It's transformed my understanding from theoretical knowledge gained in previous projects like 'SOC-Lab' and 'SOC-Automation-Project' into practical, hands-on expertise. The process of setting up each tool, integrating them, and then actively using them to hunt and respond to threats has been invaluable. This document is more than just a setup guide; it's a testament to the power of continuous learning and refinement in the ever-evolving world of cybersecurity. I'm excited to keep expanding the lab, trying new tools, and simulating even more complex attacks, always pushing the boundaries of my understanding. The journey is far from over.