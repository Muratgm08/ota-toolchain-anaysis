OTA Firmware ELF ve Bellek Uzayı Analizi
Bu analiz kapsamında, Contiki-NG üzerinde derlenen udp-server.z1 OTA alıcı firmware imajının yapısal icrası, bellek (RAM/Flash) yerleşimi ve araç zinciri (toolchain) çıktıları incelenmiştir.

###1. Dosyanın ELF Sınıfı, Mimarisi ve Giriş Adresi
Gömülü sisteme yüklenecek olan dosyanın kimliğini ve işlemciyle uyumluluğunu doğrulamak için msp430-readelf -h komutu kullanılmıştır.

muratgm08@Murat:~$ cd ~/contiki-ng/examples/bil304-ota

muratgm08@Murat:~/contiki-ng/examples/bil304-ota$ msp430-readelf -h new-firmware.z1

ELF Header:

  Magic:   7f 45 4c 46 01 01 01 ff 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            Standalone App
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Texas Instruments msp430 microcontroller
  Version:                           0x1
  Entry point address:               0x3100
  Start of program headers:          52 (bytes into file)
  Start of section headers:          94584 (bytes into file)
  Flags:                             0x10000001: architecture variant: : unknown: unknown extra flag bits also present
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         6
  Size of section headers:           40 (bytes)
  Number of section headers:         21
  Section header string table index: 18 


Analiz ve Yorum:

Class (ELF32): İmajın 32-bit formatında yapılandırıldığını gösterir. Bağlayıcı (linker) adresleri ve offset değerlerini bu yapıya göre ayarlar.

Data (Little Endian): Çok baytlı verilerin (örneğin 16-bit veya 32-bit değişkenlerin) belleğe yerleştirilme stratejisidir. En anlamsız baytın (LSB), bellek adreslemesinde en başa (en küçük adrese) yazıldığını belirtir. MSP430 donanım mimarisinin standart okuma formatı budur.

Type (EXEC): Bu dosyanın derlenmeyi bekleyen bir obje dosyası değil, tüm sembol çözümlenmeleri tamamlanmış ve doğrudan çalıştırılmaya hazır (Executable) bir imaj olduğunu kanıtlar.

OS/ABI (Standalone App): Sistemin standart masaüstü işletim sistemlerinden (UNIX/Linux) farklı olarak, Contiki-NG gibi bare-metal (donanıma doğrudan erişen) bir RTOS üzerinde "bağımsız bir uygulama" olarak koşacağını ifade eder.

Machine: Kodun hedef donanımının, projemizde kullandığımız Z1 Mote'ların kalbi olan "Texas Instruments MSP430" mikrokontrolcüsü olduğunu net bir şekilde doğrular.

Entry Point Address (0x3100): Sistem enerji aldığında veya resetlendiğinde (boot anında) Program Counter (PC) yazmacı doğrudan 0x3100 adresine atlar. İşletim sisteminin ve asıl C kodumuzun Flash bellek (ROM) üzerinde icra edilmeye başlandığı sıfır noktası bu adrestir.

###2. Temel Bölümlerin (.text, .data, .bss, .rodata, .vectors) Varlığı ve Anlamı
Firmware'in iç yapısının bellek uzaylarına (RAM ve Flash) nasıl dağıtılacağını görmek için msp430-readelf -S aracı ile bölüm (section) analizi yapılmıştır.

Kullanılan Komut ve Çıktısı:

$ msp430-readelf -S new-firmware.z1
Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg
  [ 1] .far.text         PROGBITS        00010000 00cff2 004a78 00  AX
  [ 2] .text             PROGBITS        00003100 0000f4 00976e 00  AX
  [ 3] .rodata           PROGBITS        0000c870 009864 0035fd 00   A
  [ 4] .data             PROGBITS        00001100 00ce62 000150 00  WA
  [ 5] .bss              NOBITS          00001250 00cfb2 001648 00  WA
  [ 6] .noinit           NOBITS          00002898 00cfb2 000002 00  WA
  [ 7] .vectors          PROGBITS        0000ffc0 00cfb2 000040 00  AX

  
#Analiz ve Bellek Yerleşimi Yorumu:
Çıktıdaki Flg (Bayrak) sütunu, bellek yöneticisine bölümün nasıl davranacağını söyler: A (Alloc): Bellekte yer kaplar, X (Execute): Çalıştırılabilir makine kodudur, W (Write): RAM'de değiştirilebilir veridir.

