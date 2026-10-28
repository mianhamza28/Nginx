# Nginx Load Balancing ke Scenarios (Examples ke saath)

## 1. Basic Web App (Round Robin)
**Scenario:** 3 identical servers, traffic barabar baantna hai.

```nginx
upstream web_app {
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
    server 10.0.0.3:3000;
}

server {
    listen 80;
    location / {
        proxy_pass http://web_app;
    }
}
```

## 2. Different Capacity wale Servers (Weighted)
**Scenario:** Ek server powerful hai (16 GB RAM), baqi chhote hain.

```nginx
upstream web_app {
    server 10.0.0.1:3000 weight=5;   # powerful
    server 10.0.0.2:3000 weight=2;
    server 10.0.0.3:3000 weight=1;
}
```

## 3. Lambi Requests / Heavy Processing (Least Connections)
**Scenario:** File upload, report generation, ya video processing, jahan kuch requests bohat der chalti hain.

```nginx
upstream web_app {
    least_conn;
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
}
```

## 4. Login Session Persistence (Sticky Sessions)
**Scenario:** User ka session ek server par stored hai, to usay hamesha usi server par jana chahiye.

**Option A: ip_hash**
```nginx
upstream web_app {
    ip_hash;
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
}
```

**Option B: Cookie ke basis par (open source mein)**
```nginx
upstream web_app {
    hash $cookie_sessionid consistent;
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
}
```

> Behtar approach yeh hai ke sessions Redis mein rakhein, phir sticky ki zaroorat hi nahi parti.

## 5. Failover aur Backup Server
**Scenario:** Main servers down hon to standby server kaam sambhal le.

```nginx
upstream web_app {
    server 10.0.0.1:3000 max_fails=3 fail_timeout=30s;
    server 10.0.0.2:3000 max_fails=3 fail_timeout=30s;
    server 10.0.0.9:3000 backup;
}

server {
    listen 80;
    location / {
        proxy_pass http://web_app;
        proxy_next_upstream error timeout http_500 http_502 http_503;
        proxy_connect_timeout 3s;
    }
}
```

`proxy_next_upstream` ka matlab: ek server error de to request khud dusre server par chali jaye.

