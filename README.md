### 1. Prerequisites (For Windows Users)

**Windows users must be running WSL (Windows Subsystem for Linux).**

* Open PowerShell as Administrator and run: `wsl --install`
* Reboot if prompted, then open "Ubuntu" from your start menu to set up your UNIX username and password.

---

### 2. Download, Build, and Install

Run the following commands inside your WSL (Ubuntu) terminal.

**Install Build Utilities**

```bash
sudo apt-get update
sudo apt-get install build-essential libpcre3 libpcre3-dev libssl-dev git zlib1g-dev -y

```

**Make and CD to build directory**

```bash
sudo mkdir ~/build && cd ~/build

```

**Download & unpack latest nginx-rtmp module**

```bash
sudo git clone https://github.com/arut/nginx-rtmp-module.git

```

**Download & unpack the LATEST Nginx (Version 1.29.4)**
*Updated from the original 1.14 version.*

```bash
sudo wget http://nginx.org/download/nginx-1.29.4.tar.gz
sudo tar xzf nginx-1.29.4.tar.gz
cd nginx-1.29.4

```

**Build Nginx with nginx-rtmp**

```bash
sudo ./configure --with-http_ssl_module --add-module=../nginx-rtmp-module
sudo make
sudo make install

```

**Start Nginx Server**

```bash
sudo /usr/local/nginx/sbin/nginx

```

*(To test if it works, open a browser in Windows and go to `http://localhost:8080`. You should see the "Welcome to nginx!" page or a 404 if the config isn't set yet).*

---

### 3. Set up Live Streaming Config

We need to replace the default config with the RTMP-enabled config.

**Open the config file:**

```bash
sudo nano /usr/local/nginx/conf/nginx.conf

```

**Delete everything in that file and paste this instead:**

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

        # rtmp stat
        location /stat {
            rtmp_stat all;
            rtmp_stat_stylesheet stat.xsl;
        }
        location /stat.xsl {
            # root to the library we cloned earlier
            root /home/YOUR_USERNAME/build/nginx-rtmp-module; 
        }

        # rtmp control
        location /control {
            rtmp_control all;
        }

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

            # sample recorder
            #recorder rec1 {
            #    record all;
            #    record_interval 30s;
            #    record_path /tmp;
            #    record_unique on;
            #}

            # sample HLS
            #hls on;
            #hls_path /tmp/hls;
            #hls_sync 100ms;
        }
    }
}

```

*Note: I updated the `root` path in `/stat.xsl` to point to `~/build/nginx-rtmp-module`. You may need to replace `YOUR_USERNAME` with your actual Linux username, or just use `/usr/build/...` if you created the folder at the system root in step 2.*

**Restart Nginx to apply changes:**

```bash
sudo /usr/local/nginx/sbin/nginx -s stop
sudo /usr/local/nginx/sbin/nginx

```

---

### 4. Publishing with FFmpeg

**Install FFmpeg**

```bash
sudo apt install ffmpeg -y

```

**Test Stream (File Loop)**
This is the most reliable test for WSL users.

```bash
# Download a dummy video first
wget https://sample-videos.com/video321/mp4/720/big_buck_bunny_720p_1mb.mp4 -O test.mp4

# Stream it in a loop
ffmpeg -re -stream_loop -1 -i test.mp4 -c copy -f flv rtmp://localhost/myapp/mystream

```

**Webcam Streaming (Note for WSL Users)**
*Warning: Standard WSL does not have direct access to USB webcams. Running the command below will likely fail on Windows unless you have set up `usbipd-win` to attach the USB device to Linux.*

If you are on **Native Linux**, use:

```bash
ffmpeg -f video4linux2 -i /dev/video0 -c:v libx264 -an -f flv rtmp://localhost/myapp/mystream

```

---

### 5. Playing the Stream

**Using VLC (Recommended for Windows)**

1. Open VLC Media Player on Windows.
2. Go to **Media > Open Network Stream**.
3. Enter: `rtmp://localhost/myapp/mystream`
4. Click Play.

**Using FFplay (Inside WSL)**

```bash
ffplay rtmp://localhost/myapp/mystream

```

**Using OBS (Open Broadcaster Software)**

* **Service:** Custom
* **Server:** `rtmp://localhost/myapp`
* **Stream Key:** `mystream`

---

### 6. Statistics

Navigate your Windows browser to `http://localhost:8080/stat` to see the live bandwidth and client info.

Would you like me to create a quick troubleshooting cheat sheet for common "Port 1935 in use" or "Permission denied" errors that might pop up during the session?