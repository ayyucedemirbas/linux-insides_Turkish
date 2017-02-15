Daha onceki blog yazilarimi okuduysaniz, bir suredir low level programlamayla ilgilendigimi gorursunuz. Linux icin x86_64 Assembly programlamayla ilgili yazılar yazdım aynı zamanda Linux kaynak koduna dalmaya basladim. low-level programlarin; nasıl isledigini, bilgisayarımda nasıl calistigini, bellekte nasil yer aldiklarini, cekirdegin surecleri ve bellegi nasil yonettigini, network stack'in low-level'da nasil calistigini ve diger pek cok seyi anlamaya buyuk bir ilgim var. Bu nedenle, x86_64 icin Linux cekirdegi hakkında bir dizi yazi yazmaya karar verdim. 

Profesyonel bir cekirdek hacker'ı degilim. Cekirdek kodu yazmak benim gercek isim degil. Bu benim icin sadece bir hobi. Low-level'dan hoslaniyorum ve bunun nasil calistigini gormek ilgimi cekiyor. Bu nedenle kafa karistirici bir sey gorurseniz veya herhangi bir sorunuz/fikriniz varsa, @0xAX twitter hesabimban, mail yoluyla veya GitHub'da issue olusturarak bana ulasabilirsiniz. Buna minnettar olurum. Butun yazilarim linux-insides GitHub sayfasindan erisilebilir olacak. İngilizce dil bilgimle veya yazi icerigi ile ilgili bir hata fark ederseniz, Pull Request gondermekten cekinmeyin. 

Unutmayin ki; bu resmi bir dokumantasyon degildir, sadece ogrendiklerimi paylasiyorum. 

Size gerekli olan beceriler;

   - C programlama dili bilgisi
   - Assembly kod bilgisi (AT&T soz dizimi)
Bazi aracları ogrenmeye baslarsaniz, yazilarim sırasında bazi kısımlari zaten aciklamaya calisacagim. Pekala, giriş kısmın burada son buluyor. Şimdi çekirdek ve low-level'a dalmaya başlayabiliriz. 

Kodların tamamı aslında 3.18 çekirdeği için. Değişiklikler olursa yazılarımı buna göre güncelleyeceğim. 

Sihirli Güç Düğmesi, Sonrasında Neler Oluyor?

Bu Linux çekirdeği ile ilgili bir dizi yazı olsa da, çekirdek kodundan başlayacağız - en azından bu paragrafta. Dizüstü veya masaüstü bilgisayarınızdaki sihirli güç düğmesine bastığınız anda çalışmaya başlar. Anakart güç kaynağına bir sinyal gönderiyor. Sinyali aldıktan sonra güç kaynağı bilgisayara doğru miktarda elektrik sağlar. Anakart güç iyi sinyalini aldıktan sonra, CPU'yu başlatmaya çalışır. İşlemci tüm kalan veriyi kayıtlarında sıfırlar ve her biri için önceden tanımlanmış değerleri ayarlar.

80386 ve sonraki CPU'lar, bilgisayar sıfırlandıktan sonra CPU kayıtlarında aşağıdaki önceden tanımlı verileri tanımlar:


    IP          0xfff0
    CS selector 0xf000
    CS base     0xffff0000


İşlemci Real Mode'da çalışmaya başlar. Biraz geriye dönelim ve bu modda bellek bölütlemeyi anlamaya çalışalım. Real Mode, tüm x86 uyumlu işlemcilerde, 8086'dan modern Intel 64 bit CPU'lara kadar desteklenir. 8086 işlemci, 20 bitlik bir adres veri yoluna sahiptir, bu da 0-0x100000 adres alanı (1 megabayt) ile çalışabileceği anlamına gelir. Ancak, yalnızca maksimum 2 ^ 16 - 1 adresine veya 0xffff (64 kilobayt) olan 16 bitlik yazmaçlara sahiptir. Bellek bölütleme, mevcut tüm adres alanını kullanmak için kullanılır. Tüm bellek, 65536 bayt (64 KB) küçük, sabit boyutlu bölümlere ayrılmıştır. 16 KB'lık yazmaçlarla 64 KB'ın üstündeki hafızayı ele alamayacağımızdan, alternatif bir yöntem tasarlanmıştır. Bir adres, iki bölümden oluşur: bir taban adresi olan bir Segment Selector ve bu taban adresinden bir uzaklık. Real Mode'da, bir Segment Selector'ın ilişkili taban adresi Segment Selector * 16'dır. Dolayısıyla, bellekte fiziksel bir adres almak için Segment Selector parçayı 16 ile çarpıp ofset eklemeliyiz:


    PhysicalAddress = Segment Selector * 16 + Offset

