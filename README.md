# Building a Live Streaming Server (Nginx + RTMP + DASH)

## 1. The Architecture: Under the Hood

Before we type a single command, it is important to understand what we are building. This setup mimics the architecture used by platforms like YouTube and Twitch, but on a micro-scale.

### The Pipeline

**`[Broadcaster]`**  **`[Nginx Server]`**  **`[Client/Viewer]`**

### The Protocols

We use two different protocols because they serve different purposes:

1. **Ingestion (RTMP - Real-Time Messaging Protocol):**
* **What it does:** Creates a persistent TCP "pipe" between the broadcaster (OBS/FFmpeg) and the server.
* **Why we use it:** It is low-latency and stateful. It is perfect for getting video *to* the server, but it doesn't scale well for thousands of viewers.


2. **Delivery (DASH - Dynamic Adaptive Streaming over HTTP):**
* **What it does:** The server chops the continuous video stream into tiny, static files (chunks) and creates a playlist (manifest).
* **Why we use it:** It uses standard HTTP (stateless). This means video can be cached by browsers and CDNs, allowing for massive scalability.



**The Role of `nginx-rtmp-module`:**
It acts as the translator. It listens for the incoming RTMP stream, instantly fragments it into DASH chunks, and serves those chunks via standard HTTP.

---

## 2. Prerequisites

**For Windows Users:**
You must be running **WSL (Windows Subsystem for Linux)**.

1. Open PowerShell as Administrator.
2. Run: `wsl --install`
3. Reboot if prompted, then open "Ubuntu" from your start menu to set up your UNIX username/password.

**For Linux/Mac Users:**
Open your standard terminal.

---

## 3. Installation

Run the following commands inside your terminal (WSL/Ubuntu).

### Step A: Install Dependencies

```bash
sudo apt-get update
sudo apt-get install build-essential libpcre3 libpcre3-dev libssl-dev git zlib1g-dev ffmpeg -y

```

### Step B: Download Sources

We will download the RTMP module and the **latest Nginx (v1.29.4)**.

```bash
sudo mkdir ~/build && cd ~/build

# Clone the RTMP module
sudo git clone https://github.com/arut/nginx-rtmp-module.git

# Download Nginx
sudo wget http://nginx.org/download/nginx-1.29.4.tar.gz
sudo tar xzf nginx-1.29.4.tar.gz
cd nginx-1.29.4

```

### Step C: Build and Install

```bash
# Configure Nginx with the RTMP module
sudo ./configure --with-http_ssl_module --add-module=../nginx-rtmp-module

# Compile
sudo make
sudo make install

```

---

## 4. Configuration

### Step A: Prepare Directories & Files

We need a folder for the DASH video fragments and we need to move the statistics styling file to a place where Nginx can actually find it.

```bash
# Create DASH storage folder
sudo mkdir -p /tmp/dash
sudo chmod 777 /tmp/dash

# Copy the stats styling file to the default web root
sudo cp ~/build/nginx-rtmp-module/stat.xsl /usr/local/nginx/html/

```

### Step B: Edit nginx.conf

Open the configuration file:

```bash
sudo nano /usr/local/nginx/conf/nginx.conf

```

**Delete everything** in that file and paste the following configuration. This sets up both the RTMP ingest and the DASH HTTP delivery.

```nginx
#user  nobody;
worker_processes  1;

error_log  logs/error.log debug;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       8080;
        server_name  localhost;

        # 1. Dashboard for Statistics
        location /stat {
            rtmp_stat all;
            rtmp_stat_stylesheet stat.xsl;
        }
        location /stat.xsl {
            # We moved the file here in Step 4A
            root /usr/local/nginx/html;
        }

        # 2. DASH Output (Playable in Browser)
        location /dash {
            # Serve DASH fragments from /tmp/dash
            root /tmp;
            add_header Cache-Control no-cache;
            
            # Enable CORS so web players can access the stream
            add_header Access-Control-Allow-Origin *;
        }

        # Error pages
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}

rtmp {
    server {
        listen 1935;
        ping 30s;
        notify_method get;

        application myapp {
            live on;

            # Enable DASH
            dash on;
            dash_path /tmp/dash;
            dash_fragment 3; # 3-second chunks
        }
    }
}

```

### Step C: Start Nginx

```bash
sudo /usr/local/nginx/sbin/nginx

```

*(If you need to restart it later after changing config, use: `sudo /usr/local/nginx/sbin/nginx -s reload`)*

---

## 5. Broadcasting (Ingest)

We will use FFmpeg to simulate a live stream.

**Option A: Loop a Video File (Best for testing)**

```bash
# Download a test video
wget https://sample-videos.com/video321/mp4/720/big_buck_bunny_720p_1mb.mp4 -O test.mp4

# Stream it forever
ffmpeg -re -stream_loop -1 -i test.mp4 -c copy -f flv rtmp://localhost/myapp/mystream

```

**Option B: OBS Studio**

* **Service:** Custom
* **Server:** `rtmp://localhost/myapp` (Or `rtmp://YOUR_WSL_IP/myapp` if localhost fails)
* **Stream Key:** `mystream`

---

## 6. Watching the Stream (Egress)

### Statistics Dashboard

Go to: `http://localhost:8080/stat`
*You should see a table showing "myapp", "mystream", and data throughput.*

### Playing via DASH (HTTP)

The URL structure is: `http://localhost:8080/dash/STREAM_KEY.mpd`
**URL:** `http://localhost:8080/dash/mystream.mpd`

**How to watch:**

1. **VLC:** Go to *Media > Open Network Stream* and paste the URL.
2. **Web:** Go to the [Dash.js Reference Player](https://reference.dashif.org/dash.js/latest/samples/dash-if-reference-player/index.html) and paste the URL.

---

## 7. Troubleshooting Cheat Sheet

**1. "Address already in use" Error**

* *Cause:* You tried to start Nginx but it's already running.
* *Fix:* `sudo killall nginx` then start it again.

**2. Stats page shows raw code (XML) instead of a table**

* *Cause:* Nginx can't find `stat.xsl`.
* *Fix:* Ensure you ran the `cp` command in **Step 4A** and that your `nginx.conf` points to `/usr/local/nginx/html`.

**3. OBS cannot connect to "localhost"**

* *Cause:* Windows sometimes struggles to resolve `localhost` to the WSL Linux instance.
* *Fix:* In WSL terminal, run `ip addr` to find your eth0 IP (e.g., 172.x.x.x). Use that IP in OBS instead of localhost.

**4. Permission Denied (Video won't play)**

* *Cause:* Nginx cannot write to the temp folder.
* *Fix:* `sudo chmod 777 /tmp/dash`