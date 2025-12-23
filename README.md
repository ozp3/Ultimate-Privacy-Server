# 🛡️ Ultimate Privacy Server: Pi-hole + Unbound + PiVPN + DPI Bypass

**[ 🇬🇧 English Guide ](#-english-guide) | [ 🇹🇷 Türkçe Rehber ](#-türkçe-rehber)**

---

<a name="-english-guide"></a>
# 🇬🇧 English Guide

This guide sets up a complete home privacy center that blocks ads network-wide, encrypts DNS queries, bypasses ISP censorship (DPI), and allows secure remote access via VPN.

> **🎥 Credit / Source:** The installation steps for Pi-hole, Unbound, and PiVPN in this guide are based on the video **"Bu Kara Kutu İnternetinizi Düzeltiyor"** by **Evrim Ağacı TeknoBilim**.
> * [Watch the Video Guide](https://youtu.be/SACJ1m7GXTA)
> * *Note: The **DPI Bypass (Zapret)** section and specific Router configurations below are custom additions to the original video guide.*

## 🧠 Why are we doing this?
* **Pi-hole:** Acts as a "sinkhole" for DNS queries. It checks every request your devices make; if it's an ad or tracker, it blocks it before it downloads.
* **Unbound:** Instead of asking Google (8.8.8.8) or your ISP where a website is, Unbound acts as your own recursive DNS server. It talks directly to the global root servers, ensuring no single entity logs your browsing history.
* **Zapret:** ISPs use "Deep Packet Inspection" (DPI) to analyze and throttle/block traffic. Zapret modifies packet headers (desync) to fool these inspection boxes, bypassing censorship and speed limits.
* **PiVPN:** Allows you to tunnel back to your home network when you are outside (using mobile data or public Wi-Fi), so you get ad-blocking and security everywhere.

---

## 🛠️ Server Installation Steps

### 1. Update System
We start by ensuring the operating system and package lists are up to date to avoid conflicts.
```bash
sudo apt update && sudo apt upgrade -y

```

### 2. Install Pi-hole (The Ad Blocker)

Run the automated installer.

* **Crucial Step:** During installation, you will be asked to set a **Static IP** for your device. Say "Yes" and assign an IP (e.g., `192.168.1.100`) outside your router's DHCP range to ensure the server address never changes.

```bash
curl -sSL [https://install.pi-hole.net](https://install.pi-hole.net) | bash

```

### 3. Install Unbound (Recursive DNS)

We install Unbound and download the "Root Hints" file. This file contains the addresses of the root DNS servers that run the internet.

```bash
sudo apt install unbound
wget [https://www.internic.net/domain/named.root](https://www.internic.net/domain/named.root) -qO- | sudo tee /var/lib/unbound/root.hints

```

### 4. Configure Unbound

By default, Unbound listens on port 53. Since Pi-hole already uses port 53, we must move Unbound to port **5335** and configure it for local privacy.

```bash
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf

```

Paste the following configuration :

```yaml
server:
    verbosity: 0
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    do-ip6: no
    prefer-ip6: no
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: no
    edns-buffer-size: 1472
    prefetch: yes
    num-threads: 1
    so-rcvbuf: 1m
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10

```

Restart Unbound to apply changes:

```bash
sudo service unbound restart

```

> **Integration Step:** Now we must tell Pi-hole to use Unbound.
> 1. Go to Pi-hole Admin Interface (`http://<YOUR_STATIC_IP>/admin`).
> 2. Navigate to **Settings** -> **DNS**.
> 3. **Uncheck** all upstream DNS providers (Google, OpenDNS, etc.).
> 4. In the **Custom 1 (IPv4)** box on the right, type: `127.0.0.1#5335`
> 5. Scroll down and click **Save**.
> 
> 

### 5. DPI Bypass Setup (Zapret)

This step installs Zapret and configures the `nfqws` strategy to bypass ISP throttling and blocking.

**Download and Install:**

```bash
sudo apt install curl git ipset nftables -y
git clone [https://github.com/bol-van/zapret.git](https://github.com/bol-van/zapret.git)
cd zapret
sudo ./install_easy.sh

```

**Configure the Bypass Strategy (Critical):**
The default settings might not work for every ISP. We need to edit the config file to define the specific packet manipulation method.

```bash
sudo nano /opt/zapret/config

```

**Find and change these specific lines:**

1. **MODE:** Change the operation mode to `nfqws` (Netfilter Queue Web Socket).
```bash
MODE="nfqws"

```


2. **NFQWS_OPT:** This defines *how* we fool the DPI. Paste this specific strategy:
```bash
NFQWS_OPT="--filter-tcp=80,443 --dpi-desync=fake --dpi-desync-ttl=3"

```



**Apply Changes:**

```bash
sudo service zapret restart

```

### 6. Install PiVPN (WireGuard)

This sets up a VPN server.

* **Selection:** When asked, choose **WireGuard** (it's faster and more modern than OpenVPN).
* **Port:** Note the port number (default 51820) for the next section.

```bash
curl -L [https://install.pivpn.io](https://install.pivpn.io) | bash

```

---

## 📡 Router / Modem Configuration (Essential)

Even if you install everything correctly, it won't work automatically unless you configure your Router.

1. **Login:** Go to your Router's interface (usually `192.168.1.1` or `192.168.0.1`).
2. **LAN / DHCP Settings:** Find the "DHCP Server" settings (NOT the WAN/Internet settings).
3. **DNS Assignment:** You will see fields for Primary and Secondary DNS.
* **Primary DNS:** Enter your Raspberry Pi's Static IP (e.g., `192.168.1.100`).
* **Secondary DNS:** Leave blank or enter the Pi's IP again. **DO NOT** enter Google (8.8.8.8) here, or ads will sneak through.


4. **Save & Reboot:** Restart your router to force all devices to get the new DNS settings.

---

## 📱 How to Connect Clients (VPN)

To use this system from outside your home:

1. **Create a User:** Run this on the server:
```bash
pivpn add

```


*(Enter a name, e.g., "MyPhone").*
2. **Connect Mobile Phone:**
* Install the **WireGuard** app.
* Run `pivpn -qr` on the server.
* Scan the code with the app.


3. **Connect PC/Laptop:**
* The config files are stored in `~/wireguard/configs/`.
* Copy the `.conf` file to your computer.
* Import it into the WireGuard desktop client.



---

<a name="-türkçe-rehber"></a>

# 🇹🇷 Türkçe Rehber

Bu proje; ev ağınızdaki reklamları engelleyen, DNS sorgularınızı şifreleyen, sansürleri (DPI) aşan ve dışarıdan güvenli erişim sağlayan tam kapsamlı bir gizlilik merkezidir.

> **🎥 Kaynak / Referans:** Bu rehberdeki Pi-hole, Unbound ve PiVPN kurulum adımları **Evrim Ağacı TeknoBilim** kanalının **"Bu Kara Kutu İnternetinizi Düzeltiyor"** videosuna dayanmaktadır.
> * [Video Rehberini İzle](https://youtu.be/SACJ1m7GXTA)
> * *Not: Aşağıdaki **DPI Bypass (Zapret)** bölümü ve **Modem Ayarları** detayları, videoya ek olarak bu projeye özel eklenmiştir.*
> 
> 

## 🧠 Neden bunları yapıyoruz?

* **Pi-hole:** Bir "DNS çukuru" gibi çalışır. Cihazınız bir siteye gitmek istediğinde önce Pi-hole'a sorar. Eğer o site reklam ise, Pi-hole onu engeller ve cihazınıza hiç yüklenmez.
* **Unbound:** DNS sorguları için Google (8.8.8.8) veya Türk Telekom'a güvenmek yerine, kendi DNS sunucumuzu kuruyoruz. Unbound, doğrudan internetin kök (root) sunucularıyla konuşur. Kayıt tutulmaz, gizlilik %100 sizdedir.
* **Zapret:** Servis sağlayıcılar (ISP), "Derin Paket İnceleme" (DPI) cihazlarıyla hangi siteye girdiğinizi analiz edip hızınızı düşürebilir veya engelleyebilir. Zapret, giden paketlerin yapısını değiştirerek (desync) bu kutuları şaşırtır ve engelleri aşar.
* **PiVPN:** Evde değilken bile (Mobil veri, Kafe Wi-Fi) şifreli bir tünel ile ev ağınıza bağlanmanızı sağlar. Böylece reklam engelleme ve sansürsüz internet dışarıda da sizinle olur.

---

## 🛠️ Sunucu Kurulum Adımları

### 1. Hazırlık ve Güncelleme

Olası çakışmaları önlemek için sistemi güncelliyoruz.

```bash
sudo apt update && sudo apt upgrade -y

```

### 2. Pi-hole Kurulumu (Reklam Engelleyici)

Otomatik kurulumu başlatın.

* **Kritik Adım:** Kurulum sırasında cihazınıza **Statik IP** atamanız istenecek. Buna "Evet" diyerek modeminizin DHCP aralığı dışında sabit bir IP verin (Örn: `192.168.1.100`). Bu IP adresi sistemin kalbi olacağı için değişmemeli.

```bash
curl -sSL [https://install.pi-hole.net](https://install.pi-hole.net) | bash

```

### 3. Unbound Kurulumu (DNS Çözümleyici)

Unbound'u kurup, internetin kök sunucularının adreslerini içeren "Root Hints" dosyasını indiriyoruz.

```bash
sudo apt install unbound
wget [https://www.internic.net/domain/named.root](https://www.internic.net/domain/named.root) -qO- | sudo tee /var/lib/unbound/root.hints

```

### 4. Unbound Yapılandırması

Varsayılan olarak Unbound 53. portu kullanır, ancak Pi-hole da bu portu kullandığı için çakışma olur. Unbound'u **5335** portuna taşıyıp sadece Pi-hole'u dinleyecek şekilde yapılandırıyoruz.

```bash
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf

```

Aşağıdaki ayarları dosyanın içine yapıştırın :

```yaml
server:
    verbosity: 0
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    do-ip6: no
    prefer-ip6: no
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: no
    edns-buffer-size: 1472
    prefetch: yes
    num-threads: 1
    so-rcvbuf: 1m
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10

```

Ayarları uygulamak için servisi yeniden başlatın:

```bash
sudo service unbound restart

```

> **Entegrasyon Adımı:** Şimdi Pi-hole'a Unbound'u kullanmasını söylemeliyiz.
> 1. Pi-hole Yönetim Paneline gidin (`http://<SABIT_IP_ADRESINIZ>/admin`).
> 2. **Settings** -> **DNS** sekmesine gelin.
> 3. Sol taraftaki tüm hazır DNS sağlayıcıların (Google vb.) işaretini **kaldırın**.
> 4. Sağ taraftaki **Custom 1 (IPv4)** kutusuna şunu yazın: `127.0.0.1#5335`
> 5. Sayfanın en altına inip **Save** deyin.
> 
> 

### 5. DPI Bypass Ayarları (Zapret)

Bu adım, ISP kısıtlamalarını aşmak için Zapret'i kurar ve yapılandırır.

**İndirme ve Kurulum:**

```bash
sudo apt install curl git ipset nftables -y
git clone [https://github.com/bol-van/zapret.git](https://github.com/bol-van/zapret.git)
cd zapret
sudo ./install_easy.sh

```

**Strateji Ayarı (Çok Önemli):**
Varsayılan ayarlar her ISS'de çalışmayabilir. DPI kutularını kandırmak için özel bir konfigürasyon dosyası düzenlememiz gerekiyor.

```bash
sudo nano /opt/zapret/config

```

**Dosya içinde şu satırları bulup tam olarak böyle değiştirin:**

1. **MODE:** Çalışma modunu `nfqws` yapın.
```bash
MODE="nfqws"

```


2. **NFQWS_OPT:** Bypass yöntemini belirleyen sihirli komut budur. Şunu yapıştırın:
```bash
NFQWS_OPT="--filter-tcp=80,443 --dpi-desync=fake --dpi-desync-ttl=3"

```



**Değişiklikleri Uygulayın:**

```bash
sudo service zapret restart

```

### 6. PiVPN (WireGuard) Kurulumu

VPN sunucusunu kuruyoruz.

* **Seçim:** Kurulum sorarsa **WireGuard** protokolünü seçin (Daha hızlı ve günceldir).
* **Port:** Ekranda gösterilen port numarasını (varsayılan 51820) not edin.

```bash
curl -L [https://install.pivpn.io](https://install.pivpn.io) | bash

```

---

## 📡 Modem / Router Ayarları (Zorunlu)

Her şeyi doğru kursanız bile modeminize bu sunucuyu tanıtmazsanız sistem çalışmaz.

1. **Giriş:** Modem arayüzüne (Genelde `192.168.1.1`) girin.
2. **LAN / DHCP Ayarları:** "Yerel Ağ" veya "DHCP Sunucusu" ayarlarını bulun (WAN/İnternet ayarlarını DEĞİL).
3. **DNS Atama:** Birincil ve İkincil DNS kutularını göreceksiniz.
* **1. DNS:** Raspberry Pi'nin Statik IP adresini yazın (Örn: `192.168.1.100`).
* **2. DNS:** Burayı boş bırakın veya yine Pi'nin IP'sini yazın. **ASLA** Google DNS vb. yazmayın, yoksa reklamlar oradan kaçar.


4. **Kaydet ve Başlat:** Modemi yeniden başlatın. Artık evdeki tüm cihazlar otomatik olarak Pi-hole üzerinden geçecektir.

---

## 📱 Cihazlar Nasıl Bağlanır? (İstemci)

Dışarıdayken sisteme bağlanmak için:

1. **Kullanıcı Oluştur:** Sunucuda şu komutu girin:
```bash
pivpn add

```


*(Bir isim verin, Örn: "Telefonum").*
2. **Telefondan Bağlan:**
* Mağazadan **WireGuard** uygulamasını indirin.
* Sunucuda `pivpn -qr` yazın.
* Çıkan karekodu uygulamaya okutun.


3. **Bilgisayardan Bağlan:**
* Oluşan ayar dosyaları `~/wireguard/configs/` klasöründedir.
* `.conf` uzantılı dosyayı bilgisayarınıza kopyalayın.
* WireGuard masaüstü programına "Dosyadan İçe Aktar" diyerek yükleyin.
