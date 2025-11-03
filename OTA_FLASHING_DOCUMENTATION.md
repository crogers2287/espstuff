# OTA (Over-The-Air) Flashing Guide for ESP32

## Overview

This guide documents the complete process for remotely flashing ESP32 devices using OTA (Over-The-Air) updates via ESPHome. This allows you to update firmware without physical access to the device.

## Prerequisites

### Hardware Requirements

1. **ESP32 device** - Already flashed with ESPHome firmware that has OTA enabled
2. **Network connection** - Device must be connected to WiFi and reachable on the network
3. **Stable power** - Device must remain powered during the entire update process

### Software Requirements

```bash
# ESPHome CLI installed
pip install esphome

# Or in virtual environment
source esphome-env/bin/activate
```

### Configuration Requirements

Your ESPHome YAML must include OTA configuration:

```yaml
# Required in your YAML configuration
ota:
  - platform: esphome
    password: !secret ota_password

# Required for OTA discovery
api:
  encryption:
    key: !secret api_encryption_key
```

## OTA Flashing Methods

### Method 1: ESPHome Upload Command (Recommended)

This is the easiest and most reliable method for OTA updates.

#### Step 1: Find Device IP Address

**Option A: Check Home Assistant**
```
Settings → Devices & Services → ESPHome → [Your Device] → IP Address
```

**Option B: Check Router DHCP Leases**
```
Router Admin → DHCP Clients → Find hostname: esp32c3-smoke-sensor
```

**Option C: ESPHome Discovery**
```bash
esphome discover
```

**Example output:**
```
Found 1 devices:
  - esp32c3-smoke-sensor.local at 192.168.1.197
```

**Option D: Check Device Logs** (if serial connected)
```bash
esphome logs esp32-c3-supermini-smoke-fan.yaml
```
Look for: `INFO WiFi Connected! IP Address: 192.168.1.197`

#### Step 2: Compile and Upload

**Using IP Address:**
```bash
esphome upload esp32-c3-supermini-smoke-fan.yaml --device 192.168.1.197
```

**Using mDNS Hostname:**
```bash
esphome upload esp32-c3-supermini-smoke-fan.yaml --device esp32c3-smoke-sensor.local
```

#### Step 3: Monitor Upload Progress

**Expected Output:**
```
INFO ESPHome 2025.6.3
INFO Reading configuration esp32-c3-supermini-smoke-fan.yaml...
WARNING `attenuation: 11db` is deprecated, use `attenuation: 12db` instead
INFO Connecting to 192.168.1.197 port 3232...
INFO Connected to 192.168.1.197
INFO Uploading /path/to/firmware.bin (1012240 bytes)
Uploading: [============================================] 100%
Done...

INFO Upload took 4.89 seconds, waiting for result...
INFO OTA successful
INFO Successfully uploaded program.
```

**Upload Statistics:**
- File size: ~989KB (OTA) or ~1.1MB (factory)
- Upload time: ~5 seconds over WiFi
- Connection: TCP port 3232

#### Step 4: Verify Update

**Option A: Check logs**
```bash
esphome logs esp32-c3-supermini-smoke-fan.yaml --device 192.168.1.197
```

**Expected output:**
```
INFO OTA Update Start
INFO OTA Update Successful
INFO Rebooting...
INFO MQ2 sensor warming up for 60 seconds...
```

**Option B: Check Home Assistant**
- Device should reconnect automatically
- New firmware version visible in device info
- All entities should be available

### Method 2: ESPHome Run Command (Compile + Upload)

This method compiles the firmware and uploads in one command:

```bash
esphome run esp32-c3-supermini-smoke-fan.yaml --device 192.168.1.197
```

**When to use:**
- Making configuration changes
- Want latest version compiled
- First upload after YAML changes

**Process:**
1. Compiles firmware (~30-60 seconds)
2. Uploads to device (~5 seconds)
3. Monitors device reboot

### Method 3: Web-Based OTA Upload

ESPHome devices have a built-in web server for OTA updates:

