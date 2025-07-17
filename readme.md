✅ Overview

    Each backend container:
        Is named app1, app2, ..., app9
        Exposes port 80
        Is addressable by internal DNS via Docker
    NGINX acts as the reverse proxy, using virtual hosts
    Easily extensible (e.g., add app10 later with no big changes)

🧩 Requirements & Design
1. 🔒 Keep HTTPS only at the front edge
Let your existing HTTPS proxy terminate TLS (SSL), and forward plain HTTP to the internal app router.
    💡 You don't need to generate more certs or modify TLS in the app proxy.

2. ⚙️ Forward by subdomain or path

In the HTTPS proxy (existing), you only need to forward traffic for something like:

# inside existing https.conf
location / {
    proxy_pass http://10.0.0.100:8000;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Real-IP $remote_addr;
}
Important:
    Leave server_name, TLS certs, etc. untouched
    Just route to http://<internal-proxy>:<port>

✅ Goal 

To have:
    A functioning HTTPS reverse proxy server (e.g., NGINX or Traefik) at the edge
    You want to place your multi-app HTTP reverse proxy behind it
    You do not want to touch or risk destabilizing the existing HTTPS server config

✅ Architecture

[User]
   ↓ HTTPS (443)
[test.com]
   ↓ (proxy_pass or upstream)
[HTTPS Reverse Proxy (unchanged)]
   ↓ HTTP (e.g., 10.0.0.100:8000)
[Internal Docker NGINX Proxy with 9 apps]
   ↓
[app1, app2, ..., app9]

✅ Why this works
    The edge HTTPS proxy handles all SSL
    The internal reverse proxy is just routing plain HTTP
    You can update the internal proxy without touching the fragile HTTPS config
    Traffic flow:

User (https://test.com) 
→ HTTPS proxy (secure, unchanged) 
→ HTTP internal-proxy:8000 
→ Docker apps

🚀 Deployment Tips
    Bind your internal proxy to a fixed IP or docker network connect it to the HTTPS proxy’s network
    Don't bind the internal proxy to port 80 or 443 to avoid collisions
    Use internal DNS (Docker container names) to link backend services

📁 File Structure //directory view
project-root/
├── docker-compose.yml
├── nginx/
│   ├── site.conf
│   └── proxy-headers.conf
├── app1/
│   ├── Dockerfile
│   └── index.html
├── app2/
│   ├── Dockerfile
│   └── index.html
├── app3/
│   ├── Dockerfile
│   └── index.html


📝 app1/Dockerfile //template for simple Dockerfile

FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html

✅ 1. Build the image from admin/Dockerfile
Run this from the root (where admin/ is):

>>>   docker build -t app1-image ./app1
>>>   docker build -t app2-image ./app2
>>>   docker build -t app3-image ./app3
...

    -t app1-image       — tags the image with a name
    ./app1              — path to the build context (must be a folder, not a file)

✅ 2. Run a container from that image

>>>   docker run -d --name app1-container -p 8081:80 app1-image
>>>   docker run -d --name app2-container -p 8082:80 app2-image
>>>   docker run -d --name app3-container -p 8083:80 app3-image
...

-d                          — detached mode
--name — app1-container     — optional container name
-p 8081:80                  — maps port 8081 on host to port 80 inside the container
app1-image                  — the image name you just built

👉 Then visit:
👉 http://localhost:8081

3. ✅ Internal proxy (your new 9-app router)
This proxy listens on a high port like 8000, and handles routing from Host headers or paths to individual app containers.

//app 1 
✅ 1. nginx/site.conf (template for 9 containers)

server {
    listen 80;
    server_name app1.test.com;
    location / {
        include /etc/nginx/proxy-headers.conf;
        proxy_pass http://app1:80;
    }
}

// app2 

server {
    listen 80;
    server_name app2.test.com;
    location / {
        include /etc/nginx/proxy-headers.conf;
        proxy_pass http://app2:80;
    }
}

//You can also use location /app1/ → proxy_pass http://app1:80;

// add path to map addresses to localhost
>>>   Edit /etc/hosts (Linux/macOS):

127.0.0.1 app1.test.com app2.test.com app3.test.com ... app9.test.com


⚙️ Dynamically Adding Containers

If you add containers dynamically:
    Make sure they:
        Join test-app-bridge
        Are named (e.g., app10)
    Add the server block for app10 to site.conf
    Reload NGINX inside the container:
docker exec nginx-proxy nginx -s reload
