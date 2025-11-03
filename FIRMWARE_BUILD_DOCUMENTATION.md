# ESPHome Firmware Binary Generation Guide

## Overview

This guide documents the complete process for generating firmware binary files (.bin) from ESPHome YAML configurations, specifically for the ESP32-C3 smoke sensor project.

## Why Generate Binary Files?

Binary files allow you to:
1. **Flash devices without ESPHome CLI** - Use web flashers, esptool, or other tools
2. **Distribute firmware** - Share compiled binaries with users who don't need the source
3. **Version control** - Archive specific firmware versions
4. **Offline flashing** - Flash devices without recompiling
5. **Automation** - Integrate into CI/CD pipelines

## Prerequisites

### Required Software

```bash
# Python 3.12 or higher
python3 --version

# ESPHome CLI
pip install esphome

# Or use a virtual environment (recommended)
python3 -m venv esphome-env
source esphome-env/bin/activate  # Linux/Mac
# or
esphome-env\Scripts\activate  # Windows
pip install esphome
```

### Required Files

1. **YAML Configuration** - Your ESPHome device configuration
2. **secrets.yaml** - WiFi credentials and API keys
3. **Valid board definition** - Correct ESP32 variant specification

## Build Process - Step by Step

### 1. Verify Configuration

Before building, ensure your YAML configuration is valid:

```bash
esphome config esp32-c3-supermini-smoke-fan.yaml
```

**Expected output:**
```
INFO ESPHome 2025.6.3
INFO Reading configuration esp32-c3-supermini-smoke-fan.yaml...
INFO Configuration is valid!
```

**Common Issues:**
- Missing `secrets.yaml` file
- Invalid YAML syntax (indentation errors)
- Unsupported board or platform version

### 2. Compile Firmware

Run the compile command to generate binary files:

```bash
esphome compile esp32-c3-supermini-smoke-fan.yaml
```

**Build Process Timeline:**
```
[0:00] Reading configuration
[0:05] Generating C++ source code
[0:10] Installing platform packages
[0:30] Compiling dependencies (first build only)
[2:00] Compiling main firmware
[2:10] Linking binary
[2:15] Creating .bin files
[2:20] SUCCESS
```

**First build:** ~2-3 minutes (downloads and compiles all dependencies)
**Subsequent builds:** ~30-60 seconds (cached dependencies)

### 3. Locate Generated Files

Binary files are created in the hidden `.esphome` directory:

```bash
.esphome/build/[DEVICE_NAME]/.pioenvs/[DEVICE_NAME]/
```

**For our project:**
```bash
.esphome/build/esp32c3-smoke-sensor/.pioenvs/esp32c3-smoke-sensor/
```

**Key Files Generated:**

| File | Size | Purpose |
|------|------|---------|
| `firmware.bin` | ~989KB | Main firmware (OTA updates) |
| `firmware.factory.bin` | ~1.1MB | Complete image with bootloader |
| `firmware.elf` | ~12MB | Debug symbols (not needed for flashing) |
| `bootloader.bin` | ~28KB | ESP32 bootloader |
| `partitions.bin` | ~3KB | Partition table |

### 4. Copy Files to Distribution Directory

Create a `firmware` directory for easy access:

```bash
mkdir -p firmware

# Copy factory image (for initial USB flash)
cp .esphome/build/esp32c3-smoke-sensor/.pioenvs/esp32c3-smoke-sensor/firmware.factory.bin \
   firmware/esp32c3-smoke-sensor.factory.bin

# Copy OTA image (for updates)
cp .esphome/build/esp32c3-smoke-sensor/.pioenvs/esp32c3-smoke-sensor/firmware.bin \
   firmware/esp32c3-smoke-sensor.bin
```

**Result:**
```
firmware/
├── esp32c3-smoke-sensor.factory.bin  (1.1M)
└── esp32c3-smoke-sensor.bin          (989K)
```

## Understanding the Binary Files

### firmware.factory.bin (Factory Image)

**Contents:**
- Bootloader (offset 0x0)
- Partition table (offset 0x8000)
- NVS (Non-Volatile Storage) partition
- OTA data partition
- Application firmware

**Use cases:**
- Initial USB flash of new devices
- Complete device recovery/reset
- Programming via serial connection

**Flash command:**
```bash
esptool.py --port /dev/ttyUSB0 write_flash 0x0 firmware.factory.bin
```

### firmware.bin (OTA Image)

**Contents:**
- Application firmware only (no bootloader/partitions)

**Use cases:**
- Over-The-Air (OTA) updates
- ESPHome upload command
- Web-based OTA updates

**Size advantage:** Smaller file = faster OTA updates

## Build Output Analysis

### Successful Build Output

```
Linking .pioenvs/esp32c3-smoke-sensor/firmware.elf
RAM:   [=         ]   9.2% (used 30104 bytes from 327680 bytes)
Flash: [======    ]  55.1% (used 1011870 bytes from 1835008 bytes)
Building .pioenvs/esp32c3-smoke-sensor/firmware.bin
Creating esp32c3 image...
Successfully created esp32c3 image.
```

**Key metrics:**
- **RAM Usage:** 9.2% (30KB/320KB) - Excellent, lots of headroom
- **Flash Usage:** 55.1% (989KB/1.8MB) - Good, room for growth
- **Total size:** 1,011,870 bytes (~989KB)

### Memory Optimization

If you encounter memory issues:

```yaml
# Reduce logging verbosity
logger:
  level: INFO  # or WARN instead of DEBUG

# Disable web server (saves ~50KB flash)
# web_server:
#   port: 80

# Use minimal logger (saves RAM)
logger:
  baud_rate: 0  # Disable serial logging
```

