Daha onceki blog yazilarimi okuduysaniz, bir suredir low level programlamayla ilgilendigimi gorursunuz. Linux icin x86_64 Assembly programlamayla ilgili yazýlar yazdým ayný zamanda Linux kaynak koduna dalmaya basladim. low-level programlarin; nasýl isledigini, bilgisayarýmda nasýl calistigini, bellekte nasil yer aldiklarini, cekirdegin surecleri ve bellegi nasil yonettigini, network stack'in low-level'da nasil calistigini ve diger pek cok seyi anlamaya buyuk bir ilgim var. Bu nedenle, x86_64 icin Linux cekirdegi hakkýnda bir dizi yazi yazmaya karar verdim. 
Profesyonel bir cekirdek hacker'ý degilim. Cekirdek kodu yazmak benim gercek isim degil. Bu benim icin sadece bir hobi. Low-level'dan hoslaniyorum ve bunun nasil calistigini gormek ilgimi cekiyor. Bu nedenle kafa karistirici bir sey gorurseniz veya herhangi bir sorunuz/fikriniz varsa, @0xAX twitter hesabimban, mail yoluyla veya GitHub'da issue olusturarak bana ulasabilirsiniz. Buna minnettar olurum. Butun yazilarim linux-insides GitHub sayfasindan erisilebilir olacak. Ýngilizce dil bilgimle veya yazi icerigi ile ilgili bir hata fark ederseniz, Pull Request gondermekten cekinmeyin. 
Unutmayin ki; bu resmi bir dokumantasyon degildir, sadece ogrendiklerimi paylasiyorum. 
Size gerekli olan beceriler;
- C programlama dili bilgisi
- Assembly kod bilgisi (AT&T soz dizimi)
Bazi araclarý ogrenmeye baslarsaniz, yazilarim sýrasýnda bazi kýsýmlari zaten aciklamaya calisacagim. Pekala, giriþ kýsmýn burada son buluyor. Þimdi çekirdek ve low-level'a dalmaya baþlayabiliriz. 
Kodlarýn tamamý aslýnda 3.18 çekirdeði için. Deðiþiklikler olursa yazýlarýmý buna göre güncelleyeceðim. 
Sihirli Güç Düðmesi, Sonrasýnda Neler Oluyor?
Bu Linux çekirdeði ile ilgili bir dizi yazý olsa da, çekirdek kodundan baþlayacaðýz - en azýndan bu paragrafta. Dizüstü veya masaüstü bilgisayarýnýzdaki sihirli güç düðmesine bastýðýnýz anda çalýþmaya baþlar. Anakart güç kaynaðýna bir sinyal gönderiyor. Sinyali aldýktan sonra güç kaynaðý bilgisayara doðru miktarda elektrik saðlar. Anakart güç iyi sinyalini aldýktan sonra, CPU'yu baþlatmaya çalýþýr. Ýþlemci tüm kalan veriyi kayýtlarýnda sýfýrlar ve her biri için önceden tanýmlanmýþ deðerleri ayarlar.
80386 ve sonraki CPU'lar, bilgisayar sýfýrlandýktan sonra CPU kayýtlarýnda aþaðýdaki önceden tanýmlý verileri tanýmlar:

IP          0xfff0
CS selector 0xf000
CS base     0xffff0000


Ýþlemci Real Mode'da çalýþmaya baþlar. Biraz geriye dönelim ve bu modda bellek bölütlemeyi anlamaya çalýþalým. Real Mode, tüm x86 uyumlu iþlemcilerde, 8086'dan modern Intel 64 bit CPU'lara kadar desteklenir. 8086 iþlemci, 20 bitlik bir adres veri yoluna sahiptir, bu da 0-0x100000 adres alaný (1 megabayt) ile çalýþabileceði anlamýna gelir. Ancak, yalnýzca maksimum 2 ^ 16 - 1 adresine veya 0xffff (64 kilobayt) olan 16 bitlik yazmaçlara sahiptir. Bellek bölütleme, mevcut tüm adres alanýný kullanmak için kullanýlýr. Tüm bellek, 65536 bayt (64 KB) küçük, sabit boyutlu bölümlere ayrýlmýþtýr. 16 KB'lýk yazmaçlarla 64 KB'ýn üstündeki hafýzayý ele alamayacaðýmýzdan, alternatif bir yöntem tasarlanmýþtýr. Bir adres, iki bölümden oluþur: bir taban adresi olan bir Segment Selector ve bu taban adresinden bir uzaklýk. Real Mode'da, bir Segment Selector'ýn iliþkili taban adresi Segment Selector * 16'dýr. Dolayýsýyla, bellekte fiziksel bir adres almak için Segment Selector parçayý 16 ile çarpýp ofset eklemeliyiz:

