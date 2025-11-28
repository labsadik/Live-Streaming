# **Live Streaming Platform with Nginx-RTMP, Supabase (or Alternative DB), and HLS**

## **Introduction**

This project provides a **complete live streaming solution** where:

* **Admins** can securely stream video via OBS using dynamically generated RTMP keys.
* **Users** can watch live streams on a web page with HLS playback.
* Streams are managed using **Supabase** (or an alternative DBMS like PostgreSQL/MySQL).
* The platform works on **Windows** and **Linux VPS**, and can run **24/7**.

**Key Features:**

* Admin-only dashboard to create/manage streams.
* Dynamic RTMP keys for secure streaming.
* HLS playback via HTML, Tailwind CSS, and HLS.js.
* Supports both local (Windows) and public (VPS) deployment.
* Optional alternative DBMS for those who prefer MySQL/PostgreSQL.

---

## **Project Structure**

```
live-stream-app/
│
├─ nginx-rtmp/                # Nginx-RTMP folder
│   ├─ conf/nginx.conf        # Nginx + RTMP configuration
│   ├─ hls/                   # HLS output folder
│   ├─ html/                  # Web files served by Nginx
│   │   ├─ index.html         # User viewing page
│   │   └─ admin.html         # Admin dashboard
│   └─ nginx.exe (Windows) or nginx (Linux)
│
├─ backend/                   # Node.js backend (optional)
│   ├─ api.js                 # Dynamic RTMP key creation & validation
│   └─ server.js              # Express server for Supabase API endpoints
│
├─ supabase/                  # Supabase setup (or alternative DB scripts)
│   └─ tables.sql             # SQL script for users & streams tables
│
└─ README.md                  # This file
```

---

## **1. Database Setup (Supabase / Alternative DBMS)**

You need a **database** to manage users, roles, and streams.

### **1.1 Supabase (Recommended)**