## Troubleshooting Build Issues

### Error: "Platform not installed"

```
ERROR Platform espressif32 is not installed
```

**Solution:**
```bash
# ESPHome automatically installs platforms on first build
# If this fails, try manually:
pio platform install espressif32@6.7.0
```

### Error: "No module named 'esphome'"

```
ImportError: No module named 'esphome'
```

**Solution:**
```bash
# Activate virtual environment first
source esphome-env/bin/activate
pip install esphome
```

### Error: "secrets.yaml not found"

```
ERROR Error reading file secrets.yaml: [Errno 2] No such file or directory
```

**Solution:**
```bash
# Create secrets.yaml from template
cp secrets.yaml.example secrets.yaml
# Edit with your actual credentials
nano secrets.yaml
```

### Error: Compilation timeout

```
ERROR Compilation took too long (>600s)
```

**Solution:**
```bash
# Increase timeout or check for:
# 1. Slow internet (downloading packages)
# 2. Low disk space
# 3. Resource constraints
```

### Warning: "Strapping pin GPIO8"

```
WARNING GPIO8 is a strapping PIN and should only be used for I/O with care.
```

**Not an error** - GPIO8 works fine for status LED on ESP32-C3 Super Mini. Can be safely ignored for this use case.

## GitHub Actions Integration (Failed Attempt)

We attempted to use GitHub Actions for automated builds but encountered issues:

### What We Tried

```yaml
# .github/workflows/build-esphome.yaml
- name: Build ESPHome firmware
  uses: esphome/build-action@v4  # Also tried v3
  with:
    yaml_file: esp32-c3-supermini-smoke-fan.yaml
    version: latest
```

### Why It Failed

```
ERROR: This request has been automatically failed because it uses a
deprecated version of `actions/cache: v4.0.2`
```

**Root cause:**
- ESPHome build action uses deprecated GitHub Actions cache
- GitHub enforced breaking changes on 2024-12-05
- Build action hasn't been updated yet

### Alternative: Local Build + Upload

Since GitHub Actions failed, we used this workflow:

```bash
# 1. Build locally
esphome compile esp32-c3-supermini-smoke-fan.yaml

# 2. Copy binaries
mkdir -p firmware
cp .esphome/build/esp32c3-smoke-sensor/.pioenvs/esp32c3-smoke-sensor/firmware.factory.bin firmware/
cp .esphome/build/esp32c3-smoke-sensor/.pioenvs/esp32c3-smoke-sensor/firmware.bin firmware/

# 3. Commit and push
git add firmware/
git commit -m "Add firmware binaries"
git push

# 4. Create GitHub release
gh release create v1.0.0 \
  firmware/esp32c3-smoke-sensor.factory.bin \
  firmware/esp32c3-smoke-sensor.bin \
  --title "ESP32-C3 Smoke Sensor v1.0.0" \
  --notes "Release notes..."
```

## Advanced: Platform IO Direct Build

For advanced users, you can use PlatformIO directly:

```bash
# Navigate to build directory
cd .esphome/build/esp32c3-smoke-sensor

# Run PlatformIO build
pio run

# Build output in:
# .pioenvs/esp32c3-smoke-sensor/firmware.bin
```

**Benefits:**
- More control over build process
- Custom PlatformIO settings
- Integration with PlatformIO ecosystem

## Version Control Best Practices

### What to Commit

```
.gitignore contents:
- .esphome/          # Build artifacts
- secrets.yaml       # Sensitive credentials
- esphome-env/       # Python virtual environment
```

**DO commit:**
- ✅ YAML configuration files
- ✅ `secrets.yaml.example` template
- ✅ Compiled firmware binaries (in `firmware/` directory)
- ✅ Documentation

**DON'T commit:**
- ❌ `.esphome/` build directory
- ❌ `secrets.yaml` with real credentials
- ❌ `esphome-env/` virtual environment
- ❌ Intermediate build files

### Semantic Versioning

Use git tags for firmware releases:

```bash
git tag v1.0.0
git push origin v1.0.0
```

**Version scheme:**
- `v1.0.0` - Initial release
- `v1.1.0` - New features (threshold change)
- `v1.0.1` - Bug fixes
- `v2.0.0` - Breaking changes

## Build Performance

### Build Time Breakdown

| Stage | First Build | Cached Build |
|-------|-------------|--------------|
| Configuration parsing | 5s | 2s |
| Platform installation | 60s | 0s |
| Dependency compilation | 90s | 0s |
| Main firmware compilation | 45s | 30s |
| **Total** | **~2min 30s** | **~30s** |

### Cache Locations

ESPHome caches compiled code to speed up rebuilds:

```
~/.platformio/          # PlatformIO packages (2GB+)
.esphome/build/         # Project-specific cache (100MB+)
```

**Clearing cache:**
```bash
# Full clean (forces complete rebuild)
rm -rf .esphome/
rm -rf ~/.platformio/

# Project clean only
rm -rf .esphome/build/esp32c3-smoke-sensor/
```

## Summary

**Simple build process:**
```bash
# 1. Compile
esphome compile your-config.yaml

# 2. Find binaries
find .esphome -name "firmware*.bin"

# 3. Copy to distribution folder
mkdir firmware
cp .esphome/build/*/pioenvs/*/firmware.factory.bin firmware/
cp .esphome/build/*/pioenvs/*/firmware.bin firmware/
```

**Key files:**
- `firmware.factory.bin` - Full image for USB flashing
- `firmware.bin` - OTA update image

**Build time:**
- First build: ~2-3 minutes
- Subsequent builds: ~30-60 seconds

See `OTA_FLASHING_DOCUMENTATION.md` for instructions on deploying these binaries to devices.