Örneğin, CS: IP 0x2000: 0x0010 ise, karşılık gelen fiziksel adres:

    >>> hex((0x2000 << 4) + 0x0010)
    '0x20010'


Ancak, en büyük Segment Selector'ını ve offsetini 0xffff:0xffff olarak alırsak, sonuçta adres:

    >>> hex((0xffff << 4) + 0xffff)
    '0x10ffef'

 Real Mode'da yalnızca bir megabayta erişilebildiğinden; 0x10ffef, A20'nin devre dışı kalmasıyla 0x00ffef'e dönüşecek.


Tamam, Real Mode ve bellek adreslemeyi biliyoruz. Reset'lemeden sonra Register değerlerini tartışmaya geri dönelim: 


CS kaydı iki bölümden oluşur: Görünür Segment Selector ve gizli taban adresi. Taban adresi genellikle 16 ile Segment Selector değer çarpılarak oluşturulurken, bir donanım sıfırlama sırasında CS kayıttaki Segment Selector 0xf000 ile yüklenir ve taban adresi 0xffff0000 ile yüklenir; Işlemci, CS değiştirilinceye kadar bu özel taban adresini kullanır.


Başlangıç adresi; taban adresi, EIP kaydındaki değere eklenerek oluşturulmuştur:

    >>> 0xffff0000 + 0xfff0
    '0xfffffff0'


4GB (16 bayt) olan 0xfffffff0 elde ediyoruz. Bu noktaya Sıfırlama vektörü denir. Bu, CPU'nun sıfırlamadan sonra yürütülecek ilk talimatı bulmasını beklediği hafıza yeridir. Genellikle BIOS giriş noktasına işaret eden bir atlama (jmp) yönergesi içerir. Örneğin, coreboot kaynak koduna bakarsak, şunu görürüz:


         .section ".reset"
         .code16
     .globl  reset_vector
     reset_vector:
         .byte  0xe9
         .int   _start - ( . + 2 )
         ...


Burada, 0xe9 olan jmp komutunun opcode'u ve _start-(. + 2) adresindeki hedef adresi görülebilir. Sıfırlama bölümünün 16 bayt olduğunu ve 0xfffffff0'da başladığını da görebiliriz.


     SECTIONS {
    _     ROMTOP = 0xfffffff0;
         . = _ROMTOP;
         .reset . : {
             *(.reset)
             . = 15 ;
             BYTE(0x00);
         }
     }


Şimdi BIOS başatılıyor; BIOS'u başlatıp denetledikten sonra BIOS'un önyüklenebilir bir aygıt bulması gerekir. Bir önyükleme emri, BIOS'un hangi aygıtlardan önyükleme yapmaya çalıştığını kontrol eden BIOS yapılandırmasında saklanır. BIOS bir sabit diskten önyükleme yapmaya çalışırken bir önyükleme sektörü bulmaya çalışıyor. MBR bölüm düzeniyle bölünmüş sabit sürücüler üzerinde önyükleme sektörü, her sektör 512 bayt olan ilk sektörün ilk 446 baytında depolanır. İlk sektörün son iki baytı 0x55 ve 0xaa'dır ve bu, BIOS'a bu aygıtın önyüklenebilir olduğunu belirtir. Örneğin:

 
     ;
     ; Note: this example is written in Intel Assembly syntax
     ;
     [BITS 16]
     [ORG  0x7c00]

     boot:
         mov al, '!'
         mov ah, 0x0e
         mov bh, 0x00
         mov bl, 0x07

         int 0x10
         jmp $

     times 510-($-$$) db 0

     db 0x55
     db 0xaa


