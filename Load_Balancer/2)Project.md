# Nginx Load Balancer Guide (Ubuntu + RHEL)

Is guide mein do servers par ek static website host karke Nginx ke zariye **round robin load balancing** setup ki gayi hai.

## Setup ka khaka

| Machine | OS | IP | Kaam |
|---|---|---|---|
| Server 1 | Ubuntu | `192.168.0.18` | Load balancer + website (internal port 8090) |
| Server 2 | RHEL | `192.168.0.13` | Website (port 80) |
| Client | Windows | Same network | Firefox se test |

```
                    +----------------------------+
 Firefox  --->      |  Ubuntu 192.168.0.18:80    |
 (Windows)          |  Nginx Load Balancer       |
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

## Zaroori baat: port ka intikhab

Ubuntu par apni website ke liye internal port `8090` use kiya gaya hai. `8080` isliye nahi liya kyunke us par aam taur par **Jenkins** chalta hai, aur `8081` is liye nahi ke `/etc/nginx/nginx.conf` mein pehle se maujood tha.

Naya port lene se pehle hamesha check karein ke free hai:

```bash
sudo ss -tlnp | grep 8090
```

Kuch output na aaye to port free hai.

---

## Step 1: Ubuntu par Nginx config

### 1.1 Nginx install (agar nahi hai)

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable --now nginx
```

### 1.2 Config file banayein

```bash
sudo nano /etc/nginx/conf.d/mywebsite.conf
```

Pura content yeh likhein:

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

### 1.3 Check aur reload

```bash
sudo nginx -t
sudo systemctl reload nginx
```

`nginx -t` mein "syntax is ok" aur "test is successful" aana chahiye.

> **Note:** Agar Ubuntu par default site (`sites-enabled/default`) port 80 par chal rahi ho to woh load balancer se takra sakti hai. Zaroorat ho to usay hata dein: `sudo rm /etc/nginx/sites-enabled/default`

---

## Step 2: RHEL par Nginx aur SSH ready karein

**Yeh saari commands RHEL machine par chalani hain.**

