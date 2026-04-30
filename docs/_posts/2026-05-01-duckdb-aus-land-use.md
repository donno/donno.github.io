---
layout: post
title:  Australian Land Use with DuckDB
date:   2026-05-01 11:00:00 +1030
---

Investing how much land in Australia is used for different crops.
This came about after watching the recently ["Ask Hank Anything"](0)
episode featuring Simone Giertz, where Hank compared the size of land used
for growing corn (also known as maize) in the United States of America with the
land mass of Sweden.

The Australian government publishes information about this as part of 
their "Catchment Scale Land Use of Australia" [dataset](1). The version I'm
interested in is the ESRI Shapefile version as that has the land parcels with
their natural boundaries rather than being gridded into a raster.

## Getting started

1. Downloading the [package](2).
    ```sh
    curl -LO https://data.gov.au/data/dataset/8af26be3-da5d-4255-b554-f615e950e46d/resource/b216cf90-f4f0-4d88-980f-af7d1ad746cb/download/clum_commodities_2023.zip
        ```
2. Downloading DuckDB
    ```
    curl -LO https://install.duckdb.org/v1.5.2/duckdb_cli-windows-amd64.zip
    tar xf duckdb_cli-windows-amd64.zip
    ```
3. Extract the packages
    ```sh
    tar xf clum_commodities_2023.zip
    tar xf duckdb_cli-windows-amd64.zip
    ```
4. Run DuckDB - Open a terminal and run duckdb.exe
5. Install and load the spatial extension. This enables being able to read the
   ESRI Shapefiles.
    ```
    INSTALL spatial;
    LOAD spatial;
    ```
6. Test querying the data.
    ```
    SELECT * FROM 'CLUM_Commodities_2023/CLUM_Commodities_2023.shp' LIMIT 10
    ```

The output of the above was:
```
┌──────────┬────────────┬────────────┬───────────┬─────────┬───────────────┬───────────┬───────┬────────────────┬────────────────────────┐
│ OBJECTID │ Commod_dsc │ Broad_type │ Source_yr │  State  │    Area_ha    │ Lucodev8n │ Date  │    Tertiary    │          geom          │
│  int64   │  varchar   │  varchar   │   int64   │ varchar │    double     │   int64   │ int64 │    varchar     │        geometry        │
├──────────┼────────────┼────────────┼───────────┼─────────┼───────────────┼───────────┼───────┼────────────────┼────────────────────────┤
│       20 │ soybeans   │ Pulses     │      2022 │ Qld     │ 18.2896003723 │       334 │  2023 │ 3.3.4 Oilseeds │ POLYGON ((147.103864…  │
│       21 │ soybeans   │ Pulses     │      2022 │ Qld     │ 33.8995018005 │       334 │  2023 │ 3.3.4 Oilseeds │ POLYGON ((147.109598…  │
│       27 │ soybeans   │ Pulses     │      2022 │ Qld     │ 20.5431003571 │       334 │  2023 │ 3.3.4 Oilseeds │ POLYGON ((147.107205…  │
│       28 │ soybeans   │ Pulses     │      2022 │ Qld     │  52.998500824 │       334 │  2023 │ 3.3.4 Oilseeds │ POLYGON ((147.115359…  │
│       29 │ soybeans   │ Pulses     │      2022 │ Qld     │ 15.3131999969 │       334 │  2023 │ 3.3.4 Oilseeds │ POLYGON ((147.124137…  │
│       30 │ soybeans   │ Pulses     │      2022 │ Qld     │ 16.8449001312 │       334 │  2023 │ 3.3.4 Oilseeds │ POLYGON ((147.125973…  │
│       31 │ soybeans   │ Pulses     │      2022 │ Qld     │ 53.6531982422 │       334 │  2023 │ 3.3.4 Oilseeds │ POLYGON ((147.247055…  │
│       33 │ soybeans   │ Pulses     │      2022 │ Qld     │ 7.08559989929 │       334 │  2023 │ 3.3.4 Oilseeds │ POLYGON ((147.316971…  │
│       34 │ soybeans   │ Pulses     │      2022 │ Qld     │ 14.9702997208 │       334 │  2023 │ 3.3.4 Oilseeds │ POLYGON ((147.321245…  │
│       35 │ soybeans   │ Pulses     │      2022 │ Qld     │ 3.94431996346 │       334 │  2023 │ 3.3.4 Oilseeds │ POLYGON ((147.347139…  │
├──────────┴────────────┴────────────┴───────────┴─────────┴───────────────┴───────────┴───────┴────────────────┴────────────────────────┤
│ 10 rows                                                                                                                     10 columns │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## Transforming the Data
Next convert the data into a Parquet file for further analysis.
This is mainly because I'm not interested in the geometry for this project.

```SQL
COPY (SELECT Commod_dsc, State, Area_ha, date, Tertiary FROM  'CLUM_Commodities_2023/CLUM_Commodities_2023.shp')
TO 'australia_land_use_2023.parquet' (FORMAT PARQUET);
```

## Explore data
DuckDB has a built-in web UI that works kind of like a [Jupyter Notebook](4).

Some queries:
```SQL
SELECT * FROM australia_land_use_2023 LIMIT 10;

-- List of commodities in the data set.
SELECT DISTINCT Commod_dsc FROM australia_land_use_2023;

-- List the commodities and their how many occurrences there are: 
SELECT Commod_dsc, COUNT(*) as count
FROM australia_land_use_2023
GROUP BY Commod_dsc ORDER BY count DESC;