Şu komutla derleyin ve çalıştırın: 


     nasm -f bin boot.nasm && qemu-system-x86_64 boot


Bu, QEMU'ya yeni bir disk imajı olarak oluşturduğumuz önyükleme ikili dosyasını kullanmasını söyleyecektir. Yukarıdaki montaj koduyla oluşturulan ikili, önyükleme sektörünün gerekliliklerini yerine getirdiğinden (orijin 0x7c00 olarak ayarlanır ve sihirli diziyle biter) ikili dosyayı bir disk imajının ana önyükleme kaydı (MBR) olarak değerlendirecektir.

Şunu göreceksiniz;


![alt tag](https://camo.githubusercontent.com/0c5598327b189059cac641cc7bb963dfdda96cf2/687474703a2f2f6f6936302e74696e797069632e636f6d2f327162777570302e6a7067)



Bu örnekte, kodun 16 bit gerçek modda yürütüleceğini ve bellekte 0x7c00'de başlayacağını görebilirsiniz. Başladıktan sonra; sadece "!" sembolünü yazdıran, 0x10 işlemini çağırır; kalan 510 bayt'ı sıfırlarla doldurur ve iki sihirli bayt 0xaa ve 0x55 ile bitirir.


Objdump kullanarak bunun bir ikili dökümünü görebilirsiniz:

     nasm -f bin boot.nasm
     objdump -D -b binary -mi386 -Maddr16,data16,intel boot


Gerçek dünyadaki bir önyükleme sektörü, önyükleme işlemini devam ettirmek için bir koda ve bir bit sayısı ve bir ünlem işareti yerine bir bölüm tablosuna sahiptir :) Bu noktadan sonra, BIOS, kontrolü önyükleyiciye devreder.

NOT: Yukarıda açıklandığı gibi, CPU Real Mode'dadır; Real Mode'da, hafızadaki fiziksel adresi hesaplama şu şekilde yapılır:

PhysicalAddress = Segment Selector * 16 + Offset

Tıpkı daha önce açıklandığı gibi. Sadece 16 bit genel amaçlı kayıtlarımız var; 16 bitlik bir kaydın maksimum değeri 0xffff, yani en büyük değerleri alırsak sonuç şöyle olacaktır:

     >>> hex((0xffff * 16) + 0xffff)
    '0x10ffef'

Burada 0x10ffef 1MB + 64KB - 16b'ye eşittir. Bunun tersine, 8086 işlemci (Real Mode'lu ilk işlemci), 20 bitlik bir adres satırına sahiptir. 2 ^ 20 = 1048576, 1MB olduğu için, mevcut kullanılabilir belleğin 1MB olduğu anlamına gelir. Genel Real Mode'un hafıza haritası aşağıdaki gibidir:


    0x00000000 - 0x000003FF - Real Mode Interrupt Vector Table 
    0x00000400 - 0x000004FF - BIOS Data Area
    0x00000500 - 0x00007BFF - Unused
    0x00007C00 - 0x00007DFF - Our Bootloader
    0x00007E00 - 0x0009FFFF - Unused
    0x000A0000 - 0x000BFFFF - Video RAM (VRAM) Memory
    0x000B0000 - 0x000B7777 - Monochrome Video Memory
    0x000B8000 - 0x000BFFFF - Color Video Memory
    0x000C0000 - 0x000C7FFF - Video ROM BIOS
    0x000C8000 - 0x000EFFFF - BIOS Shadow Area
    0x000F0000 - 0x000FFFFF - System BIOS

Bu yazının başında CPU tarafından yürütülen ilk talimatın 0xFFFFFFF0 adresinde olduğunu yazmıştım, 0xFFFFFF'den (1MB) daha büyüktür. CPU Real Mode'da bu adrese nasıl erişebilir? Cevap coreboot belgelerinde verilmiştir:

    0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space

Çalıştırmanın başlangıcında BIOS RAM'de değil, ROM'da bulunur. 


Bootloader