1. Sign up at [Supabase](https://supabase.com/) and create a project.
2. Enable **Authentication** (email/password).
3. Create tables:

**Users table:**

| Column   | Type | Notes             |
| -------- | ---- | ----------------- |
| id       | uuid | Primary Key       |
| email    | text | Unique            |
| role     | text | 'admin' or 'user' |
| rtmp_key | text | Unique per admin  |

**Streams table:**

| Column     | Type      | Notes                |
| ---------- | --------- | -------------------- |
| id         | uuid      | Primary Key          |
| title      | text      | Stream title         |
| rtmp_key   | text      | Generated for stream |
| status     | text      | 'live' / 'offline'   |
| created_at | timestamp | Default now()        |

### **1.2 Alternative DBMS**

* Use **PostgreSQL/MySQL** if you don’t want Supabase.
* Create the same tables.
* Use Node.js backend or PHP/Python API to generate dynamic RTMP keys.

---

## **2. Windows Setup**

### **2.1 Requirements**

* Windows 10/11
* VS Code
* OBS Studio
* Supabase account or database credentials

### **2.2 Install Nginx-RTMP**

1. Download **prebuilt Nginx-RTMP for Windows** (`nginx-rtmp-win32.zip`).
2. Extract to `C:\nginx-rtmp`.
3. Create folder for HLS:

```powershell
mkdir C:\nginx-rtmp\hls
```

4. Configure `C:\nginx-rtmp\conf\nginx.conf` (example below).

### **2.3 Nginx Configuration Example**

```nginx
worker_processes 1;
events { worker_connections 1024; }

http {
    include mime.types;
    default_type application/octet-stream;

    server {
        listen 8080;

        location / {
            root html;
            index index.html index.htm;
        }

        location /hls {
            types { application/vnd.apple.mpegurl m3u8; video/mp2t ts; }
            root C:/nginx-rtmp;
            add_header Cache-Control no-cache;
        }
    }
}

rtmp {
    server {
        listen 1935;
        chunk_size 4096;

        application live {
            live on;
            record off;
            hls on;
            hls_path C:/nginx-rtmp/hls;
            hls_fragment 3;
            hls_playlist_length 10;

            on_publish http://localhost:3000/validate; # Validate dynamic keys
        }
    }
}
```

---

### **2.4 Start Nginx**

```powershell
cd C:\nginx-rtmp
.\nginx.exe
```

* Test HTTP: [http://localhost:8080](http://localhost:8080)

---

### **2.5 Admin Workflow**

1. Open `admin.html` in browser.
2. Log in as **admin** (Supabase authentication).
3. Click **Create Stream** → generate a dynamic **RTMP key**.
4. Open OBS Studio → Stream Settings:

   * Server: `rtmp://localhost/live`
   * Stream Key: `<dynamic key>`
5. Press **Start Streaming** → HLS files are generated.

---

### **2.6 User Workflow**

* Open in browser:

```
http://localhost:8080/index.html
```

* Live stream will appear automatically.

---

### **2.7 Run Nginx 24/7**

* Use **Task Scheduler** or **NSSM** to run `nginx.exe` as a Windows service.
* Ensure ports **1935** (RTMP) and **8080** (HTTP) are open.

---

## **3. VPS / Linux Setup**

### **3.1 Requirements**

* Ubuntu 22.04 or similar
* SSH access
* Node.js backend for Supabase API (optional)
* Supabase account or DBMS

---

### **3.2 Install Nginx + RTMP**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential libpcre3 libpcre3-dev libssl-dev zlib1g-dev ffmpeg git

cd /usr/local/src
git clone https://github.com/arut/nginx-rtmp-module.git
wget http://nginx.org/download/nginx-1.25.2.tar.gz
tar -xzf nginx-1.25.2.tar.gz
cd nginx-1.25.2

./configure --with-http_ssl_module --add-module=../nginx-rtmp-module
make
sudo make install
```

* Start Nginx:

```bash
sudo /usr/local/nginx/sbin/nginx
```

---

### **3.3 Configure Nginx**

* Same as Windows, change paths for Linux:

```nginx
hls_path /usr/local/nginx/hls;
```

* Optional: `on_publish` → Node.js API validates dynamic RTMP keys.

---

### **3.4 Make Server 24/7**

Create systemd service:

```ini
[Unit]
Description=Nginx RTMP Server
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s quit
Restart=always

[Install]
WantedBy=multi-user.target
```

* Enable and start:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

* Open firewall ports:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 1935/tcp
sudo ufw enable
```

---

### **3.5 Admin & User Workflow**

* Admin creates streams using dashboard → gets RTMP key → OBS streams live.
* Users watch HLS stream:

```
http://<VPS-IP>/index.html
```

---

## **4. Dynamic RTMP Keys**

* Dynamic keys improve security: only authorized admins can stream.
* Generated via Supabase or backend API.
* Validated using `on_publish` in Nginx.

---

## **5. Frontend**

### **Admin Page**

* HTML + Tailwind CSS
* Create/manage streams
* Shows OBS server URL and dynamic RTMP key

### **User Page**

* HTML + Tailwind CSS
* HLS playback using HLS.js
* Automatically shows live stream

---

## **6. OBS Configuration**

* Server: `rtmp://<server-ip>/live`
* Stream Key: `<dynamic key>`
* Press **Start Streaming**

---

## **7. Security Recommendations**

* Use dynamic RTMP keys.
* Use HTTPS for web frontend.
* Restrict admin pages to authorized users only.
* Use firewall rules for ports 1935 and 80/8080.

---

## **8. Summary**

* Works on **Windows and Linux VPS**
* 24/7 streaming
* Admin-controlled dynamic RTMP keys
* Users watch via web page with HLS playback

---

## **Author**

**Sadik Laskar**

* LinkedIn: https://www.linkedin.com/in/unscriptedsadik
* Email: jr.sadiklaskar7@gmail.com

