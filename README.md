# 🔬 CONNECTO — Hawkins Lab Network File Transfer Tool

> **Peer-to-peer, serverless file transfer and chat over your local network — no cloud, no accounts, no file size limits.**

[![Python](https://img.shields.io/badge/Python-3.9%2B-blue?style=flat-square&logo=python)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.100%2B-009688?style=flat-square&logo=fastapi)](https://fastapi.tiangolo.com)
[![WebRTC](https://img.shields.io/badge/WebRTC-DataChannel-orange?style=flat-square)](https://webrtc.org)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)

---

## 📌 Project Overview

**Connecto** is a self-hosted, browser-based peer-to-peer (P2P) file transfer and messaging tool designed for fast, secure local-network sharing. Inspired by the retro aesthetic of *Stranger Things'* Hawkins National Laboratory, it combines a stunning **CRT terminal UI** with powerful modern web technologies.

### The Problem It Solves

Transferring large files between devices on the same network is surprisingly painful:
- Cloud services have file size limits and require accounts.
- USB drives are slow and inconvenient.
- Most sharing apps route your files through a remote server.

**Connecto** eliminates all these issues. It creates a **direct encrypted browser-to-browser data channel** so files are transferred at local-network speeds without touching the internet after the initial signaling handshake.

### How It Works at a High Level

1. Both devices open the Connecto web app on the same local network.
2. A lightweight Python signaling server helps devices discover each other and exchange WebRTC session metadata (ICE candidates, SDP offers/answers).
3. Once the WebRTC `RTCDataChannel` is established, **all data flows directly between browsers** — the server is completely bypassed.
4. Files are streamed in 64KB chunks directly into the peer's disk or RAM.

---

## ✨ Features

| Feature | Description |
|---|---|
| **Device Discovery** | Automatically scans and lists all Connecto peers on the local network in real time. |
| **QR Code Connect** | Generate a QR code for instant one-click connection from a mobile device or any browser. |
| **QR Code Scanner** | Scan a peer's QR code via your device's camera or by uploading an image file. |
| **P2P File Transfer** | Send any file type and any size directly to a peer via WebRTC DataChannel. |
| **Chunked Streaming** | Files are streamed in **64KB chunks** to handle large files without memory issues. |
| **Disk Streaming** | Uses the modern `showSaveFilePicker` API to stream files directly to disk, bypassing RAM for large files. |
| **File Integrity Check** | Verifies received file size matches sent file size before completing the transfer. |
| **Transfer Abort** | Either peer can cancel an in-progress file transfer at any time. |
| **Real-time Chat** | Send text messages over the same secure P2P data channel. |
| **Drag & Drop** | Drag files directly onto the transfer panel to send them. |
| **mDNS Advertising** | The server advertises itself on the local network via mDNS (`hawkins.local`) using Zeroconf. |
| **CRT Terminal UI** | Retro, animated CRT-style interface with scanline effects, neon glow, and pulsing animations. |
| **Mobile Responsive** | Fully responsive layout for smartphones and tablets. |
| **Session-Based QR Pairing** | Cryptographically unique session UUIDs expire after 2 minutes for security. |

---

## 🛠️ Tech Stack

### Backend

| Technology | Role |
|---|---|
| **Python 3.9+** | Core server language |
| **FastAPI** | ASGI web framework; serves the frontend HTML and REST endpoint (`/get-ip`). |
| **python-socketio** | Async Socket.IO server; handles all real-time signaling events between clients. |
| **Uvicorn** | High-performance ASGI server that runs the combined FastAPI + Socket.IO app. |
| **Zeroconf** | Registers the server as an mDNS service (`_hawkins._tcp.local.`) so devices can discover it on the LAN. |
| **Python `socket`** | Used to detect the server's local IP address for QR code URL generation. |

### Frontend (Single-file `index.html`)

| Technology | Role |
|---|---|
| **Vanilla HTML/CSS/JS** | Core frontend — no frameworks or build tools required. |
| **WebRTC (`RTCPeerConnection`)** | Establishes the encrypted P2P connection and data channel between browsers. |
| **Socket.IO Client** | Connects to the signaling server over WebSocket for handshaking (served from `/static/`). |
| **QRCode.js** | Generates the QR code canvas containing the session URL. |
| **jsQR** | Decodes QR codes from camera frames or uploaded image files. |
| **File System Access API** | (`showSaveFilePicker`) Streams large received files directly to disk without loading into RAM. |
| **CSS Animations / Variables** | Powers the CRT scanline, flicker, pulse, blink, and scanline animations. |

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                  LOCAL AREA NETWORK                      │
│                                                         │
│  ┌──────────────────┐         ┌──────────────────┐      │
│  │   Device A       │         │   Device B       │      │
│  │  (Browser)       │         │  (Browser)       │      │
│  │                  │◄───────►│                  │      │
│  │  index.html      │  WebRTC │  index.html      │      │
│  │  Socket.IO WS    │DataChan.│  Socket.IO WS    │      │
│  └────────┬─────────┘         └────────┬─────────┘      │
│           │  Socket.IO Signaling        │                │
│           │  (SDP + ICE Only)           │                │
│           ▼                             ▼                │
│  ┌─────────────────────────────────────────────────┐    │
│  │           Python Signaling Server                │    │
│  │                                                  │    │
│  │  FastAPI + python-socketio + Uvicorn             │    │
│  │  ┌──────────────┐  ┌─────────────────────────┐  │    │
│  │  │  REST API    │  │   Socket.IO Events       │  │    │
│  │  │  GET /get-ip │  │  register_device         │  │    │
│  │  │  GET /       │  │  join_session            │  │    │
│  │  └──────────────┘  │  connection_request      │  │    │
│  │                    │  connection_response      │  │    │
│  │  ┌──────────────┐  │  offer / answer          │  │    │
│  │  │   Zeroconf   │  │  ice_candidate           │  │    │
│  │  │ mDNS Publish │  │  disconnect              │  │    │
│  │  └──────────────┘  └─────────────────────────┘  │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Key Design Decision: Server-Side State is Minimal

The server only stores:
- **`devices` dict** — Active Socket.IO connections and their name/type.
- **`sessions` dict** — QR session rooms with up to 2 peer SIDs.

**After the WebRTC `RTCDataChannel` is open, the server is completely out of the loop.** All files and chat messages go directly browser-to-browser at full LAN speed.

---

## 📁 Project Structure

```
connecto/
├── main.py             # 🐍 Python backend: FastAPI + Socket.IO signaling server with mDNS
├── index.html          # 🖥️  Single-file frontend: UI, WebRTC, QR code, file transfer logic
├── static/
│   ├── socket.io.min.js    # Socket.IO client library (bundled locally, no CDN needed)
│   ├── qrcode.min.js       # QR code generation library
│   └── jsQR.js             # QR code decoding library (camera & image scanning)
├── .gitignore          # Ignores venv, __pycache__, IDE folders
└── README.md           # This file
```

### File-by-File Purpose

#### `main.py` — The Signaling Server

The only server-side Python file. Responsibilities:
- Serve `index.html` at `GET /`.
- Expose `GET /get-ip` to let the frontend know the server's LAN IP (used in QR URLs).
- Serve bundled JS libraries from `/static/`.
- Handle all Socket.IO signaling events for device registry and WebRTC negotiation.
- Register itself as `hawkins.local` via mDNS on startup using `zeroconf`.

#### `index.html` — The Entire Frontend

A ~1300-line self-contained file. It is divided into three major sections:
1. **CSS** (lines 1–487): Full design system, CRT effects, QR modal, responsive breakpoints.
2. **HTML** (lines 488–577): Semantic structure — scanner panel, QR modal, status terminal, connection prompt, and file transfer chat panel.
3. **JavaScript** (lines 578–1304): All application logic — Socket.IO events, WebRTC setup, file chunking, QR generation, QR scanning, drag-and-drop, chat.

#### `static/` — Bundled Client Libraries

Libraries are served locally to ensure the app works fully **offline** on a local network without any CDN dependency.

---

## ⚙️ How Core Functionality Works

### 1. Device Discovery

```
Browser connects via Socket.IO
  → emits 'register_device' { name, type }
  → Server adds to `devices` dict
  → Server broadcasts updated 'device_list' to all clients
  → Frontend renders clickable device list
```

Each device auto-generates a name (`SUBJECT-XXXX`) and detects its type (`TERMINAL`, `MOBILE`, or `TABLET`) from the User-Agent.

### 2. Connection via Device List (Manual)

```
User A clicks Device B in the list
  → A emits 'connection_request' to server
  → Server forwards request to B
  → B sees "INCOMING SECURE TRANSMISSION" prompt
  → B clicks ACCEPT → emits 'connection_response' (accepted: true)
  → Server forwards response to A
  → A calls setupPeerConnection(isInitiator=true)
  → WebRTC negotiation begins
```

### 3. Connection via QR Code

```
User A clicks "CONNECT VIA QR"
  → A generates crypto.randomUUID() as sessionId
  → A fetches /get-ip to get LAN IP
  → A builds URL: http://{LAN_IP}:{PORT}/?session={sessionId}
  → A generates QR from that URL
  → A emits 'join_session' { sessionId }
  
User B scans the QR (camera or image)
  → jsQR decodes the URL
  → B's browser navigates to or joins the session URL
  → B emits 'join_session' { sessionId }
  
Server sees 2 peers in session → emits 'session_ready' to both
  → Peer 0 gets { isInitiator: true }
  → Peer 1 gets { isInitiator: false }
  → WebRTC negotiation begins
```

Sessions expire after **2 minutes** if no second peer joins.

### 4. WebRTC Negotiation (SDP + ICE)

```
Initiator:
  → new RTCPeerConnection({ iceServers: [] })   // LAN only — no STUN/TURN needed
  → pc.createDataChannel('hawkins-transfer')
  → pc.createOffer() → setLocalDescription()
  → emit 'offer' { sdp } via Socket.IO

Responder:
  → receives 'offer' event
  → new RTCPeerConnection()
  → pc.setRemoteDescription(offer.sdp)
  → pc.createAnswer() → setLocalDescription()
  → emit 'answer' { sdp }

Both sides:
  → exchange ICE candidates via 'ice_candidate' events
  → RTCDataChannel opens → UI switches to transfer panel
```

> **Note:** `iceServers: []` means no STUN/TURN is configured. The app works exclusively on LANs where direct IP routing is possible. For cross-network (internet) use, add a STUN server (see [Future Improvements](#-future-improvements)).

### 5. File Transfer Mechanism

#### Sender Flow

```javascript
// 1. User selects files (browse button or drag-and-drop)
processAndSendFiles(fileList)
  → for each file:
      generate unique fileId = "file-{timestamp}-{index}"
      send f-request JSON { type, id, name, size, fileType }
      display "REQUESTING TRANSFER: ..." in chat

// 2. Receiver approves → sender gets 'f-approved'
startUploading(fileId)
  → read file via file.stream().getReader()
  → loop: read chunks (64KB)
      → send ArrayBuffer chunk via dataChannel
      → if bufferedAmount > 2MB: await 'bufferedamountlow' event  // backpressure
      → update progress bar
  → send f-end JSON signal
```

#### Receiver Flow

```javascript
// 1. Receive f-request → show ACCEPT button
// 2. User clicks ACCEPT → acceptFile(fileId, fileName)
  → if showSaveFilePicker available:
      open native Save dialog → create writable stream to disk
  → else:
      accumulate ArrayBuffers in receiveBuffer[]

// 3. Receive binary ArrayBuffer chunks:
  → if disk mode: append to writable stream via writeQueue promise chain
  → if RAM mode: push to receiveBuffer[]
  → update % progress

// 4. Receive f-end signal:
  → INTEGRITY CHECK: receivedBytes === incomingFileInfo.size
  → if disk: close writable stream
  → if RAM: create Blob → createObjectURL → render download link
  → if image (.jpg, .png, .gif): render inline preview
```

### 6. Transfer Abort / Cancellation

Either peer can abort by clicking `[X] ABORT`:
- **Sender side** sets `cancelFlags[fileId] = true`, stops the read loop, and sends `f-cancel` signal.
- **Receiver side** receives `f-cancel` and calls `activeStream.abort()` (which causes the OS to **delete the partial file**) or clears the RAM buffer.

### 7. Data Integrity Check

After receiving the `f-end` signal, the receiver compares:
```
receivedBytes === incomingFileInfo.size
```
If they don't match, an error is displayed and the file is NOT saved. This catches truncated transfers caused by network drops or stream errors.

---

## 🚀 Installation Guide

### Prerequisites

- **Python 3.9+** installed
- **pip** (Python package manager)
- Two devices connected to the **same local network** (or the same machine for testing)

### Step 1: Clone the Repository

```bash
git clone https://github.com/priyansu0007/connecto.git
cd connecto
```

### Step 2: Create a Virtual Environment (Recommended)

```bash
# Windows
python -m venv venv
venv\Scripts\activate

# macOS / Linux
python3 -m venv venv
source venv/bin/activate
```

### Step 3: Install Dependencies

```bash
pip install fastapi uvicorn python-socketio zeroconf
```

Or if a `requirements.txt` is present:

```bash
pip install -r requirements.txt
```

### Step 4: Run the Server

```bash
python main.py
```

**Expected output:**
```
Hawkins Network Signaling Server running on port 8000
Access locally at: http://localhost:8000
Access on network at: http://192.168.1.XX:8000
Registering mDNS service...
```

### Step 5: Open the App

- **Device 1:** Open `http://localhost:8000` in your browser.
- **Device 2:** Open `http://{SERVER_LAN_IP}:8000` in your browser (e.g., `http://192.168.1.42:8000`).

> The exact LAN IP is shown in the server console output.

### Custom Port

Set the `PORT` environment variable to change the listening port:

```bash
# Windows
set PORT=9090 && python main.py

# macOS / Linux
PORT=9090 python main.py
```

---

## 📖 Usage Guide

### Connecting Two Devices — Method 1: Device List

1. Open the app on both devices on the same network.
2. Each device auto-registers and appears in the **DEVICE SCANNER** panel of the other.
3. Click on the remote device's name.
4. The remote device sees an **"INCOMING SECURE TRANSMISSION"** prompt — click `ACCEPT [Y]`.
5. The **FILE TRANSMISSION PANEL** appears on both sides — you're connected!

### Connecting Two Devices — Method 2: QR Code

1. On **Device A**, click the `⬛ CONNECT VIA QR` button.
2. A QR code modal appears on Device A showing a session URL and QR code.
3. On **Device B**, open the same QR modal and click:
   - **`▶ OPEN CAMERA`** — Point the camera at Device A's QR code.
   - **`▲ SELECT QR IMAGE`** — Upload a screenshot of Device A's QR code.
4. Both devices automatically connect once the QR is scanned. No manual IP typing required!

### Sending a File

1. After connecting, use the **FILE TRANSMISSION PANEL**.
2. **Option A:** Drag and drop a file onto the `-- DRAG AND DROP HERE --` zone.
3. **Option B:** Click `[ BROWSE ]` to open a file picker. Supports multiple files.
4. The remote peer sees an **"INCOMING FILE"** notification with the file name and size.
5. The receiver clicks **`ACCEPT STREAM`**.
6. If the browser supports it, a native **Save File** dialog opens → the file streams directly to disk.
7. Progress is shown as a percentage and animated progress bar on both sides.
8. At 100%, the sender logs `*** UPLOAD COMPLETE ***` and the receiver gets a download link or sees the file saved to disk.

### Sending a Chat Message

1. After connecting, type in the `ENTER COMMAND/MESSAGE...` input field.
2. Press **Enter** or click `[ SEND ]`.
3. Messages appear in the chat box with `> TX:` (sent) and `< RX:` (received) prefixes.

### Cancelling a Transfer

Click the `[X] ABORT` button next to any in-progress upload or download to immediately cancel it. Both peers are notified and any partial files on disk are automatically deleted.

---

## 🔑 Important Code Sections

### `main.py` — `join_session()` (Lines 73–95)

The core session-pairing logic. When a second peer joins the same `sessionId`, the server emits `session_ready` to both, designating one as the **initiator** (`isInitiator: true`). This is what triggers the WebRTC offer/answer flow without any manual coordination.

```python
if len(peers) == 2:
    await sio.emit('session_ready', {'isInitiator': True, 'peerId': peers[1]}, to=peers[0])
    await sio.emit('session_ready', {'isInitiator': False, 'peerId': peers[0]}, to=peers[1])
```

### `index.html` — `setupPeerConnection()` (Lines 797–838)

Creates the `RTCPeerConnection` and wires up all WebRTC negotiation. The `isInitiator` flag determines whether this peer creates the data channel (and sends the SDP offer) or waits for the `ondatachannel` event.

### `index.html` — `startUploading()` (Lines 1039–1103)

The sender-side file streaming engine. Key features:
- Uses `file.stream().getReader()` — a streaming approach that never loads the whole file into memory.
- Sends 64KB `ArrayBuffer` chunks via the WebRTC data channel.
- Implements **backpressure**: if `dataChannel.bufferedAmount > 2MB`, it pauses and waits for `bufferedamountlow`.

### `index.html` — `acceptFile()` (Lines 841–865)

Handles the receiver choosing where to save the file. Detects if `showSaveFilePicker` is available (modern Chromium browsers) for true disk streaming, falling back to an in-RAM buffer (`receiveBuffer[]`) for other browsers.

### `index.html` — `scanFrame()` (Lines 1238–1258)

The real-time QR code scanner. On each animation frame, it:
1. Draws the current video frame to a hidden `<canvas>`.
2. Calls `jsQR(imageData)` to attempt QR decoding.
3. If a code is found, it stops the camera and calls `connectFromQr()`.

---

## 🔒 Security Considerations

| Concern | Current Approach |
|---|---|
| **Data in transit** | Data flows through WebRTC `RTCDataChannel`, which is mandated by the spec to use **DTLS encryption**. All file and chat data is encrypted end-to-end between browsers. |
| **Signaling security** | The signaling server is intended for **LAN use only**. It has no authentication. Avoid exposing it to the public internet without adding auth. |
| **Session expiry** | QR sessions expire after **2 minutes** (`QR_TIMEOUT_MS = 120000`). A new UUID is generated per session via `crypto.randomUUID()`, making sessions unguessable. |
| **Session capacity** | Each session accepts exactly **2 peers**. A third peer attempting to join receives a `session_full` error. |
| **Partial file cleanup** | When a transfer is aborted by either side, `activeStream.abort()` is called, instructing the OS to delete any partially written file data. |
| **No STUN/TURN** | `iceServers: []` means no traffic is routed through external servers. All data stays on your LAN. |

> ⚠️ **Production Warning:** The signaling server has no rate limiting, authentication, or HTTPS. For deployment beyond a trusted LAN, add TLS (via a reverse proxy like Nginx) and authentication middleware.

---

## 📈 Scalability Considerations

The current architecture is designed for **small LAN deployments** (2–10 devices). Key limitations and paths to scale:

| Limitation | Solution |
|---|---|
| In-memory `devices` and `sessions` dicts | Replace with **Redis** for multi-process/multi-server deployments |
| Single Uvicorn process | Use `uvicorn --workers N` with Gunicorn, or run behind a load balancer |
| No auth/rate limiting | Add FastAPI `Depends()` middleware + token-based auth |
| LAN-only (no STUN) | Add STUN servers for NAT traversal: `iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]` |
| 2-peer session limit | Extend session model to support group rooms (requires WebRTC mesh or SFU) |

---

## 🚀 Future Improvements

- [ ] **STUN/TURN support** — Enable cross-network connections over the internet.
- [ ] **Multi-file progress dashboard** — Track all concurrent transfers in a unified panel.
- [ ] **Group sessions** — Allow more than 2 peers via a WebRTC mesh or SFU (e.g., mediasoup).
- [ ] **End-to-end encryption layer** — Add app-level AES-GCM encryption on top of DTLS.
- [ ] **Persistent history** — IndexedDB-backed transfer and chat log.
- [ ] **HTTPS / TLS** — Enable camera access on non-`localhost` origins (required by browsers for `getUserMedia`).
- [ ] **Resume interrupted transfers** — Implement a chunk-offset resume protocol.
- [ ] **Folder transfer** — Zip-on-the-fly before streaming.
- [ ] **PWA / installable app** — Add a Service Worker manifest for offline-capable installation.
- [ ] **Docker container** — One-command `docker run` deployment.
- [ ] **Notifications API** — Notify recipient when a file arrives even if the tab is in background.

---

## 🤝 Contribution Guide

Contributions are welcome! Here's how to get started:

### 1. Fork and Clone

```bash
git clone https://github.com/priyansu0007/connecto.git
cd connecto
```

### 2. Set Up Development Environment

```bash
python -m venv venv
source venv/bin/activate   # or venv\Scripts\activate on Windows
pip install fastapi uvicorn python-socketio zeroconf
```

### 3. Run the Server in Dev Mode

```bash
python main.py
```

Uvicorn supports hot-reload for faster development:
```bash
uvicorn main:socket_app --reload --port 8000
```

### 4. Make Your Changes

- **Backend changes**: Edit `main.py`.
- **Frontend changes**: Edit `index.html` directly (no build step required).
- **Adding new JS libraries**: Place them in `static/` and add a `<script src="/static/...">` tag.

### 5. Test Your Changes

Test across at least two browser windows — one acting as device A and one as device B. For QR scanning tests, use a real mobile device.

### 6. Submit a Pull Request

1. Create a feature branch: `git checkout -b feature/my-feature`
2. Commit your changes with a clear message: `git commit -m "feat: add resume transfer support"`
3. Push and open a PR against `main`.

### Code Style Guidelines

- **Python**: Follow PEP 8. Keep async event handlers clean and short.
- **JavaScript**: Vanilla JS only. Use `const`/`let`, arrow functions, and `async/await`.
- **CSS**: Use the existing CSS custom properties (`--neon-green`, `--bg-dark`, etc.) for all styling.

---

## 📄 License

This project is licensed under the **MIT License**.

```
MIT License

Copyright (c) 2024 Connecto Contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
```

---

<div align="center">

**Built with 🧪 by the Hawkins Lab Research Division**

*"The Gate is open. The data flows."*

</div>
