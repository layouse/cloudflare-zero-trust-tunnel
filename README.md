# ☁️ Cloudflare Tunnel Kurulum Rehberi

### Debian 12 / Debian 13 için adım adım rehber

Bu rehberde, Cloudflare Zero Trust Tunnel kullanarak sunucunuzdaki web sitelerini veya uygulamaları **port açmadan**, **IP adresinizi gizleyerek** ve **HTTPS desteğiyle** internet üzerinden nasıl yayınlayabileceğinizi öğreneceksiniz.

Örnek olarak;

* `app1.example.com`
* `app2.example.com`
* `app3.example.com`

gibi birden fazla **subdomain (alt alan adı)** oluşturacağız ve bunların tamamını **tek bir Cloudflare Tunnel** üzerinden çalıştıracağız.

> 💡 İsterseniz yalnızca **1 adet subdomain** de kullanabilirsiniz. Bu rehberde mantığın daha iyi anlaşılması için 3 adet örnek üzerinden ilerlenmektedir.

---

## Neler Öğreneceksiniz?

✅ Cloudflare Tunnel kurulumu

✅ Subdomain oluşturma mantığı

✅ Birden fazla subdomain'i tek Tunnel altında çalıştırma

✅ DNS kayıtlarını otomatik oluşturma

✅ cloudflared servisini kurma

✅ Yaygın hataları çözme

---

# 1. Subdomainler İçin Web Klasörlerini Oluşturma

Öncelikle her subdomain için ayrı bir klasör oluşturacağız.

Bu örnekte:

| Subdomain        | Açılacak Sayfa |
| ---------------- | -------------- |
| app1.example.com | App 1          |
| app2.example.com | App 2          |
| app3.example.com | App 3          |

Aşağıdaki komutlar örnek amaçlıdır:

```bash
sudo mkdir -p /var/www/html/app1
sudo mkdir -p /var/www/html/app2
sudo mkdir -p /var/www/html/app3

echo "<h1>App 1</h1>" | sudo tee /var/www/html/app1/index.html

echo "<h1>App 2</h1>" | sudo tee /var/www/html/app2/index.html

echo "<h1>App 3</h1>" | sudo tee /var/www/html/app3/index.html
```

---

# 2. Apache Üzerinde Subdomainleri Tanımlama

Şimdi Apache'ye hangi subdomain'in hangi klasörü açacağını söyleyeceğiz.

### app1.example.com

```bash
sudo tee /etc/apache2/sites-available/app1.conf > /dev/null <<EOF
<VirtualHost *:80>
    ServerName app1.example.com
    DocumentRoot /var/www/html/app1
</VirtualHost>
EOF
```

### app2.example.com

```bash
sudo tee /etc/apache2/sites-available/app2.conf > /dev/null <<EOF
<VirtualHost *:80>
    ServerName app2.example.com
    DocumentRoot /var/www/html/app2
</VirtualHost>
EOF
```

### app3.example.com

```bash
sudo tee /etc/apache2/sites-available/app3.conf > /dev/null <<EOF
<VirtualHost *:80>
    ServerName app3.example.com
    DocumentRoot /var/www/html/app3
</VirtualHost>
EOF
```

Daha sonra bunları aktif edin:

```bash
sudo a2ensite app1
sudo a2ensite app2
sudo a2ensite app3

sudo systemctl reload apache2
```

---

# 3. Cloudflare Tunnel (cloudflared) Kurulumu

Önce Cloudflare'ın resmi deposunu ekleyelim:

```bash
sudo mkdir -p /usr/share/keyrings

curl -L https://pkg.cloudflare.com/cloudflare-main.gpg \
| sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared bookworm main" \
| sudo tee /etc/apt/sources.list.d/cloudflared.list

sudo apt update

sudo apt install -y cloudflared
```

Kurulumun başarılı olup olmadığını kontrol edin:

```bash
cloudflared --version
```

---

# 4. Cloudflare Hesabınıza Bağlanın

```bash
cloudflared tunnel login
```

Komutu çalıştırdıktan sonra size bir URL verilecektir.

Bu bağlantıyı tarayıcıda açın ve:

* Alan adınızı seçin
* Yetki verin
* İşlemi onaylayın

Başarılı olduğunda terminale geri dönecektir.

---

# 5. Tunnel Oluşturma

Örneğin:

```bash
cloudflared tunnel create my-tunnel
```

Başarılı olduğunda şuna benzer bilgiler göreceksiniz:

```text
Tunnel credentials written to:

/home/username/.cloudflared/xxxxxxxx-xxxx-xxxx.json

Created tunnel my-tunnel with id:

xxxxxxxx-xxxx-xxxx
```

Bu ID'yi bir sonraki adımda kullanacağız.

---

# 6. config.yml Dosyasını Oluşturma

```bash
nano ~/.cloudflared/config.yml
```

İçeriği:

```yaml
tunnel: YOUR_TUNNEL_ID

credentials-file: /home/username/.cloudflared/YOUR_TUNNEL_ID.json

ingress:

- hostname: app1.example.com
  service: http://localhost:80

- hostname: app2.example.com
  service: http://localhost:80

- hostname: app3.example.com
  service: http://localhost:80

- service: http_status:404
```

> 💡 Buradaki YOUR_TUNNEL_ID kısmını kendi Tunnel ID'niz ile değiştirin.

---

# 7. DNS Kayıtlarını Otomatik Oluşturma

Şimdi her subdomain'i Tunnel'a bağlayacağız:

```bash
cloudflared tunnel route dns my-tunnel app1.example.com

cloudflared tunnel route dns my-tunnel app2.example.com

cloudflared tunnel route dns my-tunnel app3.example.com
```

Bu komutlar Cloudflare panelinde otomatik CNAME oluşturacaktır.

---

# 8. Tunnel'ı Servis Olarak Çalıştırma

```bash
sudo cloudflared --config ~/.cloudflared/config.yml service install

sudo systemctl start cloudflared

sudo systemctl enable cloudflared
```

Durum kontrolü:

```bash
sudo systemctl status cloudflared
```

---

# ⚠️ Yaygın Hatalar

## Port 80 zaten kullanılıyor

```text
Address already in use:
make_sock: could not bind to port 80
```

Sebep:

Nginx veya başka bir servis 80 portunu kullanıyor.

Çözüm:

```bash
sudo systemctl stop nginx

sudo systemctl disable nginx

sudo systemctl restart apache2
```

---

## DNS kaydı zaten mevcut

```text
Code: 1003

An A, AAAA, or CNAME record with that host already exists
```

Çözüm:

Cloudflare paneline girin.

Eski DNS kaydını silin.

Daha sonra:

```bash
cloudflared tunnel route dns ...
```

komutunu tekrar çalıştırın.

---

## Service Conflict

```text
cloudflared service is already installed
```

Çözüm:

```bash
sudo cloudflared service uninstall

sudo rm -rf /etc/cloudflared

sudo cloudflared --config ~/.cloudflared/config.yml service install
```

---

# 🧪 Test

```bash
cloudflared tunnel --config ~/.cloudflared/config.yml --url localhost:80
```

---

# ☁️ Son Söz

Bu rehber, **layouse** tarafından Linux topluluğuna katkı sağlamak amacıyla hazırlanmıştır.

Tamamen açık kaynak paylaşılmıştır.

Tüm örnek alan adları (`example.com`) temsilidir ve kendi alan adınız ile değiştirilmelidir.
```bash
EAC365.COM
WİDİSKY.COM.TR
```
**© 2026 - Herkese açık, özgürce paylaşılabilir - EAC365 & WİDİSKY**

