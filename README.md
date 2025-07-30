### AI Based Ab Roller Rep Counter

## Motivation Story & Introduction
As a child, I never had the opportunity of being confortable with my physical image, growing up in a family with lack of education in terms of healthy lifestyle habits, I was always considered heavy. After some resilientful times, I was able to shift reality into a more healthy lifestyle, achieving the opposite.

One of my favorite exercises, "Ab Wheels" or "Ab Rollers", as someone who was very overweight, and never had the opportunity of being "visually healthy", this exercise significantly helped with my core development.

This personal journey also brought up a technical curiosity. After encountering a LinkedIn post showcasing an AI-powered push-up counter built with similar libraries, this inspired me to explore how these computer vision and machin learning techniques could be applied to the ab wheel, into creating a real-time tracking system.

With these main motivations, my goal is to build tools that can better the health of individuals.


---

## Project Goal & Functionalities
This project aims to build a **real-time system** that accurately counts repetitions of the **kneeled ab wheel exercise** using a standard webcam feed. It serves as a robust prototype for an AI-powered fitness tracker, demonstrating advanced computer vision and state machine logic.

The prototype provides:
* **Real-time Pose Estimation:** Visualizes body landmarks during the exercise.
* **Accurate Repetition Counting:** Tracks and counts completed reps.
* **Robust Error Handling:** Distinguishes between full reps, partial reps, stalled movements, and lost poses.
* **Live Visual Feedback:** Displays hip angle, current rep count, and exercise state.
* **Session Recording:** Captures the entire annotated workout session into a video file for review.



---

## Key Features & Technical Highlights
* **Core Technology:** Leverages **MediaPipe Pose** for highly accurate real-time human pose estimation.
* **Smart State Machine:** Implements a sophisticated state machine (READY, EXTENDING, EXTENDED, RETRACTING, RECOVERY) to precisely track rep phases.
* **Hysteresis & Timeout Logic:** Utilizes distinct thresholds and time-based checks to prevent false counts from minor body movements, accidental stalls, or incomplete reps. Includes a dedicated `RECOVERY` state for explicit reset after failed attempts.
* **Video Processing:** Integrates `OpenCV` for frame manipulation and `FFmpeg` (via subprocess) for robust video recording and stitching of annotated frames.
* **Colab Environment:** Designed and optimized for execution within Google Colaboratory, supporting GPU acceleration for performance.


---

## How to Run This Project
To experience the AI-Powered Ab Wheel Repetition Counter yourself:

1.  **Open in Google Colab:**
    * Go to `colab.research.google.com`.
    * Click `File > Open notebook > GitHub`.
    * Paste your repository URL: `https://github.com/machkevds/ab-roller-rep-counter-machkevds`.
    * Select the main notebook file.
2.  **Set Up Runtime:**
    * Once the notebook loads, change the runtime type to `GPU` for optimal performance: `Runtime > Change runtime type > Hardware accelerator > GPU > Save`.
3.  **Grant Webcam Permissions:**
    * When you run the main code cell, your browser will prompt for webcam access. **You MUST grant this permission.**
4.  **Run the Notebook:**
    * Execute all cells sequentially. The primary execution is within the "Live Webcam Repetition Counter" cell.
    * The system will start capturing from your webcam, display the live processed feed, and record it to a video file.
5.  **User Instructions during Live Session:**
    * **Position yourself sideways** to the camera, ensuring your full body (from knees to head) is visible.
    * **Initial Setup:** Get into your kneeling ab wheel starting position and **hold it still for about 1-3 seconds** until the `State` overlay changes from `INITIAL_SETUP` to `READY`.
    * **Perform Reps:** Execute your ab wheel rollouts. The `Reps` count will increment, and the `State` will update.
    * **Test Edge Cases:** Deliberately try:
        * Stalling at the `EXTENDED` position for >3 seconds.
        * Performing a partial rollout.
        * Briefly moving out of frame.
    * **Stop the Session:** Click the square "Stop" button on the Colab cell to end the recording.
6.  **Download Your Recorded Session:**
    * After the session stops, a video file (`final_recorded_session.mp4`) will be generated. You can download it from the Colab file browser (folder icon on left sidebar).


---

## Technologies Used

* Python
* OpenCV (`cv2`)
* MediaPipe (`mediapipe`)
* NumPy
* FFmpeg (command-line tool)
* Google Colaboratory


---

## Code Walkthrough Showing How Project Works

1.  **Import Libraries**
    ```python
    import cv2
    import mediapipe as mp
    import numpy as np
    import os
    from google.colab.output import eval_js
    from IPython.display import display, Javascript, HTML
    from base64 import b64decode, b64encode
    import time
    import subprocess
    ```
    * **`cv2`**: OpenCV for image/video processing, drawing overlays.
    * **`mediapipe`**: Google's framework for advanced pose estimation.
    * **`numpy`**: For numerical operations on image data and angles.
    * **`os`**: For operating system tasks like creating temporary directories.
    * **`eval_js`, `display`, `Javascript`**: Colab-specific tools to bridge Python with the browser's webcam.
    * **`base64`**: For encoding/decoding image data transferred via JavaScript.
    * **`time`**: For timing operations and calculating FPS.
    * **`subprocess`**: To run external commands like FFmpeg.

