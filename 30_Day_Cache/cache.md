# 30-Day Caching for Static Content in Nginx

There are two kinds of caching, and you can use both together:

1. **Browser caching:** the browser stores files locally for 30 days and doesn't even contact your server.
2. **Nginx proxy caching:** Nginx stores backend responses on its own disk, so your backend servers get fewer requests.

## 1. Browser Caching (most common)

**Static files served directly by Nginx:**

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/html;

    location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|webp|avif|woff|woff2|ttf|eot|pdf)$ {
        expires 30d;
        add_header Cache-Control "public";
        access_log off;
    }
}
```

- `expires 30d;` sets both the `Expires` header and `Cache-Control: max-age=2592000` (30 days in seconds).
- `public` allows browsers and CDNs to cache the file.
- `access_log off;` reduces log noise for static files.

**Static pages (HTML):**

```nginx
location ~* \.(html|htm)$ {
    expires 30d;
    add_header Cache-Control "public";
}
```

Caching HTML for 30 days is risky. If you edit a page, visitors with the old copy won't see the update until it expires. Most sites cache assets for 30 days (or even 1 year) and keep HTML short, such as `expires 5m;` or `no-cache`.

## 2. Caching via a Backend / Load Balancer

If Nginx proxies to backend servers, you can cache their responses on the Nginx machine:

```nginx
http {
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=static_cache:50m
                     max_size=2g inactive=30d use_temp_path=off;

    upstream web_app {
        server 10.0.0.1:3000;
        server 10.0.0.2:3000;
    }

    server {
        listen 80;
        server_name example.com;

        location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|webp|woff|woff2)$ {
            proxy_cache static_cache;
            proxy_cache_valid 200 301 302 30d;
            proxy_cache_valid 404 1m;
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503;
            proxy_cache_lock on;

            # Also tell browsers to cache for 30 days
            expires 30d;
            add_header Cache-Control "public";

            # Shows HIT / MISS in response headers
            add_header X-Cache-Status $upstream_cache_status;

            proxy_pass http://web_app;
        }

        location / {
            proxy_pass http://web_app;
        }
    }
}
```

- `proxy_cache_path`: the cache directory, shared memory zone (`keys_zone`), maximum size, and how long unused items stay (`inactive=30d`).
- `proxy_cache_valid 200 301 302 30d`: cache successful responses and redirects for 30 days.
- `proxy_cache_use_stale`: serve cached files if the backend is down.
- `proxy_cache_lock on`: if 100 users request the same uncached file, only one request goes to the backend.

**If the backend sends `Cache-Control: no-cache`, `Set-Cookie`, or `Expires` headers, Nginx won't cache.** To override that for static files:

```nginx
proxy_ignore_headers Cache-Control Expires Set-Cookie;
proxy_hide_header Set-Cookie;
```

## 3. Apply and Test

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Check the headers:

```bash
curl -I http://example.com/style.css
```

You should see:

```
Cache-Control: max-age=2592000
Expires: <date 30 days from now>
X-Cache-Status: MISS      # first request
```

Run it again and you should see `X-Cache-Status: HIT`.

## 4. Important Tips

**Cache busting:** with a 30-day cache, changed files won't reach users until it expires. Put a version in the filename or URL:

```html
<link rel="stylesheet" href="/style.css?v=2">
<script src="/app.3f9a1c.js"></script>
```

Build tools like Webpack and Vite generate hashed filenames automatically. This is the best practice, and with hashed names you can safely cache for a year using `Cache-Control "public, max-age=31536000, immutable"`.

**Clearing the Nginx cache manually:**

```bash
sudo rm -rf /var/cache/nginx/*
sudo systemctl reload nginx
```

**`add_header` inheritance gotcha:** if a `location` block has its own `add_header`, it replaces any `add_header` set at the `server` level. Repeat the headers you need inside that location.

**Browser refresh:** pressing F5 or Ctrl+F5 bypasses the cache, so test by opening the page normally or checking the Network tab in DevTools (look for "disk cache" or "memory cache").

## Quick Reference

| Directive | Purpose |
|---|---|
| `expires 30d` | Browser caches for 30 days |
| `add_header Cache-Control "public"` | Allows browsers and CDNs to cache |
| `proxy_cache_path` | Defines the cache location and size |
| `proxy_cache_valid 200 30d` | Nginx-side cache duration |
| `proxy_cache_use_stale` | Serve old cache if the backend fails |
| `$upstream_cache_status` | Debug HIT / MISS |

If you tell me whether your files are served directly from disk or from a backend (Node.js, Laravel, etc.), I can give you a final config for your setup.
