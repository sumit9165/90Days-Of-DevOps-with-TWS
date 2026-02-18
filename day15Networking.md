**very simple, easy-to-understand summaries**

---

# 🌐 Day 15 – Networking Concepts (Simple Version)
<img width="1536" height="1024" alt="NetworkingDNSIPSubnetting" src="https://github.com/user-attachments/assets/a3cdc29c-cea4-416a-84ad-4f97f3c8aca2" />
---

## 1️⃣ DNS – How Names Become IPs

### What happens when you type `google.com`?

1. Your computer asks: “What is the IP address of google.com?”
2. A DNS server looks it up.
3. It finds the IP address (like 142.x.x.x).
4. Your browser connects to that IP to load the website.

👉 DNS = the internet’s phonebook.

---

### DNS Record Types (Simple)

* **A** → Points a name to an IPv4 address
* **AAAA** → Points a name to an IPv6 address
* **CNAME** → One name points to another name
* **MX** → Tells where emails should go
* **NS** → Tells which DNS server is responsible for the domain

---

### What is TTL?

TTL (Time To Live) tells your computer:

> “How long should I remember this IP before asking again?”

---

## 2️⃣ IP Addressing

### What is an IPv4 address?

An IP address is like a home address for a device on a network.

Example:

```
192.168.1.10
```

It has:

* 4 numbers
* Each number goes from 0 to 255

---

### Public vs Private IP

| Type       | Meaning                     | Example      |
| ---------- | --------------------------- | ------------ |
| Public IP  | Used on the internet        | 8.8.8.8      |
| Private IP | Used inside homes/companies | 192.168.1.10 |

Private IPs cannot be accessed directly from the internet.

---

### Private IP Ranges (Memorize These)

* 10.0.0.0 – 10.255.255.255
* 172.16.0.0 – 172.31.255.255
* 192.168.0.0 – 192.168.255.255

If your IP starts with one of these → it’s private.

---

## 3️⃣ CIDR & Subnetting

### What does `/24` mean?

Example:

```
192.168.1.0/24
```

It means:

* 24 bits are for the network
* 8 bits are for devices

In simple terms:
👉 `/24` gives 256 total IP addresses

---

### How Many Devices Can Fit?

| CIDR | Total IPs | Usable Devices |
| ---- | --------- | -------------- |
| /24  | 256       | 254            |
| /16  | 65,536    | 65,534         |
| /28  | 16        | 14             |

(We subtract 2 because one IP is for network and one for broadcast.)

---

### Why Do We Subnet?

Subnetting means:

> Splitting a big network into smaller pieces.

Why?

* Better organization
* Better security
* Less network traffic
* Easier management

Think of it like dividing a big apartment building into smaller floors.

---

## 4️⃣ Ports – The Doors of a Computer

### What is a Port?

If IP address = house address
Then port = door number

One computer can run many services:

* Website
* SSH
* Database
* Redis

Ports help traffic go to the correct service.

---

### Common Ports (Very Important for DevOps)

| Port  | Service                |
| ----- | ---------------------- |
| 22    | SSH (remote login)     |
| 80    | HTTP (website)         |
| 443   | HTTPS (secure website) |
| 53    | DNS                    |
| 3306  | MySQL                  |
| 6379  | Redis                  |
| 27017 | MongoDB                |

These are extremely common in real-world systems.

---

## 5️⃣ Putting Everything Together

### When you run:

```
curl http://myapp.com:8080
```

What happens?

1. DNS finds the IP of `myapp.com`
2. Your system connects to that IP
3. It connects on port **8080**
4. The app must be listening on port 8080

Concepts used:

* DNS
* IP
* Port
* TCP connection

---

### If app can’t reach database at:

```
10.0.1.50:3306
```

Check:

1. Is the IP correct?
2. Is MySQL running?
3. Is port 3306 open?
4. Is firewall blocking it?
5. Are both systems in the same network/subnet?

---

# 🧠 3 Big Takeaways

1. DNS converts names into IP addresses.
2. IP addresses identify devices on networks.
3. Ports allow multiple services to run on the same machine.

---

