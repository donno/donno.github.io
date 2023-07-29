OSM to TIFF
===========

Rendering OSM data to Portable Network Graphics (PNG) then constructing a
Tagged Image File Format (TIFF) from the PNGs.

Map data from OpenStreetMap for all maps shown in images unless otherwise
stated.

This project was mostly done 2020-09-12 and this post came from various notes
I made at the time. As such, it some of the steps may be missing or inaccurate.
There could also be some extra steps I performed that weren't required.

In late 2021, I also lost a hard drive which held the data and attempt..

## Creating PNGs

## WSL Setup

- Not shown creating a vhdx.
- Mounting the vhdx the first time to create filesystem.
  ```
  # From Windows
  wsl --mount --vhd D:\vms\VirtualHardDisks\osm_postres.vhdx --bare

  # From WSL Distribution
  dmesg
  # From dmesg output see what device the virtual drive was at, in my case it
  # sde.
  # mkfs -t ext4 -i 12000 /dev/sde

  # From Windows unmount it.
  wsl --unmount --vhd D:\vms\VirtualHardDisks\osm_postres.vhdx
  ```
References: https://github.com/microsoft/WSL/discussions/5896#discussioncomment-1010426

- Remount the image 
  ```
  wsl --mount --vhd D:\vms\VirtualHardDisks\osm_postres.vhdx --name osm_postgres
  ```
- Setup the postgres database.
  ```
  sudo apt install postgresql postgis osm2pgsql
  sudo mkdir /mnt/wsl/osm_postgres/osm_psqldb
  sudo chown postgres:postgres /mnt/wsl/osm_postgres/osm_psqldb
  sudo --login --user postgres
  /usr/lib/postgresql/14/bin/initdb --username=postgres --auth=trust --encoding=utf-8 /mnt/wsl/osm_postgres/osm_psqldb
  # Modify the postgres.conf in the data folder with a different port (I used 150001)
  /usr/lib/postgresql/14/bin/pg_ctl -D /mnt/wsl/osm_postgres/osm_psqldb -l /mnt/wsl/osm_postgres/osm_psqldb/logfile start
  /usr/lib/postgresql/14/bin/createdb --port 15000 -U postgres gis
  /usr/lib/postgresql/14/bin/psql --port=15000 --username=postgres -d gis -c 'CREATE EXTENSION postgis; CREATE EXTENSION hstore;'
  osm2pgsql 
  ```
- Import the data
  ```
  osm2pgsql --port=15000 --username=postgres --database gis --create --slim -k /opt/australia-latest.osm.pbf
  osm2pgsql --port=15000 --username=postgres --database gis --slim -C 3600 --hstore -S openstreetmap-carto/openstreetmap-carto.style --tag-transform-script openstreetmap-carto/openstreetmap-carto.lua /opt/australia-latest.osm.pbf
  ```
  First command: osm2pgsql took 2087s (34m 47s) overall.
  Second commmand: osm2pgsql took 1643s (27m 23s) overall.

  This next command requires ogr2ogr (from gdal-bin)
  ```
  openstreetmap-carto$ scripts/get-external-data.py --port 15000 --user postgres  --database gis
  ```
- 

  All that and ended up with 
  ```
  RuntimeError: projection::forward not supported without proj4 support (-DMAPNIK_USE_PROJ4
  ```

## Creating the TIFF

TODO: Try to track down better history of what I did here.

I relied heavily on [Geospatial Data Abstraction Library (GDAL)][1] to handle
this part.

The process I used was:

* Create TIFF from the PNG with coordinate system information.
* Merge the individual TIFFs together into a single TIFF (TODO: Determine how I did this.)

The first stage was converting the PNG to TIFF with the corresponding
coordinate system information attached. For this I wrote script that created
a CMD script with following:
```gdal_translate -expand rgb -a_srs EPSG:4326 -a_ullr 138.53759765625 -34.908457853981375 138.54034423828125 -34.91071020549453 G:\GeoData\Generated\tiles_adelaide\17\115976\79114.png G:\GeoData\Generated\tiles_adelaide_tiff\17\115976\79114.tif```

Shown here is the zoom level 17 version rather than 15 which is what I started
off with.

Special thanks to a [post][2] by Jimmy Utterström, which essentially the same
idea as this post. The post helped me with computing the WGS84 coordinates from
the tiles and the snippets were already in Python.

An alternative to this step would have been to write out a world file (PGW)
with the coordinate system information.

TODO: Publish the script to GitHub (ref_tiles.py).

### Merging the TIFFs together

1. First option using `gdal_merge.py`
    ```
    gdal_merge.py -o adelaide_15.tif <all the separate tiffs created above>
    ```
2. Second option using `gdalbuildvrt` and then `gdal_translate`
    ```
    gdalbuildvrt mosaic.vrt <all the separate tiffs created above>
    gdal_translate mosaic.vrt adelaide_15.tif
    ```

### Future

If I was to do this again, I might instead see about using a Virtual Dataset.
This is defined by a file with the extension `vrt` that, can references each of e original PNGs and I believe can even handle having the coordinate system
information within them.

## Trouble shooting TIFF
My initial attempt at constructing a TIFF from the tiles can be seen below,
where there are two problems, but the most obvious one that I wanted to fix
first was the tile ordering.

The other problem that is not as obvious from the image is the source data was
in fact in colour and not greyscale.

![City of Adelaide with tile ordering problem](/assets/png_to_tiff_adelaide_15_broken_order.png "City of Adelaide with tile ordering problem.")

TODO: Try to find the original code to mention here.

After fixing up the tile referencing that fixed that problem, however it was
still in greyscale.

![City of Adelaide with correct tile ordering](/assets/png_to_tiff_adelaide_15_broken_order.png "City of Adelaide with correct tile ordering.")

The next thing to tackle was the colouring bug. I believe the problem was when
converting the individual tiles PNG to TIFF the fact it was in colour as lost.

The fix for that was to add `-expand rgb`.

![Subsection of city of Adelaide with correct colouring](/assets/png_to_tiff_adelaide_15_correct_colouring.png "Subsection of city of Adelaide with correct colouring.")

The resulting TIFF at zoom level 15 of just the city centre and neighbouring
suburbs was 20 megabytes. The idea was add the other zoom levels to the TIFF
but I never got around to doing that.

The resulting TIFF at zoom level 17 was 250 megabytes.
The sizes are essentially the same as the sum of the individual PNGs.

[0]: https://switch2osm.org/serving-tiles/manually-building-a-tile-server-ubuntu-22-04-lts/
[1]: https://gdal.org/
[2]: https://jimmyutterstrom.com/blog/2019/06/05/map-tiles-to-geotiff/
[3]: https://github.com/mapbox/carto
[4]: https://www.npmjs.com/package/carto/
[5]: https://github.com/mapbox/osm-bright/
[6]: https://github.com/gravitystorm/openstreetmap-carto
[7]: https://github.com/mapnik/python-mapnik/issues/246