```bash
sudo dnf install nginx openssh-server -y
sudo systemctl enable --now nginx sshd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

Agar baad mein `restorecon` "command not found" de to:

```bash
sudo dnf install policycoreutils -y
```

---

## Step 3: Website files RHEL par bhejein

### 3.1 Ubuntu se files bhejein

**Yeh command Ubuntu par chalani hai.** `RHEL_USERNAME` ki jagah RHEL ka apna username likhein:

```bash
scp -r /var/www/mywebsite/* RHEL_USERNAME@192.168.0.13:/tmp/
```

Files `/tmp/` mein isliye bheji jati hain kyunke normal user ko seedha `/usr/share/nginx/html/` mein likhne ki ijazat nahi hoti.

### 3.2 RHEL par files Nginx folder mein rakhein

**Yeh commands RHEL par chalani hain** (Ubuntu par nahi, warna "not a directory" aur "restorecon: command not found" jaise errors aayenge):

```bash
sudo cp -r /tmp/index.html /tmp/style.css /usr/share/nginx/html/
sudo restorecon -Rv /usr/share/nginx/html/
```

Agar website mein images, js ya doosre folders bhi hain to pehle `ls /tmp/` se dekh lein, phir:

```bash
sudo cp -r /tmp/* /usr/share/nginx/html/
sudo restorecon -Rv /usr/share/nginx/html/
```

`restorecon` SELinux ka context theek karta hai, warna RHEL par **403 Forbidden** aa sakta hai.

### 3.3 Ubuntu se ek hi command mein (optional)

```bash
scp -r /var/www/mywebsite/* RHEL_USERNAME@192.168.0.13:/tmp/ && \
ssh RHEL_USERNAME@192.168.0.13 "sudo cp -r /tmp/index.html /tmp/style.css /usr/share/nginx/html/ && sudo restorecon -Rv /usr/share/nginx/html/"
```

Dono jagah **same IP** (`192.168.0.13`) hona zaroori hai.

---

## Step 4: Test (Ubuntu par)

**Teeno commands Ubuntu par chalani hain:**

```bash
curl http://192.168.0.13                 # RHEL seedha
curl http://127.0.0.1:8090               # Ubuntu ki internal website
curl -s http://192.168.0.18 | head -5    # Load balancer ke zariye
```

| Command | Kya check karti hai |
|---|---|
| `curl http://192.168.0.13` | RHEL ki website theek chal rahi hai |
| `curl http://127.0.0.1:8090` | Ubuntu ki internal website theek hai |
| `curl -s http://192.168.0.18 \| head -5` | Load balancer kaam kar raha hai |

Pehli do commands fail hon to load balancer bhi sahi kaam nahi karega, isliye pehle unhein theek karein.

### Round robin ka saboot

```bash
for i in 1 2 3 4 5 6; do curl -sI http://192.168.0.18 | grep -i x-served-by || echo "RHEL"; done
```

Expected output:

```
RHEL
X-Served-By: Ubuntu
RHEL
X-Served-By: Ubuntu
RHEL
X-Served-By: Ubuntu
```

`X-Served-By: Ubuntu` aaye to Ubuntu ne jawab diya, header na aaye to RHEL ne.

---

## Step 5: Windows ke Firefox se test

Windows machine Ubuntu ke same network par honi chahiye. Firefox mein kholein:

```
http://192.168.0.18
```

Dono servers par same website hai, is liye farq nazar nahi aayega. Do tareeqe hain:

### Tareeqa A: Developer Tools

1. `F12` dabayein aur **Network** tab kholein.
2. `Ctrl+Shift+R` se hard refresh karein (cache se na aaye).
3. Sab se upar wali request (`192.168.0.18`) par click karein.
4. **Headers > Response Headers** dekhein.
5. `X-Served-By: Ubuntu` hai to Ubuntu ne jawab diya, nahi hai to RHEL ne.

5-6 baar refresh karein, header kabhi aayega kabhi nahi.

### Tareeqa B: Footer mein fark likhna

Test ke liye dono servers ke `index.html` ke footer mein farq daal dein:

- Ubuntu: `<p>&copy; 2026 MyBrand. (Ubuntu)</p>`
- RHEL: `<p>&copy; 2026 MyBrand. (RHEL)</p>`

Phir `Ctrl+Shift+R` karte jayein, footer ka naam badalta nazar aayega. Test ke baad wapas theek kar dein.

### Agar page na khule

Ubuntu ka firewall port 80 rok raha ho sakta hai:

```bash
sudo ufw allow 80/tcp
```

---

## Website update karne ka tareeqa

Jab bhi website mein tabdeeli karein, **dono servers par files update karni hongi**, warna users ko kabhi purana kabhi naya page milega.

1. Ubuntu par `/var/www/mywebsite/` mein file badlein.
2. RHEL par bhi bhejein:

```bash
scp -r /var/www/mywebsite/* RHEL_USERNAME@192.168.0.13:/tmp/ && \
ssh RHEL_USERNAME@192.168.0.13 "sudo cp -r /tmp/index.html /tmp/style.css /usr/share/nginx/html/ && sudo restorecon -Rv /usr/share/nginx/html/"
```

### `nginx -t` aur reload kab chahiye?

| Kya badla | `nginx -t` + reload chahiye? |
|---|---|
| Sirf HTML/CSS/images (website files) | Nahi |
| Nginx config (`nginx.conf`, `conf.d/*.conf`) | Haan |

Nginx website files har request par disk se padhta hai, is liye nayi files foran nazar aati hain. Config badli ho to:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## Failover test (optional)

Dekhein ke ek server band ho to kya hota hai.

Haan, bilkul, Ubuntu ki IP **192.168.0.18** hi hai, aur config mein bhi wahi use hui hai. Koi cheez badalne ki zarorat nahi.

Is setup mein IPs ka role yeh hai:

| Machine | IP | Kahan use hui |
|---|---|---|
| Ubuntu (load balancer) | 192.168.0.18 | `server_name 192.168.0.18;` aur browser/curl mein |
| RHEL (backend) | 192.168.0.16 | `upstream` ke andar `server 192.168.0.16:80;` |
| Ubuntu ki apni website | 127.0.0.1:8080 | `upstream` ke andar, sirf internal |

Yahan ek cheez confuse kar sakti hai: upstream mein Ubuntu ke liye `192.168.0.18` ki jagah `127.0.0.1:8080` isliye likha hai ke woh Ubuntu ki **apni internal website** hai jo sirf load balancer ko dikhti hai. Yeh Ubuntu ki IP ko ghalat nahi likha gaya, balke ek alag internal port hai taake port 80 load balancer ke paas rahe aur dono mein takrao na ho.

Agar chahein to `127.0.0.1:8080` ki jagah `192.168.0.18:8080` bhi likh sakte hain (dono server block mein `listen 192.168.0.18:8080;`), lekin us surat mein port 8080 network par bhi khul jayega. `127.0.0.1` zyada safe hai.

Browser mein aap hamesha **http://192.168.0.18** kholein, wahi load balancer hai aur wahin se request bari bari Ubuntu aur RHEL ko jayegi.

1. RHEL par: `sudo systemctl stop nginx`
2. Ubuntu par 6 baar `curl -sI http://192.168.0.18` chalayein. Sab jawab Ubuntu se aane chahiye, website chalti rahegi.
3. Test ke baad RHEL par: `sudo systemctl start nginx`

Nginx kisi backend ke fail hone par usay kuch der ke liye skip kar deta hai (`max_fails` aur `fail_timeout` ke hisaab se).

---

## Masail aur unke hal (Troubleshooting)

| Masla | Wajah | Hal |
|---|---|---|
| `cp: target '/usr/share/nginx/html/' is not a directory` | Command Ubuntu par chali, RHEL par nahi | Yeh command RHEL par chalayein |
| `sudo: restorecon: command not found` | Ubuntu par SELinux nahi hota, ya RHEL par package nahi | RHEL par chalayein, ya `sudo dnf install policycoreutils` |
| `curl http://127.0.0.1:8080` par Jenkins ka login page | Port 8080 par Jenkins chal raha hai | Website ke liye alag port (`8090`) use karein |
| Port 8081 par `ss` mein nginx dikhe | `nginx.conf` line 36 mein pehle se `listen 8081;` hai | `sudo grep -rn "8081" /etc/nginx/` se dhoondein, ya doosra port chunein |
| RHEL par **403 Forbidden** | SELinux context ghalat | `sudo restorecon -Rv /usr/share/nginx/html/` |
| RHEL par page nahi khulta | Firewall port 80 band | `sudo firewall-cmd --permanent --add-service=http && sudo firewall-cmd --reload` |
| `502 Bad Gateway` | Koi backend band hai ya port ghalat | Pehle dono backend ko alag alag `curl` karke check karein |
| Kabhi purana kabhi naya page | Dono servers par files same nahi | Dono par files update karein |
| Windows se page nahi khulta | Ubuntu firewall | `sudo ufw allow 80/tcp` |
| `scp` mein `Connection refused` | RHEL par `sshd` band | `sudo systemctl enable --now sshd` |

### Kaam ki debugging commands

```bash
sudo ss -tlnp | grep nginx                      # nginx kin ports par sun raha hai
sudo grep -rn "listen" /etc/nginx/              # saari listen lines
sudo tail -f /var/log/nginx/error.log           # live error log
sudo nginx -T | less                            # poori active config
```

---

## Production se pehle (aakhri safai)

- `add_header X-Served-By "Ubuntu";` sirf test ke liye tha. Chahein to delete kar ke `sudo systemctl reload nginx` kar dein.
- Footer mein jo `(Ubuntu)` / `(RHEL)` likha tha, wapas hata dein.
- Production mein HTTPS (Let's Encrypt), health checks aur weighted balancing par bhi ghaur karein.

## Agle qadam (optional)

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


RHEL par website ki file yahan hai:

```
/usr/share/nginx/html/index.html
```

**RHEL par chalayein:**

```bash
sudo nano /usr/share/nginx/html/index.html
```

Neeche scroll karke footer wali line dhoondein:

```html
<p>&copy; 2026 MyBrand. All rights reserved.</p>
```

Usay badal kar yeh likhein:

```html
<p>&copy; 2026 MyBrand. (RHEL)</p>
```

Save karein: `Ctrl+O`, `Enter`, phir `Ctrl+X`.

Ya nano ke bajaye ek command se:

```bash
sudo sed -i 's|All rights reserved.|(RHEL)|' /usr/share/nginx/html/index.html
```

`nginx -t` ya reload ki zaroorat nahi, kyunke sirf HTML file badli hai.

**Ubuntu par** bhi isi tarah, lekin file `/var/www/mywebsite/index.html` hai:

```bash
sudo sed -i 's|All rights reserved.|(Ubuntu)|' /var/www/mywebsite/index.html
```

Phir Firefox mein `http://192.168.0.18` kholein aur `Ctrl+Shift+R` karte jayein. Footer mein bari bari (Ubuntu) aur (RHEL) nazar aayega.

**Test ke baad wapas theek karne ke liye:**

```bash
# RHEL par
sudo sed -i 's|(RHEL)|All rights reserved.|' /usr/share/nginx/html/index.html

# Ubuntu par
sudo sed -i 's|(Ubuntu)|All rights reserved.|' /var/www/mywebsite/index.html
```

Ek ehtiyat: RHEL par `sed` se file badalne ke baad bhi SELinux context theek rehta hai, lekin agar kabhi 403 aaye to `sudo restorecon -Rv /usr/share/nginx/html/` chala dein.


