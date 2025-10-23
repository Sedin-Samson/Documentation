# 🧠 NGINX Configuration — Complete Notes

---

## 🧩 1️⃣ Basic Structure

```nginx
events { }

http {
    upstream my_app {
        server app_container:3000;
    }

    server {
        listen 80;
        server_name example.com;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        server_name example.com;

        ssl_certificate     /path/to/cert.pem;
        ssl_certificate_key /path/to/key.pem;

        location / {
            proxy_pass http://my_app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

---

## ⚙️ 2️⃣ Block Breakdown

### 🧱 `events { }`

* Handles **connection-level settings** (like how NGINX handles concurrent connections).
* Usually left empty in basic setups.
* Example:

  ```nginx
  events {
    worker_connections 1024;
  }
  ```

---

### 🌐 `http { }`

* The main block for **HTTP configuration**.
* Contains:
  * `upstream` groups
  * `server` blocks
  * Global HTTP settings (logging, gzip, timeouts, etc.)

---

### 🚀 `upstream`

* Defines one or more **backend servers** (also called an upstream group).
* Used to group servers for load balancing or clean configuration.

Examples:

#### 🐳 Docker-based

```nginx
upstream zapstro_upstream {
  server zapstro:3000;
}
```

(`zapstro` is a Docker container name.)

#### 🖥️ Non-Docker-based

```nginx
upstream app_backend {
  server 127.0.0.1:3000;
  server 127.0.0.1:3001;
}
```

✅ **Purpose:**
* Simplifies proxying
* Enables load balancing
* Keeps config modular

---

### 🧠 `server { }`

Each `server` block = configuration for **one domain or subdomain**.

Examples:
* `zapstro.example.com`
* `tabtobi.example.com`

You can have multiple `server` blocks inside one `http` section.

---

## 🌍 3️⃣ Inside a Server Block

### 🔹 `listen 80;`

* Handles **HTTP** (unencrypted traffic).
* Usually used just to **redirect to HTTPS**.

### 🔹 `listen 443 ssl;`

* Handles **HTTPS (SSL/TLS)** traffic.

---

### 🔹 `server_name`

* Specifies which domain this block should handle.
* Example:

  ```nginx
  server_name zapstro.example.com;
  ```

---

### 🔹 `return 301 https://$host$request_uri;`

* Redirects **HTTP → HTTPS**.
* **301** = Permanent redirect.
* `$host` = Domain from incoming request.
* `$request_uri` = Path + query string.

**Example:**

```
http://zapstro.example.com/getuser?id=1
→ https://zapstro.example.com/getuser?id=1
```

✅ Forces all traffic to use HTTPS (secure).

---

### 🔹 `ssl_certificate` & `ssl_certificate_key`

* Specify SSL cert and private key for HTTPS.
* Example:

  ```nginx
  ssl_certificate     /etc/ssl/cert.pem;
  ssl_certificate_key /etc/ssl/key.pem;
  ```

---

## 🔁 4️⃣ `location /`

* Defines how NGINX handles requests to a particular path.
* `location /` means: "Match **all paths starting from root**."

Used in Docker setups like:

```nginx
location / {
  proxy_pass http://zapstro_upstream;
}
```

### 🧭 Path-based routing (non-Docker)

If one domain hosts multiple apps:

```nginx
location /app1/ {
  proxy_pass http://127.0.0.1:3001/;
}
location /app2/ {
  proxy_pass http://127.0.0.1:4001/;
}
```

---

## 📡 5️⃣ `proxy_set_header` Explained

These headers make sure your backend app gets the correct client information.

| Header | Meaning | Example Value |
|--------|---------|---------------|
| `Host $host` | Sends original domain name to backend | `zapstro.example.com` |
| `X-Real-IP $remote_addr` | Real client IP address | `203.0.113.10` |
| `X-Forwarded-For $proxy_add_x_forwarded_for` | Chain of all proxy IPs (client + proxies) | `203.0.113.10, 172.18.0.2` |
| `X-Forwarded-Proto $scheme` | Original protocol (`http` or `https`) | `https` |

---

### ⚙️ Why we use `X-Forwarded-Proto`

* The browser → NGINX is HTTPS.
* NGINX → backend is often HTTP (inside Docker network).
* This header tells the backend:
  > "Original client used HTTPS."

Used by frameworks (Rails, Django, Node.js, Laravel, etc.) for:
* Correct redirects (`https://`)
* Secure cookies
* CORS / CSRF validation

---

## 🧩 6️⃣ `$host` vs `${ZAPSTRO_DOMAIN}`

| Variable | Defined By | Example | Purpose |
|----------|-----------|---------|---------|
| `$host` | Built-in NGINX variable | `zapstro.example.com` | Automatically set from user request |
| `${ZAPSTRO_DOMAIN}` | Environment/template variable | `zapstro.example.com` | Used in config templates (e.g., Docker ENV) |

✅ `$host` changes dynamically per request.  
✅ `${ZAPSTRO_DOMAIN}` is static (set at startup).

---

## 🔒 7️⃣ Request Flow Summary

| Step | What Happens | Port | Server Block |
|------|--------------|------|--------------|
| 1 | User hits `http://zapstro.example.com` | 80 | HTTP block redirects |
| 2 | NGINX sends back `301 https://zapstro.example.com/...` | — | Redirect response |
| 3 | Browser requests `https://zapstro.example.com` | 443 | HTTPS server block |
| 4 | NGINX proxies internally to backend | — | Uses `proxy_pass http://zapstro_upstream;` |

---

## ✅ 8️⃣ Key Takeaways

| Concept | Summary |
|---------|---------|
| `events {}` | Handles connection limits (low-level, rarely touched). |
| `http {}` | Main block for HTTP configuration. |
| `upstream` | Defines backend servers (used in Docker and non-Docker). |
| `server` | Defines how a domain/subdomain is handled. |
| `listen 80 → return 301` | Redirects HTTP → HTTPS. |
| `listen 443 ssl` | Handles secure HTTPS traffic. |
| `location /` | Routes all paths (root). |
| `proxy_set_header` | Passes correct client & request info to backend. |
| `$host` | Auto-filled domain from user request. |
| `${DOMAIN}` | Environment variable, static. |

---

## 📚 Additional Resources

* [Official NGINX Documentation](https://nginx.org/en/docs/)
* [NGINX Reverse Proxy Guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
* [SSL/TLS Configuration Best Practices](https://ssl-config.mozilla.org/)

---

**Created:** DevOps Configuration Reference  
**Version:** 1.0  
**Last Updated:** October 2025