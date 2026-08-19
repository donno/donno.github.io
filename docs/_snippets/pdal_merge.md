---
---
## Objective
Generate a Cloud Optimized Point Cloud (COPC) from a several LAS/LAZ files
and assign metadata to them.

## Pipeline

Reusable `merge.json`
```json
[
  {
    "type": "readers.las",
    "filename": "*.laz"
  },
  {
    "type": "filters.merge"
  },
  {
    "type": "writers.copc",
    "filename": "merged.copc.laz",
    "forward": "all",
    "vlrs": [
      {
        "user_id": "DATASET_META",
        "record_id": 1,
        "description": "Dataset metadata",
        "filename": "dataset-metadata.json"
      }
    ]
  }
]
```

Data specific example for the `dataset-metadata.json` file:

```json
{
  "license": "Creative Commons Attribution 4.0",
  "owner": "Example Organisation",
  "creation_date": "2026-07-21",
  "theme": "Elevation / LiDAR"
}
```

This metdata field is not standardised so tools won't know to use it.

## Execution

* `pdal pdal pipeline .\merge.json`
* `micromamba-win-64.exe run -p pdal pdal pipeline .\merge.json`

## Notes

There are some headers that are common on LAS that could be set, which would be set
in the last object in the pipeline such as:

* creation_year and creation_doy (Day of the year, starting at 1 for January 1)
* system_id and software_id
