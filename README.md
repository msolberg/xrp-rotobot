# xrp-rotobot

This is my project to control an [XRP Robot](https://introtoroboticsv2.readthedocs.io/en/latest/index.html) using an NVIDIA [Jetson Orin Nano](https://docs.nvidia.com/jetson/orin-nano-devkit/user-guide/latest/index.html). This implementation uses a USB connection from the Jetson to the controller board.

## USB Remote Control Architecture

The repository provides examples for controlling the XRP robot from a host computer (e.g., PC or SBC) over USB serial. It depends on the [xrp-usb-controller](https://github.com/msolberg/xrp-usb-controller) module.

* **[`look_around.py`](look_around.py)**: Example autonomous application combining computer vision (YOLO) with robot motion control via `XRPController`.
* **[`voice_control.py`](voice_control.py)**: Host-side voice listener that transcribes microphone audio with [Vosk](https://alphacephei.com/vosk/) and maps recognized commands to robot actions (e.g., "Red Light" stops, "Green Light" drives forward) via `XRPController`.
* **[`follow_person.py`](follow_person.py)**: Uses YOLO to track and follow people around.

---

### Deploying the Controller to the Robot

1. **Install [`usb_control.py`](https://github.com/msolberg/xrp-usb-controller/blob/main/src/xrp_usb_controller/usb_control.py) as the robot's boot script:**
   ```bash
   # Copy usb_control.py to the Pico as main.py so it starts automatically
   curl https://github.com/msolberg/xrp-usb-controller/blob/main/src/xrp_usb_controller/usb_control.py -o usb_control.py
   mpremote cp usb_control.py :main.py

   # Soft-reboot the board to start the service
   mpremote reset
   ```   
---

### Example: Running `look_around.py`

[`look_around.py`](look_around.py) captures video frames from a USB camera, runs YOLO object detection, monitors distance with the rangefinder via `XRPController`, and navigates:

```bash
# Activate virtual environment
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Launch the vision & navigation script
python look_around.py
```

* **Behavior**:
  * Reads the front distance sensor via `bot.read_rangefinder()`.
  * If the path is clear (`dist > 40 cm`), it drives forward with `bot.drive_effort(0.5, 0.5)`.
  * If an obstacle is detected, it turns with `bot.drive_effort(0.5, -0.5)`.
  * Annotates and writes output video frames to `annotated_output.mp4`.
* **Stopping**: Press `Ctrl+C` in the terminal. The `finally:` block will automatically call `bot.drive_stop()` to cut power to the motors and release camera and video writer resources cleanly.

---

### Voice Control: `voice_control.py`

[`voice_control.py`](voice_control.py) listens on the host microphone, transcribes speech offline with [Vosk](https://alphacephei.com/vosk/), and performs robot actions for recognized commands. Commands are matched case-insensitively in the `COMMANDS` map at the top of the script:

| Spoken command | Action | Wire command sent |
| :--- | :--- | :--- |
| **"Red Light"** | Stop the motors | `DRIVE,STOP` |
| **"Green Light"** | Drive forward | `DRIVE,EFFORT,0.5,0.5` |

Add more commands by adding entries to the `COMMANDS` map. The feature uses only the existing stable wire commands (`DRIVE,STOP`, `DRIVE,EFFORT`), so no robot-side changes are required.

**Setup (Jetson Orin Nano / JetPack):**

Requires PortAudio - install as an operating system package.

```bash
# Note: JetPack's root filesystem is read-only. Keep the venv and model on a
# writable partition (e.g., your home directory).
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Download and unzip the small English Vosk model (~40 MB, real-time on the Orin Nano)
wget https://alphacephei.com/vosk/models/vosk-model-small-en-us-0.15.zip
unzip vosk-model-small-en-us-0.15.zip
```

**Run:**

```bash
# Activate virtual environment
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Launch the voice listener
python voice_control.py
```

* **Behavior**:
  * Captures 16 kHz mono audio from the default input device.
  * When Vosk detects the end of an utterance, it is matched against the `COMMANDS` map and the matching action is executed.
  * Unrecognized utterances are printed and ignored.
* **Stopping**: Press `Ctrl+C` in the terminal. The `finally:` block calls `bot.drive_stop()` and closes the audio stream.
* **Model**: Set `VOSK_MODEL` to point at a different model directory if needed (e.g., `vosk-model-en-us-0.22` for higher accuracy).

---

### Command Protocol Reference

| Action | Host Python Method | Wire Command | Robot Response |
| :--- | :--- | :--- | :--- |
| **Ping / Handshake** | `bot.ping()` | `PING` | `ACK:PING,PONG` |
| **Rangefinder** | `bot.read_rangefinder()` | `RF` | `ACK:RF,<distance_cm>` |
| **IMU** | `bot.send_command("IMU")` | `IMU` | `ACK:IMU,<pitch>,<heading>,<yaw>` |
| **Stop Motors** | `bot.drive_stop()` | `DRIVE,STOP` | `ACK:DRIVE,STOP` |
| **Motor Effort** | `bot.drive_effort(left, right)` | `DRIVE,EFFORT,<l>,<r>` | `ACK:DRIVE,EFFORT` |
| **Speed (PID)** | `bot.drive_speed(left, right)` | `DRIVE,SPEED,<l>,<r>` | `ACK:DRIVE,SPEED` |
| **Arcade Drive** | `bot.drive_arcade(straight, turn)` | `DRIVE,ARCADE,<s>,<t>` | `ACK:DRIVE,ARCADE` |
| **Servo Angle** | `bot.set_servo(id, angle)` | `SERVO,<id>,<deg>` | `ACK:SERVO,<id>,<deg>` |
| **Free Servo** | `bot.free_servo(id)` | `SERVO,<id>,FREE` | `ACK:SERVO,<id>,FREE` |
| **LED Blink** | `bot.set_rgb(r, g, b)` | `RGB,<r>,<g>,<b>` | `ACK:RGB,<r>,<g>,<b>` |
| **LED Off** | `bot.led_off()` | `OFF` | `ACK:OFF` |
| **Terminate Script** | `bot.stop_program()` | `STOP` | `ACK:STOP` *(exits to REPL)* |

> **Note on `STOP` vs. `DRIVE,STOP`**:
> * Always use `bot.drive_stop()` (`DRIVE,STOP`) to halt the robot's wheels.
> * `bot.stop_program()` (`STOP`) terminates [`usb_control.py`](usb_control.py) and exits back to the MicroPython REPL prompt, which requires restarting the script before subsequent runs.
