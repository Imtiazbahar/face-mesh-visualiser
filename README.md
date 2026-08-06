# Face Mesh Visualiser

A live wireframe that tracks your face using the front camera, with a travelling
"cuttlefish" colour flash that ripples through the low-poly mesh when a face is
recognised. Runs entirely in the browser — no video ever leaves the device.

Built with [MediaPipe Face Landmarker](https://developers.google.com/mediapipe/solutions/vision/face_landmarker).

## Controls
- **Camera** — show/hide the live video behind the mesh
- **Detail** — Simple / Medium / Detailed polygon density
- **Colour** — cycle the base palette
- **Glow** — toggle the neon bloom

## Run locally
Open over a secure origin (camera needs one). Simplest:

```bash
python3 -m http.server 8777
```

then visit `http://localhost:8777`.
