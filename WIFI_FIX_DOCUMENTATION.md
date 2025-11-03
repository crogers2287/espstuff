# ESP32-C3 WiFi Connection Issues - Complete Fix Documentation

## Problem Overview

The ESP32-C3 Super Mini was experiencing persistent WiFi authentication failures when attempting to connect to the "AhoySmartHome" network. The device would repeatedly show "Auth expired" and "Authentication failed" errors despite correct credentials.

## Symptoms

```
[W][wifi:689]: Event: Disconnected ssid='AhoySmartHome' reason='Auth Expired'
[W][wifi:689]: Event: Disconnected ssid='AhoySmartHome' reason='Authentication Failed'
[W][wifi_esp32:655]: WiFi Association failed! Reconnecting...
```

The device would continuously attempt to reconnect but never successfully authenticate.

## Root Cause

**PMF (Protected Management Frames)** setting on the router was incompatible with the ESP32-C3 configuration.

- Router had PMF set to "Capable" (optional PMF support)
- ESP32-C3 running ESPHome does not fully support PMF in all configurations
- This caused authentication handshake failures even with correct WPA2-PSK credentials

## Attempted Solutions (That Didn't Work)

We tried multiple configuration changes that did **NOT** resolve the issue:

### 1. Power Save Mode Adjustments
```yaml
wifi:
  power_save_mode: none  # Disabled power saving
```
**Result:** No improvement

### 2. Output Power Adjustments
```yaml
wifi:
  output_power: 20dB  # Maximum transmission power
```
**Result:** No improvement

### 3. Static IP Configuration
```yaml
wifi:
  manual_ip:
    static_ip: 192.168.86.200
    gateway: 192.168.86.1
    subnet: 255.255.255.0
```
**Result:** No improvement - still failed at authentication stage

### 4. Fast Connect Optimization
```yaml
wifi:
  fast_connect: true  # Skip network scan
```
**Result:** No improvement

### 5. Board Type Changes
Tried different ESP32-C3 board configurations:
- `seeed_xiao_esp32c3`
- `esp32-c3-devkitm-1`

**Result:** No improvement - issue was network-side

## The Solution

### Router Configuration Change

Created a new dedicated IoT network with **PMF disabled**:

**Network Settings:**
- **SSID:** AhoyIOT
- **Password:** 106granada
- **Security:** WPA2-PSK (AES) only
- **PMF:** Disabled (critical setting)
- **Band:** 2.4GHz (ESP32-C3 does not support 5GHz)

### ESPHome Configuration

```yaml
wifi:
  ssid: AhoyIOT
  password: 106granada

  fast_connect: true
  reboot_timeout: 15min

  # Fallback access point
  ap:
    ssid: "ESP32C3-Smoke-Sensor"
    password: "smokedetector123"
```

## Why This Worked

1. **PMF Incompatibility:** ESP32-C3 with ESPHome has limited or no support for Protected Management Frames
2. **WPA2-PSK (AES):** Standard encryption works perfectly without PMF
3. **Dedicated IoT Network:** Isolates IoT devices from PMF requirements of modern WiFi 6 routers
4. **2.4GHz Band:** ESP32-C3 only supports 2.4GHz, ensuring proper frequency selection

## Verification

After implementing the fix:

```
[I][wifi:600]: WiFi Connected!
[I][wifi:603]: SSID: 'AhoyIOT'
[I][wifi:604]: IP Address: 192.168.1.197 (DHCP assigned)
[I][wifi:606]: BSSID: [MAC Address]
[I][wifi:607]: Hostname: 'esp32c3-smoke-sensor'
[I][wifi:612]: Signal strength: -52 dBm
```

Connection was stable and persistent with no further authentication issues.

## Best Practices for ESP32 WiFi Networks

### Recommended Router Settings for ESP32 Devices

1. **Create Dedicated IoT Network:**
   - Separate SSID from main network
   - 2.4GHz band only
   - WPA2-PSK (AES) encryption
   - **PMF: Disabled or Not Required**

2. **Network Configuration:**
   - DHCP enabled (unless static IPs required)
   - mDNS/Bonjour enabled for hostname resolution
   - Allow broadcast/multicast for device discovery

3. **Security Considerations:**
   - Isolate IoT VLAN from main network if possible
   - Use firewall rules to restrict IoT device access
   - Regular password rotation
   - Monitor for unusual traffic patterns

## Troubleshooting Guide

If you encounter similar WiFi issues:

### 1. Check PMF Setting First
```
Router Settings → Wireless Security → PMF (Protected Management Frames)
```
- Set to "Disabled" or "Not Required" for IoT devices
- This is the #1 cause of ESP32 authentication failures

### 2. Verify Credentials
```yaml
wifi:
  ssid: "YourSSID"  # Must match exactly (case-sensitive)
  password: "YourPassword"  # Must match exactly
```

### 3. Check Router Logs
- Look for authentication failures
- Check for MAC address blocks
- Verify device is not blacklisted

### 4. Test with Minimal Configuration
```yaml
wifi:
  ssid: YourSSID
  password: YourPassword
```
Add features incrementally after basic connection works.

### 5. Monitor ESPHome Logs
```bash
esphome logs esp32-c3-supermini-smoke-fan.yaml
```
Watch for specific error messages to identify the issue.

## Hardware-Specific Notes

### ESP32-C3 Super Mini
- Supports 2.4GHz WiFi only (802.11 b/g/n)
- Integrated PCB antenna
- Typical range: 30-50 meters indoors
- Power consumption: ~80mA active WiFi

### ESP32-C6 (for reference)
- Also experienced similar PMF issues
- Has BLE 5.0 support (C3 has BLE 5.0 as well)
- Same WiFi configuration requirements

## Configuration Files

### Working ESP32-C3 Configuration
See `esp32-c3-supermini-smoke-fan.yaml:32-42` for complete WiFi configuration.

### Router Settings Screenshot
Original troubleshooting included router configuration screenshot showing PMF setting change.

## Related Issues

- ESPHome GitHub Issue: ESP32 PMF support
- ESP-IDF WiFi authentication issues with WPA2/WPA3 transition mode
- Router firmware updates changing default PMF behavior

## Summary

**Problem:** ESP32-C3 couldn't connect to WiFi network
**Root Cause:** PMF (Protected Management Frames) incompatibility
**Solution:** Create dedicated IoT network with PMF disabled
**Result:** Stable, reliable WiFi connection with DHCP

This fix applies to most ESP32 variants (ESP32, ESP32-C3, ESP32-C6, ESP32-S3) experiencing similar authentication issues.
