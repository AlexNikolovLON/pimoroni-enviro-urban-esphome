# Pimoroni Enviro Urban with ESPHome and Home Assistant

A working ESPHome configuration for the **Pimoroni Enviro Urban (PIM629)** based on the Raspberry Pi Pico W / RP2040.

This project came out of a lot of debugging around the Plantower particulate sensor, the second I2C bus, microphone sampling, RTC behaviour, temperature self-heating and Home Assistant entity availability. The aim is to save other users the same trouble.

## What works

- Raspberry Pi Pico W support in ESPHome
- BME280 temperature, pressure and humidity
- Plantower PM1.0 / PM2.5 / PM10
- Particle counts for >0.3, >0.5, >1.0, >2.5, >5 and >10 um
- Microphone level sampling
- PCF85063A RTC handling
- POKE button
- Activity LED
- Wi-Fi signal, uptime and network information
- Home Assistant control for PM measurement interval
- Home Assistant control for temperature calibration offset
- Manual PM reading button

## Tested with

- Pimoroni Enviro Urban PIM629
- Raspberry Pi Pico W
- ESPHome 2026.8.2
- Home Assistant via the native ESPHome API

## Important findings

### Plantower PM sensor startup

The PM sensor is normally powered down. Its dedicated I2C bus may therefore report a boot-time recovery warning such as `SCL is held low`. In this configuration that is expected while the PM sensor is off.

Reliable PM readings were obtained by:

1. enabling the 5 V boost supply,
2. enabling the PM sensor,
3. waiting **15 seconds**,
4. reading and discarding the first 32-byte frame,
5. waiting 2 seconds,
6. reading, validating and publishing the second frame.

A working second frame looked like:

```text
42 4D 00 1C 00 02 00 0A 00 0A 00 02 00 0A 00 0A
02 46 00 AF 00 31 00 16 00 00 00 00 97 00 02 AC
```

That produced:

```text
PM1.0   2 ug/m3
PM2.5  10 ug/m3
PM10   10 ug/m3

>0.3 um  5820 /L
>0.5 um  1750 /L
>1.0 um   490 /L
>2.5 um   220 /L
>5.0 um     0 /L
>10  um     0 /L
```

### Plantower frame parsing

For the 32-byte frame used here:

- bytes 0-1: header `42 4D`
- bytes 2-3: frame length, expected `28`
- bytes 4-9: CF=1 PM1.0 / PM2.5 / PM10
- bytes 10-15: atmospheric PM1.0 / PM2.5 / PM10
- bytes 16-27: particle counts
- bytes 30-31: checksum

The particle count values are reported per 0.1 L, so this configuration multiplies them by 10 before publishing `/L` values to Home Assistant.

### Microphone and PM fan

The PM fan affects the microphone measurement. Microphone sampling is therefore suspended while the PM sensor is active and resumes after the fan is switched off.

### Temperature offset

The BME280 on this board can read significantly higher than ambient due to local board heating. In the test unit the raw temperature was around 29 C in a room near 21 C.

This configuration exposes **Enviro Temperature Offset** as a Home Assistant number entity. The default is `-8.0 C`, but every board and enclosure will differ, so calibrate it against a trusted room thermometer.

The BME280 oversampling is set to `1x` to reduce unnecessary self-heating.

## Installation

1. Copy `ha-eviro-urban.yaml` into your ESPHome configuration directory.
2. Copy `secrets.example.yaml` to `secrets.yaml` if you do not already have one.
3. Fill in your Wi-Fi credentials, ESPHome API encryption key and fallback hotspot password.
4. Compile and install with ESPHome.
5. Add the device to Home Assistant through the ESPHome integration.

## Secrets

Do not commit your real `secrets.yaml` file. This repository includes only `secrets.example.yaml`.

Generate an ESPHome API key in the normal ESPHome/Home Assistant workflow and place it in your local secrets file.

## Notes

This is a community configuration, not an official Pimoroni or ESPHome project. Hardware revisions may differ, so verify pin assignments before applying the configuration to another board revision.

See `docs/troubleshooting.md` for the main problems encountered during development.

## License

MIT — see `LICENSE`.