2.  **Setup Webcam & Core Variables**
    ```python
    # Start webcam via JavaScript bridge
    start_webcam_stream(width=680, height=240)

    rep_count = 0
    current_state = 'INITIAL_SETUP' # Blocking state for clean start
    aborted_rep_flag = False      # Tracks if current rep attempt was valid
    TIMEOUT_SECONDS = 3           # Timeout for stalled movements
    INITIAL_SETUP_HOLD_SECONDS = 3 # Time to hold initial pose
    ```
    * **`start_webcam_stream`**: Initializes the webcam in your browser for live input.
    * **`rep_count`**: Stores total number of completed repetitions.
    * **`current_state`**: The core of the intelligent state machine, tracking the user's progress.
    * **`aborted_rep_flag`**: Crucial flag to prevent false counts after aborted reps.
    * **`TIMEOUT_SECONDS`**: Defines how long a user can stall in a rep phase before it's considered aborted.
    * **`INITIAL_SETUP_HOLD_SECONDS`**: Defines how long the user must hold the initial setup pose.

3.  **Live Stream Loop & Pose Processing**
    ```python
    with mp_pose.Pose(...) as pose:
        while processing_active:
            frame = capture_and_process_frame_from_webcam(quality=0.8)
            # ... error handling for frame ...
            frame_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
            results = pose.process(frame_rgb)
            annotated_frame = frame.copy()
            # ... rest of loop ...
    ```
    * **`while processing_active`**: The main loop continuously captures and processes frames.
    * **`capture_and_process_frame_from_webcam`**: Gets live image from the webcam.
    * **`pose.process(frame_rgb)`**: MediaPipe model analyzes the current frame to detect the human pose and get joint landmarks.

4.  **Angle Calculation & Blocking Setup Logic**
    ```python
    hip_angle = calculate_angle(r_shoulder_pix, r_hip_pix, r_knee_pix)

    if current_state == 'INITIAL_SETUP':
        if hip_angle > INITIAL_SETUP_MIN_ANGLE and hip_angle < INITIAL_SETUP_MAX_ANGLE:
            initial_setup_hold_counter += 1
            if initial_setup_hold_counter >= INITIAL_SETUP_HOLD_FRAMES:
                current_state = 'READY'
                aborted_rep_flag = False
                # ... other setup for first rep ...
        else:
            initial_setup_hold_counter = 0
    # ... else branch for other states ...
    ```
    * **`calculate_angle`**: Determines the critical hip angle from shoulder, hip, and knee landmarks.
    * **`INITIAL_SETUP`**: This is a **blocking gate**. The system waits here until the user holds a precise kneeling position (`INITIAL_SETUP_MIN/MAX_ANGLE`) for a defined duration. This prevents false counts from initial setup movements (which happened during testing).

5.  **Repetition Logic (State Machine)**
    ```python
    elif current_state == 'READY':
        if hip_angle > retracted_threshold:
            current_state = 'EXTENDING'
            aborted_rep_flag = False # New rep begins
    elif current_state == 'EXTENDING':
        if hip_angle > extended_threshold:
            current_state = 'EXTENDED'
        elif hip_angle < retracted_threshold:
            current_state = 'INITIAL_SETUP' # Partial rep detected, reset to setup
            aborted_rep_flag = True
    elif current_state == 'EXTENDED':
        if hip_angle < retracted_threshold:
            if not aborted_rep_flag: # CHECK: Only count if not aborted
                rep_count += 1
                # ... visual cue ...
            current_state = 'READY'
            aborted_rep_flag = False # Rep cycle ends, clear flag for next
    # ... other states: RETRACTING, RECOVERY ...
    ```
    * **State Transitions**: Hip angle changes trigger precise shifts between `READY`, `EXTENDING`, `EXTENDED`, `RETRACTING`.
    * **`aborted_rep_flag`**: Set to `True` if a rep is partially performed, stalls (`TIMEOUT_FRAMES`), or pose is lost. This ensures `rep_count` only increments for truly successful, non-aborted reps.
    * **`RECOVERY` State**: Handles situations where a rep is aborted mid-way or due to a timeout. It forces a complete reset to `INITIAL_SETUP` before any new reps can be tracked.

6.  **Visual Feedback & Frame Saving**
    ```python
    cv2.putText(annotated_frame, f'Reps: {rep_count}', ...)
    # ... other cv2.putText calls ...

    frame_filename = os.path.join(temp_frame_dir, f'frame_{frame_count:05d}.jpg')
    cv2.imwrite(frame_filename, annotated_frame)
    collected_frame_paths.append(frame_filename)
    # cv2_imshow(annotated_frame) # Commented for recording stability
    ```
    * **Visual Overlays**: Information like `Hip Angle`, `Reps`, `State` are drawn directly onto the frames. Rep completions trigger a visual flash.
    * **Frame Saving**: Each annotated frame is saved as an individual JPEG image to a temporary folder (`/tmp/recorded_frames`).

7.  **Final Video Stitching (FFmpeg)**
    ```python
    # In 'finally' block after loop ends
    final_output_fps = actual_processing_fps # Calculated from processed_frames_count / duration
    ffmpeg_cmd = [
        'ffmpeg', '-y', '-r', str(final_output_fps),
        '-i', os.path.join(temp_frame_dir, 'frame_%05d.jpg'),
        '-c:v', 'libx264', '-pix_fmt', 'yuv420p',
        '-vf', f"fps={final_output_fps}", output_final_video_filename
    ]
    subprocess.run(ffmpeg_cmd, ...)
    ```
    * **Robust Output**: After the live session, all saved individual frames are stitched together by the `ffmpeg` command-line tool. The video's playback FPS is set to match the average processing speed (`actual_processing_fps`), making sure the recorded video plays back at real-time speed. This method is for producing playable `.mp4` files.


---
## Future Enhancements (Optional Ideas)

* **Auditory Feedback:** Add sound cues (e.g., a "ding" on rep completion) when migrating to a local application.
* **Form Correction:** Implement logic to detect and provide feedback on common form errors (e.g. a check for contracting pelvis area as core is engaged to avoid back injuries).
* **User Interface:** Develop a standalone desktop or mobile application for a more integrated user experience.
* **Additional Exercises:** Extend the system to count other exercises (e.g., squats, push-ups).


---
