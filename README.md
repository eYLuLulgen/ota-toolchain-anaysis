#  OTA Firmware Araç Zinciri ve Statik İkili (Binary) Analiz Raporu

Bu depo, BIL304 İşletim Sistemleri dersi kapsamında gerçekleştirilen `new-firmware.z1` dosyasının statik ikili analiz sonuçlarını içermektedir. Analiz kapsamında, Contiki-NG mimarisi altındaki firmware'in donanım özellikleri, bellek yerleşimi ve gömülü sembol referansları incelenmiştir.

---

## 1. Analiz Ortamı ve Kullanılan Araçlar
* **Ana Bilgisayar Ortamı:** Windows Subsystem for Linux (WSL2) - Ubuntu LTS
* **Kullanılan Analiz Araçları:** GNU Binutils (`readelf`, `strings`)
* **Analiz Edilen Dosya:** `new-firmware.z1`
* **Hedef Cihaz Profili:** Zolertia Z1 (MSP430 tabanlı düşük güçlü IoT düğümü)

---

##  2. İkili Kimlik ve Format Analizi (readelf -h)
`readelf -h` komutu ile firmware başlığı (ELF Header) incelenmiş ve derleme mimarisine ait temel kısıtlamalar şu şekilde tespit edilmiştir:

* **Sihirli Sayılar (Magic Bytes):** `7f 45 4c 46` — Bu bayt dizisi, dosyanın standart ELF (Executable and Linkable Format) yapısında olduğunu doğrular.
* **Sınıf ve Veri Yapısı (Class & Data):** `ELF32`, `2's complement, little endian`. Sistemin 32-bit mimari düzeninde derlendiğini ve sayısal verilerin belleğe en düşük anlamlı bayttan (LSB) başlanarak yerleştirildiğini gösterir.
* **Hedef İşlemci Mimarisi (Machine):** `Texas Instruments msp430`. Bu firmware'in doğrudan MSP430 mikrokontrolcü ailesi için çapraz derlendiğini (cross-compile) kanıtlar.
* **Giriş Noktası Adresi (Entry Point Address):** `0x3100`. Cihaz boot edildiğinde, işlemcinin Program Sayacının (PC) yürütmeye başlayacağı ilk somut bellek sınırıdır.
* **Sembol Tablosu Durumu (Symbol Table):** Dosya **stripped edilmemiştir (ayıklanmamıştır)**; tüm fonksiyon isimleri, değişken haritaları ve debug sembolleri binary içinde korunmuştur.

---

##  3. Bellek Haritası ve Bölüt Analizi (readelf -S)
Section header (bölüt başlıkları) analizi, firmware'in çalıştırılabilir kodları, sabit parametreleri ve çalışma zamanı RAM alanlarını nasıl paylaştırdığını ortaya koymaktadır:

* **`.text` (Kod Bölütü):** Ana çalıştırılabilir döngü yönergelerini barındırır. Başlangıç adresi: `0x3100`, Boyut: **38.766 bayt** (Flash/ROM bellekte saklanır).
* **`.far.text` (Uzak Kod Bölütü):** Standart 16-bit adres sınırlarının dışındaki genişletilmiş bellek segmenti işlemlerini yönetir. Boyut: **19.064 bayt** (Flash/ROM bellekte saklanır).
* **`.rodata` (Salt Okunur Veri):** Değiştirilemeyen sabit metin ifadelerini ve yapılandırma dizilerini tutar. Boyut: **13.821 bayt** (Flash/ROM bellekte saklanır).
* **`.data` (İlk Değer Atanmış Veri):** Sıfırdan farklı bir başlangıç değeriyle tanımlanmış çalışma zamanı değişkenleridir. Boyut: **336 bayt** (Hem Flash hem de RAM üzerinde yer kaplar).
* **`.bss` (İlk Değer Atanmamış Veri):** Varsayılan değeri sıfır olan statik ve global değişkenleri tutar. Boyut: **5.704 bayt** (Diskte yer kaplamaz, cihaz açıldığında doğrudan **RAM** üzerinde ayrılır).

###  Donanım Bellek Tüketim Hesaplamaları:
Elde edilen bayt boyutları doğrultusunda, firmware'in hedef donanım (Z1) üzerindeki toplam bellek maliyeti şu formülle hesaplanmıştır:

* **Toplam Flash (ROM) Tüketimi:** `.text` + `.far.text` + `.rodata` + `.data` + `.vectors`
  $$\text{Flash Kullanımı} = 38.766 + 19.064 + 13.821 + 336 + 64 = 72.051 \text{ bayt } (\sim 70.3 \text{ KB})$$
* **Toplam Statik RAM Tüketimi:** `.data` + `.bss` + `.noinit`
  $$\text{RAM Kullanımı} = 336 + 5.704 + 2 = 6.042 \text{ bayt } (\sim 5.9 \text{ KB})$$

---

##  4. Sembolik ve Ağ Yığını Analizi (readelf -s)
Sembol tablosu incelendiğinde dosya içinde **1143 adet bağımsız sembol** tespit edilmiştir. Contiki-NG işletim sisteminin IPv6 paket sıkıştırma ve ağ katmanını yöneten `sicslowpan.c` modülüne ait kritik girdiler şu şekildedir:

* **`frag_buf` (OBJECT):** `0x00001d0c` adresinde yer alan ve **1368 bayt** gibi oldukça büyük bir yer kaplayan veri yapısıdır. İndeks değeri `.bss` bölütünü işaret ettiği için doğrudan **RAM** bellek üzerinde konumlanır. Görevi, kablosuz ağ üzerinden (Over-the-Air) parça parça gelen (fragmented) 6LoWPAN paketlerini üst ağ katmanlarına iletmeden önce RAM üzerinde birleştirmektir.
* **`clear_fragments` (FUNC):** `.text` (Flash) bölütü içinde `0x0000abf2` adresinde haritalanmıştır. Paket birleştirme işlemi bittiğinde veya zaman aşımı yaşandığında RAM'deki `frag_buf` alanını temizleyerek bellek sızıntısını (memory leak) engelleyen 56 baytlık bir fonksiyondur.
* **`output` (FUNC):** `.text` (Flash) içinde `0x0000b1cc` adresinde yer alan, 3572 bayt boyutunda geniş bir kod alanına sahip ana paket çıkış fonksiyonudur. Ağ katmanından çıkan çerçevelerin MAC katmanına iletilmesini yönetir.

---

##  5. Güvenlik ve Dayanıklılık (Robustness) Değerlendirmesi
* **Bilgi Sızıntısı Riski:** `strings` komutu koşturulduğunda, binary içerisinden `App: Received request` gibi çok sayıda geliştirici logu ve modül etiketi okunabilmektedir. Bu durum tersine mühendislik faaliyetlerini kolaylaştırarak ağ mimarisinin dışarıya sızmasına neden olabilir.
* **OTA Paket Bütünlüğü:** Ağ katmanında parçalanmış paketleri toplamak için ayrılan büyük bellek alanı (`frag_buf`), kötü n
