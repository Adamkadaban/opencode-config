---
name: android-root-hiding
description: Diagnoses and bypasses root/integrity detection on Magisk-rooted Android devices connected via ADB. Use when an app refuses to run on a rooted device, shows "device modified" or "device integrity" errors, fails Play Integrity or SafetyNet checks, or when setting up root hiding after rooting a new device. Also use when debugging which detection mechanism an app uses (RootBeer, Play Integrity API, SafetyNet, custom native checks). Triggers on "rooted device detected", "device has been modified", "Play Integrity", "root detection bypass", "Shamiko", "Tricky Store", "DenyList".
allowed-tools: Bash Read Glob Grep
---

# Android Root Hiding & Integrity Bypass

You are helping the user hide root and pass device integrity checks on a Magisk-rooted Android device connected via ADB.

## When to Use

- An app shows "device modified", "rooted device detected", or similar errors
- An app refuses to launch, login, or function on a rooted device
- Play Integrity or SafetyNet attestation is failing
- Setting up root hiding on a newly rooted device
- Debugging which detection layer (local library vs server-side attestation) an app uses

## When NOT to Use

- Device is not rooted or does not use Magisk — this skill assumes Magisk
- User wants to root a device — use Android rooting guides instead
- User wants to reverse-engineer an app's full logic — use the jadx or ghidra skills
- User wants Frida-based runtime hooks — use the frida-waydroid skill
- User wants to perform an OTA update only (no root detection issue) — guide them through Magisk's "Install to Inactive Slot" directly

## Essential Principles

1. **Diagnose before fixing.** Always identify which detection layer is active (local library, Play Integrity, or both) via logcat and APK decompilation before installing modules. Blind module installation wastes reboots.
2. **Reboot, don't kill GMS.** Killing Google Play Services (`com.google.android.gms`) breaks the Play Integrity binder and produces false failures. Always prefer `adb reboot` over `killall`.
3. **Clear app data before each test.** Apps cache detection results. Always `adb shell pm clear <package>` before retesting after a configuration change.
4. **Modules are interdependent.** Shamiko requires DenyList enforcement OFF. PIF's `spoofProvider` must be OFF when Tricky Store is installed. The order and configuration matter.

## Prerequisites

- **ADB** connected and authorized (`adb devices` shows `device`, not `unauthorized`)
- **Magisk** installed with root access (verify: `adb shell su -c 'magisk -c'`)
- **Unlocked bootloader** (typical for rooted Pixels, OnePlus, etc.)

## Architecture of Root Detection

Apps typically use multiple layers. Identify which layers are active before applying fixes.

### Layer 1: Local Root Detection Libraries

**Common libraries:** RootBeer, rootdetection, custom checks.

**What they check:**
- Known root app packages via PackageManager (`com.topjohnwu.magisk`, SuperSU, Xposed, root checker apps, etc.)
- SU/Magisk binary existence in filesystem paths (`/system/bin/su`, `/system/xbin/su`, `/sbin/su`, etc.)
- `which su` / `which magisk` shell commands
- System properties: `ro.debuggable=1`, `ro.secure=0`, `ro.build.tags=test-keys`
- RW mount status on system partitions
- Native JNI checks via C libraries (e.g., `libtoolChecker.so` in RootBeer)

**Detection method:** Decompile the APK with jadx and search for detection code:
```bash
jadx --deobf --no-res -j 4 app.apk -d app-decompiled
grep -riE "rootbeer|isRooted|checkRoot|detectRoot|Superuser|busybox|test-keys|ro\.debuggable" app-decompiled/sources/
grep -riE "com\.topjohnwu\.magisk|com\.noshufou\.android\.su|eu\.chainfire\.supersu" app-decompiled/sources/
```

**Bypass:** Shamiko module in blacklist mode (see Phase 2 below).

### Layer 2: Play Integrity API (Server-Side)

