Çekirdek Önyükleme Süreci Bölüm 2

Çekirdek Kurulumunun İlk Adımları

Önceki bölümde linux çekirdeği içerisine dalmaya başladık ve çekirdek kurulum kodunun ilk bölümünü gördük. Arch/x86/boot/main.c'den main fonksiyonuna (C'de yazılan ilk fonksiyon) ilk çağrıda durduk.
Bu bölümde, çekirdek kurulum kodunu araştırmaya ve 
- korumalı modun ne olduğunu, 
- geçiş için bazı hazırlıkları, 
- heap ve konsol başlatma, 
- bellek algılama, CPU doğrulama, klavye başlatma 
- ve çok daha fazlasını göreceğiz. 

Öyleyse, devam edelim.

Korumalı Mod

Yerel Intel64 Long Mode'a geçmeden önce, çekirdek CPU'yu korumalı moda geçirmelidir.

Korumalı mod nedir? Korumalı mod, ilk olarak 1982'de x86 mimarisine eklendi ve 80286 işlemciden Intel 64'e ve Long Mode'a gelene kadar ana işlemci modlarıydı.

 Real Mode'dan uzaklaşmanın başlıca nedeni RAM'e çok sınırlı erişim olmasıdır. Önceki bölümden hatırladığınız gibi, Real Mode'da yalnızca 2 üzeri 20 byte veya 1 Megabayt, bazen sadece 640 Kilobayt RAM mevcut.

 Korumalı mod birçok değişiklik getirdi, ancak ana farklardan biri bellek yönetimindeki farktır. 20 bit adres veriyolu yerine 32 bitlik adres veri yolu getirildi. 4 Gigabyte bellek ve 1 Megabayt gerçek moda erişime izin verildi. Ayrıca, sonraki bölümlerde okuyabileceğiniz sayfalama desteği eklendi.


Korumalı modda bellek yönetimi, neredeyse bağımsız iki parçaya bölünür:

- Bölütleme
- Sayfalama
Burada sadece bölütlemeyi göreceğiz. Disk belleği bir sonraki bölümde ele alınacaktır. 

Önceki bölümde okuyabileceğiniz gibi, adresler gerçek modda iki kısımdan oluşur: 

- Segmentin taban adresi
- Segment tabanından ofset

Ve eğer bu iki parçayı biliyorsak fiziksel adresi bulabiliriz:

            PhysicalAddress = Segment Selector * 16 + Offset


Bellek bölümlemesi korumalı modda tamamen yeniden yapılandırıldı. 64 Kilobayt sabit boyutlu bölüm yok. Bunun yerine, her bir bölümün boyutu ve konumu, Bölüm Tanımlayıcıları adı verilen ilişkili bir veri yapısı tarafından tanımlanır. Bölüm tanımlayıcıları, Genel Tanımlayıcı Tablosu (GDT) adı verilen bir veri yapısında saklanır.

GDT, hafızada bulunan bir yapıdır. Belleğinde sabit bir yeri yoktur, böylece adresi GDTR özel kaydında saklanır. Daha sonra, GDT'nin Linux çekirdeği kodunda yüklendiğini göreceğiz. Belleğe yüklemek aşağıdaki gibi bir işlem olacak:

            lgdt gdt

Burada lgdt komutu global tanımlayıcı tablosunun taban adresini ve sınırını (büyüklüğü) GDTR kaydına yükler. GDTR, 48 bitlik bir kayıttır ve iki bölümden oluşur:

- Genel tanımlayıcı tablosunun boyutu (16-bit);
- Küresel tanımlayıcı tablosunun adresi (32-bit).

Yukarıda belirtildiği gibi GDT, bellek bölümlerini tanımlayan bölüm tanımlayıcıları içerir. Her tanımlayıcı 64-bit boyutundadır. Bir tanımlayıcının genel şeması şöyledir:

         31          24        19      16              7            0
         ------------------------------------------------------------
         |             | |B| |A|       | |   | |0|E|W|A|            |
         | BASE 31:24  |G|/|L|V| LIMIT |P|DPL|S|  TYPE | BASE 23:16 | 4
         |             | |D| |L| 19:16 | |   | |1|C|R|A|            |
         ------------------------------------------------------------
         |                             |                            |
         |        BASE 15:0            |       LIMIT 15:0           | 0
         |                             |                            |
         ------------------------------------------------------------

Endişelenmeyin, Real Mode'dan sonra biraz korkutucu olduğunu biliyorum, ama kolay. Örneğin LIMIT 15: 0, Tanımlayıcının 0-15 bitinin limit değeri içerdiği anlamına gelir. Geri kalan kısmı LIMIT 19: 16'da. Yani Limit boyutu 0-19, yani 20 bittir. Şimdi ona bir göz atalım:

   Sınır [20 bit] 0-15,16-19 bit. Bu, length_of_segment - 1'i tanımlar. G (Granularity) bitine bağlıdır.
   - G (bit 55) 0 ise ve segment sınırı 0 ise, segmentin boyutu 1 Byte'dir
   - G; 1 ve segment sınırı 0 ise, segmentin boyutu 4096 Byte'dir
   - G; 0 ve segment sınırı 0xfffff ise, segmentin boyutu 1 Megabayt
   - G; 1, segment sınırı 0xfffff ise, segmentin boyutu 4 Gigabyte'dır