#### Step 1: Access Web Interface

```
http://192.168.1.197/
```

Or using hostname:
```
http://esp32c3-smoke-sensor.local/
```

#### Step 2: Upload Binary File

1. Click "OTA Update" or "Choose File"
2. Select `firmware.bin` (NOT firmware.factory.bin)
3. Click "Update"

**Web interface shows:**
- Current firmware version
- Device uptime
- WiFi signal strength
- All sensor readings

#### Step 3: Wait for Update

**Progress indicators:**
- Upload progress bar (0-100%)
- Update status messages
- Automatic page refresh after reboot

**Update time:** ~10-15 seconds total

### Method 4: Home Assistant ESPHome Integration

Update directly from Home Assistant:

#### Step 1: Access ESPHome Device

```
Settings → Devices & Services → ESPHome → esp32c3-smoke-sensor
```

#### Step 2: Update Firmware

1. Click "UPDATE FIRMWARE" button
2. Upload compiled `.bin` file
3. Wait for completion

**Advantages:**
- Visual progress tracking
- Integrated with HA
- Can be automated

## OTA Password Configuration

### Setting Up OTA Password

**In secrets.yaml:**
```yaml
ota_password: "your_secure_password_here"
```

**In device YAML:**
```yaml
ota:
  - platform: esphome
    password: !secret ota_password
```

**Security best practices:**
- Use strong passwords (16+ characters)
- Different password per device
- Store in `secrets.yaml` (not in main YAML)
- Never commit `secrets.yaml` to git

### OTA Without Password (Not Recommended)

```yaml
ota:
  - platform: esphome
    # No password - anyone on network can update!
```

**Only use for:**
- Testing environments
- Isolated networks
- Development devices

## Troubleshooting OTA Issues

### Error: "Connection timeout"

```
ERROR Error connecting to 192.168.1.197: [Errno 110] Connection timed out
```

**Causes:**
1. Device is offline or unreachable
2. Wrong IP address
3. Firewall blocking port 3232
4. Device crashed/frozen

**Solutions:**
```bash
# Test basic connectivity
ping 192.168.1.197

# Check if OTA port is open
nc -zv 192.168.1.197 3232

# Try mDNS hostname instead
esphome upload esp32-c3-supermini-smoke-fan.yaml --device esp32c3-smoke-sensor.local

# Check device logs (if serial connected)
esphome logs esp32-c3-supermini-smoke-fan.yaml --device /dev/ttyUSB0
```

### Error: "Authentication failed"

```
ERROR Authentication failed
```

**Causes:**
1. Wrong OTA password
2. `secrets.yaml` not in current directory
3. Password changed but device not updated

**Solutions:**
```bash
# Verify secrets.yaml exists and has correct password
cat secrets.yaml | grep ota_password

# If password changed, must reflash via USB first
esphome upload esp32-c3-supermini-smoke-fan.yaml --device /dev/ttyUSB0
```

### Error: "Not enough space"

```
ERROR OTA update failed: Not enough space
```

**Causes:**
1. Firmware too large for partition
2. Partition scheme incorrect

**Solutions:**
```yaml
# Add to ESP32 configuration
esp32:
  board: esp32-c3-devkitm-1
  framework:
    type: esp-idf
  partition_scheme: default  # or 'minimal' for more app space
```

### Device Disconnects During Upload

**Symptoms:**
- Upload reaches 50-80%
- Connection lost
- Device doesn't reboot

**Causes:**
1. Unstable WiFi connection
2. Interference
3. Power issues
4. Weak signal strength

**Solutions:**
```bash
# Check signal strength in logs
esphome logs esp32-c3-supermini-smoke-fan.yaml

# Look for: Signal strength: -52 dBm
# Good: -30 to -60 dBm
# Fair: -60 to -70 dBm
# Poor: -70 to -80 dBm
# Bad: -80+ dBm

# Improve signal:
# - Move device closer to router
# - Add WiFi extender
# - Switch to 2.4GHz (better range)
# - Check for interference sources
```