**How it works:**
1. App calls `requestIntegrityToken()` via Play Core library
2. Google Play Services (GMS) generates a signed token using hardware key attestation
3. App sends token to its backend server
4. Server decodes token and checks verdict levels:
   - `MEETS_BASIC_INTEGRITY` - device may be rooted but not heavily modified
   - `MEETS_DEVICE_INTEGRITY` - device passes CTS, bootloader appears locked
   - `MEETS_STRONG_INTEGRITY` - hardware-backed attestation, hardest to spoof

**Detection in logcat:**
```
I/PlayCore: IntegrityService : requestIntegrityToken(...)
I/Finsky:  Integrity key attestation record generated successfully.
I/Finsky:  requestIntegrityToken() finished for <package>.
```

**Attestation failure indicators:**
```
W/TapAndPay: Device fails attestation
W/Pay:      Device fails attestation
```

**Bypass:** PlayIntegrityFork + Tricky Store (see Phases 3-4 below).

### Layer 3: Additional Signals

- **Installed root-related apps**: Root checker apps, Xposed modules, Lucky Patcher, etc. are detected by package name scanning. Uninstall them.
- **Magisk app package name**: `com.topjohnwu.magisk` is a detection target. Hide/repackage via Magisk Settings.
- **SafetyNet** (deprecated predecessor to Play Integrity): Some older apps still use it.

## Phase 1: Enable Zygisk and Configure DenyList

**Entry:** ADB connected, Magisk installed, target app package name known.
**Exit:** Zygisk enabled, target app + GMS + Play Store in DenyList, device rebooted.

### Check current state
```bash
# Check if Zygisk is enabled (look for zygisk entry)
adb shell su -c 'cat /data/adb/magisk.db 2>/dev/null | strings' | grep zygisk

# Check current DenyList
adb shell su -c 'magisk --denylist ls'

# Check installed modules
adb shell su -c 'ls /data/adb/modules/'
```

### Enable Zygisk
Must be done in Magisk app UI: **Settings -> Zygisk -> ON**. Requires reboot.

### Configure DenyList
Add the target app, Google Play Services, and Google Play Store:
```bash
adb shell su -c 'magisk --denylist add <target.package.name>'
adb shell su -c 'magisk --denylist add com.google.android.gms'
adb shell su -c 'magisk --denylist add com.android.vending'
```

Verify:
```bash
adb shell su -c 'magisk --denylist ls'
```

## Phase 2: Install Shamiko

**Entry:** Zygisk enabled and device rebooted.
**Exit:** Shamiko installed, DenyList enforcement disabled, module shows "blacklist mode" after reboot.

Shamiko provides stronger root hiding than DenyList enforcement alone. It reads the DenyList as its hide list but must have DenyList enforcement **disabled**.

### Install
```bash
# Find latest release
SHAMIKO_URL=$(curl -sL "https://api.github.com/repos/LSPosed/LSPosed.github.io/releases" | \
  python3 -c "import json,sys; data=json.load(sys.stdin)
for r in data:
    for a in r.get('assets',[]):
        if 'shamiko' in a['name'].lower() and a['name'].endswith('.zip'):
            print(a['browser_download_url']); sys.exit(0)")

curl -sL -o /tmp/Shamiko.zip "$SHAMIKO_URL"
adb push /tmp/Shamiko.zip /data/local/tmp/Shamiko.zip
adb shell su -c 'magisk --install-module /data/local/tmp/Shamiko.zip'
adb shell rm /data/local/tmp/Shamiko.zip
```

### Disable DenyList enforcement (critical for Shamiko)
Shamiko reads the DenyList as its own hide list but conflicts with Magisk's enforcement. Write a script to avoid shell quoting issues:

```bash
cat > /tmp/denyoff.sh << 'EOF'
#!/system/bin/sh
magisk --sqlite "UPDATE settings SET value=0 WHERE key='denylist';"
echo "DONE"
EOF
adb push /tmp/denyoff.sh /data/local/tmp/denyoff.sh
adb shell su -c 'sh /data/local/tmp/denyoff.sh'
adb shell rm /data/local/tmp/denyoff.sh
```