.text ve .far.text (Flash/ROM): Yazdığımız C kodlarının derlenmiş makine komutlarıdır (AX bayrağı). MSP430 mimarisinde standart .text bölümü (0x3100 adresinden başlar) 64KB sınırının altında kalırken; hafızaya sığmayan büyük fonksiyonlar veya bizim OTA güncellemesinde kullandığımız devasa gömülü payload'lar .far.text (0x10000 adresinden başlayan) üst bellek alanına yerleştirilir.

.rodata (Flash/ROM): Sadece okunabilir veri bölgesidir (A bayrağı, Write yoktur). 0x0c870 adresine yerleşmiştir ve içindeki sabitler (Örn: Log stringleri, 0xf17e sihirli sayısı) çalışma zamanında asla değiştirilemez.

.data (SRAM): İlk değeri yazılım aşamasında atanmış global değişkenleri barındırır (WA bayrağı - Hem yer kaplar hem yazılabilir). İmaj Flash'ta dursa bile, cihaz boot edildiğinde bu veriler 0x1100 başlangıç adresli RAM (SRAM) bölgesine kopyalanarak erişime açılır.

.bss (SRAM): İlk değeri atanmamış (veya sıfır atanmış) değişkenleri tutar. NOBITS türündedir, yani derlenmiş imajın boyutunu fiziksel olarak şişirmez. Ancak sistem boot edildiğinde 0x1250 adresinden itibaren 0x1648 baytlık devasa bir RAM alanını bloke edip içini sıfırlarla doldurur (Örneğin, gelen chunk sayılarını tuttuğumuz sayaçlar).

.noinit (SRAM): Gömülü sistemlere has özel bir bölümdür. Cihaz uyku moduna (Deep Sleep) girip çıksa veya Watchdog Timer yüzünden reset yese bile, içindeki veri (0x2898 adresinde) sıfırlanmaz. Kritik durum bilgilerini saklamak için kullanılır.

.vectors (Flash/ROM): Cihazın donanım kesme (interrupt) adreslerini barındırır (0xffc0). Radyo yongasından bir veri geldiğinde işlemci sıradaki işi bırakıp doğrudan buradaki adreslere başvurarak ISR (Interrupt Service Routine) akışını başlatır.

###Kod ve Veri Boyutları (Bellek Yükü Analizi)
Geliştirilen işletim sistemi imajının, hedef SoC'nin (Z1 Mote) fiziksel bellek kapasitelerini aşıp aşmadığını (Overflow) ve hafızada nasıl bir yük oluşturduğunu doğrulamak için msp430-size aracı kullanılmıştır.

Kullanılan Komut ve Çıktısı:
$ msp430-size new-firmware.z1
   text    data     bss     dec     hex filename
  71715     336    5706   77757   12fbd new-firmware.z1
Analiz ve Kapasite Yorumu:
Z1 Mote donanımı 92 KB ROM ve 8 KB RAM fiziksel sınırlarına sahiptir. Çıktıdaki bellek yükleri bu sınırlara göre değerlendirildiğinde:

text (71.715 Bayt - ~70 KB): Firmware'in diskte (Flash bellek / ROM) kaplayacağı net alandır. Cihazın 92 KB'lık ROM sınırının yaklaşık %76'sını doldurmaktadır. Boyutun bu kadar büyük olmasının ana sebebi, projenin doğası gereği OTA ile gönderilecek olan devasa firmware_payload dizisinin udp-client.c içerisine gömülmüş (statik olarak linklenmiş) olmasıdır. ROM kapasitesi aşılmadığı için imaj donanıma güvenle yazılabilir.

