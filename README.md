# A2_Mini_FPV
SIYI A2 mini → Jetson Orin NX (Low-Latency RTSP)
Receive and process the SIYI A2 mini FPV stream on Jetson Orin NX with ~≤180 ms latency using NVIDIA HW decoding.

TL;DR
	•	Stream URL: rtsp://192.168.144.25:8554/main.264
	•	Use GStreamer + nvv4l2decoder on Jetson
	•	On camera: set 720p (if OK) and turn off Distortion Correction for lowest latency

⸻

Contents
	•	Prerequisites
	•	Camera: one-time settings
	•	Quick start (display window)
	•	Use in code (OpenCV)
	•	Latency tuning notes
	•	Troubleshooting

⸻

Prerequisites
	•	Jetson Orin NX with JetPack (includes GStreamer, NVIDIA codecs)
	•	Jetson is on the same LAN as the camera (e.g., 192.168.144.x)
	•	A2 mini connected via Ethernet (directly or through your SIYI link)

Stream endpoint (fixed on A2 mini):

rtsp://192.168.144.25:8554/main.264


⸻

Camera: one-time settings

Open the SIYI FPV Windows → Gimbal Camera:
	•	Video Resolution: set to HD (720p) or Full HD (1080p).
For lower latency and lighter decode, 720p is recommended if quality is acceptable.
	•	Distortion Correction: OFF for lowest latency (disables extra on-camera processing).

Keep the camera at the highest FPS available; we only need video (no audio).
    https://siyi.biz/en/index.php?id=downloads1&asd=186 
⸻

Quick start (display window)

GStreamer + NVDEC (recommended):

gst-launch-1.0 rtspsrc location=rtsp://192.168.144.25:8554/main.264 latency=50 drop-on-latency=true \
  ! rtph264depay ! h264parse ! nvv4l2decoder enable-max-performance=1 disable-dpb=1 \
  ! nvvidconv ! fpsdisplaysink video-sink=nveglglessink sync=false text-overlay=false

Why these flags:
	•	latency=50 + drop-on-latency=true: tiny jitter buffer; drops late frames instead of queuing → stays real-time
	•	nvv4l2decoder: Jetson hardware H.264 decode (very low decode latency)
	•	sync=false: render ASAP (don’t wait for display clock)

If your network is rock-solid, try latency=30 or even latency=0. If you see stutter, raise to 60–120.

⸻

Use in code (OpenCV)

C++ (OpenCV + GStreamer)

#include <opencv2/opencv.hpp>
int main() {
  std::string pipe =
    "rtspsrc location=rtsp://192.168.144.25:8554/main.264 latency=50 drop-on-latency=true ! "
    "rtph264depay ! h264parse ! nvv4l2decoder enable-max-performance=1 disable-dpb=1 ! "
    "nvvidconv ! video/x-raw,format=BGR ! appsink";
  cv::VideoCapture cap(pipe, cv::CAP_GSTREAMER);
  if (!cap.isOpened()) return -1;
  cv::Mat frame;
  while (cap.read(frame)) {
    // TODO: process frame (BGR)
    // cv::imshow("A2 mini", frame); if (cv::waitKey(1) == 27) break;
  }
}

Python (OpenCV + GStreamer)

import cv2
pipe = (
  "rtspsrc location=rtsp://192.168.144.25:8554/main.264 latency=50 drop-on-latency=true ! "
  "rtph264depay ! h264parse ! nvv4l2decoder enable-max-performance=1 disable-dpb=1 ! "
  "nvvidconv ! video/x-raw,format=BGR ! appsink"
)
cap = cv2.VideoCapture(pipe, cv2.CAP_GSTREAMER)
if not cap.isOpened():
    raise RuntimeError("Failed to open A2 mini RTSP via GStreamer")
while True:
    ok, frame = cap.read()
    if not ok:
        break
    # TODO: process frame (BGR)
    # cv2.imshow("A2 mini", frame); 
    # if cv2.waitKey(1) & 0xFF == 27: break


⸻

Latency tuning notes
	•	UDP over TCP: rtspsrc uses UDP by default; avoid forcing TCP (adds delay).
	•	Drop strategy: drop-on-latency=true prevents queue buildup during network hiccups.
	•	Decoder buffering: disable-dpb=1 reduces decoder’s internal buffering.
	•	Resolution trade-off: 720p → lower encode + transmit + decode time (at some quality cost).
	•	Display sync: sync=false renders immediately; for pure processing use appsink (no display).

⸻

Troubleshooting
	•	No video:
	•	Confirm cabling/power and that Jetson can ping 192.168.144.25
	•	Recheck the RTSP URL exactly as above
	•	Try a simpler test:

gst-launch-1.0 rtspsrc location=rtsp://192.168.144.25:8554/main.264 ! fakesink


	•	Latency grows over time:
	•	Lower latency (e.g., 20–50) and keep drop-on-latency=true
	•	Ensure Distortion Correction = OFF, consider 720p on camera
	•	Use wired Ethernet to the SIYI air unit where possible
	•	OpenCV fallback (without GStreamer):
	•	Not recommended for latency, but if you must: set very small read buffers and grab frames as fast as possible to avoid backlog.

⸻