PhysicalAddress = Segment Selector * 16 + Offset

Örneðin, CS: IP 0x2000: 0x0010 ise, karþýlýk gelen fiziksel adres:

>>> hex((0x2000 << 4) + 0x0010)
'0x20010'

Ancak, en büyük Segment Selector'ýný ve offsetini 0xffff:0xffff olarak alýrsak, sonuçta adres:

>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'

 Real Mode'da yalnýzca bir megabayta eriþilebildiðinden; 0x10ffef, A20'nin devre dýþý kalmasýyla 0x00ffef'e dönüþecek.

Tamam, Real Mode ve bellek adreslemeyi biliyoruz. Reset'lemeden sonra Register deðerlerini tartýþmaya geri dönelim:

CS kaydý iki bölümden oluþur: Görünür Segment Selector ve gizli taban adresi. Taban adresi genellikle 16 ile Segment Selector deðer çarpýlarak oluþturulurken, bir donaným sýfýrlama sýrasýnda CS kayýttaki Segment Selector 0xf000 ile yüklenir ve taban adresi 0xffff0000 ile yüklenir; Iþlemci, CS deðiþtirilinceye kadar bu özel taban adresini kullanýr.

Baþlangýç adresi; taban adresi, EIP kaydýndaki deðere eklenerek oluþturulmuþtur:

>>> 0xffff0000 + 0xfff0
'0xfffffff0'

4GB (16 bayt) olan 0xfffffff0 elde ediyoruz. Bu noktaya Sýfýrlama vektörü denir. Bu, CPU'nun sýfýrlamadan sonra yürütülecek ilk talimatý bulmasýný beklediði hafýza yeridir. Genellikle BIOS giriþ noktasýna iþaret eden bir atlama (jmp) yönergesi içerir. Örneðin, coreboot kaynak koduna bakarsak, þunu görürüz:


    .section ".reset"
    .code16
.globl  reset_vector
reset_vector:
    .byte  0xe9
    .int   _start - ( . + 2 )
    ...

Burada, 0xe9 olan jmp komutunun opcode'u ve _start-(. + 2) adresindeki hedef adresi görülebilir. Sýfýrlama bölümünün 16 bayt olduðunu ve 0xfffffff0'da baþladýðýný da görebiliriz.

SECTIONS {
    _ROMTOP = 0xfffffff0;
    . = _ROMTOP;
    .reset . : {
        *(.reset)
        . = 15 ;
        BYTE(0x00);
    }
}

Þimdi BIOS baþatýlýyor; BIOS'u baþlatýp denetledikten sonra BIOS'un önyüklenebilir bir aygýt bulmasý gerekir. Bir önyükleme emri, BIOS'un hangi aygýtlardan önyükleme yapmaya çalýþtýðýný kontrol eden BIOS yapýlandýrmasýnda saklanýr. BIOS bir sabit diskten önyükleme yapmaya çalýþýrken bir önyükleme sektörü bulmaya çalýþýyor. MBR bölüm düzeniyle bölünmüþ sabit sürücüler üzerinde önyükleme sektörü, her sektör 512 bayt olan ilk sektörün ilk 446 baytýnda depolanýr. Ýlk sektörün son iki baytý 0x55 ve 0xaa'dýr ve bu, BIOS'a bu aygýtýn önyüklenebilir olduðunu belirtir. Örneðin:

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

Þu komutla derleyin ve çalýþtýrýn: 

nasm -f bin boot.nasm && qemu-system-x86_64 boot

