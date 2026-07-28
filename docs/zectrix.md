# Zectrix Note 4 / Note 4C

This branch adds initial TRMNL firmware support for the ESP32-S3 N16R8
versions of the Zectrix Note 4 and Note 4C.

The two targets compile successfully, but the display and deep-sleep paths
still need validation on physical devices. Back up the complete factory flash
before installing either target.

## Build targets

| Device | PlatformIO environment | TRMNL model | Display profile |
| --- | --- | --- | --- |
| Note 4 | `zectrix_note4` | `zectrix_note4` | `EP42B_400x300` |
| Note 4C | `zectrix_note4c` | `zectrix_note4c` | `EP42YR_400x300` |

Build either image from the repository root:

```sh
pio run -e zectrix_note4
pio run -e zectrix_note4c
```

Each build produces both a normal application image and a complete image that
starts at flash offset `0x0`:

```text
.pio/build/zectrix_note4/firmware.bin
.pio/build/zectrix_note4/merged_firmware.bin
.pio/build/zectrix_note4c/firmware.bin
.pio/build/zectrix_note4c/merged_firmware.bin
```

## Pin mapping

| Function | GPIO | Notes |
| --- | ---: | --- |
| E-paper power | 6 | Active HIGH; held LOW in deep sleep |
| E-paper busy | 8 | |
| E-paper reset | 9 | |
| E-paper D/C | 10 | |
| E-paper CS | 11 | |
| E-paper SCK | 12 | |
| E-paper MOSI | 13 | |
| Board power hold | 17 | Must remain HIGH, including during deep sleep |
| Home / OK button | 0 | TRMNL action button and deep-sleep wake source |
| Battery ADC | 4 | Firmware applies the board's 2:1 divider |
| Charging | 2 | Active LOW |
| Charge complete | 1 | Active HIGH |

GPIO17 is safety-critical. If it is released or driven LOW after the physical
power button is released, battery power to the ESP32-S3 is cut. The Zectrix
startup code asserts it before the rest of firmware setup and holds it across
deep sleep.

## Back up and flash

Put the device into its ESP32-S3 ROM download mode and identify its serial
port. Before the first test, use an installed `esptool` to read all 16 MB:

```sh
esptool --chip esp32s3 --port PORT read-flash \
  0x0 0x1000000 zectrix-original-16mb.bin
```

For development, let PlatformIO select all required offsets:

```sh
pio run -e zectrix_note4 -t upload --upload-port PORT
```

Use `zectrix_note4c` instead for a Note 4C.

The generated merged image can also be written at offset `0x0`:

```sh
esptool --chip esp32s3 --port PORT write-flash \
  0x0 .pio/build/zectrix_note4/merged_firmware.bin
```

Keep the physical power button pressed until the new firmware starts on the
first boot. If the device powers off unexpectedly, reconnect USB or hold the
power button, enter ROM download mode again, and restore the 16 MB backup.

## First hardware validation

Test one function at a time:

1. Open a serial monitor at 115200 baud and confirm the firmware reaches
   `bl_init()` without repeatedly rebooting.
2. Confirm the e-paper power rail switches on and the first screen refresh
   completes without a busy timeout.
3. Complete the TRMNL Wi-Fi captive portal and request a 400x300 custom-screen
   PNG.
4. Confirm battery voltage and USB/charging state in the device logs.
5. Let the device enter deep sleep. Verify that it remains powered, wakes on
   its timer, and wakes when the Home / OK button on GPIO0 is pressed.
6. Measure deep-sleep current after the e-paper rail on GPIO6 is disabled.

Note 4C currently uses the four-color `EP42YR_400x300` profile provided by
`bb_epaper` 2.1.9. It compiles and exercises TRMNL's four-color PNG decoder,
but the controller/waveform must be confirmed against the panel fitted to the
actual device batch. A refresh taking more than ten seconds is normal for this
panel class. If the first refresh times out or produces incorrect colors,
record the panel label and serial log before trying another driver.

## Deliberate limitations

- Upstream TRMNL OTA is disabled for these targets. An official OG/X image
  could use incompatible pins and power behavior.
- Factory QA is disabled because its fixtures and pin assumptions are for
  official TRMNL hardware.
- Only the Home / OK button is mapped into the current single-button TRMNL
  interaction model. The remaining Note buttons can be added later.
- Some built-in fallback BMPs and status layouts still assume the upstream
  800x480 screen. Normal custom-screen PNG delivery uses the selected 400x300
  display profile.
