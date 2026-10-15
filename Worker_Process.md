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
