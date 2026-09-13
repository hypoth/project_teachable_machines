# Project Plan: End-to-End Smart Tap Detection System
An automated system to detect if a water tap is left running using AI vision, orchestration via Node-RED, and notifications through Home Assistant.

For details on Home Assistant and NodeRed use following 
** [Setting up home Assistant](settingUpHomeAssistant.md) **

---

## 1. System Architecture Diagram

This diagram shows how data flows from the physical tap to your smart home notifications.

![Dataflow Architecture](SystemArchitectureDiagram_1.png)

### Architecture View (DOT Format)

```dot
digraph G {
    fontname="Helvetica,Arial,sans-serif"
    node [fontname="Helvetica,Arial,sans-serif" style=filled shape=box]
    edge [fontname="Helvetica,Arial,sans-serif" fontsize=10]
    bgcolor="#f9f9f9"

    // Subgraphs for physical vs virtual space
    subgraph cluster_kitchen {
        label = "Kitchen Area";
        color = lightgrey;
        style = dashed;
        tap [label="Physical Water Tap\n(Open / Closed)" color="#74b9ff"]
        camera [label="Wi-Fi Camera\n(RTSP Video Stream)" color="#ffeaa7"]
    }

    subgraph cluster_processor {
        label = "Model Processor (Option A or B)";
        color = lightgrey;
        style = dashed;
        browser [label="JS Web Browser / Node.js\n(Teachable Machine Model)" color="#fab1a0"]
    }

    subgraph cluster_hub {
        label = "Central Smart Home Hub (Mini PC / Pi)";
        color = lightgrey;
        style = dashed;
        nodered [label="Node-RED Add-on\n(Automation Logic & Timers)" color="#a29bfe"]
        ha [label="Home Assistant Core\n(State Engine)" color="#81ecec"]
    }

    subgraph cluster_outputs {
        label = "Alert Outputs";
        color = lightgrey;
        style = dashed;
        phone [label="Smartphone App\n(Push Notification)" color="#ff7675"]
        speaker [label="Smart Speaker\n(Voice Announcement)" color="#ff7675"]
    }

    // Connections
    tap -> camera [label=" Visual Status"]
    camera -> browser [label=" RTSP / Video Feed"]
    browser -> nodered [label=" HTTP POST Webhook\n(JSON: tap_open=true)"]
    nodered -> ha [label=" Triggers Entity Update"]
    ha -> phone [label=" Critical Alert Action"]
    ha -> speaker [label=" TTS Voice Action"]
}
```

---

## 2. Hardware Procurement & Budget Breakdown

| Component | Purpose | Recommended Model | Approximate Cost (INR) | Approximate Cost (USD) |
| :--- | :--- | :--- | :--- | :--- |
| **Central Hub** | Hosts Home Assistant & Node-RED 24/7 | Raspberry Pi 5 (4GB) or Refurbished Intel NUC Mini PC | ₹6,000 – ₹10,000 | \$70 – \$120 |
| **Camera** | Captures real-time video feed of the sink | TP-Link Tapo C100 or C200 (RTSP compatible) | ₹1,800 – ₹2,500 | \$20 – \$30 |
| **Visual Processor** | Device running the Teachable Machine script | Existing Mini PC (Background script) OR old spare Android/iOS tablet | ₹0 (Repurposed) | \$0 |
| **Alert Devices** | Audible or visual alert endpoints | Google Nest Mini / Echo Pop or Smart Bulb | ₹2,500 – ₹4,000 | \$30 – \$50 |
| **Total Estimated Budget** | | **Starting completely from scratch** | **₹10,300 – ₹16,500** | **\$120 – \$200** |

---

## 3. Automation Flow Logic (Node-RED)

To avoid getting spammed the exact second you intentionally turn on the tap to wash your hands, Node-RED will manage a **2-minute delay timer**.

![Automation Flow](automationFlow_3.png)


