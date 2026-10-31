# Agent norms for this repository

Read this file before you change anything here.

## What this repository is

A git-backed Portolan catalog that describes one dataset: the AGEA aerial
orthophoto mosaic of Italy, which MindEarth publishes as a Zarr v3 store on
Source Cooperative.

This catalog is a **mirror**. It holds metadata only. It copies no imagery and
re-hosts nothing. Every byte it points at belongs to MindEarth and AGEA, and
the hrefs are absolute URLs into MindEarth's Source Cooperative repository.

Nobody here produces the data. Do not add this repository's owner as a
`producer` or a `host` provider. `rashid` derives mirror status from the
provider roles, and it is correct to do so.

## Publishing is configured and not enabled

`catalog.publish.yaml` holds real values for MindEarth's Source Cooperative
repository, so the dataset's authors can publish without working out the
layout. Nobody holding this repository today has write credentials for that
account.

Do not run `python3 tools/publish.py --confirm`. The dry run is safe and takes
no credentials. The prefix writes under `stac/` rather than the repository
root, which keeps the publish additive: it adds files and overwrites none of
MindEarth's.

## The publish boundary

`catalog/` is the published catalog. Everything in it is published, and
nothing outside it ever is. Do not move a file into `catalog/` to make it
publish, and do not add a path outside `publish_dir` to `tools/publish.py`.
The boundary is the only thing standing between a scratch file and a public
bucket, and it holds because it is structural rather than a list of
exclusions.

## Data never enters git

Never commit a GeoParquet, COG, PMTiles, Zarr, or COPC file. The thumbnail is
the one binary here, and it is a 370 KB PNG rendered from a published
overview. Git keeps every version of a binary forever and deleting it later
reclaims nothing.

## The conformance allow-list

`ACCEPTED` in `tests/test_conformance.py` is empty, and it should stay empty.
The offline gate passes with no errors. Never add an entry without a matching
row in `docs/conformance.md` giving the rule, where it fires, why it is
accepted, and the issue tracking its removal.

`rashid check catalog/ --live` does report two errors, and the CI gate does
not run it. Both come from the Zarr asset href being a directory prefix
rather than a single object, so a HEAD or a ranged GET against it returns
404. They are measurement artifacts, not catalog faults, and
`docs/conformance.md` records the evidence. Do not make them disappear by
weakening the catalog: the Zarr store is right and the rule set has not
caught up with multi-file assets.

## Every claim carries its source

Each statement in `catalog/**/AGENTS.md` and `catalog/**/README.md` is quoted
from MindEarth's published README, read out of the Zarr store's own metadata,
or measured against the published array. An invented band meaning or an
invented coordinate convention produces a confident wrong answer that nothing
downstream catches.

Every Python recipe in a published `AGENTS.md` has been run against the live
store. If you change one, run it again before you commit.

## The links back to this repository

`catalog/catalog.json` carries a `vcs` link and an `issues` link, both
pointing at `github.com/cholmes/italy-aerial-catalog`. If this repository
moves, change both. Keep them absolute: the repository sits outside the
published catalog, so a relative href resolves against the public base URL
and lands on a bucket path that holds nothing.
