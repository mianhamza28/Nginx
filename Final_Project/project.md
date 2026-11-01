# Nginx Load Balancer: Complete Guide (Ubuntu + RHEL)

Is guide mein do servers par ek static website host karke Nginx ke zariye **round robin load balancing**, **30 din ki caching** aur **rate limiting** setup ki gayi hai. Is mein woh saare masail aur hal bhi shamil hain jo asal setup ke dauran saamne aaye.

## Fehrist (Contents)

1. [Setup ka khaka](#1-setup-ka-khaka)
2. [Port ka intikhab (Jenkins wala masla)](#2-port-ka-intikhab)
3. [Step 1: Ubuntu par basic load balancer](#3-step-1-ubuntu-par-basic-load-balancer)
4. [Step 2: RHEL par Nginx aur SSH](#4-step-2-rhel-par-nginx-aur-ssh)
5. [Step 3: Website files RHEL par bhejna](#5-step-3-website-files-rhel-par-bhejna)
6. [Step 4: Terminal se test](#6-step-4-terminal-se-test)
7. [Step 5: Windows Firefox se test](#7-step-5-windows-firefox-se-test)
8. [Step 6: Caching aur Rate Limiting](#8-step-6-caching-aur-rate-limiting)
9. [Cache busting (`?v=2`)](#9-cache-busting)
10. [Website update karne ka tareeqa](#10-website-update-karne-ka-tareeqa)
11. [`nginx -t` aur reload kab chahiye?](#11-nginx--t-aur-reload-kab-chahiye)
12. [Capacity / Load testing](#12-capacity--load-testing)
13. [Failover test](#13-failover-test)
14. [Troubleshooting](#14-troubleshooting)
15. [Production se pehle safai](#15-production-se-pehle-safai)
16. [Aur options](#16-aur-options)
17. [Quick reference](#17-quick-reference)

---

## 1. Setup ka khaka

| Machine | OS | IP | Kaam |
|---|---|---|---|
| Server 1 | Ubuntu | `192.168.0.18` | Load balancer + website (internal port 8090) |
| Server 2 | RHEL | `192.168.0.13` | Website (port 80) |
| Client | Windows | Same network | Firefox se test |

```
                    +----------------------------+
 Firefox  --->      |  Ubuntu 192.168.0.18:80    |
 (Windows)          |  Nginx Load Balancer       |
                    |  + Cache + Rate Limit      |
                    +-------------+--------------+
                                  |  round robin
                  +---------------+----------------+
                  |                                |
      +-----------v----------+        +------------v-----------+
      | Ubuntu 127.0.0.1:8090|        | RHEL 192.168.0.13:80   |
      | /var/www/mywebsite   |        | /usr/share/nginx/html  |
      +----------------------+        +------------------------+
```

Request 1 Ubuntu ko, request 2 RHEL ko, request 3 phir Ubuntu ko, aur isi tarah chalta rehta hai.

**Kaun si command kahan chalani hai:** Har section mein saaf likha hai ke command **Ubuntu** par chalani hai ya **RHEL** par. Ghalat machine par chalane se errors aate hain (neeche troubleshooting dekhein).

---

## 2. Port ka intikhab

Ubuntu par apni website ke liye internal port **`8090`** use kiya gaya hai. Doosre ports kyun nahi liye:

| Port | Wajah |
|---|---|
| `8080` | Is par **Jenkins** chal raha hai. `curl http://127.0.0.1:8080` par website ki jagah Jenkins ka login page aaya |
| `8081` | `/etc/nginx/nginx.conf` ki line 36 mein pehle se `listen 8081;` hai |
| `8090` | Free tha, isliye yehi use kiya |

Naya port lene se pehle hamesha check karein:

```bash
sudo ss -tlnp | grep 8090
```

Kuch output na aaye to port free hai. Kisi port ka source dhoondna ho:

```bash
sudo grep -rn "8081" /etc/nginx/
```

Jenkins ka port nahi badalna, usay jaisa hai waisa chalne dein.

---

## 3. Step 1: Ubuntu par basic load balancer

**Yeh sab Ubuntu par.**

### 3.1 Nginx install (agar nahi hai)

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable --now nginx
```

### 3.2 Config file banayein

```bash
sudo nano /etc/nginx/conf.d/mywebsite.conf
```

Pehle basic version likhein (caching/rate limit baad mein Step 6 mein lagayenge, taake pehle load balancing alag se test ho jaye):

```nginx
upstream backend_servers {
    server 192.168.0.13:80;      # RHEL
    server 127.0.0.1:8090;       # Ubuntu ki apni website
}

# Ubuntu ki website (sirf internal, bahar se nahi dikhegi)
server {
    listen 127.0.0.1:8090;
    root /var/www/mywebsite;
    index index.html;

    add_header X-Served-By "Ubuntu";    # sirf test ke liye

    location / {
        try_files $uri $uri/ =404;
    }
}

# Load balancer
server {
    listen 80;
    server_name 192.168.0.18;

    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Save: `Ctrl+O`, `Enter`, phir `Ctrl+X`.

### 3.3 Check aur reload

```bash
sudo nginx -t
sudo systemctl reload nginx
```

`nginx -t` mein "syntax is ok" aur "test is successful" aana chahiye.

> **Note:** Agar Ubuntu par default site (`sites-enabled/default`) port 80 par chal rahi ho to woh load balancer se takra sakti hai. Zaroorat ho to: `sudo rm /etc/nginx/sites-enabled/default`

---

## 4. Step 2: RHEL par Nginx aur SSH

**Yeh saari commands RHEL machine par.**

```bash
sudo dnf install nginx openssh-server -y
sudo systemctl enable --now nginx sshd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

`restorecon` "command not found" de to:

```bash
sudo dnf install policycoreutils -y
```

---

## 5. Step 3: Website files RHEL par bhejna

### 5.1 Ubuntu se files bhejein

**Yeh command Ubuntu par.** `RHEL_USERNAME` ki jagah RHEL ka asal username likhein:

```bash
scp -r /var/www/mywebsite/* RHEL_USERNAME@192.168.0.13:/tmp/
```

Files `/tmp/` mein isliye bheji jati hain kyunke normal user ko seedha `/usr/share/nginx/html/` mein likhne ki ijazat nahi hoti.

| Hissa | Matlab |
|---|---|
| `-r` | Folders bhi copy karo |
| `RHEL_USERNAME` | RHEL ka apna username |
| `192.168.0.13` | RHEL ka IP |

### 5.2 RHEL par files Nginx folder mein rakhein

**Yeh commands RHEL par** (Ubuntu par nahi):

```bash
sudo cp -r /tmp/index.html /tmp/style.css /usr/share/nginx/html/
sudo restorecon -Rv /usr/share/nginx/html/
```

Agar website mein images, js ya doosre folders bhi hain to pehle `ls /tmp/` dekh lein, phir:

```bash
sudo cp -r /tmp/* /usr/share/nginx/html/
sudo restorecon -Rv /usr/share/nginx/html/
```

`restorecon` SELinux ka context theek karta hai, warna RHEL par **403 Forbidden** aa sakta hai.

### 5.3 Ubuntu se ek hi command mein (optional)

```bash
scp -r /var/www/mywebsite/* RHEL_USERNAME@192.168.0.13:/tmp/ && \
ssh RHEL_USERNAME@192.168.0.13 "sudo cp -r /tmp/index.html /tmp/style.css /usr/share/nginx/html/ && sudo restorecon -Rv /usr/share/nginx/html/"
```

Dono jagah **same IP** (`192.168.0.13`) hona zaroori hai.

---

## 6. Step 4: Terminal se test

**Teeno commands Ubuntu par:**

```bash
curl http://192.168.0.13                 # RHEL seedha
curl http://127.0.0.1:8090               # Ubuntu ki internal website
curl -s http://192.168.0.18 | head -5    # Load balancer ke zariye
```

| Command | Kya check karti hai |
|---|---|
| `curl http://192.168.0.13` | RHEL ki website theek chal rahi hai |
| `curl http://127.0.0.1:8090` | Ubuntu ki internal website theek hai (Jenkins ka page nahi aana chahiye) |
| `curl -s http://192.168.0.18 \| head -5` | Load balancer kaam kar raha hai |

Pehli do commands fail hon to load balancer bhi sahi kaam nahi karega, isliye pehle unhein theek karein.

### Round robin ka saboot

```bash
for i in 1 2 3 4 5 6; do curl -sI http://192.168.0.18 | grep -i x-served-by || echo "RHEL"; done
```

Asal test mein yeh output aaya (RHEL aur Ubuntu bari bari):

```
RHEL
X-Served-By: Ubuntu
RHEL
X-Served-By: Ubuntu
RHEL
X-Served-By: Ubuntu
```

`X-Served-By: Ubuntu` aaye to Ubuntu ne jawab diya, header na aaye to RHEL ne.

> **Cache lagne ke baad:** Step 6 ke baad HTML cache mein aa jata hai, to yeh bari bari wala test same nahi dikhega. Wahan neeche tarika diya hai.

---

## 7. Step 5: Windows Firefox se test

Windows machine Ubuntu ke same network par honi chahiye. Firefox mein:

```
http://192.168.0.18
```

Dono servers par same website hai, is liye farq nazar nahi aayega. Do tareeqe hain:

### Tareeqa A: Developer Tools

1. `F12` dabayein, **Network** tab kholein.
2. `Ctrl+Shift+R` se hard refresh karein (cache se na aaye).
3. Sab se upar wali request (`192.168.0.18`) par click karein.
4. **Headers > Response Headers** dekhein.
5. `X-Served-By: Ubuntu` hai to Ubuntu ne jawab diya, nahi hai to RHEL ne.

5-6 baar refresh karein, header kabhi aayega kabhi nahi.

### Tareeqa B: Footer mein fark likhna

Test ke liye dono servers ki `index.html` ke footer mein farq daalein. Footer file ke **bilkul neeche** hota hai (head wala hissa upar hota hai).

**RHEL par** (file: `/usr/share/nginx/html/index.html`):

```bash
sudo sed -i 's|All rights reserved.|(RHEL)|' /usr/share/nginx/html/index.html
grep -n "RHEL" /usr/share/nginx/html/index.html
```

**Ubuntu par** (file: `/var/www/mywebsite/index.html`):

```bash
sudo sed -i 's|All rights reserved.|(Ubuntu)|' /var/www/mywebsite/index.html
grep -n "Ubuntu" /var/www/mywebsite/index.html
```

Haath se badalna ho to nano mein:

```bash
sudo nano /usr/share/nginx/html/index.html
```

Phir `Ctrl+W`, `All rights` likhein, `Enter`. Cursor seedha footer wali line par pohonch jayega. File ke aakhir par jaane ke liye `Alt+/` bhi chalta hai.

Footer wali line:

```html
<p>&copy; 2026 MyBrand. All rights reserved.</p>
```

badal kar yeh karein:

```html
<p>&copy; 2026 MyBrand. (RHEL)</p>
```

Phir Firefox mein `Ctrl+Shift+R` karte jayein, footer mein bari bari **(Ubuntu)** aur **(RHEL)** nazar aayega. Sirf HTML badli hai, to **`nginx -t` ya reload ki zaroorat nahi**.

**Test ke baad wapas theek karein:**

```bash
# RHEL par
sudo sed -i 's|(RHEL)|All rights reserved.|' /usr/share/nginx/html/index.html

# Ubuntu par
sudo sed -i 's|(Ubuntu)|All rights reserved.|' /var/www/mywebsite/index.html
```

### Agar page na khule

Ubuntu ka firewall port 80 rok raha ho sakta hai:

```bash
sudo ufw allow 80/tcp
```

---

## 8. Step 6: Caching aur Rate Limiting

Yeh sab **Ubuntu par** `/etc/nginx/conf.d/mywebsite.conf` mein hota hai, kyunke load balancer wahi hai. **RHEL par kuch badalna nahi.**

### 8.1 Pehle backup

```bash
sudo cp /etc/nginx/conf.d/mywebsite.conf /etc/nginx/conf.d/mywebsite.conf.bak
```

`.bak` naam ki wajah se nginx usay load nahi karega.

### 8.2 Final config (pura content replace karein)

```bash
sudo nano /etc/nginx/conf.d/mywebsite.conf
```

```nginx
# ---------- Rate limit aur cache zones ----------
limit_req_zone  $binary_remote_addr zone=perip_req:10m  rate=10r/s;
limit_conn_zone $binary_remote_addr zone=perip_conn:10m;

proxy_cache_path /var/cache/nginx/mysite
                 levels=1:2
                 keys_zone=mysite_cache:10m
                 max_size=200m
                 inactive=30d
                 use_temp_path=off;

# ---------- Backends ----------
upstream backend_servers {
    server 192.168.0.13:80;      # RHEL
    server 127.0.0.1:8090;       # Ubuntu
}

# Ubuntu ki website (internal)
server {
    listen 127.0.0.1:8090;
    root /var/www/mywebsite;
    index index.html;

    add_header X-Served-By "Ubuntu";

    location / {
        try_files $uri $uri/ =404;
    }
}

# ---------- Load balancer ----------
server {
    listen 80;
    server_name 192.168.0.18;

    # Rate limiting
    limit_req    zone=perip_req  burst=30 nodelay;
    limit_req_status 429;
    limit_conn   perip_conn 30;
    limit_conn_status 429;

    # Proxy settings
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # Server-side cache (load balancer par)
    proxy_cache mysite_cache;
    proxy_cache_valid 200 301 302 10m;
    proxy_cache_valid 404 1m;
    proxy_cache_lock on;
    proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    proxy_cache_bypass $http_pragma;       # Ctrl+Shift+R se cache bypass (test ke liye)
    add_header X-Cache-Status $upstream_cache_status always;

    # Static files: 30 din ki caching
    location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|webp|woff2?|ttf)$ {
        proxy_pass http://backend_servers;
        proxy_cache_valid 200 30d;
        expires 30d;
    }

    # HTML pages
    location / {
        proxy_pass http://backend_servers;
        expires 1h;
    }
}
```

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 8.3 Har hissa kya karta hai

**Browser caching (`expires 30d`)**
Browser CSS, JS aur images 30 din tak apne paas rakhta hai aur dobara server se nahi mangta. Response mein `Cache-Control: max-age=2592000` jata hai (2,592,000 seconds = 30 din).

**Server cache (`proxy_cache`)**
Load balancer backend se aaya jawab apne paas save kar leta hai. Agli request backend tak jati hi nahi. `proxy_cache_use_stale` ki wajah se backend gir jaye to bhi purana cached page mil jata hai.

| Setting | Matlab |
|---|---|
| `proxy_cache_path ... max_size=200m` | Cache ka zyada se zyada size 200 MB |
| `inactive=30d` | 30 din tak kisi ne maanga nahi to cache se nikal do |
| `proxy_cache_valid 200 30d` (static) | Static files 30 din tak valid |
| `proxy_cache_valid 200 301 302 10m` (baaqi) | HTML waghera 10 minute tak valid |
| `proxy_cache_lock on` | Ek hi file ki kai requests aayein to backend ko sirf ek jaati hai |
| `proxy_cache_bypass $http_pragma` | `Ctrl+Shift+R` cache ko bypass kar deta hai |

**Rate limiting**

| Setting | Matlab |
|---|---|
| `rate=10r/s` | Ek IP se 10 requests per second |
| `burst=30` | Achanak 30 extra requests bhi guzar jayengi (page load mein HTML + CSS + JS + images mil kar kai requests banti hain) |
| `nodelay` | Burst wali requests ko intezar nahi karwata |
| `limit_conn perip_conn 30` | Ek IP se ek waqt mein 30 se zyada connections nahi |
| `limit_req_status 429` | Limit cross hone par `429 Too Many Requests` milta hai |

### 8.4 HTML par sirf 1 ghanta kyun?

HTML par `expires 1h` rakha hai, 30 din nahi. Agar HTML par bhi 30 din lagayein to `index.html` badalne par users ko 30 din tak purana page milega, kyunke browser server se poochega hi nahi. Phir bhi chahein to `location /` mein `expires 1h;` ki jagah `expires 30d;` likh dein.

### 8.5 Caching test (asal result)

**Ubuntu par**, command do baar chalayein:

```bash
curl -sI http://192.168.0.18/style.css | grep -iE "cache-control|expires|x-cache-status"
curl -sI http://192.168.0.18/style.css | grep -iE "cache-control|expires|x-cache-status"
```

Asal output:

```
Expires: Tue, 01 Dec 2026 14:04:22 GMT
Cache-Control: max-age=2592000
X-Cache-Status: MISS

Expires: Tue, 01 Dec 2026 14:04:22 GMT
Cache-Control: max-age=2592000
X-Cache-Status: HIT
```

| Field | Matlab |
|---|---|
| `MISS` (pehli request) | Cache mein file nahi thi, backend se mangwai aur save kar li |
| `HIT` (doosri request) | Ab file load balancer ke apne cache se aayi, backend tak gayi hi nahi |
| `max-age=2592000` | 30 din |
| `Expires` | Aaj se theek 30 din baad ki date |

### 8.6 Rate limit test (asal result)

```bash
for i in $(seq 1 60); do curl -s -o /dev/null -w "%{http_code}\n" http://192.168.0.18/; done | sort | uniq -c
```

Asal output:

```
     37 200
     23 429
```

60 requests mein se 37 guzar gayin aur 23 ko `429 Too Many Requests` mila. `burst=30` ki wajah se pehli kuch requests foran guzar jati hain, baqi limit se bahar hone par block ho jati hain.

Agar `429` na aaye to zyada requests try karein:

```bash
for i in $(seq 1 200); do curl -s -o /dev/null -w "%{http_code}\n" http://192.168.0.18/; done | sort | uniq -c
```

### 8.7 Limit kam ya zyada karna

```nginx
limit_req_zone $binary_remote_addr zone=perip_req:10m rate=10r/s;
...
limit_req zone=perip_req burst=30 nodelay;
```

- **Zyada sakht:** `rate=5r/s` aur `burst=10`
- **Narm:** `rate=20r/s` aur `burst=50`

Badalne ke baad `sudo nginx -t` aur `sudo systemctl reload nginx`.

### 8.8 Cache lagne ke baad ki ehtiyatein

1. **Round robin test ab mushkil hoga.** HTML 10 minute cache mein rehta hai, to sab jawab `HIT` honge aur bari bari Ubuntu/RHEL nazar nahi aayega. Test ke liye `Ctrl+Shift+R` use karein (cache bypass), ya test ke dauran `proxy_cache mysite_cache;` wali line comment kar dein.
2. **Cache mein purana `X-Served-By` header** bhi save ho jata hai. HIT par bhi wohi server ka naam dikhta hai jis se pehli baar file aayi thi.
3. **Rate limit apke apne IP par bhi lagti hai.** Firefox se bhi kabhi `429` mile to kuch second ruk kar dobara try karein.
4. **Ek hi router ke peeche kai log (NAT)** ek hi IP se aate hain, wahan limit jaldi lag sakti hai. Zaroorat ho to `rate` ya `burst` barhayein.

---

## 9. Cache busting

30 din ki caching ka nuqsan: CSS ya JS badalne par browser purani file hi use karta rahega. Hal: file ke naam ke saath version laga dein.

```html
<link rel="stylesheet" href="style.css?v=2">
```

### Terminal se ek command mein badalna

**Ubuntu par:**

```bash
sudo sed -i 's|href="style.css[^"]*"|href="style.css?v=2"|' /var/www/mywebsite/index.html
```

**RHEL par:**

```bash
sudo sed -i 's|href="style.css[^"]*"|href="style.css?v=2"|' /usr/share/nginx/html/index.html
```

Yeh `style.css` ya `style.css?v=1`, jo bhi likha ho, usay `style.css?v=2` bana deta hai. Dobara barhana ho to `v=2` ko `v=3` likh dein.

**Check:**

```bash
grep -n "style.css" /var/www/mywebsite/index.html
```

**Ubuntu se dono servers ek saath (ssh se):**

```bash
sudo sed -i 's|href="style.css[^"]*"|href="style.css?v=3"|' /var/www/mywebsite/index.html && \
ssh RHEL_USERNAME@192.168.0.13 "sudo sed -i 's|href=\"style.css[^\"]*\"|href=\"style.css?v=3\"|' /usr/share/nginx/html/index.html"
```

**Phir load balancer ka cache saaf karein (Ubuntu par):**

```bash
sudo rm -rf /var/cache/nginx/mysite/*
```

> `?v=` sirf tab kaam aata hai jab aap ne `style.css` ki **apni content** bhi badli ho. Version number sirf browser ko batata hai ke nayi file mangwao. Aur `style.css` **dono servers** par update honi chahiye.

---

## 10. Website update karne ka tareeqa

Jab bhi website mein tabdeeli karein, **dono servers par files update karni hongi**, warna users ko kabhi purana kabhi naya page milega.

1. Ubuntu par `/var/www/mywebsite/` mein file badlein.
2. RHEL par bhejein:

```bash
scp -r /var/www/mywebsite/* RHEL_USERNAME@192.168.0.13:/tmp/ && \
ssh RHEL_USERNAME@192.168.0.13 "sudo cp -r /tmp/index.html /tmp/style.css /usr/share/nginx/html/ && sudo restorecon -Rv /usr/share/nginx/html/"
```

3. Agar CSS/JS badli hai to `?v=` badlein (Section 9).
4. Load balancer ka cache saaf karein:

```bash
sudo rm -rf /var/cache/nginx/mysite/*
sudo systemctl reload nginx
```

---

## 11. `nginx -t` aur reload kab chahiye?

| Kya badla | `nginx -t` + reload chahiye? |
|---|---|
| Sirf HTML/CSS/images (website files) | **Nahi** |
| Footer ka text badla | **Nahi** |
| Nginx config (`nginx.conf`, `conf.d/*.conf`) | **Haan** |
| RHEL par sirf files copy kin | **Nahi** |

Nginx website files har request par disk se padhta hai, is liye nayi files foran nazar aati hain. Config badli ho to:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 12. Capacity / Load testing

Maqsad: dekhna ke server kitni requests per second handle karta hai aur kab slow ya fail hota hai.

### 12.1 Tool install

```bash
# Ubuntu
sudo apt install apache2-utils -y

# RHEL
sudo dnf install httpd-tools -y
```

Test **kisi teesri machine** (Windows ya koi aur Linux) se chalana behtar hai. Load balancer par hi chalayenge to tool khud CPU kha jayega.

### 12.2 Test chalayein

```bash
ab -n 10000 -c 100 http://192.168.0.18/
```

- `-n 10000` = kul 10,000 requests
- `-c 100` = ek waqt mein 100 users
- URL ke aakhir mein `/` zaroor likhein

### 12.3 Nateeja padhna

| Field | Matlab |
|---|---|
| `Requests per second` | Asal capacity |
| `Time per request` | Ek request mein kitne ms laga |
| `Failed requests` | **0 hona chahiye** |
| `Non-2xx responses` | Errors (429, 502, 503) |
| `Percentage served within a certain time` | 95% / 99% requests kitne ms mein mukammal hui |

### 12.4 Aahista aahista load barhayein

```bash
ab -n 5000  -c 10   http://192.168.0.18/
ab -n 10000 -c 50   http://192.168.0.18/
ab -n 20000 -c 100  http://192.168.0.18/
ab -n 30000 -c 300  http://192.168.0.18/
ab -n 50000 -c 500  http://192.168.0.18/
```

Jis point par `Failed requests` aane lagein ya `Time per request` bohat barh jaye, wahi aap ke setup ki taqriban limit hai.

### 12.5 Test ke dauran monitoring

Dono servers par:

```bash
top        # ya htop
```

Ubuntu par:

```bash
sudo tail -f /var/log/nginx/error.log
```

`worker_connections are not enough` ya `Too many open files` aaye to `/etc/nginx/nginx.conf` mein barhayein (phir `nginx -t` aur reload):

```nginx
worker_processes auto;

events {
    worker_connections 4096;
}
```

### 12.6 `wrk` (zyada tez tool)

```bash
sudo apt install wrk -y
wrk -t4 -c200 -d30s http://192.168.0.18/
```

### 12.7 Bohat zaroori: rate limit pehle band karein

Rate limiting lagi ho to load test mein `429` aayenge aur server ki asal capacity pata nahi chalegi. Test se pehle config mein yeh lines **comment** kar dein:

```nginx
    # limit_req    zone=perip_req  burst=30 nodelay;
    # limit_req_status 429;
    # limit_conn   perip_conn 30;
    # limit_conn_status 429;
```

`sudo nginx -t && sudo systemctl reload nginx`. Test ke baad **wapas on kar dein**.

### 12.8 Baaqi baatein

1. **Sirf apne servers par test karein.** Kisi aur ki website par load test karna ghair qanooni hai.
2. Static website par numbers bohat ache aayenge. Asal dynamic app (database, PHP) par nateeja kam hoga.
3. Network (Wi-Fi/VM) slow ho to asal capacity nazar nahi aati.
4. Aap ke servers VMs lagte hain (hostname `osboxes`), nateeja VM ke CPU cores aur RAM par depend karega.
5. Round robin ka asar dekhna ho to test ke dauran dono servers par `top` kholein. Dono ka CPU taqriban barabar barhna chahiye.

---

## 13. Failover test

Dekhein ke ek server band ho to kya hota hai.

1. **RHEL par:** `sudo systemctl stop nginx`
2. **Ubuntu par** 6 baar `curl -sI http://192.168.0.18` chalayein. Sab jawab Ubuntu se aane chahiye, website chalti rahegi.
3. Test ke baad **RHEL par:** `sudo systemctl start nginx`

Nginx kisi backend ke fail hone par usay kuch der ke liye skip kar deta hai (`max_fails` aur `fail_timeout` ke hisaab se, Section 16 dekhein).

Cache ki wajah se `proxy_cache_use_stale` bhi kaam karta hai: agar dono backend gir jayein to bhi cached pages mil jate hain.

---

## 14. Troubleshooting

| Masla | Wajah | Hal |
|---|---|---|
| `cp: target '/usr/share/nginx/html/' is not a directory` | Command Ubuntu par chali, RHEL par nahi | Yeh command **RHEL** par chalayein |
| `sudo: restorecon: command not found` | Ubuntu par SELinux nahi hota, ya RHEL par package nahi | RHEL par chalayein, ya `sudo dnf install policycoreutils` |
| `curl http://127.0.0.1:8080` par Jenkins ka login page | Port 8080 par Jenkins chal raha hai | Alag port (`8090`) use karein |
| Port 8081 par `ss` mein nginx dikhe | `nginx.conf` line 36 mein `listen 8081;` | `sudo grep -rn "8081" /etc/nginx/`, ya doosra port chunein |
| RHEL par **403 Forbidden** | SELinux context ghalat | `sudo restorecon -Rv /usr/share/nginx/html/` |
| RHEL par page nahi khulta | Firewall port 80 band | `sudo firewall-cmd --permanent --add-service=http && sudo firewall-cmd --reload` |
| `502 Bad Gateway` | Koi backend band hai ya port ghalat | Dono backend ko alag alag `curl` karke check karein |
| Kabhi purana kabhi naya page | Dono servers par files same nahi | Dono par files update karein |
| Windows se page nahi khulta | Ubuntu firewall | `sudo ufw allow 80/tcp` |
| `scp` mein `Connection refused` | RHEL par `sshd` band | `sudo systemctl enable --now sshd` |
| Footer badla par Firefox mein purana dikha | Cache (browser ya load balancer) | `Ctrl+Shift+R`, aur `sudo rm -rf /var/cache/nginx/mysite/*` |
| CSS badli par browser mein purani nazar aati hai | 30 din ki browser cache | `style.css?v=` ka number barhayein (Section 9) |
| Sab requests `429` de rahi hain | Rate limit bohat sakht, ya NAT ke peeche kai log | `rate` / `burst` barhayein |
| `ab` test mein bohat `Non-2xx responses` | Rate limit load test ko rok rahi hai | Test ke dauran `limit_req` comment karein |
| `nginx -t` mein `proxy_cache_path` error | Cache folder ki permission ya ghalat syntax | Config dobara check karein, `sudo mkdir -p /var/cache/nginx` |
| Round robin nazar nahi aata (sab `HIT`) | HTML cache ho gayi | `Ctrl+Shift+R` use karein ya test ke liye cache comment karein |

### Kaam ki debugging commands

```bash
sudo ss -tlnp | grep nginx                      # nginx kin ports par sun raha hai
sudo grep -rn "listen" /etc/nginx/              # saari listen lines
sudo tail -f /var/log/nginx/error.log           # live error log
sudo tail -f /var/log/nginx/access.log          # live access log
sudo nginx -T | less                            # poori active config
ls /var/cache/nginx/mysite/                     # cache mein kya save hai
```

---

## 15. Production se pehle safai

- `add_header X-Served-By "Ubuntu";` sirf test ke liye tha. Chahein to delete kar ke `sudo systemctl reload nginx` kar dein.
- Footer mein jo `(Ubuntu)` / `(RHEL)` likha tha, wapas `All rights reserved.` kar dein.
- `add_header X-Cache-Status ...` bhi chahein to hata dein (debugging ke liye rakhna bhi theek hai).
- Load test ke liye jo `limit_req` / `limit_conn` lines comment ki thin, wapas on karein.
- `mywebsite.conf.bak` jaisi backup files `/etc/nginx/conf.d/` mein rehne se nuqsan nahi (`.bak` load nahi hoti), lekin zaroorat na ho to hata dein.
- Production mein **HTTPS** (Let's Encrypt), health checks aur monitoring par bhi ghaur karein.

---

## 16. Aur options

**Weighted balancing** (ek server ko zyada traffic):

```nginx
upstream backend_servers {
    server 192.168.0.13:80 weight=3;
    server 127.0.0.1:8090  weight=1;
}
```

**Least connections** (jis server par kam connections, usay request):

```nginx
upstream backend_servers {
    least_conn;
    server 192.168.0.13:80;
    server 127.0.0.1:8090;
}
```

**Failover settings** (3 baar fail hone par 30 second ke liye skip):

```nginx
upstream backend_servers {
    server 192.168.0.13:80 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8090  max_fails=3 fail_timeout=30s;
}
```

---

## 17. Quick reference

### Kya kahan chalana hai

| Kaam | Machine |
|---|---|
| Nginx config (`/etc/nginx/conf.d/mywebsite.conf`) | Ubuntu |
| `nginx -t` + reload (config badalne par) | Ubuntu |
| `scp` se files bhejna | Ubuntu |
| `cp` + `restorecon` (files rakhna) | RHEL |
| `curl` tests aur load tests | Ubuntu (ya teesri machine) |
| Cache saaf karna | Ubuntu |

### Important paths

| Cheez | Path |
|---|---|
| Ubuntu website files | `/var/www/mywebsite/` |
| RHEL website files | `/usr/share/nginx/html/` |
| Load balancer config | `/etc/nginx/conf.d/mywebsite.conf` |
| Config backup | `/etc/nginx/conf.d/mywebsite.conf.bak` |
| Cache folder | `/var/cache/nginx/mysite/` |
| Nginx error log | `/var/log/nginx/error.log` |

### Important ports

| Port | Kaun |
|---|---|
| `80` (Ubuntu) | Load balancer |
| `8090` (Ubuntu, sirf localhost) | Ubuntu ki internal website |
| `80` (RHEL) | RHEL ki website |
| `8080` | Jenkins (nahi chhedna) |
| `8081` | `nginx.conf` ka purana block (nahi chhedna) |

### Sab se zyada kaam aane wali commands

```bash
# Config check aur reload
sudo nginx -t && sudo systemctl reload nginx

# Load balancing check
for i in 1 2 3 4 5 6; do curl -sI http://192.168.0.18 | grep -i x-served-by || echo "RHEL"; done

# Cache check (do baar chalayein: MISS phir HIT)
curl -sI http://192.168.0.18/style.css | grep -iE "cache-control|expires|x-cache-status"

# Rate limit check
for i in $(seq 1 60); do curl -s -o /dev/null -w "%{http_code}\n" http://192.168.0.18/; done | sort | uniq -c

# Cache saaf karna
sudo rm -rf /var/cache/nginx/mysite/*
```

### Final status (asal tests ke nateeje)

| Feature | Nateeja |
|---|---|
| Load balancing (Ubuntu + RHEL, round robin) | Bari bari RHEL, Ubuntu aaya |
| 30 din browser cache | `Cache-Control: max-age=2592000` |
| Server-side cache | Pehli request `MISS`, doosri `HIT` |
| Rate limiting | 60 mein se 37 `200`, 23 `429` |
