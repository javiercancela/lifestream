# Lifestream: Personal Digital Activity Capture System

## Specification Document v3.0

---

## 1. Executive Summary

Lifestream is a personal digital activity capture system that records your browsing activity through a Chrome extension and processes it locally. This version uses **Native Messaging** for communication—the extension writes directly to a lightweight native host that appends to a file queue. A separate Python processor runs periodically to enrich and index the data.

### Core Principles

- **Capture is instant and reliable**: Native host just appends JSON—no processing, no failures
- **No always-on daemon for capture**: Native host only runs when Chrome sends a message
- **Batch processing is efficient**: Embeddings and summaries generated in batches
- **Graceful degradation**: Capture works even if processor isn't running
- **Privacy-first**: All data stays local; sensitive information filtered before storage

### Architecture Overview

```
┌─────────────────┐
│ Chrome Extension│
│                 │
│ • Capture text  │
│ • Capture images│
└────────┬────────┘
         │
         │ Native Messaging (stdin/stdout)
         ▼
┌─────────────────┐
│ Native Host     │  ← Rust binary, ~2MB
│                 │
│ • Validate JSON │
│ • Append to:    │
│   queue.jsonl   │
└────────┬────────┘
         │
         │ File append (instant)
         ▼
┌─────────────────┐
│ queue.jsonl     │  ← Human-readable, debuggable
│                 │
│ {"url":...}     │
│ {"url":...}     │
│ {"url":...}     │
└────────┬────────┘
         │
         │ Every 5 min (systemd timer)
         ▼
┌─────────────────┐
│ Python          │
│ Processor       │
│                 │
│ • Parse & clean │
│ • Detect type   │
│ • Sanitize      │
│ • Deduplicate   │
│ • Embed         │
│ • Summarize     │
│                 │
│ Writes to:      │
│ • SQLite DB     │
│ • ChromaDB      │
└────────┬────────┘
         │
         │ On-demand
         ▼
┌─────────────────┐
│ Query Layer     │
│                 │
│ • CLI           │
│ • Web UI        │
│ • RAG /ask      │
└─────────────────┘
```

---

## 2. System Components

### 2.1 Chrome Extension

Extracts content from pages and sends it to the native host via Native Messaging.

**Responsibilities:**
- Extract visible text content from pages
- Extract/reference images on the page
- Capture page metadata (URL, title, timestamp)
- Send snapshots to native host
- Minimal UI (status indicator, pause/resume)

**What it does NOT do:**
- No content analysis
- No HTTP requests
- No data storage

### 2.2 Native Host (Rust)

A minimal binary that Chrome launches on-demand. Receives JSON via stdin, validates it, and appends to a JSONL queue file.

**Responsibilities:**
- Receive JSON messages from extension
- Validate message structure
- Append to queue file with proper locking
- Return success/failure to extension

**What it does NOT do:**
- No content processing
- No network requests
- No database access
- No filtering (that's the processor's job)

**Why Rust:**
- Single static binary, no runtime dependencies
- Fast startup (~5ms)
- Tiny binary size (~2MB)
- Memory safe file operations with proper locking

### 2.3 Python Processor

Runs periodically via systemd timer. Reads the queue file, processes each snapshot, and stores in the database.

**Responsibilities:**
- Read and parse queue file
- Detect content type (email, chat, document, etc.)
- Extract structured metadata via parsers
- Sanitize sensitive information
- Check domain blocklist
- Deduplicate content
- Generate embeddings
- Generate summaries for conversations
- Store in SQLite + ChromaDB
- Rotate/archive processed queue entries

### 2.4 Query Layer

CLI and optional web interface for searching and asking questions.

**Responsibilities:**
- Full-text search (SQLite FTS5)
- Semantic search (ChromaDB)
- Hybrid search with rank fusion
- RAG-based Q&A
- Browse/filter activities
- Manage blocked domains

---

## 3. Chrome Extension Specification

### 3.1 Manifest (manifest.json)

```json
{
  "manifest_version": 3,
  "name": "Lifestream Capture",
  "version": "1.0.0",
  "description": "Capture browsing activity to local Lifestream archive",
  "permissions": [
    "activeTab",
    "storage",
    "tabs",
    "nativeMessaging"
  ],
  "host_permissions": [
    "<all_urls>"
  ],
  "action": {
    "default_popup": "popup.html",
    "default_icon": {
      "16": "icons/icon16.png",
      "48": "icons/icon48.png",
      "128": "icons/icon128.png"
    }
  },
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [
    {
      "matches": ["<all_urls>"],
      "js": ["content.js"],
      "run_at": "document_idle"
    }
  ],
  "commands": {
    "capture-now": {
      "suggested_key": {
        "default": "Ctrl+Shift+S"
      },
      "description": "Capture current page"
    }
  }
}
```

### 3.2 Content Script (content.js)

```javascript
// content.js - Extracts content from the current page

/**
 * Extract all visible text from the page
 */
function extractText() {
  const walker = document.createTreeWalker(
    document.body,
    NodeFilter.SHOW_TEXT,
    {
      acceptNode: function(node) {
        const style = window.getComputedStyle(node.parentElement);
        if (style.display === 'none' || style.visibility === 'hidden') {
          return NodeFilter.FILTER_REJECT;
        }
        const tagName = node.parentElement.tagName.toLowerCase();
        if (['script', 'style', 'noscript'].includes(tagName)) {
          return NodeFilter.FILTER_REJECT;
        }
        if (!node.textContent.trim()) {
          return NodeFilter.FILTER_REJECT;
        }
        return NodeFilter.FILTER_ACCEPT;
      }
    }
  );

  const textParts = [];
  let node;
  while (node = walker.nextNode()) {
    textParts.push(node.textContent.trim());
  }
  
  return textParts.join('\n');
}

/**
 * Extract images from the page
 */
function extractImages() {
  const images = [];
  const imgElements = document.querySelectorAll('img');
  
  imgElements.forEach((img) => {
    // Skip tiny images (likely icons/tracking pixels)
    if (img.naturalWidth < 100 || img.naturalHeight < 100) {
      return;
    }
    
    // Skip invisible images
    const rect = img.getBoundingClientRect();
    if (rect.width === 0 || rect.height === 0) {
      return;
    }
    
    images.push({
      src: img.src,
      alt: img.alt || '',
      width: img.naturalWidth,
      height: img.naturalHeight
    });
  });
  
  return images;
}

/**
 * Capture the current page state
 */
function capturePage() {
  return {
    url: window.location.href,
    title: document.title,
    timestamp: new Date().toISOString(),
    text: extractText(),
    images: extractImages(),
    html: document.documentElement.outerHTML
  };
}

// Listen for messages from background script
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.action === 'capture') {
    const snapshot = capturePage();
    sendResponse({ success: true, snapshot });
  }
  return true;
});

// Notify background that content script is ready
chrome.runtime.sendMessage({ action: 'content_ready' });
```

