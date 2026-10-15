# Luke Luck Likes Lakes 67

**Category:** OSINT

**Flag:** `cdctf{Japan}`

**Verification:** Flag confirmed separately; no checker submission record is available.

## Challenge Overview

A label-free terrain image shows a lake. The challenge asks for its country.

## Solution

1. Start with the challenge image (`image-1.png`) and the prompt's requested format, `cdctf{Country Name Goes Here}`. The crop is a north-up, label-free terrain map. The lake's outline and the river attached at its northeastern end are more useful than the background colors.

2. Isolate the water color, RGB `(144, 217, 237)`, from the PNG, crop the target lake to its bounding box, and normalize its silhouette. Compare the normalized shape with rasterized HydroLAKES polygons using intersection over union, while penalizing mismatched aspect ratios. The comparison ranks HydroLAKES ID **14811**, named **Chuzenjiko**, first: shoreline overlap `0.92457` without rotation or mirroring. This is [the corresponding polygon](https://services7.arcgis.com/poOcx60xJtGtoR7g/ArcGIS/rest/services/Lakes/FeatureServer/0/query?where=Hylak_id%3D14811&outFields=*&returnGeometry=true&outSR=4326&f=geojson).

3. Check the geographic identification independently. [Japan's national tourism agency](https://www.japan.travel/en/spot/1484/) identifies Lake Chūzenji near Nikkō, Japan. The task asks for the **country**, so insert `Japan` into the wrapper. This flag was confirmed separately, although no checker submission record is available.

## Flag

```text
cdctf{Japan}
```