Bu demektir ki, eğer

- G 0 ise, Limit 1 Byte terimiyle yorumlanır ve segmentin maksimum boyutu 1 Megabayt olabilir.
- G 1 ise, Limit, 4096 Bytes = 4 KBayt = 1 Sayfa olarak yorumlanır ve segmentin maksimum boyutu 4 Gigabayt olabilir. Aslında G 1 olduğunda Limit değeri 12 bit ile sola kaydırılır. Yani, 20 bit + 12 bit = 32 bit ve 232 = 4 Gigabayt.

2.Taban [32 bit] (0-15, 32-39 ve 56-63 bit) 'dedir. Segmentin başlangıç konumunun fiziksel adresini tanımlar.

3.Tür / Özellik (40-47 bit) segment türünü ve ona erişim türlerini tanımlar.

- Bit 44'teki S bayrağı tanımlayıcı türünü belirtir. S; 0 ise, bu segment bir sistem parçasıdır, oysa S 1 ise bu bir kod veya veri segmentidir (Stack segmentleri, okunması / yazılması gereken veri segmentleri olmalıdır).



Segmentin bir kod veya veri segmenti olup olmadığını belirlemek için, yukarıdaki diyagramda 0 olarak işaretlenmiş Ex (bit 43) özelliğini kontrol edebilirsiniz. 0 ise, segment bir Veri segmentidir aksi halde bir kod parçasıdır.

Bir segment aşağıdaki türlerden biri olabilir:


       |           Type Field        | Descriptor Type | Description
       |-----------------------------|-----------------|------------------
       | Decimal                     |                 |
       |             0    E    W   A |                 |
       | 0           0    0    0   0 | Data            | Read-Only
       | 1           0    0    0   1 | Data            | Read-Only, accessed
       | 2           0    0    1   0 | Data            | Read/Write
       | 3           0    0    1   1 | Data            | Read/Write, accessed
       | 4           0    1    0   0 | Data            | Read-Only, expand-down
       | 5           0    1    0   1 | Data            | Read-Only, expand-down, accessed
       | 6           0    1    1   0 | Data            | Read/Write, expand-down
       | 7           0    1    1   1 | Data            | Read/Write, expand-down, accessed
       |                  C    R   A |                 |
       | 8           1    0    0   0 | Code            | Execute-Only
       | 9           1    0    0   1 | Code            | Execute-Only, accessed
       | 10          1    0    1   0 | Code            | Execute/Read
       | 11          1    0    1   1 | Code            | Execute/Read, accessed
       | 12          1    1    0   0 | Code            | Execute-Only, conforming
       | 14          1    1    0   1 | Code            | Execute-Only, conforming, accessed
       | 13          1    1    1   0 | Code            | Execute/Read, conforming
       | 15          1    1    1   1 | Code            | Execute/Read, conforming, accessed


Gördüğümüz gibi, birinci bit (bit 43), bir veri segmenti için 0 ve bir kod parçası için 1'dir. Sonraki üç bit (40, 41, 42, 43) ya EWA (Expansion Writable Accessible) veya CRA (Conforming Readable Accessible) 'dir.

*******

1. DPL2 bitleri, 45-46 bitlerindedir. Segmentin ayrıcalık düzeyini tanımlar. 0'dan en ayrıcalıklı olan 0-3 olabilir.

2. P bayrağı (bit 47) - segmentin bellekte olup olmadığını gösterir. P 0 ise, segment yanlış olarak sunulacak ve işlemci bu segmenti okumayı reddedecektir.

3. AVL bayrağı (bit 52) - Kullanılabilir ve ayrılmış bitler. Linux'ta yok sayılır.

4. L bayrağı (bit 53) - bir kod parçasının yerel 64-bit kodu içerip içermediğini gösterir. 1 ise, kod bölümü 64 bit modunda yürütülür.

5. D / B bayrağı (bit 54) - Varsayılan / Büyük bayrak operandın boyutunu, yani 16/32 biti temsil eder. Eğer ayarlanırsa 32 bit olur aksi halde 16.


Segment kayıtları, Real Mode'da olduğu gibi segment seçicileri içerir. Bununla birlikte, korumalı modda bir segment seçici farklı şekilde ele alınır. Her Segment Tanımlayıcı, 16 bitlik bir yapıya sahip olan bir Segment Seçici'ye sahiptir:


       15              3  2   1  0
       -----------------------------
       |      Index     | TI | RPL |
       -----------------------------

