---
title: Feature of Motion change detection 
description: >-
  Machine Learning Project
author: Cronus
date: 2024-01-02 10:46:00 
categories: [Project, Edu Helper]
tags: [Machine Learning]
pin: false
image:
  path: /assets/img/ML/motion.jpeg
  width: 300
  height: 300
---

EDU_Siri provides a feature that converts lecture PPT slides into text.

To convert PPT slides into text, it is essential to detect changes in the slides. If there are any changes, the system captures an image of the slide, extracts the text using OCR, and saves it as a text file.

Here, the function responsible for detecting changes in the PPT content and capturing the screen when movement is detected is called motion change detection.

Motion capture has many well-established references, so there was no need to significantly modify or implement a new algorithm.

However, using standard example code resulted in excessive frame-by-frame image saving, leading to unnecessary storage consumption and performance optimization issues. To address this, I modified the code to check for motion changes and save frames every two seconds.

## 1. Load Video File & Basic Setup
```python
cap = cv.VideoCapture(video_name)
fgbg = cv.createBackgroundSubtractorMOG2(history=500, varThreshold=500, detectShadows=0)

# Setting up to capture images at specified intervals
start_time = int(time.time())
i = 0  # File naming index

while True:
    instant_time = int(time.time())

    # Get the current frame count and total frame count
    if cap.get(cv.CAP_PROP_POS_FRAMES) == cap.get(cv.CAP_PROP_FRAME_COUNT):
        break

    ret, frame = cap.read()

    width = frame.shape[1]
    height = frame.shape[0]
    frame = cv.resize(frame, (int(width * 0.8), int(height * 0.8)))  # Resize the display frame
```

## 2. Subtract Background & Get Object Information
```python
fgmask = fgbg.apply(frame)  # Background subtraction
nlabels, labels, stats, centroids = cv.connectedComponentsWithStats(fgmask)
```
The goal is to detect motion changes in objects within the video. Since other elements are unnecessary, we use fgbg.apply() to remove all background elements, leaving only the moving objects.

![1.gif](/assets/img/ML/motion1.gif)

The connectedComponentsWithStats() function retrieves four key elements:
```
retval
label    - Label map where each object is assigned a number
stats    - N x 5 matrix (N = number of objects + 1), each row represents an object with (x, y) top-left coordinates and area
centroid - N x 2 matrix representing the center of mass (x, y) coordinates
```

To detect object motion, we only need three factors: total area change, coordinate change, and center of mass change. Thus, we extract stats and centroids.

## 3. Check Object Motion & Save Frame
```python
for index, centroid in enumerate(centroids):
    if stats[index][0] == 0 and stats[index][1] == 0:
        continue
    if np.any(np.isnan(centroid)):
        continue

    x, y, width, height, area = stats[index]
    centerX, centerY = int(centroid[0]), int(centroid[1])

    # Detect motion based on area
    if area > 400:
        if instant_time > start_time + 2:
            cv.imwrite("%s/%s.png" % (dir_name, i), frame)
            start_time = instant_time
```
We compare the obtained object motion values with the area threshold. If it exceeds a certain value, the system captures a screenshot and saves it to a file.

![2.png](/assets/img/ML/motion2.png)

## Full Code
```python
def motion_change_detection(video_name, dir_name) -> str:
    base_path = os.getcwd()
    video_name = base_path + "\\video\\" + str(video_name)
    cap = cv.VideoCapture(video_name)

    fgbg = cv.createBackgroundSubtractorMOG2(history=500, varThreshold=500, detectShadows=0)
    start_time = int(time.time())  # Set capture interval
    i = 0  # File naming index

    try:
        while True:
            instant_time = int(time.time())

            if cap.get(cv.CAP_PROP_POS_FRAMES) == cap.get(cv.CAP_PROP_FRAME_COUNT):
                break

            ret, frame = cap.read()
            width = frame.shape[1]
            height = frame.shape[0]
            frame = cv.resize(frame, (int(width * 0.8), int(height * 0.8)))

            fgmask = fgbg.apply(frame)  # Background subtraction
            nlabels, labels, stats, centroids = cv.connectedComponentsWithStats(fgmask)

            for index, centroid in enumerate(centroids):
                if stats[index][0] == 0 and stats[index][1] == 0:
                    continue
                if np.any(np.isnan(centroid)):
                    continue

                x, y, width, height, area = stats[index]
                centerX, centerY = int(centroid[0]), int(centroid[1])

                # Detect motion based on area
                if area > 400:
                    if instant_time > start_time + 2:
                        cv.imwrite("%s/%s.png" % (dir_name, i), frame)
                        start_time = instant_time
                        i += 1

            cv.imshow('mask', fgmask)
            cv.imshow('frame', frame)

            k = cv.waitKey(30) & 0xff
            if k == 27:
                break

        cap.release()
        print("\n+==================================+")
        print("| Detecting motion change is done! |")
        print("+==================================+")

    except OSError:
        error("Failed to save file in directory that you want. Please check permission of directory or check filename")
```