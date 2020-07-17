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

 Korumalı mod birçok değişiklik getirdi, ancak ana farklardan biri bellek yönetimindeki farktır. 20 bit adres veriyolu yerine 32 bitlik adres veri yolu getirildi. 4 Gigabyte bellek ve 1 Megabayt Real Mode'a erişime izin verildi. Ayrıca, sonraki bölümlerde okuyabileceğiniz sayfalama desteği eklendi.


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

Hdr; boot_params.hdr dosyasına kopyalandıktan sonra, bir sonraki adım arch/x86/ boot/early_serial_console.c'de tanımlanan console_init işlevini çağırarak konsol başlatma işlemidir.

Komut satırında earlyprintk seçeneğini bulmaya çalışır ve arama başarılı olursa, seri bağlantı noktasının bağlantı noktası adresini ve baud hızını ayrıştırır ve seri bağlantı noktasını başlatır. earlyprintk komut satırı seçeneğinin değeri bunlardan biri olabilir:

- serial,0x3f8,115200
- serial,ttyS0,115200
- ttyS0,115200

Seri bağlantı noktası başlatıldıktan sonra ilk çıktıyı görebiliriz:

       if (cmdline_find_option_bool("debug"))
           puts("early console in setup code\n");

puts'un tty.c'de tanımlanması . Gördüğümüz gibi, putchar methodunu çağırarak bir döngüde ifadeyi karakter karakter yazdırır. Şimdi, putchar uygulanmasına bakalım:


       void __attribute__((section(".inittext"))) putchar(int ch)
       {
          if (ch == '\n')
              putchar('\r');

            bios_putchar(ch);

           if (early_serial_base != 0)
              serial_putchar(ch);
          }


__attribute __((section(".Inittext"))) bu kodun .inittext bölümünde olacağı anlamına gelir. Bunu setup.ld linker dosyasında bulabilirsiniz.


Her şeyden önce, putchar \n sembolünü denetler ve eğer bulunursa, önce \r yazdırır. Bundan sonra 0x10 kesme çağrısıyla BIOS'u çağırarak VGA ekranında karakter çıktılar:

         static void __attribute__((section(".inittext"))) bios_putchar(int ch)
         {
           struct biosregs ireg;

               initregs(&ireg);
               ireg.bx = 0x0007;
               ireg.cx = 0x0001;
               ireg.ah = 0x0e;
               ireg.al = ch;
               intcall(0x10, &ireg, NULL);
           }

Burada; initregs, biosreg yapılarını alır ve önce memset fonksiyonunu kullanarak biosreg'leri sıfırlar ve ardından kayıt değerleri ile doldurur.

    
            memset(reg, 0, sizeof *reg);
            reg->eflags |= X86_EFLAGS_CF;
            reg->ds = ds();
            reg->es = ds();
            reg->fs = fs();
            reg->gs = gs();

Memset'in uygulamasına bakalım:

           GLOBAL(memset)
             pushw   %di
             movw    %ax, %di
             movzbl  %dl, %eax
             imull   $0x01010101,%eax
             pushw   %cx
             shrw    $2, %cx
             rep; stosl
             popw    %cx
             andw    $3, %cx
             rep; stosb
             popw    %di
             retl
           ENDPROC(memset)

Yukarıda okuduğunuz gibi memcpy methodu gibi hızlı arama çağrı kurallarını kullanır; bu fonksiyon ax, dx ve cx kayıtlarından parametreler alıyor anlamına gelir.

Genellikle memset'in uygulaması bir memcpy'nin uygulaması gibidir. di yazmacının değerini stack'e kaydeder ve ax değerini biosreg yapılarının adresi olan di'ye koyar. Sonraki, dl değerini eax kaydının düşük 2 baytına kopyalanan movzbl talimatıdır. eax'in geri kalan 2 yüksek baytını sıfırlarla doldurulur.


Sonraki talimat eax'i 0x01010101 ile çarpar. Bunun nedeni, memset'in aynı anda 4 bayt kopyalamasıdır. Örneğin, bir yapıyı memset ile 0x7 ile doldurmamız gerekir. Eax, bu durumda 0x00000007 değerini içerecektir. Yani eğer eax'i 0x01010101 ile çarparsak, 0x07070707 elde edeceğiz ve şimdi bu 4 bayt'ı yapıya kopyalayabiliriz. Memset rep kullanır; eax'i es: di'ye kopyalamak için stosl talimatları.


Memset fonksiyonunun geri kalan kısmı memcpy ile hemen hemen aynıdır.


