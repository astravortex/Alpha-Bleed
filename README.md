# Alpha Bleed 1.0 — User Guide

**Alpha Bleed** is a Photon Studio plugin for extending the RGB colour of visible pixels into transparent or otherwise masked pixels without changing the image's alpha channel. It is useful for preparing sprites, cut-out artwork, texture maps, and other raster assets for texture filtering, mipmapping, scaling, and compositing.

The plugin supports **8-bit and 16-bit RGB documents** and creates a new output layer. The selected source layer is not modified.

---

## 1. Quick start

1. Open an RGB document in Photon Studio.
2. Make sure no active selection is present. Deselect any marquee, lasso, or other active selection.
3. In the Layers panel, select **exactly one raster/pixel layer** to process.
4. Open **Filter → Alpha Bleed…**.
5. Choose the mask channel, Threshold, and Fill Radius.
6. Click **Apply Alpha Bleed**.
7. Review the new `Alpha Bleed — [source layer name]` output layer (or the set of tile layers for a large image).
8. To export an RGBA PNG while preserving RGB values under fully transparent pixels, click **Export PNG (preserve hidden RGB)** before closing or reloading the plugin panel.

The original source layer remains unchanged. Alpha Bleed creates a separate result layer above the source.

## 2. What Alpha Bleed does

Transparent pixels often retain RGB values that do not match the edge colours of the visible artwork. When an image is filtered, resized, or sampled into mipmaps, those unrelated hidden colours can produce fringes, dark outlines, or colour halos around otherwise clean edges.

Alpha Bleed propagates nearby RGB colour into target pixels while leaving their alpha values untouched:

- Pixels whose selected mask-channel value is at or above the Threshold are treated as **source pixels**.
- Pixels below the Threshold are potential **target pixels**.
- The plugin spreads colour from source pixels into connected target pixels using an 8-neighbour breadth-first fill.
- A newly reached pixel receives the floor-average RGB value of its currently solid 8-neighbour pixels.
- The pixel's original alpha value is preserved exactly.

The result changes RGB data, not the silhouette or transparency of the artwork.

## 3. Settings

### Mask channel

The selected channel determines which pixels are considered source pixels.

| Option | Meaning |
|---|---|
| **A — Alpha** | Uses the alpha channel. This is the default and is usually the best choice for transparent artwork. |
| **R — Red** | Uses the red channel as the source mask. |
| **G — Green** | Uses the green channel as the source mask. |
| **B — Blue** | Uses the blue channel as the source mask. |

The selected channel is used only to identify source and target pixels. The plugin fills the RGB channels together; it does not change the alpha channel.

### Threshold

Threshold is the minimum value, inclusive, for a pixel to qualify as a source pixel:

`source pixel = selected channel value >= Threshold`

The valid range depends on the source layer's bit depth:

| Source depth | Valid Threshold range | Maximum channel value |
|---|---:|---:|
| 8-bit | 1–255 | 255 |
| 16-bit | 1–65,535 | 65,535 |

**Default: 200** for both bit depths.

Threshold uses the source layer's native numeric scale. The same numeric value does not represent the same relative level in 8-bit and 16-bit images. For example, 8-bit Threshold **200** is approximately equivalent to 16-bit Threshold **51,400** when matching the same normalized channel level.

Use the slider or type a value into the numeric field. The two controls stay synchronized. Values are saved in the plugin settings.

### Fill Radius

Fill Radius limits how far colour can propagate from the source boundary.

| Value | Behaviour |
|---|---|
| **0** | No radius limit. Fill the entire connected target area. This is the default. |
| **1** | Fill the first neighbouring propagation ring. |
| **2** | Allow up to two propagation rings. |
| **Higher values** | Allow correspondingly more propagation rings, up to the plugin limit. |

The maximum Fill Radius is **16,384**. Use a limited radius when only a narrow colour extension is needed. Use `0` when the entire connected transparent region should receive edge colour.

### Reset

**Reset** restores the default settings:

- Mask channel: `A — Alpha`
- Threshold: `200`
- Fill Radius: `0`

## 4. Image dimensions and aspect ratio

### Is a particular aspect ratio required?

**No. Alpha Bleed does not require a specific aspect ratio.** Square, portrait, landscape, widescreen, panoramic, and custom proportions can all be processed. The aspect ratio itself is not used by the algorithm.

Examples of valid proportions include:

- 1:1 — square textures and icons
- 4:3 or 3:2 — landscape artwork
- 16:9 or 21:9 — widescreen artwork
- 2:3 or 3:4 — portrait artwork
- Custom texture-atlas or sprite-sheet proportions

What matters is the pixel dimensions, the number of pixels in the selected layer, and the Photon transfer limits described below. A wide image is not inherently less suitable than a square image; it simply needs to fit the same dimension and transfer constraints.