- Dizin, GDT'deki tanımlayıcının dizin numarasını gösterir.
- TI (Tablo Göstergesi), tanımlayıcıyı nereden bulacağınızı gösterir. Eğer 0 ise, Genel Tanımlayıcı Tablosunda (GDT) arama yapın, aksi takdirde Yerel Tanımlayıcı Tablosuna (LDT) bakacaktır.
- Ve RPL Requester'ın Ayrıcalık Seviyesi'dir.

Her segment kaydı görünür ve gizli bir parçaya sahiptir.

- Görünür - Segment Seçici burada saklanır
- Gizli - Segment Tanımlayıcısı (taban, sınır, özellikler, bayraklar)

Fiziksel adresi korumalı modda almak için aşağıdaki adımlar gereklidir:

- The segment selector must be loaded in one of the segment registers
- The CPU tries to find a segment descriptor by GDT address + Index from selector and load the descriptor into the hidden part of the segment register
- Base address (from segment descriptor) + offset will be the linear address of the segment which is the physical address (if paging is disabled).

Şematik olarak şöyle görünecektir:


 ![alt tag](http://i.imgur.com/zBQ9aLo.png)
    

Real Mode'dan korumalı moda geçiş algoritması şöyledir:

- Kesimleri devre dışı bırak
- GDT'yi lgdt talimatı ile tanımlayın ve yükleyin
- CR0'da PE (Koruma Etkinleştir) bitini ayarlayın (Kontrol Kayıt 0)
- Korumalı mod koduna atla

Bir sonraki bölümdeki linux çekirdeğinde korumalı moda geçişin tamamını göreceğiz, ancak korumalı moda geçmeden önce biraz daha hazırlık yapmamız gerekiyor.

Şimdi arch/x86/boot/main.c'ye bakalım. Klavye başlatma, heap başlatma vb. Gerçekleştiren bazı rutinleri görebiliriz ... Şimdi bir göz atalım.


Önyükleme parametrelerini "sıfır noktası" na kopyalama


"Main.c" nin ana rutinden başlayacağız. Ana olarak adlandırılan ilk method copy_boot_params (void) 'dır. Çekirdek kurulum hearder'ını arch/x86/ include/uapi/asm/bootparam.h dosyasında tanımlanan boot_params yapısına kopyalar.

Boot_params yapısı, struct setup_header hdr alanını içerir. Bu yapı, linux önyükleme protokolünde tanımlananlarla aynı alanları içerir ve önyükleme yükleyicisi tarafından doldurulur ve ayrıca çekirdek derleme zamanında da doldurulur. Copy_boot_params iki şey yapar:

1. Hdr'yi header.S'den setup_header alanındaki boot_params yapısına kopyalar.

2. Çekirdek eski komut satırı protokolüyle yüklenmişse, çekirdek komut satırına işaretçi günceller.

Copy.S kaynak dosyasında tanımlanan memcpy işleviyle hdr'yi kopyaladığına dikkat edin. Dosyanın içeriğine bir göz atalım:


       GLOBAL(memcpy)
           pushw   %si
           pushw   %di
           movw    %ax, %di
           movw    %dx, %si
           pushw   %cx
           shrw    $2, %cx
           rep; movsl
           popw    %cx
           andw    $3, %cx
           rep; movsb
           popw    %di
           popw    %si
           retl
       ENDPROC(memcpy)


Evet, sadece C koduna geçtik ve tekrar derlemeye başladık :) Öncelikle, burada tanımlanan memcpy ve diğer yordamların iki makrosuyla başlayıp bittiğini görebilirsiniz: GLOBAL ve ENDPROC. GLOBAL, globl yönergesini ve bunun için etiketi tanımlayan arch / x86 / include / asm / linkage.h'de tanımlanmıştır. ENDPROC, include / linux / linkage.h dosyasında tanımlanmıştır ve bu isim sembolünü bir fonksiyon adı olarak işaretler ve ad sembolünün büyüklüğü ile biter.


Memcpy'nin uygulanması kolaydır. Başlangıçta, değerleri memcpy sırasında değişeceğinden değerlerini korumak için si ve di kayıtlarından yığına değerleri iter. Memcpy (ve copy.S'deki diğer işlevler) hızlı arama çağrı kurallarını kullanır. Böylece gelen parametreleri ax, dx ve cx kayıtlarından alır. Memcpy'şu şekilde çağırılır;


       memcpy(&boot_params.hdr, &hdr, sizeof hdr);


Bu yüzden,

- ax, boot_params.hdr adresini içerir
- dx, hdr adresini içerir
- cx bayt olarak hdr boyutunu içerecektir


Memcpy boot_params.hdr adresini di'ye koyar ve boyutu Stack'e kaydeder. Bundan sonra 2 boyutta sağa kayar (veya 4'te bölünür) ve si'den di'ye 4 bayt kopyalar. Bundan sonra, hdr boyutunu tekrar geri koyar, 4 bayt hizalar ve geri kalan baytları si'den bayt baytına kopyalar (daha fazlası varsa). Ve bu kopyalama işlemi bittikten sonra si ve di değerlerini sonunda desteden geri yükler.


Konsolu Başlatma














