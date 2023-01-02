OSM to TIFF
===========

Rendering OSM data to  Portable Network Graphics (PNG) then constructing a
Tagged Image File Format (TIFF) from the PNGs.

Map data from OpenStreetMap for all maps shown in images unless otherwise
stated.

This project was mostly done 2020-09-12 and this post came from various notes
I made at the time. As such, it some of the steps may be missing or inaccurate.
There could also be some extra steps I performed that weren't required.

## Creating PNGs

### First time setup

This is the set-up needed for osm2pgsql.

* Create the database storage
    ```
    .\initdb.exe --username=postgres --auth=trust --encoding=utf-8  -D G:\GeoData\Generated\postgres_data
    ```
* Start the server
    ```
    .\pg_ctl -D G:\GeoData\Generated\postgres_data -l G:\GeoData\Generated\postgres_data\logfile start
    ```
* Create the database called 'gis' which is where the data will be imported
    into as well as load the PostGIS extensions for the database.
    Without this you get:
    `Connection to database failed: FATAL:  database "gis" does not exist`.
    ```
   .\createdb -U postgres gis
   .\psql --username postgres -d gis -c 'CREATE EXTENSION postgis; CREATE EXTENSION hstore;'
    ```
* Import the data
    ```
    .\osm2pgsql.exe -c -d gis --slim -k G:\GeoData\Source\australia-oceania-latest.osm.pbf
    ```
    This took about 30 minutes.
* Apply the rendering / carto scripts. I believe tweaks the database for the
  purpose of making the rendering easier.
  ```
  .\osm2pgsql.exe --slim -d gis -C 3600 --hstore -S openstreetmap-carto/openstreetmap-carto.style --tag-transform-script openstreetmap-carto/openstreetmap-carto.lua G:\GeoData\Source\australia-oceania-latest.osm.pbf --user postgres
  ```
* Clone the OpenStreetMap Carto repository.
  ```
  git clone https://github.com/gravitystorm/openstreetmap-carto.git
  ```
* Inside the repository, run the script
  ```
  cd openstreetmap-carto
  scripts/get-external-data.py --username postgres
  scripts/get-fonts.sh
  ```
  At the time, I jumped through hoops to install all the relevant fonts via
  Ubuntu's package manager (apt) however when revisiting it for this article I
  noticed the get-fonts.sh script.
  The command that I used at the time was:
  ```
  apt install fonts-arundina fonts-sil-padauk fonts-khmeros fonts-gargi fonts-tibetan-machine fonts-droid-fallback fonts-open-sans
  ```

The remaining steps I performed are missing as I used a Ubuntu instance on
Windows Subsystem for Linux (WSL) which I have since deleted without taking
a copy of my `.bash_history`. As such my notes so I am actually missing
the commands taht creates the PNGs.

For generating the tiles, you most likely will have better luck, following the
Switch2OSM's article ["Manually building a tile server"](0).

### CartoCSS

1. Install [carto](3) through [npm](4).
2. Clone the Git [repository](5) for the OSM Bright style.
3. Follow the setup instructions in the README.md
    * Download shapefile
    * Rename/copy configure.py.sample to configure.py
    * Tweak the variables to match the setting. The importer should be
        osm2pgsql (which is the default at time of writing this).
        * Run make
4. Run Carto to convert from the MML format to Mapnik XML.

I don't think I bothered with shapeindex as its a one-off task and I wasn't
going to reuse the shapefiles.
```
npm -g carto
git clone https://github.com/mapbox/osm-bright.git
# <follow the instructions in the README.txt>
carto $env:HOME\MapBoxProject\OSMBright\project.mml > osm_bright.xml
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

TODO: Publish the script to GitHub (ref_tiles.py)..


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
