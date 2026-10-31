# AGENTS.md — AGEA Orthophoto Mosaic of Italy

Guidance for AI agents and automated clients working with this collection.

**One rule survives every edit to this file.** Every claim here is either
quoted from a source or measured from the data. If you cannot point at where a
fact came from, it does not belong in this file. An agent acting on an
invented coordinate convention or an invented band meaning produces a
confident wrong answer, and nothing downstream catches it.

Sources used below: MindEarth's published
[README](https://data.source.coop/mindearth/italy-aerial/README.md), the
store's own
[root metadata](https://data.source.coop/mindearth/italy-aerial/italy-aerial.zarr/zarr.json),
and measurements taken against the live store on 2026-10-31. Every Python
block was run before it was written here.

## Overview

One Zarr v3 store holds the whole mosaic. There is no per-scene tiling and no
item index to search: the collection has a single item because the data is a
single continuous array.

| | |
|---|---|
| Store | `https://data.source.coop/mindearth/italy-aerial/italy-aerial.zarr` |
| Array | `rgba`, dims `(band, y, x)`, `uint8` |
| Level-0 shape | `(4, 2899968, 2244608)` |
| Valid shape | `(4, 2898688, 2230272)` |
| CRS | EPSG:3857 |
| Level-0 resolution | 0.597164283559817 m/pixel |
| Bands | `red`, `green`, `blue`, `alpha` |
| Chunks / shards | inner `4x1024x1024`, shard `4x16384x16384`, zstd |
| Pyramid | 12 levels, groups `0` to `11` |
| Region | `eu-central-1`, anonymous access |

## Accessing the data

Source Cooperative serves no directory listing over plain HTTP. An HTTP Zarr
store fails on this dataset. Use an S3-style store, which works anonymously.

```bash
pip install "zarr>=3" xarray obstore pyproj numpy
```

```python
import xarray as xr
from obstore.store import S3Store
from zarr.storage import ObjectStore

def open_level(level=0):
    store = ObjectStore(
        S3Store(
            "mindearth",
            prefix="italy-aerial/italy-aerial.zarr",
            endpoint="https://data.source.coop",
            region="eu-central-1",
            skip_signature=True,                 # anonymous
            virtual_hosted_style_request=False,  # path-style URLs
        ),
        read_only=True,
    )
    return xr.open_zarr(store, group=str(level), consolidated=False)["rgba"]

rgba = open_level(0)
print(rgba.shape, rgba.dtype)   # (4, 2899968, 2244608) uint8
```

Nothing downloads until you call `.compute()`. Only the shards you touch are
fetched.

Pass `group=str(level)` to reach a pyramid level. `consolidated=False` is
required: the consolidated metadata lives inside each group, not at the store
root in the layout `xarray` expects.

## Schema and field notes

The array is RGBA, and **alpha is a coverage mask rather than a colour**.
Alpha is 0 wherever there is no AGEA imagery, which covers the sea,
neighbouring countries, and the padding at the edge of the stored grid. Mask
with `subset.sel(band=3) == 0` before you treat any pixel as a measurement.

| Band index | Name | Meaning |
|---|---|---|
| 0 | red | Red channel of the rendered AGEA tile imagery |
| 1 | green | Green channel |
| 2 | blue | Blue channel |
| 3 | alpha | 0 means no coverage, 255 means imagery |

Pyramid levels, each a group in the store. Resolution is level 0 multiplied by
two per level, read from the `multiscales` layout in the store's root
metadata.

| Level | Shape (y, x) | Resolution |
|---|---|---|
| 0 | 2899968 x 2244608 | 0.60 m |
| 1 | 1449984 x 1122304 | 1.19 m |
| 2 | 724992 x 561152 | 2.39 m |
| 3 | 362496 x 280576 | 4.78 m |
| 4 | 181248 x 140288 | 9.55 m |
| 5 | 90624 x 70144 | 19.11 m |
| 6 | 45312 x 35072 | 38.22 m |
| 7 | 22656 x 17536 | 76.44 m |
| 8 | 11328 x 8768 | 152.87 m |
| 9 | 5664 x 4384 | 305.75 m |
| 10 | 2832 x 2192 | 611.50 m |
| 11 | 1416 x 1096 | 1222.99 m |

## Data quality and usage notes

These are the traps that produce silently wrong answers.

**The y axis descends.** Rows run north to south, so a y slice goes from the
larger coordinate to the smaller one: `slice(ymax, ymin)`. Writing
`slice(ymin, ymax)` returns an empty array rather than an error, which is the
single easiest mistake to make here.

**Coordinates are EPSG:3857 metres, not degrees.** Convert a WGS84 bounding
box before slicing. A lon/lat value passed straight into `.sel` selects a
pixel a few metres from the origin, off the coast of west Africa.

**The stored array is padded.** Level 0 is `(4, 2899968, 2244608)` but the
valid extent is `(4, 2898688, 2230272)`, from the `valid_shape` attribute on
group `0`. The padding is alpha 0. Measured on the live store, the far corner
pixel `rgba[:, -1, -1]` is `[0 0 0 0]`.

**Most of the grid is empty.** Italy is a poor fit for its own bounding box.
Measured across pyramid level 11, 25.4 percent of pixels have alpha above 0.
Expect roughly three quarters of any country-wide read to be nodata.

**The imagery is display tiles, not reflectance.** Quoting MindEarth's README:
the values "come from AGEA's rendered 8-bit tile service, so they are display
imagery, not calibrated reflectance, and not suitable for quantitative
radiometry."

**Resolution is coarser than AGEA's source.** AGEA produces the orthophotos at
20 cm. This mosaic is built from the Web Mercator tile service, whose finest
zoom corresponds to about 40 to 48 cm on the ground in Italy. Basilicata is
available only at about 180 cm and is upsampled four times, so it carries no
extra detail and looks blurrier. For full 20 cm detail, use the
[Geoportale AGEA](https://geoportale.agea.gov.it/).

**Years vary by area, and seams are expected.** The priority mosaic takes 2024
where it exists, then 2023, then 2022. Adjacent areas can come from different
years, and the mosaic is not radiometrically balanced. A colour step across a
region boundary is the data, not an artifact of your read.

**Dates are year-precise at best.** The collection's temporal extent spans
2022-01-01 to 2024-12-31 because AGEA publishes flight years rather than
per-pixel acquisition dates. Do not present a date from this collection as the
acquisition date of a given pixel.

## Example queries

Every block below was run against the live store on 2026-10-31. The printed
values are the output that run produced.

### Read a lon/lat bounding box at full resolution

```python
from pyproj import Transformer

to3857 = Transformer.from_crs(4326, 3857, always_xy=True).transform

def read_bbox(rgba, west, south, east, north):
    xmin, ymin = to3857(west, south)
    xmax, ymax = to3857(east, north)
    return rgba.sel(x=slice(xmin, xmax), y=slice(ymax, ymin))  # y descends

sub = read_bbox(open_level(0), 7.815056, 45.86911, 7.838886, 45.877512).compute()
print(sub.shape)        # (4, 2250, 4442)
```

### Choose a pyramid level for a target ground resolution

Reading a large area at level 0 is expensive. Pick the level that matches the
resolution you actually need.

```python
import numpy as np

BASE = 0.597164283559817

def level_for(target_res_m):
    return int(min(11, max(0, np.floor(np.log2(target_res_m / BASE)))))

print(level_for(10), level_for(100))   # 4 7
```

### Read a region at a coarse level

```python
region = read_bbox(open_level(7), 12.2, 41.6, 12.8, 42.1).compute()
print(region.shape)     # (4, 977, 873)
```

### Measure coverage from the alpha band

```python
alpha = region.sel(band=3).values
print("coverage %.2f%%" % (100 * (alpha > 0).mean()))   # coverage 93.05%
```

### Display a subset

```python
import matplotlib.pyplot as plt

img = sub.transpose("y", "x", "band").values    # (y, x, 4) RGBA
plt.imshow(img)
plt.axis("off")
plt.show()
```

### Save a subset as a GeoTIFF

```python
import rioxarray  # noqa: F401

sub.rio.write_crs("EPSG:3857").rio.to_raster("subset.tif", compress="deflate")
```

### Read the grid without reading imagery

The store's root metadata is a single 97 KB JSON document holding the
multiscale layout, the WKT2, the tile matrix set and the AGEA source
endpoints. Fetch it to confirm the grid before you read any pixels.

```bash
curl -s https://data.source.coop/mindearth/italy-aerial/italy-aerial.zarr/zarr.json \
  | jq '.attributes.multiscales.layout[0]'
```

## Related collections

None. This catalog holds one collection.

The upstream repository, [mindearth/italy-aerial](https://source.coop/mindearth/italy-aerial),
publishes no STAC of its own, so this collection carries a `via` link to it
rather than a `canonical` link.

## Structure

Assets and structural links resolve relative to the object that carries them.
The `data`, `zarr-metadata` and `source-readme` assets use absolute URLs into
MindEarth's repository, because this catalog mirrors that data and does not
re-host it.
