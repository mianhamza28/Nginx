Bilkul. Matlab yeh hai: nginx mein **worker process** woh processes hain jo asal mein requests handle karte hain. Recommendation yeh hai ke inki tadaad aap ke server ke **CPU cores** ke barabar rakhein, taake har core ek worker ko mil jaye aur CPU ka poora faida uthaya ja sake.

## Example se samjhein

Maan lein aap ke server mein **4 CPU cores** hain:

- `worker_processes 1;` → sirf 1 core kaam kar raha hai, baqi 3 farigh. Traffic zyada aaye to slow ho jayega.
- `worker_processes 4;` → har core par 1 worker, sab cores kaam mein. Yeh best hai.
- `worker_processes 20;` → cores 4 hain, workers 20, to woh ek doosre ke saath CPU share karenge aur faida ki jagah overhead badhega.

Aise samjhein: 4 counters wali dukaan mein 4 cashier best hain. 1 cashier se line lambi hogi, aur 20 cashier 4 counters par khade ho kar aapas mein takrayenge.

## Apne server ke cores check karein

```bash
nproc
```

Agar output `2` aaye to aap ke paas 2 cores hain.

## Config mein kaise set karein

```bash
nano /etc/nginx/nginx.conf
```

Sab se upar (http block se bahar) yeh line hoti hai:

```nginx
worker_processes  1;
```

Isay apne core count ke mutabiq badal dein, example ke liye 2 cores:

```nginx
worker_processes  2;
```

**Ya sab se aasan tareeqa**: `auto` likh dein, nginx khud cores gin kar set kar lega:

```nginx
worker_processes  auto;
```

Aap ke liye main `auto` hi recommend karunga, kyunke VM ke cores badlein to dobara config nahi badalni parti.

Phir save kar ke:

```bash
nginx -t
systemctl reload nginx
```

## Check karein ke kaam kiya

```bash
ps aux | grep "nginx: worker"
```

Jitne `nginx: worker process` nazar aayein, utne workers chal rahe hain. Abhi aap ke status output mein sirf 1 worker tha (`23929`), kyunke default `worker_processes 1;` hai.

## Ek aur related setting

Isi ke saath `worker_connections` hoti hai, jo batati hai ke **har worker** kitne connections ek saath sambhal sakta hai:

```nginx
events {
    worker_connections  1024;
}
```

Total capacity ka andaza: `worker_processes × worker_connections`. Misal ke taur par 4 workers × 1024 = taqreeban 4096 connections ek saath.

Agar `nproc` ka output bata dein to main bata dunga ke aap ke liye kitne workers hone chahiye.



`worker_connections` ek worker process ki **ek waqt mein kitne connections sambhalne ki limit** hai.

Connection ka matlab hai koi bhi open link: browser se aaya hua client, ya nginx se kisi backend server (jaise `app1`/`app2`) tak jane wala link.

## Example

Maan lein `worker_connections 1024;` hai:

- Ek worker 1024 connections ek saath handle kar sakta hai.
- 1025wan user aaye to usay wait karna parega ya error milega (`worker_connections are not enough` error log mein aata hai).

Aise samjhein: ek cashier ke paas ek waqt mein 1024 customers ki line sambhalne ki capacity hai.

## Total capacity

```
worker_processes × worker_connections = total connections
```

| worker_processes | worker_connections | Total |
|---|---|---|
| 1 | 1024 | 1024 |
| 2 | 1024 | 2048 |
| 4 | 1024 | 4096 |

## Reverse proxy / load balancer mein dhyan dein

Agar nginx proxy ka kaam kar raha hai (jaise aap ka `app1`/`app2` wala setup), to har user request ke liye **2 connections** lagte hain: ek user se nginx tak, doosra nginx se backend tak. To asal capacity taqreeban aadhi ho jati hai:

```
4 workers × 1024 = 4096 connections → taqreeban 2048 users
```

## Aap ne jo do blocks bheje

Dono `events { }` block ka hissa hain. Asal config aisi hoti hai:

```nginx
events {
    worker_connections  1024;
}
```

Aap ke dusre snippet mein closing `}` nazar nahi aa raha. Config mein har `{` ka ek `}` hona zaroori hai, warna `nginx -t` syntax error dega.

## Kab badhana chahiye?

Chhote ya medium server ke liye **1024 bilkul theek hai**. Sirf tab badhayein jab error log mein yeh aaye:

```
worker connections are not enough
```

Tab badhane ka tareeqa:

```nginx
events {
    worker_connections  4096;
}
```

Phir:

```bash
nginx -t
systemctl reload nginx
```

Ek chhoti baat: system ki file limit (`ulimit -n`) bhi itni honi chahiye. Agar aap `worker_connections` ko bohat bara karein to nginx config mein `worker_rlimit_nofile 8192;` bhi add karni par sakti hai. Abhi default par rehna safe hai.
