# SentinelX: A Secure AI-Based Edge Surveillance Framework Using Multi-Sensor Fusion for Perimeter Intrusion Detection

**An AI-powered, low-cost perimeter surveillance framework combining computer vision, geometric localization, and Wi-Fi RSSI observations for intrusion monitoring.**

## Overview

SentinelX is a multi-camera edge surveillance framework designed to detect potential intrusions into a protected perimeter. It combines YOLOv8n-based object detection, monocular distance estimation, approximate geometric localization, and perimeter-boundary analysis with Wi-Fi Received Signal Strength Indicator (RSSI) observations.

The system uses two ESP32-CAM modules and a PC webcam positioned at three different locations around the monitored area. A local processing computer performs visual inference, estimates object distances and viewing angles, and combines eligible observations to estimate a person's approximate position relative to the perimeter.

A complementary RF subsystem records nearby wireless activity using the computer's existing Wi-Fi interface. These observations provide supporting contextual evidence rather than directly detecting people. A rule-based decision module combines the available visual and RF information to inform intrusion decisions.

The system provides a Flask-based surveillance dashboard, detection logging, voice alerts, and email notifications while keeping video processing on the local network.

## Key Features

- **YOLOv8n Object Detection:** Detects people and supported animal classes in camera frames.
- **Multi-Camera Surveillance:** Integrates two ESP32-CAM modules and a PC webcam.
- **Monocular Distance Estimation:** Estimates approximate camera-to-object distance using bounding-box height and assumed real-world object height.
- **Horizontal Viewing-Angle Estimation:** Calculates an object's approximate horizontal angle relative to the camera.
- **Geometric Localization:** Estimates a local 2D position from distance observations across three cameras with known geometry, subject to temporal association.
- **Perimeter Intrusion Analysis:** Classifies estimated positions relative to the monitored boundary and evaluates potential boundary crossings.
- **Wi-Fi RSSI Monitoring:** Records nearby wireless networks and signal-strength values as complementary environmental observations.
- **Rule-Based Sensor Fusion:** Combines visual intrusion decisions, localization residuals, detection confidence, and RF changes to inform the final decision.
- **Live Surveillance Dashboard:** Displays camera feeds and detection information through Flask.
- **Voice and Email Alerts:** Provides audible intrusion notifications and configured object-detection email alerts.
- **Detection Logging:** Stores observations and measurements for subsequent analysis.
- **Local Edge Processing:** Performs inference and decision-making on an on-site computer without cloud-based video analytics.

## System Architecture

```text
        ESP32-CAM 1       ESP32-CAM 2       PC Webcam
              |                |                |
              +----------------+----------------+
                               |
                               v
                     Frame Acquisition
                               |
                               v
                      YOLOv8n Detection
                               |
                               v
                 Distance and Angle Estimation
                               |
                               v
                 Temporal Camera Association
                               |
                               v
                   Geometric Localization
                               |
                               v
                     Perimeter Analysis
                               |
                               +-------------------+
                               |                   |
                               |             Wi-Fi RSSI
                               |             Monitoring
                               |                   |
                               +---------+---------+
                                         |
                                         v
                               Fusion and Decision
                                         |
                      +------------------+------------------+
                      |                  |                  |
                      v                  v                  v
                Flask Dashboard     Voice Alert        Email Alert
                      |
                      v
                 Event Logging
```

*The diagram summarizes the intended processing pipeline described in the project manuscript.*

## Technology Stack

| Component | Technology |
|---|---|
| Programming language | Python |
| Object detection | Ultralytics YOLOv8n |
| Computer vision | OpenCV |
| Numerical processing | NumPy |
| Camera hardware | ESP32-CAM and PC webcam |
| Camera communication | HTTP JPEG capture endpoints |
| Web backend | Flask |
| Video streaming | MJPEG |
| Detection interface | JSON endpoint |
| Wireless sensing | Host Wi-Fi interface and RSSI |
| Notifications | Email and text-to-speech voice alerts |
| Concurrency | Python threading |

