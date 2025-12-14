# HDMI Communication on Raspberry Pi 5: EDID, CEC, and Custom Device Names

## Overview

When you plug an HDMI cable between devices, there's more happening than just video transmission. This guide explores the data exchange over HDMI connections, how to query it on Raspberry Pi 5, and how to set custom device names that appear on your TV's input menu.

## HDMI Communication Fundamentals

### EDID: Extended Display Identification Data

EDID is a **one-way communication protocol** where the display device sends its capabilities to the source device over an I2C bus built into the HDMI cable (pins 15/16 - SDA/SCL).

**Direction of communication:**
```
Display (TV/Monitor) ---EDID---> Source (Raspberry Pi)
                                 No data sent back
```

The display continuously presents EDID data at I2C address 0x50. When the source detects the HPD (Hot Plug Detect) signal, it reads this data to configure optimal output settings.

**EDID contains:**
- Display name (manufacturer and model)
- Supported resolutions and refresh rates
- Color depths and formats (RGB, YCbCr)
- Physical dimensions
- Audio capabilities (sample rates, formats, channels)
- HDR support (HDR10, Dolby Vision)
- Timing parameters and sync polarities

**Key point:** The display never knows what's plugged into it. Your TV has no idea it's connected to a Raspberry Pi versus a Chromecast versus a gaming console - there's no reverse identification in the EDID protocol.

### CEC: Consumer Electronics Control

CEC provides **bidirectional control** over the same HDMI cable, allowing devices to:
- Power on/off
- Change input sources
- Adjust volume
- Send playback commands
- **Broadcast device names** (OSD strings)

This is how devices like Chromecast show custom names in your TV's input menu.

## Querying HDMI/EDID on Raspberry Pi 5

Raspberry Pi 5 uses a new display architecture. The old `tvservice` command from Pi 4 is deprecated.

### Display Information Commands

**Quick display info:**
```bash
kmsprint
```

**Detailed EDID information:**
```bash
kmsprint -m
```

**Install additional tools:**
```bash
sudo apt install edid-decode drm-info
```

**Read and decode EDID:**
```bash
# Direct EDID read from sysfs
cat /sys/class/drm/card*/card*-HDMI-*/edid | edid-decode
```

**Check all connected HDMI ports:**
```bash
for edid in /sys/class/drm/card*/card*-HDMI-*/edid; do
    echo "Port: $edid"
    edid-decode < "$edid" | grep -i "display product name"
done
```

### Example Output

When you run `edid-decode`, you'll see detailed information like:

```
Manufacturer: SAM
Model: 2841
Serial Number: 1234567
Display Product Name: Samsung QN55
...
Supported Resolutions:
  1920x1080 @ 60Hz
  3840x2160 @ 30Hz
Audio Capabilities:
  PCM 2-channel, 48kHz
  AC3 5.1-channel
```

## Understanding CEC Devices

### Install CEC Tools

```bash
sudo apt install cec-utils
```

### Scan the CEC Bus

```bash
echo 'scan' | cec-client -s -d 1
```

**Example output:**
```
CEC bus information
===================
device #0: TV
address:       0.0.0.0
active source: no
vendor:        Samsung
osd string:    TV
CEC version:   1.4
power status:  on

device #1: Recorder 1
address:       3.0.0.0
active source: no
vendor:        Pulse Eight
osd string:    CECTester
CEC version:   1.4
power status:  on

device #4: Playback 1
address:       1.0.0.0
active source: no
vendor:        Google
osd string:    Chromecast
CEC version:   2.0
power status:  standby
```

Notice how Chromecast shows a custom "osd string" while the Raspberry Pi defaults to "CECTester". This is what appears in your TV's input menu.

## Setting a Custom Device Name

### Why "CECTester" or "Videocore" Appears

By default, Raspberry Pi uses the Pulse Eight libcec library, which broadcasts "CECTester" as the device name. Some TVs may show "Videocore" based on GPU identification attempts.

### Change Device Name Temporarily

**Method 1: Direct command**
```bash
echo 'osd_name YourDeviceName' | cec-client -s -d 1
```

**Method 2: Using hex encoding**
For a device name like "KlapperAI":
```bash
echo 'tx 10:47:4B:6C:61:70:70:65:72:41:49' | cec-client -s -d 1
```

