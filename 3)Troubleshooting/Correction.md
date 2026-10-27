
## Masla 1: Port 80 par Docker nginx ka kabza

**Kya hua:** Browser mein `192.168.0.18` kholne par hamari website ke bajaye "My Load balancing application" wala page aa raha tha (kabhi app1, kabhi app2).

**Wajah:** Server par pehle se ek Docker `nginx` container chal raha tha jo port 80 par load balancer bana hua tha. `ss -tlnp` mein `docker-proxy` port 80 par dikha, aur `docker ps` mein `nginx`, `app1`, `app2` nazar aaye. Hamari config bhi `listen 80;` par thi, lekin port 80 pehle se Docker ke paas tha.

**Solution:** `docker stop nginx` se us container ko band kiya.

## Masla 2: Hamari config load hi nahi ho rahi thi

**Kya hua:** Docker nginx band karne aur restart ke baad bhi port 80 par koi nginx nahi aaya. Sirf 8081 (nginx) aur 8080 (java) nazar aa rahe thay.

**Wajah:** `/etc/nginx/nginx.conf` mein yeh line maujood hi nahi thi:
```nginx
include /etc/nginx/conf.d/*.conf;
```
Is liye `conf.d/mywebsite.conf` ko nginx ne kabhi padha hi nahi. Yeh humne `nginx -T | grep -E "listen|server_name|include"` se pakra, jisme sirf `include mime.types;` nazar aaya.

**Solution:** `nginx.conf` ke `http { }` block mein woh `include` line add ki, phir `nginx -t` aur `systemctl restart nginx`. Uske baad port 80 par host nginx nazar aaya.

## Masla 3: Reboot ke baad Docker nginx wapas aa gaya

**Kya hua:** System restart ke baad website phir nahi chali.

**Wajah:** Docker container ki restart policy shayad `always` thi, to reboot par woh khud start ho kar port 80 wapas le leta.

**Solution:**
```bash
docker stop nginx
docker update --restart=no nginx
systemctl enable nginx
systemctl restart nginx
```
Ab Docker nginx khud start nahi hota, aur host nginx reboot par khud start hota hai.

## Masla 4: Firewall (ufw) mein port 80 band tha

**Kya hua:** Server par `curl localhost` kaam kar raha tha, lekin browser se site nahi khul rahi thi.

**Wajah:** `ufw status` mein sirf 22, 3000 aur 8080 allowed thay, 80 nahi.

**Solution:**
```bash
ufw allow 80/tcp
```

## Short summary

| Masla | Wajah | Hal |
|---|---|---|
| Load balancer page aa raha tha | Docker nginx ne port 80 le rakha tha | `docker stop nginx` |
| Config load nahi hui | `nginx.conf` mein `include conf.d/*.conf` nahi tha | Line add ki |
| Reboot ke baad wapas masla | Docker nginx auto-start | `docker update --restart=no nginx` |
| Browser se site nahi khuli | ufw ne port 80 block kiya tha | `ufw allow 80/tcp` |

## Yaad rakhne wali baat

Ek server par ek port (jaise 80) par ek hi service chal sakti hai. Agar dobara load balancer chalana ho, to ya usay doosre port par chalayein, ya hamari website ko doosre port par shift karein.

Hum ne total **4 files** par kaam kiya. Do naye files banayi, do purani files mein tabdeeli ki.

## 1. `/var/www/mywebsite/index.html` (nayi file)

**Kya hai:** Website ka asal page (Home, Services, About, Contact).

**Kaun si jagah:** Website ka folder `/var/www/mywebsite/`

**Kaam:** Browser jab `192.168.0.18` kholta hai to Nginx yahi file dikhata hai.

## 2. `/var/www/mywebsite/style.css` (nayi file)

**Kya hai:** Website ka design (rang, layout, buttons, cards).

**Kaun si jagah:** Wahi folder `/var/www/mywebsite/`

**Kaam:** `index.html` mein yeh line hai: `<link rel="stylesheet" href="style.css">`. Isi se browser design load karta hai. Isko `.conf` mein likhne ki zaroorat nahi thi.

## 3. `/etc/nginx/conf.d/mywebsite.conf` (nayi file)

**Kya hai:** Nginx ki site config.

**Kaun si jagah:** `/etc/nginx/conf.d/`

**Isme kya likha:**
```nginx
server {
    listen 80;
    server_name 192.168.0.18;

    root /var/www/mywebsite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

**Kaam:** Nginx ko batati hai ke port 80 par aane wali request ko `/var/www/mywebsite/` folder se serve karna hai.

## 4. `/etc/nginx/nginx.conf` (purani file, tabdeeli ki)

**Kya hai:** Nginx ki main config file.

**Humne kya badla:** `http { }` block ke andar yeh line add ki:
```nginx
include /etc/nginx/conf.d/*.conf;
```

**Kaam:** Is line ke baghair Nginx `conf.d` folder ki files (yaani `mywebsite.conf`) padhta hi nahi tha.

## Commands se jo settings badli (files nahi, lekin zaroori)

| Command | Kaam |
|---|---|
| `chown` / `chmod` on `/var/www/mywebsite` | Nginx ko files padhne ki permission |
| `docker stop nginx` | Docker wale nginx ko band kiya |
| `docker update --restart=no nginx` | Reboot par Docker nginx khud start na ho |
| `systemctl enable nginx` | Reboot par host nginx khud start ho |
| `ufw allow 80/tcp` | Firewall mein port 80 khola |

## Folder structure ek nazar mein

```
/var/www/mywebsite/
├── index.html      ← website ka content
└── style.css       ← website ka design

/etc/nginx/
├── nginx.conf      ← main config (include line add ki)
└── conf.d/
    └── mywebsite.conf   ← hamari site ki config
```

Yeh sab files ka rishta aise hai: browser → port 80 → `nginx.conf` → `conf.d/mywebsite.conf` → `/var/www/mywebsite/index.html` → `style.css`.
