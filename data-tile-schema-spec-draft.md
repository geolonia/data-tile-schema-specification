# Data Tile Schema Specification

**Version:** 0.1.0 (Draft)

**Status:** Proposal

**Date:** 2026-01-28

**Author:** Geolonia Inc.

## Abstract

This specification defines a metadata schema for raster data tiles (such as elevation tiles,population density tiles, etc.) that enables map libraries to automatically decode and visualize tile data without hardcoded processing logic.

The schema extends TileJSON 3.0.0, adding encoding information that describes how RGB pixel values should be converted to meaningful data values.

## Motivation

Current data tile formats (Mapbox Terrain-RGB, Terrarium, etc.) encode numerical values into RGB pixels, but the decoding logic must be hardcoded into each client implementation. This creates several problems:

- New data tile formats require client-side code changes
- Visualization parameters are not discoverable
- Interoperability between different providers is limited

By providing a machine-readable schema, map libraries can automatically decode any compliant data tile without format-specific implementations.

## Schema Definition

### Root Object

This specification extends TileJSON 3.0.0. All TileJSON properties are supported, with additional properties for data tile encoding.

### TileJSON Base Properties (from TileJSON 3.0.0)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| tilejson | string | Yes | Version of TileJSON spec (e.g., “3.0.0”) |
| tiles | array | Yes | Array of tile endpoint URLs. {z} , {x} , {y} are replaced with tile coordinates |
| name | string | No | Human-readable name of the tileset |
| description | string | No | Text description of the tileset |
| version | string | No | Semver version of the tiles. Default: “1.0.0” |
| attribution | string | No | Attribution string (may contain HTML) |
| scheme | string | No | "xyz" or "tms" . Default: “xyz” |
| minzoom | integer | No | Minimum zoom level. Default: 0 |
| maxzoom | integer | No | Maximum zoom level. Default: 30 |
| bounds | array | No | [west, south, east, north] in WGS84. Default: [-180, -85.05112877980659, 180, 85.0511287798066] |
| center | array | No | [longitude, latitude, zoom] for default view |
| fillzoom | integer | No | Zoom level for generating overzoomed tiles |

### Data Tile Schema Extensions

**This specification adds the following properties to the TileJSON root object:**

| Property | Type | Required for TileJSON 3.0.0 |    Required for Data Tile Schema | Description |
|----------|------|----------|-------------|-------------|
| datatileschema | string | No | Yes | Version of this specification (e.g., “0.1.0”) |
| encoding | object | No | Yes | Encoding configuration (see below) |
| data_range | object | No | No | Expected range of decoded values |
| nodata | object | No | No | No-data value configuration |
| visualization | object | No | No | Recommended visualization parameters |

### Encoding Object

The encoding object describes how RGB pixel values are converted to data values.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| channels | object | Yes | Channel multipliers (see below) |
| offset | number | Yes | Value added after channel calculation |
| unit | string | No | Unit of the decoded value (e.g., “meters”, “celsius”) |
| precision | number | No | Smallest representable value difference |

### Channels Object

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| R | object | Yes | Red channel configuration |
| G | object | Yes | Green channel configuration |
| B | object | Yes | Blue channel configuration |
| A | object | No | Alpha channel configuration (optional) |

Each channel object contains:

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| multiplier | number | Yes | Value to multiply the channel value (0-255) by |

### Decoding Formula

The decoded value is calculated as:

```
value = R × channels.R.multiplier
      + G × channels.G.multiplier
      + B × channels.B.multiplier
      + offset
```

If alpha channel is defined:

```
value = R × channels.R.multiplier
      + G × channels.G.multiplier
      + B × channels.B.multiplier
      + A × channels.A.multiplier
      + offset
```

### Data Range Object

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| min | number | No | Minimum expected value |
| max | number | No | Maximum expected value |

### No-Data Object

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| rgb | array | No | RGB values indicating no-data (e.g., [0, 0, 0]) |
| rgba | array | No | RGBA values indicating no-data |
| value | any | No | Value to return for no-data pixels (typically null) |

### Visualization Object

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| recommended_styles | array | No | List of recommended visualization styles |
| color_ramps | object | No | Named color ramp definitions |
| hillshade | object | No | Hillshade parameters |

### Hillshade Object

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| azimuth | number | 315 | Light source azimuth in degrees |
| altitude | number | 45 | Light source altitude in degrees |
| exaggeration | number | 1.0 | Vertical exaggeration factor |

## Examples

### Mapbox Terrain-RGB

```json
{
  "tilejson": "3.0.0",
  "datatileschema": "0.1.0",
  "name": "Mapbox Terrain-RGB",
  "description": "Mapbox global elevation tiles",
  "version": "1.0.0",
  "attribution": "<a href=\"https://www.mapbox.com/\">© Mapbox</a>",
  "scheme": "xyz",
  "tiles": [
    "https://api.mapbox.com/v4/mapbox.terrain-rgb/{z}/{x}/{y}.pngraw?access_token={accessToken}"
  ],
  "minzoom": 0,
  "maxzoom": 15,
  "bounds": [-180, -85.051129, 180, 85.051129],
  "encoding": {
    "channels": {
      "R": { "multiplier": 6553.6 },
      "G": { "multiplier": 25.6 },
      "B": { "multiplier": 0.1 }
    },
    "offset": -10000,
    "unit": "meters",
    "precision": 0.1
  },
  "data_range": {
    "min": -10000,
    "max": 1667721.5
  },
  "visualization": {
    "recommended_styles": ["hillshade", "elevation_color"],
    "hillshade": {
      "azimuth": 315,
      "altitude": 45,
      "exaggeration": 1.0
    }
  }
}
```

### Terrarium (AWS/Mapzen)

