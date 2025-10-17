# Recording Script:

The `record.sh` script automates the recording and snapshot capture of RTSP camera streams using **FFmpeg**.

It supports operations to start, stop, check status, capture snapshots, and clean up old processes.

This script is ideal for automated video capture system.

### Script: 

```bash
#!/bin/bash
# record.sh
# Usage:
#   ./record.sh start <url> <name> <trid>
#   ./record.sh stop <name>
#   ./record.sh snapshot <url> <name> <trid>
#   ./record.sh status <name>
#   ./record.sh cleanup

BASE_DIR="/home/recorder"
DATA_DIR="$BASE_DIR/Data"
VIDEO_DIR="$DATA_DIR"
IMAGE_DIR="$DATA_DIR"
PID_DIR="/tmp"

mkdir -p "$VIDEO_DIR" "$IMAGE_DIR" "$PID_DIR"

start_recording() {
    url="$1"
    name="$2"
    trid="$3"
    if [ -z "$url" ] || [ -z "$name" ] || [ -z "$trid" ]; then
        echo "Usage: $0 start <url> <name> <trid>"
        exit 1
    fi

    mkdir -p "$VIDEO_DIR/$name/videos"
    output="$VIDEO_DIR/$name/videos/${trid}.mp4"
    pid_file="$PID_DIR/record_${name}.pid"

    if [ -f "$pid_file" ]; then
        echo "Recording already running for $name"
        exit 1
    fi

    echo "Starting recording for $name..."
    nohup ffmpeg -rtsp_transport tcp -i "$url" -c copy -f mp4 "$output" >/dev/null 2>&1 &
    echo $! > "$pid_file"
    echo "Recording started. PID $(cat $pid_file)"
}

stop_recording() {
    name="$1"
    if [ -z "$name" ]; then
        echo "Usage: $0 stop <name>"
        exit 1
    fi

    pid_file="$PID_DIR/record_${name}.pid"
    if [ ! -f "$pid_file" ]; then
        echo "No recording found for $name"
        exit 1
    fi

    pid=$(cat "$pid_file")
    echo "Stopping recording for $name (PID $pid)..."
    kill "$pid" >/dev/null 2>&1
    rm -f "$pid_file"
    echo "Recording stopped for $name."
}

snapshot() {
    url="$1"
    name="$2"
    trid="$3"
    if [ -z "$url" ] || [ -z "$name" ] || [ -z "$trid" ]; then
        echo "Usage: $0 snapshot <url> <name> <trid>"
        exit 1
    fi

    mkdir -p "$IMAGE_DIR/$name/images"
    output="$IMAGE_DIR/$name/images/${trid}.jpg"
    echo "Capturing snapshot for $name..."

    ffmpeg -y -rtsp_transport tcp -timeout 5000000 -i "$url" -frames:v 1 "$output" >/dev/null 2>&1

    if [ $? -eq 0 ]; then
        echo "Snapshot saved: $output"
    else
        echo "ERROR: Failed to take snapshot for $name"
    fi
}

status() {
    name="$1"
    if [ -z "$name" ]; then
        echo "Usage: $0 status <name>"
        exit 1
    fi

    pid_file="$PID_DIR/record_${name}.pid"
    if [ ! -f "$pid_file" ]; then
        echo "No recording running for $name"
        exit 0
    fi

    pid=$(cat "$pid_file")
    if ps -p "$pid" >/dev/null 2>&1; then
        echo "Recording is running for $name (PID $pid)"
    else
        echo "Recording not active for $name (stale PID file)"
        rm -f "$pid_file"
    fi
}

cleanup() {
    echo "Cleaning up old PID files..."
    find "$PID_DIR" -type f -name "record_*.pid" -exec rm -f {} \;
    echo "Cleanup done."
}

case "$1" in
    start)
        start_recording "$2" "$3" "$4"
        ;;
    stop)
        stop_recording "$2"
        ;;
    snapshot)
        snapshot "$2" "$3" "$4"
        ;;
    status)
        status "$2"
        ;;
    cleanup)
        cleanup
        ;;
    *)
        echo "Usage: $0 {start|stop|snapshot|status|cleanup}"
        exit 1
        ;;
esac

```

### Usage: 

```bash
./record.sh <command> [arguments]
```

### Commands: 

| Command                        | Description                                       |
| ------------------------------ | ------------------------------------------------- |
| `start <url> <name> <trid>`    | Start recording a live RTSP stream                |
| `stop <name>`                  | Stop the ongoing recording                        |
| `snapshot <url> <name> <trid>` | Capture a single image (snapshot) from the stream |
| `status <name>`                | Check if a recording is currently active          |
| `cleanup`                      | Remove stale PID files                            |
### Directory Structure:

