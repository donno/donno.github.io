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