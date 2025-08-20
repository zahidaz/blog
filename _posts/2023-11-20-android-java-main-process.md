---
layout: post
title: "Android: The path that leads to Java Main"
date: 2023-11-20
categories: [android, java, internals]
tags: [android, java, zygote, init, app_process]
---

In the domain of Java programming, the `public static void main(String[] args)` method is a common sight, serving as the entry point for numerous Java applications. Nevertheless, as one delves into Android development, seemingly applications operate without the need for the traditional main method. This article seeks to shed light on the topic by exploring not only the existence of a main method in Android but also its role and location within a typical Android application process.

<!--more-->


To gain a better understanding of the context and how everything fits into the larger picture, I will begin by elucidating how the Android operating system initiates and leads to the creation of your application. Given the goal of precision and brevity, this article will be published in parts, with this being the first part.

Starting from the very basics, though space and time are limited, I must commence somewhere. I will not delve into the Android startup process preceding the kernel, assuming the kernel is loaded (there are lots of good resources dealing with this topic), and will begin with the initiation of the first user space process.

Assuming the Android kernel is loaded, among the first process which is launched by the kernel is named `init`, but keep in mind init is not the only process started by the kernel.

The kernel launches the init binary from `/system/bin/` directory in multiple ways depending on the version of android the difference is explained in detail in the Generic Boot Partition. The init process will in turn loads and pares a bunch of `.rc` files.

## Why .rc

> Script file containing startup instructions for an application program (or an entire operating system), usually a text file containing commands of the sort that might have been invoked manually once the system was running but are to be executed automatically each time the system starts up. (sense 1).

Thus, it would seem that the "rc" part stands for "runcom", which I believe can be expanded to "run commands". In fact, this is exactly what the file contains, commands that bash should run. (SE)

Android has its own **Android Init Language** explain in detailed by `init/README.md`!

The `init.rc` plays a crucial role as one of the initial files to be parsed. This substantial file initiates numerous services and daemons in stages, with one notable entity being the renowned **zygote**.

## Android init Trigger Sequence

Init uses the following sequence of triggers during early boot. These are the built-in triggers defined in `init.cpp`.

1. **early-init** - The first in the sequence, triggered after cgroups has been configured but before ueventd's coldboot is complete.
2. **init** - Triggered after coldboot is complete.
3. **charger** - Triggered if `ro.bootmode == "charger"`.
4. **late-init** - Triggered if `ro.bootmode != "charger"`, or via healthd triggering a boot from charging mode.

During the parsing of the `init.rc` file by the init process, a key observation is the `import /init.${ro.zygote}.rc` line at the top that imports another `.rc` file. Given the presence of both 32-bit and 64-bit or other architectures, the resolution of the `ro.zygote` system property becomes significant. It dynamically determines the appropriate name of the `.rc` file situated in the same directory, adapting to the specific architecture in use.

You can see the read only `ro.zygote` system property by getting an adb shell to your device and running `getprop ro.zygote`

```
import /init.${ro.zygote}.rc # <----

# ...

# Mount filesystems and start core system services.
on late-init
    # ... 
    # Now we can start zygote.
    trigger zygote-start # <----
    # ... 
```

So init process is loaded and parsed the `init.rc` file, it will trigger the `late-init` stage as follow (from `init.cpp`).

```cpp
std::string bootmode = GetProperty("ro.bootmode", "");
if (bootmode == "charger") {
    am.QueueEventTrigger("charger");
} else {
    am.QueueEventTrigger("late-init"); // <----
}
```

The `late-init`, in turn, triggers `zygote-start`, consequently initiating the associated service.

```
# ....

# to start-zygote in device's init.rc to unblock zygote start.
on zygote-start
    #...
    start zygote # <----
    #...
```

We're not finished yet; recall the import line. The subsequent details unfold in the imported file. As an illustration, let's examine the 64-bit version of the `.rc` file. It contains a wealth of information, but for a comprehensive understanding, you can refer to the Android init language README. Below, I've included the first line, which defines a service named "zygote." This service is associated with the **THE ACTUAL** file to be executed, located at `/system/bin/app_process64`.

```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
```

## Android Service

A service is a program that starts with service and is started by the init process. Generally, it runs in another child process of init. Therefore, it is necessary to determine whether the corresponding executable file exists before starting the service. The sub-processes generated by init are defined in the rc file, and each service in it will generate sub-processes through the fork method at startup. The form of Services is as follows:

```
service <name> <pathname> [<argument> ]*
 <option>
 <option>
 ...
```

Among them:
- **name**: service name
- **pathname**: the program location corresponding to the current service
- **option**: the option set by the current service
- **argument**: optional parameter


## The App Process

Now, transitioning from the init process, we move to the next user process, which is `app_process` which is started a service. Notably, it isn't named zygote since it can operate in various modes and is initiated as a Command Line process. One of these modes is the **zygote**, and this process assumes the role of the zygote process only when the `--zygote` flag is passed to it, as indicated in the `init.rc` file. It's essential to emphasize that we remain in native land, given that this program is written in C++.

Here is a snippet of the C++ main function for the `app_process`. Initially, it creates an `AppRuntime` object, passing it two different Java class names based on whether it is started as the zygote or not.

```cpp
int main(int argc, char* const argv[])
{
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
//...
    // arguments :
    //
    // --zygote : Start in zygote mode
    // --start-system-server : Start the system server.
    // --application : Start in application (stand alone, non zygote) mode.
    // --nice-name : The nice name for this process.

bool zygote = false;
    while (i < argc) {
//...
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        }
// ....
    }
// ...
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (!className.empty()) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } 
// ...
}
```

Let's examine the `AppRuntime` class. In zygote mode, this class initiates the first Java process which later will be forked and host android apps. Pay attention to the `callMain` function call, which is defined in its superclass `AndroidRuntime`.

```cpp
// frameworks/base/cmds/app_process/app_main.cpp
/*
 * Main entry of app process.
 *
 * Starts the interpreted runtime, then starts up the application.
 *
 */

class AppRuntime : public AndroidRuntime {
    
  virtual void onStarted()
      {
       ....
          AndroidRuntime* ar = AndroidRuntime::getRuntime();
          ar->callMain(mClassName, mClass, mArgs); // WELL WELL ...
       ...
      }
}
```

To further explain, the Java environment is launched by the `runtime.start` method, implemented in the `AndroidRuntime`. This method calls the overridden method in the `AppRuntime`, and within this method, we invoke the `callMain` function of the `AndroidRuntime`.

```cpp
status_t AndroidRuntime::callMain(const String8& className, jclass clazz,
    const Vector<String8>& args){
//...
    methodId = env->GetStaticMethodID(clazz, "main", "([Ljava/lang/String;)V");
//...
    env->CallStaticVoidMethod(clazz, methodId, strArray); 
  // Java Main is acutally called here
//...
}
```

Observing the code above, it becomes evident that there **is** a Java main in Android. Additionally, we now understand all the steps leading to the execution of this function. With this, we initiate the Java environment for apps, which I will delve into in the next part.

You maybe even able to run arbitrary java .jar files using the app_process: https://github.com/connglli/blog-notes/issues/3

## Thank you

Please feel free to share your opinions, suggest corrections, or provide additional information.

I plan to share more detailed and technical insights on mobile platforms, security, and technology in general. Occasionally, I may also include non-technical writings. If you're interested, consider following for future updates.