|Directory|Purpose|
|---|---|
|`/home/recorder/Data/<name>/videos`|Stores video recordings per camera|
|`/home/recorder/Data/<name>/images`|Stores snapshots per camera|
|`/tmp/record_<name>.pid`|Stores PID file of active recording process|
### Script Variables:

|Variable|Default Value|Description|
|---|---|---|
|`BASE_DIR`|`/home/recorder`|Base working directory|
|`DATA_DIR`|`$BASE_DIR/Data`|Root for all recordings and images|
|`VIDEO_DIR`|`$DATA_DIR`|Video storage directory|
|`IMAGE_DIR`|`$DATA_DIR`|Snapshot storage directory|
|`PID_DIR`|`/tmp`|Location to store PID files of active recordings|

### Command Details:

### 🟢 `start_recording`

**Usage:**

```bash
./record.sh start <url> <name> <trid>`
```

**Description:**  
Starts recording the RTSP stream from `<url>` and saves it as `<trid>.mp4` inside the directory for `<name>`.

**Process:**

1. Validates arguments.
    
2. Creates directories if missing:  
    `"$VIDEO_DIR/$name/videos"`
    
3. Checks for an existing PID file (prevents multiple concurrent recordings).
    
4. Uses `ffmpeg` to record:
    
    `ffmpeg -rtsp_transport tcp -i "$url" -c copy -f mp4 "$output"`
    
5. Runs `ffmpeg` in the background via `nohup`.
    
6. Saves the process ID (PID) to `/tmp/record_<name>.pid`.
    

**Example:**

`./record.sh start rtsp://192.168.1.10/stream1 Camera1 20251017T1530`

**Output:**

`Starting recording for Camera1... Recording started. PID 2451`

---

### 🔴 `stop_recording`

**Usage:**

`./record.sh stop <name>`

**Description:**  
Stops a currently running recording for the specified `<name>`.

**Process:**

1. Reads the PID from `/tmp/record_<name>.pid`.
    
2. Sends a `kill` signal to the process.
    
3. Deletes the PID file.
    

**Example:**

`./record.sh stop Camera1`

**Output:**

`Stopping recording for Camera1 (PID 2451)... Recording stopped for Camera1.`

---

### 📸 `snapshot`

**Usage:**

`./record.sh snapshot <url> <name> <trid>`

**Description:**  
Captures a single snapshot (JPEG image) from an RTSP stream.

**Process:**

1. Validates arguments.
    
2. Creates target directory:  
    `"$IMAGE_DIR/$name/images"`
    
3. Runs FFmpeg command:
    
    `ffmpeg -y -rtsp_transport tcp -timeout 5000000 -i "$url" -frames:v 1 "$output"`
    
4. Saves image as `<trid>.jpg`.
    

**Example:**

`./record.sh snapshot rtsp://192.168.1.10/stream1 Camera1 20251017T1532`

**Output:**

`Capturing snapshot for Camera1... Snapshot saved: /home/recorder/Data/Camera1/images/20251017T1532.jpg`

---

### 🔍 `status`

**Usage:**

`./record.sh status <name>`

**Description:**  
Checks if the recording process for `<name>` is active.

**Process:**

1. Looks for the PID file.
    
2. Verifies if the process is still running using `ps -p`.
    
3. If inactive but PID file exists, removes stale file.
    

**Example:**

`./record.sh status Camera1`

**Possible Outputs:**

`Recording is running for Camera1 (PID 2451)`

or

`Recording not active for Camera1 (stale PID file)`

or

`No recording running for Camera1`

---

### 🧹 `cleanup`

**Usage:**

`./record.sh cleanup`

**Description:**  
Deletes all PID files in `/tmp` that match `record_*.pid`.  
Useful after a system reboot or script crash.

**Example:**

`./record.sh cleanup`

**Output:**

`Cleaning up old PID files... Cleanup done.`

---

## **Dependencies**

- **ffmpeg** (required)
    
    - Install on Ubuntu/Debian:
        
        `sudo apt install ffmpeg -y`
1. Validates arguments.
    
2. Creates target directory:  
    `"$IMAGE_DIR/$name/images"`
    
3. Runs FFmpeg command:
    
    `ffmpeg -y -rtsp_transport tcp -timeout 5000000 -i "$url" -frames:v 1 "$output"`
    
4. Saves image as `<trid>.jpg`.
    

**Example:**

```bash
./record.sh snapshot rtsp://localhost/MR02 MR02 T153272360
```

**Output:**

`Capturing snapshot for Camera1... Snapshot saved: /home/recorder/Data/Camera1/images/20251017T1532.jpg` 


## **Error Handling**

|Error Message|Cause|Resolution|
|---|---|---|
|`Recording already running for <name>`|A PID file exists|Stop current recording or remove stale PID|
|`No recording found for <name>`|No active recording|Verify the name or check `status`|
|`ERROR: Failed to take snapshot for <name>`|Stream unavailable or incorrect URL|Recheck RTSP URL or network|
|`Usage: $0 start <url> <name> <trid>`|Missing arguments|Pass all required parameters|