### Verify after reboot
Look for blacklist mode confirmation in module description:
```bash
adb shell su -c 'cat /data/adb/modules/zygisk_shamiko/module.prop' | grep description
# Should contain "blacklist mode"
```

## Phase 3: Install PlayIntegrityFork

**Entry:** Logcat shows `requestIntegrityToken()` calls (app uses Play Integrity).
**Exit:** PIF installed with `zygisk/` dir containing `.so` files, `autopif4.sh` run, logcat shows `PIF/Java:DG` spoofing lines after reboot.

PlayIntegrityFork (by osm0sis) is the actively maintained successor to the original Play Integrity Fix. It spoofs build fingerprint and system properties to GMS DroidGuard.

**Important:** The original chiteroman/PlayIntegrityFix is discontinued. Use the fork. Also beware of third-party forks (e.g., "PIFS China") that may lack Zygisk components and do nothing useful.

### Install
```bash
PIF_URL=$(curl -sL "https://api.github.com/repos/osm0sis/PlayIntegrityFork/releases/latest" | \
  python3 -c "import json,sys; data=json.load(sys.stdin)
for a in data.get('assets',[]):
    if a['name'].endswith('.zip') and 'PlayIntegrity' in a['name']:
        print(a['browser_download_url']); sys.exit(0)")

curl -sL -o /tmp/PIF.zip "$PIF_URL"
adb push /tmp/PIF.zip /data/local/tmp/PIF.zip
adb shell su -c 'magisk --install-module /data/local/tmp/PIF.zip'
adb shell rm /data/local/tmp/PIF.zip
```

Reboot, then generate a fingerprint:
```bash
adb shell su -c 'sh /data/adb/modules/playintegrityfix/autopif4.sh'
```

This selects a Pixel Beta canary build fingerprint (rotates periodically; check expiry date in output).

### Verify PIF is hooking
```bash
adb logcat -d | grep "PIF/"
```

Look for:
```
D/PIF/Native: Spoofing Build Fields enabled!
D/PIF/Native: Spoofing System Properties enabled!
D/PIF/Java:DG: [DEVICE]: <real> -> <spoofed>
D/PIF/Java:DG: [FINGERPRINT]: <real> -> <spoofed>
```

### Enable verbose logging for debugging
Edit the config to set `verboseLogs=100`:
```bash
cat > /tmp/pif_verbose.sh << 'EOF'
#!/system/bin/sh
sed -i 's/verboseLogs=0/verboseLogs=100/' /data/adb/modules/playintegrityfix/custom.pif.prop
EOF
adb push /tmp/pif_verbose.sh /data/local/tmp/pif_verbose.sh
adb shell su -c 'sh /data/local/tmp/pif_verbose.sh'
adb shell rm /data/local/tmp/pif_verbose.sh
```

## Phase 4: Install Tricky Store (Hardware Attestation)

**Entry:** PIF is hooking (confirmed via logcat), but logcat still shows `Device fails attestation`.
**Exit:** Tricky Store installed with `keybox.xml` present, target app in target list, `spoofProvider=0` in PIF config, device rebooted.

If Play Integrity still fails after PIF, the hardware key attestation is revealing the unlocked bootloader. Tricky Store spoofs the keystore attestation certificate chain.

### Install
```bash
TS_URL=$(curl -sL "https://api.github.com/repos/5ec1cff/TrickyStore/releases/latest" | \
  python3 -c "import json,sys; data=json.load(sys.stdin)
for a in data.get('assets',[]):
    if a['name'].endswith('.zip'):
        print(a['browser_download_url']); sys.exit(0)")

curl -sL -o /tmp/TrickyStore.zip "$TS_URL"
adb push /tmp/TrickyStore.zip /data/local/tmp/TrickyStore.zip
adb shell su -c 'magisk --install-module /data/local/tmp/TrickyStore.zip'
adb shell rm /data/local/tmp/TrickyStore.zip
```

### Configure target list
Tricky Store needs to know which apps to spoof attestation for. It may auto-generate a target list, or you can set it manually:
```bash
adb shell su -c 'cat /data/adb/tricky_store/target.txt' | head -20
```