### Upload Successful But Device Won't Boot

**Symptoms:**
- Upload completes 100%
- Device doesn't reconnect
- No response from device

**Recovery:**
```bash
# Try power cycle
# Unplug device, wait 10 seconds, replug

# If still not working, reflash via USB
esphome upload esp32-c3-supermini-smoke-fan.yaml --device /dev/ttyUSB0

# Check serial logs for boot errors
esphome logs esp32-c3-supermini-smoke-fan.yaml --device /dev/ttyUSB0
```

## OTA Best Practices

### Pre-Update Checklist

- ✅ Device is online and responding
- ✅ Strong WiFi signal (-60 dBm or better)
- ✅ Stable power source
- ✅ Configuration validated: `esphome config your-file.yaml`
- ✅ Test build successful: `esphome compile your-file.yaml`
- ✅ Backup current firmware if critical

### During Update

- ✅ Don't unplug device
- ✅ Don't disconnect WiFi
- ✅ Don't interrupt upload
- ✅ Monitor progress
- ✅ Wait for confirmation

### Post-Update Verification

```bash
# 1. Check logs for successful boot
esphome logs esp32-c3-supermini-smoke-fan.yaml --device 192.168.1.197

# 2. Verify all sensors working
# Look for sensor readings in logs

# 3. Check Home Assistant integration
# All entities should be available

# 4. Test smoke sensor
# Trigger sensor and verify reading changes

# 5. Verify threshold
# Check that 0.8V threshold is active
```

## Advanced OTA Features

### Safe Mode

ESPHome includes automatic safe mode if OTA upload fails:

```yaml
ota:
  - platform: esphome
    password: !secret ota_password
    safe_mode: true  # Enable automatic safe mode
```

**How it works:**
1. Device fails to boot after OTA
2. Automatically enters safe mode
3. Minimal firmware loads
4. OTA port stays open for recovery
5. Can flash working firmware

### OTA with mDNS

mDNS allows using hostname instead of IP:

```yaml
# Automatically enabled with api component
api:
  encryption:
    key: !secret api_encryption_key
```

**Usage:**
```bash
# Instead of IP address
esphome upload device.yaml --device 192.168.1.197

# Use hostname
esphome upload device.yaml --device esp32c3-smoke-sensor.local
```

**Advantages:**
- Works even if IP changes (DHCP)
- Easier to remember
- Works across subnets (with mDNS relay)

### Rollback Strategy

If new firmware has issues:

#### Method 1: Reflash Previous Version

```bash
# Keep old firmware.bin files
firmware/
├── esp32c3-smoke-sensor-v1.0.0.bin
├── esp32c3-smoke-sensor-v1.1.0.bin
└── esp32c3-smoke-sensor.bin  # Latest

# Flash previous version
esphome upload --device 192.168.1.197 --file firmware/esp32c3-smoke-sensor-v1.0.0.bin
```

#### Method 2: Git Revert

```bash
# Revert to previous configuration
git checkout HEAD~1 -- esp32-c3-supermini-smoke-fan.yaml

# Recompile and upload
esphome run esp32-c3-supermini-smoke-fan.yaml --device 192.168.1.197
```

## OTA Performance Optimization

### Faster Uploads

```yaml
# Reduce logging during OTA
logger:
  level: INFO  # Less verbose than DEBUG

# Disable web server temporarily
# web_server:
#   port: 80

# Optimize WiFi
wifi:
  power_save_mode: light  # Better than 'none' for OTA
  fast_connect: true
```

### Reduce Binary Size

```yaml
# Remove unnecessary components
# Smaller binary = faster upload

# Minimal logging
logger:
  baud_rate: 0

# No web server
# web_server:
#   port: 80

# Disable unused sensors
# Comment out sensors you don't need
```

## Real-World OTA Example

### Our Successful OTA Flash

**Device:** ESP32-C3 Super Mini (Smoke Sensor)
**IP Address:** 192.168.1.197
**OTA Password:** your_ota_password
**Firmware Size:** 1,012,240 bytes (~989KB)

