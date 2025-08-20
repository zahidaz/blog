---
layout: post
title: "Manually Installing and Configuring Frida on Jailbroken iOS Devices"
date: 2023-11-29
categories: [ios, jailbreak, security]
tags: [frida, ios, jailbreak, instrumentation, palera1n, cybersecurity]
---

For individuals unfamiliar with Frida, it stands out as a robust dynamic instrumentation toolkit that empowers developers by enabling the injection of scripts (JavaScripts) into the processes of applications. This capability facilitates real-time manipulation and analysis of processes, making it a valuable tool frequently employed in application instrumentation.

<!--more-->

Recently, I found myself in the position of needing to install Frida on my newly jailbroken iOS device, utilising the palera1n jailbreak. Following the official Frida installation guide for jailbroken devices, I encountered a challenge. The guide recommended adding https://build.frida.re to the Cydia sources. However, my jailbroken iOS, courtesy of palera1n, featured Zebra and Sileo instead of Cydia.

In a scenario where I'm accustomed to Android's straightforward process - downloading the frida-server binary, pushing it with adb, and executing it through the shell - But package management abstracted away the details of how the installation really work, So I decided to install it manually, detailing the processes here.

## Getting the Frida Binary

Noting that one of the goals I aimed to achieve was to make Frida accessible through the network. First, we need to locate the downloadable artifact on the Frida release page on GitHub. The desired artifact, also utilised by the package manager, is at the very bottom of the release artefacts with an interesting name suffix of `iphoneos` although I was specifically looking for something named iOS.

There are two files with the last part of the name as `…iphoneos-arm.deb` and `…iphoneos-arm64.deb`. Download the one that matches the architecture of your device. One way to verify this is by downloading AIDA64 and checking the information for yourself.

If you download and extract the file, you will observe the following directory structure:

```
─── var
    └── jb
        ├── Library
        │   └── LaunchDaemons
        │       └── re.frida.server.plist
        └── usr
            ├── lib
            │   └── frida
            │       └── frida-agent.dylib
            └── sbin
                └── frida-server
```

## Installation Process

The `/var` directory will be directly under the root, and the remaining files will be copied to their respective directories. One approach is to copy each file individually and then use launchctl to load the `re.frida.server.plist` file, but this method is not particularly clean. Instead, a more streamlined approach involves using scp utility to copy the `.deb` file to the `/private/var/mobile` directory, followed by utilizing ssh to access the device and employ `dpkg -i` to install the `.deb` file directly from the specified directory.

```bash
root@iphone # cd /private/var/mobile
root@iphone # dpkg -i frida_16.1.8_iphoneos-arm64.deb
```

With this method, all files will be seamlessly copied to their respective directories. Afterward, you can easily check if Frida is running:

```bash
iphone:/var/jb/Library/LaunchDaemons root# ps aux | grep frida
root  1670   ..... /var/jb/usr/sbin/frida-server
```

## Configuring Network Access

Alright, Frida is installed, and I can access it when connected via USB. However, the next step is making it accessible through the network. Given that it is initiated with the KeepAlive key, even if we terminate it, it will restart automatically. It's worth noting that initiating another instance will result in two copies of the same program running concurrently, which is not an ideal situation in my view.

Remember the `re.frida.server.plist` file in the extracted package? We now need to locate it in `/var/jb/Library/LaunchDaemons`. Here is the content of the file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>re.frida.server</string>
    <key>Program</key>
    <string>/var/jb/usr/sbin/frida-server</string>

    <key>ProgramArguments</key>
    <array>
        <string>/var/jb/usr/sbin/frida-server</string>
    </array>

    <key>UserName</key>
    <string>root</string>
    <key>POSIXSpawnType</key>
    <string>Interactive</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>ThrottleInterval</key>
    <integer>5</integer>
    <key>ExecuteAllowed</key>
    <true/>
</dict>
</plist>
```

## Understanding Property List Files

A `.plist` file in iOS, or Property List file, is a structured XML or binary file used to store configuration settings and other data. It serves as a lightweight and human-readable format for storing preferences, metadata, and other relevant information for applications and the operating system. Developers commonly use `.plist` files to manage configuration details and ensure efficient data storage and retrieval within iOS applications.

## Modifying the Configuration

Well, we need to modify this section:

```xml
<key>ProgramArguments</key>
<array>
    <string>/var/jb/usr/sbin/frida-server</string>
</array>
```

Here are step-by-step commands to accomplish that:

```bash
root@iphone # pwd
/private/var/mobile

# You may need to install a text editor
root@iphone# apt-get install nano

root@iphone # cd /var/jb/Library/LaunchDaemons
root@iphone # ls
re.frida.server.plist
# ... more .plist files

root@iphone # nano re.frida.server.plist
```

To make it accessible on the network, we need to modify the specified section in the `re.frida.server.plist` file as follows:

Add:
```xml
<string>-l</string>
<string>0.0.0.0</string>
```

After:
```xml
<string>/var/jb/usr/sbin/frida-server</string>
```

Following the editing of the `.plist` file, we need to load the modified file:

```bash
# while still in /var/jb/Library/LaunchDaemons
root@iphone # launchctl unload re.frida.server.plist
root@iphone # launchctl load re.frida.server.plist
```

By using grep for frida-server in ps aux, we can verify that the changes have taken effect:

```bash
root@iphone# ps aux | grep frida
... /var/jb/usr/sbin/frida-server -l 0.0.0.0
```

Now, from the host/PC, you can issue commands to the device on the same network/Wi-Fi:

```bash
# frida-ps -H <iphone-ip>
```

## Conclusion

This manual installation process provides a deeper understanding of how Frida operates on iOS devices and offers more control over the configuration process. While package managers abstract away these details for convenience, knowing the underlying structure helps troubleshoot issues and customize installations for specific needs.