biosregs yapısı memset ile doldurulduktan sonra bios_putchar, bir karakter yazdıran 0x10 kesmesini çağırır. Ardından, seri bağlantı noktasının başlatılıp başlatılmadığını kontrol eder ve orada ayarlanmışsa seri_putchar ve inb / outb talimatlarıyla bir karakter yazar.

Heap'i Başlatmak


Stack ve bss bölümü, header.S'de (önceki bölüme bakın) hazırlandıktan sonra, çekirdeğin init_heap işlevi ile heap'i başlatması gerekiyor.


Her şeyden önce init_heap, çekirdek kurulum header'ındaki loadflags'tan CAN_USE_HEAP bayrağını denetler ve bu bayrak ayarlanmışsa heap'in sonunu hesaplar:


         char *stack_end;

          if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
             asm("leal %P1(%%esp),%0"
             : "=r" (stack_end) : "i" (-STACK_SIZE));


Veya başka bir deyişle  stack_end = esp - STACK_SIZE


Sonra heap_end hesaplaması geliyor:


           heap_end = (char *)((size_t)boot_params.hdr.heap_end_ptr + 0x200);

Heap_end_ptr veya _end + 512 (0x200h) anlamına gelir. Son kontrol, heap_end'in stack_end'den büyük olup olmadığıdır. Eğer öyleyse, stack_end, heap_end'e eşit olarak atanır.

Şimdi heap başlatılıyor ve bunu GET_HEAP methodunu kullanarak uygulayabiliriz. Nasıl kullanıldığını, nasıl kullanılacağını ve bir sonraki yazılarda nasıl uygulanacağını göreceğiz.


CPU Doğrulama

Gördüğümüz gibi bir sonraki adım, arch/x86/boot/cpu.c'den validate_cpu ile cpu doğrulamasıdır.

Check_cpu işlevini çağırır ve cpu seviyesini ve gereken cpu düzeyini ona aktarır ve çekirdeğin doğru cpu düzeyinde başlatıldığını denetler.


       check_cpu(&cpu_level, &req_level, &err_flags);
       if (cpu_level < req_level) {
           ...
           return -1;
        }


Check_cpu; cpu bayrakları, x86_64 (64-bit) işlemci durumunda uzun modun varlığını kontrol eder, işlemcinin vendor'ını kontrol eder ve eksik olduğu durumlarda AMD için SSE + SSE2'yi kapatmak gibi bazı vendor'lar için hazırlık yapar.


Belleğin Algılanması


Bir sonraki adım, detect_memory fonksiyonuyla bellek algılama yöntemidir. detect_memory temel olarak mevcut RAM'in, cpu'ya bir haritasını sağlar. 0xe820, 0xe801 ve 0x88 gibi bellek algılama için farklı programlama arabirimleri kullanır. Burada yalnızca 0xE820 uygulanmasını göreceğiz.


Arch/x86/boot/memory.c kaynak dosyasından detect_memory_e820 uygulamasına bakalım. Öncelikle, detect_memory_e820 fonksiyonu yukarıda gösterildiği gibi biosreg yapılarını başlatır ve kayıtları 0xe820 çağrısı için özel değerlerle doldurur:


       initregs(&ireg);
       ireg.ax  = 0xe820;
       ireg.cx  = sizeof buf;
       ireg.edx = SMAP;
       ireg.di  = (size_t)&buf;


- ax fonksiyonun sayısını içerir (bizim durumumuzda 0xe820)
- cx kayıt, bellek hakkında veri içeren arabellek boyutunu içerir
- edx, SMAP sihirli numarasını içermelidir
- es: di, bellek verilerini içerecek olan arabellek adresini içermelidir
- ebx sıfır olmalı.

Sonrası, bellekle ilgili verilerin toplanacağı bir döngüdür. Adres tahsisi tablosundan bir satır yazan 0x15 BIOS kesmesinin çağrısından başlar. Bir sonraki satırı elde etmek için bu kesmeyi tekrar çağırmamız gerekir (döngüde yaptığımız şey). Bir sonraki çağrının öncesinde ebx, daha önce döndürülen değeri içermelidir:

          intcall(0x15, &ireg, &oreg);
          ireg.ebx = oreg.ebx;

Sonuçta, adres tahsis tablosundan veri toplamak için döngü içinde yinelemeler yapar ve bu verileri e820entry dizisine yazar:


- Bellek segmentinin başlangıcı
- Bellek segmentinin boyutu
- Bellek segmentinin tipi (ayrılabilir, kullanılabilir vs ...).