data + bss (6.042 Bayt - ~5.9 KB): Kodun çalışma zamanında (Run-time) sistemin RAM (SRAM) üzerinde bloke edeceği toplam kalıcı bellektir (336 + 5706). Cihazın 8 KB olan toplam RAM kapasitesinin büyük bir kısmı (%75'i) işgal edilmiştir.

Mühendislik Çıktısı: RAM üzerinde geriye sadece ~2 KB'lık bir alan kalmıştır. Bu kalan küçük alan, fonksiyon çağrıları, lokal değişkenler ve işletim sistemi yığıtı (Stack) için kullanılacaktır. RAM'in bu kadar kritik bir sınırda olması, OTA projemizde verileri neden tek seferde (bütün bir dizi olarak) değil de, 48 baytlık küçük chunk'lar halinde işlediğimizin en büyük teknik kanıtıdır. Eğer verileri doğrudan RAM'de birleştirmeye çalışsaydık, cihaz kesinlikle "Stack Overflow" yiyerek çökecekti. Biz bunun yerine gelen 48 baytlık paketleri RAM'de bekletmeden doğrudan CFS (Flash) üzerine yazarak sistemin ayakta kalmasını sağladık.

###4. Sembol Tablosu ve Anlamlı Semboller
Gömülü işletim sisteminin hangi kütüphaneleri kullandığını, fonksiyon ve değişkenlerin hafızada ne kadar alan kapladığını görmek için msp430-nm aracı kullanılmıştır. Sistemde en çok yer kaplayan donanım ve yazılım bloklarını analiz etmek amacıyla sembol tablosu boyutlarına göre büyükten küçüğe sıralanmıştır.

Kullanılan Komut ve Çıktısı:
$ msp430-nm --size-sort -r new-firmware.z1 | head -n 15
0000118a t input
00000df4 t output
00000992 T uip_process
00000742 T vuprintf
00000558 b frag_buf
00000550 b buframmem_memb_mem
00000454 t dio_input
00000438 T rpl_ext_header_update
000003e6 T tcpip_ipv6_output
00000344 T rpl_process_dio

Analiz ve Sembol Yorumları:
Çıktıdaki ilk sütun sembolün boyutunu (Hexadecimal bayt cinsinden), ikinci sütun bellekteki konumunu (T/t: .text/Flash, b/B: .bss/RAM), üçüncü sütun ise adını temsil etmektedir.

Ağ İşletim Sistemi Yükü (T ve t Sembolleri): Çıktıda en çok yer kaplayan fonksiyonların tamamen ağ iletişimi (IPv6 ve RPL) üzerine olduğu görülmektedir. Örneğin uip_process (0x992 = ~2.4 KB) mikro IP ağ yığınının kalbidir ve Flash bellekte ciddi bir alan kaplamaktadır. Aynı şekilde rpl_process_dio, dio_input ve tcpip_ipv6_output gibi semboller, cihazımızın bir IoT ağında yönlendirici (router) veya uç düğüm olarak görev yapabilmesi için arka planda çalışan devasa Contiki-NG ağ protokollerini ifade eder.

Formatlı Çıktı Maliyeti (vuprintf): 0x0742 (~1.8 KB) boyutundaki bu fonksiyon, kod içerisinde kullandığımız LOG_INFO ve printf tarzı string basma işlemlerinin derleyici tarafındaki karşılığıdır. Kısıtlı gömülü sistemlerde ekrana formatlı yazı basmanın ROM üzerinde ne kadar pahalı bir işlem olduğu bu sembolle kanıtlanmaktadır.

RAM Tüketimi (b Sembolleri): frag_buf (0x558 = 1368 Bayt) ve buframmem_memb_mem (1360 Bayt) sembolleri .bss segmentinde yani cihazın RAM'inde yer almaktadır. Cihazın toplam 8 KB RAM'i olduğu düşünüldüğünde, sadece gelen/giden 6LoWPAN ağ paketlerinin parçalanması (fragmentation) ve tamponlanması (buffer) işlemleri için RAM'in yaklaşık 2.7 KB'lık çok büyük bir bölümünün ayrıldığı açıkça tespit edilmiştir. OTA sürecinde neden devasa diziler kullanamayacağımızın ve bellek taşması (Stack Overflow) tehlikesinin en somut kanıtı bu sembollerdir.

###5. Kesme Vektörleri ve Başlangıç Adresi ile İlişkili Bilgiler
Sistemin donanımsal olaylara (radyodan paket gelmesi, zamanlayıcı dolması vb.) nasıl tepki verdiğini ve ilk enerjilendiğinde kodun neresinden başlayacağını nasıl bildiğini analiz etmek için objdump aracıyla kesme vektör (Interrupt Vector) tablosu incelenmiştir.

Kullanılan Komut ve Çıktısı:
$ msp430-objdump -d -j .vectors new-firmware.z1
Disassembly of section .vectors:
0000ffc0 <__ivtbl_32>:
    ffc0:       76 33 76 33 76 33 76 33 76 33 76 33 76 33 76 33 
    ffd0:       76 33 76 33 76 33 76 33 76 33 76 33 76 33 76 33 
    ffe0:       f4 36 78 37 3e 35 c2 35 76 33 76 33 76 33 ae 37 
    fff0:       24 36 8c 37 d8 37 76 33 fe 35 76 33 76 33 00 31

Analiz ve Donanım Refleksleri Yorumu:
MSP430 mimarisinde ROM belleğin en sonundaki 0xFFC0 ile 0xFFFF arasındaki 64 baytlık alan, Kesme Vektör Tablosu (Interrupt Vector Table) için donanımsal olarak rezerve edilmiştir.

Başlangıç Adresinin (Reset Vector) Kanıtı: Tablonun en sonundaki (en yüksek öncelikli) adres 0xFFFE adresidir. Çıktıda fff0 satırının en sonuna bakıldığında 00 31 baytları görülmektedir. Sistemimiz "Little Endian" formatında çalıştığı için cihaz bu baytları ters çevirerek 0x3100 olarak okur. Bu, 1. maddede bulduğumuz "Entry Point" (Giriş Adresi) değerinin ta kendisidir! Yani Z1 Mote'a pil takıldığında, donanım doğrudan 0xFFFE adresine bakar, oradaki 0x3100 adresini alır ve asıl işletim sistemini oradan boot eder.

Varsayılan (Dummy) Kesmeler: Tabloda sürekli tekrar eden 76 33 (0x3376) adresleri dikkat çekmektedir. Bunlar, cihazın kullanmadığı donanım kesmeleri için atanmış varsayılan "Dummy ISR" (Boş Kesme Servis Rutini) adresleridir. Beklenmeyen bir donanım hatası olursa, işlemci rastgele bir belleğe atlayıp çökmek yerine 0x3376 adresindeki güvenli fonksiyona yönlendirilerek sistemin stabilitesi korunur.

Aktif Donanım Kesmeleri: f4 36 (0x36f4) veya 0x3778 gibi adresler, projemizde aktif olarak kullanılan donanımların (CC2420 radyo çipi, donanım zamanlayıcıları) ISR adresleridir. Radyodan bir OTA paketi düştüğünde işlemci mevcut işini askıya alır, bu tablodaki adrese zıplar, paketi işler ve geri döner.


###6. Dosyanın Neden "Ham Binary" Değil de "ELF Executable" Olarak Değerlendirildiği
Geliştirilen işletim sistemi imajının (new-firmware.z1), doğrudan mikrokontrolcüye flaşlanan ham bir .bin (raw binary) dosyası yerine neden yapılandırılmış bir ELF (Executable and Linkable Format) dosyası olarak derlendiği/değerlendirildiği şu temel mimari ve operasyonel farklarla açıklanmaktadır:

Yapısal Harita ve Meta Veri (Headers): Ham binary dosyaları yalnızca makine kodlarından (1 ve 0'lar) oluşur; hiçbir başlık veya meta veri içermez. Cihazın ilk adresinden itibaren körlemesine çalıştırılır. Oysa ELF dosyası, içerisinde "ELF Header" ve "Section Header" gibi haritalar barındırır. Bu haritalar, bağlayıcıya (linker) ve sisteme bir kullanım kılavuzu sunarak kodun neresinin Flash belleğe (.text), neresinin RAM'e (.data veya .bss) konumlandırılması gerektiğini açıkça söyler.

Hata Ayıklama (Debugging) Gücü: Ham bir binary dosyası çöktüğünde (Crash), sistemde sadece 0x4F12 gibi anlamsız bir donanım adresinde hata alındığı görülür. Ancak ELF formatı, .symtab (Sembol Tablosu) gibi özel bölümler barındırır. Bu sayede hata ayıklayıcılar (GDB) veya Cooja simülatörü, bu adresi okuyup bize insan dilinde "Sistem udp_rx_callback fonksiyonunda çöktü" diyebilmektedir.

Simülasyon ve Cooja Uyumluluğu: Projemizi test ettiğimiz Cooja bir donanım emülatörüdür. Cooja'nın, yazdığımız C kodlarındaki satırları simülasyon ekranındaki sanal Z1 mote'ların hareketleriyle eşleştirebilmesi, logları ve bellek haritasını izleyebilmesi tamamen ELF formatının sunduğu bu zengin yapısal veriler sayesinde mümkün olmaktadır.

Sonuç (Executable Kanıtı): new-firmware.z1 dosyası, tüm kütüphaneleri bağlanmış (linked), adresleri donanıma (MSP430) göre çözümlenmiş ve Type: EXEC bayrağıyla işaretlenmiştir. Bu nedenle sadece kör bir kod yığını (ham binary) değil, Z1 Mote üzerinde çalışmaya tam olarak hazır, akıllı bir "ELF Executable" imajıdır.