## How It Works

### 1. Camera Acquisition

Two ESP32-CAM modules expose JPEG images through HTTP `/capture` endpoints. The processing computer requests snapshots approximately every five seconds. A PC webcam supplies frames continuously through OpenCV.

Each camera has a separate processing worker, with acquisition and processing handled independently.

### 2. AI-Based Object Detection

The application runs a pretrained YOLOv8n model on the acquired frames. The configured classes are:

- Person
- Cat
- Dog
- Horse
- Sheep
- Cow
- Elephant
- Zebra
- Giraffe

Detection records include timestamps, class labels, confidence values, and bounding-box measurements. Animal detections support general surveillance, while person detections are used by the perimeter-decision pipeline.

### 3. Distance Estimation

SentinelX uses a pinhole-camera approximation to estimate the distance between a camera and a detected object:

\[
D \approx \frac{H F}{h}
\]

Where:

- \(D\) is the estimated distance.
- \(H\) is the assumed real-world object height.
- \(F\) is the calibrated focal length in pixel units.
- \(h\) is the detected bounding-box height in pixels.

The result depends on camera calibration, the assumed object height, and the quality of the detection.

### 4. Horizontal Viewing-Angle Estimation

The horizontal angle is estimated from the object's bounding-box center:

\[
\theta = \tan^{-1}\left(\frac{x-C_x}{F}\right)
\]

Here, \(x\) is the bounding-box center, \(C_x\) is the image center, and \(F\) is the horizontal focal length in pixel units.

Both distance and angle are approximate. The current manuscript describes using the distance estimates for geometric localization.

### 5. Geometric Localization

The three camera positions are measured and represented in a common local coordinate system. When person detections from all three cameras fall within a configured temporal association window, their estimated distances can be combined to estimate a local two-dimensional position.

The position is evaluated against the monitored perimeter to determine whether it is inside, outside, on the boundary, or associated with a potential crossing.

A least-squares solution accounts for inconsistent distance measurements, and its residual provides an indication of geometric disagreement.

The described evaluation assumes one person in the monitored area at a time. Multi-person cross-camera tracking is outside the current scope.

### 6. RF/RSSI Observation

The RF subsystem uses the processing computer's existing Wi-Fi interface to scan nearby wireless networks and record their identifiers and RSSI values.

Scans are compared against a baseline to identify changes in the detectable wireless environment. These changes are passed to the fusion module as supporting evidence.

**Important:** RSSI changes do not directly identify a person, establish an intrusion, or determine the person's location. They can be affected by obstacles, distance, antenna orientation, and environmental variation.

### 7. Fusion and Intrusion Decision

The decision module considers the visual perimeter result, localization residual, YOLO confidence, and RF changes.

Under the rule described in the manuscript, a person estimated to be inside the perimeter or crossing its boundary produces an intrusion event. RF changes can provide additional supporting evidence for that event, but RF changes alone do not generate an intrusion decision.

### 8. Dashboard and Alerts

The Flask application provides a monitoring interface with separate camera panels.

- **Live video:** Processed camera frames are exposed through `/video/<cam_key>` using MJPEG streaming.
- **Detection data:** The `/detections` endpoint provides the latest camera detection records as JSON.
- **Voice alerts:** A fixed alert message is converted to speech and played through the processing computer's speaker.
- **Email notifications:** Configured object-detection emails include camera information, class, confidence, estimated distance, viewing angle, timestamp, and an annotated image.
- **Cooldown:** Object-detection emails use a 20-second cooldown for each camera and class.
- **Logging:** Detection and evaluation observations can be recorded for analysis.

The email cooldown prevents repeated notifications for the same camera and class within the configured interval.

Its distance estimates, localization results, RF observations, threat assessments, and alerts may be inaccurate. It should not be treated as a validated security product without appropriate testing, documented results, and human oversight.
