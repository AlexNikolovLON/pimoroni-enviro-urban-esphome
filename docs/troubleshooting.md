# Troubleshooting

This file records the main problems encountered while bringing the Pimoroni Enviro Urban PIM629 into ESPHome/Home Assistant.

## PM I2C bus reports SCL held low at boot

The Plantower PM sensor is normally powered down. Because its dedicated I2C bus is initialised while the sensor is off, ESPHome may log a warning such as:

```text
Recovery: failed, SCL is held low on the bus
```

If valid PM frames are later read after the sensor is powered, this warning can be treated as a startup condition rather than a fatal bus failure.

## Main I2C bus recovery warning

The main bus may also briefly report SDA held low during startup while still discovering the RTC at `0x51` and BME280 at `0x77`. If both devices scan and work normally afterwards, the warning is not necessarily fatal.

## Particle counts are zero but PM mass values are non-zero

Short warm-up times produced valid PM mass readings while all particle-count fields remained zero.

The reliable sequence used here is:

1. enable 5 V boost,
2. wait 500 ms,
3. enable PM sensor,
4. wait 15 seconds,
5. read and discard one 32-byte frame,
6. wait 2 seconds,
7. read and publish the second frame.

This changed the result from zero particle counts to valid measurements.

## Plantower frame parsing

The second frame is validated before publishing:

- header must be `0x42 0x4D`
- frame length must be 28
- checksum is the sum of bytes 0 through 29
- checksum must match bytes 30-31

Published atmospheric PM values are taken from bytes 10-15. Particle counts come from bytes 16-27.

The count fields are per 0.1 litre and are multiplied by 10 before publishing as particles per litre.

## PM fan contaminates microphone readings

The PM fan is close enough to the onboard microphone to affect noise measurements. The configuration therefore stops microphone sampling while a PM cycle is active and waits briefly after sensor power-off before resuming.

## BME280 temperature reads too high

The onboard BME280 can read substantially above room temperature because of local board/self-heating. On the test board it reported around 29 C in a room around 21 C.

The configuration exposes `Enviro Temperature Offset` in Home Assistant so the correction can be tuned without recompiling. The example default is -8.0 C.

BME280 oversampling is set to 1x to avoid unnecessary heating.

## Home Assistant entities appear unavailable

An earlier configuration structure caused entities to appear unavailable in the normal Home Assistant UI even though the ESPHome API connection and Home Assistant state machine were receiving live values.

The current configuration structure resolved that issue. If migrating from an earlier experimental configuration, remove or rename obsolete entities as needed, then use the current YAML as the baseline rather than mixing sections from older versions.

## Useful expected log sequence

A successful PM cycle should look broadly like:

```text
PM measurement started - microphone suspended
PM sensor fan started
FIRST DISCARD: 42 4D ...
RAW SECOND: 42 4D ...
PM atmospheric: ...
PM CF1: ...
Particles/L: ...
PM sensor powered OFF
PM measurement finished - microphone resumed
```

A known-good frame observed during testing was:

```text
42 4D 00 1C 00 02 00 0A 00 0A 00 02 00 0A 00 0A
02 46 00 AF 00 31 00 16 00 00 00 00 97 00 02 AC
```

which decoded to:

```text
PM1.0    2 ug/m3
PM2.5   10 ug/m3
PM10    10 ug/m3
>0.3 um 5820 /L
>0.5 um 1750 /L
>1.0 um  490 /L
>2.5 um  220 /L
>5.0 um    0 /L
>10 um     0 /L
```
