# Orion's Stabilizer

A Python-based video stabilisation tool built from scratch using optical flow and affine transform estimation. No wrappers, no black boxes — every stage of the pipeline is implemented directly using OpenCV and NumPy.

https://github.com/Onion-was-here/orions-stabilizer/assets/demo.mp4

---

## Demo

> Original footage on the left, stabilised output on the right.

_(Replace this line with your side-by-side video or GIF once rendered)_

---

## How it works

The stabiliser runs in three stages.

### Stage 1 — Feature tracking

For each consecutive frame pair, the algorithm detects up to 300 corner features using the Shi-Tomasi corner detector (`cv.goodFeaturesToTrack`). These corners are re-detected every frame rather than carried forward, which prevents tracking drift over long clips.

Once corners are found, Lucas-Kanade optical flow (`cv.calcOpticalFlowPyrLK`) tracks where each corner moved in the next frame. LK operates on an image pyramid — it tracks at a coarse scale first to handle fast motion, then refines at full resolution. Only points with a `status=1` confidence flag are kept.

The matched point pairs are passed to `cv.estimateAffinePartial2D` with RANSAC, which fits a 2×3 affine matrix describing the camera's translation, rotation, and scale change between the two frames. RANSAC automatically discards outlier point matches without them corrupting the estimate.

Each matrix is then decomposed into human-readable components:

```
dx    — horizontal translation in pixels
dy    — vertical translation in pixels
angle — rotation in degrees
scale — zoom factor (1.0 = no change)
```

### Stage 2 — Trajectory smoothing

The per-frame translation and rotation values are accumulated into a full camera trajectory using a cumulative sum. This converts frame-to-frame deltas ("moved 2px right") into absolute camera position over time ("at frame 50, camera is 47px right of start").

The raw trajectory is then smoothed using a moving average filter. The filter slides a window of `2r+1` frames across the trajectory and replaces each value with the local average — this preserves slow intentional camera movement while removing high-frequency jitter.

The correction for each frame is the difference between the smoothed and raw trajectory. A spike clamp (`np.clip`) prevents any single frame's correction from exceeding a set threshold, handling sudden camera movements without creating visible jumps.

### Stage 3 — Frame warping and output

For each frame, a new 2×3 correction matrix is built from the correction values using `cv.getRotationMatrix2D` plus a translation offset. `cv.warpAffine` applies this matrix to every pixel in the frame, physically shifting and rotating it back onto the smooth camera path.

Edges that appear black after warping are hidden by cropping a fixed percentage off each side of every frame. The corrected frames are written sequentially to an output file using `cv.VideoWriter`.

---

## Parameters

| Parameter            | Default | Effect                                                                                                             |
| -------------------- | ------- | ------------------------------------------------------------------------------------------------------------------ |
| `SMOOTH_RADIUS`      | `50`    | Smoothing window half-width in frames. Higher = more locked off. Max useful value is ~half your total frame count. |
| `MAX_CORRECTION_PX`  | `30`    | Clamp limit for x/y corrections. Prevents spike frames from overcorrecting.                                        |
| `MAX_CORRECTION_DEG` | `1.0`   | Clamp limit for rotation corrections.                                                                              |
| `CROP_RATIO`         | `0.05`  | Fraction of frame cropped from each edge. Increase if black borders are visible.                                   |

---

## Tech stack

- Python 3.13
- OpenCV 4.10
- NumPy
- Matplotlib (visualisation only)

---

## Limitations

- Designed for handheld shake and high-frequency jitter. Does not handle rolling shutter distortion.
- Scene cuts will produce a spike in the correction values — the clamp handles this but stabilisation quality drops around the cut.
- Processing is single-threaded and proportional to frame count and resolution.
- Output codec is `mp4v`. For better compression, re-encode the output with FFmpeg using `libx264`.