GRUB 2 ve syslinux gibi Linux'u önyükleyebilen bir dizi önyükleyici var. Linux çekirdeğinin Linux desteği uygulamak için bir önyükleyicinin gereksinimlerini belirten bir Önyükleme protokolü vardır. Bu örnek, GRUB 2'yi açıklayacaktır.

BIOS, bir önyükleme aygıtı seçti ve kontrolü önyükleme kesimi koduna aktardığı için yürütme boot.img'den başlatılır. Bu kod, mevcut sınırlı miktarda alan nedeniyle çok basittir ve GRUB 2'nin temel görüntüsünün konumuna atlamak için kullanılan bir işaretçi içerir. Çekirdek imajı diskboot.img ile başlar ve genellikle ilk bölümden hemen sonra kullanılmayan alana ilk bölümden önce kaydedilir. Yukarıdaki kod, GRUB 2'nin çekirdeğini ve dosya sistemlerini işlemek için kullanılan sürücüleri içeren çekirdek görüntüsünün geri kalanını belleğe yükler. Çekirdek imajının geri kalanını yükledikten sonra, grub_main'i çalıştırır.

Grub_main konsolu başlatır, modüllerin temel adresini alır, kök aygıtını ayarlar, grub yapılandırma dosyasını yükler / ayrıştırır, modülleri yükler vb. Çalıştırma bitince, grub_main grub'ı normal moda taşır. Grub_normal_execute (grub-core / normal / main.c'den) son hazırlıkları tamamlar ve bir işletim sistemi seçmek için bir menü gösterir. Grub menü girişlerinden birini seçtiğimizde grub_menu_execute_entry çalıştırılır, grub'ın önyükleme komutunun çalıştırılması ve seçilen işletim sisteminin önyüklenmesi.

Çekirdek önyükleme protokolünü okuyabileceğimiz gibi, önyükleyici, çekirdek kurulum kodundan 0x01f1 ofsetten başlayan çekirdek kurulum header'ının bazı alanlarını okumalı ve doldurmalıdır. Çekirdek header'ı  arch/86/boot/header.S,  aşağıdakilerden başlıyor:

      .globl hdr
     hdr:
         setup_sects: .byte 0
         root_flags:  .word ROOT_RDONLY
         syssize:     .long 0
         ram_size:    .word 0
         vid_mode:    .word SVGA_MODE
         root_dev:    .word 0
         boot_flag:   .word 0xAA55

Önyükleyicinin bunu komut satırından alınan ya da hesaplanan değerlerle header'ların geri kalanını doldurması gerekir. (Çekirdek kurulum header'ının tüm alanlarına ilişkin açıklamaları tam olarak ele almayacağız, bunun yerine çekirdeğin bunları nasıl kullandığını tartıştığımızda önyükleme protokolündeki tüm alanların bir açıklamasını bulabilirsiniz.)  
 

Çekirdek önyükleme protokolünde görebileceğiniz gibi, çekirdek yüklendikten sonra bellek haritası aşağıdaki gibi olacaktır:


             | Protected-mode kernel  |
     100000   +------------------------+
             | I/O memory hole        |
     0A0000   +------------------------+
             | Reserved for BIOS      | Leave as much as possible unused
              ~                        ~
             | Command line           | (Can also be below the X+10000 mark)
             X+10000  +------------------------+
              | Stack/heap             | For use by the kernel real-mode code.

X+08000  +------------------------+
             | Kernel setup           | The kernel real-mode code.
             | Kernel boot sector     | The kernel legacy boot sector.
             X +------------------------+
             | Boot loader            | <- Boot sector entry point 0x7C00
             001000   +------------------------+
             | Reserved for MBR/BIOS  |
             000800   +------------------------+
             | Typically used by MBR  |
             000600   +------------------------+
              | BIOS use only          |
             000000   +------------------------+


Yani, önyükleyici kontrolü çekirdeğe aktardığında şuradan başlar:

     X + sizeof(KernelBootSector) + 1

Burada X, çekirdek önyükleme sektörünün yüklenmekte olduğu adresidir. Benim durumumda; X,  0x10000'dır. Bellek dökümünde görebileceğimiz gibi:


