# GeoStamp — Geotag Your Event Photos

> A modern, browser-based geotagging tool that burns in professional GPS map overlays and writes true EXIF GPS & timestamp metadata directly into your photos — 100% client-side.

---

## Table of Contents

1. [Overview](#overview)
2. [Key Features](#key-features)
3. [How GeoStamp Works](#how-geostamp-works)
4. [The Science of Geotagging & Metadata](#the-science-of-geotagging--metadata)
   - [What is EXIF Metadata?](#what-is-exif-metadata)
   - [How GPS Data is Stored in EXIF](#how-gps-data-is-stored-in-exif)
   - [The Decimal to DMS Rational Conversion](#the-decimal-to-dms-rational-conversion)
5. [Accuracy of Geotags & Location Services](#accuracy-of-geotags--location-services)
   - [GNSS Satellites vs. A-GPS](#gnss-satellites-vs-a-gps)
   - [Expected Accuracy Benchmarks](#expected-accuracy-benchmarks)
   - [Factors Affecting Precision](#factors-affecting-precision)
6. [Why Burned-in Overlays + EXIF Metadata?](#why-burned-in-overlays--exif-metadata)
7. [User Guide & How to Use](#user-guide--how-to-use)
8. [Customization & Typography Controls](#customization--typography-controls)
9. [Privacy & Security](#privacy--security)
10. [Tech Stack](#tech-stack)
11. [Author & License](#author--license)

---

## Overview

Modern event documentation, community service activities (like NSS, NCC, CSR initiatives), fieldwork reports, property inspections, and research surveys often mandate **geotagged proof of attendance**. 

However, photos taken with standard cameras or photos received via messaging apps often lose their location data. **GeoStamp** solves this problem by allowing users to:
- Take a photo directly on mobile or upload any existing JPEG/PNG.
- Set or capture real-time GPS coordinates and address information.
- Render a customizable, high-resolution **GPS Map Overlay** (including an interactive mini-map with Leaflet pins and zoom indicators).
- Embed authentic **EXIF GPS tags and timestamps** directly into the binary header of the downloaded image.

---

## Key Features

- **Live Dynamic Overlay**: Automatically composites a real map card, place name, full address, coordinates, and formatted timestamp onto the photo.
- **True EXIF GPS Metadata Embedding**: Uses `piexifjs` to write binary `GPSIFD` tags (`GPSLatitude`, `GPSLongitude`, `GPSLatitudeRef`, `GPSLongitudeRef`, `DateTimeOriginal`, `Software`) into the JPEG image structure.
- **Custom Typography & Sizing**: Real-time sliders to customize Header font size, Address font size, and Coordinate/Date font sizes on the image.
- **Interactive Mini-Map**:
  - Choose between **Street Map** and **Satellite Map** imagery.
  - Sizable map card with customizable zoom levels (Zoom 14 to 19).
  - Includes Leaflet-style zoom controls (`+` / `−`), royal blue location marker, and attribution banner.
  - Option to toggle the mini-map on or off.
- **Overlay Styling Controls**: Adjustable dark gradient opacity (20% to 100%), overall overlay scale (60% to 160%), and custom text color picker.
- **Reverse Geocoding**: Auto-fills city and street address from GPS coordinates using OpenStreetMap Nominatim.
- **100% Client-Side & Private**: Your photos are processed exclusively inside your web browser. No files or personal data are ever uploaded to any backend server.
- **Zero Install Needed**: Runs instantly on any modern browser across mobile and desktop.

---

## How GeoStamp Works

GeoStamp operates in four synchronized pipeline stages:

```
[ Photo Upload / Camera ] 
          │
          ▼
[ Geolocation / Coordinates ] ──► [ Reverse Geocoding (Nominatim) ]
          │
          ▼
[ HTML5 Canvas Engine ] ◄── [ Map Tile Compositing (OSM / ESRI) ]
          │                   [ User Customization Sliders ]
          ▼
[ Binary EXIF Encoder ] ──► [ Download Geotagged JPEG with GPS Tags ]
```

1. **Image Ingestion**: Loads the image locally into an in-memory `HTMLImageElement` without compression artifacts.
2. **Location Capture & Map Generation**:
   - Acquires device coordinates via the HTML5 Geolocation API (`navigator.geolocation.getCurrentPosition`).
   - Translates latitude and longitude into Web Mercator tile indices at the chosen zoom level ($z$).
   - Fetches and stitches the surrounding 9 map tiles into an off-screen canvas, cropping the exact coordinate center.
3. **Canvas Compositing**:
   - Renders a multi-stop linear gradient at the base of the image to ensure high contrast and readability.
   - Draws the mini-map card, Leaflet controls, center pin, and wrapped text typography with mathematical scaling proportional to the source image resolution.
4. **Binary EXIF Injection**:
   - Converts decimal degrees into rational Degrees, Minutes, and Seconds (DMS) tuples.
   - Constructs binary IFD (Image File Directory) blocks: `0th IFD`, `Exif IFD`, and `GPS IFD`.
   - Injects the EXIF payload into the JPEG binary stream (`APP1` marker segment) and triggers the download.

---

## The Science of Geotagging & Metadata

### What is EXIF Metadata?

**EXIF** stands for **Exchangeable Image File Format**. It is a standardized metadata specification created by JEITA (Japan Electronics and Information Technology Industries Association) for digital still cameras.

When a digital camera or phone takes a picture, it writes technical data alongside the raw pixel information in the file header:
- Camera make, model, lens parameters
- Exposure time, F-number, ISO speed
- Date and time of capture
- GPS location parameters

### How GPS Data is Stored in EXIF

GPS information is stored in a dedicated subgroup called the **GPS IFD (Image File Directory)**. The standard tags include:

| Tag Name | Tag ID | Description | Example Format |
|---|---|---|---|
| `GPSLatitudeRef` | `0x0001` | North or South latitude | `'N'` or `'S'` |
| `GPSLatitude` | `0x0002` | Degrees, Minutes, Seconds | `[[28, 1], [37, 1], [4635, 100]]` |
| `GPSLongitudeRef` | `0x0003` | East or West longitude | `'E'` or `'W'` |
| `GPSLongitude` | `0x0004` | Degrees, Minutes, Seconds | `[[77, 1], [22, 1], [1742, 100]]` |
| `DateTimeOriginal` | `0x9003` | Timestamp when photo was taken | `"YYYY:MM:DD HH:MM:SS"` |

### The Decimal to DMS Rational Conversion

EXIF standard does not allow raw floating-point numbers (e.g. `28.629541`). Instead, GPS coordinates must be converted into three unsigned rational numbers (fractions) representing:
1. **Degrees ($D$)**: Integer degrees $\lfloor \text{deg} \rfloor$.
2. **Minutes ($M$)**: Integer minutes $\lfloor (\text{deg} - D) \times 60 \rfloor$.
3. **Seconds ($S$)**: Fractional seconds $((\text{deg} - D) \times 60 - M) \times 60$.

In GeoStamp, this conversion is implemented as:

$$\text{Decimal: } 28.629541^\circ \implies 28^\circ\ 37'\ 46.35'' \implies \left[\left[\frac{28}{1}\right], \left[\frac{37}{1}\right], \left[\frac{4635}{100}\right]\right]$$

When viewing the exported JPEG in file properties (Windows, macOS Finder, Google Photos, or Adobe Lightroom), the operating system reads these rational fractions and displays the exact map pin.

---

## Accuracy of Geotags & Location Services

### GNSS Satellites vs. A-GPS

Smartphones and modern devices do not solely rely on direct orbital satellite signals. They utilize **Assisted GPS (A-GPS)**, which combines multiple positioning technologies:

```
┌─────────────────────────────────────────────────────────────┐
│                     Positioning Methods                     │
├──────────────────────────┬──────────────────────────────────┤
│ 1. GNSS Satellites       │ GPS (US), GLONASS (RU),          │
│    (Direct line-of-sight)│ Galileo (EU), BeiDou (CN), NavIC │
├──────────────────────────┼──────────────────────────────────┤
│ 2. Wi-Fi Positioning     │ Nearby Wi-Fi BSSID database      │
│    (Indoor / Urban)      │ lookup                           │
├──────────────────────────┼──────────────────────────────────┤
│ 3. Cellular Towers       │ Cell ID and signal triangulation │
│    (Base station signals)│                                  │
└──────────────────────────┴──────────────────────────────────┘
```

### Expected Accuracy Benchmarks

| Environment | Primary Technology Used | Typical Accuracy Range |
|---|---|---|
| **Open Sky / Outdoors** | GNSS Satellite Lock | **$\pm$ 3 to 5 meters** (High Precision) |
| **Dense Urban / Between Tall Buildings** | A-GPS + Multi-path Reflection | **$\pm$ 10 to 30 meters** (Moderate) |
| **Indoors / Basements** | Wi-Fi Triangulation + Cellular | **$\pm$ 20 to 100+ meters** (Approximate) |
| **Cellular Tower Only** | Cell Tower ID | **$\pm$ 500m to 2 km** (Low) |

### Factors Affecting Precision

1. **Dilution of Precision (DOP)**: Geometric geometry of visible satellites in the sky. When satellites are spread out across the hemisphere, accuracy is highest.
2. **Urban Canyons & Reflections**: Tall concrete structures can bounce satellite signals before they reach the antenna, introducing slight transit delays.
3. **Cached / Stale Device Location**: If the device was recently sleeping or in power-saving mode, the first location returned may be a cached position until fresh satellite ephemeris data is downloaded.

*GeoStamp allows users to fine-tune both the exact decimal coordinates and street address manually to ensure 100% precision even if the device was indoors during the photo capture.*

---

## Why Burned-in Overlays + EXIF Metadata?

| Feature | Visual Burned-in Overlay | EXIF Metadata Only | GeoStamp (Both) |
|---|---|---|---|
| **Visible in Printouts / PDFs** | Yes | No | **Yes** |
| **Visible in Social Media & Chat Apps** | Yes (WhatsApp/Instagram strip EXIF) | Lost upon upload | **Retained Visually** |
| **Indexed by Photo Software (Lightroom/Google Photos)** | No (Just pixels) | Yes (Reads EXIF) | **Yes** |
| **Tamper-Evident for Official Reports** | High visibility | Hidden in header | **Complete Dual Proof** |

Most social media platforms (WhatsApp, Instagram, Facebook, X) strip EXIF metadata during compression for user privacy. By **both** burning in a clear visual stamp and embedding the underlying EXIF data, GeoStamp ensures your photos remain verifiable across all platforms, reports, and archives.

---

## User Guide & How to Use

1. **Upload or Capture Photo**:
   - Drag and drop your image into the dropzone, click to browse, or tap on mobile to open the camera directly.
2. **Set Location**:
   - Click **"Use my current location"** to auto-detect coordinates and reverse-geocode your address.
   - Or manually enter Latitude, Longitude, City, and Address.
   - Choose your preferred map style (**Street Map** or **Satellite**).
3. **Set Date & Time**:
   - Defaults to the current timestamp. Modify the date and time freely if geotagging an earlier event.
4. **Customize Appearance**:
   - Use the sliders in **Section 4** to adjust font sizes, overlay scale, opacity, or mini-map dimensions in real-time.
5. **Download**:
   - Choose **JPEG (with GPS metadata)** for full EXIF embedding or **PNG (lossless)**.
   - Click **"Download geotagged photo"**.

---

## Customization & Typography Controls

GeoStamp includes full real-time customization sliders:

- **Place / Header Font Size**: `14px` to `38px`
- **Address Font Size**: `10px` to `28px`
- **Coordinates & Date Font Size**: `10px` to `26px`
- **Overall Overlay Size**: `60%` to `160%` (scales the entire overlay proportionally)
- **Overlay Dark Opacity**: `20%` to `100%` (controls background transparency)
- **Text Color Picker**: Customize font color (default pure white `#ffffff`)
- **Mini-Map Controls**: Toggle visibility, adjust map size (`50%` to `160%`), and set map zoom level (`Level 14` to `Level 19`).
- **1-Click Reset**: Restore all customizations to default settings anytime.

---

## Privacy & Security

- **Zero Cloud Uploads**: GeoStamp runs entirely in your local browser runtime.
- **No Analytics / No Tracking**: Photos never leave your device.
- **Offline Capable**: Once the page is loaded, core canvas compositing works even without an active internet connection.

---

## Tech Stack

- **Core**: Vanilla HTML5, JavaScript (ES6+), Canvas API
- **Styling**: Vanilla CSS with custom dark mode design system
- **Metadata Engine**: [`piexifjs`](https://github.com/hMatoba/piexifjs) (EXIF read/write library)
- **Mapping & Tiles**: ESRI World Street Map, OpenStreetMap tiles, ESRI World Imagery
- **Geocoding**: OpenStreetMap Nominatim API

---

## Author & License

Made by [Harsimran Singh](https://github.com/Harsimran-singh-7765).

Distributed under the [MIT License](LICENSE). Contributions, feedback, and stars are welcome!
