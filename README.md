# sdr-enthusiasts/airspy_adsb

Docker container running AirSpy's `airspy_adsb` receiver. Designed to work in tandem with other sdr-enthusiasts containers. Builds and runs on x86_64, arm64 and arm32v7.

`airspy_adsb` receives ADS-B data, and provides a BEAST socket for other containers to consume BEAST data.

It will provide BEAST protocol on TCP port `30005`.

## Environment Variables

| Environment Variable                 | `airspy_adsb`</br>option | Description                                                                                                                         | Default                                                |
| ------------------------------------ | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| `AIRSPY_ADSB_SERIAL`                 | `-s`                     | Device serial number                                                                                                                | _unset_                                                |
| `AIRSPY_ADSB_TIMEOUT`                | `-t`                     | Aircraft timeout in seconds                                                                                                         | `60`                                                   |
| `AIRSPY_ADSB_RF_GAIN`                | `-g`                     | RF gain: `0` to `21` or `auto`                                                                                                      | `auto`                                                 |
| `AIRSPY_ADSB_FEC_BITS`               | `-f`                     | Forward Error Correction (FEC) bits                                                                                                 | `1`                                                    |
| `AIRSPY_ADSB_PREAMBLE_FILTER`        | `-e`                     | Preamble filter: `1` to `60`                                                                                                        | `4`                                                    |
| `AIRSPY_ADSB_CPUTIME_TARGET`         | `-C`                     | CPU processing time target (percentage): `5` to `95`                                                                                | Disabled</br>_(adjusts preamble filter while running)_ |
| `AIRSPY_ADSB_PREAMBLE_FILTER_MAX`    | `-E`                     | Maximum preamble filter when using CPU target: `1` to `60`                                                                          | `60`                                                   |
| `AIRSPY_ADSB_PREAMBLE_FILTER_NONCRC` | `-P`                     | Non-CRC Preamble filter: `1` to `$AIRSPY_ADSB_PREAMBLE_FILTER`                                                                      | Disabled                                               |
| `AIRSPY_ADSB_WHITELIST_THRESHOLD`    | `-w`                     | Whitelist threshold: `1` to `20`                                                                                                    | `5`                                                    |
| `AIRSPY_ADSB_MLAT_FREQ`              | `-m`                     | MLAT frequency in MHz: `12`, `20` or `24` (Airspy R2 only)                                                                          | _unset_                                                |
| `AIRSPY_ADSB_VERBATIM_MODE`          | `-n`                     | Set to `true` to enable Verbatim mode                                                                                               | _unset_                                                |
| `AIRSPY_ADSB_DX_MODE`                | `-x`                     | Set to `true` to enable DX mode                                                                                                     | _unset_                                                |
| `AIRSPY_ADSB_REDUCE_IF_BW`           | `-r`                     | Set to `true` reduce the IF bandwidth to 4 MHz                                                                                      | _unset_                                                |
| `AIRSPY_ADSB_RSSI_MODE`              | `-R`                     | RSSI mode: `snr` (ref = 42 dB), `rms` (default: rms)                                                                                | `rms`                                                  |
| `AIRSPY_ADSB_IGNORE_DF_TYPES`        | `-D`                     | Ignore these DF types (comma separated list)                                                                                        | `24,25,26,27,28,29,30,31`                              |
| `AIRSPY_ADSB_BIAS_TEE`               | `-b`                     | Set to `true` to enable Bias-Tee                                                                                                    | _unset_                                                |
| `AIRSPY_ADSB_BIT_PACKING`            | `-p`                     | Set to `true` to enable Bit Packing                                                                                                 | _unset_                                                |
| `AIRSPY_ADSB_VERBOSE`                | `-v`                     | Enable Verbose mode                                                                                                                 | _unset_                                                |
| `AIRSPY_ADSB_STATS`                  | `-S`                     | Set to `true` to enable statistics in `/run/airspy_adsb` (this needs to be shared with a `tar1090` instance)                        | _unset_                                                |
| `AIRSPY_ADSB_ARCH`                   | N/A                      | Forces a specific architecture binary. Supports `arm64`, `armv7`, `arm`, `nehalem`, `x86_64` or `i386`. If unset, will auto-detect. | _unset_                                                |

## Using with `readsb`

