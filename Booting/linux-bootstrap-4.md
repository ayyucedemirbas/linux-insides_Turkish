
## Bölüm 4: Çekirdek Önyükleme Süreci


64-bit moda geçiş
--------------------------------------------------------------------------------

Bu yazı `Çekirdek önyükleme süreci` yazı dizisinin 4. yazısı. Bu bölümde [korumalı mod](http://en.wikipedia.org/wiki/Protected_mode)'un(protected mode) ilk adımlarını CPU'nun [long mode](http://en.wikipedia.org/wiki/Long_mode) ve [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)'yi desteklediği, sayfa tablolarını hazırlayan [sayfalama](http://en.wikipedia.org/wiki/Paging)'yı ve en sonda da [long mode](https://en.wikipedia.org/wiki/Long_mode)'a geçişi tartışacağız.

**NOT: Bu bölümde bolca Assembly kodu bulunmaktadır, bu yüzden eğer Assembly ile haşır neşir değilseniz takıldığınız yerlerde Google'a veya bir kitaba danışmanızı öneririm.**

Önceki [bölümde](https://github.com/ayyucedemirbas/linux-insides_Turkish/blob/master/Booting/linux-bootstrap-3.md) [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pmjump.S)'deki 32-bit giriş noktasına atlamada(jump) kalmıştık:

```assembly
jmpl	*%eax
```
32-bit giriş noktasının adresini içeren `eax` registerını hatırlamış olmalısınız. Bunu [linux kernel x86 boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) okuyabilirsiniz:

```
When using bzImage, the protected-mode kernel was relocated to 0x100000
```

32-bit giriş noktasındaki register değerlerine bakarak doğru olduğuna emin olalım:

```
eax            0x100000	1048576
ecx            0x0	    0
edx            0x0	    0
ebx            0x0	    0
esp            0x1ff5c	0x1ff5c
ebp            0x0	    0x0
esi            0x14470	83056
edi            0x0	    0
eip            0x100000	0x100000
eflags         0x46	    [ PF ZF ]
cs             0x10	16
ss             0x18	24
ds             0x18	24
es             0x18	24
fs             0x18	24
gs             0x18	24
```

Gördüğümüz üzere `cs` registerı `0x10`(önceki bölümden hatırlayacağınız üzere bu, Global Descriptor Table'daki ikinci indekstir), `eip` registerı `0x100000` içeriyor ve tüm bölütlerin temel adresi, kod bölütü dahil sıfırdır. Buradan fiziksel adreslerini elde edebiliriz ve boot protokolünde belirtildiği üzere bu `0:0x100000` veya kısaca `0x100000` olur. Şimdi 32-bit giriş noktasından başlayalım.

32-bit giriş noktası
--------------------------------------------------------------------------------

32-bit giriş noktasının tanımını [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) assembly kaynak kodunda bulabilirsiniz:

```assembly
	__HEAD
	.code32
ENTRY(startup_32)
....
....
....
ENDPROC(startup_32)
```

Öncelikle neden `sıkıştırılmış(compressed)` dizin? Aslında `bzimage`, gzip'lenmiş `vmlinux + header + çekirdek kurulum kodu` dur. Çekirdek kurulum kodunu önceki bölümlerde görmüştük. Yani `head_64.S` dosyasının esas amacı long mode'a girmeye hazırlanmak, girdikten sonra da çekirdeği decompress(sıkıştırmayı çözme/geri alma) etmelidir. Çekirdeğin tüm decompress aşamalarını bu bölümde göreceğiz.

`arch/x86/boot/compressed` dizininde 2 tane dosya var:

* [head_32.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_32.S)
* [head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S)

Fakat biz sadece `head_64.S` dosyasını işleyeceğiz çünkü, hatırlayacağınız üzere bu kitap sadece `x86_64` ile alakalı; Bizim durumumuzda `head_32.S` kullanılmamaktadır. Şimdi [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/Makefile)'e bakalım:

```Makefile
vmlinux-objs-y := $(obj)/vmlinux.lds $(obj)/head_$(BITS).o $(obj)/misc.o \
	$(obj)/string.o $(obj)/cmdline.o \
	$(obj)/piggy.o $(obj)/cpuflags.o
```

`$(obj)/head_$(BITS).o` not edin. Bu, `$(BITS)` değerine göre hangi dosyaya linkleyeceğimizi seçeceğiz, yani ya head_32.o ya da head_64.o dosyasına. `$(BITS)` başka bir yerde daha, [arch/x86/Makefile](https://github.com/torvalds/linux/blob/master/arch/x86/Makefile) içinde .config dosyasına dayanarak tanımlandı:

```Makefile
ifeq ($(CONFIG_X86_32),y)
        BITS := 32
        ...
        ...
else
        BITS := 64
        ...
        ...
endif
```

Şimdi nereden başlayacağımızı biliyoruz, haydi başlayalım.

Gerekirse bölütleri(segments) tekrar yükle
--------------------------------------------------------------------------------

Yukarıda gösterildiği üzere, [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) assembly kaynak kodundan başlıyoruz. İlk olarak `startup_32` tanımından önce bölüm özelliğinin tanımını görüyoruz:

```assembly
    __HEAD
	.code32
ENTRY(startup_32)
```

`__HEAD` [include/linux/init.h](https://github.com/torvalds/linux/blob/master/include/linux/init.h) header dosyasında tanımlanmış ve aşağıdaki bölümün tanımına genişleyen bir makrodur:

```
#define __HEAD		.section	".head.text","ax"
```

`.head.text` ile isim ve `ax` işaretlenir. Bizim durumumuzda, bu işaretler bu kısmın [çalıştırılabilir](https://en.wikipedia.org/wiki/Executable) ya da diğer bir deyişle kod içerdiğini gösterir. Bu kısmın tanımını [arch/x86/boot/compressed/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/vmlinux.lds.S) dosyasında bulabiliriz:

```
SECTIONS
{
	. = 0;
	.head.text : {
		_head = . ;
		HEAD_TEXT
		_ehead = . ;
	}
```

Eğer `GNU LD`(linker scripting language)'nin söz dizimine tanıdık değilseniz, daha fazla bilgiye bu [dökümantasyon](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts)'dan ulaşabilirsiniz. Kısaca, `.` sembolü linker - konum sayacının özel bir değişkenidir. Bizim durumumuzda konum sayacına sıfır atıyoruz. Bu şu anlama gelir; kodumuzun hafızada `0` konumundan çalıştırılacağı anlamına gelir. Ayrıca, bu bilgiyi yorumlarda da görebiliriz:

```
Be careful parts of head_64.S assume startup_32 is at address 0.
```

Peki şuan nerede olduğumuzu biliyoruz, şimdi `startup_32` fonksiyonunun içine bakmamızın tam sırası.

`startup_32` fonksiyonunun başlarında, `cld`(clear) komutunu görüyoruz ki bu komut [FLAGS](https://en.wikipedia.org/wiki/FLAGS_register) kaydındaki `DF` bitini temizler. Direction flag(DF) temizlendiğinde, [stos](http://x86.renejeschke.de/html/file_module_x86_id_306.html), [scas](http://x86.renejeschke.de/html/file_module_x86_id_287.html) gibi string komutları ve diğerleri `esi` ya da `edi` index kayıtlarını arttırır. Direction Flag'i temizlememiz gerek çünkü daha sonra string komutlarını kullanarak sayfa tabloları vb. temizleyeceğiz.

`DF` bitini temizledikten sonraki adım, `loadflags` başlık alanındaki `KEEP_SEGMENTS` adlı flag'i kontrol etmektir. Hatırlayacağınız üzere `loadflags` i ilk [bölümde](https://github.com/ayyucedemirbas/linux-insides_Turkish/blob/master/Booting/linux-bootstrap-1.md) görmüştük. O bölümde `CAN_USE_HEAP` flag'ini yığını(heap) kullanabilmek için kontrol etmiştik. Şimdiyse `KEEP_SEGMENTS` flag'ini kontrol etmemiz gerekiyor. Bu flagler linux'un [boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) dökümanında anlatılıyor:

```
Bit 6 (write): KEEP_SEGMENTS
  Protocol: 2.07+
  - If 0, reload the segment registers in the 32bit entry point.
  - If 1, do not reload the segment registers in the 32bit entry point.
    Assume that %cs %ds %ss %es are all set to flat segments with
	a base of 0 (or the equivalent for their environment).
```

Yani eğer `KEEP_SEGMENTS` biti `loadflags` de ayarlanmazsa, `ds`, `ss` ve `es` bölüt kayıtlarını tabanı `0` olan düz bir bölüte resetlememiz gerekir.

```
	C
	testb $(1 << 6), BP_loadflags(%esi)
	jnz 1f

	cli
	movl	$(__BOOT_DS), %eax
	movl	%eax, %ds
	movl	%eax, %es
	movl	%eax, %ss
```

`__BOOT_DS`in `0x18` olduğunu unutma([Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table)da veri bölütünün indeksi). Eğer `KEEP_SEGMENTS` ayarlıysa, en yakın `1f` etiketine atlarız(jump) ya da eğer ayarlanmamışsa bölüt kayıtlarını `__BOOT_DS` ile güncelleriz. Oldukça kolay gözüküyor, ama şimdi ilginç bir şey geliyor. Eğer önceki [bölümü](https://github.com/ayyucedemirbas/linux-insides_Turkish/blob/master/Booting/linux-bootstrap-3.md) okuduysanız, hatırlayacağınız üzere [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pmjump.S)te [protected moduna](https://en.wikipedia.org/wiki/Protected_mode) geçtikten sonra bu bölüt kayıtlarını güncellemiştik. Pekala neden tekrar bölüt kayıtlarının değerlerine bakmamız gerekiyor? Cevabı kolay. Linux çekirdeği ayrıca bir 32-bit önyükleme(boot) protokolüne sahip ve eğer bootloader Linux çekirdeğini yüklemek için kullanırsa,  all code before the `startup_32` will be missed. Bu durumda `startup_32`, bootloader'ın hemen ardından Linux çekirdeğinin ilk giriş noktası olacaktır ve segment kayıtlarının bilinen durumda olacağına dair bir garanti bulunmamaktadır.

`KEEP_SEGMENTS` flag'ini kontrol ettikten ve bölüt kayıtlarına doğru değerleri koyduktan sonraki adım nerede yüklediğimiz ve çalıştırmak için derlenmiş olan arasındaki farkı hesaplamaktır. `setup.ld.S`in `.head.text` bölümünün başında `. = 0` tanımını içerdiğini unutmayın. Bu tanım, bu bölümdeki kodun `0` adresinden çalıştırılması için derlendiği anlamına gelir. Bunu `objdump` çıktısında görebiliriz:

```
arch/x86/boot/compressed/vmlinux:     file format elf64-x86-64


Disassembly of section .head.text:

0000000000000000 <startup_32>:
   0:   fc                      cld
   1:   f6 86 11 02 00 00 40    testb  $0x40,0x211(%rsi)
```

`objdump` bize `startup_32`nin adresinin `0` olduğunu söyler. Fakat aslında öyle değil. Şu anki amacımız aslında tam olarak nerede olduğumuzu bilmek. `rip` göreceli adreslemeyi(relative adressing) desteklediği için [Long modunda](https://en.wikipedia.org/wiki/Long_mode) yapmak oldukça basit ama biz şu anda [protected modundayız](https://en.wikipedia.org/wiki/Protected_mode)
//to be continued


