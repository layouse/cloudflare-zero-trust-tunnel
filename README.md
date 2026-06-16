# ☁️ Cloudflare Zero Trust Tunnel — Debian 12/13 Kurulum Rehberi

> Debian 12/13 üzerinde **Cloudflare Zero Trust Tunnel** (`cloudflared`) kullanarak Apache sanal konakları ile birden fazla alt alan adını (subdomain) güvenli bir şekilde yayınlamak için gerekli tüm adımları, konfigürasyon detaylarını ve karşılaşılan hataların çözümlerini içeren kapsamlı rehber.

---

## 🚀 Özellikler

- 🔐 **Zero Trust** mimarisi ile güvenli tünel
- 🌐 **Apache Virtual Hosts** ile çoklu subdomain desteği
- 📦 **cloudflared** kurulumu ve yapılandırması
- 🧩 DNS rotalarının otomatik oluşturulması
- ⚠️ Yaygın hatalar ve çözümleri (Port çakışması, DNS çakışması, servis çakışması)
- 🇹🇷🇺🇸 Türkçe & İngilizce dil desteği

---

## 📋 İçindekiler

1. [Kurulum Adımları](#kurulum-adımları)
2. [Apache Sanal Konak Yapılandırması](#1-apache-sanal-konak-yapılandırması)
3. [Cloudflare Tunnel Kurulumu](#3-cloudflare-tunnel-kurulumu)
4. [DNS Rotalarının Oluşturulması](#4-dns-rotalarını-oluşturma)
5. [Servis Kurulumu ve Başlatılması](#5-cloudflared-servisini-kurma-ve-başlatma)
6. [Karşılaşılan Hatalar ve Çözümleri](#-karşılaşılan-yaygın-hatalar-ve-çözümleri)
7. [Kaynaklar](#-kaynaklar)
8. [Katkıda Bulunanlar](#-katkıda-bulunanlar)
--- 

---

## 🔧 Kurulum Adımları

### 1. Apache Sanal Konak Yapılandırması

Her alt alan adı için bir web dizini ve sanal konak dosyası oluşturulur. Aşağıdaki örnekte `example.com` ana alan adınız ve `app1`, `app2`, `app3` alt alan adlarınız olsun.

```bash
# Web dizinlerini oluştur
sudo mkdir -p /var/www/html/app1 /var/www/html/app2 /var/www/html/app3

# Örnek index dosyaları
echo "<h1>App 1</h1>" | sudo tee /var/www/html/app1/index.html
echo "<h1>App 2</h1>" | sudo tee /var/www/html/app2/index.html
echo "<h1>App 3</h1>" | sudo tee /var/www/html/app3/index.html

# Sanal konak dosyalarını oluştur
sudo tee /etc/apache2/sites-available/app1.conf > /dev/null <<EOF
<VirtualHost *:80>
    ServerName app1.example.com
    DocumentRoot /var/www/html/app1
</VirtualHost>
EOF

sudo tee /etc/apache2/sites-available/app2.conf > /dev/null <<EOF
<VirtualHost *:80>
    ServerName app2.example.com
    DocumentRoot /var/www/html/app2
</VirtualHost>
EOF

sudo tee /etc/apache2/sites-available/app3.conf > /dev/null <<EOF
<VirtualHost *:80>
    ServerName app3.example.com
    DocumentRoot /var/www/html/app3
</VirtualHost>
EOF

# Konakları etkinleştir ve Apache'yi yeniden yükle
sudo a2ensite app1 app2 app3
sudo systemctl reload apache2
