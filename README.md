[README.md](https://github.com/user-attachments/files/33537079/README.md)
# ANKAFEST 2027 — Festival Kayıt, QR Bilet ve Geçiş Otomasyonu

Havacılık, uzay ve teknoloji festivali için uçtan uca kayıt ve alan yönetimi sistemi. Katılımcı tek bir formu doldurur; sistem kimlik doğrulamasını yapar, kişiye özel QR bilet üretir, PDF bileti e-posta ile gönderir, festival alanındaki giriş-çıkışları turnike ekranı üzerinden kaydeder ve yönetime anlık rapor sunar.

**Canlı site:** https://alinalovers.github.io/ankafest/
**Demo admin PIN:** `sedcom61`
**İletişim:** ankafestt06@gmail.com · +90 505 176 75 37

---

## 1. Problem

Teknoloji festivallerinde kayıt ve alan yönetimi genellikle üç ayrı araçla yürütülür: bir form hizmeti, bir elektronik tablo ve kapıda tutulan kâğıt liste. Bunun pratikteki sonuçları şunlardır:

- Kimlik doğrulaması yapılmadığı için hatalı ve mükerrer kayıtlar oluşur
- Bilet gönderimi elle yapılır, yüzlerce katılımcıda sürdürülemez
- Alandaki anlık doluluk bilinmez; kapasite ve güvenlik planlaması için veri yoktur
- Giriş-çıkış saatleri tutulmadığı için katılım süresi ve panel yoğunluğu ölçülemez
- Kişisel veriler denetimsiz biçimde dağınık tablolarda kalır
- Yabancı katılımcılar için ayrı bir arayüz bulunmaz

## 2. Çözüm

Tek bir web arayüzü ve arkasındaki n8n otomasyonu bu sorunları birlikte çözer.

| Aşama | Ne oluyor | Nerede çalışıyor |
|---|---|---|
| Kayıt | T.C. kimlik algoritması, telefon ve e-posta doğrulaması, mükerrer kayıt kontrolü | Tarayıcı + n8n |
| Bilet | Benzersiz kod (`ANK-2027-XXXXX`), QR kod, PDF bilet üretimi | Tarayıcı + n8n |
| Gönderim | PDF ekli e-postanın otomatik iletilmesi | n8n → Gmail |
| Kayıt deposu | Kalıcı ve ortak veri saklama | n8n → Google Sheets |
| Geçiş | QR okutma, giriş/çıkış otomatik belirleme, turnike ekranı | Admin paneli → n8n |
| Raporlama | Anlık doluluk, saatlik yoğunluk, ortalama kalış süresi | Admin paneli |
| Yönlendirme | Katılımcıya panel, atölye ve proje önerisi | Yapay zekâ + yedek öneri motoru |

## 3. Projeyi ayıran yönler

**Harici kütüphane olmadan PDF üretimi.** Bilet PDF'i, bu proje için yazılmış saf JavaScript bir üreticiyle oluşturulur. Aynı kod hem tarayıcıda hem n8n Code düğümünde çalışır. Türkçe karakterler, PDF yazı tipi kodlamasında `/Differences` tablosu kullanılarak çözülmüştür. Ücretli PDF servisi veya dış bağımlılık yoktur.

**Durum bilgisi türetilir, saklanmaz.** Katılımcının içeride olup olmadığı ayrı bir alanda tutulmaz; son hareket kaydından hesaplanır. Çift okutma veya kopan bağlantı durumunda veri tutarsızlığı oluşmaz.

**Turnike deneyimi.** QR okutulduğunda ekran tam sayfa yeşile dönerek "HOŞ GELDİNİZ" ve katılımcının adını gösterir; çıkışta mavi ekranda alanda kalış süresini yazar; geçersiz bilette kırmızı uyarı verir. Sesli bip ve titreşim desteklenir, ekran altı saniye sonra kendiliğinden bekleme durumuna döner. Kapıdaki tablet için tam ekran kiosk modu vardır.

**Her tarayıcıda çalışan QR okuyucu.** Tarayıcıda yerleşik `BarcodeDetector` varsa o kullanılır, yoksa jsQR ile kare kare çözülür. Böylece Chrome, Safari, Firefox ve iOS dahil her ortamda kamera taraması çalışır.

**Üç katmanlı yapay zekâ asistanı.** Model servisine ulaşılamazsa veya kota dolarsa siteye gömülü kural tabanlı öneri motoru devreye girer. Sahada internet sorunu olsa bile asistan boş ekran göstermez.

**İki dilli arayüz.** Tüm menüler, form, SSS, iletişim, turnike ekranı ve asistan Türkçe ve İngilizce çalışır. Dil tercihi tarayıcıda saklanır.

**İstemci tarafında sır tutulmaz.** Model API anahtarı n8n'de kalır. Geçiş webhook adresi site dosyasına yazılmaz; yalnızca kapıdaki cihaza elle tanımlanır.

**Cihazdan bağımsız ortak veri.** Admin paneli hangi bilgisayardan açılırsa açılsın aynı listeyi gösterir; veri tarayıcıda değil Google Sheets'te tutulur, liste 20 saniyede bir tazelenir.

## 4. Mimari

```
Katılımcı → Web sitesi (tek dosya statik HTML, GitHub Pages)
                │
                ├── tarayıcıda: doğrulama → QR → PDF bilet (anında indirilebilir)
                │
                └── POST /webhook/ankafest-kayit ──► n8n
                                                      ├── Doğrula ve Hazırla (Code)
                                                      ├── QR Üret (HTTP Request)
                                                      ├── PDF Bilet Oluştur (Code)
                                                      ├── Gmail ile Bilet Gönder
                                                      ├── Google Sheets: Kayıt Kaydet
                                                      └── Yanıt: Kayıt Başarılı

Kapı görevlisi → Admin paneli (QR okut / kod gir)
                └── POST /webhook/ankafest-gecis ──► n8n → Sheets: Hareketler

Admin paneli (her cihaz)
                └── GET  /webhook/ankafest-liste ──► n8n → Sheets oku → ortak liste

Kariyer asistanı
                └── POST /webhook/ankafest-asistan ─► n8n → Gemini API → yanıt
                     (servis yanıt vermezse → siteye gömülü öneri motoru)
```

Site tamamen statiktir, sunucu gerektirmez; tüm iş mantığı n8n akışlarındadır. Bu ayrım sayesinde site herhangi bir statik barındırmada (GitHub Pages, Netlify, kurum sunucusu) yayınlanabilir.

**Terimler**
- **webhook**: dışarıdan HTTP isteğiyle tetiklenen n8n giriş noktası
- **Code node**: n8n içinde JavaScript çalıştıran düğüm
- **binary alan**: n8n'de dosya taşıyan alan (QR görseli ve PDF bilet)
- **CORS**: tarayıcının farklı alan adına istek atmasını düzenleyen güvenlik kuralı; webhook düğümlerinde izinli kaynak ayarıyla çözülmüştür

## 5. Ekranlar

**Kayıt.** Ad, soyad, T.C. kimlik numarası, telefon, e-posta, doğum tarihi (yaş otomatik hesaplanır), cinsiyet, şehir, katılımcı tipi, ilgi alanları ve KVKK onayı. Doğrulama alan bazındadır, hatalı alan kırmızı çerçeveyle işaretlenir. "Örnek veriyle doldur" düğmesi her basışta rastgele, algoritmaya uygun test verisi üretir.

**Biletim.** Kayıt kodu, T.C. kimlik numarası veya e-posta ile bilet yeniden görüntülenir; PDF ve PNG olarak indirilebilir, e-posta yeniden gönderilebilir.

**Program.** Dokuz panel ve atölyeden oluşan festival programı. Kariyer asistanı önerilerini bu liste üzerinden yapar.

**Kariyer Asistanı.** İlgi alanı ve bölüm bilgisine göre panel, atölye, proje fikri ve kariyer adımı önerir.

**SSS.** ANKAFEST nedir, nasıl kayıt olunur, yarışma başvuru takvimi, giriş serbestliği ve yaş sınırı.

**İletişim.** Tıklanabilir e-posta ve telefon kartları, festival alanı bilgisi.

**Admin Paneli** (PIN korumalı)
- *Canlı Özet*: toplam kayıt, anlık doluluk, günlük giriş-çıkış, saatlik yoğunluk grafiği, son hareketler, veri kaynağı durumu
- *Giriş/Çıkış Okuma*: kamera ile QR tarama, turnike ekranı, kamera seçimi, sesli uyarı, kiosk modu, elle kod girişi ve USB barkod okuyucu desteği, içeridekiler ve kalış süreleri
- *Katılımcılar*: arama, duruma göre filtreleme, tek tuşla giriş-çıkış, CSV dışa aktarma
- *Hareket Raporu*: güne göre giriş-çıkış sayıları, ortalama kalış süresi, hareket dökümü
- *Ayarlar*: webhook adresleri, kapı geçiş anahtarı, PIN, QR içerik modu, bağlantı testi

## 6. VIP katılımcılar

Katılımcı tipi olarak VIP seçilebilir. Ayrıca **Voltran Üyesiyim**, **Starlit Üyesiyim** ve **LET Üyesiyim** ilgi alanlarından birini işaretleyen katılımcı otomatik olarak VIP sayılır; bu kural hem istemcide hem n8n akışında uygulanır.

VIP biletler ayrı tasarlanır: sayfayı saran altın çerçeve, koyu üst bant, altın VIP rozeti, altın çerçeveli QR kod, "VIP Davetiye" başlığı ve ayrıcalıklar kutusu (hızlı giriş bandı, panellerde ön sıra, VIP dinlenme alanı).

## 7. Kullanılan teknolojiler

| Katman | Teknoloji | Gerekçe |
|---|---|---|
| Arayüz | Tek dosya HTML, CSS, saf JavaScript | Çerçeve ve derleme adımı yok; tek dosya her ortamda çalışır |
| QR üretimi | qrcode-generator (istemci), HTTP QR servisi (n8n) | İstemcide anında önizleme, sunucuda bilet içine gömme |
| QR okuma | BarcodeDetector, yedeğinde jsQR | Her tarayıcıda kamera desteği |
| PDF üretimi | Projeye özel saf JavaScript üretici | Ücretli servis ve kütüphane bağımlılığı yok |
| Otomasyon | n8n | Görsel akış, denetlenebilirlik, kendi sunucunda çalışabilme |
| E-posta | Gmail API (OAuth2) | Kurumsal gönderim, ek dosya desteği |
| Veri deposu | Google Sheets | Kurulum gerektirmeyen ortak depo, canlı izlenebilirlik |
| Yapay zekâ | Google Gemini (ücretsiz katman) | Ek maliyet yok; anahtar sunucu tarafında saklanır |
| Yayın | GitHub Pages | Ücretsiz, sürüm kontrolüyle bütünleşik |

## 8. Depo içeriği

| Dosya | Açıklama |
|---|---|
| `index.html` | Web sitesinin tamamı: kayıt, bilet, program, asistan, SSS, iletişim, admin paneli |
| `ankafest-n8n-workflow.json` | Ana akış: kayıt, QR, PDF, e-posta, geçiş kaydı |
| `ankafest-liste-n8n.json` | Ortak liste akışı: admin panelinin veri kaynağı |
| `ankafest-asistan-n8n.json` | Kariyer asistanı akışı |
| `ANKAFEST-kurulum.md` | Ayrıntılı kurulum ve sorun giderme kılavuzu |

## 9. Kurulum

### 9.1 Google e-tablosu

Yeni bir e-tablo oluşturun ve iki sayfa açın.

**Kayitlar** sayfası, birinci satır:
```
kayitKodu | adSoyad | tc | tcMaskeli | telefon | email | cinsiyet | yas | sehir | biletTipi | vip | ilgi | olusturma
```

**Hareketler** sayfası, birinci satır:
```
kayitKodu | adSoyad | hareket | kapi | zamanISO | tarih | saat
```

Tablo bağlantısındaki `/d/` ile `/edit` arasındaki metin tablo kimliğidir.

> Telefon numaraları `0 5XX XXX XX XX` biçiminde saklanır. Google Sheets `+` ile başlayan değerleri formül olarak yorumlayıp `#ERROR!` yazdığı için başa artı işareti konmaz.

### 9.2 n8n akışları

Üç akışı `Import from File` ile içeri aktarın.

| Akış | Webhook yolu | Metot |
|---|---|---|
| Ana akış | `/ankafest-kayit`, `/ankafest-gecis` | POST |
| Ortak liste | `/ankafest-liste` | GET |
| Asistan | `/ankafest-asistan` | POST |

Her akışta sırasıyla:
1. Gmail düğümünde OAuth2 kimlik bilgisi oluşturun (Google Cloud Console → Gmail API → OAuth istemcisi)
2. Google Sheets düğümlerinde kimlik bilgisini seçin, tablo kimliğini ve sayfa adını girin
3. Asistan akışında `x-goog-api-key` başlığına Google AI Studio'dan alınan anahtarı yazın
4. Akışları **Active** duruma getirin, Production URL'leri not alın

Doğrulama: liste webhook adresini tarayıcıya yapıştırdığınızda `{"durum":"ok","kayitlar":[...],"hareketler":[...]}` biçiminde yanıt dönmelidir.

### 9.3 Site

`index.html` dosyasının başındaki ayar bloğuna kendi adreslerinizi yazın:

```js
var VARSAYILAN = {
  kayitUrl:   "https://SUNUCUNUZ/webhook/ankafest-kayit",
  gecisUrl:   "",   // güvenlik gereği boş; kapı cihazında elle girilir
  asistanUrl: "https://SUNUCUNUZ/webhook/ankafest-asistan",
  listeUrl:   "https://SUNUCUNUZ/webhook/ankafest-liste",
  gecisKey:   "",   // kapı anahtarı dosyaya yazılmaz
  pin:        "ankafest2027",
  qrMod:      "masked"
};
```

Dosyayı depoya `index.html` adıyla yükleyin, **Settings → Pages → Branch: main → Save** ile yayınlayın.

> Kamera taraması yalnızca `https` adreslerde çalışır. GitHub Pages ve Netlify bu koşulu sağlar; dosyayı bilgisayardan çift tıklayarak açarsanız kamera izni alınamaz.

## 10. Kullanım senaryosu

**Kayıt öncesi.** Katılımcı siteye girer, formu doldurur. Hatalı T.C. numarası veya telefon anında işaretlenir. KVKK onayı verilmeden kayıt oluşturulamaz.

**Kayıt anı.** Ekranda bilet ve QR görünür, PDF indirilebilir. Aynı anda n8n tetiklenir; PDF ekli e-posta katılımcıya gider, kayıt e-tabloya yazılır.

**Festival günü, kapıda.** Görevli admin panelinin geçiş sekmesini açar, kamerayı başlatır ve kiosk moduna geçer. Okutulan her bilette turnike ekranı giriş veya çıkış kararını gösterir.

**Festival günü, yönetim.** Organizasyon ekibi canlı özet ekranından anlık doluluğu, saatlik giriş yoğunluğunu ve kalış sürelerini izler; veriler CSV olarak dışa aktarılabilir.

**Katılımcı yönlendirmesi.** Kariyer asistanı, katılımcının ilgi alanına göre programdan panel ve atölye önerir, somut proje fikri ve ilk adımı verir.

## 11. Doğrulama ve test

Geliştirme sırasında şu kontroller yapılmıştır:

- **T.C. kimlik algoritması**: geçerli ve geçersiz numaralarla sınanmıştır; 10. ve 11. hane kontrolü resmî kurala uygundur
- **PDF çıktısı**: üretilen dosyadan metin çıkarılarak Türkçe karakterlerin doğru kodlandığı, QR görselinin gömüldüğü, standart ve VIP tasarımların örtüşme olmadan yerleştiği doğrulanmıştır
- **Arayüz akışı**: boş form reddi, hatalı T.C. reddi, mükerrer kayıt engeli, bilet sorgulama, giriş-çıkış durumu değişimi, geçersiz kod reddi, rapor hesapları ve CSV dışa aktarımı otomatik testlerle sınanmıştır
- **QR okuyucu**: JSON ve düz kod okuma, aynı kodun beş saniye içinde tekrar okutulmasının engellenmesi, sekme değişiminde kameranın kapanması, izin reddi ve kamera bulunamaması gibi hata yolları sınanmıştır
- **Ortak veri**: n8n'in döndürebileceği yedi farklı yanıt biçimi (nesne, dizi, ham öğe listesi, düz satır listesi, farklı sütun adları) ile sınanmış, hepsinin doğru ayrıştırıldığı doğrulanmıştır
- **İki dillilik**: dil değişiminde menü, form, SSS, iletişim, program ve turnike ekranının çevrildiği doğrulanmıştır
- **VIP kuralı**: üyelik ilgi alanı işaretlendiğinde kaydın VIP'e yükseldiği ve VIP biletinin üretildiği sınanmıştır

## 12. Güvenlik ve KVKK

- **QR içeriği**: T.C. kimlik numarası varsayılan olarak maskelidir (`100******46`). QR kopyalanabilir bir veri olduğundan açık numara yazmak kişisel veri riski doğurur. Ayarlardan değiştirilebilir; maskeli mod önerilir.
- **Açık onay**: kayıt formunda KVKK aydınlatma onayı zorunludur.
- **Veri minimizasyonu**: QR yalnızca kapıda doğrulama için gereken alanları taşır.
- **Anahtar yönetimi**: model API anahtarı yalnızca n8n tarafındadır, istemci koduna yazılmaz.
- **Uç ayrımı**: kayıt ucu doğası gereği herkese açıktır; geçiş ucu istemciye gömülmez, yalnızca yetkili cihaza tanımlanır ve kapı anahtarı ile korunur.
- **Mükerrer kayıt**: aynı T.C. ile ikinci kayıt reddedilir.

## 13. Bilinen sınırlar

| Sınır | Etkisi | Önerilen çözüm |
|---|---|---|
| Kamera yalnızca https adreslerde açılır | Yerel dosyada tarama yapılamaz | Siteyi yayınlanmış adresten açmak |
| Google Sheets eşzamanlı yoğun yazmada yavaşlar | Çok kapılı büyük etkinliklerde gecikme | Postgres veya n8n Data Table'a geçiş |
| Gemini ücretsiz katmanı kota ve yoğunluk sınırına tabidir | Asistan bazen model yanıtı veremez | Kural tabanlı motor otomatik devreye girer |
| Kayıt ucu herkese açıktır | Sahte kayıt gönderilebilir | Oran sınırlama ve bot koruması (Turnstile, hCaptcha) |
| QR imzasızdır | Kod kopyalanarak sahte bilet üretilebilir | HMAC ile imzalama |

## 14. Yol haritası

1. **QR imzalama (HMAC)**: kayıt koduna gizli anahtarla üretilen kısa imza eklenir; geçiş ucu imzayı doğrulamayan bileti reddeder
2. **Donanım entegrasyonu**: Raspberry Pi, USB barkod okuyucu ve röle kartıyla turnike kontrolü; bağlantı koptuğunda yerel kuyruk
3. **Zamanlanmış görevler**: gün sonunda açık kalan oturumların otomatik kapatılması, kapasite aşımında yöneticiye uyarı
4. **Veritabanı geçişi**: eşzamanlı yazma güvenliği için Postgres ve kayıt kodunda tekillik kısıtı
5. **Yarışma başvuru modülü**: takım kaydı, teknik rapor yükleme ve jüri değerlendirme akışı

## 15. Ekip

| Ad | Rol |
|---|---|
| Mustafa Alp KOÇAK| Sistem tasarımı, arayüz geliştirme, otomasyon akışları |
| Sedat Cömert DEMİRTAŞ| Sistem tasarımı, arayüz geliştirme, otomasyon akışları |
| OĞuzhan ADA| Sistem tasarımı, arayüz geliştirme, otomasyon akışları |
| Ali ÖZDEMİR| Sistem tasarımı, arayüz geliştirme, otomasyon akışları |
| Neva Nur KALKAN| Sistem tasarımı, arayüz geliştirme, otomasyon akışları |
| Nil Asya KİSE| Sistem tasarımı, arayüz geliştirme, otomasyon akışları |

## 16. Lisans

Eğitim ve yarışma amaçlı geliştirilmiştir.