### Logic View (DOT Format)
```dot
digraph AutomationLogic {
    fontname="Helvetica,Arial,sans-serif"
    node [fontname="Helvetica,Arial,sans-serif" style=filled shape=rect]
    edge [fontname="Helvetica,Arial,sans-serif" fontsize=10]
    rankdir=LR;

    incoming [label="HTTP Input Node\n(Receives Webhook from Model)" color="#ffeaa7"]
    switch_node [label="Switch Node\n(Check Status)" color="#fab1a0"]
    trigger_timer [label="Trigger Node\n(Wait 2 Minutes)" color="#a29bfe"]
    reset_node [label="Reset Signal" color="#ff7675"]
    ha_notify [label="Home Assistant Call Service\n(Send Notifications)" color="#81ecec"]

    incoming -> switch_node
    switch_node -> trigger_timer [label="If state == 'Open'"]
    switch_node -> reset_node [label="If state == 'Closed'"]
    reset_node -> trigger_timer [label=" Cancels ongoing timer"]
    trigger_timer -> ha_notify [label=" If Tap Open for > 2 mins"]
}
```

---

## 4. Phase-by-Phase Implementation Blueprint

### Phase 1: Machine Learning Model Training
1. Secure the Wi-Fi camera above the sink. Ensure the field of view clearly captures both the **tap handle orientation** and the **sink water impact zone**.
2. Open [Teachable Machine](https://withgoogle.com) on a computer/tablet. Create a new **Standard Image Model**.
3. Record 200 frames for `Class 1: Tap Closed` and another 200 frames with running water for `Class 2: Tap Open`.
4. Train the model. Verify accuracy using the live preview pane.
5. Click **Export Model** and select **Upload (shareable link)**. Save the generated `tm.hs` URL endpoint.

### Phase 2: Central Hub Installation
1. Flash **Home Assistant OS (HAOS)** onto your Mini PC or Raspberry Pi using BalenaEtcher.
2. Complete the initial Home Assistant onboarding setup wizard.
3. Go to **Settings > Add-ons > Add-on Store**, find **Node-RED**, and click **Install**. Ensure "Show in sidebar" is checked before starting the addon.

### Phase 3: Deployment of the Model Runner
* **Option A (Dedicated Tablet):** Create a basic HTML file hosting the Teachable Machine Javascript library. Open this page on your old tablet browser 24/7. Use the tablet's front camera pointing at the sink.
* **Option B (Server Container):** Host a simple Node.js execution wrapper script on your Mini PC Docker container. The script pulls frames directly from the Tapo Camera's RTSP feed link (`rtsp://username:password@camera_ip:554/stream1`) and processes it programmatically.

### Phase 4: Setting up the Webhook & Automation Node
1. Inside Node-RED, drag an `http in` node onto the canvas. Set the method to `POST` and URL to `/tap_status`.
2. Insert a basic code snippet to send JSON payloads from your model runner whenever the state shifts:
   ```json
   { "state": "Open" }
   ```
3. Attach a **Switch Node** separating messages based on `msg.payload.state`.
4. Route the "Open" output into a **Trigger Node** configured to wait 2 minutes, then output a payload. Configure it to be resettable if an incoming message containing "Closed" passes through.
5. Link the completed trigger to a Home Assistant `call service` node targeted at `notify.notify` or your Smart Speaker TTS entity.

---

## 5. Risk Assessment & Mitigations

* **False Positives (e.g., Dirty dishes blocking the tap view):** 
  * *Mitigation:* Ensure you include varying background scenes (empty sink vs piled dishes) inside your original `Tap Closed` training dataset frames.
* **Sudden Lighting Shifts (Day vs Night):** 
  * *Mitigation:* Turn on your kitchen overhead light during the evening data gathering phase to mix multiple lighting conditions into the machine learning datasets.
* **Web Browser Crashing/Freezing (Option A):** 
  * *Mitigation:* Implement a simple daily "heartbeat" node check inside Node-RED. If no webhook communication ping is seen for more than 6 hours, fire a maintenance alert to your mobile device.



## 6. Model Runner: HTML/JavaScript Frontend Architecture & Code

This section provides the implementation details for **Phase 3 (Option A)**. It uses the web browser's native capabilities to run your exported Teachable Machine model locally, stream video from either an attached webcam or a remote camera feed, and dispatch secure HTTP POST webhooks directly to your local Node-RED instance.


### 6.1 Project Directory Structure
For clean deployment and ease of updates, organize your local project files using the standard layout below

```text
tap-detector-runner/
├── index.html        # Core user interface and camera display canvas
├── css/
│   └── style.css     # Clean layout styling and visual status indicators
└── js/
    ├── app.js        # Core application logic, model parsing, and webhooks
    └── config.js     # User configuration file (URLs, thresholds, API endpoints)
```

### 6.2 Implementation CodeFile: 
#### js/config.js
This file isolates the parameters you need to change when deploying the system.
```javascript
const CONFIG = {
// Paste your exported Teachable Machine model URL here (include the trailing slash)
MODEL_URL: "withgoogle.com",
// The IP address and port of your Home Assistant Node-RED HTTP input node
NODE_RED_WEBHOOK_URL: "http://HOME_ASSISTANT_IP:1880/tap_status",
// Confidence threshold (0.0 to 1.0) before changing states
CONFIDENCE_THRESHOLD: 0.85,

// How often to check the webcam stream for a prediction (in milliseconds)
PREDICTION_INTERVAL_MS: 200};
```


#### File: `index.html`
The frontend framework that loads the official Google TensorFlow.js libraries securely and sets up the viewing container.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI Tap Detection Hub</title>
    <link rel="stylesheet" href="css/style.css">
    
    <!-- Load TensorFlow.js and Teachable Machine libraries -->
    <script src="https://jsdelivr.net"></script>
    <script src="https://jsdelivr.net"></script>
</head>
<body>
    <div class="container">
        <h1>AI Tap Monitor</h1>
        
        <div id="status-card" class="status-box initializing">
            <span id="status-text">System Initializing...</span>
        </div>

        <div class="video-wrapper">
            <div id="webcam-container"></div>
        </div>

        <div class="metrics-panel">
            <div class="metric-row">
                <span>Tap Closed Probability:</span>
                <span id="prob-closed" class="prob-val">0%</span>
            </div>
            <div class="metric-row">
                <span>Tap Open Probability:</span>
                <span id="prob-open" class="prob-val">0%</span>
            </div>
        </div>
    </div>

    <script src="js/config.js"></script>
    <script src="js/app.js"></script>
</body>
</html>
```


#### File: `css/style.css`
A responsive, high-visibility user interface style layer designed for wall-mounted tablet displays.

```css
body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background-color: #1e272e;
    color: #ffffff;
    margin: 0;
    padding: 20px;
    display: flex;
    justify-content: center;
}
.container {
    max-width: 500px;
    width: 100%;
    text-align: center;
}
h1 { 
    margin-bottom: 20px; 
    font-weight: 400; 
    color: #dcdde1; 
}
.video-wrapper {
    background: #2f3640;
    padding: 10px;
    border-radius: 12px;
    box-shadow: 0 8px 16px rgba(0,0,0,0.3);
    margin-bottom: 20px;
}
#webcam-container canvas {
    width: 100% !important;
    height: auto !important;
    border-radius: 8px;
    transform: scaleX(-1); /* Flips camera horizontally for natural preview */
}
.status-box {
    padding: 15px;
    border-radius: 8px;
    font-weight: bold;
    font-size: 1.2rem;
    margin-bottom: 20px;
    transition: background 0.3s ease;
}
.initializing { background-color: #7f8c8d; }
.closed { background-color: #27ae60; }
.open { 
    background-color: #c0392b; 
    animation: pulse 1.5s infinite; 
}
.metrics-panel {
    background: #2f3640;
    padding: 15px;
    border-radius: 8px;
    text-align: left;
}
.metric-row {
    display: flex;
    justify-content: space-between;
    padding: 8px 0;
    border-bottom: 1px solid #485460;
}
.metric-row:last-child { border: none; }
.prob-val { font-family: monospace; font-weight: bold; }

@keyframes pulse {
    0% { opacity: 1; }
    50% { opacity: 0.6; }
    100% { opacity: 1; }
}
```

#### File: js/app.js
The engine that runs the asynchronous processing loops, manages application state transitions, and avoids network spam by firing webhooks only when changes happen.
```javascriptlet model, webcam, lastState = null;

// Initialize the environment on page load
async function init() 
{
  const statusText = document.getElementById("status-text");
  const statusCard = document.getElementById("status-card");
  try {
    const modelURL = CONFIG.MODEL_URL + "model.json";
    const metadataURL = CONFIG.MODEL_URL + "metadata.json";

    // Load the trained neural network model
    statusText.innerText = "Loading AI Model...";
    model = await tmImage.load(modelURL, metadataURL);

    // Setup the webcam stream parameters
    statusText.innerText = "Accessing Camera Feed...";
    const flipHorizontal = true;
    webcam = new tmImage.Webcam(400, 400, flipHorizontal);
    await webcam.setup();
    await webcam.play();

    // Inject the stream canvas into the HTML page layout
    document.getElementById("webcam-container").appendChild(webcam.canvas);
    statusCard.className = "status-box closed";
    statusText.innerText = "System Active: Monitoring Tap";

    // Begin the cyclical prediction loop
    window.requestAnimationFrame(loop);
  } catch (error) {
    console.error(error);
    statusCard.className = "status-box open";
    statusText.innerText = "Initialization Error. Check logs/camera permissions.";
  }
}

async function loop() {
  webcam.update(); 
  // Update the webcam frame canvas element
  await predict();

  // Throttle the execution speed slightly based on config properties
  setTimeout(() => {
    window.requestAnimationFrame(loop);
    }, CONFIG.PREDICTION_INTERVAL_MS);
}


async function predict() {
  // Run the current camera canvas frame through the tensor architecture
  const prediction = await model.predict(webcam.canvas);
  let probClosed = 0, probOpen = 0;
  
  // Parse the array results based on label metadata strings
  for (let i = 0; i < prediction.length; i++) {
    if (prediction[i].className === "Tap Closed") {
      probClosed = prediction[i].probability;
    } else if (prediction[i].className === "Tap Open") {
      probOpen = prediction[i].probability;
    }
  }
  // Refresh UI readout displays
  document.getElementById("prob-closed").innerText = (probClosed * 100).toFixed(0) + "%";
  document.getElementById("prob-open").innerText = (probOpen * 100).toFixed(0) + "%";
  
  // Determine state threshold matches
  if (probOpen >= CONFIG.CONFIDENCE_THRESHOLD && lastState !== "Open") {
    updateSystemState("Open");
  } else if (probClosed >= CONFIG.CONFIDENCE_THRESHOLD && lastState !== "Closed") {
    updateSystemState("Closed");
  }
}
  
function updateSystemState(newState) {
  lastState = newState;
  const statusCard = document.getElementById("status-card");
  const statusText = document.getElementById("status-text");
  
  if (newState === "Open") {
    statusCard.className = "status-box open";
    statusText.innerText = "ALERT: Tap Detected Open!";
  } else {
    statusCard.className = "status-box closed";
    statusText.innerText = "System Active: Monitoring Tap";
  }// Fire state update to Node-RED Endpoint
  sendStateWebhook(newState);
}

async function sendStateWebhook(stateValue) {
  try {
    const response = await fetch(CONFIG.NODE_RED_WEBHOOK_URL, {
      method: "POST",
      headers: {
        "Content-Type": "application/json"
      },
      body: JSON.stringify({ state: stateValue })
      });
    
      if (!response.ok) {
        console.warn(Webhook failed with HTTP status: ${response.status});
      }
  } 
  catch (networkError) {
    console.error("Network connectivity issue. Failed to reach Node-RED:", networkError);
  }
}// Fire the initializer engine

init();

```

## 6.3 Deployment Verification Check

To run the setup locally:

1. **Open the folder structure** inside your favorite IDE or text editor.
2. **Edit `js/config.js`** to point to your hosted Teachable Machine model URL and your target Node-RED webhook address.

> ⚠️ **Important:** Because browser policies restrict camera access and file-fetching (CORS) on local `file://` protocols, you must run this script through a local web server.

### Quick Option
Run the following command from inside the `tap-detector-runner/` directory:

```bash
python -m http.server 8000
```

Then, browse directly to `http://localhost:8000`.



## 7. Node-RED Automation Flow Configuration

The following JSON block represents the complete Node-RED automation logic. It creates an HTTP POST endpoint (`/tap_status`), parses the incoming Teachable Machine state, and implements a 2-minute timer that resets automatically if the tap is turned off before the time expires.

### 7.1 Import Instructions
1. Open your **Node-RED** workspace inside Home Assistant.
2. Click the **Menu icon** (top-right corner) and select **Import**.
3. Copy the entire JSON code block below and paste it into the import window.
4. Click **Import** and drop the nodes onto your canvas.
5. Double-click the **Home Assistant Notification** node to confirm or update your target notification service (e.g., changing `notify.notify` to your smartphone or smart speaker entity).
6. Click **Deploy** in the top-right corner.

### 7.2 Automation Flow (JSON Format)
```json
[
    {
        "id": "flow_tap_detection",
        "type": "tab",
        "label": "Tap Detection Flow",
        "disabled": false,
        "info": ""
    },
    {
        "id": "node_http_in",
        "type": "http in",
        "z": "flow_tap_detection",
        "name": "HTTP POST /tap_status",
        "url": "/tap_status",
        "method": "post",
        "upload": false,
        "swaggerDoc": "",
        "x": 160,
        "y": 180,
        "wires": [
            [
                "node_http_response",
                "node_switch_state"
            ]
        ]
    },
    {
        "id": "node_http_response",
        "type": "http response",
        "z": "flow_tap_detection",
        "name": "Success Response",
        "statusCode": "200",
        "headers": {},
        "x": 410,
        "y": 120,
        "wires": []
    },
    {
        "id": "node_switch_state",
        "type": "switch",
        "z": "flow_tap_detection",
        "name": "Check Tap State",
        "property": "payload.state",
        "propertyType": "msg",
        "rules": [
            {
                "t": "eq",
                "v": "Open",
                "vt": "str"
            },
            {
                "t": "eq",
                "v": "Closed",
                "vt": "str"
            }
        ],
        "checkall": "true",
        "repair": false,
        "outputs": 2,
        "x": 400,
        "y": 180,
        "wires": [
            [
                "node_trigger_timer"
            ],
            [
                "node_change_reset"
            ]
        ]
    },
    {
        "id": "node_trigger_timer",
        "type": "trigger",
        "z": "flow_tap_detection",
        "op1": "",
        "op2": "Warning: The kitchen tap has been left running!",
        "op1type": "nul",
        "op2type": "str",
        "duration": "2",
        "extend": false,
        "overrideDelay": false,
        "units": "min",
        "reset": "reset",
        "bytopic": "all",
        "topic": "topic",
        "outputs": 1,
        "name": "Wait 2 Minutes",
        "x": 660,
        "y": 160,
        "wires": [
            [
                "node_ha_notification"
            ]
        ]
    },
    {
        "id": "node_change_reset",
        "type": "change",
        "z": "flow_tap_detection",
        "name": "Format Reset Signal",
        "rules": [
            {
                "t": "set",
                "p": "payload",
                "pt": "msg",
                "to": "reset",
                "tot": "str"
            }
        ],
        "action": "",
        "property": "",
        "from": "",
        "to": "",
        "reg": false,
        "x": 640,
        "y": 220,
        "wires": [
            [
                "node_trigger_timer"
            ]
        ]
    },
    {
        "id": "node_ha_notification",
        "type": "api-call-service",
        "z": "flow_tap_detection",
        "name": "Home Assistant Notification",
        "server": "",
        "version": 5,
        "debugenabled": false,
        "domain": "notify",
        "service": "notify",
        "areaId": [],
        "deviceId": [],
        "entityId": [],
        "data": "{\"title\":\"Water Alert\",\"message\":\"{{payload}}\"}",
        "dataType": "json",
        "mergeContext": "",
        "mustacheAltTags": false,
        "outputProperties": [],
        "queue": "none",
        "x": 930,
        "y": 160,
        "wires": [
            []
        ]
    }
]
```


## 8. Hosting the Frontend
Home Assistant OS is a locked-down, minimal operating system designed only to run Home Assistant and its official add-ons. Because it is locked down, you cannot run your custom Docker containers directly inside HAOS.Instead, you have two choices for where to install Docker and run your model runner

### Architecture Choices: Where to Install Docker

OPTION 1: Two Separate Devices (Highly Recommended for Raspberry Pi)
``` text
+-----------------------------------+      +-----------------------------------------+

|    DEVICE 1: Raspberry Pi         |      |    DEVICE 2: Old Phone / Tablet / PC    |
|  - Home Assistant OS (HAOS)       | ===> |  - Runs a Web Browser  |
|  - Node-RED Add-on                | <=== |  - Hosts & Executes the Model Runner    |
+-----------------------------------+      +-----------------------------------------+
```

OPTION 2: Single Powerful Device (Mini PC / Intel NUC)
``` text
+------------------------------------------------------------------------------------+

|                        MINI PC RUNNING A GENERIC LINUX OS (Ubuntu/Debian)          |
|  +---------------------------+   +---------------------------+                     |
|  |   Home Assistant Supervised |   |   Custom Docker Container |                     |
|  |   & Node-RED Add-on       |   |   (Model Runner Hub)      |                     |
|  +---------------------------+   +---------------------------+                     |
+------------------------------------------------------------------------------------+
```

### Locally via a Docker Container (Option 2)

To comply with modern browser security policies (preventing `CORS` and local file isolation errors), the HTML/JavaScript engine must be served over an HTTP protocol rather than being opened directly from a disk folder. 

Below is the blueprint to wrap your project directory into a lightweight **Nginx container** using Docker.

### 8.1 Updated Directory Layout
Add a single file named `Dockerfile` to the root folder of your project:

```text
tap-detector-runner/
├── Dockerfile          # Container build script
├── index.html
├── css/
│   └── style.css
└── js/
    ├── app.js
    └── config.js
```

---

### 8.2 The Build Script

#### File: `Dockerfile`
This configuration pulls a minimal production image of the Nginx web server and copies your project assets directly into its hosting root directory.

```dockerfile
# Use a lightweight, stable Nginx image based on Alpine Linux
FROM nginx:alpine

# Remove default static landing page files from Nginx
RUN rm -rf /usr/share/nginx/html/*

# Copy your local source files into the container webroot
COPY . /usr/share/nginx/html/

# Expose port 80 to your local home network
EXPOSE 80

# Run Nginx in the foreground to keep the container persistent
CMD ["nginx", "-g", "daemon off;"]
```

---

### 8.3 Container Management Commands

Execute the following commands from your terminal inside the `tap-detector-runner/` folder directory to deploy the runner application.

#### 1. Build the Container Image
Compile your code into a portable Docker image tagged as `tap-monitor`:
```bash
docker build -t tap-monitor:latest .
```

#### 2. Launch the Application Container
Run the container continuously in the background. This command forwards port `8080` on your host device to port `80` inside the container:
```bash
docker run -d \
  --name=tap_ai_runner \
  --restart=always \
  -p 8080:80 \
  tap-monitor:latest
```

---

### 8.4 Verification and Access
* **Accessing the Hub:** Open any browser on your home network and navigate to `http://YOUR_SERVER_IP:8080` (or `http://localhost:8080` if running on your machine).
* **Camera Access Note:** Web browsers require an encrypted connection (`HTTPS`) to grant camera access **unless** the domain name is explicitly evaluated as safe. Modern browsers classify `localhost` and `127.0.0.1` as automatically secure. 
* *Tip:* If you are hosting this on a dedicated server (like a Mini PC) and accessing it via an IP address from a separate kitchen tablet, you may need to open `chrome://flags/#unsafely-treat-insecure-origin-as-secure` on your tablet's browser and add your hosting IP address (`http://YOUR_SERVER_IP:8080`) to bypass the hardware block.



## 9. Installing Docker and Deploying the Model Runner

### 9.1 Where to Install This Hardware Layer
* **If you are using a Raspberry Pi for Home Assistant (Option 1):** Do **NOT** install Docker on the Pi. Keep your Pi dedicated to HAOS. Instead, install Docker on a separate computer, or skip Docker entirely by opening the `index.html` file directly on an old tablet's web browser.
* **If you are using a Mini PC (Option 2):** You should install a standard Linux OS (like Ubuntu Server) first. You then install Docker onto the OS, allowing you to run both Home Assistant and your custom Model Runner side-by-side in separate containers.

---

### 9.2 Step-by-Step Docker Installation

If you are setting up the Model Runner on a Linux machine (Ubuntu/Debian) or a separate computer, follow these steps to install Docker:

#### Step 1: Clean Out Old Versions
Remove any conflicting legacy packages before beginning:
```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

#### Step 2: Set Up Docker's Official Repository
1. Update your system package index and install prerequisite tools:
   ```bash
   sudo apt-get update
   sudo apt-get install ca-certificates curl
   sudo install -m 0755 -d /etc/apt/keyrings
   ```
2. Download and save Docker's official GPG security key:
   ```bash
   sudo curl -fsSL https://docker.com -o /etc/apt/keyrings/docker.asc
   sudo chmod a+r /etc/apt/keyrings/docker.asc
   ```
3. Add the repository to your system's software source list:
   ```bash
   echo \
     "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://docker.com \
     $(. /etc/core/os-release && echo "$VERSION_CODENAME") stable" | \
     sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   sudo apt-get update
   ```

#### Step 3: Install the Docker Engine
Install the core Docker runtime environment and its companion tools:
```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### Step 4: Verify the Installation
Confirm the background engine is running successfully by launching a test container:
```bash
sudo docker run hello-world
```

---

### 9.3 Deploying Your Model Runner Files

Once Docker is verified, follow this workflow to upload, build, and launch your Teachable Machine visual processor code:

```text
+-------------------+      +-------------------+      +--------------------+

|  1. Transfer Files | ---> |  2. Build Image   | ---> |  3. Run Container  |
|  (SFTP / VS Code) |      |  (docker build)   |      |  (docker run -d)   |
+-------------------+      +-------------------+      +--------------------+
```

1. **Transfer files:** Move your project folder (`tap-detector-runner/` containing your `Dockerfile`, `index.html`, `css/`, and `js/`) onto the machine running Docker using an SFTP client or an app like VS Code.
2. **Open the Terminal:** Change directories into your project folder:
   ```bash
   cd path/to/tap-detector-runner
   ```
3. **Build the image:** Run the compilation command to package your static HTML app:
   ```bash
   sudo docker build -t tap-monitor:latest .
   ```
4. **Launch the web server:** Run the container persistently. This map routes public traffic from port `8080` directly to the hosted application container:
   ```bash
   sudo docker run -d \
     --name=tap_ai_runner \
     --restart=always \
     -p 8080:80 \
     tap-monitor:latest
   ```
5. **Test Access:** Open a web browser on any device in your home network and go to `http://<SERVER_IP>:8080` to see your live Teachable Machine monitoring page.

