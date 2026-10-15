# Luke Luck Likes Lakes 69

**Category:** OSINT

**Flag:** `cdctf{Misquamaebin Lake}`

**Verification:** Accepted after the challenge creator corrected the key.

## Challenge Overview

A lake image provides the outline of the central water body. The challenge asks for its official name.

## Solution

1. The prompt asks for the *name of the lake*. Open `lucklucklake69.png`. It shows multiple connected-looking blue areas; the target is the central lake, and the neighboring shorelines help disambiguate visually similar lakes.

2. Select the water pixels with RGB `(144, 217, 237)` and flood-fill the central water component (the `lake69_match_topo_geo.py` source uses pixel `(300,180)` as its seed). Normalize that component and compare its silhouette against lake polygons. In the `lake69_topo_results_geo.txt`, GLWD geometry **1389** has the best score, with about `0.741` silhouette intersection over union and bounding box near `91.15–91.04°W, 53.45–53.54°N`. The neighboring lakes also line up, as checked by `lake69_constellation_geo.py`.

3. Look up that location in [Natural Resources Canada's official names record FEAJE](https://geogratis.gc.ca/services/geoname/en/geonames/FEAJE). It names **Misquamaebin Lake** at `53.501376°N, 91.095239°W`; the [official shoreline feature](https://geogratis.gc.ca/services/geoname/en/features/0c5f6f7a849c20c3c10d183986969cdf.json) and its local rendering give an independent outline comparison. The central water body, not an island or a neighboring lake, is Misquamaebin Lake.

4. Put the exact official name, including “Lake” and the space, into the stated `cdctf{Name of Lake Goes Here}` format. Several correct spellings were rejected while the challenge key was erroneous; the accepted answer was reported as `cdctf{Misquamaebin Lake}` was accepted after the creator fixed the key.

## Flag

```text
cdctf{Misquamaebin Lake}
```