Bu, QEMU'ya yeni bir disk imajý olarak oluþturduðumuz önyükleme ikili dosyasýný kullanmasýný söyleyecektir. Yukarýdaki montaj koduyla oluþturulan ikili, önyükleme sektörünün gerekliliklerini yerine getirdiðinden (orijin 0x7c00 olarak ayarlanýr ve sihirli diziyle biter) ikili dosyayý bir disk imajýnýn ana önyükleme kaydý (MBR) olarak deðerlendirecektir.

Þunu göreceksiniz;

resim



Bu örnekte, kodun 16 bit gerçek modda yürütüleceðini ve bellekte 0x7c00'de baþlayacaðýný görebilirsiniz. Baþladýktan sonra; sadece "!" sembolünü yazdýran, 0x10 iþlemini çaðýrýr; kalan 510 bayt'ý sýfýrlarla doldurur ve iki sihirli bayt 0xaa ve 0x55 ile bitirir.

Objdump kullanarak bunun bir ikili dökümünü görebilirsiniz:

nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot

Gerçek dünyadaki bir önyükleme sektörü, önyükleme iþlemini devam ettirmek için bir koda ve bir bit sayýsý ve bir ünlem iþareti yerine bir bölüm tablosuna sahiptir :) Bu noktadan sonra, BIOS, kontrolü önyükleyiciye devreder.
NOT: Yukarýda açýklandýðý gibi, CPU Real Mode'dadýr; Real Mode'da, hafýzadaki fiziksel adresi hesaplama þu þekilde yapýlýr:

PhysicalAddress = Segment Selector * 16 + Offset

Týpký daha önce açýklandýðý gibi. Sadece 16 bit genel amaçlý kayýtlarýmýz var; 16 bitlik bir kaydýn maksimum deðeri 0xffff, yani en büyük deðerleri alýrsak sonuç þöyle olacaktýr:

>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'

Burada 0x10ffef 1MB + 64KB - 16b'ye eþittir. Bunun tersine, 8086 iþlemci (Real Mode'lu ilk iþlemci), 20 bitlik bir adres satýrýna sahiptir. 2 ^ 20 = 1048576, 1MB olduðu için, mevcut kullanýlabilir belleðin 1MB olduðu anlamýna gelir. Genel Real Mode'un hafýza haritasý aþaðýdaki gibidir:

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

Bu yazýnýn baþýnda CPU tarafýndan yürütülen ilk talimatýn 0xFFFFFFF0 adresinde olduðunu yazmýþtým, 0xFFFFFF'den (1MB) daha büyüktür. CPU Real Mode'da bu adrese nasýl eriþebilir? Cevap coreboot belgelerinde verilmiþtir:

0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space

Çalýþtýrmanýn baþlangýcýnda BIOS RAM'de deðil, ROM'da bulunur. 


Bootloader


GRUB 2 ve syslinux gibi Linux'u önyükleyebilen bir dizi önyükleyici var. Linux çekirdeðinin Linux desteði uygulamak için bir önyükleyicinin gereksinimlerini belirten bir Önyükleme protokolü vardýr. Bu örnek, GRUB 2'yi açýklayacaktýr.

BIOS, bir önyükleme aygýtý seçti ve kontrolü önyükleme kesimi koduna aktardýðý için yürütme boot.img'den baþlatýlýr. Bu kod, mevcut sýnýrlý miktarda alan nedeniyle çok basittir ve GRUB 2'nin temel görüntüsünün konumuna atlamak için kullanýlan bir iþaretçi içerir. Çekirdek imajý diskboot.img ile baþlar ve genellikle ilk bölümden hemen sonra kullanýlmayan alana ilk bölümden önce kaydedilir. Yukarýdaki kod, GRUB 2'nin çekirdeðini ve dosya sistemlerini iþlemek için kullanýlan sürücüleri içeren çekirdek görüntüsünün geri kalanýný belleðe yükler. Çekirdek imajýnýn geri kalanýný yükledikten sonra, grub_main'i çalýþtýrýr.
Grub_main konsolu baþlatýr, modüllerin temel adresini alýr, kök aygýtýný ayarlar, grub yapýlandýrma dosyasýný yükler / ayrýþtýrýr, modülleri yükler vb. Çalýþtýrma bitince, grub_main grub'ý normal moda taþýr. Grub_normal_execute (grub-core / normal / main.c'den) son hazýrlýklarý tamamlar ve bir iþletim sistemi seçmek için bir menü gösterir. Grub menü giriþlerinden birini seçtiðimizde grub_menu_execute_entry çalýþtýrýlýr, grub'ýn önyükleme komutunun çalýþtýrýlmasý ve seçilen iþletim sisteminin önyüklenmesi.
Çekirdek önyükleme protokolünü okuyabileceðimiz gibi, önyükleyici, çekirdek kurulum kodundan 0x01f1 ofsetten baþlayan çekirdek kurulum header'ýnýn bazý alanlarýný okumalý ve doldurmalýdýr. Çekirdek header'ý  arch/86/boot/header.S,  aþaðýdakilerden baþlýyor:

 .globl hdr
