To edit your Nginx config file (e.g., `/etc/nginx/nginx.conf`) on your VPS, follow these exact steps:

---

### ✅ Step-by-Step to Edit the File

1. **Open the file with `nano` (text editor):**

```bash
sudo nano /etc/nginx/nginx.conf
```

> If you see a permission denied error, make sure you’re using `sudo`.

---

2. **Scroll down to the `http { ... }` block**
   Use your arrow keys to find this section:

```nginx
http {
    ...
}
```

---

3. **Add this line inside the `http` block**
   Anywhere inside the `http` block (preferably under the other settings), **add**:

```nginx
client_max_body_size 50M;
```

Example:

```nginx
http {
    sendfile on;
    tcp_nopush on;
    types_hash_max_size 2048;

    client_max_body_size 50M;

    include /etc/nginx/mime.types;
    ...
}
```

---

4. **Save the file:**

* Press `CTRL + O` to write changes
* Press `Enter` to confirm the filename
* Press `CTRL + X` to exit

---

5. **Test the config for syntax errors:**

```bash
sudo nginx -t
```

You should see:

```bash
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

---

6. **Reload Nginx to apply the changes:**

```bash
sudo systemctl reload nginx
```

---

Now try uploading a file larger than 1MB again — it should work perfectly.