### Dimension limits

The plugin's document geometry limit is:

- Maximum width: **16,384 pixels**
- Maximum height: **16,384 pixels**
- Maximum document area: **268,435,456 pixels**

These are hard geometry limits, not a guarantee that a full-canvas layer of that size can be read in one operation. The current Photon SDK limits a single RGBA PixelTransfer to **256 MiB**. Alpha Bleed reads the selected layer as one source transfer, so the selected layer's actual pixel bounds must also fit that transfer limit.

| Source format | Bytes per RGBA pixel | Maximum pixels in one 256 MiB transfer |
|---|---:|---:|
| RGBA8 | 4 bytes | 67,108,864 pixels |
| RGBA16 | 8 bytes | 33,554,432 pixels |

Examples for a selected layer that covers the full listed area:

| Layer dimensions | RGBA8 source read | RGBA16 source read |
|---|---:|---:|
| 4096 × 4096 | 64 MiB — within limit | 128 MiB — within limit |
| 8192 × 4096 | 128 MiB — within limit | 256 MiB — at the limit |
| 8192 × 8192 | 256 MiB — at the limit | 512 MiB — exceeds the limit |
| 16,384 × 16,384 | 1 GiB — exceeds the limit | 2 GiB — exceeds the limit |

The **256 MiB limit applies to the selected layer's input transfer**, not just to the output tiles. Splitting the output into tiles does not bypass this source-read limit. If the selected layer exceeds it, process a smaller layer or a lower-resolution copy.

For 16-bit images, an additional **512 MiB safety limit** applies when Alpha Bleed needs to reconstruct a full-canvas output buffer from a layer that does not already occupy the full canvas. Available system memory can impose a lower practical limit.

### Recommended image preparation

For predictable results:

- Use a raster/pixel layer containing the artwork you want to process.
- Keep the layer's canvas position and bounds as intended for the final texture.
- Use transparency around the artwork when using the default Alpha mask channel.
- Make sure the selected source pixels contain the RGB colours you want to extend.
- Remove any active selection before running the plugin.

If the source layer extends beyond the document canvas, the output image is clipped to the document canvas.

## 5. Supported documents and layers

The current build expects:

- An open Photon Studio document in **RGB** colour mode.
- Exactly **one selected raster/pixel layer**.
- No active document selection.
- Four-channel **RGBA8** or **RGBA16** full-range integer pixel data.

Additional colour requirements:

- **8-bit:** the layer must report an sRGB colour space or profile.
- **16-bit:** the document must be RGB. The pixel transfer must be full-range integer data.

The plugin does not process multiple selected layers in one operation. Text, vector, shape, adjustment, and other non-raster layers should be rasterized before use.

## 6. Output layer behaviour

Alpha Bleed creates new output content and leaves the selected source layer unchanged.

### Single-layer output

- 8-bit images up to **4096 × 4096** use Photon's existing single-image layer-creation path.
- 16-bit images up to **4096 × 4096** use a native-depth direct pixel write into one output layer.

### Tiled output for larger images

When either image dimension exceeds 4096 pixels, the output is written as aligned tiles:

- Tile width: up to **4096 pixels**
- Tile height: up to **2048 pixels**
- Maximum tile area: **8,388,608 pixels**
- Each tile is created as a separate raster layer.
- Each tile creation is a separate undo step.

For example, an 8192 × 8192 output is arranged as 8 tiles (2 columns × 4 rows). A 16,384 × 16,384 output would require 32 tiles, provided the source-read and memory limits are also satisfied.

Do not move individual tile layers independently if they are intended to remain aligned. Keep all output tiles together in the layer stack. You may group them using Photon Studio's normal layer tools after processing.

## 7. PNG export

Use **Export PNG (preserve hidden RGB)** to export the exact processed RGBA buffer retained by the plugin.

The custom PNG export:

- Preserves the source's 8-bit or 16-bit channel depth.
- Writes RGBA PNG (colour type 6).
- Preserves RGB values beneath fully transparent pixels (`Alpha = 0`).
- Preserves alpha values without changing them during the bleed operation.
- Uses PNG compression when supported by the Photon runtime.

### Export limits and colour profiles

Photon's current binary file writer has a **64 MiB maximum per file write**. The compressed PNG must fit within this limit. If it does not, the processed layer remains in the document; use Photon's native document/image export or reduce the image dimensions.

For larger PNG files, the Photon runtime must also provide the native DEFLATE/`CompressionStream` capability. If this capability is unavailable, large PNG export is not available in that runtime; update Photon Studio or use its native export.

The custom PNG exporter preserves pixel values and bit depth, but it does **not embed the source ICC profile**. If the source uses a non-sRGB profile and colour-managed interchange is important, use Photon's native export so the required profile can be embedded or handled by Photon.

