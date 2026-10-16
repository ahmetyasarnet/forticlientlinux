# FortiClient VPN ile Linux'tan IPsec IKEv2 + FortiToken bağlantısı

Ücretsiz **FortiClient VPN** kullanarak Linux'tan (masaüstü, WSL veya GUI'siz Ubuntu Server) bir FortiGate **IPsec IKEv2** dial-up VPN'ine **FortiToken** ile bağlanma rehberi.

Kısaca yöntem:

1. IPsec profilini çalışan bir FortiClient'tan (macOS) XML olarak dışa aktar
2. XML'den sadece IPsec bölümünü ayır
3. Linux'taki ücretsiz FortiClient VPN'e GUI üzerinden **bir kez** içe aktar
4. Bundan sonra bağlantıyı tamamen terminalden kur, token kodunu terminale gir

```
$ forticlient vpn connect PROFIL -u kullanici
Password:
Status: Connecting...
=====================
Authentication Required
---------------------
A FortiToken code is required for IPsec-VPN login authentication.
=====================
FortiToken: 123456
Status: Connected
```

---

## İçindekiler

- [Arka plan: neden bu yol?](#arka-plan-neden-bu-yol)
- [Test edilen ortam](#test-edilen-ortam)
- [Gereksinimler](#gereksinimler)
- [1. (İsteğe bağlı) FortiGate'te test tüneli](#1-isteğe-bağlı-fortigatete-test-tüneli)
- [2. macOS'ta profili oluştur ve dışa aktar](#2-macosta-profili-oluştur-ve-dışa-aktar)
- [3. XML'den sadece IPsec bölümünü ayır](#3-xmlden-sadece-ipsec-bölümünü-ayır)
- [4. Linux'a FortiClient VPN kur](#4-linuxa-forticlient-vpn-kur)
- [5. XML'i içe aktar (GUI, bir kerelik)](#5-xmli-içe-aktar-gui-bir-kerelik)
- [6. Terminalden bağlan](#6-terminalden-bağlan)
- [7. GUI'siz Ubuntu Server'da kullanım](#7-guisiz-ubuntu-serverda-kullanım)
- [Bilinen kısıtlar](#bilinen-kısıtlar)
- [Sorun giderme](#sorun-giderme)
- [Kaynaklar](#kaynaklar)

---

## Arka plan: neden bu yol?

- **FortiOS 7.6.3 ile SSL VPN tunnel mode kaldırıldı**, yerine IPsec geldi. Linux'ta SSL VPN için yaygın kullanılan `openfortivpn` IPsec desteklemiyor. FortiGate'iniz hâlâ SSL VPN sunuyorsa `openfortivpn` en kolay yol olmaya devam ediyor.
- **strongSwan** FortiGate'in IPsec tüneline token'sız kullanıcıyla sorunsuz bağlanabiliyor. Ancak FortiToken açıkken FortiGate, şifreden sonra token kodunu ayrı bir soru olarak soruyor ve standart strongSwan bu soruya cevap veremiyor.
- **FortiClient Linux'ta arayüzden IPsec profili oluşturulamıyor.** Fortinet resmi olarak Linux'ta kişisel IPsec tünelini desteklemiyor, IPsec'in EMS üzerinden dağıtılmasını öneriyor. Ancak **XML içe aktarma** ile IPsec profili eklenebiliyor ve FortiClient token sorusunu kendisi yönetiyor.
- `repo.fortinet.com` deposundaki paket **ücretli** FortiClient. EMS'e bağlanmadan VPN'i kullandırmıyor (`FortiClient VPN trial has expired`). Bu rehber **ücretsiz FortiClient VPN** paketini kullanıyor.

> **Not:** Bu yöntem resmi olarak desteklenen bir senaryo değil. FortiClient'ın gelecekteki sürümlerinde çalışmayabilir. Bağlandığınız VPN başka bir kuruma aitse, kullanım şeklinin o kurumun politikasına uygun olduğundan emin olun.

## Test edilen ortam

| Bileşen | Sürüm |
|---|---|
| FortiGate | FortiOS 7.2.8 |
| Linux istemci | FortiClient VPN **7.4.3.5411** (ücretsiz, `.deb`) |
| Linux | Ubuntu 22.04 (WSL2 üzerinde, WSLg ile) |
| Dışa aktarma | FortiClient VPN 7.4 (macOS) |
| Tünel | IPsec IKEv2, Pre-shared key, EAP, FortiToken Mobile |

## Gereksinimler

- FortiGate'te IKEv2 dial-up IPsec tüneli ve FortiToken atanmış bir kullanıcı
- O tünele bağlanabilen bir **macOS** FortiClient (profili dışa aktarmak için)
- Linux makinede `python3` ve GUI'yi bir kez açabilmek için bir ekran: masaüstü, WSLg ya da SSH X11 yönlendirmesi

---

## 1. (İsteğe bağlı) FortiGate'te test tüneli

Kendi FortiGate'inizde denemek istiyorsanız aşağıdaki yapı test edildi. Mevcut bir IKEv2 tüneline bağlanacaksanız bu adımı atlayın.

> **Not:** GUI'deki *IPsec Wizard → Remote Access → FortiClient* şablonu tüneli **IKEv1** olarak oluşturur. FortiClient 7.4 IPsec'te sadece **IKEv2** destekliyor. EAP ayarları da sadece CLI'dan yapılabiliyor. Bu yüzden aşağıdaki CLI yapılandırmasını kullanın.

Değerleri kendi ortamınıza göre değiştirin: `wan1`, `internal`, ağ adresleri, grup ve PSK.

```
config firewall address
    edit "ikev2-pool"
        set type iprange
        set start-ip 10.212.51.1
        set end-ip 10.212.51.20
    next
    edit "lan-subnet"
        set subnet 192.168.1.0 255.255.255.0
    next
end

config vpn ipsec phase1-interface
    edit "ikev2-dialup"
        set type dynamic
        set interface "wan1"
        set ike-version 2
        set peertype any
        set net-device disable
        set mode-cfg enable
        set proposal aes256-sha256 aes128-sha256
        set dhgrp 14
        set eap enable
        set eap-identity send-request
        set authusrgrp "VPN_GRUBU"
        set ipv4-start-ip 10.212.51.1
        set ipv4-end-ip 10.212.51.20
        set ipv4-split-include "lan-subnet"
        set psksecret "UZUN_RASTGELE_PSK"
    next
end

config vpn ipsec phase2-interface
    edit "ikev2-dialup"
        set phase1name "ikev2-dialup"
        set proposal aes256-sha256 aes128-sha256
        set dhgrp 14
    next
end

config firewall policy
    edit 0
        set name "ikev2-dialup-to-lan"
        set srcintf "ikev2-dialup"
        set dstintf "internal"
        set srcaddr "ikev2-pool"
        set dstaddr "lan-subnet"
        set action accept
        set schedule "always"
        set service "ALL"
    next
end
```

- IP havuzu, varsa SSL VPN havuzuyla çakışmamalı.
- `ipv4-split-include` ile split tunnel açık. Sunucunun tüm trafiği tünele girmez, SSH oturumu kopmaz.
- Kullanıcıya FortiToken'ı **User & Authentication → User Definition** ekranından atayın.

## 2. macOS'ta profili oluştur ve dışa aktar

### 2.1 Profili oluştur

Profil zaten FortiClient'ta varsa bu adımı atlayın. Yoksa **Remote Access → Configure VPN**:

| Alan | Değer |
|---|---|
| VPN | IPsec VPN |
| Connection Name | Örneğin `IPSEC_VPN` |
| Remote Gateway | FortiGate WAN IP'si veya DNS adı |
| Authentication Method | Pre-shared key |
| IKE Version (Advanced) | 2 |
| Phase 1 / Phase 2 | FortiGate ile aynı: AES256-SHA256 / AES128-SHA256, DH 14 |

Önce **macOS'ta bağlanıp** token ile çalıştığını doğrulayın. Bu profil Linux'a olduğu gibi taşınacak.

### 2.2 Dışa aktar

macOS'taki GUI yedeği okunabilir bir XML üretmiyor. FortiClient'ın komut satırı aracını kullanın:

```bash
cd "/Library/Application Support/Fortinet/FortiClient/bin"
sudo ./FCConfig -f ~/Downloads/forticlient-export.xml -m all -o export
```

### 2.3 Kontrol et

```bash
grep -n -E "<ipsecvpn>|<name>|<version>|<authentication_method>" ~/Downloads/forticlient-export.xml
grep -c "EncX" ~/Downloads/forticlient-export.xml
```

Beklenen:
- Bağlantı `<ipsecvpn>` altında
- Bağlantının `<version>` değeri `2`
- `EncX` sayısı `0`

> **PSK biçimi önemli.** Linux FortiClient, PSK'nın `Enc ...` biçimini çözebiliyor. **Şifreyle korunan** dışa aktarmada oluşan `EncX ...` biçimini çözemiyor ve bağlantı IKE aşamasında düşüyor. `FCConfig` ile şifresiz dışa aktarma `Enc` biçimini üretiyor. Windows'tan şifreli yedek alınırsa bu sorunla karşılaşılabilir.

> **Güvenlik:** Bu dosya şifrelenmiş PSK'yı ve kayıtlı kullanıcı bilgilerinizi içerir. Kimseyle paylaşmayın, git'e koymayın, işiniz bitince silin.

## 3. XML'den sadece IPsec bölümünü ayır

macOS'tan alınan dosyada VPN dışında macOS'a özel birçok ayar var. Linux'a sadece IPsec bağlantısını aktarmak daha güvenli.

Dosyayı Linux makineye kopyalayın, sonra:

```bash
mkdir -p ~/forti-import && chmod 700 ~/forti-import
cp /DOSYANIN/YOLU/forticlient-export.xml ~/forti-import/export.xml
chmod 600 ~/forti-import/export.xml
cd ~/forti-import

python3 - <<'EOF'
import xml.etree.ElementTree as ET

src = ET.parse("export.xml").getroot()
ipsec = src.find("./vpn/ipsecvpn")
if ipsec is None:
    raise SystemExit("ipsecvpn bolumu bulunamadi")

root = ET.Element("forticlient_configuration")
vpn = ET.SubElement(root, "vpn")
vpn.append(ipsec)
ET.ElementTree(root).write("linux-import.xml", encoding="utf-8", xml_declaration=True)

for c in ipsec.findall("./connections/connection"):
    ike = c.find("ike_settings")
    print("baglanti:", c.findtext("name"),
          "| IKE:", ike.findtext("version") if ike is not None else "?",
          "| auth:", ike.findtext("authentication_method") if ike is not None else "?")
EOF

chmod 600 linux-import.xml
```

Beklenen çıktı:

```
baglanti: IPSEC_VPN | IKE: 2 | auth: Preshared Key
```

## 4. Linux'a FortiClient VPN kur

> **Dikkat:** `repo.fortinet.com` deposundaki `forticlient` paketi **ücretli** sürüm. Lisans olmadan `FortiClient VPN trial has expired` hatası verir. Ücretsiz **FortiClient VPN** paketini kullanın.

Daha önce ücretli paketi kurduysanız önce tamamen kaldırın:

```bash
sudo apt purge -y forticlient
sudo rm -f /etc/apt/sources.list.d/repo.fortinet.com.list
sudo rm -rf /etc/forticlient /opt/forticlient
sudo apt update
```

Ücretsiz paketi kurun. Güncel sürüm için [Fortinet indirme sayfasındaki](https://www.fortinet.com/support/product-downloads) **FortiClient VPN → Linux .deb** bağlantısına bakın.

```bash
wget https://filestore.fortinet.com/forticlient/downloads/forticlient_vpn_7.4.3.5411_amd64.deb
sudo apt install -y ./forticlient_vpn_7.4.3.5411_amd64.deb
```

Kontrol:

```bash
systemctl status forticlient --no-pager | head -3
forticlient vpn list
```

- Servis `active (running)` olmalı.
- `forticlient vpn` komutları **normal kullanıcıyla** çalıştırılmalı. Root ile `Please run this command under non-root user.` hatası verir.
- `vpn list` çıktısı `(No VPN profile found)` olmalı. `trial has expired` görüyorsanız ücretli paket hâlâ kurulu demektir.

> **Not:** Ücretsiz sürümde `forticlient vpn import` komutu **yok**. Ücretli sürümde var. İçe aktarma bu yüzden bir kez GUI üzerinden yapılıyor.

> **IKE portları:** FortiClient UDP 500/4500 kullanır. Makinede strongSwan veya başka bir IPsec servisi çalışıyorsa önce durdurun: `sudo systemctl disable --now strongswan`

## 5. XML'i içe aktar (GUI, bir kerelik)

XML'i normal kullanıcının ev klasörüne koyun:

```bash
cp ~/forti-import/linux-import.xml ~/linux-import.xml
chmod 600 ~/linux-import.xml
```

GUI'yi açın:

```bash
forticlient gui &
```

Açılmazsa: `/opt/forticlient/start-fortitray-launcher.sh &`

Pencerede:

1. **Configure VPN** ekranına gidin
2. **XML → Import XML Configuration** seçeneğini seçin
3. Dosya seçme penceresinde dosya görünmüyorsa **Ctrl+L** ile adres satırını açıp tam yolu yazın: `/home/KULLANICI/linux-import.xml`
4. Şifre sorarsa boş bırakın (şifresiz dışa aktarıldı)

Kontrol:

```bash
forticlient vpn list
```

```
VPNs:
  Personal VPNs:
    IPSEC_VPN
```

İçe aktarılan profil GUI'de düzenlenemiyor. Değişiklik gerekirse macOS'ta düzenleyip yeniden dışa aktarın.

GUI'yi kapatın ve XML'i silin:

```bash
rm ~/linux-import.xml ~/forti-import/*.xml
```

## 6. Terminalden bağlan

```bash
forticlient vpn connect IPSEC_VPN -u KULLANICI_ADI
```

1. `Password:` sorusuna şifrenizi girin
2. `FortiToken:` sorusu gelince FortiToken uygulamasındaki kodu **hemen** girin

> **Token zamanlaması:** Kodlar 30-60 saniyede bir değişiyor. Soru geldiğinde kodun süresi bitmek üzereyse yenisini bekleyin. Soru ile cevap arasında uzun süre geçerse `IPsec VPN login failed due to incorrect FortiToken` hatası alırsınız.

> GUI açıksa terminale `Receive websocket ... UNHANDLED` gibi log satırları düşebilir. Bağlantıyı etkilemez, ama GUI'yi kapatmak kafa karışıklığını önler.

Durum ve test:

```bash
forticlient vpn status
ping -c 3 IC_AGDAKI_BIR_IP
```

Bağlantıyı kapatmak:

```bash
forticlient vpn disconnect
```

Diğer seçenekler (`forticlient vpn connect --help`):

| Seçenek | Anlamı |
|---|---|
| `-s`, `--save-password` | Şifreyi hatırla |
| `-w`, `--always-up` | Bağlantıyı canlı tutmaya çalış |
| `-a`, `--auto-connect` | Açılışta bağlan |

## 7. GUI'siz Ubuntu Server'da kullanım

Bağlantı tamamen terminalden yapıldığı için sunucuda ekran gerekmez. GUI'ye sadece **5. adımdaki bir kerelik içe aktarma** için ihtiyaç var. Bunu **SSH X11 yönlendirmesi** ile yapabilirsiniz: program sunucuda çalışır, penceresi sizin bilgisayarınızda açılır.

### 7.1 Sunucuda hazırlık

```bash
wget https://filestore.fortinet.com/forticlient/downloads/forticlient_vpn_7.4.3.5411_amd64.deb
sudo apt install -y ./forticlient_vpn_7.4.3.5411_amd64.deb
sudo apt install -y xauth
grep -i '^X11Forwarding' /etc/ssh/sshd_config
```

Son komut `X11Forwarding yes` göstermeli. Ubuntu'da varsayılan olarak açık.

### 7.2 İçe aktarma

İstemci tarafında bir X sunucusu gerekir:

| İstemci | X sunucusu |
|---|---|
| Linux masaüstü | Hazır |
| Windows | WSL2 (WSLg hazır) üzerinden `ssh -X` |
| macOS | [XQuartz](https://www.xquartz.org/) |

```bash
scp linux-import.xml KULLANICI@SUNUCU:~/
ssh -X KULLANICI@SUNUCU
```

Sunucuda:

```bash
chmod 600 ~/linux-import.xml
forticlient gui &
```

Pencere sizin ekranınızda açılır. 5. adımdaki gibi içe aktarın, sonra:

```bash
forticlient vpn list
rm ~/linux-import.xml
```

### 7.3 Günlük kullanım

SSH bağlantısı kopsa bile VPN oturumu açık kalsın diye `tmux` içinde çalıştırın:

```bash
tmux new -s vpn
forticlient vpn connect IPSEC_VPN -u KULLANICI_ADI
```

- tmux'tan çıkmak (oturum açık kalır): `Ctrl+B`, ardından `D`
- Geri dönmek: `tmux attach -t vpn`

> **SSH'ı kaybetme riski:** Profil tüm trafiği tünele yönlendiriyorsa (full tunnel), sunucuya bağlandığınız SSH oturumu kopabilir. İlk denemeyi tmux içinde yapın ve sunucuya konsol ya da aynı yerel ağdan erişiminiz olsun.

---

## Bilinen kısıtlar

- **Gözetimsiz bağlantı mümkün değil.** Her yeni bağlantıda FortiToken kodu bir insan tarafından girilmeli. `-w` bağlantıyı canlı tutmaya çalışır ama tünel tamamen düşerse token yeniden sorulur. Tamamen otomatik bağlantı için FortiGate tarafında token'sız, sertifika tabanlı ayrı bir tünel gerekir.
- **İçe aktarılan profil düzenlenemiyor.** Değişiklik için kaynak makinede düzenleyip yeniden dışa aktarın.
- **Ücretsiz sürümde CLI ile içe aktarma yok.** GUI'nin bir kez açılabilmesi gerekiyor.
- **Resmi destek yok.** Fortinet, Linux'ta kişisel IPsec tünelini resmi olarak desteklemiyor. FortiClient güncellemeleri bu yöntemi bozabilir.
- **Push bildirim ve SAML** bu rehberde test edilmedi. Test edilen yöntem FortiToken Mobile kodunun elle girilmesi.

## Sorun giderme

| Belirti | Olası sebep / çözüm |
|---|---|
| `FortiClient VPN trial has expired` | Ücretli paket kurulu. [4. adımdaki](#4-linuxa-forticlient-vpn-kur) gibi tamamen kaldırıp ücretsiz paketi kurun. |
| `Please run this command under non-root user.` | `forticlient vpn` komutlarını normal kullanıcıyla çalıştırın. |
| `vpn import` komutu yok | Ücretsiz sürümde yok. GUI'den içe aktarın. |
| Dosya seçme penceresinde XML görünmüyor | Filtre gizliyor olabilir. **Ctrl+L** ile tam yolu yazın. |
| `ipsecvpn bolumu bulunamadi` | Profil IPsec değil, SSL VPN olarak tanımlı. |
| IKE aşamasında bağlantı düşüyor, kimlik doğrulamaya hiç gelmiyor | PSK `EncX` biçiminde olabilir. Şifresiz dışa aktarın. Ya da makinede başka bir IPsec servisi 500/4500 portlarını tutuyor: `sudo ss -ulpn \| grep -E ':500 \|:4500 '` |
| `IPsec VPN login failed due to incorrect FortiToken` | Kod soruyla cevap arasında eskimiş. Bağlantıyı yeniden başlatın, soru gelir gelmez yeni kodu girin. |
| SSH bağlantısı VPN'e bağlanınca koptu | Profil full tunnel. Konsoldan bağlanıp `forticlient vpn disconnect` yapın. Mümkünse FortiGate'te split tunnel kullanın. |

FortiGate tarafında kimlik doğrulama adımlarını görmek için:

```
diagnose debug reset
diagnose vpn ike log-filter dst-addr4 ISTEMCI_DIS_IP
diagnose debug application ike -1
diagnose debug application fnbamd -1
diagnose debug enable
```

Kapatmak için `diagnose debug disable`. FortiOS 7.2'de filtre komutu `log-filter` (tireli) ve `dst-addr4` şeklinde. Daha yeni sürümlerde sözdizimi farklı olabilir.

## Kaynaklar

- [FortiOS 7.6.3: SSL VPN tunnel mode kaldırıldı](https://openvpn.net/fortinet-ssl-vpn-migration/)
- [Fortinet Community: FortiClient VPN only with IPsec on Linux (XML içe aktarma önerisi)](https://community.fortinet.com/support-forum-92/forticlient-vpn-only-with-ipsec-on-linux-223880)
- [Fortinet Community: Linux FortiClient, şifreli XML profilindeki PSK sorunu](https://community.fortinet.com/support-forum-92/forticlient-linux-never-decrypts-the-ipsec-preshared-key-from-an-saml-xml-profile-ike-auth-fails-with-failed-to-decrypt-pre-shared-key-229475)
- [Fortinet Community: macOS FortiClient yapılandırma dışa/içe aktarma (FCConfig)](https://community.fortinet.com/support-forum-92/forticlient-macos-configuration-restore-162225)
- [Fortinet Community: Ubuntu'da "FortiClient has expired" (ücretli/ücretsiz sürüm farkı)](https://community.fortinet.com/support-forum-92/ubuntu-s-forticlient-has-expired-211133)
- [FortiClient 7.4 XML referansı: IPsec VPN](https://docs.fortinet.com/document/forticlient/7.4.6/xml-reference-guide/739387)
- [FortiClient 8.0 yönetim kılavuzu: IPsec VPN bağlantısı yapılandırma](https://docs.fortinet.com/document/forticlient/8.0.0/administration-guide/494391/configuring-an-ipsec-vpn-connection)
- [Fortinet: IKEv2 dial-up IPsec ile FortiToken MFA için gereken sürümler](https://community.fortinet.com/fortigate-3/technical-tip-required-firmware-software-versions-for-using-fortitoken-mobile-or-otp-mfa-with-fortigate-ikev2-dialup-ipsec-219452)
- [Fortinet: IKEv2 dial-up IPsec, RADIUS ve FortiToken MFA](https://community.fortinet.com/fortigate-3/technical-tip-ikev2-dial-up-ipsec-tunnel-with-radius-and-fortitoken-mfa-112975)
