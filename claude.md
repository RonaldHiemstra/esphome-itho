# Project Context for Claude

## Project Overview

This is an ESPHome-based project for monitoring and controlling an ITHO mechanical ventilation system using an ESP32 microcontroller with a CC1101 868MHz RF transceiver. The system acts as an additional remote control and status monitor, integrating with Home Assistant via MQTT.

## Hardware Setup

- **ESP32 DevKit** (ESP-WROOM-32)
- **CC1101 868MHz RF module** connected via SPI
- **Target device:** ITHO Daalderop CVE-S ECO ventilation unit
- **Compatible with:** ITHO RFT bedieningsschakelaar (RF remote control switch)

### Pin Connections

- SCK: GPIO18
- MISO: GPIO19
- MOSI: GPIO23
- CS: GPIO22
- GDO0: GPIO16

## Project Files

- **itho-ventilation.yaml** - Main ESPHome configuration file
- **readme.md** - User documentation and project overview
- **.gitattributes** - Git configuration (enforces LF line endings)

## Key Technical Details

### RF Protocol (868.2999 MHz)

- **Modulation:** 2-FSK
- **Symbol rate:** 38.383 kBaud
- **Sync word:** 0xB3 0x2A
- **Packet length:** 63 bytes raw → 24 bytes decoded
- **Encoding:** Custom Manchester-like encoding (NOT standard Manchester)
- **Preamble:** 7 bytes (6 auto + 1 manual)

### Custom Encoding Details

**CRITICAL:** The ITHO protocol uses a **custom Manchester-like encoding** that differs from standard Manchester. Do NOT enable CC1101's built-in `manchester: true` setting. The decoder:

1. Starts from STARTBYTE=2 (skips first 2 bytes after sync)
2. Extracts even bits (0, 2, 4, 6) from each 10-bit group
3. Inserts 1-0 pattern every 8 bits during encoding

### Packet Types

**Remote Commands** (decoded bytes 5-10):

- Low: `22 F1 03 00 02 04`
- Medium: `22 F1 03 00 03 04`
- High: `22 F1 03 00 04 04`
- Timer: `22 F3 03 00 00 0A/14/1E`

**Status Broadcasts** (from ventilation unit):

- Format: `1A.XX.XX.XX.XX.XX.XX.XX.31.D9.11.00.06.YY...`
- Byte 0: `0x1A` (status identifier)
- Byte 12: `0x06` (status type)
- Byte 13: Speed indicator (0x30-0x37=Low, 0x38-0x7F=Medium, 0x80+=High)

### Security Feature: Device Whitelist

The configuration includes TWO separate whitelists:

1. **Ventilation unit whitelist** - For status messages (byte[0]==0x1A)
2. **Remote control whitelist** - For command packets (byte[5]==0x22)

Packets from unknown devices are logged and rejected. Device IDs are 3-byte values extracted from decoded packets.

## Code Architecture

### Main Components

1. **CC1101 Configuration** - RF transceiver setup with ITHO-specific parameters
2. **Packet Decoder Lambda** - Processes incoming RF packets and extracts commands
3. **Command Encoder Scripts** - Builds and transmits RF packets
4. **Sensors & Text Sensors** - Expose state to Home Assistant
5. **Control Buttons** - User interface for sending commands

### Important Lambdas

- **on_boot:** Extracts device MAC address for RF transmission
- **on_packet:** Decodes received packets and updates sensors
- **send_command:** Encodes and transmits ventilation commands (Low/Medium/High)
- **send_join_command:** Pairing sequence (currently not working - needs debugging)

## Known Issues & Limitations

1. **Pairing not functional** - Join command sequence doesn't successfully register with the unit
2. **Preamble optimization** - Could simplify to `num_preamble: 4` once transmission is confirmed working
3. **Timer command not implemented** - Only Low/Medium/High commands supported
4. **Manual whitelist management** - Device IDs must be added manually to code

## Development Guidelines

### When Modifying RF Protocol

- Always test with actual hardware - RF timing is critical
- Check ESPEasy IthoCC1101.cpp for reference implementation
- Maintain exact byte patterns for command arrays
- Preserve Manchester encoding logic exactly as implemented

### When Adding Features

- Update both YAML configuration and readme.md documentation
- Add new device IDs to appropriate whitelist
- Test MQTT integration separately from Home Assistant
- Log extensively for debugging RF issues

### When Debugging

- Check ESP32 logs for "rssi", "lqi", and "Decoded" messages
- Unknown device warnings indicate whitelist issues
- Packet counter increments confirm transmission attempts
- Use `format_hex_pretty()` for readable byte arrays

## Integration Points

- **MQTT broker** - Primary communication with Home Assistant
- **WiFi credentials** - Stored in ESPHome secrets
- **OTA updates** - Enabled with password protection
- **Web server** - Available on port 80 for diagnostics

## References

- [ESPHome CC1101 Component](https://esphome.io/components/cc1101/)
- [ESPEasy IthoCC1101 Library](https://github.com/letscontrolit/ESPEasy/blob/mega/lib/Itho/IthoCC1101.cpp)

## Quick Start for AI Assistance

When helping with this project:

1. Understand the custom Manchester encoding - it's not standard
2. Check both whitelists when modifying device acceptance logic
3. Maintain exact byte patterns for RF commands
4. Test RF changes with physical hardware
5. Update documentation when making significant changes
6. Consider MQTT standalone operation (independent of Home Assistant)