### 3.3 Background Script (background.js)

```javascript
// background.js - Coordinates capture and sends to native host

const NATIVE_HOST_NAME = 'com.lifestream.host';
const CAPTURE_INTERVAL_MS = 60000;
const TAB_ACTIVE_THRESHOLD_MS = 3000;

let captureTimers = {};
let nativePort = null;

/**
 * Connect to native host
 */
function connectNativeHost() {
  if (nativePort) {
    return nativePort;
  }
  
  try {
    nativePort = chrome.runtime.connectNative(NATIVE_HOST_NAME);
    
    nativePort.onMessage.addListener((response) => {
      if (response.error) {
        console.error('Native host error:', response.error);
      }
    });
    
    nativePort.onDisconnect.addListener(() => {
      nativePort = null;
      if (chrome.runtime.lastError) {
        console.error('Native host disconnected:', chrome.runtime.lastError.message);
      }
    });
    
    return nativePort;
  } catch (error) {
    console.error('Failed to connect to native host:', error);
    return null;
  }
}

/**
 * Send snapshot to native host
 */
function sendToNativeHost(snapshot) {
  const port = connectNativeHost();
  if (port) {
    try {
      port.postMessage(snapshot);
      return true;
    } catch (error) {
      console.error('Failed to send to native host:', error);
      nativePort = null;
      return false;
    }
  }
  return false;
}

/**
 * Capture from a specific tab
 */
async function captureTab(tabId) {
  const settings = await chrome.storage.local.get(['enabled', 'blocklist']);
  
  if (settings.enabled === false) {
    return;
  }
  
  try {
    const tab = await chrome.tabs.get(tabId);
    
    // Skip chrome:// and other restricted URLs
    if (!tab.url || !tab.url.startsWith('http')) {
      return;
    }
    
    // Check blocklist
    const url = new URL(tab.url);
    const blocklist = settings.blocklist || [];
    if (blocklist.some(domain => url.hostname.includes(domain))) {
      return;
    }
    
    // Request capture from content script
    const response = await chrome.tabs.sendMessage(tabId, { action: 'capture' });
    
    if (response?.success && response?.snapshot) {
      sendToNativeHost(response.snapshot);
    }
  } catch (error) {
    // Content script might not be loaded
    console.debug('Could not capture tab', tabId, error.message);
  }
}

/**
 * Start periodic capture for a tab
 */
function startPeriodicCapture(tabId) {
  stopPeriodicCapture(tabId);
  
  chrome.storage.local.get(['captureInterval'], (settings) => {
    const interval = settings.captureInterval || CAPTURE_INTERVAL_MS;
    captureTimers[tabId] = setInterval(() => {
      captureTab(tabId);
    }, interval);
  });
}

/**
 * Stop periodic capture for a tab
 */
function stopPeriodicCapture(tabId) {
  if (captureTimers[tabId]) {
    clearInterval(captureTimers[tabId]);
    delete captureTimers[tabId];
  }
}

// Initialize default settings
chrome.runtime.onInstalled.addListener(() => {
  chrome.storage.local.set({
    enabled: true,
    blocklist: [
      'chase.com',
      'bankofamerica.com',
      'wellsfargo.com',
      'citi.com',
      'capitalone.com',
      'fidelity.com',
      'schwab.com',
      'vanguard.com',
      'paypal.com',
      '1password.com',
      'lastpass.com',
      'bitwarden.com'
    ],
    captureInterval: CAPTURE_INTERVAL_MS
  });
});

// Capture when tab becomes active
chrome.tabs.onActivated.addListener((activeInfo) => {
  Object.keys(captureTimers).forEach(tabId => {
    if (parseInt(tabId) !== activeInfo.tabId) {
      stopPeriodicCapture(parseInt(tabId));
    }
  });
  
  setTimeout(() => {
    captureTab(activeInfo.tabId);
    startPeriodicCapture(activeInfo.tabId);
  }, TAB_ACTIVE_THRESHOLD_MS);
});

// Capture when page finishes loading
chrome.tabs.onUpdated.addListener((tabId, changeInfo, tab) => {
  if (changeInfo.status === 'complete' && tab.active) {
    captureTab(tabId);
  }
});

// Stop capture when tab is closed
chrome.tabs.onRemoved.addListener((tabId) => {
  stopPeriodicCapture(tabId);
});

// Handle keyboard shortcut
chrome.commands.onCommand.addListener((command) => {
  if (command === 'capture-now') {
    chrome.tabs.query({ active: true, currentWindow: true }, (tabs) => {
      if (tabs[0]) {
        captureTab(tabs[0].id);
      }
    });
  }
});

// Handle messages from popup
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.action === 'get_status') {
    sendResponse({ 
      connected: nativePort !== null,
      enabled: true // Will be updated from storage
    });
  } else if (message.action === 'capture_current') {
    chrome.tabs.query({ active: true, currentWindow: true }, (tabs) => {
      if (tabs[0]) {
        captureTab(tabs[0].id);
        sendResponse({ success: true });
      }
    });
    return true;
  }
});
```

### 3.4 Popup UI (popup.html + popup.js)