The hex values are ASCII codes for each character preceded by the CEC "Set OSD Name" command (0x47).

### Make Device Name Persistent

Create or edit the CEC client configuration:

```bash
sudo nano /etc/cec-client.conf
```

Add these lines:
```
device_name KlapperAI
device_type 1
```

**Device types:**
- 0 = TV
- 1 = Recording device
- 3 = Tuner
- 4 = Playback device
- 5 = Audio system

Type 1 (Recording device) works well for Raspberry Pi projects.

### Verify the Change

```bash
echo 'scan' | cec-client -s -d 1
```

Look for your device in the output - the `osd string` field should now show your custom name:

```
device #1: Recorder 1
address:       3.0.0.0
vendor:        Pulse Eight
osd string:    KlapperAI        <-- Your custom name
CEC version:   1.4
```

**Check your TV:** Navigate to the input menu on your TV. Instead of "HDMI 1" or "Videocore", you should see "KlapperAI" (or whatever name you chose).

## Practical Applications

### Multi-Pi Identification

When running multiple Raspberry Pi devices on different HDMI inputs:
```bash
# Living Room Pi
device_name LivingRoom-Pi

# Kitchen Pi  
device_name Kitchen-Pi

# Garage Pi
device_name Garage-Pi
```

Each will appear with its custom name in the TV's input menu, making switching between them trivial.

### Integration with Home Automation

Combine CEC control with MQTT/Node-RED for automated scenarios:

**Power on TV when Pi boots:**
```bash
echo 'on 0' | cec-client -s -d 1
```

**Switch to this input:**
```bash
echo 'as' | cec-client -s -d 1
```

**Power off TV:**
```bash
echo 'standby 0' | cec-client -s -d 1
```

### Kiosk Display Management

For Raspberry Pi kiosk displays showing Node-RED dashboards or Unity window projects:
- Custom device names help identify which Pi controls which display
- CEC commands can power manage displays during off-hours
- Input switching can be automated based on schedules or MQTT triggers

## Quick Reference

### Essential Commands

```bash
# Query display EDID
edid-decode < /sys/class/drm/card1/card1-HDMI-A-1/edid

# Scan CEC devices
echo 'scan' | cec-client -s -d 1

# Set device name (temporary)
echo 'osd_name DeviceName' | cec-client -s -d 1

# Power TV on
echo 'on 0' | cec-client -s -d 1

# Power TV off
echo 'standby 0' | cec-client -s -d 1

# Make this Pi the active source
echo 'as' | cec-client -s -d 1
```

### Configuration Files

**CEC persistent config:**
`/etc/cec-client.conf`

**Boot config (Pi firmware):**
`/boot/firmware/config.txt`

## Troubleshooting

### CEC Not Working

1. **Enable CEC on your TV** - Often called "Anynet+" (Samsung), "Bravia Sync" (Sony), "SimpLink" (LG), or "HDMI-CEC"
2. **Check physical connection** - Some cheap HDMI cables don't wire the CEC pins
3. **Verify cec-utils installation** - `cec-client --version`
4. **Try different HDMI port** - Not all TV ports support CEC equally

### Name Not Appearing

1. **Reboot the TV** - Some TVs cache device names
2. **Unplug/replug HDMI** - Forces re-detection
3. **Check config file syntax** - Ensure no typos in `/etc/cec-client.conf`
4. **Verify with scan** - Confirm the osd string changed before checking TV

### Permission Issues

If you get permission errors:
```bash
# Add your user to video group
sudo usermod -a -G video $USER

# Log out and back in for group change to take effect
```

## Conclusion

HDMI carries far more than just video and audio. Understanding EDID and CEC protocols allows you to:
- Query display capabilities programmatically
- Identify devices by custom names
- Automate display control
- Build sophisticated multi-display systems

For Raspberry Pi projects, especially kiosk displays and home automation installations, these capabilities transform HDMI from a simple video cable into a bidirectional control and identification system.

---

**Hardware Used:**
- Raspberry Pi 5
- Various HDMI displays (Samsung TV tested)
- Standard HDMI cables

**Software Requirements:**
- Raspberry Pi OS (Bookworm or later)
- cec-utils package
- edid-decode package
- drm-info package (optional)

**Repository:** https://github.com/RichFesler/HDMI-CEC-RaspberryPi5

**License:** MIT

**Contributions:** Issues and pull requests welcome
