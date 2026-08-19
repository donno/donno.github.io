---
---
# Misc

Single line snippets that I didn't justify giving them their own page.

# Convert UTC time to Adealide time.

```
import datetime
import pytz
adelaide_tz = pytz.timezone('Australia/Adelaide')
datetime.datetime.fromisoformat('2022-08-06 05:10:11+00:00').astimezone(adelaide_tz)
```

# Import rootfs tarball to WSL.
In this example I downloaded the core rootfs for x86_64 for Chimera Linux from
https://repo.chimera-linux.org/live/latest

Next imported it into Windows Subsystem for Linux (WSL).
```
wsl --import Chimera  C:\ProgramData\WSLDistroStorage\Chimera D:\Downloads\2021_Downloads\chimera-linux-x86_64-ROOTFS-20240122-core.tar.gz
```

Use it:
```
wsl -d Chimera
# lsb_release
LSB Version:    1.0
Distributor ID: Chimera
Description:    Chimera Linux
Release:        rolling
Codename:       chimera
``` 

# Export container filesystem to tar.
```
podman pull busybox
podman run --name plain_busybox busybox
podman export plain_busybox --output busybox.tar
```
Using the import rootfs tarball above you can use that to go from a
Container Image to Container to tarball to WSL distribution.

# Convert OSM to Versatiles
My initial thoughts on Versatiles is it seems to be re-inventing the wheel
because they have some disagreement about the existing wheel.

For OpenMapTiles schema verse Shortbread it seems to come down the the former
having an Attribution clause as its licensed CC-BY where the latter is CC0.

```
podman run --rm --privileged -it -v G:\GeoData\Generated\OSM:/app/result ghcr.io/versatiles-org/versatiles-tilemaker:v0.7.3 generate_tiles.sh https://download.geofabrik.de/australia-oceania/australia-240426.osm.pbf australia-240426 "459.04,-47.21,523.92,-8.44"
```

The `--privileged` is needed to use the ramdisk option.
The above doesn't work,

# Parse listing with AWK

## Example Input
```
2023-12-27 10:30:24  283297278 USGS_LPC_NV_NorthWestElko_2020_D20_11TNG820470.laz
```

For awk this will be parsed as:
```
$1 = 2023-12-27
$2 = 10:30:24
$3 = 283297278
$4 = USGS_LPC_NV_NorthWestElko_2020_D20_11TNG820470.laz
```

## Bytes
```sh
awk '{ total += $3 } END { printf "total=%d\n", total }' input.txt
```

## MiB
```sh
awk '{ total += $3 } END { printf "bytes=%d, MiB=%.2f\n", total, total / 1024 / 1024 }' input.txt
```

## GiB
```sh
awk '{ total += $3 } END { printf "%.2f GiB\n", total / 1024 / 1024 / 1024 }' input.txt
```

## Automatic

```sh
awk '
{ total += $3 }
END {
    if (total >= 1024^3)
        printf "%.2f GiB\n", total / 1024^3
    else if (total >= 1024^2)
        printf "%.2f MiB\n", total / 1024^2
    else if (total >= 1024)
        printf "%.2f KiB\n", total / 1024
    else
        printf "%d bytes\n", total
}' input.txt
```

Compact
```sh
awk '{t+=$3} END {if(t>=1024^3) printf "%.2f GiB\n",t/1024^3; else if(t>=1024^2) printf "%.2f MiB\n",t/1024^2; else if(t>=1024) printf "%.2f KiB\n",t/1024; else printf "%d bytes\n",t}' input.txt
```


