Below is a **controlled, repeatable lab procedure** to **intentionally recreate the exact failure pattern** you described on **your own NGINX instance**, without guessing and without damaging the host beyond recovery.

This mirrors **502s + high CPU + disk fill due to debug logging + dead upstream**.

---

# 🔥 NGINX FAILURE REPRODUCTION LAB (SAFE & REVERSIBLE)

> **DO NOT run this on a production system.**
> Use a VM, container, or disposable cloud instance.

---

## 0️⃣ Preconditions

* Ubuntu 22.04
* nginx installed
* A fake backend service (we will create one)
* Root or sudo access

```bash
sudo apt update
sudo apt install -y nginx curl
```

---

## 1️⃣ Create a Dummy Backend (Upstream)

Create a simple backend listening on port **8080**.

```bash
cat << 'EOF' | sudo tee /usr/local/bin/backend.sh
#!/bin/bash
while true; do
  echo -e "HTTP/1.1 200 OK\r\nContent-Length: 2\r\n\r\nOK" \
  | nc -l -p 8080 -q 1
done
EOF
```

```bash
sudo chmod +x /usr/local/bin/backend.sh
sudo nohup /usr/local/bin/backend.sh &
```

Verify backend is alive:

```bash
curl http://127.0.0.1:8080
```

---

## 2️⃣ Configure NGINX Reverse Proxy

Create a test site.

```bash
sudo tee /etc/nginx/sites-available/failure-lab << 'EOF'
server {
    listen 80;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_connect_timeout 1s;
        proxy_read_timeout 1s;
    }
}
EOF
```

Enable it:

```bash
sudo ln -s /etc/nginx/sites-available/failure-lab /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

Confirm healthy state:

```bash
curl -I http://localhost
```

You should see **HTTP/1.1 200 OK**.

---

## 3️⃣ 🔥 INTENTIONAL BREAK #1 — Enable Debug Logging

Edit nginx config:

```bash
sudo nano /etc/nginx/nginx.conf
```

Change:

```nginx
error_log /var/log/nginx/error.log debug;
```

Reload (do NOT restart):

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 4️⃣ 🔥 INTENTIONAL BREAK #2 — Kill Backend

```bash
sudo pkill -f backend.sh
```

Confirm backend is down:

```bash
curl http://127.0.0.1:8080
```

Connection refused = good (for the lab).

---

## 5️⃣ Generate Load (Trigger CPU + Logs)

Run this **in another terminal**:

```bash
while true; do curl -s http://localhost >/dev/null; done
```

Within seconds you will observe:

### 🔍 CPU spike

```bash
top
```

NGINX worker will hit high CPU.

### 💾 Disk fill

```bash
sudo du -sh /var/log/nginx
df -h /
```

### 🌐 502 errors

```bash
curl -I http://localhost
```

---

## 6️⃣ Validate Failure Signals (What Interviewers Expect)

```bash
tail -n 20 /var/log/nginx/error.log
```

You should see:

```
connect() failed (111: Connection refused) while connecting to upstream
```

```bash
ss -tulpn | grep nginx
```

NGINX still listening.

```bash
journalctl -u nginx -n 20
```

No crash loop.

✔️ **Service is running but unhealthy**

---

## 7️⃣ 🧯 Recovery Procedure (Fix It Properly)

### Disable debug logging

```bash
sudo sed -i 's/debug;/warn;/' /etc/nginx/nginx.conf
```

### Reload (not restart)

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Truncate logs

```bash
sudo truncate -s 0 /var/log/nginx/error.log
```

### Restart backend

```bash
sudo nohup /usr/local/bin/backend.sh &
```

Verify recovery:

```bash
curl -I http://localhost
top
df -h /
```

---

## 8️⃣ Cleanup (Optional)

```bash
sudo rm /etc/nginx/sites-enabled/failure-lab
sudo rm /etc/nginx/sites-available/failure-lab
sudo systemctl reload nginx
sudo pkill -f backend.sh
```

---

## 🧠 What This Lab Teaches (Why This Is Powerful)

You now **physically experienced**:

| Symptom         | Cause                   |
| --------------- | ----------------------- |
| 502 Bad Gateway | Dead upstream           |
| High CPU        | Debug logging + retries |
| Disk pressure   | Error log explosion     |
| nginx “running” | But unhealthy           |

This is **exactly** what real production incidents look like.

---

## 🎯 Want to Level This Up?

I can next help you:

* Turn this into a **one-command chaos script**
* Reproduce it **inside Docker**
* Add **rate-limiting & circuit-breakers**
* Create a **formal SRE interview runbook**
* Simulate **ephemeral disk exhaustion alerts**

Just tell me how deep you want to go.
