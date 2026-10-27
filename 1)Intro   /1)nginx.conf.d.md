## Seedha jawab

- **`conf.d/*.conf`**: roz ke kaam mein **baar baar** badalti hai.
- **`nginx.conf`**: **kabhi kabhi**, bohat kam.

## `conf.d/*.conf` mein kab changes hoti hain

Yeh woh file hai jisme har website ki apni settings hoti hain. In cases mein badlein:

| Kaam | Kya badlenge |
|---|---|
| Nayi website add karni ho | Nayi `.conf` file banayein |
| Port badalna ho (80 se 8082) | `listen 8082;` |
| Domain ya IP badalna ho | `server_name` |
| Website ka folder badalna ho | `root /var/www/naya-folder;` |
| HTTPS (SSL) lagana ho | `listen 443 ssl;` aur certificate ke path |
| Kisi app par forward karna ho (proxy) | `location / { proxy_pass http://127.0.0.1:5000; }` |
| Load balancing karni ho | `upstream { }` block |
| Redirect lagana ho (http se https) | `return 301 https://...;` |
| 404 page ya cache rules badalne hon | `error_page`, `expires` |

Har change ke baad:
```bash
nginx -t
systemctl reload nginx
```

## `nginx.conf` mein kab changes hoti hain

Yeh Nginx ki **global** settings ki file hai. Aksar ek baar set karke chhod dete hain. In cases mein badlein:

| Kaam | Kya badlenge |
|---|---|
| `conf.d` ki files load karni hon | `include /etc/nginx/conf.d/*.conf;` (hamare saath yahi hua) |
| Nginx kis user se chalega | `user www-data;` |
| Worker processes / connections | `worker_processes`, `worker_connections` |
| Upload ki limit badhani ho | `client_max_body_size 50M;` |
| Gzip compression on karni ho | `gzip on;` |
| Global logging badalni ho | `access_log`, `error_log` |
| Timeouts badalne hon | `keepalive_timeout`, `proxy_read_timeout` |
| Security headers sab sites ke liye | `server_tokens off;` |

## Asaan rule

- **Ek website ka masla** (port, domain, folder, SSL): `conf.d` ki us site ki file badlein.
- **Poore Nginx ka masla** (sab sites par asar): `nginx.conf` badlein.

## Hamare project mein kab kya badla

| Waqt | File | Kya badla |
|---|---|---|
| Shuru mein | `conf.d/mywebsite.conf` | Nayi file banayi |
| Masla milne par | `nginx.conf` | `include conf.d` line add ki (sirf 1 baar) |
| Port test karte waqt | `conf.d/mywebsite.conf` | `listen 80` se `8082`, phir wapas `80` |

Yani `nginx.conf` mein humne sirf **ek baar** change kiya, aur `conf.d/mywebsite.conf` mein **bar bar**. Aage bhi aisa hi hoga.

## Zaroori baat

`nginx.conf` mein changes ke baad `nginx -t` zaroor chalayein, kyunki is mein galti ho to **poora Nginx** band ho sakta hai, sirf ek site nahi. Is liye is file ko chhedne se pehle backup le lein:
```bash
cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
```
