# Docker Setup Guide

This setup provides a clean, reproducible environment for deploying Docker images running on Tactical 1000 switches with full Linux capabilities.

## Prerequisites

- Build Ubuntu distribution using debootstrap from [Debootstrap](https://github.com/novarq/debootstrap)
- Configure network connectivity following the [Initial Network Configuration Guide](https://github.com/novarq/tactical-1000/blob/main/docs/initial-network-configuration-guide.md)
- Login to your Ubuntu system on the Novarq Tactical 1000 switch

## Installation Steps

### 1. Add Docker's Official GPG Key

```bash
apt-get update
apt-get install ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
```

### 2. Add Docker Repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
```

### 3. Install Docker

```bash
apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 4. Configure iptables

Switch to iptables legacy mode for compatibility:

```bash
update-alternatives --set iptables /usr/sbin/iptables-legacy
update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
systemctl restart docker
```

### 5. Verify Installation

Check Docker service status:

```bash
systemctl status docker
```

Expected output:
```
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Thu 2025-07-17 09:01:44 UTC; 1min 56s ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 527 (dockerd)
      Tasks: 10
     Memory: 30.7M (peak: 31.3M)
        CPU: 2.175s
```

### 6. Test Docker Installation

Run the hello-world container:

```bash
docker run hello-world
```

Expected output:
```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
c9c5fd25a1bd: Pull complete
Digest: sha256:ec153840d1e635ac434fab5e377081f17e0e15afab27beb3f726c3265039cfff
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

## Documentation

For detailed documentation and advanced configuration options, visit the [official Docker documentation](https://docs.docker.com/engine/install/ubuntu/).