Ensure the target app and `com.google.android.gms` are in the list.

### Keybox
Tricky Store needs a `keybox.xml` in `/data/adb/tricky_store/`. It may auto-download one during installation. Check:
```bash
adb shell su -c 'ls -la /data/adb/tricky_store/keybox.xml'
```

If missing, consult the Tricky Store XDA thread for sourcing valid keyboxes.

### Clean up target list
Remove packages that should not be present (e.g., the Magisk package if it was hidden):
```bash
cat > /tmp/clean_ts.sh << 'EOF'
#!/system/bin/sh
sed -i '/com.topjohnwu.magisk/d' /data/adb/tricky_store/target.txt
EOF
adb push /tmp/clean_ts.sh /data/local/tmp/clean_ts.sh
adb shell su -c 'sh /data/local/tmp/clean_ts.sh'
adb shell rm /data/local/tmp/clean_ts.sh
```

## Phase 5: Hide the Magisk App

**Entry:** Root hiding modules installed.
**Exit:** `adb shell pm list packages | grep -i magisk` returns nothing.

Repackage the Magisk app under a random package name so package scanners cannot find `com.topjohnwu.magisk`.

This must be done in the Magisk app UI: **Settings -> Hide the Magisk app**. It will prompt for a new app name and repackage itself.

Verify the original package is gone:
```bash
adb shell pm list packages | grep -i magisk
# Should return nothing
```

## Phase 6: Remove Root-Related Apps

**Entry:** Detection persists after Phases 1-5, or as a preventive measure.
**Exit:** No root-indicator packages remain installed.

Uninstall any root checker, Xposed manager, or similar apps that appear in detection libraries' package lists:
```bash
# Common root-indicator packages to check for and remove
for pkg in com.jrummyapps.rootchecker com.devadvance.rootcloak de.robv.android.xposed \
  com.saurik.substrate com.amphoras.hidemyroot com.formyhm.hideroot \
  com.noshufou.android.su eu.chainfire.supersu com.koushikdutta.superuser; do
  adb shell pm list packages 2>/dev/null | grep -q "$pkg" && \
    echo "REMOVING: $pkg" && adb shell pm uninstall "$pkg"
done
```

## Debugging: Identify What's Detecting Root

### Step 1: Capture logcat during app launch
```bash
adb logcat -c
adb shell pm clear <target.package>
adb logcat -v time > /tmp/app_logcat.txt &
LOGPID=$!
# User opens the app manually (stopped apps cannot be launched via ADB after pm clear)
sleep 20
kill $LOGPID
```

### Step 2: Analyze detection vectors
```bash
# RootBeer / local root checks
grep -iE "rootbeer|root.*detect|isRooted|QLog|toolChecker" /tmp/app_logcat.txt

# Play Integrity
grep -iE "IntegrityService|requestIntegrity|Finsky.*integrity|DroidGuard" /tmp/app_logcat.txt

# Attestation failures
grep -iE "Device fails attestation|attestation.*fail" /tmp/app_logcat.txt

# PIF hook activity
grep "PIF/" /tmp/app_logcat.txt

# SafetyNet (legacy)
grep -iE "safetynet" /tmp/app_logcat.txt
```

### Step 3: Decompile and find detection code
```bash
# Pull APK
PKG=<target.package>
APK_PATH=$(adb shell pm path $PKG | head -1 | sed 's/package://')
adb pull "$APK_PATH" /tmp/app.apk

# Decompile (code only, with deobfuscation)
jadx --deobf --no-res -j 4 /tmp/app.apk -d /tmp/app-decompiled

# Search for detection patterns
grep -riE "rootbeer|isRooted|checkRoot|detectRoot|RootBeer" /tmp/app-decompiled/sources/
grep -riE "PlayIntegrity|SafetyNet|IntegrityManager|requestIntegrityToken" /tmp/app-decompiled/sources/
grep -riE "device.*modified|tamper|jailbreak|factory.*settings" /tmp/app-decompiled/sources/
grep -riE "com\.topjohnwu\.magisk|com\.noshufou\.android\.su|Superuser" /tmp/app-decompiled/sources/
```