```json
{
  "tilejson": "3.0.0",
  "datatileschema": "0.1.0",
  "name": "Terrarium",
  "description": "Terrarium elevation tiles (AWS/Mapzen format)",
  "version": "1.0.0",
  "attribution": "<a href=\"https://registry.opendata.aws/terrain-tiles/\">Terrain Tiles on AWS</a>",
  "scheme": "xyz",
  "tiles": [
    "https://s3.amazonaws.com/elevation-tiles-prod/terrarium/{z}/{x}/{y}.png"
  ],
  "minzoom": 0,
  "maxzoom": 15,
  "bounds": [-180, -85.051129, 180, 85.051129],
  "encoding": {
    "channels": {
      "R": { "multiplier": 256 },
      "G": { "multiplier": 1 },
      "B": { "multiplier": 0.00390625 }
    },
    "offset": -32768,
    "unit": "meters",
    "precision": 0.00390625
  },
  "data_range": {
    "min": -32768,
    "max": 32768
  }
}
```

### GSI (Geospatial Information Authority of Japan) Elevation Tiles

```json
{
  "tilejson": "3.0.0",
  "datatileschema": "0.1.0",
  "name": "GSI Elevation Tiles",
  "description": "国土地理院標高タイル (DEM10B)",
  "version": "1.0.0",
  "attribution": "<a href=\"https://maps.gsi.go.jp/development/ichiran.html\">国土地理院</a>",
  "scheme": "xyz",
  "tiles": [
    "https://cyberjapandata.gsi.go.jp/xyz/dem_png/{z}/{x}/{y}.png"
  ],
  "minzoom": 1,
  "maxzoom": 14,
  "bounds": [122.0, 20.0, 154.0, 46.0],
  "center": [139.7, 35.7, 10],
  "encoding": {
    "channels": {
       "R": { "multiplier": 655.36 },
       "G": { "multiplier": 2.56 },
       "B": { "multiplier": 0.01 }
    },
    "offset": -100000,
    "unit": "meters",
    "precision": 0.01
  },
  "nodata": {
    "rgb": [128, 0, 0],
    "value": null
  },
  "visualization": {
    "recommended_styles": ["hillshade", "elevation_color"],
    "color_ramps": {
      "elevation": {
        "stops": [
          [0, "#0000ff"],
          [500, "#00ff00"],
          [2000, "#ffff00"],
          [4000, "#ff0000"]
        ]
      }
    }
  }
}
```

### Population Density Tile (Hypothetical)

```json
{
  "tilejson": "3.0.0",
  "datatileschema": "0.1.0",
  "name": "Population Density",
  "description": "Population per square kilometer",
  "version": "1.0.0",
  "attribution": "Example Data Provider",
  "scheme": "xyz",
  "tiles": [
    "https://example.com/tiles/population/{z}/{x}/{y}.png"
  ],
  "minzoom": 0,
  "maxzoom": 12,
  "bounds": [-180, -85.051129, 180, 85.051129],
  "encoding": {
    "channels": {
      "R": { "multiplier": 65536 },
      "G": { "multiplier": 256 },
      "B": { "multiplier": 1 }
    },
    "offset": 0,
    "unit": "people/km²",
    "precision": 1
  },
  "data_range": {
    "min": 0,
    "max": 16777215
  },
  "nodata": {
    "rgb": [0, 0, 0],
    "value": null
  },
  "visualization": {
    "recommended_styles": ["choropleth"],
    "color_ramps": {
      "density": {
        "stops": [
          [0, "#f7fbff"],
          [100, "#c6dbef"],
          [1000, "#6baed6"],
          [10000, "#2171b5"],
          [50000, "#08306b"]
        ]
      }
    }
  }
}
```

## File Location

The schema file SHOULD be served at one of the following locations:

1. As the tileset metadata itself (recommended): The JSON file at the tileset root serves as both TileJSON and Data Tile Schema

https://example.com/tiles/elevation.json

2. Alongside existing TileJSON: If modifying existing TileJSON is not possible, a separate schema file

https://example.com/tiles/elevation/datatileschema.json

## Backwards Compatibility

This specification is designed as a superset of TileJSON 3.0.0. Clients that do not support Data Tile Schema will simply ignore the extension properties ( `datatileschema`, `encoding`, `data_range`, `nodata`, `visualization`) and can still use the standard TileJSON properties.

## Specification Identification

To identify that a TileJSON file includes Data Tile Schema extensions, check for the `datatileschema` property. The value indicates the version of this presence of the specification.

## Client Implementation Guidelines

### Decoding

1. Fetch the schema from either TileJSON or standalone endpoint
2. For each pixel, apply the decoding formula
3. Check for no-data values before decoding
4. Return the calculated value with appropriate unit

### Caching

Clients SHOULD cache the schema alongside tile metadata, as encoding parameters do not change frequently.

### Error Handling

If the schema is not available, clients MAY fall back to hardcoded implementations for known formats (identified by source URL patterns), but SHOULD prefer schema-based decoding when available.

## Security Considerations

Clients SHOULD validate that multiplier and offset values are within reasonable bounds to prevent numerical overflow.

## Future Extensions

Potential future additions to this specification:

- Support for multi-band tiles (beyond RGBA)
- Logarithmic or other non-linear encoding schemes
- Tile-level metadata (e.g., actual min/max values per tile)
- Compression hints

## References

- [TileJSON Specification](https://github.com/mapbox/tilejson-spec)
- [Mapbox Terrain-RGB](https://docs.mapbox.com/help/troubleshooting/access-elevation-data/)
- [Terrarium Format](https://github.com/tilezen/joerd/blob/master/docs/formats.md)

## License

This specification is released under [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).
