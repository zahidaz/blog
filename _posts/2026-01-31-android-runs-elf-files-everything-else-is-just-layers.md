---
layout: post
title: "Android Runs ELF Files: Everything Else Is Just Layers"
date: 2026-01-31
categories: [android, architecture, runtime]
tags: [android, elf, art, runtime, native code, ndk, dex, bytecode, zygote, jni, react native, flutter, python, javascript, android internals, app execution, dalvik, compiler]
---

If you've ever wondered how apps written in Python, JavaScript, or C++ can run on Android when everyone says "Android is for Java and Kotlin," you're asking the right question. The answer isn't complicated, but it does require understanding what Android actually does at its core.

<!--more-->

## Android Is Just Another Operating System

Strip away the framework, the APIs, and the developer tools, and Android is a Linux-based operating system. Like any OS, its fundamental job is to execute CPU instructions. That's it. The CPU doesn't care whether your code started as Java, Kotlin, C++, Python, or JavaScript. It only executes machine code.

On Android, this machine code comes in a specific format: ELF (Executable and Linkable Format). This is the standard executable format for Linux systems. When your device runs anything - whether it's a system service, a native library, or the Android Runtime itself - it's executing ELF files.

## The Two Ways Android Lets You Write Apps

Android gives you exactly two pathways to run your code:

### Path 1: DEX Bytecode Through ART

You write code in Java or Kotlin. The build system compiles this into DEX (Dalvik Executable) bytecode. When you install the app, the Android Runtime (ART) takes that DEX file and compiles it into native machine code using a tool called `dex2oat`. This compiled code is stored in OAT (Optimized Android File Type) format, which is actually an ELF file containing your app's native instructions.

Modern ART uses a hybrid approach: it performs ahead-of-time (AOT) compilation for baseline code at install time, then uses just-in-time (JIT) compilation during execution to optimize frequently-used code paths. This balances fast startup with runtime performance.

Here's the key point: ART itself is a shared library. It lives on your device as `libart.so` - an ELF shared library. When you install an app, the `dex2oat` tool compiles your DEX bytecode into native machine code and packages it in an OAT file. This OAT file is also an ELF shared library (not a standalone executable). When your app runs, ART loads this compiled code and executes it. You're not running bytecode in a virtual machine. You're running native code that was compiled from bytecode, managed by the ART runtime.

The location of ART on your device:
- `/apex/com.android.runtime/lib64/libart.so` (Android 10+)
- `/system/lib64/libart.so` (older versions)

### Path 2: Native Code via NDK

You write code in C, C++, or any language that compiles to native machine code. The NDK (Native Development Kit) compiles this directly into `.so` (shared object) files - ELF libraries. Your app loads these libraries at runtime using `System.loadLibrary()`, and the code executes directly on the CPU.

This is how game engines like Unity and Unreal work. It's also how you can reuse existing C/C++ libraries in your Android app without rewriting them.

## Everything Else Builds on These Two Paths

Once you understand these two fundamental paths, you can understand how every other technology runs on Android.

### Python on Android
A Python interpreter is just a program. Someone compiles the Python interpreter (written in C) as a native library. Your app loads this library, and now you can execute Python code. Projects like Chaquopy do exactly this - they package the CPython interpreter as a `.so` file, load it via JNI, and expose it to your Java/Kotlin code.

### JavaScript Engines (React Native, Cordova)
Same principle. V8, JavaScriptCore, or Hermes are JavaScript engines written in C++. They compile to native libraries. Your app loads the engine, feeds it JavaScript code, and the engine executes it. React Native does this with its "bridge" - a message-passing system where JavaScript sends serialized commands to Java, and Java sends results back. (Note: React Native is moving to a new architecture called JSI that replaces this bridge with direct synchronous C++ communication, but the bridge model illustrates the concept well.)

### Flutter
Flutter works similarly but skips JavaScript entirely. It packages the Dart VM (also written in C++) as a native library. Your Dart code compiles to native ARM/x86 code (in release mode) or runs in the Dart VM (in debug mode). Flutter's engine renders directly to a canvas using Skia (a 2D graphics library), bypassing Android's View system entirely. The result: Flutter apps are native executables talking to a native rendering engine.

### WebView-Based Apps
WebView is a system component that embeds the Chromium browser engine. The web content (HTML/CSS/JavaScript) runs inside this embedded browser. To interact with Android APIs, developers use `addJavascriptInterface()` to create a bridge. This injects Java objects into the JavaScript context, allowing JavaScript code to call Java methods.

## But Running Code Isn't Enough

Here's where it gets interesting. An Android app isn't just code running on a device. It's code running within Android's application framework. Your app needs to:

- Respond to lifecycle events (onCreate, onPause, onDestroy)
- Request permissions
- Access system services (camera, location, notifications)
- Handle intents and inter-process communication
- Manage UI rendering through the view hierarchy

This is why you can't just compile a C++ program and run it on Android like you would on a Linux desktop. Android has its own way of starting and managing applications.

## How Android Loads and Starts Your App

When you tap an app icon, here's what happens:

1. **The Launcher sends an Intent** to the system, saying "start this app"

2. **ActivityManagerService (AMS)** receives the intent and checks if the app's process exists. If not, it asks Zygote to create one.

