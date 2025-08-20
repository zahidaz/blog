---
layout: post
title: "Frida Python Multiprocessing: Solving Hanging Code Issues"
date: 2024-07-03
categories: [python, frida, development]
tags: [frida, python, multiprocessing, instrumentation, debugging, concurrency]
---

When working with the Frida instrumentation toolkit in a Python multiprocessing environment, you may encounter issues due to the way modules are loaded and managed across processes. This article provides a detailed guide on how to properly use Frida with Python's multiprocessing module, avoiding some bad surprises.

<!--more-->

## What Is Wrong

The core issue arises from the way Frida behaves in python multiprocessing. Frida, when imported at one process and later new process is started some methods will hang such as `frida.get_device_manager().add_remote_device(dev_name)`, especially if the default fork start method is used. This could because fork copies the parent process's memory space to the child process, leading to conflicts.

## How Python Imports Work

In Python, when a module is imported, it is loaded into memory, and its top-level code is executed. This happens only once per process, and the module is then cached. Subsequent imports of the same module in the same process will fetch the module from the cache, avoiding re-execution.

## Solution Overview

To avoid these issues, we can:

1. Use the spawn start method for multiprocessing.
2. Import Frida within the target function rather than at the top of the script.

## Example

Here's a step-by-step example explanation:

### 1. Importing Modules

We start by importing the required modules: multiprocessing and importlib.

```python
import multiprocessing as mp
import importlib
```

### 2. Defining the Target Function

The target function `get_frida_device` imports Frida within the function scope and reloads it to ensure proper initialization. This function also demonstrates adding and removing a remote device using Frida.

```python
def get_frida_device(called_from: str = None):
    import frida # Import frida here to avoid issues
    from frida.core import Device, Session, Script
    importlib.reload(frida)
    
    if called_from:
        print(f"Called from {called_from}")
    
    dev_name = "192.168.50.92:27042"
    print("Before get_frida_device")

    # This will get stuck if called in a subprocess without proper handling
    device = frida.get_device_manager().add_remote_device(dev_name)
    print(device)
    print("After get_frida_device")

    frida.get_device_manager().remove_remote_device(dev_name)
```

### 3. Setting the Multiprocessing Context

We set the multiprocessing context to spawn to avoid issues with the 'fork' method.

```python
ctx = mp.get_context('spawn') # Use 'spawn' to avoid issues
```

### 4. Creating and Starting the Process

We create a new process targeting the `get_frida_device` function and start it.

```python
p = mp.Process(target=get_frida_device)
p.start()
p.join()
```

### 5. Calling the Function in the Main Process

Finally, we call the `get_frida_device` function in the main process for comparison.

```python
get_frida_device("main")
```

## Full Code Listing

Below is the complete script that incorporates the steps mentioned above: to reproduce you can move the import statements to the top of the file and/or using fork multiprocessing context.

```python
import multiprocessing as mp
import importlib

def get_frida_device(called_from:str = None):
    import frida # import frida here to avoid the issue
    # import frida  # don't import at the top of the file because it will break the multiprocessing
    from frida.core import Device, Session, Script

    importlib.reload(frida)
    if called_from:
        print(f"called from {called_from}")
    dev_name = "192.168.50.92:27042"
    print("before get_frida_device")

    # if in sub process, it will stuck here
    device = frida.get_device_manager().add_remote_device(dev_name)
    print(device)
    print("after get_frida_device")
    frida.get_device_manager().remove_remote_device(dev_name)
 
ctx = mp.get_context('spawn') # use 'spawn' to avoid the issue
# ctx = mp.get_context('fork') # fork may cause the issue too

p = mp.Process(target=get_frida_device)
p.start()
p.join()

get_frida_device("main")
```

## Key Points to Remember

- **Import Location Matters**: Import Frida within the function that will use it, not at the module level when using multiprocessing.
- **Use Spawn Method**: The 'spawn' start method creates a fresh Python interpreter process, avoiding memory space conflicts.
- **Reload Module**: Using `importlib.reload(frida)` ensures proper initialization in each process.
- **Avoid Fork**: The 'fork' method can cause hanging issues with Frida due to memory space copying.

## Final Words

With an understanding of this insidious issue and equipped with information on how to address and avoid it until it is permanently resolved by the developers you will have easy time instrumenting your next application.

Stay Curious.