![alt tag](https://camo.githubusercontent.com/a450cde6ad2b342ce5d8a312c185f6bf80b065b6/687474703a2f2f6f6935372e74696e797069632e636f6d2f3136626b636f322e6a7067)

Önyükleyici, Linux çekirdeğini belleğe yükledi, header alanlarını doldurdu ve karşılık gelen bellek adresine atladı. Artık doğrudan çekirdek kurulum koduna geçebiliriz.


Çekirdek Kurulumunun Başlangıcı

Son olarak, çekirdekteyiz! Teknik olarak, çekirdek henüz çalışmıyor; İlk olarak çekirdeği, bellek yöneticisini, süreç yöneticisini vb. Kurmamız gerekir. Çekirdek kurulumunun çalışması, _start'da arch / x86 / boot / header.S'den başlar. Önceden birkaç talimat olduğundan ilk bakışta biraz tuhaf. 

Uzun zaman önce, Linux çekirdeği kendi önyükleyicisini kullanıyordu. Ancak şimdi, şu komutu çalıştırırsanız;

    qemu-system-x86_64 vmlinuz-3.18-generic


şunu göreceksiniz:


![alt tag](https://camo.githubusercontent.com/cff3757c397b6d4ebf2706a874a729cad5eefaef/687474703a2f2f6f6936302e74696e797069632e636f6d2f723032786b7a2e6a7067)


Aslında header.S, MZ'den başlar (yukarıdaki resme bakın), PE hata mesajını yazdıran ve aşağıdaki PE header'ı:

     #ifdef CONFIG_EFI_STUB
     # "MZ", MS-DOS header
     .byte 0x4d
     .byte 0x5a
     #endif
     ...
     ...
     ...
     pe_header:
        .ascii "PE"
        .word 0


UEFI ile bir işletim sistemini yüklemek için buna ihtiyaç duyuyor. Şu anda bunun iç işleyişine bakmayacağız ve ilerleyen bölümlerde de bunu ele alacağız.

Gerçek çekirdek kurulum giriş noktası şöyledir:

     // header.S line 292
     .globl _start
     _start:


Önyükleyici(grub2 ve diğerleri), bu noktayı (MZ'den 0x200 ofset) biliyor ve header.S'nin bir hata mesajı yazdıran .bstext bölümünden başlamasına rağmen, doğrudan ona bir sıçrama yapıyor:

     //
     // arch/x86/boot/setup.ld
     //
     . = 0;                    // current position
      .bstext : { *(.bstext) }  // put .bstext section to position 0
     .bsdata : { *(.bsdata) }


Çekirdek kurulum giriş noktası şöyledir:

    .globl _start
    _start:
      .byte  0xeb
      .byte  start_of_setup-1f
     1:
        //
       // rest of the header
       //

Burada start_of_setup-1f noktasına atlayan bir jmp komut opcode (0xeb) görebilirsiniz. Nf gösteriminde 2f, aşağıdaki yerel 2: etiketini belirtir; Bizim durumumuzda, atlama sonrasında bulunan etiket 1 ve kurulum başlığının geri kalanını içeriyor. Kurulum başlığının hemen sonrasında, .entrytext bölümünü görüyoruz; bu bölüm, start_of_setup etiketinden başlıyor.

Aslında çalışan ilk kod budur (elbette önceki atlama talimatları hariç). Çekirdek kurulumu bootloader'dan kontrol aldıktan sonra, ilk jmp komutu, çekirdek gerçek Real Mode'unun başlangıcından itibaren 0x200 ofset'inde, yani ilk 512 bayttan sonra yer alır. Bu, hem Linux çekirdeği önyükleme protokolünü okuyabilir hem de grub2 kaynak kodunda görebiliriz:

     segment = grub_linux_real_target >> 4;
     state.gs = state.fs = state.es = state.ds = state.ss = segment;
      state.cs = segment + 0x20;

Bu, çekirdek kurulumu başladıktan sonra segment kayıtlarının aşağıdaki değerleri içermesi anlamına gelir:
 
     gs = fs = es = ds = ss = 0x1000
     cs = 0x1020

Benim durumumda, çekirdek 0x10000'a yüklendi.

Start_of_setup atlamasının ardından, çekirdeğin aşağıdakileri yapması gerekir:

- Tüm segment kayıt değerlerinin eşit olduğundan emin ol
- Gerekirse doğru bir yığını ayarla.
- Bss'yi ayarla
- Main.c dosyasındaki C koduna atla

Şimdi uygulamaya bakalım.

 
Segment registers align


Her şeyden önce, çekirdek ds ve es segment kayıtlarının aynı adrese işaret etmesini sağlar. Sonra, cld yönergesi kullanarak yön bayrağını temizler:

    movw    %ds, %ax
    movw    %ax, %es
    cld

Daha önce yazmış olduğum gibi, grub2, çekirdeği kurulum kodunu 0x10000 adresinde ve cs'yi 0x1020'de yükler; çünkü çalıştırma dosyanın başından başlamaz;

    _start:
    .byte 0xeb
    .byte start_of_setup-1f
    jump, which is at a 512 byte offset from 4d 5a. It also needs to align cs from 0x10200 to 0x10000, as well as all other segment registers. After that, we set up the stack:


    pushw   %ds
    pushw   $6f
    lretw

Bu, ds değerini 6 etiketinin adresiyle yığına iter ve lretw komutunu çalıştırır. lretw komutu çağırıldığında, etiket 6'nın adresini komut gösterici kaydına yükler ve cs'yi ds değeriyle yükler. Daha sonra, ds ve cs aynı değerleri alacaktır.
 
Stack Kurulumu

Kurulum kodunun neredeyse tamamı gerçek modda C dil ortamına hazırlanıyor. Bir sonraki adım ss kayıt değerini kontrol etmek ve eğer yanlışsa düzeltmektir:

    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f

Bu, 3 farklı senaryonun ortaya çıkmasına neden olabilir:

- Ss'in geçerli değeri 0x10000'tür (cs'in yanındaki tüm diğer bölüm kayıtları gibi)
- Ss geçersiz ve CAN_USE_HEAP bayrağı ayarlanmış (aşağıya bakın)
- Ss geçersiz ve CAN_USE_HEAP bayrağı ayarlanmamış (aşağıya bakın)

Bu üç senaryonun hepsine sırayla bakalım:

- Ss'nin doğru adrese sahip. (0x10000). Bu durumda, etiket 2'ye gidiyoruz:

     2:  andw    $~3, %dx
     jnz     3f
     movw    $0xfffc, %dx
    3:  movw    %ax, %ss
     movzwl  %dx, %esp
    sti

Burada, dx (bootloader tarafından verilen sp içeriyor) 4 bayt'a ve sıfır olup olmadığına ilişkin bir hizaya geldiklerini görebiliriz. Sıfır ise, dx'e 0xfffc (64 KB'lik maksimum segment boyutundan önce 4 bayt hizalı adres) koyarız. Sıfır değilse, önyükleyici (benim durumumda 0xf7f4) tarafından verilen sp'yi kullanmaya devam ederiz. Bundan sonra, ax değerini 0x10000'lük doğru segment adresini ss içine yerleştirdik ve doğru bir sp ayarladı. Artık doğru bir yığınımız var:

![alt tag](https://camo.githubusercontent.com/fd6508ced7cf2bc23a5401388f47c4bf75ffa53b/687474703a2f2f6f6935382e74696e797069632e636f6d2f313669776369732e6a7067)

- İkinci senaryoda, (ss! = Ds). Önce, _end'in değerini (kurulum kodunun sonundaki adres) dx'e koyar ve yığını kullanıp kullanamayacağımızı test etmek için testb komutunu kullanarak loadflags başlık alanını kontrol ederiz. Loadflags, aşağıdaki gibi tanımlanan bir bit maskesi header'ı dır:

 
  //#define LOADED_HIGH     (1<<0)
  
  //#define QUIET_FLAG      (1<<5)
  
  //#define KEEP_SEGMENTS   (1<<6)
  
  //#define CAN_USE_HEAP    (1<<7)

Ve önyükleme protokolünü okuyabildiğimiz için,

Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.


CAN_USE_HEAP biti ayarlanmışsa, heap_end_ptr'ı dx'e (_end'i işaret eder) yerleştirir ve ona STACK_SIZE (minimum yığın boyutu, 512 bayt) ekleriz. Bundan sonra dx taşınmazsa (taşınmayacak, dx = _end + 512), etiket 2'ye atlanır (önceki durumda olduğu gibi) ve doğru bir yığın oluşur.



 ![alt tag](https://camo.githubusercontent.com/49e2c3b70d081f745bc6e53f72df96b806e27cb6/687474703a2f2f6f6936322e74696e797069632e636f6d2f6472376235772e6a7067)


CAN_USE_HEAP ayarlanmadığında _end ile _end + STACK_SIZE arasında en az bir yığın kullanırız:

![alt tag](https://camo.githubusercontent.com/33f54e2576edfb855cc90f4669fd49c5556a1aa6/687474703a2f2f6f6936302e74696e797069632e636f6d2f323877303531792e6a7067)

BSS Kurulumu


Ana C koduna atlayabilmemiz için gerçekleşmesi gereken son iki adım BSS alanını kuruyor ve "sihirli" imzayı kontrol ediyor. İlk olarak imza kontrolü:

   cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad

Bu, setup_sig'yi sihirli sayı 0x5a5aaa55 ile karşılaştırır. Eşit değillerse ölümcül bir hata bildirilir. Sihirli sayı eşleşirse, bir dizi doğru bölüm kaydı ve bir yığımız olduğunu bilerek, yalnızca C koduna atlamadan önce BSS bölümünü kurmamız gerekir.

BSS bölümü statik olarak ayrılmış, başlatılmamış verileri depolamak için kullanılır. Linux dikkatli bir şekilde bu bellek alanını aşağıdaki kod kullanılarak sıfırlanmasını sağlar:

 movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl

İlk olarak, __bss_start adresi di'ye taşınır. Daha sonra, _end + 3 adresi (+3 - 4 bayta hizalanır) cx'e taşınır. Eax kaydı silinir (bir xor komutu kullanılarak) ve bss bölüm boyutu (cx-di) hesaplanır ve cx'e yerleştirilir. Daha sonra, cx dörde bölünür (bir 'word' boyutu) ve stosl talimatı eax (sıfır) değerini di'nin gösterdiği adrese depolayarak di'yi dört arttırarak tekrar cx'e kadar tekrarlar Sıfıra ulaşır). Bu kodun net etkisi, sıfırların __bss_start'dan _end'e bellekteki tüm kelimeleri kullanarak yazıldığıdır:


![alt tag](https://camo.githubusercontent.com/76b4eece266aae9a5e435c07744abdc2fb7fd6fa/687474703a2f2f6f6935392e74696e797069632e636f6d2f32396d326579722e6a7067)


Main'e Atlamak 

İşte hepsi bu - yığın ve BSS'ye sahibiz, bu yüzden main() C metoduna atlayabiliriz:

 calll main

Main() metodu arch/ x86/ boot/main.c dosyasında bulunur. Bir sonraki bölümde bunun ne yaptığını okuyabilirsiniz.

Sonuç

Linux-insides hakkındaki ilk yazının sonuna geldik. Sorularınız veya önerileriniz varsa, @0xAX twitter hesabımdan, mail yoluyla veya GitHub'da isuue açarak bana ulaşabilirsiniz. Sonraki bölümde Linux çekirdeği setup'ındaki , memset, memcpy, initialprintk, konsol uygulaması ve başlatılması gibi bellek rutinlerini uygulayan ilk C kodunu ve çok daha fazlasını göreceğiz. İngilizce ana dilim değil ve bu durum için özür dilerim. Herhangi bir hata bulursanız lütfen linux-inside'a PR gönderin.

Linkler

- Intel 80386 programmer's reference manual 1986
- Minimal Boot Loader for Intel® Architecture
- 8086
- 80386
- Reset vector
- Real mode
- Linux kernel boot protocol
- CoreBoot developer manual
- Ralf Brown's Interrupt List
- Power supply
- Power good signal