## 6. HTTPS (SSL Termination)
**Scenario:** SSL sirf Nginx par lagani hai, backend servers plain HTTP par chalein.

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;   # HTTP se HTTPS redirect
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/ssl/example.com.crt;
    ssl_certificate_key /etc/ssl/example.com.key;

    location / {
        proxy_pass http://web_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

## 7. Path-Based Routing (Microservices)
**Scenario:** `/api` ek group of servers par, `/admin` doosre par, aur baqi frontend par.

```nginx
upstream frontend { server 10.0.1.1:3000; server 10.0.1.2:3000; }
upstream api      { server 10.0.2.1:8000; server 10.0.2.2:8000; }
upstream admin    { server 10.0.3.1:9000; }

server {
    listen 80;

    location /api/   { proxy_pass http://api; }
    location /admin/ { proxy_pass http://admin; }
    location /       { proxy_pass http://frontend; }
}
```

## 8. Domain-Based Routing (Multiple Websites)
**Scenario:** Ek Nginx par kai domains, har ek ke apne backend.

```nginx
server {
    listen 80;
    server_name shop.example.com;
    location / { proxy_pass http://shop_servers; }
}

server {
    listen 80;
    server_name blog.example.com;
    location / { proxy_pass http://blog_servers; }
}
```

## 9. WebSocket (Chat / Live Apps)
**Scenario:** Real-time chat ya notifications.

```nginx
upstream chat {
    ip_hash;
    server 10.0.0.1:4000;
    server 10.0.0.2:4000;
}

server {
    listen 80;
    location /ws/ {
        proxy_pass http://chat;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 3600s;
    }
}
```

## 10. Caching ke saath Load Balancing
**Scenario:** Static ya baar baar aane wala content cache karke backend ka load kam karna.

```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=mycache:10m max_size=1g inactive=60m;

server {
    listen 80;
    location / {
        proxy_cache mycache;
        proxy_cache_valid 200 10m;
        proxy_cache_use_stale error timeout updating http_500 http_502;
        proxy_pass http://web_app;
    }
}
```

`proxy_cache_use_stale` ki wajah se backend down hone par bhi purana cached content serve hota rehta hai.

## 11. Rate Limiting (DDoS / Abuse se bachao)
**Scenario:** Ek IP se zyada requests rokni hain.

```nginx
http {
    limit_req_zone $binary_remote_addr zone=perip:10m rate=10r/s;

    server {
        location /api/ {
            limit_req zone=perip burst=20 nodelay;
            proxy_pass http://api;
        }
    }
}
```

## 12. TCP/UDP Load Balancing (Database, DNS)
**Scenario:** HTTP nahi balki MySQL ya DNS traffic balance karna. Iske liye `stream` block use hota hai.

```nginx
stream {
    upstream mysql_read {
        least_conn;
        server 10.0.5.1:3306;
        server 10.0.5.2:3306;
    }

    server {
        listen 3306;
        proxy_pass mysql_read;
    }
}

# DNS (UDP) ke liye
stream {
    upstream dns_servers {
        server 10.0.6.1:53;
        server 10.0.6.2:53;
    }
    server {
        listen 53 udp;
        proxy_pass dns_servers;
    }
}
```

## 13. Canary Deployment (Naya Version Test Karna)
**Scenario:** Naye version par sirf 10% traffic bhejna.

```nginx
upstream stable { server 10.0.0.1:3000; server 10.0.0.2:3000; }
upstream canary { server 10.0.0.9:3000; }

split_clients "${remote_addr}" $backend {
    10%     canary;
    *       stable;
}

server {
    listen 80;
    location / {
        proxy_pass http://$backend;
    }
}
```

## 14. Blue-Green / Zero Downtime Deployment
**Scenario:** Server par update karni hai bina users ko affect kiye.

1. Server ko `down` mark karein:
```nginx
server 10.0.0.1:3000 down;
```
2. `sudo nginx -t && sudo systemctl reload nginx`
3. Us server par deploy karein.
4. `down` hata dein aur dobara reload karein.
5. Baqi servers par bhi yehi dohrayein.

Reload graceful hota hai, yani chalti hui requests nahi tootti.

## 15. Keepalive (Performance Boost)
**Scenario:** Backend se baar baar naya connection banane ka overhead kam karna.

```nginx
upstream web_app {
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
    keepalive 32;
}

server {
    location / {
        proxy_pass http://web_app;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

## 16. Docker Compose Example
**Scenario:** Local ya small deployment mein 3 app containers ke saamne Nginx.

`docker-compose.yml`
```yaml
services:
  app:
    image: myapp:latest
    deploy:
      replicas: 3

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app
```

`nginx.conf`
```nginx
events {}
http {
    upstream app_servers {
        server app:3000;   # Docker DNS khud saare replicas mein rotate karta hai
    }
    server {
        listen 80;
        location / {
            proxy_pass http://app_servers;
        }
    }
}
```

## Quick Summary Table

| Scenario | Kya use karein |
|---|---|
| Identical servers | Round robin (default) |
| Alag capacity | `weight` |
| Lambi requests | `least_conn` |
| Session same server par | `ip_hash` ya `hash $cookie_...` |
| Server down hone par bachao | `backup`, `max_fails`, `proxy_next_upstream` |
| HTTPS | SSL termination |
| Microservices | `location` based routing |
| Real-time apps | WebSocket headers |
| DB / DNS | `stream` block |
| Safe release | `split_clients` (canary) |

Agar aap bata dein ke aapka asli use case kya hai (Node.js, Django, Laravel, kitne servers, kahan host hai), to main usi ke hisab se ek ready-to-use config bana dunga.