### Step 4: Check what non-root processes can see
These commands run as the ADB shell user (not root), simulating what a DenyListed app sees:
```bash
adb shell which su           # Should return nothing if Shamiko is working
adb shell which magisk       # Should return nothing if Shamiko is working
adb shell getprop ro.boot.verifiedbootstate  # Should show "green" (spoofed)
adb shell getprop ro.debuggable              # Should be "0"
adb shell getprop ro.secure                  # Should be "1"
adb shell getprop ro.build.tags              # Should be "release-keys"
```

**Note:** These commands test from the shell user's perspective, not the app's. Shamiko only hides from processes in the DenyList. The shell user may still see su/magisk binaries even when the target app cannot.

## Testing Cycle

After each change:
1. Clear the target app's data: `adb shell pm clear <package>`
2. Kill GMS only if necessary (prefer reboot to avoid breaking PI binder): `adb reboot`
3. Wait for full boot: poll `adb shell getprop sys.boot_completed`
4. Wait 15s extra for GMS to initialize
5. Have the user open the app from the launcher (stopped apps cannot be ADB-launched after `pm clear`)
6. Check the result

## The Full Module Stack (Summary)

The complete root-hiding stack, in order of necessity:

| Module | Purpose | Required? |
|--------|---------|-----------|
| **Zygisk** (built into Magisk) | Framework for hiding modules | Yes |
| **Shamiko** | Hides root artifacts from DenyListed apps | Yes |
| **PlayIntegrityFork** | Spoofs build fingerprint to pass PI DEVICE verdict | Yes, if app checks PI |
| **Tricky Store** | Spoofs hardware key attestation | Yes, if PI still fails |
| **Hide Magisk App** (built into Magisk) | Repackages Magisk under random name | Yes |

Additional actions:
- Uninstall root checker / Xposed / root-related apps
- Add target app + `com.google.android.gms` + `com.android.vending` to DenyList
- Disable DenyList enforcement when using Shamiko

## Common Pitfalls

1. **DenyList enforcement ON with Shamiko**: Shamiko requires enforcement OFF. It reads the list directly.
2. **Killing GMS before testing**: Causes "Binder has died" errors in Play Integrity. Reboot instead.
3. **Fake PIF modules**: Third-party forks (e.g., "PIFS China") may lack Zygisk components and do nothing. Verify the module has a `zygisk/` directory with `.so` files.
4. **Stale fingerprints**: PIF canary fingerprints expire. Re-run `autopif4.sh` periodically.
5. **Root checker apps installed**: Apps like `com.jrummyapps.rootchecker` are in detection library package lists. Their mere presence triggers detection even if root itself is hidden.
6. **Not clearing app data**: Apps may cache previous detection results. Always `pm clear` before retesting.
7. **`spoofProvider` with Tricky Store**: When using Tricky Store for attestation, disable `spoofProvider` in PIF config to avoid conflicts. Tricky Store handles the keystore.

## OTA Updates on Rooted A/B Devices

When updating Android on a rooted device with A/B partitions:

1. Go to **Settings -> System -> System update** and download the update
2. **Do NOT press "Restart"** from the system updater
3. Open **Magisk app -> Install -> "Install to Inactive Slot (After OTA)"**
4. Let Magisk patch the inactive slot and reboot through Magisk
5. Verify: `adb shell getprop ro.build.version.security_patch` (should be newer)
6. Verify: `adb shell su -c 'magisk -c'` (root should still work)

## Success Criteria

- [ ] Target app launches without root/integrity/tamper detection errors
- [ ] Logcat shows no `Device fails attestation` warnings
- [ ] `adb shell su -c 'magisk -c'` confirms root still works
- [ ] Shamiko module.prop shows "blacklist mode"
- [ ] PIF logcat shows `Spoofing Build Fields enabled!` and fingerprint spoofing lines
- [ ] `adb shell pm list packages | grep -i magisk` returns nothing (Magisk app hidden)
- [ ] No root-indicator apps remain installed
