# Installation Guide: Home Assistant OS & Node-RED on Raspberry Pi
A step-by-step framework to deploy Home Assistant OS (HAOS) and configure Node-RED for automation workflows.

---

## 1. System Topology Overview

The diagram below maps how your Raspberry Pi manages incoming camera data via the Docker Runner, processes logic in Node-RED, and translates states into Home Assistant Core actions.

```text
+---------------------------------------------------------------------------------+

|                          RASPBERRY PI ARCHITECTURE                              |
+---------------------------------------------------------------------------------+

|                                                                                 |
|   [ Wi-Fi Camera ] ---> (Port 8080: Docker Model Runner HTML/JS Engine)         |
|                                     |                                           |
|                                     v (HTTP POST /tap_status)                   |
|                    +----------------------------------------+                   |
|                    |     Node-RED Add-on (Supervisor)       |                   |
|                    |  - Receives Webhook                    |                   |
|                    |  - Evaluates 2-Minute Logic State      |                   |
|                    +----------------------------------------+                   |
|                                     |                                           |
|                                     v (Internal Websocket / API)                |
|                    +----------------------------------------+                   |
|                    |         Home Assistant Core            |                   |
|                    |  - Manages States & Action Handlers    |                   |
|                    +----------------------------------------+                   |
|                                     |                                           |
|                +--------------------+--------------------+                      |
|                |                                         |                      |
|                v                                         v                      |
|     [ Smart Speaker TTS ]                      [ Smartphone Push Notification ] |
|                                                                                 |
+---------------------------------------------------------------------------------+
```

---

## 2. Prerequisites & Hardware Checklist

Before beginning, ensure you have gathered the following core hardware elements:
* **MicroSD Card:** Minimum 32GB (64GB+ strongly recommended). Must be rated **Class 10, A2, or High-End Endurance** to withstand high read/write cycles.
* **Power Supply:** Official Raspberry Pi USB-C power supply adapter to prevent voltage drops.
* **Network Connection:** An Ethernet cable connected directly to your router is preferred for initial onboarding stability.

---

## 3. Step-by-Step Installation

### Step 3.1: Flash Home Assistant OS
1. Download and install **Raspberry Pi Imager** on your computer.
2. Insert your MicroSD card into your computer's card reader slot.
3. Open Raspberry Pi Imager and click **Choose OS**.
4. Navigate through the menus: **Other specific-purpose OS** > **Home assistants and home automation** > **Home Assistant**.
5. Select the exact version matching your hardware architecture (e.g., **Home Assistant OS for Raspberry Pi 4** or **Raspberry Pi 5**).
6. Click **Choose Storage** and target your inserted MicroSD card.
7. Click **Next** / **Write** and confirm the formatting prompt. Wait for the flashing and verification processes to finish.

```text
[Raspberry Pi Imager] -> [Choose OS] -> [Home Assistant] -> [Select Storage] -> [Write]
```

### Step 3.2: First Boot & Initialization
1. Safely eject the MicroSD card and insert it into your Raspberry Pi.
2. Connect your Ethernet cable to the Pi, then plug in the power supply cable.
3. Allow **10 to 15 minutes** for the Pi to boot, configure its partitions, and fetch the latest Home Assistant software updates. Do not interrupt power during this sequence.
4. On a computer connected to the same local network, open a web browser and navigate to:
   ```text
   http://homeassistant.local:8123
   ```
   *(If the URL fails to resolve, check your router's device list to find the exact IP address assigned to the Pi, and navigate to `http://YOUR_PI_IP:8123`)*.
5. Follow the onscreen instructions to create your owner/administrator profile, name your home instance, and save your geographic coordinates.

---

## 4. Installing and Configuring the Node-RED Add-on

Home Assistant OS includes a built-in store that manages background tools. Follow these steps to attach Node-RED securely:

### Step 4.1: Add-on Procurement
1. Inside your Home Assistant dashboard sidebar, navigate to **Settings** > **Add-ons**.
2. Click the **Add-on Store** button located in the bottom-right corner.
3. Use the search bar to locate **Node-RED**. Click on the search item card.
4. Click the **Install** button. The initial background setup may require up to 2-3 minutes.

### Step 4.2: Critical Configuration Tweaks
Before hitting the start switch, you must configure security parameters to allow remote webhooks (like those sent from your Docker container runner):

```text
+--------------------------------------------------------+

|              NODE-RED CONFIGURATION PANEL              |
+--------------------------------------------------------+

|                                                        |
|  [ Options Tab ]                                       |
|  - credential_secret: "YourRandomSecurePasswordHere"   |
|                                                        |
|  [ Toggle Switches ]                                   |
|  - Start on boot:      [ ON ]                          |
|  - Watchdog:           [ ON ]                          |
|  - Show in sidebar:    [ ON ]  <--- (Enables UI access) |
|                                                        |
+--------------------------------------------------------+
```

1. Switch to the **Configuration** tab at the top of the Node-RED Add-on page.
2. Locate the `credential_secret` field block. Type a secure, long string passphrase (this encrypts credentials saved inside your flows).
3. Click **Save** in the bottom-right corner of the configuration block.
4. Move back to the **Info** tab.
5. Toggle the switches for **Start on boot**, **Watchdog** (restarts the app if it freezes), and **Show in sidebar**.
6. Click the large green **Start** button. Wait for the red circular icon to shift to a steady green state.

---

## 5. Merging the Ecosystem Together

To let your external Docker Container model runner communicate with Node-RED, you must verify network paths. 

```text
   [Docker Model Runner Engine] 
                |
                v Sends network data packets to:
   http://<YOUR_RASPBERRY_PI_IP>:1880/endpoint
```

* **Default Security Pass-through:** The Home Assistant community Node-RED add-on exposes port `1880` natively over your internal private network (`HTTP`), bypassing home firewall layers. 
* **Verifying Endpoint Integrity:** Ensure the `NODE_RED_WEBHOOK_URL` defined inside your frontend project runner's `js/config.js` file matches your target network path structure precisely:
  ```javascript
  NODE_RED_WEBHOOK_URL: "http://192.168.1"
  ```
  *(Replace `192.168.1.50` with the actual static local network IP address assigned to your physical Raspberry Pi server instance)*.

