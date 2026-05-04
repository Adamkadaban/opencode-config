---
name: frida-waydroid
description: Dynamic instrumentation of Android apps using Frida on Waydroid. Use when hooking Java/native methods, bypassing anti-debugging or anti-Frida detection, manipulating SharedPreferences, extracting secrets, or performing runtime analysis of Android apps running in Waydroid.
---

# Frida on Waydroid - Android Dynamic Instrumentation

You are helping the user perform dynamic analysis of Android applications using Frida with Waydroid as the Android runtime environment.

## Prerequisites

- **Waydroid** installed and running (`waydroid session start`)
- **Frida CLI tools** on the host (`frida`, `frida-ps`, `frida-ls-devices`)
- **frida-server** matching your Frida version, running inside Waydroid as root
- **ADB** connected to Waydroid (`adb connect 192.168.240.112:5555` — IP may vary)
- ADB authorization accepted (click "Allow" in Waydroid UI if prompted)

## Critical Environment Constraints

These are hard-won lessons. Violating any will waste significant time.

### 1. USE FRIDA CLI ONLY — NOT Python API

**The Frida Python API (`frida.get_usb_device()`) does NOT support the Java bridge on Waydroid.** `Java.available` will be `false` and `Java.perform()` throws `ReferenceError: 'Java' is not defined`.

```bash
# CORRECT
frida -U -f com.example.app -l hook.js --no-pause

# WRONG — Java bridge will NOT work
python3 -c "import frida; device = frida.get_usb_device(); ..."
```

### 2. Frida API Version Quirks

```javascript
// CORRECT — two-step pattern
var libc = Process.findModuleByName("libc.so");
var fopen = libc.findExportByName("fopen");

// WRONG — does not exist in this version
var fopen = Module.findExportByName("libc.so", "fopen");
```

### 3. Waydroid Shell — Pipe Commands

`waydroid shell` does not accept flags with dashes as arguments:

```bash
# WRONG
sudo waydroid shell -c "ls -la /data"

# CORRECT
echo 'ls -la /data' | sudo waydroid shell sh
```

### 4. Architecture is x86_64

Waydroid on this machine runs **x86_64 Android**. Always use `lib/x86_64/` native libraries, not arm64-v8a.

### 5. Always Use Timeouts

```bash
timeout 30 frida -U -f com.example.app -l hook.js --no-pause
timeout 10 frida-ps -U
```

## Setup

```bash
# Start Waydroid
waydroid status
waydroid session start

# Start frida-server inside Waydroid
echo 'chmod +x /data/local/tmp/frida-server && /data/local/tmp/frida-server -D &' | sudo waydroid shell sh

# Verify
timeout 10 frida-ps -U
```

## Core Hooking Patterns

### Java Method Hook

```javascript
Java.perform(function() {
    var Cls = Java.use("com.example.app.TargetClass");
    Cls.targetMethod.implementation = function(arg1, arg2) {
        console.log("[*] called: " + arg1 + ", " + arg2);
        var result = this.targetMethod(arg1, arg2);
        console.log("[*] returned: " + result);
        return result;
    };
    // Overloaded methods
    Cls.method.overload("long", "long").implementation = function(a, b) {
        return this.method(a, b);
    };
});
```

**Spawn vs Attach:**
- **Spawn** (`frida -U -f pkg -l hook.js --no-pause`) — hooks active before app init
- **Attach** (`frida -U pkg -l hook.js`) — hooks into running app

### Native Function Hook

```javascript
Interceptor.attach(Process.findModuleByName("libc.so").findExportByName("fopen"), {
    onEnter: function(args) {
        this.path = args[0].readUtf8String();
    },
    onLeave: function(retval) {
        console.log("[*] fopen(" + this.path + ") -> " + retval);
    }
});
```

### Universal Anti-Detection Bypass

Block `/proc/` reads to defeat both `/proc/self/maps` scanning and `TracerPid` checks:

```javascript
var libc = Process.findModuleByName("libc.so");
Interceptor.attach(libc.findExportByName("fopen"), {
    onEnter: function(args) { this.path = args[0].readUtf8String(); },
    onLeave: function(retval) {
        if (this.path && this.path.indexOf("/proc/") !== -1) {
            retval.replace(ptr(0));
        }
    }
});
```

This works regardless of whether detection strings (`frida`, `gadget`, `gum-js`, `linjector`) are encrypted with rolling XOR or other ciphers — bypass at the I/O level, not the string level.

## Best Practices

1. **Force-stop before modifying files** — Android caches SharedPreferences in memory
2. **Use spawn mode for init-time hooks** — detection checks run in `onCreate()`/static init
3. **Combine static + dynamic** — jadx/radare2 tells you WHAT to hook, Frida gets the values
4. **Check for decoy flags** — words like `f4k3`, `n1c3_try`, `wr0ng` mean detection wasn't bypassed
5. **Save hooks as `.js` files** — reusable, reproducible, iterable

## Quick Reference

```bash
# --- Setup ---
waydroid status                                    # Check state
adb connect 192.168.240.112:5555                   # Connect ADB
echo '/data/local/tmp/frida-server -D &' | sudo waydroid shell sh
timeout 10 frida-ps -U                             # Verify

# --- App Management ---
adb install app.apk
adb shell am start -n pkg/.Activity
adb shell am force-stop pkg
adb shell pm clear pkg

# --- Frida ---
frida -U -f pkg -l hook.js --no-pause              # Spawn + hook
frida -U pkg -l hook.js                            # Attach + hook
frida -U -f pkg -l bypass.js -l extract.js --no-pause  # Multiple scripts

# --- Files ---
echo 'cat /path' | sudo waydroid shell sh          # Read inside Waydroid
adb pull /remote ./local                            # Pull out
adb push ./local /remote                            # Push in
```

## Supporting Documentation

- **[references/workflows.md](references/workflows.md)** — End-to-end workflows: CTF flag extraction, SSL pinning bypass, class tracing, SharedPreferences manipulation, native library enumeration
- **[references/anti-detection.md](references/anti-detection.md)** — Comprehensive anti-Frida/anti-debug bypass patterns, root detection bypass, detection technique reference table
- **[references/troubleshooting.md](references/troubleshooting.md)** — Problem/solution pairs for all known issues on this environment

