# lscweather

A storm chaser tracking map: live GPS positions, radar and NWS warnings on one
page, with each chaser's YouTube stream playing in place on the map.

Chasers already report position to SpotterNetwork and already stream to
YouTube, but nothing joined the two. Following a chase meant a radar tab, a
position tab and a stream tab, and working out by eye which stream belonged to
which dot.

**This is in progress.** The map works. The live-stream detection has never run
against a real broadcast. See Not done before relying on anything here.

Copyright (c) 2026 Live Storm Chasers LLC. All rights reserved.
Source-visible, not open source — reading is fine, using it needs written
permission. See LICENSE.

---

## Why

**One HTML file, no build step.** The whole application is `index.html`. No
bundler, no package manifest, no toolchain to keep alive between storm seasons.
A fix is one file upload. The cost is a large single file with no module
boundaries, which is the right trade for something edited in bursts months
apart.

**Vector tiles, not raster.** The first working version used a raster basemap.
Road colour, label typeface and label size are all baked into the tile image
there, so the only control available is a CSS filter over the whole tile at
once. That dead end is gotcha 4 below. Vector tiles make roads, labels and
borders individually addressable, at the cost of rewriting all the layer code.
Worth it: the map is the product and it has to stay legible under radar.

**Two server-side proxies, not by preference.** The position feed refuses
browser requests outright (gotcha 3) and the live-stream lookup needs an API key
that cannot ship in client code. Neither proxy exists because a backend was
wanted.

## Files

```
index.html    the application
chaser.html   from an abandoned backend build — see Not done
admin.html    from an abandoned backend build — see Not done
```

Only `index.html` is live. The other two call an API that was never deployed.

## Use

Deploy to any static host. The two endpoints near the top of the script block
have to point at running proxies:

```js
const WORKER_URL = '...';   // position proxy
const YT_WORKER  = '...';   // live-stream detector
```

The chaser roster is directly below them:

```js
const CHASER_CONFIG = [
  { sn_name:'Exact Name In Position Feed', display_name:'Shown On Map',
    streaming:false, stream_url:'', viewers:0 },
];
```

`sn_name` must match the position feed exactly, including spelling and middle
names. A mismatch is silent — the chaser simply never appears on the map.

## The position proxy

Not in this repository. Contract it must satisfy — `GET`, no parameters,
returns GeoJSON:

```
{
  "type": "FeatureCollection",
  "features": [{
    "type": "Feature",
    "geometry": { "type": "Point", "coordinates": [lon, lat] },
    "properties": {
      "name":      "Exact Name In Position Feed",
      "timestamp": "YYYY-MM-DD HH:MM:SS UTC",
      "status":    "MOVING" | "STATIONARY",
      "heading":   0-360 | null
    }
  }]
}
```

`heading` is null when stationary, which is why the marker's direction arrow
only rotates when a chaser is moving. There is no speed field anywhere in the
source data, so the map cannot show one.

## The live-stream proxy

Not in this repository. Contract — `GET`, no parameters:

```
{
  "ok": true,
  "live": {
    "Exact Name In Position Feed": {
      "video_id": "...",
      "url":      "https://www.youtube.com/watch?v=...",
      "viewers":  1234
    }
  },
  "checked_at": 1234567890
}
```

It polls one channel for active live streams and matches each to a chaser by a
hashtag in the title, description or tags. An empty `live` object means nobody
is streaming — the page treats absence as offline, so a stream that ends clears
itself within one poll with no extra signalling.

## Layer stack

The order is load-bearing and three insertion points have to stay in this
relationship. Bottom to top:

```
basemap                vector tiles
warning fills          inserted before the radar layer
radar                  inserted before the first road layer
roads, casings         from the basemap style
state, county borders  GeoJSON, before the first symbol layer
warning outlines       no insertion point — top of the stack
chaser markers         DOM markers, above the canvas
labels, shields        basemap symbol layers
```

The point: warning fills tint the ground beneath the storm rather than sitting
on top of it, roads and highway shields stay readable through precipitation,
and warning outlines are never obscured.

Move the radar above the first road layer and roads disappear under heavy
returns. Give the warning outlines an insertion point and they end up under the
radar in exactly the conditions where they matter most.

## Warnings

Four products, each with its own fill and outline layer pair. Dash is a
layer-level property, not a data property — see gotcha 2.

| Product                     | Fill    | Opacity | Outline | Width | Style  |
|-----------------------------|---------|---------|---------|-------|--------|
| Tornado Warning             | #ff0000 | 0.35    | #ff0000 | 3     | solid  |
| Tornado Watch               | #ff9900 | 0.25    | #ff9900 | 2.5   | dashed |
| Severe Thunderstorm Warning | #ffff00 | 0.35    | #ffff00 | 3     | solid  |
| Severe Thunderstorm Watch   | #ffff00 | 0.20    | #ffff00 | 2     | dashed |

Each toggle flips layer visibility rather than refiltering, so it costs nothing
and does not touch the data.

## Chaser markers

Three states:

```
streaming   red dome, pulsing ring   a live stream is matched to this chaser
active      green dome               position within the last 30 minutes
stale       grey dome                older than that, or absent from the feed
```

A single inline SVG per marker: dark sphere, coloured dome, and a pointer that
rotates to the reported heading. The pointer is omitted when heading is null.