3. **Zygote** is a special process that starts when Android boots. It pre-loads all the Android framework classes and the ART runtime into memory. When AMS asks for a new app process, Zygote forks itself (using Linux's copy-on-write mechanism). The new process is a copy of Zygote, already loaded with everything an Android app needs.

4. **The new process starts executing at `ActivityThread.main()`**. This is the entry point for every Android app. It's a Java class, but it's called from native code via JNI - this is the handoff from the native world to the Java world.

5. **ActivityThread** sets up the main thread (the UI thread) and creates a message loop using Looper. It then tells AMS "I'm ready."

6. **AMS responds** by instructing the app to create its Application object and start the requested Activity.

7. **Your code finally runs** - onCreate(), onStart(), onResume() get called in sequence.

The important thing to understand: your app's process is a fork of Zygote. It inherits ART and the framework classes. Your DEX code was already compiled to native code during installation. When ActivityThread loads your Application class, it's loading and executing that native code.

**When native libraries load:** If your app uses native code (NDK libraries, game engines, Python interpreters), they typically load during the Application class initialization. You call `System.loadLibrary("mylibrary")` in a static block or in `Application.onCreate()`. This happens before any Activity starts. The library loading:
- Finds the `.so` file in your APK's `lib/` directory for your device's architecture
- Maps it into your process's memory space
- Calls the library's `JNI_OnLoad()` function (if it exists) to register native methods
- Makes all exported symbols available to your app

Example from a native app:
```java
public class MyApplication extends Application {
    static {
        System.loadLibrary("native-lib");  // Loads libnative-lib.so
    }
    
    @Override
    public void onCreate() {
        super.onCreate();
        // Native library is now loaded and ready
        // Can call native methods from here
    }
}
```

This is the moment execution truly spans both worlds - your Java/Kotlin code in ART and your native C/C++ code running directly on the CPU, bridged through JNI.

## Interacting with the Operating System

This is the real challenge for alternative runtimes. Running a Python interpreter is easy. Making it work like an Android app is hard.

You need to:

### 1. Hook into the Application lifecycle
Your custom runtime needs to receive onCreate, onPause, etc. You typically do this by writing a wrapper Activity or Application class in Java/Kotlin that forwards lifecycle events to your custom runtime.

Example: Kivy (a Python framework) has a Java bootstrap that creates a PythonActivity. This Activity's lifecycle methods call into the Python runtime via JNI.

### 2. Bridge to Android APIs
The Python/JavaScript/C++ code running in your executor needs a way to call Android framework methods. This requires:

- Creating JNI bindings (for native code)
- Using reflection or code generation to expose Java classes to your runtime
- Marshaling data between your runtime's types and Java types

React Native does this with its "bridge" - a message-passing system where JavaScript sends serialized commands to Java, and Java sends results back. (Note: React Native is moving to a new architecture called JSI that replaces this bridge with direct synchronous C++ communication, but the bridge model illustrates the concept well.)

### 3. Manage the UI
Android's UI is built on the View system. If you want to display anything, you either:
- Use Android's native Views (by calling into the framework)
- Render to a Surface directly using OpenGL/Vulkan (what game engines do)
- Embed a WebView and render HTML (what Cordova does)

Each approach requires different bridging mechanisms, but they all ultimately call into Android framework code.

## Why This Architecture Exists

You might wonder: why not just let developers write ELF executables directly? Why the DEX → ART → native code pipeline?

Several reasons:

**Platform independence**: DEX bytecode is architecture-agnostic. The same APK runs on ARM, ARM64, x86, and x86_64 devices. ART handles the compilation to the correct native code for each device.

**Optimization opportunities**: ART can optimize code at install time based on the specific device. It can use profile-guided optimization to precompile frequently-used methods.

**Security**: The bytecode verification and AOT compilation process enforces type safety and prevents certain classes of bugs and exploits.

**Managed runtime benefits**: Garbage collection, exception handling, and debugging support come for free when you use ART.

For native code, you give up these benefits in exchange for direct hardware access and maximum performance.

## Putting It All Together

Android runs ELF executables. That's the base layer.

ART is an ELF executable that runs DEX bytecode (by compiling it to native code first). Your Java/Kotlin apps run through ART.

Native libraries are ELF executables that run directly. Your NDK code, game engines, and custom runtimes (Python, JavaScript) run this way.

All of these need to integrate with Android's application framework to be proper "apps." This means hooking into the lifecycle, bridging to system APIs, and managing UI rendering.

React Native, Flutter, Python apps - they all follow the same pattern. Load a runtime executor (JavaScript engine, Dart VM, Python interpreter), feed it your code, bridge to Android APIs through JNI or reflection. The foundation is the same: ELF executables running on Linux, hooking into Android's application framework.

The real work is in the bridging. How efficiently can the framework translate between your code and Android's APIs? How well does it handle lifecycle events? How much overhead does the bridge add? These are engineering problems, not fundamental limitations.

Python on Android works because Android is a general-purpose operating system that runs executables. JavaScript apps work because you can compile a JavaScript engine and load it. There's no special magic. You're just running code through different executors, all of which eventually produce CPU instructions.