```html
<!-- popup.html -->
<!DOCTYPE html>
<html>
<head>
  <style>
    body {
      width: 260px;
      padding: 16px;
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
      font-size: 14px;
    }
    .header {
      display: flex;
      align-items: center;
      justify-content: space-between;
      margin-bottom: 16px;
    }
    .title {
      font-weight: 600;
      font-size: 15px;
    }
    .status {
      display: flex;
      align-items: center;
      gap: 8px;
      margin-bottom: 16px;
      padding: 10px;
      background: #f8fafc;
      border-radius: 8px;
    }
    .status-dot {
      width: 8px;
      height: 8px;
      border-radius: 50%;
      background: #22c55e;
    }
    .status-dot.disabled { background: #94a3b8; }
    .status-dot.error { background: #ef4444; }
    .status-text {
      font-size: 13px;
      color: #64748b;
    }
    .toggle {
      position: relative;
      width: 40px;
      height: 22px;
    }
    .toggle input {
      opacity: 0;
      width: 0;
      height: 0;
    }
    .toggle-slider {
      position: absolute;
      cursor: pointer;
      inset: 0;
      background: #cbd5e1;
      border-radius: 22px;
      transition: 0.2s;
    }
    .toggle-slider:before {
      position: absolute;
      content: "";
      height: 16px;
      width: 16px;
      left: 3px;
      bottom: 3px;
      background: white;
      border-radius: 50%;
      transition: 0.2s;
    }
    .toggle input:checked + .toggle-slider {
      background: #3b82f6;
    }
    .toggle input:checked + .toggle-slider:before {
      transform: translateX(18px);
    }
    .btn {
      width: 100%;
      padding: 10px;
      border: none;
      border-radius: 8px;
      background: #3b82f6;
      color: white;
      font-size: 14px;
      cursor: pointer;
      margin-bottom: 8px;
    }
    .btn:hover { background: #2563eb; }
    .btn-secondary {
      background: #f1f5f9;
      color: #475569;
    }
    .btn-secondary:hover { background: #e2e8f0; }
    .hint {
      font-size: 11px;
      color: #94a3b8;
      text-align: center;
      margin-top: 8px;
    }
  </style>
</head>
<body>
  <div class="header">
    <span class="title">Lifestream</span>
    <label class="toggle">
      <input type="checkbox" id="enableToggle" checked>
      <span class="toggle-slider"></span>
    </label>
  </div>
  
  <div class="status">
    <div class="status-dot" id="statusDot"></div>
    <span class="status-text" id="statusText">Ready to capture</span>
  </div>
  
  <button class="btn" id="captureBtn">Capture Now</button>
  <button class="btn btn-secondary" id="openDashboard">Open Dashboard</button>
  
  <div class="hint">Ctrl+Shift+S to capture anytime</div>
  
  <script src="popup.js"></script>
</body>
</html>
```

```javascript
// popup.js

async function updateUI() {
  const settings = await chrome.storage.local.get(['enabled']);
  const enabled = settings.enabled !== false;
  
  document.getElementById('enableToggle').checked = enabled;
  
  const statusDot = document.getElementById('statusDot');
  const statusText = document.getElementById('statusText');
  
  if (!enabled) {
    statusDot.className = 'status-dot disabled';
    statusText.textContent = 'Capture paused';
  } else {
    statusDot.className = 'status-dot';
    statusText.textContent = 'Ready to capture';
  }
}

document.getElementById('enableToggle').addEventListener('change', (e) => {
  chrome.storage.local.set({ enabled: e.target.checked });
  updateUI();
});

document.getElementById('captureBtn').addEventListener('click', () => {
  chrome.runtime.sendMessage({ action: 'capture_current' });
  
  const btn = document.getElementById('captureBtn');
  btn.textContent = 'Captured!';
  setTimeout(() => { btn.textContent = 'Capture Now'; }, 1000);
});

document.getElementById('openDashboard').addEventListener('click', () => {
  // Open the web UI if running, otherwise show instructions
  chrome.tabs.create({ url: 'http://localhost:7777' });
});

updateUI();
```

### 3.5 Native Messaging Host Manifest

Chrome requires a manifest file to locate the native host.

**Linux:** `~/.config/google-chrome/NativeMessagingHosts/com.lifestream.host.json`

```json
{
  "name": "com.lifestream.host",
  "description": "Lifestream capture host",
  "path": "/usr/local/bin/lifestream-host",
  "type": "stdio",
  "allowed_origins": [
    "chrome-extension://EXTENSION_ID_HERE/"
  ]
}
```

### 3.6 Extension File Structure

```
lifestream-extension/
├── manifest.json
├── background.js
├── content.js
├── popup.html
├── popup.js
└── icons/
    ├── icon16.png
    ├── icon48.png
    └── icon128.png
```

---

## 4. Native Host Specification (Rust)

### 4.1 Overview

