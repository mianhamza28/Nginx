# Nginx Rate Limiting

Rate limiting ka matlab hai ke ek client (usually IP) ek certain time mein kitni requests bhej sakta hai, usay limit karna. Is se brute-force attacks, scraping, DDoS aur API abuse se bachao hota hai, aur backend servers overload nahi hote.

## Kaise kaam karta hai (Leaky Bucket)

Nginx **leaky bucket** algorithm use karta hai. Socho ek bucket hai jisme requests aati hain aur ek fixed speed se bahar nikalti hain (process hoti hain). Agar bucket bhar jaye to nayi requests reject ho jati hain.

## Do directives zaroori hain

**1. `limit_req_zone`** (http block mein): limit define karta hai.
**2. `limit_req`** (server ya location mein): limit apply karta hai.

## Basic Example

```nginx
http {
    limit_req_zone $binary_remote_addr zone=perip:10m rate=10r/s;

    server {
        listen 80;

        location / {
            limit_req zone=perip;
            proxy_pass http://web_app;
        }
    }
}
```

**Samajhein:**
- `$binary_remote_addr`: client ka IP (key). Binary form memory bachata hai.
- `zone=perip:10m`: zone ka naam `perip`, aur 10 MB memory. 1 MB mein lagbhag 16,000 IPs store hote hain.
- `rate=10r/s`: 10 requests per second. Yeh effectively 1 request har 100ms ke barabar hai.

> Agar `rate=10r/s` hai aur ek saath 2 requests aa jayen, to doosri reject ho jayegi, kyunke Nginx milliseconds mein hisab lagata hai. Isi liye `burst` use hota hai.

## Burst

```nginx
location / {
    limit_req zone=perip burst=20;
    proxy_pass http://web_app;
}
```

`burst=20` ka matlab: rate se zyada aane wali 20 requests tak queue mein rakhi jayengi aur rate ke hisab se process hongi. 20 se zyada aayen to reject (503).

## Burst + nodelay

```nginx
location / {
    limit_req zone=perip burst=20 nodelay;
    proxy_pass http://web_app;
}
```

`nodelay` mein burst wali requests ko **foran** process kar diya jata hai (queue mein intezar nahi), lekin burst slots rate ke hisab se dobara khali hote hain. Is se user ko slow response nahi milta.

## Burst + delay (Nginx 1.15.7+)

```nginx
limit_req zone=perip burst=20 delay=8;
```

Pehli 8 requests foran, 9 se 20 tak delay ke saath, uske baad reject. `nodelay` aur queue ke beech ka balance.

## Scenarios

### 1. Login Page (Brute-force se bachao)
```nginx
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

location /login {
    limit_req zone=login burst=3 nodelay;
    proxy_pass http://web_app;
}
```
Sirf 5 requests per minute. Password guess karna bohat mushkil ho jata hai.

### 2. API Rate Limit
```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;

location /api/ {
    limit_req zone=api burst=50 nodelay;
    proxy_pass http://api_servers;
}
```

### 3. Multiple Limits (Alag alag locations par)
```nginx
http {
    limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login:10m   rate=5r/m;
    limit_req_zone $binary_remote_addr zone=api:10m     rate=50r/s;

    server {
        location /      { limit_req zone=general burst=20 nodelay; proxy_pass http://web_app; }
        location /login { limit_req zone=login burst=3 nodelay;    proxy_pass http://web_app; }
        location /api/  { limit_req zone=api burst=100 nodelay;    proxy_pass http://api; }
    }
}
```

### 4. Aik location par kai limits ek saath
```nginx
location /api/ {
    limit_req zone=perip burst=20 nodelay;
    limit_req zone=perserver burst=1000 nodelay;
    proxy_pass http://api;
}
```
Dono limits apply hoti hain, jo zyada strict ho wo lagti hai.

### 5. API Key ya User ke hisab se limit
```nginx
limit_req_zone $http_x_api_key zone=apikey:10m rate=100r/s;

location /api/ {
    limit_req zone=apikey burst=200 nodelay;
    proxy_pass http://api;
}
```
IP ki jagah header ya cookie ko key bana sakte hain.

### 6. Trusted IPs ko exempt karna (Whitelist)
```nginx
geo $limited {
    default         1;
    10.0.0.0/8      0;      # internal network
    203.0.113.5     0;      # office IP
}

map $limited $limit_key {
    0   "";
    1   $binary_remote_addr;
}

limit_req_zone $limit_key zone=perip:10m rate=10r/s;
```
Jab key khali ("") ho to us request par limit lagti hi nahi.

## Error Code Customize Karna

Default mein limit cross hone par **503** milta hai. Behtar hai **429 (Too Many Requests)** dena:

```nginx
location /api/ {
    limit_req zone=api burst=50 nodelay;
    limit_req_status 429;
    proxy_pass http://api;
}
```

Custom message ke saath:
```nginx
limit_req_status 429;
error_page 429 /429.json;

location = /429.json {
    internal;
    default_type application/json;
    return 429 '{"error":"Too many requests, baad mein try karein"}';
}
```

## Logging aur Testing

Rejected requests ka log level set karein:
```nginx
limit_req_log_level warn;
```

Dry run mode (sirf log, block nahi karta), naya limit test karne ke liye:
```nginx
limit_req zone=perip burst=20 nodelay;
limit_req_dry_run on;
```

Test karne ke liye:
```bash
# 30 requests jaldi bhejein
for i in {1..30}; do curl -s -o /dev/null -w "%{http_code}\n" http://example.com/; done
```
Shuru mein 200 aur phir 503 (ya 429) aana chahiye.

## Connection Limiting (Alag cheez)

Rate limiting requests per second hai. Concurrent connections limit karne ke liye `limit_conn` hota hai:

```nginx
limit_conn_zone $binary_remote_addr zone=conn_perip:10m;

server {
    location /download/ {
        limit_conn conn_perip 5;     # ek IP se max 5 simultaneous connections
        limit_rate 500k;             # per connection speed limit
    }
}
```

## Zaroori Baatein

1. **Load balancer ke peeche ho to real IP ka dhyan rakhein.** Agar Nginx ke aage Cloudflare ya koi aur proxy hai, to `$binary_remote_addr` unka IP dikhayega aur sab users ek hi IP lagenge. Is ke liye `realip` module use karein:
```nginx
set_real_ip_from 103.21.244.0/22;   # proxy ka IP range
real_ip_header X-Forwarded-For;
```

2. **Zone memory bhar jaye** to Nginx purani entries hata deta hai, aur agar phir bhi jagah na mile to 503 deta hai. 10m bohat se sites ke liye kaafi hai.

3. **Shared zone:** Ek zone ko multiple locations mein use kar sakte hain, tab un sab ka mila kar ek hi limit count hota hai.

4. **Yeh sirf ek Nginx instance ke liye hai.** Agar kai Nginx servers hain, to har ek ka counter alag hota hai.

## Quick Reference

| Directive | Kaam |
|---|---|
| `limit_req_zone` | Key, memory aur rate define karta hai |
| `limit_req` | Zone ko location par apply karta hai |
| `burst` | Extra requests ki queue |
| `nodelay` | Burst requests ko foran process karna |
| `delay` | Pehli N requests foran, baqi delay |
| `limit_req_status` | Reject par status code (429) |
| `limit_req_dry_run` | Sirf log, block nahi |
| `limit_conn` | Concurrent connections ki limit |

Agar aap chahein to main login page, public API, ya Cloudflare ke peeche wale setup ke liye ek complete production-ready config bana dun.
