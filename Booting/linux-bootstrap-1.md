Daha önceki blog yazılarımı okuduysanız, bir süredir low level programlamayla ilgilendiğimi görürsünüz. Linux icin x86_64 Assembly programlamayla ilgili yazılar yazdım aynı zamanda Linux kaynak koduna dalmaya başladım. low-level programların; nasıl işlediğini, bilgisayarımda nasıl çalıştığını, bellekte nasıl yer aldıklarını, çekirdeğin süreçleri ve bellegi nasıl yönettiğini, network stack'in low-level'da nasıl çalıştığını ve diğer pek çok şeyi anlamaya büyük bir ilgim var. Bu nedenle, x86_64 icin Linux çekirdegi hakkında bir dizi yazı yazmaya karar verdim. 

Profesyonel bir çekirdek hacker'ı değilim. Çekirdek kodu yazmak benim gerçek işim değil. Bu benim için sadece bir hobi. Low-level'dan hoşlanıyorum ve bunun nasıl çalıştığını görmek ilgimi çekiyor. Bu nedenle kafa karıştırıcı bir sey görürseniz veya herhangi bir sorunuz/fikriniz varsa, [@0xAX](https://twitter.com/0xAX) Twitter hesabımdan, [e-posta](anotherworldofworld@gmail.com) yoluyla veya GitHub'da [issue](https://github.com/0xAX/linux-internals/issues/new) oluşturarak bana ulaşabilirsiniz. Buna minnettar olurum. Bütün yazılarım linux-insides GitHub sayfasından erişilebilir olacak. İngilizce dil bilgimle veya yazı içeriği ile ilgili bir hata fark ederseniz, Pull Request gondermekten çekinmeyin. 

Unutmayın ki; bu resmi bir dokümantasyon değildir, sadece öğrendiklerimi paylaşıyorum. 

Size gerekli olan beceriler;

   - C programlama dili bilgisi
   - Assembly kod bilgisi (AT&T soz dizimi)
Bazı araçları öğrenmeye başlarsanız, yazılarım sırasında bazı kısımları zaten açıklamaya çalışacağım. Pekala, giriş kısmı burada son buluyor. Şimdi çekirdek ve low-level'a dalmaya başlayabiliriz. 

Kodların tamamı aslında 3.18 çekirdeği için. Değişiklikler olursa yazılarımı buna göre güncelleyeceğim. 

Sihirli Güç Düğmesi, Sonrasında Neler Oluyor?

Bu Linux çekirdeği ile ilgili bir dizi yazı olsa da, çekirdek kodundan başlayacağız - en azından bu paragrafta. Dizüstü veya masaüstü bilgisayarınızdaki sihirli güç düğmesine bastığınız anda çalışmaya başlar. Anakart güç kaynağına bir sinyal gönderiyor. Sinyali aldıktan sonra güç kaynağı bilgisayara doğru miktarda elektrik sağlar. Anakart güç iyi sinyalini aldıktan sonra, CPU'yu başlatmaya çalışır. İşlemci tüm kalan veriyi kayıtlarında sıfırlar ve her biri için önceden tanımlanmış değerleri ayarlar.

80386 ve sonraki CPU'lar, bilgisayar sıfırlandıktan sonra CPU kayıtlarında aşağıdaki önceden tanımlı verileri tanımlar:

```
IP          0xfff0
CS selector 0xf000
CS base     0xffff0000
```

İşlemci Real Mode'da çalışmaya başlar. Biraz geriye dönelim ve bu modda bellek bölütlemeyi anlamaya çalışalım. Real Mode, tüm x86 uyumlu işlemcilerde, 8086'dan modern Intel 64 bit CPU'lara kadar desteklenir. 8086 işlemci, 20 bitlik bir adres veri yoluna sahiptir, bu da 0-0x100000 adres alanı (1 megabayt) ile çalışabileceği anlamına gelir. Ancak, yalnızca maksimum 2 ^ 16 - 1 adresine veya 0xffff (64 kilobayt) olan 16 bitlik yazmaçlara sahiptir. Bellek bölütleme, mevcut tüm adres alanını kullanmak için kullanılır. Tüm bellek, 65536 bayt (64 KB) küçük, sabit boyutlu bölümlere ayrılmıştır. 16 KB'lık yazmaçlarla 64 KB'ın üstündeki hafızayı ele alamayacağımızdan, alternatif bir yöntem tasarlanmıştır. Bir adres, iki bölümden oluşur: bir taban adresi olan bir Segment Selector ve bu taban adresinden bir uzaklık. Real Mode'da, bir Segment Selector'ın ilişkili taban adresi Segment Selector * 16'dır. Dolayısıyla, bellekte fiziksel bir adres almak için Segment Selector parçayı 16 ile çarpıp ofset eklemeliyiz:

```
PhysicalAddress = Segment Selector * 16 + Offset
```
Örneğin, ```CS:IP``` ```0x2000:0x0010``` ise, karşılık gelen fiziksel adres:
   
```
python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```
Ancak, en büyük Segment Selector'ını ve offsetini ```0xffff:0xffff``` olarak alırsak, sonuçta adres:

```
python
>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'
```
	
 ilk megabaytı geçen ```65520``` baytıdır. Gerçek modda yalnızca bir megabayt erişilebilir olduğundan, ```0x10ffef```, [A20 satırı](https://en.wikipedia.org/wiki/A20_line) devre dışı bırakıldığında ```0x00ffef``` olur.	

Tamam, Real Mode ve bellek adreslemeyi biliyoruz. Reset'lemeden sonra Register değerlerini tartışmaya geri dönelim: 


```CS``` kaydı iki bölümden oluşur: Görünür Segment Selector ve gizli taban adresi. Taban adresi genellikle 16 ile Segment Selector değer çarpılarak oluşturulurken, bir donanım sıfırlama sırasında CS kayıttaki Segment Selector ```0xf000``` ile yüklenir ve taban adresi ```0xffff0000``` ile yüklenir; İşlemci, ```CS``` değiştirilinceye kadar bu özel taban adresini kullanır.


Başlangıç adresi; taban adresi, EIP kaydındaki değere eklenerek oluşturulmuştur:

```
python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

4GB (16 bayt) olan ```0xfffffff0``` elde ediyoruz. Bu noktaya [reset vektörü](https://en.wikipedia.org/Reset_vector) denir. Bu, CPU'nun sıfırlamadan sonra yürütülecek ilk talimatı bulmasını beklediği hafıza yeridir. Genellikle [BIOS](https://en.wikipedia.org/wiki/BIOS) giriş noktasına işaret eden bir atlama (```jmp```) yönergesi içerir. Örneğin, [coreboot](https://www.coreboot.org) kaynak koduna (```src/cpu/x86/16bit/reset16.inc```) bakarsak, şunu görürüz:
        
```	
    /* SPDX-License-Identifier: GPL-2.0-only */

	.section ".reset", "ax", %progbits
	.code16
.globl	_start
_start:
	.byte  0xe9
	.int   _start16bit - ( . + 2 )

     /* Note: The above jump is hand coded to work around bugs in binutils.
	 * 5 byte are used for a 3 byte instruction.  This works because x86
	 * is little endian and allows us to use supported 32bit relocations
	 * instead of the weird 16 bit relocations that binutils does not
	 * handle consistently between versions because they are used so rarely.
	 */
	.previous
```
		 
Burada, ```0xe9``` olan ```jmp``` komutunun [opcode'u](http://ref.x86asm.net/coder32.html$xE9) ve ```_start-(. + 2)``` adresindeki hedef adresi görülebilir. 

```sıfırlama``` bölümünün ```16``` bayt olduğunu ve ```0xfffffff0``` (```src/cpu/x86/16bit/reset16.ld```) adresinden başlayarak derlendiğini görebiliriz.

```
SECTIONS {
    /* Trigger an error if I have an unuseable start address */
    _bogus = ASSERT(_start16bit >= 0xffff0000, "_start16bit too low. Please report.");
    _ROMTOP = 0xfffffff0;
    . = _ROMTOP;
    .reset . : {
        *(.reset);
        . = 15;
        BYTE(0x00);
    }
}
```

Şimdi BIOS başatılıyor; BIOS'u başlatıp denetledikten sonra BIOS'un önyüklenebilir bir aygıt bulması gerekir. Bir önyükleme emri, BIOS'un hangi aygıtlardan önyükleme yapmaya çalıştığını kontrol eden BIOS yapılandırmasında saklanır. BIOS bir sabit diskten önyükleme yapmaya çalışırken bir önyükleme sektörü bulmaya çalışıyor. [MBR bölüm düzeniyle](https://en.wikipedia.org/wiki/Master_boot_record) bölünmüş sabit sürücüler üzerinde önyükleme sektörü, her sektör ```512``` bayt olan ilk sektörün ilk ```446``` baytında depolanır. İlk sektörün son iki baytı ```0x55``` ve ```0xaa'dır``` ve bu, BIOS'a bu aygıtın önyüklenebilir olduğunu belirtir. Örneğin:

```
	 assembly
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
 ```

Şu komutla derleyin ve çalıştırın: 

```
 nasm -f bin boot.nasm && qemu-system-x86_64 boot
```

Bu, [QEMU'ya](https://www.qemu.org/) yeni bir disk imajı olarak oluşturduğumuz ```önyükleme``` ikili dosyasını kullanmasını söyleyecektir. Yukarıdaki assembly koduyla oluşturulan ikili, önyükleme sektörünün gerekliliklerini yerine getirdiğinden (orijin ```0x7c00``` olarak ayarlanır ve sihirli diziyle biter) QEMU ikili dosyayı bir disk imajının ana önyükleme kaydı (MBR) olarak değerlendirecektir.

Şunu göreceksiniz;


![alt tag](https://github.com/ayyucedemirbas/linux-insides_Turkish/blob/master/Booting/images/simple_bootloader.png)

Bu örnekte, kodun ```16-bit``` gerçek modda yürütüleceğini ve bellekte ```0x7c00'de``` başlayacağını görebilirsiniz. Başladıktan sonra; sadece ```!``` sembolünü yazdıran, [0x10](http://www.ctyme.com/intr/rb-0106.htm) işlemini çağırır; kalan ```510``` bayt'ı sıfırlarla doldurur ve iki sihirli bayt ```0xaa``` ve ```0x55``` ile bitirir.


```objdump``` kullanarak bunun bir ikili dökümünü görebilirsiniz:
    
```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

Gerçek dünyadaki bir önyükleme sektörü, önyükleme işlemini devam ettirmek için bir koda ve bir bit sayısı ve bir ünlem işareti yerine bir bölüm tablosuna sahiptir :) Bu noktadan sonra, BIOS, kontrolü önyükleyiciye devreder.

NOT: Yukarıda açıklandığı gibi, CPU Real Mode'dadır; Real Mode'da, hafızadaki fiziksel adresi hesaplama şu şekilde yapılır:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

Tıpkı daha önce açıklandığı gibi. Sadece 16 bit genel amaçlı registerlarımız var; 16 bitlik bir kaydın maksimum değeri ```0xffff```, yani en büyük değerleri alırsak sonuç şöyle olacaktır:

```
python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```
Burada 0x10ffef ```(1MB + 64KB - 16b) - 1'e``` eşittir. Bunun tersine, [8086](https://en.wikipedia.org/wiki/Intel_8086) işlemci (Real Mode'lu ilk işlemci), 20 bitlik bir adres satırına sahiptir. ```2^20 = 1048576```, 1MB ve ```2^20 - 1``` olduğu için, mevcut kullanılabilir belleğin 1MB olduğu anlamına gelir. Genel Real Mode'un hafıza haritası aşağıdaki gibidir:

```
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
```
Bu yazının başında CPU tarafından yürütülen ilk talimatın ```0xFFFFFFF0``` adresinde olduğunu yazmıştım, ```0xFFFFFF'den``` (1MB) daha büyüktür. CPU Real Mode'da bu adrese nasıl erişebilir? Cevap [coreboot](https://www.coreboot.org/Developer_Manual/Memory_map) belgelerinde verilmiştir:

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space
```
Çalıştırmanın başlangıcında BIOS RAM'de değil, ancak ROM'da bulunur. 


Bootloader
--------------

[GRUB 2](https://www.gnu.org/software/grub) ve [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project) gibi Linux'u önyükleyebilen bir dizi önyükleyici var. Linux çekirdeğinin Linux desteği uygulamak için bir önyükleyicinin gereksinimlerini belirten bir [Önyükleme protokolü](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt) vardır. Bu örnek, GRUB 2'yi açıklayacaktır.

BIOS, bir önyükleme aygıtı seçti ve kontrolü önyükleme kesimi koduna aktardığı için yürütme [boot.img'den](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD) başlatılır. Bu kod, mevcut sınırlı miktarda alan nedeniyle çok basittir ve GRUB 2'nin temel görüntüsünün konumuna atlamak için kullanılan bir işaretçi içerir. Çekirdek imajı [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD) ile başlar ve genellikle ilk bölümden hemen sonra kullanılmayan alana ilk bölümden önce kaydedilir. Yukarıdaki kod, GRUB 2'nin çekirdeğini ve dosya sistemlerini işlemek için kullanılan sürücüleri içeren çekirdek görüntüsünün geri kalanını belleğe yükler. Çekirdek imajının geri kalanını yükledikten sonra, [grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c) fonksiyonunu çalıştırır.

```grub_main``` fonksiyonu konsolu başlatır, modüllerin temel adresini alır, kök aygıtını ayarlar, grub yapılandırma dosyasını yükler/ayrıştırır, modülleri yükler vb. Çalıştırma bitince, ```grub_main``` grub'ı normal moda taşır. ```grub_normal_execute``` (```grub-core/normal/main.c```'den) son hazırlıkları tamamlar ve bir işletim sistemi seçmek için bir menü gösterir. Grub menü girişlerinden birini seçtiğimizde ```grub_menu_execute_entry``` çalıştırılır, grub'ın ```önyükleme``` komutunun çalıştırır ve seçilen işletim sisteminin önyüklenmesini gerçekleştirir.

Çekirdek önyükleme protokolünü okuyabileceğimiz gibi, önyükleyici, çekirdek kurulum kodundan ```0x01f1``` ofsetten başlayan çekirdek kurulum header'ının bazı alanlarını okumalı ve doldurmalıdır. Önyükleme'deki [linker script](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) koduna bu offsetin değerini doğrulamak için bakabilirsiniz. Çekirdek header'ı  [arch/86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S),  aşağıdaki koddan başlıyor:

```
assembly
.globl hdr
     hdr:
         setup_sects: .byte 0
         root_flags:  .word ROOT_RDONLY
         syssize:     .long 0
         ram_size:    .word 0
         vid_mode:    .word SVGA_MODE
         root_dev:    .word 0
         boot_flag:   .word 0xAA55
```
	  

Önyükleyici bunu ve [bu örnek](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L354) gibi Linux önyükleme protokolünde yalnızca ```yazma``` olarak işaretlenen başlıkların geri kalanını komut satırından alınan veya önyükleme sırasında hesaplanan değerlerle doldurmalıdır. (Şimdilik çekirdek kurulum başlığının tüm alanları için tam açıklamaları ve açıklamaları gözden geçirmeyeceğiz, ancak çekirdeğin bunları nasıl kullandığını tartışırken yapacağız.[boot protokolü](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156))'nde tüm alanların bir tanımını bulabilirsiniz.

Çekirdek önyükleme protokolünde görebildiğimiz gibi, çekirdek yüklendikten sonra bellek aşağıdaki gibi eşleştirilir:

```
	   shell
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
```

Yani, önyükleyici kontrolü çekirdeğe aktardığında, ilgili kod şuradan başlar:
	
```
     X + sizeof(KernelBootSector) + 1
```
	 
Burada ```X```, çekirdek önyükleme sektörünün yüklenmekte olduğu adresidir. Benim durumumda; ```X```,  ```0x10000```'dır. Bellek dökümünde görebileceğimiz gibi:

![alt tag](https://github.com/ayyucedemirbas/linux-insides_Turkish/blob/master/Booting/images/kernel_first_address.png)

Önyükleyici, Linux çekirdeğini belleğe yükledi, header alanlarını doldurdu ve karşılık gelen bellek adresine atladı. Artık doğrudan çekirdek kurulum koduna geçebiliriz.


Çekirdek Kurulumunun Başlangıcı
-----------------

Sonunda, çekirdeğin içindeyiz! Teknik olarak, çekirdek henüz çalışmadı. İlk olarak, çekirdek kurulum kısmı, açılış ve bellek yönetimi ile ilgili bazı şeyleri, birkaçınının yapılandırılmasını ayarlamalıdır. Tüm bunlar yapıldıktan sonra, çekirdek kurulum kısmı gerçek çekirdeği açar ve ona atlar. Kurulum bölümünün yürütülmesi, [_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L292) sembolündeki [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) adresinden başlar.

İlk bakışta biraz garip görünebilir, çünkü ondan önce birkaç talimat var. Uzun zaman önce, Linux çekirdeğinin kendi bootloader'ı vardı. Ancak şimdi, örneğin,

```
    qemu-system-x86_64 vmlinuz-3.18-generic
```

şunu göreceksiniz:

![alt tag](https://github.com/ayyucedemirbas/linux-insides_Turkish/blob/master/Booting/images/try_vmlinuz_in_qemu.png)


Aslında ```header.S```, [MZ](https://en.wikipedia.org/wiki/DOS_MZ_exeecutable)'den başlar (yukarıdaki resme bakın), hata mesajını yazdıran ve aşağıdaki [PE](https://en.wikipedia.org/wiki/Portable_Executable) header'ı:


```
assembly
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
```

[UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) ile bir işletim sistemini yüklemek için buna ihtiyaç duyuyor. Şu anda bunun iç işleyişine bakmayacağız ve ilerleyen bölümlerde de bunu ele alacağız.

Gerçek çekirdek kurulum giriş noktası şöyledir:
	
```
	assembly
     // header.S line 292
     .globl _start
     _start:
```

Önyükleyici(GRUB2 ve diğerleri), bu noktayı (```MZ```'den ```0x200``` ofset) biliyor ve ```header.S```'nin bir hata mesajı yazdıran ```.bstext``` bölümünden başlamasına rağmen, doğrudan ona bir sıçrama yapıyor:
	
```
	 //
     // arch/x86/boot/setup.ld
     //
     . = 0;                    // current position
      .bstext : { *(.bstext) }  // put .bstext section to position 0
     .bsdata : { *(.bsdata) }
```

Çekirdek kurulum giriş noktası şöyledir:


```
	assembly
    .globl _start
    _start:
      .byte  0xeb
      .byte  start_of_setup-1f
     1:
        //
       // rest of the header
       //
```
	
Burada ```start_of_setup-1f``` noktasına atlayan bir ```jmp``` komut opcode (```0xeb```) görebilirsiniz. ```Nf``` gösteriminde ```2f```, aşağıdaki yerel ```2:``` etiketini belirtir; Bizim durumumuzda, atlama sonrasında bulunan etiket ```1:``` ve kurulum başlığının geri kalanını içeriyor. Kurulum başlığının hemen sonrasında, ```.entrytext``` bölümünü görüyoruz; bu bölüm, ```start_of_setup``` etiketinden başlıyor.

Aslında çalışan ilk kod budur (elbette önceki atlama talimatları hariç). Çekirdek kurulumu bootloader'dan kontrol aldıktan sonra, ilk ```jmp``` komutu, çekirdek gerçek Real Mode'unun başlangıcından itibaren ```0x200``` ofset'inde, yani ilk 512 bayttan sonra yer alır. Bu, hem Linux çekirdeği önyükleme protokolünü okuyabilir hem de GRUB2 kaynak kodunda görebiliriz:

```
	 C
	 segment = grub_linux_real_target >> 4;
     state.gs = state.fs = state.es = state.ds = state.ss = segment;
     state.cs = segment + 0x20;
```
	 
Benim durumumda, çekirdek ```0x10000``` fiziksel adresine yüklenir. Bu, çekirdek kurulumu başladıktan sonra segment kayıtlarının aşağıdaki değerlere sahip olduğu anlamına gelir:
	
```	
     gs = fs = es = ds = ss = 0x1000
     cs = 0x1020
```
```start_of_setup``` atlamasının ardından, çekirdeğin aşağıdakileri yapması gerekir:

- Tüm segment kayıt değerlerinin eşit olduğundan emin ol
- Gerekirse doğru bir yığını(stack) ayarla.
- [bss](https://en.wikipedia.org/wiki/.bss)'i ayarla
- [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) dosyasındaki C koduna atla

Şimdi uygulamaya(uyarlanmasına) bakalım.

 
Segment yazmaçlarının hizalanması
----------------------------------------------------------------------------------------------------

Her şeyden önce, çekirdek ```ds``` ve ```es``` segment yazmaçlarının aynı adrese işaret etmesini sağlar. Sonra, ```cld``` talimatı kullanarak yön bayrağını temizler:
	
```
	assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```
Daha önce yazmış olduğum gibi, ```grub2```, çekirdeği kurulum kodunu ```0x10000``` adresinde ve  ```0x1020```'deki ```cs```'yi yükler; çünkü çalıştırma dosyanın başından başlamaz, ama buradaki atlama yerinden;

```
	assembly
    _start:
    .byte 0xeb
    .byte start_of_setup-1f
```
	
jump, which is at a 512 byte offset from 4d 5a. It also needs to align cs from 0x10200 to 0x10000, as well as all other segment registers. After that, we set up the stack:

[4d 5a](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L46)'dan ```512``` bayt offset'inde olan yerdedir. Ayrıca, ```cs```'yi ```0x1020``` den ```0x1000```'e ve diğer tüm segment yazmaçlarına hizalamamız gerekir. Bundan sonra yığını(stack) kurduk:

```
	assembly
    pushw   %ds
    pushw   $6f
    lretw
```


[6](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L602)'nın adresinden itibaren yer alan ```ds``` değerini yığına(stack) iter(push eder). ```lretw``` talimatını etiketler ve yürütür. ```lretw``` talimatı çağrıldığında, ```6``` etiketinin adresini [instruction pointer](https://en.wikipedia.org/wiki/Program_counter) yazmacı'na kaydeder ve ```ds``` değeriyle ```cs``` değerini yükler. Daha sonra, `ds` ve` cs` aynı değerlere sahip olacaktır.
 
Stack Kurulumu
--------------------------------------------------------------------------------------------------------------

Kurulum kodunun neredeyse tamamı C dili ortamını gerçek modda hazırlamak içindir. Bir sonraki [adım](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L575) ```ss``` yazmacının değerini kontrol ediyor ve doğru bir yığın(stack) oluşturuyor eğer ```ss``` yanlışsa:
	
```
	assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```
Bu, 3 farklı senaryonun ortaya çıkmasına neden olabilir:

* ```ss```'in geçerli değeri ```0x10000```'tür (```cs```'in yanındaki tüm diğer bölüm yazmaçları gibi)
* ```ss``` geçersiz ve ```CAN_USE_HEAP``` bayrağı ayarlanmış (aşağıya bakın)
* ```ss``` geçersiz ve ```CAN_USE_HEAP``` bayrağı ayarlanmamış (aşağıya bakın)

Bu üç senaryonun hepsine sırayla bakalım:

* ```ss```'nin doğru adrese sahip. (```0x10000```). Bu durumda, [2](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L589) etiketine gidiyoruz:
	
```	
	assembly
2:  andw    $~3, %dx
     jnz     3f
     movw    $0xfffc, %dx
3:  movw    %ax, %ss
     movzwl  %dx, %esp
    sti
```
	
Burada, ```dx``` (bootloader tarafından verilen ```sp``` değerini içeren) değerini ```4``` bayt'a hizasına koyarız ve sıfır olup olmadığını kontrol ederiz. Sıfır ise, ```dx```'i ```0xfffc``` (64 KB'lik maksimum segment boyutundan önce 4 bayt hizalı adres) koyarız. Sıfır değilse, önyükleyici (benim durumumda ```0xf7f4```) tarafından verilen ```sp```'yi kullanmaya devam ederiz. Bundan sonra, ```ax``` (```0x1000```) değerinin segment adresini ```ss``` içine yerleştirdik. Artık doğru bir yığınımız var:

![alt tag](https://github.com/ayyucedemirbas/linux-insides_Turkish/blob/master/Booting/images/stack1.png)

* İkinci senaryoda, (```ss``` != ```ds```). Önce, [_end](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld)'in değerini (kurulum kodunun sonundaki adres) ```dx```'e koyar ve yığını(stack) kullanıp kullanamayacağımızı test etmek için ```testb``` talimatını kullanarak ```loadflags``` başlık alanını kontrol ederiz. [loadflags](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L320), aşağıdaki gibi tanımlanan bir bit maskesi header'ı olmaktadır:

```
  C
  #define LOADED_HIGH     (1<<0)
  #define QUIET_FLAG      (1<<5)
  #define KEEP_SEGMENTS   (1<<6)
  #define CAN_USE_HEAP    (1<<7)
```
Ve önyükleme protokolünü okuyabildiğimiz için,

```
Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.
```

```CAN_USE_HEAP``` biti ayarlanmışsa, ```heap_end_ptr```'ı ```dx```'e (```_end```'i işaret eder) yerleştirir ve ona ```STACK_SIZE``` (minimum yığın(stack) boyutu, ```1024``` bayt) ekleriz. Bundan sonra ```dx``` taşınmazsa (taşınmayacak, ```dx = _end + 1024```), etiket ```2```'ye atlanır (önceki durumda olduğu gibi) ve doğru bir yığın oluşturur.



 ![alt tag](https://github.com/ayyucedemirbas/linux-insides_Turkish/blob/master/Booting/images/stack2.png)


* ```CAN_USE_HEAP``` ayarlanmadığında ```_end``` ile ```_end + STACK_SIZE``` arasında en az bir yığın kullanırız:

![alt tag](https://github.com/ayyucedemirbas/linux-insides_Turkish/blob/master/Booting/images/minimal_stack.png)

BSS Kurulumu
----------------------------------------------------------------------------------------------------------------

Ana C koduna atlayabilmemiz için gerçekleşmesi gereken son iki adım [BSS](https://en.wikipedia.org/wiki/.bss) alanını kuruyor ve "sihirli" imzayı kontrol ediyor. İlk olarak imza kontrolü:
	
```
   assembly
   cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```

Bu, [setup_sig](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld)'yi sihirli sayı ```0x5a5aaa55``` ile karşılaştırır. Eşit değillerse ölümcül bir hata bildirilir. Sihirli sayı eşleşirse, bir dizi doğru bölüm yazmacı ve bir yığımız olduğunu bilerek, yalnızca C koduna atlamadan önce BSS bölümünü kurmamız gerekir.

BSS bölümü statik olarak tahsis edilmiş, başlatılmamış verileri depolamak için kullanılır. Linux dikkatli bir şekilde bu bellek alanını aşağıdaki kod kullanılarak sıfırlanmasını sağlar:

```
assembly
movw    $__bss_start, %di
movw    $_end+3, %cx
xorl    %eax, %eax
subw    %di, %cx
shrw    $2, %cx
rep; stosl
```

İlk olarak, [__bss_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) adresi ```di```'ye taşınır. Daha sonra, ```_end + 3``` adresi (+3 - 4 bayta hizalanır) ```cx```'e taşınır. ```eax``` yazmacı silinir (bir ```xor``` talimatı kullanılarak) ve bss bölüm boyutu (```cx-di```) hesaplanır ve ```cx```'e yerleştirilir. Daha sonra, ```cx``` dörde bölünür (bir 'word' boyutu) ve ```stosl``` talimatı ```eax``` (sıfır) değerini ```di```'nin gösterdiği adrese depolayarak ```di```'yi dört arttırarak tekrar ```cx```'e kadar tekrarlar Sıfıra ulaşır). Bu kodun net etkisi, sıfırların ```__bss_start```'dan ```_end```'e bellekteki tüm kelimeleri kullanarak yazıldığıdır:


![alt tag](https://github.com/ayyucedemirbas/linux-insides_Turkish/blob/master/Booting/images/bss.png)


Main'e Atlamak
--------------------------------------------------------------------------------------------------------------------

İşte hepsi bu! Yığın(stack) ve BSS'ye sahibiz, böylece ```main()``` C fonksiyonuna atlayabiliriz:
```
assembly
 calll main
```

```main()``` fonksiyonu [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) dosyasında bulunur. Bir sonraki bölümde bunun ne yaptığını okuyabilirsiniz.

Sonuç
---------------------------------------------------------------------------------------

Linux-insides hakkındaki ilk yazının sonuna geldik. Sorularınız veya önerileriniz varsa, [@0xAX](https://twitter.com/0xAX) Twitter hesabımdan, [e-posta](anotherworldofworld@gmail.com) yoluyla veya GitHub'da [issue](https://github.com/0xAX/linux-internals/issues/new) açarak bana ulaşabilirsiniz. Sonraki bölümde Linux çekirdeği kurulumundaki , ```memset```, ```memcpy```, ```earlyprintk```, konsol uygulaması ve başlatılması gibi bellek rutinlerini uygulayan ilk C kodunu ve çok daha fazlasını göreceğiz. 

* İngilizce ana dilim değil ve bu durum için özür dilerim. Herhangi bir hata bulursanız lütfen [linux-insides](https://github.com/0xAX/linux-internals)'a PR(Pull Request) gönderin.

Linkler
------------------------------------------------------------------------------------------------------------------------

  * [Intel 80386 programmer's reference manual 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
  * [Minimal Boot Loader for Intel® Architecture](https://www.cs.cmu.edu/~410/doc/minimal_boot.pdf)
  * [Minimal Boot Loader in Assembler with comments](https://github.com/Stefan20162016/linux-insides-code/blob/master/bootloader.asm)
  * [8086](https://en.wikipedia.org/wiki/Intel_8086)
  * [80386](https://en.wikipedia.org/wiki/Intel_80386)
  * [Reset vector](https://en.wikipedia.org/wiki/Reset_vector)
  * [Real mode](https://en.wikipedia.org/wiki/Real_mode)
  * [Linux kernel boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)
  * [coreboot developer manual](https://www.coreboot.org/Developer_Manual)
  * [Ralf Brown's Interrupt List](http://www.ctyme.com/intr/int.htm)
  * [Power supply](https://en.wikipedia.org/wiki/Power_supply)
  * [Power good signal](https://en.wikipedia.org/wiki/Power_good_signal)