For the most reliable export of large tiled results, export during the same plugin panel session in which Alpha Bleed was run. The exact processed pixel buffer is retained in panel memory for this export. Keep the panel session open until the export has completed.

## 8. Processing and memory notes

RGBA pixel data is memory-intensive:

- RGBA8 uses 4 bytes per pixel.
- RGBA16 uses 8 bytes per pixel.
- 8192 × 8192 RGBA8 input is 256 MiB before processing overhead.
- 8192 × 8192 RGBA16 input is 512 MiB and exceeds the current single-transfer limit.

The fill algorithm also needs a source-mask bitset, active BFS frontiers, output references, and—in the case of PNG export—compression buffers. Real memory use is therefore higher than the raw RGBA size. Close unnecessary documents and applications before processing very large assets. Avoid running multiple large image operations at once.

The algorithm periodically yields to the host while scanning and filling so the panel can update progress and respond to cancellation.

## 9. Troubleshooting

### “An RGB document is required”

Convert the document to RGB colour mode in Photon Studio, then run Alpha Bleed again.

### “Select exactly one raster layer”

Select one raster/pixel layer in the Layers panel. Deselect any additional selected layers. Rasterize text, vector, shape, or other non-raster content first.

### “Deselect the active selection before running”

Clear the current selection in Photon Studio and retry. Alpha Bleed processes the full selected layer, not the current selection area.

### “No source pixels were found”

No pixel in the selected mask channel met the Threshold. Try lowering Threshold or choosing another mask channel. For transparent artwork, start with `A — Alpha`.

### “Threshold exceeds the bit-depth maximum”

Use a value from 1–255 for 8-bit layers or 1–65,535 for 16-bit layers. Remember that 16-bit uses a different numeric scale; approximately 51,400 in 16-bit corresponds to 200 in 8-bit.

### “PixelTransfer exceeds 256 MiB” or “Invalid or oversized RGBA image”

The selected layer is too large for one source read in the current Photon SDK. Reduce the selected layer's pixel bounds, split the asset into smaller layers and process them separately, or use a lower-resolution copy. The output-tile system does not remove the source-read limit.

### “No fill was created” or “source pixels already have black RGB”

The selected mask may not identify the intended donor pixels, or the RGB values of the source pixels may be entirely black. Verify that the correct layer and mask channel are selected and that the source pixels contain the colours you want to propagate.

### PNG export reports that the file is too large

The compressed PNG exceeds Photon's 64 MiB binary write limit. The processed result remains in the document. Use Photon's native export or reduce the image dimensions.

### PNG colours look different in another application

The custom exporter preserves numeric pixel values and bit depth but does not embed the source ICC profile. Use Photon's native export for profile-aware output.

## 10. Technical specifications

| Specification | Value |
|---|---|
| Plugin name | Alpha Bleed |
| Release | 1.0 |
| Manifest version | 1.0.0 |
| Photon SDK manifest target | SDK v1 |
| Document colour mode | RGB |
| Supported pixel formats | RGBA8, RGBA16 (full-range integer) |
| Supported mask channels | A, R, G, B |
| Default mask channel | A — Alpha |
| Threshold, 8-bit | 1–255 |
| Threshold, 16-bit | 1–65,535 |
| Default Threshold | 200 |
| Fill Radius | 0–16,384 |
| Default Fill Radius | 0 (unlimited connected fill) |
| Propagation | 8-neighbour breadth-first fill |
| RGB fill rule | Floor-average of currently solid 8-neighbour RGB values |
| Alpha handling | Original alpha values are preserved |
| Maximum document dimension | 16,384 px per side |
| Maximum document pixel area | 268,435,456 pixels |
| Maximum input PixelTransfer | 256 MiB |
| RGBA8 maximum pixels per input transfer | 67,108,864 |
| RGBA16 maximum pixels per input transfer | 33,554,432 |
| Maximum reconstructed 16-bit canvas buffer | 512 MiB |
| Output tile size | Up to 4096 × 2048 px |
| PNG output | RGBA PNG, 8-bit or 16-bit |
| Maximum Photon binary file write | 64 MiB |
| Source layer | Not modified; separate output layer(s) are created |

## 11. Recommended workflow summary

For most transparent artwork, start with:

- **Mask channel:** `A — Alpha`
- **Threshold:** `200` for 8-bit; approximately `51,400` for a similar normalized threshold in 16-bit
- **Fill Radius:** `0` for the full connected transparent region, or a small positive value for limited edge padding

Always check the result over both a light and a dark background. For production texture exports, confirm that the PNG bit depth is correct and that RGB values beneath transparent pixels have been retained. For colour-managed delivery, use Photon's native export when ICC profile embedding is required.

---

**Alpha Bleed 1.0**  
User documentation for the Photon Studio plugin.