The native host is a minimal Rust binary that:
1. Reads JSON messages from stdin (Chrome's Native Messaging protocol)
2. Validates the message structure
3. Appends to a JSONL queue file with proper locking
4. Writes a response to stdout

### 4.2 Native Messaging Protocol

Chrome sends messages with a 4-byte length prefix (little-endian), followed by JSON.

```
[4 bytes: length][JSON payload]
```

The host must respond in the same format.

### 4.3 Rust Implementation

```toml
# Cargo.toml
[package]
name = "lifestream-host"
version = "1.0.0"
edition = "2021"

[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
fs2 = "0.4"
dirs = "5.0"
chrono = { version = "0.4", features = ["serde"] }

[profile.release]
opt-level = "z"
lto = true
strip = true
```

```rust
// src/main.rs

use fs2::FileExt;
use serde::{Deserialize, Serialize};
use std::fs::{File, OpenOptions};
use std::io::{self, Read, Write};
use std::path::PathBuf;

#[derive(Debug, Deserialize)]
struct Snapshot {
    url: String,
    title: String,
    timestamp: String,
    text: String,
    images: Vec<Image>,
    html: String,
}

#[derive(Debug, Deserialize, Serialize)]
struct Image {
    src: String,
    alt: String,
    width: u32,
    height: u32,
}

#[derive(Debug, Serialize)]
struct QueueEntry {
    url: String,
    title: String,
    timestamp: String,
    text: String,
    images: Vec<Image>,
    html: String,
    received_at: String,
}

#[derive(Debug, Serialize)]
struct Response {
    success: bool,
    #[serde(skip_serializing_if = "Option::is_none")]
    error: Option<String>,
}

fn get_queue_path() -> PathBuf {
    let data_dir = dirs::data_local_dir()
        .unwrap_or_else(|| PathBuf::from("."))
        .join("lifestream");
    
    std::fs::create_dir_all(&data_dir).ok();
    data_dir.join("queue.jsonl")
}

fn read_message() -> io::Result<Vec<u8>> {
    let mut stdin = io::stdin();
    
    // Read 4-byte length prefix
    let mut len_bytes = [0u8; 4];
    stdin.read_exact(&mut len_bytes)?;
    let len = u32::from_le_bytes(len_bytes) as usize;
    
    // Sanity check (max 100MB)
    if len > 100 * 1024 * 1024 {
        return Err(io::Error::new(io::ErrorKind::InvalidData, "Message too large"));
    }
    
    // Read message body
    let mut buffer = vec![0u8; len];
    stdin.read_exact(&mut buffer)?;
    
    Ok(buffer)
}

fn write_message(response: &Response) -> io::Result<()> {
    let json = serde_json::to_vec(response)?;
    let len = json.len() as u32;
    
    let mut stdout = io::stdout();
    stdout.write_all(&len.to_le_bytes())?;
    stdout.write_all(&json)?;
    stdout.flush()?;
    
    Ok(())
}

fn append_to_queue(snapshot: Snapshot) -> io::Result<()> {
    let queue_path = get_queue_path();
    
    // Open file with append mode
    let mut file = OpenOptions::new()
        .create(true)
        .append(true)
        .open(&queue_path)?;
    
    // Lock file for exclusive access
    file.lock_exclusive()?;
    
    // Create queue entry with receive timestamp
    let entry = QueueEntry {
        url: snapshot.url,
        title: snapshot.title,
        timestamp: snapshot.timestamp,
        text: snapshot.text,
        images: snapshot.images,
        html: snapshot.html,
        received_at: chrono::Utc::now().to_rfc3339(),
    };
    
    // Write as JSON line
    let json = serde_json::to_string(&entry)?;
    writeln!(file, "{}", json)?;
    
    // Unlock
    file.unlock()?;
    
    Ok(())
}

fn process_message(data: &[u8]) -> Response {
    // Parse JSON
    let snapshot: Snapshot = match serde_json::from_slice(data) {
        Ok(s) => s,
        Err(e) => {
            return Response {
                success: false,
                error: Some(format!("Invalid JSON: {}", e)),
            };
        }
    };
    
    // Basic validation
    if snapshot.url.is_empty() {
        return Response {
            success: false,
            error: Some("Empty URL".to_string()),
        };
    }
    
    // Append to queue
    match append_to_queue(snapshot) {
        Ok(_) => Response {
            success: true,
            error: None,
        },
        Err(e) => Response {
            success: false,
            error: Some(format!("Failed to write: {}", e)),
        },
    }
}

fn main() {
    // Read message from Chrome
    let data = match read_message() {
        Ok(d) => d,
        Err(e) => {
            let response = Response {
                success: false,
                error: Some(format!("Failed to read: {}", e)),
            };
            write_message(&response).ok();
            return;
        }
    };
    
    // Process and respond
    let response = process_message(&data);
    write_message(&response).ok();
}
```

### 4.4 Build and Install

```bash
# Build release binary
cd lifestream-host
cargo build --release

# Install binary
sudo cp target/release/lifestream-host /usr/local/bin/

# Install native messaging manifest
mkdir -p ~/.config/google-chrome/NativeMessagingHosts
cat > ~/.config/google-chrome/NativeMessagingHosts/com.lifestream.host.json << 'EOF'
{
  "name": "com.lifestream.host",
  "description": "Lifestream capture host",
  "path": "/usr/local/bin/lifestream-host",
  "type": "stdio",
  "allowed_origins": [
    "chrome-extension://YOUR_EXTENSION_ID/"
  ]
}
EOF
```

### 4.5 Native Host File Structure

```
lifestream-host/
├── Cargo.toml
├── Cargo.lock
└── src/
    └── main.rs
```

---

## 5. Queue File Format

### 5.1 Location

```
~/.local/share/lifestream/queue.jsonl
```

### 5.2 Format

One JSON object per line (JSONL). Each entry contains:

```json
{
  "url": "https://mail.google.com/mail/u/0/#inbox/abc123",
  "title": "Important Email - user@example.com - Gmail",
  "timestamp": "2025-01-07T14:30:00.000Z",
  "text": "From: sender@example.com\nTo: me@example.com\n\nHello...",
  "images": [
    {"src": "https://...", "alt": "Profile", "width": 200, "height": 200}
  ],
  "html": "<!DOCTYPE html>...",
  "received_at": "2025-01-07T14:30:01.234Z"
}
```

### 5.3 Queue Management

The processor handles queue rotation:

1. Read all entries from `queue.jsonl`
2. Process each entry
3. Move processed entries to `queue.processed.jsonl` (for debugging)
4. Truncate `queue.jsonl`
5. Periodically delete old processed files

---

## 6. Python Processor Specification

### 6.1 Technology Stack

| Component | Technology | Rationale |
|-----------|------------|-----------|
| **Language** | Python 3.11+ | Rich ML/NLP ecosystem |
| **Database** | SQLite + FTS5 | Zero-config, portable |
| **Vector Store** | ChromaDB | Embedded, no server |
| **Embeddings** | sentence-transformers | Local, fast, private |
| **LLM** | Ollama (local) or Claude API | Summarization, RAG |
| **HTML Parsing** | BeautifulSoup4 | Robust HTML handling |
| **Scheduling** | systemd timer | Reliable, no daemon |

### 6.2 Data Model

```python
from datetime import datetime
from enum import Enum
from typing import Optional
from pydantic import BaseModel

class ContentType(str, Enum):
    EMAIL = "email"
    CHAT = "chat"
    DOCUMENT = "document"
    SOCIAL = "social"
    MEETING = "meeting"
    ARTICLE = "article"
    SEARCH = "search"
    GENERAL = "general"

class QueueEntry(BaseModel):
    """Raw entry from queue file."""
    url: str
    title: str
    timestamp: str
    text: str
    images: list[dict]
    html: str
    received_at: str

class Activity(BaseModel):
    """Processed and enriched activity record."""
    id: str
    url: str
    domain: str
    title: str
    timestamp: datetime
    captured_at: datetime
    content_type: ContentType
    text_content: str
    summary: Optional[str]
    metadata: dict
    images: list[dict]
    is_sensitive: bool
    redacted: bool
    embedding: Optional[list[float]]

class Conversation(BaseModel):
    """Aggregated conversation."""
    id: str
    content_type: ContentType
    source_domain: str
    title: str
    participants: list[str]
    message_count: int
    first_seen: datetime
    last_seen: datetime
    summary: Optional[str]
    activity_ids: list[str]
```

### 6.3 Database Schema

```sql
-- Core activities table
CREATE TABLE activities (
    id TEXT PRIMARY KEY,
    url TEXT NOT NULL,
    domain TEXT NOT NULL,
    title TEXT NOT NULL,
    timestamp DATETIME NOT NULL,
    captured_at DATETIME NOT NULL,
    content_type TEXT NOT NULL,
    text_content TEXT NOT NULL,
    summary TEXT,
    metadata TEXT,
    images TEXT,
    is_sensitive BOOLEAN DEFAULT FALSE,
    redacted BOOLEAN DEFAULT FALSE,
    content_hash TEXT
);

CREATE INDEX idx_activities_timestamp ON activities(timestamp DESC);
CREATE INDEX idx_activities_domain ON activities(domain);
CREATE INDEX idx_activities_type ON activities(content_type);
CREATE UNIQUE INDEX idx_activities_hash ON activities(content_hash);

-- Full-text search
CREATE VIRTUAL TABLE activities_fts USING fts5(
    title,
    text_content,
    summary,
    content='activities',
    content_rowid='rowid',
    tokenize='porter unicode61'
);

-- FTS sync triggers
CREATE TRIGGER activities_ai AFTER INSERT ON activities BEGIN
    INSERT INTO activities_fts(rowid, title, text_content, summary)
    VALUES (new.rowid, new.title, new.text_content, new.summary);
END;

CREATE TRIGGER activities_ad AFTER DELETE ON activities BEGIN
    INSERT INTO activities_fts(activities_fts, rowid, title, text_content, summary)
    VALUES('delete', old.rowid, old.title, old.text_content, old.summary);
END;

-- Conversations
CREATE TABLE conversations (
    id TEXT PRIMARY KEY,
    content_type TEXT NOT NULL,
    source_domain TEXT NOT NULL,
    title TEXT NOT NULL,
    participants TEXT,
    message_count INTEGER DEFAULT 1,
    first_seen DATETIME NOT NULL,
    last_seen DATETIME NOT NULL,
    summary TEXT,
    activity_ids TEXT
);

-- Processing state
CREATE TABLE processor_state (
    key TEXT PRIMARY KEY,
    value TEXT,
    updated_at DATETIME
);

-- Blocked domains
CREATE TABLE blocked_domains (
    domain TEXT PRIMARY KEY,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO blocked_domains (domain) VALUES
    ('chase.com'), ('bankofamerica.com'), ('wellsfargo.com'),
    ('citi.com'), ('capitalone.com'), ('fidelity.com'),
    ('schwab.com'), ('vanguard.com'), ('paypal.com'),
    ('venmo.com'), ('1password.com'), ('lastpass.com'),
    ('bitwarden.com'), ('dashlane.com');
```

### 6.4 Content Type Detection

```python
import re
from urllib.parse import urlparse

DOMAIN_PATTERNS = {
    ContentType.EMAIL: [
        r'mail\.google\.com',
        r'app\.hey\.com',
        r'outlook\.(live|office)\.com',
    ],
    ContentType.CHAT: [
        r'app\.slack\.com',
        r'discord\.com/channels',
        r'teams\.microsoft\.com',
    ],
    ContentType.DOCUMENT: [
        r'docs\.google\.com',
        r'notion\.so',
        r'dropbox\.com/paper',
    ],
    ContentType.SOCIAL: [
        r'twitter\.com',
        r'x\.com',
        r'linkedin\.com',
    ],
    ContentType.MEETING: [
        r'meet\.google\.com',
        r'zoom\.us',
    ],
    ContentType.SEARCH: [
        r'google\.com/search',
        r'duckduckgo\.com',
        r'bing\.com/search',
    ],
}

def detect_content_type(url: str) -> ContentType:
    for content_type, patterns in DOMAIN_PATTERNS.items():
        for pattern in patterns:
            if re.search(pattern, url, re.IGNORECASE):
                return content_type
    return ContentType.GENERAL
```

### 6.5 Parsers

```python
from abc import ABC, abstractmethod
from bs4 import BeautifulSoup

class BaseParser(ABC):
    @abstractmethod
    def parse(self, entry: QueueEntry) -> dict:
        pass
    
    @abstractmethod
    def extract_clean_text(self, entry: QueueEntry) -> str:
        pass

class GmailParser(BaseParser):
    def parse(self, entry: QueueEntry) -> dict:
        metadata = {
            'email_from': None,
            'email_to': [],
            'subject': None,
            'thread_id': None,
        }
        
        # Extract subject from title
        if ' - ' in entry.title:
            parts = entry.title.split(' - ')
            if len(parts) >= 3 and 'Gmail' in parts[-1]:
                metadata['subject'] = parts[0]
        
        # Extract thread ID from URL
        if '#' in entry.url:
            hash_part = entry.url.split('#')[-1]
            parts = hash_part.split('/')
            if len(parts) >= 2:
                metadata['thread_id'] = parts[-1]
        
        return metadata
    
    def extract_clean_text(self, entry: QueueEntry) -> str:
        soup = BeautifulSoup(entry.html, 'html.parser')
        for elem in soup.find_all(['script', 'style', 'nav']):
            elem.decompose()
        main = soup.find('div', {'role': 'main'})
        if main:
            return main.get_text(separator='\n', strip=True)
        return entry.text

class SlackParser(BaseParser):
    def parse(self, entry: QueueEntry) -> dict:
        metadata = {
            'workspace': None,
            'channel': None,
            'channel_type': 'channel',
            'participants': [],
        }
        
        url_parts = entry.url.split('/')
        for i, part in enumerate(url_parts):
            if part.startswith('T') and len(part) > 8:
                metadata['workspace'] = part
            if part.startswith('C') and len(part) > 8:
                metadata['channel'] = part
            if part.startswith('D') and len(part) > 8:
                metadata['channel'] = part
                metadata['channel_type'] = 'dm'
        
        return metadata
    
    def extract_clean_text(self, entry: QueueEntry) -> str:
        soup = BeautifulSoup(entry.html, 'html.parser')
        messages = []
        for msg in soup.find_all('div', {'data-qa': 'message_container'}):
            sender = msg.find('button', {'data-qa': 'message_sender_name'})
            content = msg.find('div', {'data-qa': 'message_content'})
            sender_name = sender.get_text(strip=True) if sender else 'Unknown'
            msg_text = content.get_text(strip=True) if content else ''
            if msg_text:
                messages.append(f"{sender_name}: {msg_text}")
        return '\n'.join(messages) if messages else entry.text

class GenericParser(BaseParser):
    def parse(self, entry: QueueEntry) -> dict:
        return {}
    
    def extract_clean_text(self, entry: QueueEntry) -> str:
        soup = BeautifulSoup(entry.html, 'html.parser')
        for elem in soup.find_all(['script', 'style', 'nav', 'footer', 'aside']):
            elem.decompose()
        main = (
            soup.find('main') or
            soup.find('article') or
            soup.find('div', {'role': 'main'}) or
            soup.body
        )
        if main:
            return main.get_text(separator='\n', strip=True)
        return entry.text

PARSERS = {
    ContentType.EMAIL: GmailParser(),
    ContentType.CHAT: SlackParser(),
    ContentType.GENERAL: GenericParser(),
}

def get_parser(content_type: ContentType) -> BaseParser:
    return PARSERS.get(content_type, GenericParser())
```

### 6.6 Sanitization

```python
import re

REDACT_PATTERNS = [
    (r'(password|passwd|pwd|secret|token|api[_-]?key)\s*[:=]\s*[\'"]?[\w\-\.]+[\'"]?', '[REDACTED]'),
    (r'\b\d{4}[\s\-]?\d{4}[\s\-]?\d{4}[\s\-]?\d{4}\b', '[REDACTED_CC]'),
    (r'\b\d{3}[\s\-]?\d{2}[\s\-]?\d{4}\b', '[REDACTED_SSN]'),
    (r'(verification|auth|2fa|otp)\s*(code|pin)?\s*[:=]?\s*\d{4,8}', '[REDACTED_2FA]'),
]

def sanitize_text(text: str) -> tuple[str, bool]:
    redacted = False
    for pattern, replacement in REDACT_PATTERNS:
        if re.search(pattern, text, re.IGNORECASE):
            text = re.sub(pattern, replacement, text, flags=re.IGNORECASE)
            redacted = True
    return text, redacted

def is_domain_blocked(domain: str, blocked: set[str]) -> bool:
    parts = domain.split('.')
    for i in range(len(parts)):
        if '.'.join(parts[i:]) in blocked:
            return True
    return False
```

### 6.7 Main Processor

```python
import json
import hashlib
import uuid
from datetime import datetime
from pathlib import Path
from sentence_transformers import SentenceTransformer

class Processor:
    def __init__(self, db, vector_db):
        self.db = db
        self.vector_db = vector_db
        self.embedder = SentenceTransformer('all-MiniLM-L6-v2')
        self.blocked_domains = self.db.get_blocked_domains()
        
        self.queue_path = Path.home() / '.local/share/lifestream/queue.jsonl'
        self.processed_path = Path.home() / '.local/share/lifestream/queue.processed.jsonl'
    
    def run(self):
        """Process all entries in the queue."""
        if not self.queue_path.exists():
            return
        
        # Read all entries
        entries = []
        with open(self.queue_path, 'r') as f:
            for line in f:
                line = line.strip()
                if line:
                    try:
                        entries.append(QueueEntry(**json.loads(line)))
                    except Exception as e:
                        print(f"Failed to parse line: {e}")
        
        if not entries:
            return
        
        # Process entries
        activities = []
        for entry in entries:
            activity = self.process_entry(entry)
            if activity:
                activities.append(activity)
        
        # Batch embed
        if activities:
            texts = [a.text_content[:8000] for a in activities]
            embeddings = self.embedder.encode(texts, batch_size=32)
            for activity, embedding in zip(activities, embeddings):
                activity.embedding = embedding.tolist()
        
        # Store
        for activity in activities:
            self.db.insert_activity(activity)
            self.vector_db.add(activity.id, activity.embedding, {
                'content_type': activity.content_type.value,
                'domain': activity.domain,
            })
        
        # Archive processed entries
        with open(self.processed_path, 'a') as f:
            for entry in entries:
                f.write(json.dumps(entry.dict()) + '\n')
        
        # Clear queue
        self.queue_path.write_text('')
        
        print(f"Processed {len(activities)} activities from {len(entries)} entries")
    
    def process_entry(self, entry: QueueEntry) -> Activity | None:
        """Process a single queue entry."""
        from urllib.parse import urlparse
        
        # Extract domain
        parsed = urlparse(entry.url)
        domain = parsed.netloc.lower()
        
        # Check blocklist
        if is_domain_blocked(domain, self.blocked_domains):
            return None
        
        # Detect content type
        content_type = detect_content_type(entry.url)
        
        # Parse with appropriate parser
        parser = get_parser(content_type)
        metadata = parser.parse(entry)
        clean_text = parser.extract_clean_text(entry)
        
        # Sanitize
        clean_text, redacted = sanitize_text(clean_text)
        
        # Check for duplicates
        content_hash = hashlib.sha256(clean_text.encode()).hexdigest()[:32]
        if self.db.exists_by_hash(content_hash):
            return None
        
        # Create activity
        return Activity(
            id=str(uuid.uuid4()),
            url=entry.url,
            domain=domain,
            title=entry.title,
            timestamp=datetime.fromisoformat(entry.timestamp.replace('Z', '+00:00')),
            captured_at=datetime.fromisoformat(entry.received_at.replace('Z', '+00:00')),
            content_type=content_type,
            text_content=clean_text,
            summary=None,
            metadata=metadata,
            images=entry.images,
            is_sensitive=False,
            redacted=redacted,
            embedding=None,
        )
```

### 6.8 Systemd Timer

```ini
# ~/.config/systemd/user/lifestream-processor.service
[Unit]
Description=Lifestream Queue Processor

[Service]
Type=oneshot
ExecStart=/usr/local/bin/lifestream process
```

```ini
# ~/.config/systemd/user/lifestream-processor.timer
[Unit]
Description=Run Lifestream processor every 5 minutes

[Timer]
OnBootSec=1min
OnUnitActiveSec=5min
AccuracySec=1min

[Install]
WantedBy=timers.target
```

```bash
# Enable and start
systemctl --user daemon-reload
systemctl --user enable lifestream-processor.timer
systemctl --user start lifestream-processor.timer
```

---

## 7. Query Layer

### 7.1 Search Engine

```python
from dataclasses import dataclass

@dataclass
class SearchResult:
    activity: Activity
    score: float
    snippet: str

class SearchEngine:
    def __init__(self, db, vector_db, embedder):
        self.db = db
        self.vector_db = vector_db
        self.embedder = embedder
    
    def search(
        self,
        query: str,
        mode: str = "hybrid",
        content_types: list[str] | None = None,
        domains: list[str] | None = None,
        since: datetime | None = None,
        until: datetime | None = None,
        limit: int = 20,
    ) -> list[SearchResult]:
        
        if mode == "fulltext":
            return self._fulltext_search(query, content_types, domains, since, until, limit)
        elif mode == "semantic":
            return self._semantic_search(query, content_types, domains, since, until, limit)
        else:
            fts = self._fulltext_search(query, content_types, domains, since, until, limit * 2)
            sem = self._semantic_search(query, content_types, domains, since, until, limit * 2)
            return self._reciprocal_rank_fusion([fts, sem])[:limit]
    
    def _fulltext_search(self, query, content_types, domains, since, until, limit):
        # SQLite FTS5 query
        ...
    
    def _semantic_search(self, query, content_types, domains, since, until, limit):
        embedding = self.embedder.encode(query)
        results = self.vector_db.query(embedding, limit=limit)
        ...
    
    def _reciprocal_rank_fusion(self, result_lists, k=60):
        scores = {}
        items = {}
        for results in result_lists:
            for rank, result in enumerate(results):
                aid = result.activity.id
                if aid not in scores:
                    scores[aid] = 0
                    items[aid] = result
                scores[aid] += 1 / (k + rank + 1)
        sorted_ids = sorted(scores, key=scores.get, reverse=True)
        return [items[aid] for aid in sorted_ids]
```

### 7.2 RAG Pipeline

```python
class RAGPipeline:
    def __init__(self, search_engine, llm):
        self.search = search_engine
        self.llm = llm
    
    def ask(self, question: str, context_limit: int = 10) -> dict:
        results = self.search.search(question, mode="hybrid", limit=context_limit)
        
        if not results:
            return {
                "answer": "I couldn't find relevant information in your archive.",
                "sources": []
            }
        
        context_parts = []
        sources = []
        for r in results:
            a = r.activity
            context_parts.append(
                f"[{a.content_type.value.upper()}] {a.timestamp:%Y-%m-%d %H:%M}\n"
                f"Source: {a.domain}\n"
                f"Content: {a.text_content[:1500]}"
            )
            sources.append({
                "id": a.id,
                "type": a.content_type.value,
                "title": a.title,
                "url": a.url,
                "timestamp": a.timestamp.isoformat(),
            })
        
        prompt = f"""Answer based on these activities from my archive:

Question: {question}

Context:
{chr(10).join(context_parts)}

Answer concisely based only on the context above."""

        answer = self.llm.complete(prompt)
        return {"answer": answer, "sources": sources}
```

### 7.3 CLI

```python
import typer
from rich.console import Console
from rich.table import Table

app = typer.Typer(name="lifestream")
console = Console()

@app.command()
def process():
    """Process the capture queue."""
    processor = Processor(db, vector_db)
    processor.run()

@app.command()
def search(
    query: str,
    mode: str = typer.Option("hybrid", help="fulltext, semantic, hybrid"),
    limit: int = typer.Option(10),
):
    """Search your activity archive."""
    results = search_engine.search(query, mode=mode, limit=limit)
    
    table = Table()
    table.add_column("Type")
    table.add_column("Date")
    table.add_column("Title")
    table.add_column("Domain")
    
    for r in results:
        a = r.activity
        table.add_row(
            a.content_type.value,
            a.timestamp.strftime("%Y-%m-%d %H:%M"),
            a.title[:50],
            a.domain,
        )
    
    console.print(table)

@app.command()
def ask(question: str):
    """Ask a question about your activities."""
    result = rag.ask(question)
    console.print(f"\n[bold]{result['answer']}[/bold]\n")
    
    if result['sources']:
        console.print("[dim]Sources:[/dim]")
        for s in result['sources']:
            console.print(f"  • [{s['type']}] {s['title'][:40]}")

@app.command()
def stats():
    """Show capture statistics."""
    stats = db.get_stats()
    console.print(f"Total activities: {stats['total']}")
    console.print(f"Today: {stats['today']}")
    console.print(f"By type: {stats['by_type']}")

@app.command()
def serve(
    host: str = typer.Option("127.0.0.1"),
    port: int = typer.Option(7777),
):
    """Start the web UI."""
    import uvicorn
    uvicorn.run("lifestream.api:app", host=host, port=port)

@app.command()
def block(domain: str):
    """Add domain to blocklist."""
    db.add_blocked_domain(domain)
    console.print(f"Blocked: {domain}")

@app.command()
def unblock(domain: str):
    """Remove domain from blocklist."""
    db.remove_blocked_domain(domain)
    console.print(f"Unblocked: {domain}")

if __name__ == "__main__":
    app()
```

### 7.4 Optional Web API

```python
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles

app = FastAPI(title="Lifestream")

@app.get("/health")
def health():
    return {"status": "ok", "queue_size": get_queue_size()}

@app.get("/activities")
def list_activities(limit: int = 50, offset: int = 0):
    return db.list_activities(limit=limit, offset=offset)

@app.post("/search")
def search(query: str, mode: str = "hybrid", limit: int = 20):
    return search_engine.search(query, mode=mode, limit=limit)

@app.post("/ask")
def ask(question: str):
    return rag.ask(question)

# Mount static files for web UI
app.mount("/", StaticFiles(directory="static", html=True))
```

---

## 8. Directory Structure

```
lifestream/
├── README.md
├── install.sh                    # Installation script
│
├── extension/                    # Chrome extension
│   ├── manifest.json
│   ├── background.js
│   ├── content.js
│   ├── popup.html
│   ├── popup.js
│   └── icons/
│
├── host/                         # Rust native host
│   ├── Cargo.toml
│   └── src/
│       └── main.rs
│
├── processor/                    # Python processor & query layer
│   ├── pyproject.toml
│   └── src/
│       └── lifestream/
│           ├── __init__.py
│           ├── cli.py
│           ├── config.py
│           ├── models.py
│           ├── db/
│           │   ├── __init__.py
│           │   ├── sqlite.py
│           │   └── vectors.py
│           ├── pipeline/
│           │   ├── __init__.py
│           │   ├── processor.py
│           │   ├── detection.py
│           │   ├── parsers.py
│           │   └── sanitizer.py
│           ├── search/
│           │   ├── __init__.py
│           │   └── engine.py
│           ├── ai/
│           │   ├── __init__.py
│           │   ├── embeddings.py
│           │   └── rag.py
│           └── api/
│               ├── __init__.py
│               └── routes.py
│
├── systemd/                      # Systemd units
│   ├── lifestream-processor.service
│   └── lifestream-processor.timer
│
└── tests/
```

---

## 9. Data Directory Structure

```
~/.local/share/lifestream/
├── queue.jsonl              # Incoming captures (written by native host)
├── queue.processed.jsonl    # Processed entries (for debugging)
├── lifestream.db            # SQLite database
├── vectors/                 # ChromaDB storage
└── images/                  # Downloaded images (optional)
```

---

## 10. Installation

```bash
#!/bin/bash
# install.sh

set -e

echo "Installing Lifestream..."

# 1. Build and install native host
echo "Building native host..."
cd host
cargo build --release
sudo cp target/release/lifestream-host /usr/local/bin/
cd ..

# 2. Install native messaging manifest
echo "Installing native messaging manifest..."
mkdir -p ~/.config/google-chrome/NativeMessagingHosts

# Get extension ID (user must update this after loading extension)
read -p "Enter your Chrome extension ID: " EXT_ID

cat > ~/.config/google-chrome/NativeMessagingHosts/com.lifestream.host.json << EOF
{
  "name": "com.lifestream.host",
  "description": "Lifestream capture host",
  "path": "/usr/local/bin/lifestream-host",
  "type": "stdio",
  "allowed_origins": [
    "chrome-extension://${EXT_ID}/"
  ]
}
EOF

# 3. Install Python processor
echo "Installing Python processor..."
cd processor
pip install -e .
cd ..

# 4. Create data directory
mkdir -p ~/.local/share/lifestream

# 5. Install systemd timer
echo "Installing systemd timer..."
mkdir -p ~/.config/systemd/user
cp systemd/lifestream-processor.service ~/.config/systemd/user/
cp systemd/lifestream-processor.timer ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable lifestream-processor.timer
systemctl --user start lifestream-processor.timer

echo ""
echo "Installation complete!"
echo ""
echo "Next steps:"
echo "1. Load the extension from extension/ in Chrome (chrome://extensions)"
echo "2. Copy the extension ID and update ~/.config/google-chrome/NativeMessagingHosts/com.lifestream.host.json"
echo "3. Reload the extension"
echo ""
echo "Commands:"
echo "  lifestream stats     - Show statistics"
echo "  lifestream search    - Search your archive"
echo "  lifestream ask       - Ask a question"
echo "  lifestream serve     - Start web UI"
```

---

## 11. Implementation Phases

### Phase 1: Native Host (Week 1)
- [ ] Rust project setup
- [ ] Native messaging protocol implementation
- [ ] Queue file writing with locking
- [ ] Build and packaging

### Phase 2: Chrome Extension (Week 1-2)
- [ ] Manifest and permissions
- [ ] Content script for extraction
- [ ] Background script with native messaging
- [ ] Popup UI
- [ ] Testing with native host

### Phase 3: Processor Core (Week 2-3)
- [ ] Queue file reading
- [ ] Content type detection
- [ ] Basic parsers (Gmail, Slack, generic)
- [ ] Sanitization
- [ ] SQLite storage

### Phase 4: Search (Week 3-4)
- [ ] FTS5 integration
- [ ] Embedding generation
- [ ] ChromaDB integration
- [ ] Hybrid search
- [ ] CLI search command

### Phase 5: AI Features (Week 4-5)
- [ ] RAG implementation
- [ ] Ask command
- [ ] Conversation summarization

### Phase 6: Polish (Week 5-6)
- [ ] Web UI
- [ ] Installation script
- [ ] Systemd units
- [ ] Documentation
- [ ] Additional parsers

---

## 12. Example Usage

```bash
# Check queue status
cat ~/.local/share/lifestream/queue.jsonl | wc -l
# Output: 23 entries waiting

# Manually trigger processing
lifestream process
# Output: Processed 23 activities

# Search
lifestream search "project proposal"
# ┌────────┬──────────────────┬─────────────────────────┬─────────────────┐
# │ Type   │ Date             │ Title                   │ Domain          │
# ├────────┼──────────────────┼─────────────────────────┼─────────────────┤
# │ email  │ 2025-01-05 14:30 │ Re: Project Proposal    │ mail.google.com │
# │ chat   │ 2025-01-05 11:20 │ #product                │ app.slack.com   │
# │ doc    │ 2025-01-04 16:45 │ Q1 Proposal Draft       │ docs.google.com │
# └────────┴──────────────────┴─────────────────────────┴─────────────────┘

# Ask a question
lifestream ask "What was decided about the launch date?"
# Based on your Slack conversation in #product on January 5th, the team
# decided to push the launch to February 15th to allow more QA time.
#
# Sources:
#   • [chat] #product - app.slack.com
#   • [email] Re: Launch Timeline

# Start web UI
lifestream serve
# Running on http://127.0.0.1:7777
```

---

## 13. Trade-offs & Considerations

### Benefits of This Architecture

1. **No always-on daemon for capture** - Native host only runs when Chrome sends a message
2. **Instant capture** - Just appending JSON, no processing delay
3. **Graceful degradation** - Capture works even if Python isn't installed
4. **Efficient batching** - Embeddings generated in batches (GPU-friendly)
5. **Debuggable** - Queue is human-readable JSONL
6. **Simple extension** - No HTTP, no complex state management

### Trade-offs

1. **5-minute delay** - Search won't include the last few minutes of activity
2. **Two languages** - Rust for host, Python for processing
3. **Systemd dependency** - Timer-based processing (could use cron instead)
4. **Extension ID requirement** - Must update manifest after loading extension

### Mitigations

- Reduce timer interval if fresher data needed (e.g., 1 minute)
- Could add manual "process now" CLI command
- Could add inotify watcher for instant processing (optional enhancement)

---

*Document Version: 3.0*
*Architecture: Native Messaging → File Queue → Batch Processor*
*Last Updated: 2025-01-07*
