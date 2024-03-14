# Convert GeoJSON to mbtiles

This uses [tippecanoe][0] by Mapbox and I've used version 1.36.0. I have
since discovered there is a currently [active fork][1]


## Build tippecanoe
These are the steps I performed, for someone else following along you can avoid
the Windows Subsystem for Linux.

1. Start Ubuntu 22.04 in Windows Subsystem for Linux
2. Download and extract https://github.com/mapbox/tippecanoe/archive/refs/tags/1.36.0.tar.gz to ~/tippecanoe-1.36.0
3. Build the library (I did this a while ago, so I don't have the exact steps).
    * `cd ~/tippecanoe-1.36.0`
    * `make -j`

## Convert GeoJSON to mbtiles
This snippet is not intended to be generic but show a particular use I had,
hence the paths are specific to my scenario.

`zcat /mnt/g/GeoData/Source/Microsoft/AustraliaBuildingFootprints/Australia_2020-06-21.geojson.zip | ./tippecanoe -zg -o /mnt/g/GeoData/Generated/AustraliaBuildingFootprints_2020-06-21.mbtiles --drop-densest-as-needed`

## Convert  .osm.pbf to mbtiles

`tippecanoe` can also be used to do this however the newer tool on the scene is
`tilemaker`, this tool does not support GeoJSON in 2023, however it is on the
developer's feature list.

1. Start Ubuntu 22.04 in Windows Subsystem for Linux
2. `mkdir ~/tilemaker-v2.4.0`
3. `cd ~/tilemaker-v2.4.0`
4. `curl -OL https://github.com/systemed/tilemaker/releases/download/v2.4.0/tilemaker-ubuntu-22.04.zip && unzip tilemaker-ubuntu-22.04.zip`
5. `./build/tilemaker --input /mnt/g/GeoData/Source/OpenStreetMap/osm/australia-2023-12-28.osm.pbf --output /mnt/g/GeoData/Generated/OpenStreetMap/australia-2023-12-28.mbtiles --config ./resources/config-openmaptiles.json --process resources/process-openmaptiles.lua`

This expects shapefiles for coastline and landcover to be available as seen
below.
* https://osmdata.openstreetmap.de/download/coastlines-split-4326.zip
* https://naciscdn.org/naturalearth/10m/cultural/ne_10m_urban_areas.zip
* https://naciscdn.org/naturalearth/10m/physical/ne_10m_antarctic_ice_shelves_polys.zip
* https://naciscdn.org/naturalearth/10m/physical/ne_10m_glaciated_areas.zip


## Extract Adelaide from Australia osm.pbf

This uses [osmium-tool][3].

`osmium extract  Source/OpenStreetMap/osm/australia-2023-12-28.osm.pbf --bbox 137.994,-35.619,139.495,-34.359 --output Generated/OpenStreetMap/adelaide-2023-12-28.osm.pbf`

[0]: https://github.com/mapbox/tippecanoe
[1]: https://github.com/felt/tippecanoe
[2]: https://github.com/systemed/tilemaker
[3]: https://osmcode.org/osmium-tool/