**Example Workflow:**

```bash
# Start recording
./record.sh start rtsp://192.168.1.10/live Camera1 20251017T1530

# Check status
./record.sh status Camera1

# Capture snapshot
./record.sh snapshot rtsp://192.168.1.10/live Camera1 20251017T1531

# Stop recording
./record.sh stop Camera1

# Clean up stale PID files
./record.sh cleanup

```

## **Notes**

- Uses `nohup` to allow recordings to persist even after user logout.
    
- Output (videos and images) are organized by camera name and timestamp ID.
    
- Designed for **Linux environments** (tested on Ubuntu).
    
- PID management ensures only one active recording per camera.


# Recording API:

The Purpose of this API is to automate the recording process. To start recording and Stop recording when the round is started and stopped.

### Base URL:

```http
https://rec.liv007.site/record_api.php
```


The following is the PHP code of the API:

```php
<?php
// record_api.php
header('Content-Type: application/json');

// Secure with a secret key
$SECRET_KEY = "q3uoPqVC0st3HL1eE3c9";
if (!isset($_GET['key']) || $_GET['key'] !== $SECRET_KEY) {
    http_response_code(403);
    echo json_encode(["error" => "Forbidden"]);
    exit;
}

$action = $_GET['action'] ?? '';
$url    = $_GET['url'] ?? '';
$name   = $_GET['name'] ?? '';
$trid   = $_GET['trid'] ?? '';

$SCRIPT = "/home/recorder/record.sh";

switch ($action) {
    case 'start':
        if (empty($url) || empty($name) || empty($trid)) {
            echo json_encode(["error" => "Missing parameters"]);
            exit;
        }

        $cmd = "nohup bash -c 'sleep 50 && $SCRIPT start " .
               escapeshellarg($url) . " " .
               escapeshellarg($name) . " " .
               escapeshellarg($trid) .
               "' > /dev/null 2>&1 &";

        shell_exec($cmd);

        echo json_encode([
            "status" => "scheduled",
            "action" => "start",
            "name" => $name,
            "trid" => $trid,
            "message" => "Recording will start in 50 seconds"
        ]);
        break;

    case 'stop':
        if (empty($name)) {
            echo json_encode(["error" => "Missing parameters"]);
            exit;
        }

        $cmd = "nohup bash -c 'sleep 10 && $SCRIPT stop " . escapeshellarg($name) . "' > /dev/null 2>&1 &";
        shell_exec($cmd);

        echo json_encode([
            "status" => "scheduled",
            "action" => "stop",
            "name" => $name,
            "message" => "Recording will stop in 10 seconds"
        ]);
        break;

    case 'snapshot':
        if (empty($url) || empty($name) || empty($trid)) {
            echo json_encode(["error" => "Missing parameters"]);
            exit;
        }

        $output = shell_exec("$SCRIPT snapshot " .
            escapeshellarg($url) . " " .
            escapeshellarg($name) . " " .
            escapeshellarg($trid) . " 2>&1");

        echo json_encode([
            "status" => "ok",
            "action" => "snapshot",
            "name" => $name,
            "trid" => $trid,
            "message" => trim($output)
        ]);
        break;

    case 'status':
        if (empty($name)) {
            echo json_encode(["error" => "Missing parameters"]);
            exit;
        }

        $output = shell_exec("$SCRIPT status " . escapeshellarg($name) . " 2>&1");

        echo json_encode([
            "status" => "ok",
            "action" => "status",
            "name" => $name,
            "message" => trim($output)
        ]);
        break;

    default:
        echo json_encode(["error" => "Invalid action"]);
        break;
}
?>
```


## 🔐 Authentication

All requests **must include** a valid secret key via the `key` query parameter.  
If the key is missing or invalid, the API will respond with:

`{   "error": "Forbidden" }`

**Example:**

`?key=q3uoPqVC0st3HL1eE3c9`

---

## ⚙️ Endpoints

### 1. **Start Recording**

**Endpoint:**

`GET /record_api.php?action=start&key=SECRET_KEY&url={RTSP_URL}&name={NAME}&trid={TOURNAMENT_ID}`

**Description:**  
Schedules a recording process to start after **50 seconds**.  
This is done asynchronously in the background using a shell script.

**Parameters:**

| Parameter | Type   | Required | Description                   |
| --------- | ------ | -------- | ----------------------------- |
| `action`  | string | ✅        | Must be `start`               |
| `key`     | string | ✅        | Secret API key                |
| `url`     | string | ✅        | RTSP or stream URL            |
| `name`    | string | ✅        | Unique session or camera name |
| `trid`    | string | ✅        | Tournament or tracking ID     |

**Example Request:**