hdr:
    setup_sects: .byte 0
    root_flags:  .word ROOT_RDONLY
    syssize:     .long 0
    ram_size:    .word 0
    vid_mode:    .word SVGA_MODE
    root_dev:    .word 0
    boot_flag:   .word 0xAA55

Önyükleyicinin bunu komut satýrýndan alýnan ya da hesaplanan deðerlerle header'larýn geri kalanýný doldurmasý gerekir. (Çekirdek kurulum header'ýnýn tüm alanlarýna iliþkin açýklamalarý tam olarak ele almayacaðýz, bunun yerine çekirdeðin bunlarý nasýl kullandýðýný tartýþtýðýmýzda önyükleme protokolündeki tüm alanlarýn bir açýklamasýný bulabilirsiniz.)

Çekirdek önyükleme protokolünde görebileceðiniz gibi, çekirdek yüklendikten sonra bellek haritasý aþaðýdaki gibi olacaktýr:

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


Yani, önyükleyici kontrolü çekirdeðe aktardýðýnda þuradan baþlar:

X + sizeof(KernelBootSector) + 1

Burada X, çekirdek önyükleme sektörünün yüklenmekte olduðu adresidir. Benim durumumda; X,  0x10000'dýr. Bellek dökümünde görebileceðimiz gibi:

resim

Önyükleyici, Linux çekirdeðini belleðe yükledi, header alanlarýný doldurdu ve karþýlýk gelen bellek adresine atladý. Artýk doðrudan çekirdek kurulum koduna geçebiliriz.

Çekirdek Kurulumunun Baþlangýcý

Son olarak, çekirdekteyiz! Teknik olarak, çekirdek henüz çalýþmýyor; Ýlk olarak çekirdeði, bellek yöneticisini, süreç yöneticisini vb. Kurmamýz gerekir. Çekirdek kurulumunun çalýþmasý, _start'da arch / x86 / boot / header.S'den baþlar. Önceden birkaç talimat olduðundan ilk bakýþta biraz tuhaf. 

Uzun zaman önce, Linux çekirdeði kendi önyükleyicisini kullanýyordu. Ancak þimdi, þu komutu çalýþtýrýrsanýz;

qemu-system-x86_64 vmlinuz-3.18-generic


þunu göreceksiniz:


resim


Aslýnda header.S, MZ'den baþlar (yukarýdaki resme bakýn), PE hata mesajýný yazdýran ve aþaðýdaki PE header'ý:

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


UEFI ile bir iþletim sistemini yüklemek için buna ihtiyaç duyuyor. Þu anda bunun iç iþleyiþine bakmayacaðýz ve ilerleyen bölümlerde de bunu ele alacaðýz.

Gerçek çekirdek kurulum giriþ noktasý þöyledir:

// header.S line 292
.globl _start
_start:


Önyükleyici(grub2 ve diðerleri), bu noktayý (MZ'den 0x200 ofset) biliyor ve header.S'nin bir hata mesajý yazdýran .bstext bölümünden baþlamasýna raðmen, doðrudan ona bir sýçrama yapýyor:

//
// arch/x86/boot/setup.ld
//
. = 0;                    // current position
.bstext : { *(.bstext) }  // put .bstext section to position 0
.bsdata : { *(.bsdata) }


Çekirdek kurulum giriþ noktasý þöyledir:

 .globl _start
_start:
    .byte  0xeb
    .byte  start_of_setup-1f
1:
    //
    // rest of the header
    //

Burada start_of_setup-1f noktasýna atlayan bir jmp komut opcode (0xeb) görebilirsiniz. Nf gösteriminde 2f, aþaðýdaki yerel 2: etiketini belirtir; Bizim durumumuzda, atlama sonrasýnda bulunan etiket 1 ve kurulum baþlýðýnýn geri kalanýný içeriyor. Kurulum baþlýðýnýn hemen sonrasýnda, .entrytext bölümünü görüyoruz; bu bölüm, start_of_setup etiketinden baþlýyor.

Aslýnda çalýþan ilk kod budur (elbette önceki atlama talimatlarý hariç). Çekirdek kurulumu bootloader'dan kontrol aldýktan sonra, ilk jmp komutu, çekirdek gerçek Real Mode'unun baþlangýcýndan itibaren 0x200 ofset'inde, yani ilk 512 bayttan sonra yer alýr. Bu, hem Linux çekirdeði önyükleme protokolünü okuyabilir hem de grub2 kaynak kodunda görebiliriz:

segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;

Bu, çekirdek kurulumu baþladýktan sonra segment kayýtlarýnýn aþaðýdaki deðerleri içermesi anlamýna gelir:

gs = fs = es = ds = ss = 0x1000
cs = 0x1020

Benim durumumda, çekirdek 0x10000'a yüklendi.

Start_of_setup atlamasýnýn ardýndan, çekirdeðin aþaðýdakileri yapmasý gerekir:

- Tüm segment kayýt deðerlerinin eþit olduðundan emin ol
- Gerekirse doðru bir yýðýný ayarla.
- Bss'yi ayarla
- Main.c dosyasýndaki C koduna atla

Þimdi uygulamaya bakalým.


Segment registers align


Her þeyden önce, çekirdek ds ve es segment kayýtlarýnýn ayný adrese iþaret etmesini saðlar. Sonra, cld yönergesi kullanarak yön bayraðýný temizler:

 movw    %ds, %ax
    movw    %ax, %es
    cld

Daha önce yazmýþ olduðum gibi, grub2, çekirdeði kurulum kodunu 0x10000 adresinde ve cs'yi 0x1020'de yükler; çünkü çalýþtýrma dosyanýn baþýndan baþlamaz;

_start:
    .byte 0xeb
    .byte start_of_setup-1f

jump, which is at a 512 byte offset from 4d 5a. It also needs to align cs from 0x10200 to 0x10000, as well as all other segment registers. After that, we set up the stack:

pushw   %ds
    pushw   $6f
    lretw

Bu, ds deðerini 6 etiketinin adresiyle yýðýna iter ve lretw komutunu çalýþtýrýr. lretw komutu çaðýrýldýðýnda, etiket 6'nýn adresini komut gösterici kaydýna yükler ve cs'yi ds deðeriyle yükler. Daha sonra, ds ve cs ayný deðerleri alacaktýr.

Stack Kurulumu

Kurulum kodunun neredeyse tamamý gerçek modda C dil ortamýna hazýrlanýyor. Bir sonraki adým ss kayýt deðerini kontrol etmek ve eðer yanlýþsa düzeltmektir:

 movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f

Bu, 3 farklý senaryonun ortaya çýkmasýna neden olabilir:
- Ss'in geçerli deðeri 0x10000'tür (cs'in yanýndaki tüm diðer bölüm kayýtlarý gibi)
- Ss geçersiz ve CAN_USE_HEAP bayraðý ayarlanmýþ (aþaðýya bakýn)
- Ss geçersiz ve CAN_USE_HEAP bayraðý ayarlanmamýþ (aþaðýya bakýn)

Bu üç senaryonun hepsine sýrayla bakalým:

- Ss'nin doðru adrese sahip. (0x10000). Bu durumda, etiket 2'ye gidiyoruz:

2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti

Burada, dx (bootloader tarafýndan verilen sp içeriyor) 4 bayt'a ve sýfýr olup olmadýðýna iliþkin bir hizaya geldiklerini görebiliriz. Sýfýr ise, dx'e 0xfffc (64 KB'lik maksimum segment boyutundan önce 4 bayt hizalý adres) koyarýz. Sýfýr deðilse, önyükleyici (benim durumumda 0xf7f4) tarafýndan verilen sp'yi kullanmaya devam ederiz. Bundan sonra, ax deðerini 0x10000'lük doðru segment adresini ss içine yerleþtirdik ve doðru bir sp ayarladý. Artýk doðru bir yýðýnýmýz var:

resim

- Ýkinci senaryoda, (ss! = Ds). Önce, _end'in deðerini (kurulum kodunun sonundaki adres) dx'e koyar ve yýðýný kullanýp kullanamayacaðýmýzý test etmek için testb komutunu kullanarak loadflags baþlýk alanýný kontrol ederiz. Loadflags, aþaðýdaki gibi tanýmlanan bir bit maskesi header'ý dýr:

#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)

Ve önyükleme protokolünü okuyabildiðimiz için,

Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.


CAN_USE_HEAP biti ayarlanmýþsa, heap_end_ptr'ý dx'e (_end'i iþaret eder) yerleþtirir ve ona STACK_SIZE (minimum yýðýn boyutu, 512 bayt) ekleriz. Bundan sonra dx taþýnmazsa (taþýnmayacak, dx = _end + 512), etiket 2'ye atlanýr (önceki durumda olduðu gibi) ve doðru bir yýðýn oluþur.

resim


CAN_USE_HEAP ayarlanmadýðýnda _end ile _end + STACK_SIZE arasýnda en az bir yýðýn kullanýrýz:

resim

BSS Kurulumu


Ana C koduna atlayabilmemiz için gerçekleþmesi gereken son iki adým BSS alanýný kuruyor ve "sihirli" imzayý kontrol ediyor. Ýlk olarak imza kontrolü:

cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad

Bu, setup_sig'yi sihirli sayý 0x5a5aaa55 ile karþýlaþtýrýr. Eþit deðillerse ölümcül bir hata bildirilir. Sihirli sayý eþleþirse, bir dizi doðru bölüm kaydý ve bir yýðýmýz olduðunu bilerek, yalnýzca C koduna atlamadan önce BSS bölümünü kurmamýz gerekir.

BSS bölümü statik olarak ayrýlmýþ, baþlatýlmamýþ verileri depolamak için kullanýlýr. Linux dikkatli bir þekilde bu bellek alanýný aþaðýdaki kod kullanýlarak sýfýrlanmasýný saðlar:

 movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl

Ýlk olarak, __bss_start adresi di'ye taþýnýr. Daha sonra, _end + 3 adresi (+3 - 4 bayta hizalanýr) cx'e taþýnýr. Eax kaydý silinir (bir xor komutu kullanýlarak) ve bss bölüm boyutu (cx-di) hesaplanýr ve cx'e yerleþtirilir. Daha sonra, cx dörde bölünür (bir 'word' boyutu) ve stosl talimatý eax (sýfýr) deðerini di'nin gösterdiði adrese depolayarak di'yi dört arttýrarak tekrar cx'e kadar tekrarlar Sýfýra ulaþýr). Bu kodun net etkisi, sýfýrlarýn __bss_start'dan _end'e bellekteki tüm kelimeleri kullanarak yazýldýðýdýr:

resim

Main'e Atlamak 

Ýþte hepsi bu - yýðýn ve BSS'ye sahibiz, bu yüzden main() C metoduna atlayabiliriz:

 calll main

Main() metodu arch/ x86/ boot/main.c dosyasýnda bulunur. Bir sonraki bölümde bunun ne yaptýðýný okuyabilirsiniz.

Sonuç

Linux-insides hakkýndaki ilk yazýnýn sonuna geldik. Sorularýnýz veya önerileriniz varsa, @0xAX twitter hesabýmdan, mail yoluyla veya GitHub'da isuue açarak bana ulaþabilirsiniz. Sonraki bölümde Linux çekirdeði setup'ýndaki , memset, memcpy, initialprintk, konsol uygulamasý ve baþlatýlmasý gibi bellek rutinlerini uygulayan ilk C kodunu ve çok daha fazlasýný göreceðiz. Ýngilizce ana dilim deðil ve bu durum için özür dilerim. Herhangi bir hata bulursanýz lütfen linux-inside'a PR gönderin.

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

