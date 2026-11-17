# Farmers Friend — Personal Server & Custom Domain Hosting Guide

Your production website bundle is packaged into a single archive:
**`farmers-friend-deploy.zip`**

It contains the complete compiled website (`index.html`, `.htaccess`, and `assets/`), plus an example Nginx config (`nginx.conf.example`).

---

## 1. Option A — Shared Hosting / cPanel / Hostinger / DirectAdmin (Easiest)
1. Point your personal domain DNS (`A` record) to your hosting server IP and enable **SSL (HTTPS)**.
2. Open **File Manager** in cPanel / hPanel and go to `public_html/` (or your domain's document root).
3. Upload **`farmers-friend-deploy.zip`** and click **Extract** directly inside `public_html/`.
4. Verify `index.html`, `.htaccess`, and the `assets/` folder are directly inside `public_html/`.

---

## 2. Option B — Personal Linux VPS (Nginx)
1. Upload `farmers-friend-deploy.zip` to your server and unzip it:
   ```bash
   sudo mkdir -p /var/www/farmers-friend
   sudo unzip -o farmers-friend-deploy.zip -d /var/www/farmers-friend
   ```
2. Configure Nginx using the included `nginx.conf.example` (replace `yourdomain.com` with your domain):
   ```nginx
   server {
       listen 80;
       server_name yourdomain.com www.yourdomain.com;
       root /var/www/farmers-friend;
       index index.html;

       location / {
           try_files $uri $uri/ /index.html;
       }

       location ~* \.(js|css|jpg|jpeg|png|webp|svg|woff2)$ {
           expires 1y;
           add_header Cache-Control "public, immutable";
       }
   }
   ```
3. Enable HTTPS with Let's Encrypt Certbot:
   ```bash
   sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
   ```

---

## 3. Critical Step for Firebase Login & Database on Your Personal Domain
After your personal domain is live over **HTTPS**:
1. Open the **Firebase Console** for project **`farmers-friends-c3f12`**.
2. Go to **Authentication → Settings → Authorized domains**.
3. Click **Add domain** and add both:
   - `yourdomain.com`
   - `www.yourdomain.com`
4. Without this step, Google Sign-In and Email Verification will be blocked on your custom domain (`auth/unauthorized-domain`).

Admin access remains restricted exclusively to **`info.pncom@gmail.com`** (email verified).
