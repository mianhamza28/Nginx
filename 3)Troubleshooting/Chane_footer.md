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
