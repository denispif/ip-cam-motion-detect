# IP Cam Motion Detection

A Jupyter notebook that captures JPEG snapshots from an IP camera and performs basic motion detection with OpenCV.

Source inspiration: https://pyimagesearch.com/2015/05/25/basic-motion-detection-and-tracking-with-python-and-opencv/

## What the notebook does

The notebook in `motion_detect.ipynb` implements a simple frame-differencing motion detector:

1. Connects to an IP camera snapshot endpoint over HTTP.
2. Authenticates with Basic Auth credentials.
3. Downloads a single JPEG frame for each polling cycle.
4. Decodes the JPEG bytes into an OpenCV image.
5. Resizes the image to a fixed width for consistent processing.
6. Converts the frame to grayscale.
7. Applies Gaussian blur to reduce noise.
8. Stores the first processed frame as the background reference.
9. Computes the absolute difference between the current frame and the reference frame.
10. Thresholds the difference image to isolate changed pixels.
11. Dilates the threshold mask to fill gaps.
12. Finds contours in the binary mask.
13. Filters contours by area to ignore tiny changes.
14. Draws a bounding box around the first significant moving region.
15. Marks the frame as `Occupied` when motion is detected, otherwise `Unoccupied`.
16. Overlays a timestamp and room status on the frame.
17. Displays three views side-by-side:
    - annotated camera frame
    - threshold mask
    - frame delta image

## Notebook structure

### 1. Imports

The notebook imports the following libraries:

- `cv2` for image processing and contour detection
- `urllib3` for HTTP requests to the camera
- `imutils` for frame resizing and contour handling
- `time` for polling delays
- `numpy` for byte-to-array conversion
- `datetime` for timestamp overlay
- `matplotlib` for inline image display
- `IPython.display.clear_output` and `IPython.display.display` for refreshing notebook output

### 2. Camera configuration

The notebook defines camera settings with environment-variable overrides:

- `CAMERA_IP` / `IP_CAM_HOST`
- `CAMERA_PORT` / `IP_CAM_PORT`
- `CAMERA_USER` / `IP_CAM_USER`
- `CAMERA_PASSWORD` / `IP_CAM_PASSWORD`
- `CAMERA_JPEG_SIZE` / `IP_CAM_JPEG_SIZE`

The snapshot URL is built as:

`http://{host}:{port}/snap.jpg?JpegSize=XL`

Credentials are sent through HTTP Basic Auth headers when both user and password are provided.

### 3. IP camera wrapper

The `IPCamera` class:

- stores the camera URL
- creates an `urllib3.PoolManager` with retries (`total=2`) and timeout (`5s` by default, configurable)
- builds auth headers only when credentials are provided
- fetches a frame from the snapshot endpoint
- handles request/decode failures gracefully
- decodes the response bytes into an OpenCV image

The helper function `get_frame()` simply returns `cam.get_frame()`.

### 4. Motion detection loop

The notebook initializes:

- baseline frame state (`first_frame`)
- tunable detection settings (min area, threshold value, blur kernel, dilate iterations)
- polling controls:
  - `POLL_INTERVAL_SECONDS` / `IP_CAM_POLL_SECONDS` (default `1`)
  - `MAX_CONSECUTIVE_FETCH_FAILURES` / `IP_CAM_MAX_FETCH_FAILURES` (default `5`)
  - `STOP_ON_MOTION` / `IP_CAM_STOP_ON_MOTION` (default `1`)

Then it enters an infinite loop that:

- fetches the next frame
- retries when frame fetch fails and stops only after repeated failures
- resizes the frame
- converts it to grayscale
- blurs it
- stores the first processed frame as the baseline background
- compares subsequent frames against that baseline
- detects moving regions from contour areas
- annotates the image with status and timestamp
- renders debug views in the notebook
- waits between iterations based on configurable poll interval
- stops when motion is found (default) or on `KeyboardInterrupt`

## Motion detection algorithm details

The detection logic is based on background subtraction against the very first frame:

- `frame_delta = cv2.absdiff(first_frame, gray)` computes pixel changes.
- `cv2.threshold(..., 25, 255, cv2.THRESH_BINARY)` keeps only larger differences.
- `cv2.dilate(..., iterations=2)` expands bright regions and fills small holes.
- `cv2.findContours(...)` extracts connected moving regions.
- `cv2.contourArea(c)` measures each region size.
- If a contour area is greater than or equal to `min_area`, the frame is marked as motion detected.

## Output views

The notebook displays three panels:

1. **Frame** — the original frame with status text, timestamp, and motion bounding box.
2. **Threshold** — the binary image showing changed regions.
3. **Frame Delta** — the absolute difference between the reference frame and current frame.

These views help debug false positives and tune the contour area threshold.

## Notes and limitations

- The reference background is only the first frame and is never updated.
  - This makes the detector simple, but sensitive to lighting changes.
- The notebook polls a snapshot endpoint instead of reading a continuous RTSP/video stream.
- Credentials can be provided through environment variables instead of hard-coding.
- The loop breaks on the first detected motion event.
- A very small `MOTION_MIN_AREA` may cause false positives from noise or compression artifacts.

## How to use

1. Open `motion_detect.ipynb` in Jupyter.
2. Install the required Python packages if needed:
   - `opencv-python`
   - `urllib3`
   - `imutils`
   - `numpy`
   - `matplotlib`
   - `ipython`
3. Set your camera connection values in the notebook or with environment variables:
   - `IP_CAM_HOST`
   - `IP_CAM_PORT`
   - `IP_CAM_USER`
   - `IP_CAM_PASSWORD`
4. Run the notebook cells in order.
5. Watch the three output panels for detected motion.
6. Adjust `MOTION_MIN_AREA`, threshold, blur, or frame width values if detection is too sensitive or not sensitive enough.

## Possible improvements

- Update the background model over time instead of using only the first frame.
- Save snapshots when motion is detected.
- Record video clips around detection events.
- Add notification support.
- Move credentials into environment variables or a config file.
- Replace notebook polling with a standalone script or service.
- Support RTSP or MJPEG streams directly.
