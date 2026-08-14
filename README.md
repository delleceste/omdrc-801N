# omdrc-801N

open-media-drc configuration files for the Trieste room with the Nautilus 801.

This is the *site data* half of an [open-media-drc](https://github.com/delleceste/open-media-drc)
installation: the room-correction filters and BruteFIR configs for one physical
room. The engine lives in its own repository and ships only the generic `flat`
(dirac pulse, no correction) set; everything room-specific is here, so the two
can be versioned, reviewed and deployed independently.

Nothing here is secret — it is published as a complete worked example of what a
real, measured, verified deployment actually looks like.

## Layout

```text
configs/<geometry>/brutefir-<rate>[@<design>].conf.in   BruteFIR templates
filters/<geometry>/<rate>/[@<design>/]{L,R}.raw         FLOAT64_LE FIR coefficients
filters/<geometry>/provenance/<design>.json             hash-bound manifest
filters/<geometry>/provenance/<design>.source.json      the build recipe
filters/<geometry>/analysis/<design>.json               precomputed response curves
filters/<geometry>/source/<design>/                     the REW exports it was built from
filters/<geometry>/rew/                                 original REW source material
```

A *geometry* is a physical setup (speaker position). A *design* is one immutable
filter revision within it, selectable at runtime and addressed as `@<design>`.

`120.blue` is currently the only set here: the 2025 measurements as the `default`
design, plus a verified `@rscreen-20260812` A/B design.

`configs/*/*.conf` is gitignored: those are rendered from the `.conf.in`
templates at install time, when CMake rewrites `@REPO_DIR@` to the installed
site directory.

## Using it

Point the engine checkout's CMake at this one — the search path is ordered and
first match wins, so the engine keeps supplying `flat`:

```sh
cmake -S . -B build \
  -DOMDRC_SITE_DATA_DIRS="$PWD;$HOME/devel/omdrc-801N" \
  -DGEOMETRY=flat -DGEOMETRIES=120.blue
sudo cmake --install build
```

Better, put it in the engine checkout's `host.cmake`, which is where box-specific
values belong:

```cmake
set(OMDRC_SITE_DATA_DIRS "${CMAKE_SOURCE_DIR};$ENV{HOME}/devel/omdrc-801N"
    CACHE STRING "Search path for configs/<geo> + filters/<geo>")
```

The design tooling reads the same split through `OMDRC_SITE_ROOT` (or
`--site-root`), so a new design published on the design machine lands here:

```sh
export OMDRC_SITE_ROOT=~/devel/omdrc-801N
python3 scripts/new_filter_design.py --source-root … --source-ref … \
        --declaration … --write
```

Commit and push here, pull on the playback box, reinstall. See *Keeping room data
out of the engine repository* in the engine repo's `scripts/README.md`.

## Verifying

Every design carries a manifest binding its source exports, runtime coefficients,
BruteFIR configs, headroom and graph data to one another by SHA-256. Check them
from the engine checkout:

```sh
python3 scripts/verify_filter_bundle.py --all --require-sources \
        --site-root ~/devel/omdrc-801N
```

The web UI's response page releases the stored room curves only when the running
coefficients still match the manifest exactly.
