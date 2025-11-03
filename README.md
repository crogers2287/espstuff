# ESPHome Smoke Sensor Configurations

This repository contains ESPHome configurations for smoke detection systems using MQ2 sensors.

## Hardware

- **ESP32-C3 Super Mini** - Working configuration with analog ADC support
- **ESP32-C6** - Experimental configurations (ADC support may be limited)
- **MQ2 Smoke/Gas Sensor** - Analog and digital outputs

## Configurations

### Working Configurations

- **esp32-c3-supermini-smoke-fan.yaml** - Main working config with:
  - Analog ADC sensor on GPIO4
  - Digital sensor on GPIO3
  - 0.8V smoke detection threshold
  - WiFi connectivity to "AhoyIOT" network
  - Home Assistant integration

### Experimental Configurations

- **esp32-c6-smoke-sensor.yaml** - Binary sensor only (no ADC)
- **esp32-c6-smoke-sensor-analog.yaml** - ADC attempt (may have compilation issues)
- **esp32-c6-smoke-sensor-resistance.yaml** - Resistance sensor approach

## Wiring

### MQ2 to ESP32-C3 Super Mini

```
MQ2 A0  → GPIO4 (Analog)
MQ2 D0  → GPIO3 (Digital)
MQ2 VCC → 5V
MQ2 GND → GND
```

## Building Firmware

### Option 1: GitHub Actions (Automated)

The repository includes GitHub Actions that automatically build firmware binaries on every push.

1. Push your code to GitHub
2. Go to the "Actions" tab
3. Wait for the build to complete
4. Download the firmware artifacts from the workflow run

Firmware files are available as:
- Individual config artifacts (30-day retention)
- Combined "all-firmware-builds" artifact (90-day retention)

### Option 2: Local Build

```bash
# Install ESPHome
pip install esphome

# Compile firmware
esphome compile esp32-c3-supermini-smoke-fan.yaml

# Flash over USB
esphome run esp32-c3-supermini-smoke-fan.yaml --device /dev/ttyUSB0

# Flash over WiFi (OTA)
esphome run esp32-c3-supermini-smoke-fan.yaml
```

## Setup

1. Copy `secrets.yaml.example` to `secrets.yaml`
2. Fill in your WiFi credentials and encryption keys
3. Flash the firmware to your ESP32 device
4. Wait 60 seconds for the MQ2 sensor to warm up
5. Add the device to Home Assistant

## Home Assistant Integration

The sensor exposes these entities:

- **MQ2 Smoke Level** - Analog voltage reading (0-3.3V)
- **MQ2 Smoke Detected (Digital)** - Hardware digital output
- **Smoke Detected (Threshold)** - Triggers when voltage > 0.8V
- **Smoke Sensor Uptime** - Device uptime
- **Smoke Sensor WiFi Signal** - WiFi signal strength

## Automation Example

Create an automation in Home Assistant to trigger a fan when smoke is detected:

```yaml
alias: Smoking Room Fan Control
trigger:
  - platform: state
    entity_id: binary_sensor.smoke_detected_threshold
    to: "on"
action:
  - service: switch.turn_on
    target:
      entity_id: switch.your_wifi_fan_switch
```

## Troubleshooting

### WiFi Connection Issues

If you experience WiFi authentication failures:
- Ensure PMF (Protected Management Frames) is disabled on your router
- Use WPA2-PSK (AES) authentication
- Consider creating a dedicated 2.4GHz IoT network

### ADC Not Working

- ESP32-C6 ADC support in ESPHome may be limited with ESP-IDF framework
- Use ESP32-C3 for reliable ADC functionality

### Sensor Always Reading ON

- Check that you're using the analog output (A0), not just digital (D0)
- Adjust the potentiometer on the MQ2 module
- Monitor the "MQ2 Smoke Level" sensor to see actual voltage readings

## Known Issues

- **ESP32-C6 ADC**: ESPHome 2025.7.3 has broken ADC support for ESP32-C6 with ESP-IDF framework
- **PMF Compatibility**: Some routers with PMF enabled may cause WiFi connection issues

## License

This configuration is provided as-is for personal use.