Bunun sonucunu dmesg çıktısında görebilirsiniz:


       [    0.000000] e820: BIOS-provided physical RAM map:
       [    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
       [    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
       [    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
       [    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000003ffdffff] usable
       [    0.000000] BIOS-e820: [mem 0x000000003ffe0000-0x000000003fffffff] reserved
       [    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved


Klavyenin Başlatılması


Bir sonraki adım, keyboard_init () fonksiyonunun çağrısıyla klavyenin başlatılmasıdır. İlk önce keyboard_init, initregs fonksiyonunu kullanarak kayıt başlatır ve klavye durumunu almak için 0x16 kesmesini çağırır.


           initregs(&ireg);
        ireg.ah = 0x02;     /* Get keyboard status */
        intcall(0x16, &ireg, &oreg);
        boot_params.kbd_status = oreg.al;


Bundan sonra tekrar hızını ve gecikmesini ayarlamak için tekrar 0x16 yı çağırır.



         ireg.ax = 0x0305;   /* Set keyboard repeat rate */
       intcall(0x16, &ireg, NULL);


Sorgulama


Sonraki adımlar, farklı parametreleri sorgular. Bu sorgularla ilgili ayrıntılara girmeyeceğiz, ancak sonraki bölümlerde buna geri döneceğiz. Bu fonksiyonlara kısaca göz atalım:

Query_mca rutini, makine model numarası, alt model numarası, BIOS revizyon düzeyi ve diğer donanıma özgü özellikleri almak için 0x15 BIOS kesmesini çağırır:

       
         int query_mca(void)
           {
              struct biosregs ireg, oreg;
              u16 len;

              initregs(&ireg);
              ireg.ah = 0xc0;
              intcall(0x15, &ireg, &oreg);

              if (oreg.eflags & X86_EFLAGS_CF)
              return -1;  /* No MCA present */

              set_fs(oreg.es);
              len = rdfs16(oreg.bx);

              if (len > sizeof(boot_params.sys_desc_table))
              len = sizeof(boot_params.sys_desc_table);

              copy_from_fs(&boot_params.sys_desc_table, oreg.bx, len);
              return 0;
            }


ah register'ını 0xc0 ile doldurur ve 0x15 BIOS kesmesini çağırır. Kesmeyi çalıştırdıktan sonra sonra taşıma bayrağını denetler ve 1 olarak ayarlanırsa BIOS MCA'yı desteklemez. Taşıma bayrağı 0 olarak ayarlanırsa, ES:BX, sistem bilgi tablosuna bir işaretçi içerecektir ve şuna benzer:

        

           Offset  Size    Description
           00h    WORD    number of bytes following
           02h    BYTE    model (see #00515)
           03h    BYTE    submodel (see #00515)
           04h    BYTE    BIOS revision: 0 for first release, 1 for 2nd, etc.
           05h    BYTE    feature byte 1 (see #00510)
           06h    BYTE    feature byte 2 (see #00511)
           07h    BYTE    feature byte 3 (see #00512)
           08h    BYTE    feature byte 4 (see #00513)
           09h    BYTE    feature byte 5 (see #00514)
           ---AWARD BIOS---
           0Ah  N BYTEs   AWARD copyright notice
           ---Phoenix BIOS---
           0Ah    BYTE    ??? (00h)
           0Bh    BYTE    major version
           0Ch    BYTE    minor version (BCD)
           0Dh  4 BYTEs   ASCIZ string "PTL" (Phoenix Technologies Ltd)
           ---Quadram Quad386---
           0Ah 17 BYTEs   ASCII signature string "Quadram Quad386XT"
           ---Toshiba (Satellite Pro 435CDS at least)---
           0Ah  7 BYTEs   signature "TOSHIBA"
           11h    BYTE    ??? (8h)
           12h    BYTE    ??? (E7h) product ID??? (guess)
           13h  3 BYTEs   "JPN"


Ardından, set_fs rutini çağırır ve es kaydının değerini buna geçiririz. Set_fs'in uygulanması oldukça basit:


          static inline void set_fs(u16 seg)
          {
            asm volatile("movw %0,%%fs" : : "rm" (seg));
          }



Bu işlev, seg parametresinin değerini alır ve onu fs yazmacına koyan satır içi montaj içerir. Boot.h'de set_fs gibi birçok fonksiyon vardır, örneğin set_gs, fs, gs gibi bir değeri okumak için vs...



Query_mca sonunda yalnızca es: bx tarafından işaret edilen tabloyu boot_params.sys_desc_table'a kopyalar.


Sonraki adım, query_ist işlevini çağırarak Intel SpeedStep bilgilerini elde etmektir. Her şeyden önce CPU seviyesini kontrol eder ve doğruysa, bilgi almak için 0x15'i çağırır ve sonucu boot_params'a kaydeder.



Aşağıdaki query_apm_bios işlevi, BIOS'dan Gelişmiş Güç Yönetimi bilgisi alır. Query_apm_bios 0x15 BIOS kesmesini de çağırır, ancak ah = 0x53 ile APM kurulumunu kontrol ederek. 0x15 çalıştırılmasından sonra, query_apm_bios fonksiyonları, PM signature'ını kontrol eder (0x504d olmalı), bayrağı taşır (APM destekleniyorsa 0 olmalı) ve cx kayıt değerini (0x02 ise korumalı mod arabirimi desteklenir) kontrol eder.



Sonra tekrar 0x15'i çağırır, ancak ax = 0x5304 APM arabirimi bağlantısını keserek ve 32-bit korumalı mod arabirimini bağlayarak bunu yapar. Sonunda, boot_params.apm_bios_info dosyasını BIOS'dan alınan değerlerle doldurur.


Query_apm_bios öğesinin yalnızca yapılandırma dosyasında CONFIG_APM veya CONFIG_APM_MODULE ayarlanması durumunda çalıştırılacağını unutmayın:



      #if defined(CONFIG_APM) || defined(CONFIG_APM_MODULE)
        query_apm_bios();
      #endif


BIOS'tan Gelişmiş Disk Sürücüsü bilgilerini sorgulayan, son query_edd fonksiyonudur. Query_edd uygulamasına bakalım.


Her şeyden önce edd seçeneğini çekirdeğin komut satırından okuyor ve kapalı olarak ayarlanırsa, query_edd sadece geri dönüyor.



EDD etkinleştirilmişse, query_edd BIOS destekli sabit diskler üzerinden gider ve EDD bilgilerini aşağıdaki döngüde sorgular:



       for (devno = 0x80; devno < 0x80+EDD_MBR_SIG_MAX; devno++) {
           if (!get_edd_info(devno, &ei) && boot_params.eddbuf_entries < EDDMAXNR) {
               memcpy(edp, &ei, sizeof ei);
               edp++;
               boot_params.eddbuf_entries++;
               }
               ...
               ...
               ...


Burada ilk sabit sürücü 0x80 ve EDD_MBR_SIG_MAX makrosunun değeri 16'dır. edd_info yapıları dizisine veri toplar. get_edd_info, 0x41 kulanarak ah ile 0x13 kesmesini çağırarak EDD varlığını denetler ve EDD varsa, get_edd_info yine 0x13 kesmesini çağırır, ancak ah ile 0x48 ve si, EDD bilgilerinin depolandığı buffer adresini içerir.



Sonuç


Linux-insides hakkındaki ikinci yazının sonuna geldik.Bir sonraki bölümde, korumalı moda geçmeden önce doğrudan video modu ayarı ve hazırlıkların geri kalan kısmını göreceğiz.


Sorularınız veya önerileriniz varsa, @0xAX twitter hesabımdan, mail yoluyla veya GitHub'da isuue açarak bana ulaşabilirsiniz.

İngilizce ana dilim değil ve bu durum için özür dilerim. Herhangi bir hata bulursanız lütfen linux-inside'a PR gönderin.


Linkler


* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Protected mode](http://wiki.osdev.org/Protected_Mode)
* [Long mode](http://en.wikipedia.org/wiki/Long_mode)
* [Nice explanation of CPU Modes with code](http://www.codeproject.com/Articles/45788/The-Real-Protected-Long-mode-assembly-tutorial-for)
* [How to Use Expand Down Segments on Intel 386 and Later CPUs](http://www.sudleyplace.com/dpmione/expanddown.html)
* [earlyprintk documentation](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/earlyprintk.txt)
* [Kernel Parameters](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/kernel-parameters.rst)
* [Serial console](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/serial-console.rst)
* [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep)
* [APM](https://en.wikipedia.org/wiki/Advanced_Power_Management)
* [EDD specification](http://www.t13.org/documents/UploadedDocuments/docs2004/d1572r3-EDD3.pdf)
* [TLDP documentation for Linux Boot Process](http://www.tldp.org/HOWTO/Linux-i386-Boot-Code-HOWTO/setup.html) (old)





























