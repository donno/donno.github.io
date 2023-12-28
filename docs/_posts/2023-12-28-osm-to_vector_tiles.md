---
layout: post
title:  "OSM to Vector Tiles"
date:   2023-12-28 23:00:00 +0930
---

Rendering [OpenStreetMap](0) (OSM) data to vector tiles in [MBTiles](1).

The tiles are in the [Mapbox Vector Tile Format](2) and [tilemaker](3) will be
used to do the heavy lifting.

The tilemaker tool aims to be 'stack-free' where you don't need a database or
the dozen of other executables and as such only requires single executable.
When I looked it going from OSM data to raster (PNG and TIFF), I used the
typical stack of using a Postgres server and `osm2pgsql` and several other
tools to load the data into the database.

This came about because I was looking into the dataset that Microsoft released
which included building [footprints for Australia](9) and wanted to see that
data with OpenStreetMap within QGIS. For that use I could have simply added
an XYZ layer which used OpenStreetMap's raster tiles for context.

## Set-up TileMaker Windows
1. `curl -LO https://github.com/systemed/tilemaker/releases/download/v2.4.0/tilemaker-windows.zip`
2. Extract build\RelWithDebInfo within the ZIP to tilemaker
3. Extract resources within the ZIP to tilemaker\resources

## Set-up TileMaker Ubuntu 22.04
```
curl -LO https://github.com/systemed/tilemaker/releases/download/v2.4.0/tilemaker-ubuntu-22.04.zip
unzip tilemaker-ubuntu-22.04.zip -d ~/tilemaker-v2.4.0
cp ~/tilemaker-v2.4.0/build/tilemaker ~/tilemaker-v2.4.0/
chmod +x ~/tilemaker-v2.4.0/tilemaker
```

## Data Preparation

Extracting out a sub-section of the data is not required, specially if the
data is already small enough for the area you are interested in.

* Download an extract of the OpenStreemMap data from [Geofabrik](4).
  ```
  curl -L http://download.geofabrik.de/australia-oceania/australia-latest.osm.pbf --output Source/OpenStreetMap/osm/australia-$(date -u +"%Y-%m-%d").osm.pbf
  ```
  The following version has the date hard-coded to match the rest of the dates.
  ```
  curl -L http://download.geofabrik.de/australia-oceania/australia-latest.osm.pbf --output Source/OpenStreetMap/osm/australia-2023-12-28.osm.pbf
  ```
* Use [osmium tools](5) to extract a subsection from the previous download.
    * For Ubuntu 22.04, the tools can be an be installed via
      `apt install osmium-tool`
    * Use osmium tool to extract a subsection out of the source file.
    ```
    osmium extract Source/OpenStreetMap/osm/australia-2023-12-28.osm.pbf --bbox 137.994,-35.619,139.495,-34.359 --output Generated/OpenStreetMap/adelaide-2023-12-28.osm.pbf
    ```

Not included above is the setting up the additional shapefiles required by
tilemaker to produce maps similar to OpenStreetMap that include data that isn't
available from the OSM file.

Each of the shapefiles for coastline and landcover are available from these
places:
* https://osmdata.openstreetmap.de/download/coastlines-split-4326.zip
* https://naciscdn.org/naturalearth/10m/cultural/ne_10m_urban_areas.zip
* https://naciscdn.org/naturalearth/10m/physical/ne_10m_antarctic_ice_shelves_polys.zip
* https://naciscdn.org/naturalearth/10m/physical/ne_10m_glaciated_areas.zip

These ZIPs can be extracted and then assembled into the locations provided by
`resources/config-openmaptiles.json` from tilemaker.

## Generating Tiles
```
~/tilemaker-v2.4.0/tilemaker --input Generated/OpenStreetMap/adelaide-2023-12-28.osm.pbf --output Generated/OpenStreetMap/adelaide-2023-12-28.mbtiles --config ./resources/config-openmaptiles.json --process resources/process-openmaptiles.lua --bbox 137.994,-35.619,139.495,-34.359
```

I tested tilemaker v2.4.0 on both Ubuntu 22.04 and Windows 10 (22H2).

## Viewing Tiles
QGIS was used to confirm that the produced tiles were correct.

1. Start QGIS. For reference the version used was 3.32.1.
2. Add Vector Tile layer, this is done via **Layer > Add Layer > Add Vector Tile Layer.**
3. Select **File** as the **Source Type**
4. Browse to the mbtiles file created, via the triple dot button.
5. Click **Add** then **Close**
6. Observe you will likely see an error about _Error loading style: Style not found in database_.
   The map will have a very generic style where polygon, lines and points are
   each assigned a colour.
7. Load a Mapbox GL Style JSON File to load style that matches OSM.
    1. [Download][7] the style, for example [OSM Bright GL Style][6].
       An alternative would be [Positron GL style][7].
    2. Extract the ZIP.
    3. Open up the properties for the Vector Tile layer. This can be done by
      double clicking on the layer or using the secondary mouse button and
      choosing **Properties...** from the context menu.
    4. Click **Style** then **Load Style**
    5. Under **Load style** select **From File** if it is not already selected.
    6. Browse for the style file, for example `style-cdn.json`
    7. Click **Load Style** and the panel should close.
    8. Click *OK* on the Layer Properties panel to apply the new changes and
       close the panel.
8. The map should now be stylised similar to OpenStreetMap.

![Adelaide rendered by QGIS from vector tiles using the default style](/assets/2023-12-28_qgis_default_view_for_vector_tiles.png "Adelaide rendered by QGIS from vector tiles using the default style.")

![Adelaide rendered by QGIS from vector tiles using the OSM Bright GL style](/assets/2023-12-28_qgis_vector_tiles_osm_bright.png "Adelaide rendered by QGIS from vector tiles using OSM Bright GL style.")

You can create another style for the layer and load another Maptekbox GL style
file in which allows you to switch between them by right clicking on the layer
and choosing the other style.

The following was taken by creating a "Positron" style and loading in the
corresponding style.

![Adelaide rendered by QGIS from vector tiles using the Positron GL style](/assets/2023-12-28_qgis_vector_tiles_positron.png "Adelaide rendered by QGIS from vector tiles using Positron GL style.")

[0]: https://www.openstreetmap.org/
[1]: https://github.com/mapbox/mbtiles-spec
[2]: https://github.com/mapbox/vector-tile-spec
[3]: https://github.com/systemed/tilemaker
[4]: https://download.geofabrik.de/
[5]: https://osmcode.org/osmium-tool/
[6]: https://github.com/openmaptiles/osm-bright-gl-style
[7]: https://github.com/openmaptiles/osm-bright-gl-style/releases/download/v1.9/v1.9.zip
[8]: https://github.com/openmaptiles/positron-gl-style
[9]: https://github.com/microsoft/AustraliaBuildingFootprints