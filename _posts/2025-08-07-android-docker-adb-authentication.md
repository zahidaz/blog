---
layout: post
title: "Debug Android from Docker: ADB Authentication Solutions"
date: 2025-08-07
categories: [android, docker, development]
tags: [adb, docker, android, debugging, containers, usb]
---

Docker containers can't trigger the USB authorization dialog that ADB normally uses. Solution: Generate shared ADB keys, copy the private key to your Docker container, automatically install the public key on your Android device, then connect via TCP/IP.

<!--more-->

## The Problem: Docker Breaks ADB's Normal Flow

When you connect an Android device to your PC via USB, ADB automatically handles authentication. Your device shows a dialog asking you to authorize the connection, and once confirmed, cryptographic keys are exchanged and stored for future use.

This seamless process completely breaks down in Docker containers. Containers run in isolated environments that cannot trigger the USB authorization dialog or easily access the host's ADB keys. For developers building Android apps in containerized environments, this creates a frustrating barrier.

## Understanding ADB's Security Model

ADB uses public-private key cryptography for device authentication. When you first connect a device:

- The private key stays on your host machine at `~/.android/adbkey`
- The public key gets copied to your Android device at `/data/misc/adb/adbkeys`
- Future connections use this key pair to authenticate without user intervention. On some devices, the key may expire and should be renewed periodically.

The challenge is getting your Docker container to use shared keys that your Android device trusts.

## The Solution: Generate and Share Keys

### Step 1: Create a Shared Key Pair

Generate a new shared key pair that both your host and containers can use:

```bash
# Generate new ADB keys
adb keygen shared_adb_key
adb pubgen shared_adb_key shared_adb_key.pub
```

This creates `shared_adb_key` (private) and `shared_adb_key.pub` (public) that you can distribute across your development environment.

### Step 2: Install Keys on Your Host System

Replace your host's ADB key with the shared version:

```bash
# Backup existing keys (Optional)
mv ~/.android/adbkey ~/.android/adbkey.backup
mv ~/.android/adbkey.pub ~/.android/adbkey.pub.backup

# Install shared keys on host (Optional)
cp shared_adb_key ~/.android/adbkey
cp shared_adb_key.pub ~/.android/adbkey.pub
```

### Step 3: Copy Private Key to Docker Containers

Mount or copy the shared private key into your containers:

```bash
# Option 1: Mount as volume
docker run -v $(pwd)/shared_adb_key:/root/.android/adbkey your-image

# Option 2: Copy in Dockerfile
COPY shared_adb_key /root/.android/adbkey
```

### Step 4: Automatically Install Public Key on Android Device

Use an automated script to install the public key on your Android device:

```bash
#!/bin/bash
serial="$1" # the first argument to the script

# Temporarily disable SELinux enforcement
adb -s $serial shell su -c "setenforce 0"

#------ Install the shared public key ------
adb -s $serial push "shared_adb_key.pub" "/data/local/tmp/adbkey_shared.pub"
adb -s $serial shell su -c "rm /data/misc/adb/adb_keys"
adb -s $serial shell su -c "mv /data/local/tmp/adbkey_shared.pub /data/misc/adb/adb_keys"

# Restore SELinux 
adb -s $serial shell su -c "setenforce 1"

# reboot if needed
# adb -s $serial reboot

adb kill-server && adb start-server
```

Run with your device serial number:

```bash
./install_adb_key.sh ABC123456789
```

### Step 5: Switch to Network Connection

Enable TCP/IP debugging on your device and connect over the network:

```bash
# Enable TCP/IP mode (device must be USB-connected initially to host)
adb tcpip 5555

# Connect over network (device can now be disconnected from USB)
adb connect 192.168.1.100:5555
```

### Step 6: Execute Commands from Docker

Use the network connection to run ADB commands from your container:

```bash
# Target specific device by IP
adb -s 192.168.1.100:5555 shell

# Install apps
adb -s 192.168.1.100:5555 install app.apk

# View logs
adb -s 192.168.1.100:5555 logcat
```

## Troubleshooting Common Issues

- **Device not reachable from container**: Ensure your Docker network configuration allows access to the device's IP address. You may need to use `--network host` or configure custom bridge networks.
- **Permission denied errors**: The automated key installation requires root access on your Android device and proper SELinux policy handling.
- **Connection drops**: Network-based ADB connections are less stable than USB. Implement retry logic in your automation scripts.

## Prerequisites

Before implementing this solution:

- ADB installed on your host machine
- Android device with USB debugging enabled
- Root access on Android device (for automated key installation)
- Network connectivity between container and device
- Ability to generate and distribute shared keys across your team

## Alternative: Copy Existing Host Keys

If you prefer the traditional approach of copying existing host keys, you can still do that:

```bash
# Copy existing host keys to container
docker run -v ~/.android:/root/.android your-image
```

However, the shared key generation approach above is cleaner for team environments and CI/CD pipelines.

By generating shared keys and automating their installation, you can maintain full debugging capabilities in containerized development environments while ensuring consistent authentication across your entire team.