```yaml
version: "3.8"

volumes:
  readsbpb_rrd:

services:
  airspy_adsb:
    image: ghcr.io/sdr-enthusiasts/airspy_adsb:latest
    tty: true
    container_name: airspy_adsb
    hostname: airspy_adsb
    restart: always
    device_cgroup_rules:
      - 'c 189:* rwm'
    environment:
      - AIRSPY_ADSB_VERBOSE=true
      - AIRSPY_ADSB_TIMEOUT=90
      - AIRSPY_ADSB_FEC_BITS=1
      - AIRSPY_ADSB_WHITELIST_THRESHOLD=5
      - AIRSPY_ADSB_PREAMBLE_FILTER_NONCRC=8
      - AIRSPY_ADSB_CPUTIME_TARGET=60
      - AIRSPY_ADSB_PREAMBLE_FILTER_MAX=20
      - AIRSPY_ADSB_MLAT_FREQ=12
      - AIRSPY_ADSB_RF_GAIN=auto
      - AIRSPY_ADSB_STATS=true
    volumes:
      - /dev:/dev:ro
      - /run/airspy_adsb:/run/airspy_adsb


  ultrafeeder:
    image: ghcr.io/sdr-enthusiasts/docker-adsb-ultrafeeder:latest
    tty: true
    container_name: ultrafeeder
    hostname: ultrafeeder
    restart: unless-stopped
    ports:
      - 8080:80
    environment:
      - LOGLEVEL=error
      - TZ=${FEEDER_TZ}
      - READSB_LAT=${FEEDER_LAT}
      - READSB_LON=${FEEDER_LONG}
      - READSB_ALT=${FEEDER_ALT_M}m
      - READSB_RX_LOCATION_ACCURACY=2
      - READSB_STATS_RANGE=true
      - READSB_NET_ENABLE=true
      - READSB_NET_ONLY=true
      - ULTRAFEEDER_CONFIG=
        adsb,dump978,30978,uat_in;
        adsb,feed.adsb.fi,30004,beast_reduce_plus_out;
        adsb,in.adsb.lol,30004,beast_reduce_plus_out;
        adsb,feed.airplanes.live,30004,beast_reduce_plus_out;
        adsb,feed.planespotters.net,30004,beast_reduce_plus_out;
        adsb,feed.theairtraffic.com,30004,beast_reduce_plus_out;
        adsb,data.avdelphi.com,24999,beast_reduce_plus_out;
        adsb,skyfeed.hpradar.com,30004,beast_reduce_plus_out;
        adsb,feed.radarplane.com,30001,beast_reduce_plus_out;
        adsb,dati.flyitalyadsb.com,4905,beast_reduce_plus_out;
        mlat,feed.adsb.fi,31090,39000;
        mlat,in.adsb.lol,31090,39001;
        mlat,feed.airplanes.live,31090,39002;
        mlat,mlat.planespotters.net,31090,39003;
        mlat,feed.theairtraffic.com,31090,39004;
        mlat,skyfeed.hpradar.com,31090,39005;
        mlat,feed.radarplane.com,31090,39006;
        mlat,dati.flyitalyadsb.com,30100,39007;
        mlathub,piaware,30105,beast_in;
        mlathub,rbfeeder,30105,beast_in;
        mlathub,radarvirtuel,30105,beast_in;
        mlathub,planewatch,30105,beast_in
      - UUID=${MULTIFEEDER_UUID}
      - MLAT_USER=${FEEDER_NAME}
      - UPDATE_TAR1090=true
      - TAR1090_DEFAULTCENTERLAT=${FEEDER_LAT}
      - TAR1090_DEFAULTCENTERLON=${FEEDER_LONG}
      - TAR1090_MESSAGERATEINTITLE=true
      - TAR1090_PAGETITLE=${FEEDER_NAME}
      - TAR1090_PLANECOUNTINTITLE=true
      - TAR1090_SITESHOW=true
      - TAR1090_RANGE_OUTLINE_COLORED_BY_ALTITUDE=true
      - TAR1090_RANGE_OUTLINE_WIDTH=2.0
      - TAR1090_RANGERINGSDISTANCES=50,100,150,200
      - TAR1090_USEROUTEAPI=true
      - GRAPHS1090_DARKMODE=true
      - ENABLE_AIRSPY=true
    volumes:
      - /opt/adsb/ultrafeeder/globe_history:/var/globe_history
      - /opt/adsb/ultrafeeder/graphs1090:/var/lib/collectd
      - /proc/diskstats:/proc/diskstats:ro
      - /run/airspy_adsb:/run/airspy_adsb
    tmpfs:
      - /run:exec,size=256M
      - /tmp:size=128M
      - /var/log:size=32M
```
