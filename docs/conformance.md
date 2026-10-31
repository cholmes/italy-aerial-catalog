# Portolan Conformance

Conformance means passing [rashid](https://github.com/portolan-sdi/rashid),
not claiming to conform, so it runs in CI:

```bash
python3 tests/test_conformance.py
```

That gate fails on any error-severity finding whose rule is not listed below.
The list starts empty and it must never grow without a row here. A known
deviation with an issue number is a debt someone can pay off. A silently
widened allow-list is a false claim about what this catalog conforms to.

## The rashid version floor

The gate needs rashid `>=0.1.5,<0.2.0`. It reads `rashid --version` and fails
outside that range. It also fails when rashid is absent, and prints the install
command. A skip would report a green run for a catalog that no validator read.

The floor is 0.1.5 because rules PTL-LNK-007, PTL-LNK-008, PTL-LNK-009 and
PTL-AST-006 do not exist below it. The gate asserts all four. An older rashid
reports a pass for a catalog that it never checked against them. The same range
is in `portolan-cli/pyproject.toml` and in the CI install step.

The upper bound stops an unreviewed 0.2 rule set from changing what this gate
means. Raise both bounds together when you move to 0.2, and read the new rules
first.

This file also records workarounds for the other validator CI runs. Those are
not conformance debts, because the catalog is correct and the validator is not.
They live here so nobody has to read CI code to find out why a gate skips
something.

## Accepted deviations

None. `ACCEPTED` in `tests/test_conformance.py` is empty, and the offline gate
passes with no errors:

```
rashid check catalog/ --no-data
0 error(s), 4 warning(s), 1 info(s) across 3 files.
```

The four warnings are `PTL-AST-003` asking for `file:size` and
`file:checksum` on the `data` asset, in the collection and in the item. A Zarr
store is a directory of thousands of objects with no single byte stream, so
neither field has a value to carry. The two upstream metadata assets,
`zarr-metadata` and `source-readme`, do carry both.

The one info is `PTL-PRO-002` suggesting a `canonical` link. MindEarth
publishes no STAC of its own, so there is nothing to point a `canonical` link
at. The collection carries `via` instead.

## Zarr is not a recognized Portolan format

Passing is not the same as being supported. Portolan has no Zarr profile. No
rule affirmatively recognizes a Zarr store as cloud-native the way one
recognizes a COG, PMTiles, GeoParquet or COPC asset, so this catalog passes by
not being caught rather than by being blessed.

The practical consequences today:

- `rashid`'s `_has_spatial_media_type` lists PMTiles, `image/tiff*` and COPC.
  A Zarr media type is not spatial to it. This collection is still classified
  geospatial, but only because its item declares a geometry.
- The Portolan browser cannot render the store. It recognizes
  `application/vnd.zarr; version=3; profile=multiscales` through `stac-js`,
  which places it in `wozMediaTypes` and therefore in the browser's mappable
  set, but the rendering path behind that is still being built.
- No conversion or thumbnail tooling in `portolan-cli` reads Zarr. The
  thumbnail in this catalog was rendered by hand from the store's own level-11
  overview.

The media type on the `data` asset is exactly
`application/vnd.zarr; version=3; profile=multiscales`. `stac-js` matches
media types by exact string after lowercasing, so the spacing and the
parameter order are load-bearing. Do not reformat them.

## The live probe reports two errors, and both are artifacts

`rashid check catalog/ --live` reports:

```
error PTL-LIV-001  host 'data.source.coop': range requests unsupported:
      a ranged GET returned 404, expected 206 Partial Content
error PTL-LIV-002  asset 'data': HEAD returned no usable Content-Length
```

Both come from the same cause. The `data` asset's href is
`.../italy-aerial.zarr`, which is a key prefix rather than an object. A HEAD
or a GET against a prefix returns 404, so the probe concludes the host serves
no ranges and reports no Content-Length.

The host does serve ranges. Against any real object inside the store:

```console
$ curl -s -o /dev/null -D - -r 0-99 \
    https://data.source.coop/mindearth/italy-aerial/italy-aerial.zarr/zarr.json
HTTP/2 206
content-length: 100
content-range: bytes 0-99/97679
accept-ranges: bytes
```

`PTL-LIV-001` draws a host-wide conclusion from one probe of one asset, so a
multi-file asset anywhere in a catalog mislabels the whole host. That is worth
fixing upstream in `rashid`: a directory-style asset needs either an exemption
from the byte probes or a declared entry point to probe instead. Until a
Portolan Zarr profile exists there is no spec-level answer for which object
represents a multi-file asset, so this is recorded here rather than worked
around in the catalog.

These two findings are not in `ACCEPTED`, because `tests/test_conformance.py`
runs with `--no-data` and never sees them. Nothing is being suppressed.

<!--
When you accept one, add a row and a section explaining it, like this:

| Rule | Where | Why accepted | Tracking |
|---|---|---|---|
| PTL-VIZ-001 | all thumbnails | WebP is not yet permitted; the size saving is 4x | portolan-spec#121 |

Then add the rule id to ACCEPTED in tests/test_conformance.py. Both, or
neither.
-->

## Validator workarounds

### stac-check reports a dialect crash on every collection

`tests/test_stac_valid.py` exempts one stac-check failure:

```
'list' object has no attribute 'get'
[Schema: https://schemas.portolan-sdi.org/portolan/vX.Y.Z/schema.json]. Error in Extensions.
```

The Portolan schema is valid draft-07, and rashid validates catalogs against it
cleanly. `stac-validator`, which stac-check uses, hardcodes the JSON Schema
2020-12 dialect and ignores the `$schema` a schema declares. The profile schema
uses the draft-07 tuple form of `items` in `valid_bbox`, which means something
different under 2020-12, so the library raises instead of validating.

Tracked upstream at <https://github.com/stac-utils/stac-check/issues/159>,
and on the Portolan side at
<https://github.com/portolan-sdi/portolan-spec/issues/157>.

The exemption matches that exact message, and only when the failing schema is a
Portolan profile schema. Every other stac-check error still fails the build,
and the gate prints how many objects took the exemption.

The exemption expires on its own. The gate fails once stac-check stops emitting
the crash on a collection or item that declares the profile schema, and tells
you to delete both the exemption and this section. CI installs stac-check
unpinned, so the next release triggers that without anyone watching for it.

`tests/test_stac_valid.py` also fails when stac-check is absent, and prints the
install command. It takes no version floor and no pin. The rashid floor exists
because that gate asserts four named rules. This gate asserts no stac-check
rule. It needs the opposite property. A pin holds the exemption open after the
upstream fix ships.
