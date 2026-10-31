# italy-aerial-catalog

A git-backed [Portolan](https://www.portolan-sdi.org/) catalog describing the
AGEA aerial orthophoto mosaic of Italy, which
[MindEarth](https://mindearth.ai) publishes as a cloud-native Zarr v3 array on
Source Cooperative at
[mindearth/italy-aerial](https://source.coop/mindearth/italy-aerial).

**This is a third-party mirror, and it is not published anywhere yet.** The
repository holds metadata only. It copies no imagery and re-hosts nothing:
every data href is an absolute URL into MindEarth's repository. The intent is
to hand this to the dataset's authors so they can publish it themselves.

## What is in here

| Path | What it is |
|---|---|
| `catalog/` | The catalog itself. Everything here publishes; nothing outside it does. |
| `catalog/catalog.json` | Root catalog, one child collection. |
| `catalog/agea-orthophoto/` | The collection, its single item, its docs and its thumbnail. |
| `catalog.publish.yaml` | Where the catalog would publish. Configured, not enabled. |
| `tests/` | The gates. Run them before every commit. |
| `tools/publish.py` | The publisher. Dry runs without credentials. |
| `docs/conformance.md` | What validates, what does not, and why. |

## Publishing is configured and switched off

`catalog.publish.yaml` points at
`s3://mindearth/italy-aerial/stac`, so the authors can publish without
working out the layout. Nobody holding this repository today has write
credentials for that account, and `tools/publish.py` uploads only when given
`--confirm`.

The prefix is a `stac/` subdirectory rather than the repository root, on
purpose. Publishing syncs `catalog/` to the prefix one-to-one, so a root
prefix would overwrite the `README.md` that MindEarth already serves at
`https://data.source.coop/mindearth/italy-aerial/README.md`. Writing under
`stac/` only adds files.

If MindEarth would rather have `catalog.json` at the repository root, that is
a two-line change to `write_prefix` and `public_base`, plus the `self` link
in each STAC object. It is their call, and it means deciding what happens to
the existing README.

There is no publish workflow in CI. Publishing is a deliberate local command.

## Working on it

```bash
python3 tests/run_all.py      # every gate
python3 tools/publish.py      # dry run, uploads nothing, needs no credentials
```

The conformance gate needs `rashid`:

```bash
uv tool install 'rashid>=0.1.8,<0.2.0'
```

Validate by hand, including the checks CI skips:

```bash
rashid check catalog/            # reads asset bytes
rashid check catalog/ --live     # probes the host too
```

## State of play

The offline validation is clean: `rashid check catalog/` reports zero errors.

That is not the same as Portolan supporting Zarr, and it does not yet. No rule
recognizes a Zarr store as a cloud-native format the way one recognizes a COG
or GeoParquet, so this catalog passes by not being caught. The `--live` probe
reports two errors, both artifacts of a Zarr asset href being a directory
prefix rather than a single object. `docs/conformance.md` carries the
evidence, including a `curl` transcript showing the host does serve range
requests correctly.

The `data` asset uses the media type
`application/vnd.zarr; version=3; profile=multiscales`, which `stac-js`
already classifies as a mappable web-optimized GeoZarr type. Matching is an
exact string comparison after lowercasing, so the spacing and parameter order
matter.

## License

The catalog metadata in this repository is released under the terms in
[LICENSE](LICENSE).

The data it describes is **CC-BY-4.0** and belongs to AGEA, with the derived
Zarr dataset produced by MindEarth. Required attribution:

> Contains orthophotos © AGEA – Agenzia per le Erogazioni in Agricoltura,
> voli 2022–2024, CC BY 4.0, modified by MindEarth.