The 30-minute staleness threshold and the 30-second poll interval are
independent. Lengthening the poll past 30 minutes would make every chaser
permanently stale.

## What was verified

```
radar tiles available to zoom 7 on the free tier; maxNativeZoom 6 with
  256px tiles renders clean at every map zoom
search.list costs 100 quota units per call, videos.list costs 1
poll intervals: positions 30s, live-stream check 60s, radar 5 min,
  warnings 3 min
staleness threshold 30 min
```

No load-time figures. They were never measured, and an invented number is worse
than none.

## Things that cost time

1. **Radar tiles stop at zoom 7 on the free tier.** Past that the provider
   returns a "Zoom Level Not Supported" placeholder image, tiled across the
   whole viewport. It is not an error and nothing appears in the console. Values
   of 11, 12 and 13 were tried before the cap was identified. Tile size and
   `maxNativeZoom` move together: 256px tiles with `maxNativeZoom: 6` is the
   working pair. An earlier attempt used 512px tiles, which need
   `zoomOffset: -1`, which shifts the effective cap by one and made the symptom
   look inconsistent between attempts.

2. **Data-driven `line-dasharray` is not supported.** A `['get','dash']`
   expression does not throw. The layer is accepted, the source has data, and
   nothing ever draws. The fill layer beside it rendered fine, which made the
   data look correct and sent the search in the wrong direction for a long time.
   Fix is one layer per product with the dash set at layer level.

3. **The position feed refuses browsers.** Direct requests return 403. Two
   public CORS proxies were tried against it and both were blocked as well. A
   server-side proxy is the only route that works; the time spent on proxy
   services was wasted.

4. **Raster label tiles cannot be restyled.** Under the raster basemap labels
   are pixels in a PNG — size and typeface fixed, only a filter over the whole
   tile available. An attempt to hide them and draw city labels from a
   populated-places dataset instead was abandoned: a hardcoded list missed
   towns, and the full dataset drew labels on top of the ones already in the
   tile. This is the reason for the vector migration, not a preference for it.

5. **A road colour is two layers.** Vector styles draw a casing beneath each
   road fill. Setting only the fill leaves the old casing colour showing as a
   fringe along every road. Casings are set to a darkened version of the chosen
   colour.

6. **Quota arithmetic sets the poll interval.** `search.list` at 100 units
   against a default 10,000 unit daily allocation is 99 calls a day, or one
   roughly every 14 minutes, which is useless for catching a stream going live.
   A 60-second interval needs about 144,000 units a day and therefore an
   approved increase. Check the arithmetic before changing `YT_REFRESH`.

## Not done

**Live-stream detection has never run against a real broadcast.** The rendering
half was exercised by forcing a past video into the roster — marker turns red,
popup opens, player embeds. The detection half is untested: `search.list`
returning an item that is genuinely live, the marker flipping without a reload,
and the marker clearing itself when the broadcast ends.

**The quota increase is not approved.** Until it is, a 60-second interval
exhausts the daily allocation in roughly 100 minutes and the detector returns
nothing for the rest of the day.

**Warning rendering has not been seen during an outbreak.** Checked against a
handful of concurrent polygons. How several dozen overlapping polygons read,
and whether the fill opacities still work when watches and warnings stack, is
unknown.

**`chaser.html` and `admin.html` do not work.** They are left over from an
earlier build with a Node backend that was never deployed. They post to API
routes that do not exist. They are still in the repository because the markup is
worth keeping if that direction is ever picked back up.

**Endpoints are hardcoded in the page.** Both proxy URLs sit in the source. They
should come from config the host supplies, so the same file can run against a
staging proxy without an edit.

**No position history.** Only the current point is drawn — no breadcrumb trail,
no way to see where a chaser has been.

**Viewer counts are as stale as the last poll.** Taken from the live streaming
details on the video, not smoothed between polls, so the number steps rather
than counts.

**No mobile layout.** The settings panel, chaser table and status bar all assume
a desktop viewport. On a phone they overlap the map and each other.

**Popup dragging is mouse only.** Touch drag existed in the raster build and was
not carried across during the rewrite.

**The roster is hardcoded.** Adding a chaser is a code edit and a redeploy, in
two places: the page roster and the proxy configuration. There is no admin
interface — see `admin.html` above for the direction that was abandoned.

**A failing proxy shows one line of status text.** No retry indication, no
distinction between a misconfigured endpoint and a downed one, no error detail.

## Attribution

Watch and warning polygons come from the National Weather Service public alerts
API. Radar imagery comes from a third-party mosaic provider. Chaser positions
come from SpotterNetwork. Basemap tiles are built from OpenStreetMap data.
Rendering is MapLibre GL JS; playback uses hls.js for direct streams and the
YouTube embedded player for YouTube streams.

This is not an official source of weather information. It displays NWS products
but is not operated by or endorsed by the National Weather Service.

## Licence

All rights reserved, source-visible. Read it, study it, quote short excerpts
with attribution. Using it in a project needs written permission first, and
permission comes with attribution terms. See LICENSE for the full text.

The public data this code reads is not covered by that. NWS products are public
domain and OpenStreetMap data has its own licence; the terms here apply only to
the code.