**Command Used:**
```bash
source esphome-env/bin/activate
esphome upload esp32-c3-supermini-smoke-fan.yaml --device 192.168.1.197
```

**Output:**
```
INFO ESPHome 2025.6.3
INFO Reading configuration esp32-c3-supermini-smoke-fan.yaml...
WARNING `attenuation: 11db` is deprecated, use `attenuation: 12db` instead
WARNING GPIO8 is a strapping PIN... [safe to ignore]
INFO Connecting to 192.168.1.197 port 3232...
INFO Connected to 192.168.1.197
INFO Uploading /path/to/firmware.bin (1012240 bytes)
Uploading: [============================================] 100%
Done...

INFO Upload took 4.89 seconds, waiting for result...
INFO OTA successful
INFO Successfully uploaded program.
```

**Timeline:**
- 0:00 - Configuration read
- 0:02 - Connected to device
- 0:02-0:07 - Upload firmware (5 seconds)
- 0:07 - Verification
- 0:08 - Device rebooting
- 0:15 - Device back online
- 0:75 - Sensor ready (60s warmup)

**Success Indicators:**
1. ✅ Upload reached 100%
2. ✅ "OTA successful" message
3. ✅ Device reconnected to WiFi
4. ✅ Sensors resumed operation
5. ✅ New threshold (0.8V) active

## Security Considerations

### OTA Security

**Enabled by default:**
- Password authentication
- Encrypted connection (with API encryption)
- Local network only

**Additional security:**
```yaml
# Restrict OTA to specific IPs
wifi:
  ap:
    ssid: "Fallback-AP"
    password: "12345678"
  # No public OTA access

# Use strong passwords
ota:
  - platform: esphome
    password: "use-32-character-random-password-here"

# Enable API encryption
api:
  encryption:
    key: "base64-encoded-32-byte-key-here"
```

### Network Isolation

For production deployments:

1. **VLAN isolation** - Separate IoT network
2. **Firewall rules** - Restrict port 3232 access
3. **VPN access** - OTA only through VPN
4. **Certificate pinning** - Future ESPHome feature

## Comparison: OTA vs USB Flashing

| Feature | OTA | USB |
|---------|-----|-----|
| **Physical access** | Not required | Required |
| **Speed** | ~5 seconds | ~15 seconds |
| **Reliability** | 98% (network dependent) | 99.9% |
| **Recovery** | Safe mode available | Manual recovery |
| **Convenience** | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **Initial setup** | Requires USB first | Direct |
| **Remote devices** | Perfect | Impossible |

**Use OTA for:**
- Regular updates
- Remote devices
- Production deployments
- Multiple devices

**Use USB for:**
- Initial flash
- Bricked devices
- Changed OTA password
- Network issues

## Summary

### Quick OTA Flash

```bash
# 1. Activate environment
source esphome-env/bin/activate

# 2. Upload firmware
esphome upload your-config.yaml --device 192.168.1.X

# 3. Verify
esphome logs your-config.yaml --device 192.168.1.X
```

### Key Points

- ✅ OTA requires device already has ESPHome with OTA enabled
- ✅ Device must be on network and reachable
- ✅ Upload takes ~5 seconds for ~1MB firmware
- ✅ Password authentication protects against unauthorized updates
- ✅ Safe mode provides automatic recovery
- ✅ Much more convenient than USB for updates

### Prerequisites Summary

**Before OTA:**
1. Device flashed via USB with OTA-enabled firmware
2. Device connected to WiFi
3. Know device IP address or hostname
4. Have OTA password
5. Stable network connection

**For successful OTA:**
- Strong WiFi signal (-60 dBm or better)
- Stable power supply
- Valid firmware binary
- Correct password
- Device not in use during update

## References

- **ESPHome OTA Documentation:** https://esphome.io/components/ota.html
- **WiFi Fix Documentation:** See `WIFI_FIX_DOCUMENTATION.md`
- **Firmware Build Guide:** See `FIRMWARE_BUILD_DOCUMENTATION.md`