`GET https://rec.liv007.site/record_api.php?action=start&key=q3uoPqVC0st3HL1eE3c9&url=rtsp://192.168.1.10:554/stream&name=Camera1&trid=Match42`

**Example Response:**

`{   "status": "scheduled",   "action": "start",   "name": "Camera1",   "trid": "Match42",   "message": "Recording will start in 50 seconds" }`

---

### 2. **Stop Recording**

**Endpoint:**

`GET /record_api.php?action=stop&key=SECRET_KEY&name={NAME}`

**Description:**  
Schedules a stop recording command after **10 seconds**.

**Parameters:**

|Parameter|Type|Required|Description|
|---|---|---|---|
|`action`|string|✅|Must be `stop`|
|`key`|string|✅|Secret API key|
|`name`|string|✅|Name used during recording start|

**Example Request:**

`GET https://rec.liv007.site/record_api.php?action=stop&key=q3uoPqVC0st3HL1eE3c9&name=Camera1`

**Example Response:**

`{   "status": "scheduled",   "action": "stop",   "name": "Camera1",   "message": "Recording will stop in 10 seconds" }`

---

### 3. **Take Snapshot**

**Endpoint:**

`GET /record_api.php?action=snapshot&key=SECRET_KEY&url={RTSP_URL}&name={NAME}&trid={TOURNAMENT_ID}`

**Description:**  
Captures a single snapshot from the given stream immediately and saves it via the recording script.

**Parameters:**

|Parameter|Type|Required|Description|
|---|---|---|---|
|`action`|string|✅|Must be `snapshot`|
|`key`|string|✅|Secret API key|
|`url`|string|✅|RTSP or stream URL|
|`name`|string|✅|Camera or stream name|
|`trid`|string|✅|Tournament or tracking ID|

**Example Request:**

`GET https://yourdomain.com/record_api.php?action=snapshot&key=q3uoPqVC0st3HL1eE3c9&url=rtsp://192.168.1.10:554/stream&name=Camera1&trid=Match42`

**Example Response:**

`{   "status": "ok",   "action": "snapshot",   "name": "Camera1",   "trid": "Match42",   "message": "Snapshot saved as /home/recorder/Data/images/Camera1_Match42_20251017_153000.jpg" }`

---

### 4. **Check Status**

**Endpoint:**

`GET /record_api.php?action=status&key=SECRET_KEY&name={NAME}`

**Description:**  
Returns the current recording status of the specified camera or stream.

**Parameters:**

|Parameter|Type|Required|Description|
|---|---|---|---|
|`action`|string|✅|Must be `status`|
|`key`|string|✅|Secret API key|
|`name`|string|✅|Name used during recording|

**Example Request:**

`GET https://rec.liv007.site/record_api.php?action=status&key=q3uoPqVC0st3HL1eE3c9&name=Camera1`

**Example Response:**

`{   "status": "ok",   "action": "status",   "name": "Camera1",   "message": "Recording process is running (PID 3471)" }`

---

## ⚠️ Error Responses

|HTTP Code|Example Response|Meaning|
|---|---|---|
|`403`|`{"error": "Forbidden"}`|Missing or invalid key|
|`400`|`{"error": "Missing parameters"}`|Required query parameters are missing|
|`200`|`{"error": "Invalid action"}`|Action name is not recognized|

---

## 🧩 Backend Integration

Each API call executes a shell script:

`/home/recorder/record.sh`

Supported script commands:

- `start {url} {name} {trid}`
    
- `stop {name}`
    
- `snapshot {url} {name} {trid}`
    
- `status {name}`
    

The API schedules `start` and `stop` asynchronously using `nohup` and background execution (`&`), ensuring non-blocking performance.

---

## ✅ Example Workflow

1. **Start round:**  
    → Call `/record_api.php?action=start...`  
    ⏱ Starts recording in 50s.
    
2. **During round:**  
    → Optionally call `/record_api.php?action=snapshot...`  
    📸 Captures live image.
    
3. **End round:**  
    → Call `/record_api.php?action=stop...`  
    ⏱ Stops recording in 10s.


### Summary: 

This API is used in the Marble Racing Backend Code. When the New Round API is hit in the backend it will also hit this record API which will start the recording of the round. The 50 second delay in API is added in order to skip the intro video which is played at the start of the new round. This way we have recording of each round when it is started. 

To Stop the recording this record API is hit when the Set Result API is hit. This way when the round is ended the recording stops and we gets the whole video of the round. The 10 second delay is added at stop to get the ranking section in the live stream. If we don't add the delay here it will stop exactly when the API for set Result is hit. 

To take a Snapshot we have added this API at the set rank API of marble racing. This way when the marbles reach the finish line the snapshot of the finish point is taken. We have not set it in Set result because if we set this API in set Result a Logo will interrupt the screenshot. To get a clean screenshot we are using it at set Rank.
