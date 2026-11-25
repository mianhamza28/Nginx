# Nginx Load Balancer + HTTPS: Complete Guide (Ubuntu + RHEL + Let's Encrypt)

Is guide mein do servers par ek static website host karke Nginx ke zariye **round robin load balancing**, **30 din ki caching**, **rate limiting** aur domain `hamza-devops.work.gd` par **HTTPS (Let's Encrypt)** setup ki gayi hai. Is mein woh saare masail aur hal bhi shamil hain jo asal setup ke dauran saamne aaye.

**Zaroori:** Yeh site sirf **LAN** (ghar/office network) mein khulti hai, kyunke domain private IP `192.168.0.18` par point karta hai. Cloudflare is domain ke saath nahi laga, wajah Section 9 mein hai.

**Kaam ka tarteeb:** Steps 1 se 6 pehle plain HTTP par karein aur test karein. Domain aur HTTPS Step 7 aur 8 mein lagta hai. HTTPS ke baad kuch test commands badal jati hain, wo Section 11 mein hain.

## Fehrist

1. Setup ka khaka
2. Port ka intikhab (Jenkins wala masla)
3. Step 1: Ubuntu par basic load balancer
4. Step 2: RHEL par Nginx aur SSH
5. Step 3: Website files RHEL par bhejna
6. Step 4: Terminal se test
7. Step 5: Windows Firefox se test
8. Step 6: Caching aur Rate Limiting
9. Step 7: Domain aur DNS (DNSExit, Cloudflare ka masla)
10. Step 8: HTTPS (Let's Encrypt)
11. HTTPS ke baad tests (jo commands badli)
12. Cache busting (`?v=2`)
13. Website update karne ka tareeqa
14. `nginx -t` aur reload kab chahiye?
15. Capacity / Load testing
16. Failover test
17. Troubleshooting
18. Maintenance (certificate renewal waghera)
19. Production se pehle safai
20. Aur options
21. Aage ka raasta: internet par aur Cloudflare
22. Quick reference

---

## 1. Setup ka khaka

| Machine | OS | IP | Kaam |
| --- | --- | --- | --- |
| Server 1 | Ubuntu | `192.168.0.18` | Load balancer + website (internal port 8090) |
| Server 2 | RHEL | `192.168.0.13` | Website (port 80) |
| Client | Windows | Same network | Firefox se test |

Final setup (HTTPS ke baad):

```
 Firefox (Windows)
   https://hamza-devops.work.gd   (DNS ya hosts file -> 192.168.0.18)
            |
 +----------v-----------------------------+
 |  Ubuntu 192.168.0.18                   |
 |  :80   -> 301 redirect -> https        |
 |  :443  -> Nginx Load Balancer          |
 |           SSL + Cache + Rate Limit     |
 +----------+-----------------------------+
            |  round robin
   +--------+-------------------+
   |                            |
 +-v--------------------+   +---v-------------------+
 | Ubuntu 127.0.0.1:8090|   | RHEL 192.168.0.13:80  |
 | /var/www/mywebsite   |   | /usr/share/nginx/html |
 +----------------------+   +-----------------------+
```

Request 1 Ubuntu ko, request 2 RHEL ko, request 3 phir Ubuntu ko, aur isi tarah chalta rehta hai. Load balancer se backends tak traffic LAN ke andar plain HTTP par jata hai, HTTPS sirf browser aur load balancer ke beech hai.

**Kaun si command kahan chalani hai:** Har section mein saaf likha hai ke command **Ubuntu**, **RHEL** ya **Windows** par chalani hai. Ghalat machine par chalane se errors aate hain (Section 17 dekhein).

---

## 2. Port ka intikhab

Ubuntu par apni website ke liye internal port **`8090`** use kiya gaya hai. Doosre ports kyun nahi liye:

| Port | Wajah |
| --- | --- |
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

Pehle basic version likhein (caching, rate limit aur HTTPS baad mein lagayenge, taake pehle load balancing alag se test ho jaye):

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
| --- | --- |
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

**Teeno commands Ubuntu par** (yeh abhi plain HTTP wale stage ke liye hain):

```bash
curl http://192.168.0.13                 # RHEL seedha
curl http://127.0.0.1:8090               # Ubuntu ki internal website
curl -s http://192.168.0.18 | head -5    # Load balancer ke zariye
```

| Command | Kya check karti hai |
| --- | --- |
| `curl http://192.168.0.13` | RHEL ki website theek chal rahi hai |
| `curl http://127.0.0.1:8090` | Ubuntu ki internal website theek hai (Jenkins ka page nahi aana chahiye) |
| `curl -s http://192.168.0.18` | Load balancer kaam kar raha hai |

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

> **Cache aur HTTPS lagne ke baad:** Step 6 ke baad HTML cache mein aa jata hai, aur Step 8 ke baad `http://192.168.0.18` redirect de deta hai. Tab yeh test is tarah chalta hai (Section 11): `curl -sI -H "Pragma: no-cache" https://hamza-devops.work.gd`

---

## 7. Step 5: Windows Firefox se test

Windows machine Ubuntu ke same network par honi chahiye. Firefox mein (is stage par, HTTPS se pehle):

```
http://192.168.0.18
```

Dono servers par same website hai, is liye farq nazar nahi aayega. Do tareeqe hain:

### Tareeqa A: Developer Tools

1. `F12` dabayein, **Network** tab kholein.
2. `Ctrl+Shift+R` se hard refresh karein (cache se na aaye).
3. Sab se upar wali request par click karein.
4. **Headers > Response Headers** dekhein.
5. `X-Served-By: Ubuntu` hai to Ubuntu ne jawab diya, nahi hai to RHEL ne.

5-6 baar refresh karein, header kabhi aayega kabhi nahi.

### Tareeqa B: Footer mein fark likhna

Test ke liye dono servers ki `index.html` ke footer mein farq daalein. Footer file ke **bilkul neeche** hota hai.

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

### 8.2 Config (pura content replace karein)

Yeh abhi HTTP wala version hai. HTTPS wala final version Section 10.5 mein hai.

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

**Browser caching (`expires 30d`)** Browser CSS, JS aur images 30 din tak apne paas rakhta hai aur dobara server se nahi mangta. Response mein `Cache-Control: max-age=2592000` jata hai (2,592,000 seconds = 30 din).

**Server cache (`proxy_cache`)** Load balancer backend se aaya jawab apne paas save kar leta hai. Agli request backend tak jati hi nahi. `proxy_cache_use_stale` ki wajah se backend gir jaye to bhi purana cached page mil jata hai.

| Setting | Matlab |
| --- | --- |
| `proxy_cache_path ... max_size=200m` | Cache ka zyada se zyada size 200 MB |
| `inactive=30d` | 30 din tak kisi ne maanga nahi to cache se nikal do |
| `proxy_cache_valid 200 30d` (static) | Static files 30 din tak valid |
| `proxy_cache_valid 200 301 302 10m` (baaqi) | HTML waghera 10 minute tak valid |
| `proxy_cache_lock on` | Ek hi file ki kai requests aayein to backend ko sirf ek jaati hai |
| `proxy_cache_bypass $http_pragma` | `Ctrl+Shift+R` (ya `Pragma: no-cache` header) cache ko bypass kar deta hai |

**Rate limiting**

| Setting | Matlab |
| --- | --- |
| `rate=10r/s` | Ek IP se 10 requests per second |
| `burst=30` | Achanak 30 extra requests bhi guzar jayengi (page load mein HTML + CSS + JS + images mil kar kai requests banti hain) |
| `nodelay` | Burst wali requests ko intezar nahi karwata |
| `limit_conn perip_conn 30` | Ek IP se ek waqt mein 30 se zyada connections nahi |
| `limit_req_status 429` | Limit cross hone par `429 Too Many Requests` milta hai |

### 8.4 HTML par sirf 1 ghanta kyun?

HTML par `expires 1h` rakha hai, 30 din nahi. Agar HTML par bhi 30 din lagayein to `index.html` badalne par users ko 30 din tak purana page milega, kyunke browser server se poochega hi nahi. Phir bhi chahein to `location /` mein `expires 1h;` ki jagah `expires 30d;` likh dein.

### 8.5 Caching test (asal result, HTTP stage)

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
| --- | --- |
| `MISS` (pehli request) | Cache mein file nahi thi, backend se mangwai aur save kar li |
| `HIT` (doosri request) | Ab file load balancer ke apne cache se aayi, backend tak gayi hi nahi |
| `max-age=2592000` | 30 din |
| `Expires` | Request ke din se theek 30 din baad ki date |

### 8.6 Rate limit test (asal result, HTTP stage)

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

1. **Round robin test ab mushkil hoga.** HTML 10 minute cache mein rehta hai, to sab jawab `HIT` honge aur bari bari Ubuntu/RHEL nazar nahi aayega. Test ke liye `Pragma: no-cache` header ya `Ctrl+Shift+R` use karein (cache bypass), ya test ke dauran `proxy_cache mysite_cache;` wali line comment kar dein.
2. **Cache mein purana `X-Served-By` header** bhi save ho jata hai. HIT par bhi wohi server ka naam dikhta hai jis se pehli baar file aayi thi.
3. **Rate limit apke apne IP par bhi lagti hai.** Firefox se bhi kabhi `429` mile to kuch second ruk kar dobara try karein.
4. **Ek hi router ke peeche kai log (NAT)** ek hi IP se aate hain, wahan limit jaldi lag sakti hai. Zaroorat ho to `rate` ya `burst` barhayein.

---

## 9. Step 7: Domain aur DNS (DNSExit, Cloudflare ka masla)

### 9.1 Maqsad aur hadd

Domain `hamza-devops.work.gd` ko `192.168.0.18` par point karna hai taake `https://hamza-devops.work.gd` khule. `192.168.0.18` private address hai, is liye domain **sirf aap ke LAN ke andar** khulega. Internet par bahar ke log ya mobile data se nahi.

### 9.2 Cloudflare is domain ke saath kyun nahi chala

Cloudflare dashboard mein yeh raasta try kiya gaya: **Domains > Overview > Onboard a domain (Add a site) > Connect a domain** aur domain `hamza-devops.work.gd` likha. Jawab mein yeh error aaya:

> Please ensure you are providing the root domain and not any subdomains (e.g., example.com, not subdomain.example.com)

Matlab:

- `hamza-devops.work.gd`, `work.gd` ka subdomain hai, aur Cloudflare Free plan par sirf root domain add hota hai. Is liye Cloudflare ne domain qabool nahi kiya aur us ke nameservers bhi nahi diye.
- **Registrations** wala menu naya domain **kharidne** ke liye hota hai, is mein aap ka kaam nahi.
- Agar domain add ho bhi jaye to Cloudflare private IP (`192.168.x.x`) tak pohonch hi nahi sakta. Is ka hal Cloudflare Tunnel hota hai, lekin wo bhi domain Cloudflare par active hone ke baad. Is ka raasta Section 21 mein hai.

Is liye is setup mein SSL **Let's Encrypt (certbot)** se liya gaya (Section 10). SSL encryption ke lihaz se dono ka certificate browser ko same TLS encryption deta hai, farq sirf Cloudflare ki extra suhulat (CDN, DDoS protection) mein hai.

### 9.3 DNS provider: DNSExit

Domain ke nameservers `ns10.dnsexit.com`, `ns11.dnsexit.com`, `ns12.dnsexit.com`, `ns13.dnsexit.com` hain, yani DNS records DNSExit ke panel mein banenge.

> **Khabardar:** Panel ke **Modify Name Servers** page par **kuch bhi na likhein aur Submit na karein**. Wo page nameservers badalne ke liye hai, DNS records ke liye nahi. Wahan galat ya khali value daalne se domain DNSExit se hat jayega aur kaam karna band kar dega.

### 9.4 A record `192.168.0.18` par karna

DNSExit ke records page par **Host Records (A & AAAA)** mein `hamza-devops.work.gd.` ka A record pehle se maujood tha (purana IP `154.208.38.229`). Usay badalna hai, naya nahi banana.

1. **Add A/Host Record** ya **Add New Host** ka form **na** bharein. Wahan se naya record banane par ek hi naam ke do A record ho jate hain (purana public IP aur naya local IP) aur browser kabhi khulta hai kabhi nahi.
2. Existing row par **Edit** dabayein. Page ka title **Update Host for HAMZA-DEVOPS.WORK.GD** hona chahiye.
3. **IP is:** `Static` rehne dein.
4. **TTL:** hours ka dropdown `0` karein aur minutes `5` karein. Dhyan rahe ke agar sirf minutes badle aur hours `8` reh gaye to TTL `8 hours 5 minutes` ban jata hai.
5. **IP Address** ke pehle box mein purana IP mita kar `192.168.0.18` likhein. Baqi teen boxes khali rehne dein.
6. Neela **Submit** button dabayein.
7. Records ki list mein wapas aa kar check karein ke IP `192.168.0.18` dikha raha hai.

`www.hamza-devops.work.gd` ka CNAME record `hamza-devops.work.gd` ko point karta hai, usay chhedne ki zaroorat nahi.

> **DNSExit ki ek aur baat:** Kuch pages (jaise TXT record wala page) par upar laal **Modified** aur **Save Changes** button nazar aata hai, aur message likha hota hai "DNS changes are not saved yet". Jab tak **Save Changes** na dabayein, record DNS par live nahi hota.

### 9.5 Check: DNSExit ka apna jawab

**Windows par** (cache ko bypass kar ke seedha DNSExit se poochna):

```
nslookup hamza-devops.work.gd ns10.dnsexit.com
```

Jawab mein `Address: 192.168.0.18` aana chahiye. Is output ko padhne ke liye:

- Agar "Non-authoritative answer" likha ho to jawab kisi cache se aaya hai. Seedha DNSExit se aaye to yeh lafz nahi hota.
- Output ke shuru mein "DNS request timed out" ki ek line normal ho sakti hai. Wo server ke naam ki reverse lookup hoti hai, asal jawab uske neeche aata hai.
- Agar abhi bhi purana IP (`154.208.38.229`) aaye, to tabdeeli save nahi hui. Section 9.4 dobara karein, aur Submit ke baad list mein IP dekhein.

### 9.6 Purana IP cache mein atka rahe (hosts file ka hal)

DNSExit mein record badalne ke baad bhi Ubuntu, Windows ya router purana IP kuch ghante tak yaad rakh sakte hain (purane record ka TTL 8 ghante ka tha). Ubuntu par check:

```bash
getent hosts hamza-devops.work.gd
```

Agar `154.208.38.229` aaye to site timeout degi, kyunke browser purane public IP par ja raha hai. Hal:

**Ubuntu par cache saaf karein:**

```bash
sudo resolvectl flush-caches
getent hosts hamza-devops.work.gd
```

**Phir bhi purana aaye to router ko restart karein** (power off/on), ya intezar karein (zyada se zyada 8 ghante).

**Foran chalane ke liye hosts file mein temporary entry:**

Ubuntu par:

```bash
echo "192.168.0.18 hamza-devops.work.gd" | sudo tee -a /etc/hosts
getent hosts hamza-devops.work.gd
```

Windows par:

1. Notepad ko **Run as administrator** kholein.
2. **File > Open** par jayein, file type **All Files (\*.\*)** karein (warna hosts file nazar nahi aati).
3. Path: `C:\Windows\System32\drivers\etc\hosts`
4. Sab se neeche nayi line par yeh likhein (line ke shuru mein `#` na ho):

   ```
   192.168.0.18 hamza-devops.work.gd
   ```
5. **Ctrl+S** se save karein. Agar **Save As** ki window aa jaye to **Don't Save** karein, kyunke wo nayi khali file save ho rahi hoti hai. Hosts file admin Notepad mein **File > Open** se dobara kholein.
6. Command Prompt mein:

   ```
   ipconfig /flushdns
   ping hamza-devops.work.gd
   ```

   Jawab `192.168.0.18` se aana chahiye.

Agar Command Prompt (Administrator) se line daalni ho:

```
echo 192.168.0.18 hamza-devops.work.gd>> C:\Windows\System32\drivers\etc\hosts
```

Yeh line command ke taur par **nahi** likhni (bina `echo` ke likhne par `is not recognized as an internal or external command` error aata hai), wo hosts file ke **andar** likhi jati hai.

Jab public DNS khud theek ho jaye to dono jagah se yeh hosts line **hata dein** (Section 18).

---

## 10. Step 8: HTTPS (Let's Encrypt)

Server public internet se directly verify nahi ho sakta (private IP hai), is liye Let's Encrypt ka **DNS challenge** use hota hai. Certbot ek TXT value deta hai jo DNSExit mein daalni hoti hai. Is tareeqe mein port 80 ya 443 ka bahar se khulna zaroori nahi.

### 10.1 Certbot install (Ubuntu par)

```bash
sudo apt update
sudo apt install certbot -y
```

### 10.2 Certificate maangein (Ubuntu par)

```bash
sudo certbot certonly --manual --preferred-challenges dns -d hamza-devops.work.gd
```

Certbot pehli baar email, terms (`Y`) aur newsletter ka sawal poochta hai. Phir yeh dikhata hai:

```
Please deploy a DNS TXT record under the name:
_acme-challenge.hamza-devops.work.gd
with the following value:
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Press Enter to Continue
```

**Enter abhi na dabayein.** Terminal wahin khula rehne dein.

### 10.3 TXT record DNSExit mein daalein

| Field | Value |
| --- | --- |
| Type | `TXT` |
| Name | `_acme-challenge` (DNSExit khud poora naam `_acme-challenge.hamza-devops.work.gd.` dikhata hai) |
| Value | certbot ki di hui string, bilkul wahi |
| TTL | chhota rakhein |

1. DNSExit ke domain DNS page par **TXT Records** ke hisse mein **Add TXT Record** dabayein.
2. Value certbot ke terminal se copy-paste karein. Ek harf bhi alag hua to certbot fail hoga. Khaas tor par capital `I` aur chhote `l` ka dhyan rakhein, dono dekhne mein ek jaise lagte hain.
3. Record add hone par message aata hai "TXT Record ... has been successfully added", lekin saath hi "**DNS changes are not saved yet**" bhi likha hota hai. Upar laal **Save Changes** button dabana **zaroori** hai. Jab tak save na ho, record live nahi hota aur certbot ko nazar nahi aata.

### 10.4 Enter dabane se pehle check karein

**Windows par:**

```
nslookup -type=TXT _acme-challenge.hamza-devops.work.gd ns10.dnsexit.com
```

Jawab mein `text = "..."` ke andar wahi value aani chahiye jo certbot ne di.

- Agar sab requests "DNS request timed out" ho jayein, to is ka matlab yeh nahi ke record nahi hai. `ns10.dnsexit.com` ne us waqt jawab nahi diya. 1-2 minute baad dobara chalayein, ya doosre servers (`ns11`, `ns12`, `ns13`) aur Google DNS se poochein:

  ```
  nslookup -type=TXT _acme-challenge.hamza-devops.work.gd ns11.dnsexit.com
  nslookup -type=TXT _acme-challenge.hamza-devops.work.gd 8.8.8.8
  ```
- Ubuntu se bhi check ho sakta hai (Windows ka network masla ho to):

  ```bash
  sudo apt install dnsutils -y
  dig TXT _acme-challenge.hamza-devops.work.gd @ns10.dnsexit.com +short
  ```
- Kisi se bhi jawab na aaye to DNSExit par dekhein ke **Save Changes** waqai ho gaya hai (laal "Modified" ka nishan hat jana chahiye).

Value nazar aa jaye to Ubuntu terminal mein **Enter** dabayein. Kamyabi par:

```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/hamza-devops.work.gd/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/hamza-devops.work.gd/privkey.pem
```

Check:

```bash
sudo ls /etc/letsencrypt/live/hamza-devops.work.gd/
```

Asal output: `cert.pem  chain.pem  fullchain.pem  privkey.pem  README`.

**Agar value alag nikli ya certbot ka session band ho gaya:** certbot ki command dobara chalayein. Woh **nayi value** dega. Us surat mein DNSExit mein purane TXT record ko **Edit** karke nayi value daalein (naya record na banayein, warna do records ho jayenge), **Save Changes** dabayein, `nslookup` se nayi value check karein, aur tab Enter dabayein.

> Certificate ban jane ke baad `_acme-challenge` TXT record ki zaroorat nahi rehti. Wo HTTPS ko nuqsan nahi pahunchata, chahein to baad mein delete kar dein. Agli renewal par naya TXT record chahiye hoga (Section 18).

### 10.5 Final Nginx config (`/etc/nginx/conf.d/mywebsite.conf`)

Pehle backup lein, phir poori file replace karein:

```bash
sudo cp /etc/nginx/conf.d/mywebsite.conf /etc/nginx/conf.d/mywebsite.conf.bak
sudo nano /etc/nginx/conf.d/mywebsite.conf
```

Is mein port 80 sirf redirect karta hai, aur load balancer ab `443 ssl` par hai:

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

# ---------- HTTP -> HTTPS redirect ----------
server {
    listen 80;
    server_name hamza-devops.work.gd 192.168.0.18;
    return 301 https://hamza-devops.work.gd$request_uri;
}

# ---------- Load balancer (HTTPS) ----------
server {
    listen 443 ssl;
    server_name hamza-devops.work.gd 192.168.0.18;

    ssl_certificate     /etc/letsencrypt/live/hamza-devops.work.gd/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/hamza-devops.work.gd/privkey.pem;

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

    # Server-side cache
    proxy_cache mysite_cache;
    proxy_cache_valid 200 301 302 10m;
    proxy_cache_valid 404 1m;
    proxy_cache_lock on;
    proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    proxy_cache_bypass $http_pragma;
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

Is config ki khaas baatein:

- `X-Forwarded-Proto $scheme` ab `https` bhejta hai, jo sahi hai.
- Rate limit aur cache sirf `443` wale block mein hain. `80` sirf redirect karta hai.
- `server_name` mein `192.168.0.18` bhi hai, is liye `http://192.168.0.18` bhi domain par redirect hota hai. `https://192.168.0.18` seedha kholne par certificate warning aati hai, kyunke certificate domain ke naam par hai, IP par nahi.

### 10.6 Firewall, check aur reload

```bash
sudo ufw status
sudo ufw allow 443/tcp
sudo nginx -t && sudo systemctl reload nginx
sudo ss -tlnp | grep nginx
```

`ss` ke output mein `:80` ke saath `:443` bhi nazar aana chahiye. Is setup mein yeh ports nazar aaye: `8081` (`nginx.conf` ka purana block, nahi chhedna), `127.0.0.1:8090` (Ubuntu ki internal website), `80` aur `443`.

Agar `nginx -t` mein `cannot load certificate` error aaye to certificate abhi banaya hi nahi gaya, pehle Section 10.4 poora karein.

### 10.7 Firefox mein test

Windows par pehle Section 9.6 ki hosts entry (agar DNS abhi purane IP par ho), phir Firefox ko poori tarah band kar ke dobara kholein aur `https://hamza-devops.work.gd` likhein (`https://` ke saath). Address bar mein lock nazar aana chahiye.

---

## 11. HTTPS ke baad tests (jo commands badli)

HTTPS lagne ke baad `http://192.168.0.18` par jo purane tests chalte the, wo ab `301` redirect de rahe hain. Isliye ab yeh commands use karein. Ubuntu par hosts entry (Section 9.6) honi chahiye, warna har command mein `--resolve hamza-devops.work.gd:443:192.168.0.18` lagayein.

| Kaam | Purani command (HTTP) | Ab (HTTPS) |
| --- | --- | --- |
| Load balancer check | `curl -sI http://192.168.0.18` | `curl -sI https://hamza-devops.work.gd` |
| Round robin | `curl -sI http://192.168.0.18` | `curl -sI -H "Pragma: no-cache" https://hamza-devops.work.gd` |
| Cache test | `http://192.168.0.18/style.css` | `https://hamza-devops.work.gd/style.css` |
| Rate limit | `http://192.168.0.18/` | `https://hamza-devops.work.gd/` |

### 11.1 HTTPS aur certificate

```bash
curl -sI https://hamza-devops.work.gd | head -5
curl -vI https://hamza-devops.work.gd 2>&1 | grep -iE "subject|issuer|expire|SSL certificate verify"
```

Dekhna hai:

- `HTTP/1.1 200 OK`
- `issuer: ... Let's Encrypt ...`
- `SSL certificate verify ok`
- `expire date:` wali line mein certificate ki asli expiry. Is setup mein `Feb 22 18:38:55 2027 GMT` aayi.

> **Dhyan:** `curl` ke output mein jo `Expires: ...` header aata hai (jaise `Tue, 24 Nov 2026 ...`), wo web page ka browser cache header hai (HTML par `expires 1h`), certificate ki expiry nahi.

### 11.2 HTTP se HTTPS redirect

```bash
curl -sI http://hamza-devops.work.gd | head -3
```

Aana chahiye: `HTTP/1.1 301 Moved Permanently` aur `Location: https://hamza-devops.work.gd/`.

Agar reload ke foran baad test chalane par `200 OK` aaye, to command dobara chalayein. DNS ko bypass kar ke seedha Nginx se test:

```bash
curl -sI -H "Host: hamza-devops.work.gd" http://127.0.0.1
```

### 11.3 Round robin (cache bypass ke saath)

```bash
for i in 1 2 3 4 5 6; do curl -sI -H "Pragma: no-cache" https://hamza-devops.work.gd | grep -i x-served-by || echo "RHEL"; done
```

`Pragma: no-cache` header cache ko bypass karta hai (`proxy_cache_bypass $http_pragma`), is liye har request backend tak jati hai aur Ubuntu/RHEL bari bari nazar aate hain. Header ke baghair HTML cache se (`HIT`) aata hai aur har baar same server ka naam dikhta hai.

### 11.4 Cache test

Do baar chalayein:

```bash
curl -sI https://hamza-devops.work.gd/style.css | grep -iE "cache-control|expires|x-cache-status"
curl -sI https://hamza-devops.work.gd/style.css | grep -iE "cache-control|expires|x-cache-status"
```

Pehli request `MISS`, doosri `HIT`, aur `Cache-Control: max-age=2592000` (30 din). Cache key mein `https` shamil hai, is liye HTTP stage ki cached files is par asar nahi karteen, HTTPS par pehli request dobara `MISS` hogi.

### 11.5 Rate limit test

```bash
for i in $(seq 1 60); do curl -s -o /dev/null -w "%{http_code}\n" https://hamza-devops.work.gd/; done | sort | uniq -c
```

Is mein `200` aur `429` dono aane chahiye. Har `curl` naya TLS handshake karta hai, is liye numbers HTTP stage ke nateeje (37 `200`, 23 `429`) se thode alag ho sakte hain. `429` na aaye to 200 requests try karein.

### 11.6 Domain ke baghair (DNS bypass) test

```bash
curl -sI --resolve hamza-devops.work.gd:443:192.168.0.18 https://hamza-devops.work.gd | head -5
```

---

## 12. Cache busting

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

## 13. Website update karne ka tareeqa

Jab bhi website mein tabdeeli karein, **dono servers par files update karni hongi**, warna users ko kabhi purana kabhi naya page milega.

1. Ubuntu par `/var/www/mywebsite/` mein file badlein.
2. RHEL par bhejein:

```bash
scp -r /var/www/mywebsite/* RHEL_USERNAME@192.168.0.13:/tmp/ && \
ssh RHEL_USERNAME@192.168.0.13 "sudo cp -r /tmp/index.html /tmp/style.css /usr/share/nginx/html/ && sudo restorecon -Rv /usr/share/nginx/html/"
```

3. Agar CSS/JS badli hai to `?v=` badlein (Section 12).
4. Load balancer ka cache saaf karein:

```bash
sudo rm -rf /var/cache/nginx/mysite/*
sudo systemctl reload nginx
```

---

## 14. `nginx -t` aur reload kab chahiye?

| Kya badla | `nginx -t` + reload chahiye? |
| --- | --- |
| Sirf HTML/CSS/images (website files) | **Nahi** |
| Footer ka text badla | **Nahi** |
| Nginx config (`nginx.conf`, `conf.d/*.conf`) | **Haan** |
| RHEL par sirf files copy kin | **Nahi** |
| Naya SSL certificate banaya (renewal) | **Haan** (reload se naya certificate load hota hai) |
| DNS ya hosts file badli | **Nahi** |

Nginx website files har request par disk se padhta hai, is liye nayi files foran nazar aati hain. Config badli ho to:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 15. Capacity / Load testing

Maqsad: dekhna ke server kitni requests per second handle karta hai aur kab slow ya fail hota hai.

### 15.1 Tool install

```bash
# Ubuntu
sudo apt install apache2-utils -y

# RHEL
sudo dnf install httpd-tools -y
```

Test **kisi teesri machine** (Windows ya koi aur Linux) se chalana behtar hai. Load balancer par hi chalayenge to tool khud CPU kha jayega. Us machine par bhi domain ki hosts entry (Section 9.6) honi chahiye.

### 15.2 Test chalayein

```bash
ab -n 10000 -c 100 https://hamza-devops.work.gd/
```

- `-n 10000` = kul 10,000 requests
- `-c 100` = ek waqt mein 100 users
- URL ke aakhir mein `/` zaroor likhein
- HTTPS par har naya connection TLS handshake karta hai, jo CPU lagata hai. Zyada realistic numbers ke liye `-k` (keep-alive) bhi try karein: `ab -k -n 10000 -c 100 https://hamza-devops.work.gd/`

### 15.3 Nateeja padhna

| Field | Matlab |
| --- | --- |
| `Requests per second` | Asal capacity |
| `Time per request` | Ek request mein kitne ms laga |
| `Failed requests` | **0 hona chahiye** |
| `Non-2xx responses` | Errors (429, 502, 503) |
| `Percentage served within a certain time` | 95% / 99% requests kitne ms mein mukammal hui |

### 15.4 Aahista aahista load barhayein

```bash
ab -n 5000  -c 10   https://hamza-devops.work.gd/
ab -n 10000 -c 50   https://hamza-devops.work.gd/
ab -n 20000 -c 100  https://hamza-devops.work.gd/
ab -n 30000 -c 300  https://hamza-devops.work.gd/
ab -n 50000 -c 500  https://hamza-devops.work.gd/
```

Jis point par `Failed requests` aane lagein ya `Time per request` bohat barh jaye, wahi aap ke setup ki taqriban limit hai.

### 15.5 Test ke dauran monitoring

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

### 15.6 `wrk` (zyada tez tool)

```bash
sudo apt install wrk -y
wrk -t4 -c200 -d30s https://hamza-devops.work.gd/
```

### 15.7 Bohat zaroori: rate limit pehle band karein

Rate limiting lagi ho to load test mein `429` aayenge aur server ki asal capacity pata nahi chalegi. Test se pehle `443` wale load balancer block mein yeh lines **comment** kar dein:

```nginx
    # limit_req    zone=perip_req  burst=30 nodelay;
    # limit_req_status 429;
    # limit_conn   perip_conn 30;
    # limit_conn_status 429;
```

`sudo nginx -t && sudo systemctl reload nginx`. Test ke baad **wapas on kar dein**.

### 15.8 Baaqi baatein

1. **Sirf apne servers par test karein.** Kisi aur ki website par load test karna ghair qanooni hai.
2. Static website par numbers bohat ache aayenge. Asal dynamic app (database, PHP) par nateeja kam hoga.
3. Network (Wi-Fi/VM) slow ho to asal capacity nazar nahi aati.
4. Aap ke servers VMs lagte hain (hostname `osboxes`), nateeja VM ke CPU cores aur RAM par depend karega.
5. Round robin ka asar dekhna ho to test ke dauran dono servers par `top` kholein. Dono ka CPU taqriban barabar barhna chahiye.

---

## 16. Failover test

Dekhein ke ek server band ho to kya hota hai.

1. **RHEL par:** `sudo systemctl stop nginx`
2. **Ubuntu par** 6 baar yeh chalayein (cache ko bypass karne ke liye `Pragma: no-cache` zaroori hai, warna cached jawab milte rahenge aur failover nazar nahi aayega):

   ```bash
   for i in 1 2 3 4 5 6; do curl -sI -H "Pragma: no-cache" https://hamza-devops.work.gd | head -1; done
   ```

   Sab jawab Ubuntu se aane chahiye, website chalti rahegi.
3. Test ke baad **RHEL par:** `sudo systemctl start nginx`

Nginx kisi backend ke fail hone par usay kuch der ke liye skip kar deta hai (`max_fails` aur `fail_timeout` ke hisaab se, Section 20 dekhein).

Cache ki wajah se `proxy_cache_use_stale` bhi kaam karta hai: agar dono backend gir jayein to bhi cached pages mil jate hain.

---

## 17. Troubleshooting

### 17.1 Masail aur hal

| Masla | Wajah | Hal |
| --- | --- | --- |
| `cp: target '/usr/share/nginx/html/' is not a directory` | Command Ubuntu par chali, RHEL par nahi | Yeh command **RHEL** par chalayein |
| `sudo: restorecon: command not found` | Ubuntu par SELinux nahi hota, ya RHEL par package nahi | RHEL par chalayein, ya `sudo dnf install policycoreutils` |
| `curl http://127.0.0.1:8080` par Jenkins ka login page | Port 8080 par Jenkins chal raha hai | Alag port (`8090`) use karein |
| Port 8081 par `ss` mein nginx dikhe | `nginx.conf` line 36 mein `listen 8081;` | `sudo grep -rn "8081" /etc/nginx/`, ya doosra port chunein |
| RHEL par **403 Forbidden** | SELinux context ghalat | `sudo restorecon -Rv /usr/share/nginx/html/` |
| RHEL par page nahi khulta | Firewall port 80 band | `sudo firewall-cmd --permanent --add-service=http && sudo firewall-cmd --reload` |
| `502 Bad Gateway` | Koi backend band hai ya port ghalat | Dono backend ko alag alag `curl` karke check karein |
| Kabhi purana kabhi naya page | Dono servers par files same nahi | Dono par files update karein |
| Windows se page nahi khulta | Ubuntu firewall | `sudo ufw allow 80/tcp` aur `sudo ufw allow 443/tcp` |
| `scp` mein `Connection refused` | RHEL par `sshd` band | `sudo systemctl enable --now sshd` |
| Footer badla par Firefox mein purana dikha | Cache (browser ya load balancer) | `Ctrl+Shift+R`, aur `sudo rm -rf /var/cache/nginx/mysite/*` |
| CSS badli par browser mein purani nazar aati hai | 30 din ki browser cache | `style.css?v=` ka number barhayein (Section 12) |
| Sab requests `429` de rahi hain | Rate limit bohat sakht, ya NAT ke peeche kai log | `rate` / `burst` barhayein |
| `ab` test mein bohat `Non-2xx responses` | Rate limit load test ko rok rahi hai | Test ke dauran `limit_req` comment karein |
| `nginx -t` mein `proxy_cache_path` error | Cache folder ki permission ya ghalat syntax | Config dobara check karein, `sudo mkdir -p /var/cache/nginx` |
| Round robin nazar nahi aata (sab `HIT`) | HTML cache ho gayi | `Pragma: no-cache` header ya `Ctrl+Shift+R` use karein |

### 17.2 Domain, DNS aur HTTPS ke masail

| Masla | Wajah | Hal |
| --- | --- | --- |
| Cloudflare: "provide the root domain and not any subdomains" | `hamza-devops.work.gd` subdomain hai, Free plan sirf root domain leta hai | Is domain ke saath Cloudflare nahi chalega. Let's Encrypt (Section 10) ya naya root domain (Section 21) |
| Cloudflare ka **Registrations** menu | Yeh domain kharidne ke liye hai | **Domains > Overview > Onboard a domain** par jayein |
| DNSExit par nameservers wala page kholne par form khali dikhta hai | Yeh nameservers badalne ka page hai | **Kuch na likhein, Submit na karein.** DNS records wale page par jayein |
| A record ke do alag IP | Edit ki jagah **Add New Host** se naya record bana diya | Purana record **Edit** karein, extra record delete karein |
| TTL `8 hours 5 minutes` ban gaya | Minutes badle, hours dropdown `8` par reh gaya | Hours ko `0` karein |
| `nslookup ... ns10.dnsexit.com` purana IP de raha hai | Tabdeeli save nahi hui | Update Host dobara karein, Submit dabayein, list mein IP check karein |
| `getent hosts` ya `nslookup` purana IP `154.208.38.229` de raha hai | Ubuntu, Windows ya router ka cache (purana TTL 8 ghante) | Section 9.6: cache flush, router restart, ya hosts file entry |
| Firefox: **The connection has timed out** | Windows par domain purane public IP par resolve ho raha tha | Hosts entry, `ipconfig /flushdns`, `ping` mein `192.168.0.18` dekhein |
| TXT record banane ke baad certbot ko nazar nahi aata | DNSExit par **Save Changes** nahi dabaya | Laal **Save Changes** dabayein, phir `nslookup -type=TXT` se check |
| `nslookup -type=TXT` mein "DNS request timed out" | DNSExit server ne us waqt jawab nahi diya (record na hone ka matlab nahi) | Dobara chalayein, `ns11` / `ns12` / `ns13` ya `8.8.8.8` try karein, Ubuntu par `dig` |
| Certbot: challenge fail | TXT value mein ek bhi harf alag (`I` aur `l` ka farq), ya purani value | Value dobara copy karein, certbot dobara chala kar nayi value lein, purana TXT **Edit** karein |
| `nginx -t`: `cannot load certificate` | Certificate abhi bana hi nahi | `sudo ls /etc/letsencrypt/live/hamza-devops.work.gd/`, na ho to Section 10.4 |
| Certbot ne naya folder banaya (`...-0001`) | Pehle se us naam ka certificate maujood tha | Nginx mein `ssl_certificate` ka path naye folder par karein |
| Command Prompt mein `192.168.0.18 hamza-devops.work.gd` likhne par `is not recognized as an internal or external command` | Yeh hosts file ke **andar** likhi jane wali line hai, command nahi | Notepad (admin) se hosts file mein likhein |
| Notepad mein **Save As** window aa gayi | Nayi khali (Untitled) file save ho rahi hai, hosts file khuli hi nahi | **Don't Save**, phir admin Notepad mein **File > Open** se hosts file kholein |
| Hosts file Open window mein nazar nahi aati | File type `Text Documents (*.txt)` hai | **All Files (\*.\*)** chunein |
| `curl http://hamza-devops.work.gd` par `301` ki jagah `200 OK` | Test reload ke foran baad chal gaya | Dobara chalayein, ya `curl -sI -H "Host: hamza-devops.work.gd" http://127.0.0.1` |
| `curl` mein `Expires: Tue, 24 Nov 2026 ...` dekh kar certificate expiry ka shak | Yeh web page ka cache header hai (`expires 1h`) | Certificate ki asli expiry `expire date:` wali line mein hoti hai |
| `https://192.168.0.18` par certificate warning | Certificate domain ke naam par hai, IP par nahi | Hamesha `https://hamza-devops.work.gd` use karein |

### 17.3 Agar Firefox mein phir bhi timeout aaye: checklist

1. `ping hamza-devops.work.gd` (Windows) mein `192.168.0.18` aana chahiye. Warna hosts entry check karein.
2. Hosts line ke aage `#` na ho, aur file ka naam sirf `hosts` ho (`hosts.txt` nahi).
3. Windows mein check (port 443 khula hai?):

   ```
   powershell -command "Test-NetConnection 192.168.0.18 -Port 443"
   ```

   `TcpTestSucceeded : True` aana chahiye.
4. Agar server VM hai to network adapter **Bridged** ho (NAT nahi).
5. Firefox: **Settings > Privacy & Security > DNS over HTTPS** off karein.
6. Firefox mein proxy, VPN ya security extension band karke Private Window mein try karein.
7. Firefox ko poori tarah band karke dobara kholein, aur address mein `https://` poora likhein.

### 17.4 Kaam ki debugging commands

```bash
sudo ss -tlnp | grep nginx                      # nginx kin ports par sun raha hai
sudo grep -rn "listen" /etc/nginx/              # saari listen lines
sudo tail -f /var/log/nginx/error.log           # live error log
sudo tail -f /var/log/nginx/access.log          # live access log
sudo nginx -T | less                            # poori active config
sudo nginx -T 2>/dev/null | grep -nE "listen|server_name|configuration file"
ls /var/cache/nginx/mysite/                     # cache mein kya save hai
getent hosts hamza-devops.work.gd               # Ubuntu ko domain kis IP par dikh raha hai
sudo certbot certificates                       # certificate aur uski expiry
```

---

## 18. Maintenance (certificate renewal waghera)

### 18.1 Certificate renewal (har 90 din)

Let's Encrypt certificate **90 din** ka hota hai. Is setup ke certificate ki expiry **22 Feb 2027** hai. DNS challenge manual hone ki wajah se **auto-renew nahi hota**, is liye us se pehle (behtar hai ek hafte pehle) dobara karna hoga:

```bash
sudo certbot certonly --manual --preferred-challenges dns -d hamza-devops.work.gd
```

1. Agar certbot poochay ke maujooda certificate ka kya karein, to **renew** chunein.
2. Certbot **nayi TXT value** dega (purani kaam nahi aayegi). DNSExit mein purane `_acme-challenge` TXT record ko **Edit** karke nayi value daalein aur **Save Changes** dabayein.
3. `nslookup -type=TXT _acme-challenge.hamza-devops.work.gd ns10.dnsexit.com` se nayi value check karein, phir certbot mein Enter dabayein.
4. Naya certificate load karne ke liye reload karein:

   ```bash
   sudo nginx -t && sudo systemctl reload nginx
   ```
5. Expiry check:

   ```bash
   sudo certbot certificates
   ```

Agar certbot ne naya folder (`hamza-devops.work.gd-0001`) banaya ho to Nginx ke `ssl_certificate` paths wahi karein. Renewal ko khud chalane ke liye DNSExit ke API ke saath certbot hook bhi banaya ja sakta hai, lekin wo is setup mein try nahi kiya gaya.

### 18.2 Hosts entries hatana

Jab public DNS (`hamza-devops.work.gd`) khud `192.168.0.18` par theek ho jaye (`nslookup hamza-devops.work.gd` mein `192.168.0.18` aaye, aur router/ISP ka cache khatam ho jaye), to temporary hosts lines hata dein:

- **Ubuntu:** `sudo nano /etc/hosts` aur `192.168.0.18 hamza-devops.work.gd` wali line delete karein.
- **Windows:** Notepad (admin) se `C:\Windows\System32\drivers\etc\hosts` kholein aur wohi line delete karein, phir `ipconfig /flushdns`.

### 18.3 Baaqi yaad rakhne ki baatein

- Site sirf **LAN** (ghar/office network) mein khulegi, kyunke `192.168.0.18` private IP hai.
- `_acme-challenge` TXT record certificate banne ke baad rakhna zaroori nahi. Hata dein to agli renewal par naya banana hoga.
- Server-side cache ko test ke liye bypass karna ho to `Ctrl+Shift+R` (ya `Pragma: no-cache` header).
- Config badalne se pehle backup lein: `sudo cp /etc/nginx/conf.d/mywebsite.conf /etc/nginx/conf.d/mywebsite.conf.bak`

---

## 19. Production se pehle safai

- `add_header X-Served-By "Ubuntu";` sirf test ke liye tha. Chahein to delete kar ke `sudo systemctl reload nginx` kar dein.
- Footer mein jo `(Ubuntu)` / `(RHEL)` likha tha, wapas `All rights reserved.` kar dein.
- `add_header X-Cache-Status ...` bhi chahein to hata dein (debugging ke liye rakhna bhi theek hai).
- Load test ke liye jo `limit_req` / `limit_conn` lines comment ki thin, wapas on karein.
- Temporary hosts entries (Ubuntu aur Windows) hata dein jab DNS theek ho jaye (Section 18.2).
- `mywebsite.conf.bak` jaisi backup files `/etc/nginx/conf.d/` mein rehne se nuqsan nahi (`.bak` load nahi hoti), lekin zaroorat na ho to hata dein.
- Certificate ki renewal ki tareekh calendar mein likh lein (Section 18.1).
- Agar site internet par sab ke liye chalani ho to Section 21 dekhein. Wahan health checks aur monitoring par bhi ghaur karein.

---

## 20. Aur options

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

## 21. Aage ka raasta: internet par aur Cloudflare

Is waqt site sirf LAN mein hai aur SSL Let's Encrypt se hai. Agar baad mein site internet par sab ke liye chalani ho ya Cloudflare ki suhulat (CDN, DDoS protection, auto-renew SSL) chahiye, to yeh raaste hain. **Yeh hissa is setup mein try nahi kiya gaya**, isliye commands ko pehli baar chalane se pehle dhyan se parhein.

### 21.1 Naya root domain + Cloudflare Tunnel

1. Ek asal root domain len (`.com`, `.xyz` waghera, Cloudflare ke **Buy a domain** se ya kisi aur registrar se).
2. Cloudflare mein **Add a site > Connect a domain** se wo domain add karein, **Free** plan chunein, jo 2 nameservers milein wo registrar ke panel mein lagayein, aur status **Active** hone tak intezar karein.
3. Ubuntu par `cloudflared` install karein:

   ```bash
   wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
   sudo dpkg -i cloudflared-linux-amd64.deb
   cloudflared tunnel login
   cloudflared tunnel create mywebsite
   ```
4. `/etc/cloudflared/config.yml` mein (UUID aur domain apne likhein). **Is setup mein port 80 sirf redirect karta hai**, is liye tunnel ko `http://localhost:80` par nahi balke HTTPS port par bhejna hoga, warna redirect ka loop banega:

   ```yaml
   tunnel: <UUID>
   credentials-file: /etc/cloudflared/<UUID>.json
   
   ingress:
     - hostname: yourdomain.com
       service: https://localhost:443
       originRequest:
         noTLSVerify: true
     - service: http_status:404
   ```
5. DNS aur service:

   ```bash
   sudo mkdir -p /etc/cloudflared
   sudo cp ~/.cloudflared/<UUID>.json /etc/cloudflared/
   cloudflared tunnel route dns mywebsite yourdomain.com
   sudo cloudflared service install
   sudo systemctl enable --now cloudflared
   ```
6. Nginx ke `443` block mein naya domain `server_name` mein jodein, aur asal visitor ka IP lene ke liye yeh do lines daalein (warna rate limit sab visitors ko ek hi IP `127.0.0.1` samajh kar sab par mil kar lagegi):

   ```nginx
   set_real_ip_from 127.0.0.1;
   real_ip_header CF-Connecting-IP;
   ```
7. Cloudflare dashboard mein **SSL/TLS** mode **Full** rakhein, aur **Always Use HTTPS** on karein.

### 21.2 Cloudflare ke baghair, public IP se

Is mein DNS A record public IP par jata hai, router par port 80/443 forward hoti hai, aur Let's Encrypt ka HTTP challenge chal sakta hai. Is ke liye **public IP** chahiye. Check: router ke admin page par WAN IP dekhein aur `whatismyip.com` ke IP se milayein. Dono same hon to public IP hai, alag hon to ISP CGNAT use kar raha hai aur yeh raasta nahi chalega.

### 21.3 Sirf test ke liye, domain ke baghair

```bash
cloudflared tunnel --url https://localhost:443 --no-tls-verify
```

Is se `https://xxxx.trycloudflare.com` jaisa free HTTPS link milta hai (account nahi chahiye). Yeh permanent nahi hota, sirf testing ke liye hai.

---

## 22. Quick reference

### Kya kahan chalana hai

| Kaam | Machine |
| --- | --- |
| Nginx config (`/etc/nginx/conf.d/mywebsite.conf`) | Ubuntu |
| `nginx -t` + reload (config ya certificate badalne par) | Ubuntu |
| Certbot (certificate banana ya renew karna) | Ubuntu |
| TXT aur A record (DNS) | DNSExit panel (browser mein) |
| `scp` se files bhejna | Ubuntu |
| `cp` + `restorecon` (files rakhna) | RHEL |
| `curl` tests aur load tests | Ubuntu (ya teesri machine) |
| Hosts file entry (temporary) | Ubuntu aur Windows |
| Cache saaf karna | Ubuntu |

### DNS records (DNSExit)

| Type | Name | Value |
| --- | --- | --- |
| A | `hamza-devops.work.gd` | `192.168.0.18` (purana IP `154.208.38.229` tha) |
| CNAME | `www.hamza-devops.work.gd` | `hamza-devops.work.gd` |
| TXT | `_acme-challenge.hamza-devops.work.gd` | certbot ki di hui value (sirf certificate banane ke waqt) |

### Important paths

| Cheez | Path |
| --- | --- |
| Ubuntu website files | `/var/www/mywebsite/` |
| RHEL website files | `/usr/share/nginx/html/` |
| Load balancer config | `/etc/nginx/conf.d/mywebsite.conf` |
| Config backup | `/etc/nginx/conf.d/mywebsite.conf.bak` |
| SSL certificate | `/etc/letsencrypt/live/hamza-devops.work.gd/fullchain.pem` |
| SSL private key | `/etc/letsencrypt/live/hamza-devops.work.gd/privkey.pem` |
| Cache folder | `/var/cache/nginx/mysite/` |
| Nginx error log | `/var/log/nginx/error.log` |
| Windows hosts file | `C:\Windows\System32\drivers\etc\hosts` |

### Important ports

| Port | Kaun |
| --- | --- |
| `80` (Ubuntu) | HTTP, sirf HTTPS par redirect |
| `443` (Ubuntu) | Load balancer (HTTPS) |
| `8090` (Ubuntu, sirf localhost) | Ubuntu ki internal website |
| `80` (RHEL) | RHEL ki website |
| `8080` | Jenkins (nahi chhedna) |
| `8081` | `nginx.conf` ka purana block (nahi chhedna) |

### Sab se zyada kaam aane wali commands

```bash
# Config check aur reload
sudo nginx -t && sudo systemctl reload nginx

# HTTPS aur certificate check
curl -sI https://hamza-devops.work.gd | head -5
curl -vI https://hamza-devops.work.gd 2>&1 | grep -iE "subject|issuer|expire|SSL certificate verify"

# HTTP -> HTTPS redirect check (301 aana chahiye)
curl -sI http://hamza-devops.work.gd | head -3

# Load balancing check (cache bypass ke saath)
for i in 1 2 3 4 5 6; do curl -sI -H "Pragma: no-cache" https://hamza-devops.work.gd | grep -i x-served-by || echo "RHEL"; done

# Cache check (do baar chalayein: MISS phir HIT)
curl -sI https://hamza-devops.work.gd/style.css | grep -iE "cache-control|expires|x-cache-status"

# Rate limit check
for i in $(seq 1 60); do curl -s -o /dev/null -w "%{http_code}\n" https://hamza-devops.work.gd/; done | sort | uniq -c

# Cache saaf karna
sudo rm -rf /var/cache/nginx/mysite/*

# Certificate ki expiry
sudo certbot certificates
```

```
:: Windows (CMD)
ipconfig /flushdns
ping hamza-devops.work.gd
nslookup hamza-devops.work.gd ns10.dnsexit.com
type C:\Windows\System32\drivers\etc\hosts
```

### Final status (asal tests ke nateeje)

| Feature | Nateeja |
| --- | --- |
| Load balancing (Ubuntu + RHEL, round robin) | Bari bari RHEL, Ubuntu aaya |
| 30 din browser cache | `Cache-Control: max-age=2592000` |
| Server-side cache | Pehli request `MISS`, doosri `HIT` |
| Rate limiting | 60 mein se 37 `200`, 23 `429` (HTTP stage ka nateeja) |
| Domain DNS | `hamza-devops.work.gd` par `192.168.0.18` (DNSExit se confirm) |
| SSL certificate | Let's Encrypt, `fullchain.pem` aur `privkey.pem` ban gaye, expiry 22 Feb 2027 |
| Nginx | `:80` (redirect) aur `:443` (HTTPS) par sun raha hai, `nginx -t` successful |


# LAN par Nginx Load Balancer ke liye HTTPS (Let's Encrypt) - Complete Guide

**Domain:** `hamza-devops.work.gd`
**Load balancer:** Ubuntu `192.168.0.18` (Nginx)
**Backends:** RHEL `192.168.0.13:80` aur Ubuntu local site `127.0.0.1:8090`
**Certificate:** Let's Encrypt (manual DNS challenge)

---

## 1. Masla kya tha?

Maqsad yeh tha ke ghar/office ke LAN mein `https://hamza-devops.work.gd` khule aur address bar mein **lock** nazar aaye.

Shuru mein:

- Nginx sirf `:80` par sun raha tha (sirf HTTP).
- Domain ka public DNS purane IP `154.208.38.229` par ja raha tha, jabke site `192.168.0.18` (private IP) par chal rahi hai.
- Let's Encrypt certificate chahiye tha, lekin server public internet se directly verify nahi ho sakta, is liye **DNS challenge** use karna pada.

---

## 2. Architecture (kaise kaam karta hai)

```
Windows PC (Firefox)
      |
      |  https://hamza-devops.work.gd  (hosts file -> 192.168.0.18)
      v
Ubuntu Nginx :443 (SSL + rate limit + cache)
      |
      +--> RHEL 192.168.0.13:80
      +--> Ubuntu 127.0.0.1:8090
```

---

## 3. Solution - Step by Step

### Step 1: Certificate banana (DNS challenge)

Certbot ke manual DNS mode mein certbot ek TXT value deta hai. Usay DNS provider (DNSExit) mein record bana kar daalte hain:

| Field | Value |
|-------|-------|
| Type | `TXT` |
| Name | `_acme-challenge.hamza-devops.work.gd` |
| Value | certbot ki di hui string |

Record publish hone ke baad check karein:

```bash
nslookup -type=TXT _acme-challenge.hamza-devops.work.gd ns10.dnsexit.com
```

Jab TXT value nazar aaye, certbot mein Enter dabayein. Certificate yahan ban jata hai:

```
/etc/letsencrypt/live/hamza-devops.work.gd/fullchain.pem
/etc/letsencrypt/live/hamza-devops.work.gd/privkey.pem
```

> Certificate ban jane ke baad TXT record ki zaroorat nahi rehti. Wo HTTPS ko nuqsan nahi pahunchata, baad mein delete kar sakte hain.

### Step 2: Nginx config (`/etc/nginx/conf.d/mywebsite.conf`)

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

# ---------- HTTP -> HTTPS redirect ----------
server {
    listen 80;
    server_name hamza-devops.work.gd 192.168.0.18;
    return 301 https://hamza-devops.work.gd$request_uri;
}

# ---------- Load balancer (HTTPS) ----------
server {
    listen 443 ssl;
    server_name hamza-devops.work.gd 192.168.0.18;

    ssl_certificate     /etc/letsencrypt/live/hamza-devops.work.gd/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/hamza-devops.work.gd/privkey.pem;

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

    # Server-side cache
    proxy_cache mysite_cache;
    proxy_cache_valid 200 301 302 10m;
    proxy_cache_valid 404 1m;
    proxy_cache_lock on;
    proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    proxy_cache_bypass $http_pragma;
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

Test aur reload:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### Step 3: Firewall

```bash
sudo ufw status
sudo ufw allow 443/tcp
```

(Is setup mein 443 pehle se khula tha.)

### Step 4: Hosts file (LAN ke liye zaroori)

Public DNS abhi purane IP par hai, is liye har client par domain ko `192.168.0.18` se jodna padta hai.

**Ubuntu** (`/etc/hosts`):

```
192.168.0.18 hamza-devops.work.gd
```

**Windows** - Notepad ko **Run as administrator** kholein, **File > Open**, file type **All Files (\*.\*)**, path:

```
C:\Windows\System32\drivers\etc\hosts
```

Sabse neeche nayi line par (aage `#` na ho):

```
192.168.0.18 hamza-devops.work.gd
```

Save (Ctrl+S), phir CMD mein:

```
ipconfig /flushdns
ping hamza-devops.work.gd
```

Jawab `192.168.0.18` se aana chahiye.

---

## 4. Verification (Test)

**Ubuntu par:**

```bash
curl -sI https://hamza-devops.work.gd | head -5
curl -vI https://hamza-devops.work.gd 2>&1 | grep -iE "subject|issuer|expire|SSL certificate verify"
curl -sI http://hamza-devops.work.gd | head -3
```

Dekhna hai:

- `HTTP/1.1 200 OK` (HTTPS par)
- `issuer: ... Let's Encrypt ...`
- `SSL certificate verify ok`
- HTTP par `301 Moved Permanently` aur `Location: https://hamza-devops.work.gd/`

**Windows par:** Firefox mein `https://hamza-devops.work.gd` kholein, lock nazar aana chahiye.

---

## 5. Jo Masle aaye aur un ke Hal

| # | Masla | Wajah | Hal |
|---|-------|-------|-----|
| 1 | CMD mein `192.168.0.18 hamza-devops.work.gd` likhne par `is not recognized as an internal or external command` | Yeh line command nahi hai, hosts file ke **andar** likhi jati hai | Notepad (admin) se hosts file mein likhein, ya admin CMD mein `echo 192.168.0.18 hamza-devops.work.gd>> C:\Windows\System32\drivers\etc\hosts` |
| 2 | `curl http://hamza-devops.work.gd` par `200 OK` aaya, `301` nahi | Test reload ke foran baad chal gaya tha | Dobara chalane par `301` aaya. Seedha test: `curl -sI -H "Host: hamza-devops.work.gd" http://127.0.0.1` |
| 3 | `nginx -t` OK, lekin yaqeen nahi ke purani file active hai ya nahi | Duplicate `listen 80` ka shak | `sudo nginx -T 2>/dev/null \| grep -nE "listen\|server_name\|configuration file"` se dekha, sirf `mywebsite.conf` mein `listen 80` tha |
| 4 | Firefox: **The connection has timed out** | Windows par domain purane public IP `154.208.38.229` par resolve ho raha tha (hosts entry lagi hi nahi thi) | Hosts file mein entry daali, `ipconfig /flushdns` chalaya, `ping` ne `192.168.0.18` dikhaya |
| 5 | Notepad mein **Save As** window aa gayi | Nayi khali (Untitled) file save ho rahi thi, hosts file khuli hi nahi thi | **Don't Save**, phir admin Notepad mein **File > Open** se hosts file kholi |
| 6 | Hosts file Open window mein nazar nahi aa rahi thi | File type `Text Documents (*.txt)` tha | **All Files (\*.\*)** chuna |
| 7 | `curl` output mein `Expires: Tue, 24 Nov 2026 ...` dekh kar certificate expiry ka shak | Yeh web page ka cache header hai (`expires 1h`), certificate ki expiry nahi | Certificate ki asli expiry `expire date: Feb 22 18:38:55 2027 GMT` hai |
| 8 | DNS ka TXT record `_acme-challenge` abhi bhi maujood | Certificate banne ke baad record hatana zaroori nahi | Chahein to baad mein DNSExit se delete karein |

---

## 6. Agar phir bhi Firefox mein timeout aaye - Checklist

1. `ping hamza-devops.work.gd` mein `192.168.0.18` aana chahiye. Warna hosts entry check karein.
2. Hosts line ke aage `#` na ho, aur file ka naam sirf `hosts` ho (`hosts.txt` nahi).
3. Windows mein check:
   ```
   powershell -command "Test-NetConnection 192.168.0.18 -Port 443"
   ```
   `TcpTestSucceeded : True` aana chahiye.
4. Agar VM hai to network adapter **Bridged** ho (NAT nahi).
5. Firefox: Settings > Privacy & Security > **DNS over HTTPS** off karein.
6. Firefox mein proxy/VPN/security extension band karke Private Window mein try karein.
7. Firefox poora band karke dobara kholein.

---

## 7. Maintenance - Yaad rakhne ki baatein

- **Certificate 90 din ka hai.** Is certificate ki expiry **22 Feb 2027** hai. Manual DNS challenge mein auto-renew nahi hota, is liye us se pehle dobara certbot chalana hoga (naya TXT value banega, purana kaam nahi aayega):
  ```bash
  sudo certbot certonly --manual --preferred-challenges dns -d hamza-devops.work.gd
  sudo nginx -t && sudo systemctl reload nginx
  ```
- Jab public DNS (`hamza-devops.work.gd`) khud `192.168.0.18` par theek ho jaye, to Ubuntu ki `/etc/hosts` aur Windows ki hosts file se yeh line **hata dein**.
- Site sirf LAN (ghar/office network) mein khulegi, kyunke `192.168.0.18` private IP hai.
- `http://192.168.0.18` bhi domain par redirect hota hai. IP par seedha HTTPS kholne se certificate warning aayegi, kyunke certificate domain ke naam par hai, IP par nahi.
- Server-side cache ko test ke liye bypass karna ho to **Ctrl+Shift+R** (config mein `proxy_cache_bypass $http_pragma` hai).

---

## 8. Quick Command Cheat Sheet

```bash
# Nginx config check + reload
sudo nginx -t && sudo systemctl reload nginx

# Saari active config dekhna
sudo nginx -T | less

# Certificate ki tafseel
curl -vI https://hamza-devops.work.gd 2>&1 | grep -iE "subject|issuer|expire|SSL certificate verify"

# Nginx logs
sudo tail -f /var/log/nginx/error.log

# Firewall
sudo ufw status
```

```
:: Windows (CMD)
ipconfig /flushdns
ping hamza-devops.work.gd
type C:\Windows\System32\drivers\etc\hosts
```