-- List the commodities and their land usage in hectares.
SELECT Commod_dsc, SUM(Area_ha) as total_area
FROM australia_land_use_2023
GROUP BY Commod_dsc ORDER BY total_area DESC;

-- List the commodities and their land usage in square kilometres.
SELECT Commod_dsc as commodities, SUM(Area_ha) * 0.01 as total_area
FROM australia_land_use_2023
GROUP BY Commod_dsc ORDER BY total_area DESC;

-- List the commodities and their land usage in square kilometres - rounded to
-- four decimal places.
SELECT Commod_dsc as commodities, ROUND(SUM(Area_ha) * 0.01, 4) as total_area
FROM australia_land_use_2023
GROUP BY Commod_dsc ORDER BY total_area DESC;
```

A hectares is 10000m², or 0.01 km²

## Results
|     commodities      | total_area  |
|----------------------|------------:|
| cattle meat          | 592776.5517 |
| cattle dairy         | 17495.1264  |
| sugar cane           | 5724.9786   |
| cotton               | 3546.4185   |
| sheep                | 2790.3753   |
| grapes               | 1883.9775   |
| horses               | 945.7716    |
| sorghum              | 628.7508    |
| grapes wine          | 498.8644    |
| macadamias           | 407.78      |
| citrus               | 381.701     |
| wheat                | 337.291     |
| olives               | 336.1936    |
| cattle               | 271.6118    |
| almonds              | 262.0005    |
| pigs                 | 219.8151    |
| avocados             | 192.4341    |
| barley               | 183.0572    |
| mangoes              | 159.3148    |
| bananas              | 135.6556    |
| chickens meat        | 132.8076    |
| molluscs             | 131.7495    |
| lentils              | 128.7922    |
| soybeans             | 106.0606    |
| oats                 | 97.4443     |
| vegetables and herbs | 96.4415     |
| chickens             | 86.1136     |
| grapes table         | 84.7563     |
| tea tree             | 65.2842     |
| sandalwood           | 63.3466     |
| turf                 | 60.4177     |
| apples               | 37.9361     |
| carrots              | 35.1529     |
| grapes dried         | 34.2522     |
| pears                | 34.2353     |
| peas                 | 31.7424     |
| oil mallee           | 30.2169     |
| potatoes             | 29.607      |
| rice                 | 26.802      |
| sheep wool           | 26.2108     |
| blueberries          | 24.5511     |
| walnuts              | 24.3185     |
| beans                | 24.3114     |
| hazelnuts            | 24.2792     |
| lucerne              | 23.6889     |
| canola               | 22.9188     |
| vegetables           | 21.2902     |
| cherries             | 21.0187     |
| peaches              | 20.8755     |
| pineapples           | 19.6939     |
| melons               | 17.5811     |
| pecans               | 16.5068     |
| field peas           | 14.381      |
| pyrethrum            | 13.2498     |
| crustaceans          | 12.6089     |
| chia                 | 12.1202     |
| chickens eggs        | 11.8731     |
| maize                | 10.8415     |
| sheep meat           | 9.7956      |
| watermelons          | 9.3334      |
| field beans          | 9.1745      |
| eucalyptus oil       | 9.0261      |
| plums                | 8.8615      |
| nectarines           | 8.801       |
| flowers and bulbs    | 8.6494      |
| alpacas              | 7.5168      |
| tomatoes             | 7.1377      |
| pistachios           | 7.0366      |
| truffles             | 6.9572      |
| cucurbits            | 6.5652      |
| bees                 | 6.4876      |
| chickpeas            | 6.3521      |
| algae                | 6.3496      |
| lupins               | 6.0495      |
| apricots             | 5.6585      |
| strawberries         | 4.7869      |
| onions               | 4.1619      |
| asparagus            | 4.1164      |
| vetches              | 3.8065      |
| rye cereal           | 3.1165      |
| finfish              | 2.8121      |
| coffee               | 2.7075      |
| alkaloid poppies     | 2.6433      |
| goats                | 2.385       |
| cattle stud          | 2.3505      |
| jojoba               | 2.2474      |
| pomegranate          | 1.9307      |
| deer                 | 1.6826      |
| raspberries          | 1.6155      |
| triticale            | 1.4352      |
| christmas trees      | 1.356       |
| passionfruit         | 1.2529      |
| lavender             | 1.1937      |
| nashi pears          | 1.1479      |
| kiwifruit            | 1.1342      |
| crocodiles           | 1.1012      |

The items with less than  1 km² has been omitted from this post.

## References
* Land use: ABARES 2024, Catchment Scale Land Use of Australia – Update December 2023 version 2, Australian Bureau of Agricultural and Resource Economics and Sciences, Canberra, June, CC BY 4.0, DOI: 10.25814/2w2p-ph98
* Commodities: ABARES 2024, Catchment Scale Land Use of Australia – Commodities – Update December 2023, Australian Bureau of Agricultural and Resource Economics and Sciences, Canberra, February CC BY 4.0. DOI: 10.25814/zfjz-jt75

## Future

* List of land mass and size of countries
* See if the United States of America has a similar data set available.

[0]: https://www.youtube.com/watch?v=8irhCqM7XUo&
[1]: https://data.gov.au/data/dataset/catchment-scale-land-use-of-australia-and-commodities-update-december-2023
[2]: https://data.gov.au/data/dataset/catchment-scale-land-use-of-australia-and-commodities-update-december-2023/resource/b216cf90-f4f0-4d88-980f-af7d1ad746cb
[3]: https://duckdb.org/2025/03/12/duckdb-ui
[4]: https://jupyter.org/