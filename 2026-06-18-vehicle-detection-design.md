# HK Traffic Vehicle Detection Web App

## Overview

A static web app that displays a Hong Kong map with 995 traffic camera locations. Users select a camera to view its live snapshot with real-time YOLO-based vehicle detection (bounding boxes + count).

## Data Source

- **Camera list API**: ArcGIS FeatureServer (Hong Kong Transport Department)
  - URL: `https://portal.csdi.gov.hk/server/rest/services/common/td_rcd_1638952287148_39267/FeatureServer/0/query?f=json&where=1%3D1&outFields=*&resultRecordCount=1000`
  - Returns 995 features with attributes: `OBJECTID`, `key_`, `region`, `district`, `description`, `url`
  - Each `url` field contains a JPG snapshot: `https://tdcctv.data.one.gov.hk/{key_}.JPG`
  - Geometry: point features in HK Grid (EPSG:2326), convertible to lat/lng

## Architecture

**Multi-file static web app**, no build tools.

### Project Structure

```
traffic-vehicle-detection/
├── index.html          # Entry point with layout shell
├── js/
│   ├── app.js          # Main app state, UI controller, React root
│   ├── map.js          # Leaflet map init, camera markers, marker interactions
│   └── detector.js     # ONNX Runtime session, YOLO preprocessing/inference/postprocessing, canvas drawing
├── css/
│   └── style.css       # Custom styles (panel sizing, canvas overlay)
└── assets/
    └── favicon.png     # App icon (from web-app-generator assets)
```

### Layout

Two-panel layout:
- **Left (60%)**: Leaflet map with camera cluster markers (color-coded by region)
- **Right (40%)**: Detection panel containing:
  - Camera info header (description, region, district)
  - Snapshot canvas (image with bounding box overlay)
  - Vehicle count badge
  - Refresh controls (auto-refresh toggle + interval selector + manual refresh button)
  - Status indicator (loading, detecting, idle)

### Data Flow

1. App loads -> fetch all 995 cameras from FeatureServer API
2. Render markers on Leaflet map (clustered for performance)
3. User clicks a marker -> select camera
4. Load snapshot image from `tdcctv.data.one.gov.hk/{key_}.JPG`
5. Draw image onto `<canvas>`, resize to model input size (640x640)
6. Run YOLO ONNX inference via ONNX Runtime Web
7. Apply NMS (Non-Maximum Suppression) postprocessing
8. Draw bounding boxes + class labels on canvas overlay
9. Update vehicle count
10. If auto-refresh enabled: schedule next fetch+detect cycle (default 15s)

## Tech Stack

| Concern | Technology | Source |
|---------|-----------|--------|
| UI framework | React 18 | CDN (unpkg) production build |
| Styling | Tailwind CSS v3 | CDN (cdn.tailwindcss.com) |
| Icons | Lucide | CDN (unpkg) |
| Map | Leaflet 1.9 | CDN (unpkg) |
| ML inference | ONNX Runtime Web | CDN (jsdelivr) |
| Object detection | YOLOv8n (nano) ONNX | CDN (configurable URL, default from HF/Ultralytics) |
| Transpilation | Babel Standalone | CDN (unpkg) |

## Detection Engine: ONNX Runtime Web + YOLOv8n

### Model

YOLOv8n (nano) in ONNX format:
- Input: 640x640 RGB image (normalized to [0,1])
- Output: 84 x N tensor (84 = 4 bbox coordinates + 80 COCO class probabilities)
- Model size: ~6MB
- Default model URL: `https://huggingface.co/Xenova/yolov8n/resolve/main/onnx/model_quantized.onnx`
- Model URL is configurable in app code

### Preprocessing

1. Load snapshot image to canvas
2. Resize to 640x640 maintaining aspect ratio + pad with gray
3. Extract pixel data, normalize to [0,1]
4. Reshape to [1, 3, 640, 640] Float32Array tensor

### Postprocessing

1. Parse model output tensor
2. Filter detections by confidence threshold (0.25)
3. Filter to vehicle classes only: car(2), bus(5), truck(7), motorcycle(3), bicycle(1)
4. Apply NMS with IoU threshold 0.45
5. Scale bounding boxes back to original image coordinates

### Vehicle Classes Detected

| COCO Class ID | Name | Display Label |
|---------------|------|---------------|
| 1 | bicycle | Bicycle |
| 2 | car | Car |
| 3 | motorcycle | Motorcycle |
| 5 | bus | Bus |
| 7 | truck | Truck |

## CORS Strategy

Both the ArcGIS API and image CDN may have CORS restrictions:

1. **ArcGIS API**: Supports CORS natively (test shows it works with `f=json`)
2. **Snapshot images**: `tdcctv.data.one.gov.hk` - if CORS blocked, use a public CORS proxy:
   - Fallback: `https://corsproxy.io/?` prefix
   - Alternative: `https://api.allorigins.win/raw?url=` prefix

## Auto-Refresh

- Toggle button to enable/disable
- Interval selector: 10s, 15s (default), 30s, 60s
- Timer resets when user selects a different camera
- Visual indicator shows countdown to next refresh
- Manual "Refresh" button always available

## Error Handling

- Map load failure: show retry button
- Camera API failure: show error toast with retry
- Image load failure (CORS/network): show placeholder + error message
- Model load failure: show error with retry, fallback message
- Inference failure: log error, show last known result

## Responsive Design

- Desktop: side-by-side map (60%) + panel (40%)
- Tablet: map on top (50vh), panel below
- Mobile: full-width stacked, map collapsible
