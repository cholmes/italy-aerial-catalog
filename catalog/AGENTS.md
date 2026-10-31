# AGENTS.md — Italy Aerial Orthophoto

Guidance for AI agents and automated clients working with this catalog.

**One rule survives every edit to this file.** Every claim here is either
quoted from a source or measured from the data. If you cannot point at where a
fact came from, it does not belong in this file. An agent acting on an
invented coordinate convention or an invented band meaning produces a
confident wrong answer, and nothing downstream catches it.

## What this catalog holds

Public root URL: `https://data.source.coop/mindearth/italy-aerial/stac/catalog.json`

One collection, `agea-orthophoto`: the AGEA national aerial orthophoto mosaic
of Italy, flights 2022 to 2024, as a single Zarr v3 store. The collection
guide at [agea-orthophoto/AGENTS.md](agea-orthophoto/AGENTS.md) carries the
band table, the pyramid-level table, the traps, and worked Python that has
been run against the published store. Read it before querying. This page only
orients you.

This catalog is a **mirror**. It holds metadata and points at MindEarth's
bytes on Source Cooperative. It copies nothing and re-hosts nothing.

## How to read it

The data is Zarr, not GeoParquet, so there is no SQL path and DuckDB is the
wrong tool here. One format, one access pattern:

```python
import xarray as xr
from obstore.store import S3Store
from zarr.storage import ObjectStore

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

rgba = xr.open_zarr(store, group="0", consolidated=False)["rgba"]
print(rgba.shape, rgba.dtype)   # (4, 2899968, 2244608) uint8
```

That block was run against the live store on 2026-10-31 and printed the value
shown. Nothing downloads until you call `.compute()`.

To read the grid without reading imagery, fetch the store's own root metadata,
a single 97 KB JSON document:

```bash
curl -s https://data.source.coop/mindearth/italy-aerial/italy-aerial.zarr/zarr.json \
  | jq '.attributes.multiscales.layout[0]'
```

## Join keys

None. This catalog holds one collection and one item, and nothing in it joins
to anything else. The data is a raster array, not a table.

## Quirks that produce silently wrong answers

The full list is in the collection guide. Four matter enough to repeat here,
because each returns a plausible result rather than an error.

- **The y axis descends.** Slice it `slice(ymax, ymin)`. The other order
  returns an empty array.
- **Coordinates are EPSG:3857 metres.** Convert a WGS84 bounding box first. A
  raw lon/lat selects a pixel near the projection origin, off west Africa.
- **The stored array is padded** to a multiple of the shard size. Level 0 is
  `(4, 2899968, 2244608)`; the valid extent is `(4, 2898688, 2230272)`, from
  the `valid_shape` attribute on group `0`. Padding is alpha 0.
- **Alpha is a coverage mask, not a colour.** Mask on `band=3` before treating
  any pixel as data. About three quarters of the country-wide grid is nodata.

One more that misleads rather than breaks: the pixel values are rendered
8-bit display tiles from AGEA's web map service, not calibrated reflectance.
Do not compute an index from them.

## Structure

Assets and structural links resolve relative to the object that carries them,
and nothing here carries a `self` link, so a client tracks its own location.

That is deliberate. This catalog is browsable from more than one base: the
GitHub repository that generates it, and whatever bucket prefix it is
eventually published to. A `self` link names one fixed location, and clients
resolve every relative href against it, so a catalog read from anywhere else
would chase its children and thumbnails to a host that does not serve them.

Data assets use absolute URLs into MindEarth's Source Cooperative repository,
because this catalog mirrors that data rather than re-hosting it. The
thumbnail is the only asset served from the catalog's own prefix.

## Conformance

The Portolan validator reports no errors against this catalog, but do not
read that as full support. Portolan has no Zarr profile yet: no rule
affirmatively recognizes a Zarr store as a cloud-native format the way one
recognizes a COG, PMTiles or GeoParquet, so the catalog passes by not being
caught rather than by being blessed.

Two live-probe checks do fail, and both are artifacts of a Zarr asset being a
directory prefix rather than one object: a HEAD or a ranged GET against the
store's href returns 404. The host itself serves ranges correctly, which is
verifiable against any single object inside the store. The repository's
`docs/conformance.md` records the evidence. Nothing here has been weakened to
make a rule pass.
