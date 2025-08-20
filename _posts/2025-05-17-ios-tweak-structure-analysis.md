---
layout: post
title: "Dissecting an iOS Tweak: Structure and Installation Deep Dive"
date: 2025-04-17
categories: [ios, jailbreak, reverse-engineering]
tags: [ios, jailbreak, tweaks, cydia, substrate, mobile-security, reverse-engineering]
---

For pentesters, security researchers, and hobbyists using jailbroken iOS devices, understanding the structure and lifecycle of a jailbreak tweak can help them better understand how the system works under the hood. This post walks through a real-world example and explains how a tweak is distributed, installed, and functions on a jailbroken device.

<!--more-->

## Installation and Distribution

When an iOS device is jailbroken, tools like Cydia or Sileo are usually installed. These serve as package managers that allow users to install jailbreak tweaks and libraries. They rely on Cydia Substrate (aka MobileSubstrate) to inject tweaks into running apps and processes.

Jailbreak tweaks are distributed as Debian `.deb` packages. These packages are hosted on repositories (e.g., https://havoc.app), and Cydia/Sileo retrieves them from there. You can add repositories manually or use ones preloaded in your package manager.

To manually explore available packages from the command line for a repository:

```bash
curl https://havoc.app/Packages
# or
curl -L https://havoc.app/Packages.gz | gzip -d
```

The Packages file is a plain-text file where each package entry is separated by a blank line. Details: [Debian Control Fields](https://www.debian.org/doc/debian-policy/ch-controlfields.html).

### Example Entry In The Packages file

```
Package: com.icraze.hestia
Name: Hestia
Version: 1.6.1
Section: Tweaks
Architecture: iphoneos-arm
Depends: mobilesubstrate (>= 0.9.5000), sed, firmware (>= 11.0), com.spark.libsparkapplist, preferenceloader
Filename: api/download/package/61f8adaee40b9eda48112f7f/com.icraze.hestia_1.6.1.deb
```

You can manually download the `.deb` using the Filename path:

```
https://havoc.app/api/download/package/61f8adaee40b9eda48112f7f/com.icraze.hestia_1.6.1.deb
```

## Inspecting the Package

To inspect the downloaded package:

```bash
dpkg -I com.icraze.hestia_1.6.1.deb
```

This reveals metadata, including the dependencies and post-installation script `postinst`, which runs after installation.

To extract contents:

```bash
ar x com.icraze.hestia_1.6.1.deb
# Produces: control.tar.gz, data.tar.lzma, debian-binary
```

### control.tar.gz - Metadata and Scripts

This archive contains the package's control metadata and maintainer scripts. Key files include `control` (package information like name, version, dependencies) and scripts such as `postinst`, which are executed during install or removal to perform custom actions like file renaming or cleanup.

Unpacking this archive gives:

```
control/
├── control      # Metadata (same as `dpkg -I`)
└── postinst     # Post-installation script
```

`postinst` contents:

```bash
#!/bin/bash
RND_STR_ONE=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 4 | head -n 1)
RND_STR_TWO=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 4 | head -n 1)
RND_STR="_${RND_STR_ONE}HEST${RND_STR_TWO}_"

mv "/Library/MobileSubstrate/DynamicLibraries/Hestia.dylib" "/Library/MobileSubstrate/DynamicLibraries/${RND_STR}.dylib"
mv "/Library/MobileSubstrate/DynamicLibraries/Hestia.plist" "/Library/MobileSubstrate/DynamicLibraries/${RND_STR}.plist"

sed -i "s/DynamicLibraries\/Hestia/DynamicLibraries\/${RND_STR}/g" "/Library/dpkg/info/com.icraze.hestia.list"
```

In this case the script obfuscates the tweak's filenames post-installation useful for bypassing jailbreak detection.

### data.tar.lzma - Actual Tweak Files

This archive contains the actual payload of the tweak. When unpacked, it reveals the file structure that will be copied to the root filesystem during installation; typically including the `.dylib` and `.plist` files for injection, preference bundles for Settings integration, and other resources. Analyzing this structure shows exactly what the tweak modifies or extends on the device.

```
Library/
├── MobileSubstrate
│   └── DynamicLibraries
│       ├── Hestia.dylib
│       └── Hestia.plist
├── PreferenceBundles
│   └── HestiaPrefs.bundle
│       ├── Root.plist
│       ├── Info.plist
│       └── HestiaPrefs # (executable)
└── PreferenceLoader
    └── Preferences
        └── HestiaPrefs.plist
```

Files are copied to these locations during installation relative to the root `/`. A respring is required to load the changes.

## MobileSubstrate: Code Injection

MobileSubstrate (also known as Cydia Substrate) is the primary framework that enables runtime code injection on jailbroken iOS devices. It works by injecting custom dynamic libraries (`.dylib`) into running processes at launch time.

This injection is configured through `.plist` filter files located in `/Library/MobileSubstrate/DynamicLibraries`, which define which processes should load the tweak based on bundle identifiers or executable names.

When a target process starts, the jailbroken system's kernel patches ensure that any matching `.dylib` files are loaded into the process's address space before its main execution begins. This allows the injected code to override system behavior or extend functionality.

### Key Components

- **.dylib** - The tweak logic, compiled for ARM targets.
- **.plist** - Specifies which processes/bundles to inject into.

Example `.plist`:

```xml
Filter = {
    Bundles = ("com.apple.springboard");
};
```

## PreferenceLoader: Settings Integration

PreferenceLoader enables tweaks to add configuration interfaces directly into the iOS Settings app. It reads `.plist` registration files that link to a preference bundle, which defines the UI and handles storing user settings using CFPreferences.

Sample `Root.plist` entry:

```xml
<dict>
    <key>items</key>
    <array>
        <dict>
            <key>cell</key><string>PSSwitchCell</string>
            <key>defaults</key><string>com.hestia.prefs</string>
            <key>key</key><string>enabled</string>
        </dict>
    </array>
</dict>
```

## Developing Tweaks

Developing an iOS jailbreak tweak typically begins with setting up [Theos](https://theos.dev/), a cross-platform build system for creating software on jailbroken devices. It supports macOS, Linux, and Windows (via WSL) and helps the process of compiling, packaging, and deploying tweaks.

Tweaks are generally written in Objective-C or Swift, but most developers use Logos, a Theos preprocessor that simplifies hooking into Objective-C methods and C functions with minimal boilerplate.

A typical workflow involves generating a project template using Theos, implementing the desired logic using Logos syntax, and building the project to produce a `.deb` package. This package can then be installed on a jailbroken device for testing or distribution.

For a complete guide, refer to the official [Theos documentation](https://theos.dev/docs/) and the [MTACS TweakGuide](https://github.com/AeonLucid/MTAC).

## Security Implications

Understanding tweak structure reveals several security considerations:

- **Code Injection**: Tweaks can modify any app's behavior, potentially bypassing security measures
- **Privilege Escalation**: Many tweaks run with elevated privileges
- **Data Access**: Tweaks can access sensitive data from injected processes
- **Persistence**: Tweaks survive app updates and system reboots
- **Detection Evasion**: Advanced tweaks employ obfuscation to avoid detection

## Conclusion

iOS jailbreak tweaks represent a fascinating intersection of reverse engineering, system programming, and mobile security. By understanding their structure and installation process, security researchers can better analyze potential threats, while developers can create more sophisticated modifications to iOS behavior.

The ecosystem continues to evolve with each iOS version, creating an ongoing cat-and-mouse game between Apple's security measures and the jailbreaking community's ingenuity.