## Çekirdeğin Önyüklenmesi İşlemi 5.Bölüm
--------------------------------------------
### Çekirdeğin Açılması (Kernel Decompression)

Bu, çekirdek önyükleme işlemi serisinin beşinci bölümüdür. Önceki bölümde 64 bit moduna geçtik ve bu kısımda kaldığımız yerden devam edeceğiz. Çekirdek açma işlemine hazırlık, yer değiştirme ve çekirdek açma işleminin kendisi için atılan adımları inceleyeceğiz. 
Tekrar çekirdek koduna bakalım.

#### Çekirdeğin Açılması İçin Hazırlık
64 bit giriş noktası startup_64'e girmeden hemen önce durduk. (Kaynak kodu: https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S)
Bir önceki bölümde startup_32'den startup_64'e atlamayı ele aldık:


        pushl	$__KERNEL_CS
	      leal	startup_64(%ebp), %eax
	      ...
	      ...
	      ...
        pushl	%eax
       	...
	      ...
	      ...
      	lret
        
Yeni bir Genel Tanımlayıcı Tablosu yüklediğimizden ve CPU yeni bir moda geçti (bizim durumumuzda 64 bit modu), segment registerlarını tekrar startup_64 fonksiyonunun başında ayarladık:
 
         .code64
	       .org 0x200
       ENTRY(startup_64)
	       xorl	%eax, %eax
	       movl	%eax, %ds
	       movl	%eax, %es
	       movl	%eax, %ss
         movl	%eax, %fs
         movl	%eax, %gs
     
cs registerının dışındaki tüm segment regislerları artık long modda resetleniyor. 

Bir sonraki adım, çekirdeğin yüklenmek üzere derlendiği konum ile gerçekte yüklendiği konum arasındaki farkı hesaplamaktır:

        #ifdef CONFIG_RELOCATABLE
	         leaq	startup_32(%rip), %rbp
	         movl	BP_kernel_alignment(%rsi), %eax
	         decl	%eax
	         addq	%rax, %rbp
	         notq	%rax
	         andq	%rax, %rbp
	         cmpq	$LOAD_PHYSICAL_ADDR, %rbp
	         jge	1f
        #endif
	         movq	$LOAD_PHYSICAL_ADDR, %rbp
        1:
	         movl	BP_init_size(%rsi), %ebx
	         subl	$_end, %ebx
	         addq	%rbp, %rbx
           
rbp registerı, sıkıştırılmış çekirdeğin başlangıç adresini içerir. Bu kod yürütüldükten sonra, rbx registerı, açma işlemi için çekirdek kodunun yerini değiştireceği adresi içerecektir. 
Bunu daha önce startup_32 fonksiyonunda yapmıştık (bunu önceki bölümde okuyabilirsiniz), ancak önyükleyici şimdi 64 bit önyükleme protokolünü kullanabildiğinden ve startup_32 artık yürütülmüyor.


Bir sonraki adımda, stack pointerı ayarlıyoruz, flag registerını resetliyoruz ve 32 bit spesifik değerleri 64 bit protokolünden olanlarla değiştirmek için GDT'yi tekrar kuruyoruz:

            leaq	boot_stack_end(%rbx), %rsp

           leaq	gdt(%rip), %rax
           movq	%rax, gdt64+2(%rip)
           lgdt	gdt64(%rip)

           pushq	$0
           popfq
           
 Eğer lgdt gdt64 (% rip) komutundan sonra instructiona bir göz atarsanız, bazı ek kodlar olduğunu göreceksiniz. Bu kod, gerektiğinde 5 seviyeli sayfalamayı (https://lwn.net/Articles/708526/) etkinleştirmek için trambolin oluşturur. 
 Bu kitapta yalnızca 4 seviyeli sayfalamayı ele alacağız, bu nedenle bu kod atlanacaktır.

Yukarıda da görebileceğiniz gibi, rbx registerı çekirdek açma kodunun başlangıç adresini içerir ve bu adresi rsp registerına stackin tepesine işaret eden bir boot_stack_end ofsetiyle koyarız. 
Bu adımdan sonra stack doğru olacaktır. boot_stack_end sabitinin tanımını arch/x86/boot/compressed/head_64.S (https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S)
assembly kaynak kodu dosyasının sonunda bulabilirsiniz:

            	.bss
	            .balign 4
        boot_heap:
	            .fill BOOT_HEAP_SIZE, 1, 0
        boot_stack:
	            .fill BOOT_STACK_SIZE, 1, 0
        boot_stack_end:
	
 .bss sectionının sonunda  .pgtable 'ın hemen öncesinde yer alır. arch/x86/boot/compressed/vmlinux.lds.S (https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/vmlinux.lds.S) linker scriptine bakarsanız, .bss ve .pgtable tanımlarını burada bulabilirsiniz.

Stack şimdi doğru olduğundan, sıkıştırılmış çekirdeği, sıkıştırılmış çekirdeğin yeniden yerleştirme adresini hesapladığımızda, yukarıdaki adrese kopyalayabiliriz. Ayrıntılara girmeden önce, bu assembly koduna bir göz atalım:

	pushq	%rsi
	leaq	(_bss-8)(%rip), %rsi
	leaq	(_bss-8)(%rbx), %rdi
	movq	$_bss, %rcx
	shrq	$3, %rcx
	std
	rep	movsq
	cld
	popq	%rsi
	
Bu instruction dizisi sıkıştırılmış çekirdeği açılmış olduğu yerin üstüne kopyalar.
İlk önce rsi'yi stacke push ediyoruz. Bu register artık boot_params'a  önyükleme ile ilgili verileri içeren gerçek mod yapısı olan bir pointerı sakladığından (bu yapının çekirdek kurulumunun başında doldurulduğunu unutmayın), rsi değerini korumamız gerekir. Bu kodu yürüttükten sonra pointerı boot_params'a, rsi'nin arkasına (arkası olarak çevrildiğinden emin değilim, yeniden olabilir) pop ediyoruz.
Sonraki iki leaq instructionı, rbs ve rbx registerlarının etkin adreslerini _bss - 8 ofsetiyle hesaplar ve sonuçları sırasıyla rsi ve rdi'ye atar. Bu adresleri neden hesaplıyoruz? Sıkıştırılmış çekirdek imajı bu kod (startup_32'den geçerli koda) ve dekompresyon kodu arasında bulunur. Bunu bu linker scriptine bakarak doğrulayabilirsiniz - arch/x86/boot/compressed/vmlinux.lds.S:(https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/vmlinux.lds.S)

	. = 0;
	.head.text : {
		_head = . ;
		HEAD_TEXT
		_ehead = . ;
	}
	.rodata..compressed : {
		*(.rodata..compressed)
	}
	.text :	{
		_text = .; 	/* Text */
		*(.text)
		*(.text.*)
		_etext = . ;
	}

.head.text bölümünün startup_32 içerdiğini unutmayın. Bir önceki bölümden hatırlayabilirsiniz:

	__HEAD
	.code32
        ENTRY(startup_32)
        ...
        ...
        ...


.text sectionı, açma(decompression) kodunu içerir


```assembly
	.text
relocated:
...
...
...
/*
 * Do the decompression, and jump to the new kernel.. (Sıkıştırmayı aç, ve yeni kernela atla)
 */
...
```
Ve .rodata..compressed sıkıştırılmış çekirdek imajını içerir. Yani rsi _bss - 8 mutlak adresini içerecek ve rdi _bss - 8'in yer değiştirme relative adresini içerecektir. Bu adresleri registerlarda sakladığımız gibi, _bss adresini rcx registerına koyarız. vmlinux.lds.S linker scriptinde görebileceğiniz gibi, tüm bölümlerin sonunda setup/kernel kodu ile birlikte bulunur. Şimdi movsq komutuyla rsi'den rdi'ye, bir seferde 8 bayt veri kopyalamaya başlayabiliriz.Verileri kopyalamadan önce bir std instructionı uyguladığımızı unutmayın. 

Bu, DF bayrağını belirler, yani rsi ve rdi azalacaktır. Başka bir deyişle, baytları geriye doğru kopyalayacağız. Sonunda, cld komutuyla DF bayrağını temizleriz ve boot_params yapısını rsi'ye geri yükleriz.

Şimdi taşıma işleminden sonra .text bölümünün adresinde bir pointerımız var ve buna atlayabiliriz(jump):

```assembly
	leaq	relocated(%rbx), %rax
	jmp	*%rax
```
Kernelın açılmasından önceki son dokunuşlar
-------------------------------------------------------------------------------
Önceki paragrafta .text bölümünün taşınan etiketle başladığını gördük. Yaptığımız ilk şey bss bölümünü(section) şu şekilde temizlemektir:

```assembly
	xorl	%eax, %eax
	leaq    _bss(%rip), %rdi
	leaq    _ebss(%rip), %rcx
	subq	%rdi, %rcx
	shrq	$3, %rcx
	rep	stosq
```

.bss bölümünü başlatmamız gerekiyor, çünkü yakında C koduna geçeceğiz. Burada sadece eax'ı temizliyoruz, rdi'ye _bss ve rcx'e _ebss adreslerini koyuyoruz ve rep stosq komutuyla .bss'yi sıfırlarla dolduruyoruz.
Sonunda extract_kernel fonksiyonuna bir çağrı görebiliriz:


```assembly
	pushq	%rsi
	movq	%rsi, %rdi
	leaq	boot_heap(%rip), %rsi
	leaq	input_data(%rip), %rdx
	movl	$z_input_len, %ecx
	movq	%rbp, %r8
	movq	$z_output_len, %r9
	call	extract_kernel
	popq	%rsi
```
Daha önce olduğu gibi, boot_params'a olan pointerı muhafaza etmek için rsi'yi stacke push ediyoruz. Ayrıca rsi içeriğini rdi'ye kopyalarız. Sonra, rsi'yi çekirdeğin sıkıştırmasının açılacağı alanı gösterecek şekilde ayarladık. Son adım, extract_kernel fonksiyonu için parametreleri hazırlamak ve çekirdeği açmak için çağırmaktır. extract_kernel fonksiyonu arch/x86/boot/compressed/misc.c kaynak kodu dosyasında tanımlanır ve altı argüman alır:

* 'rmode'  - bootloader tarafından veya erken çekirdek başlatma sırasında doldurulmuş boot_params yapısına bir pointer;
* 'heap'  - önyükleme heapinin başlangıç adresini temsil eden boot_heap için bir pointer;
* 'input_data'  - sıkıştırılmış çekirdeğin başlangıcına bir pointer ya da başka bir deyişle, arch/x86/boot/compressed/vmlinux.bin.bz2  dosyasına bir pointer
* 'input_len'  - sıkıştırılmış çekirdeğin boyutu;
* 'output'  - sıkıştırılmış çekirdeğin başlangıç adresi;
* 'output_len'  - sıkıştırılmış çekirdeğin boyutu;

Tüm argümanlar [System V Application Binary Interface](http://www.x86-64.org/documentation/abi.pdf) 'e göre registerlardan geçirilecektir. Tüm hazırlıkları bitirdik ve şimdi çekirdeği açabiliriz.

Çekirdeğin Açılması
----------------------------------------------------------------------------------
Önceki paragrafta gördüğümüz gibi, extract_kernel fonsiyonu arch/x86/boot/compressed/misc.c (https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/misc.c) kaynak kodu dosyasında tanımlanır ve altı argüman alır. Bu fonksiyon önceki bölümlerde gördüğümüz video/console başlatma ile başlar. Bunu tekrar yapmalıyız çünkü real modda (https://www.wikiwand.com/en/Real_mode) başlayıp başlamadığımızı veya bir önyükleyicinin (bootloader)kullanılıp kullanılmadığını veya önyükleyicinin 32 veya 64 bit önyükleme protokolünü kullanıp kullanmadığını bilmiyoruz.

İlk başlatma adımlarından sonra, boş belleğin başlangıcına ve sonuna olan pointerları saklarız:

        free_mem_ptr     = heap;
        free_mem_end_ptr = heap + BOOT_HEAP_SIZE;
	
Burada heap,extract_kernel fonksiyonunun arch/x86/boot/compressed/head_64.S (https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/misc.c)'e aktarılan ikinci parametresidir.

        leaq	boot_heap(%rip), %rsi
	
Yukarıda gördüğünüz gibi boot_heap

      boot_heap:
	     .fill BOOT_HEAP_SIZE, 1, 0
	     
şeklinde tanımlanıyor

burada BOOT_HEAP_SIZE, 0x10000'e (bir bzip2 çekirdeğinin 0x400000 değeri) genişleyen ve heapin boyutunu temsil eden bir makrodur.

Heap pointerlarını başlattıktan sonra, sonraki adım arch/x86/boot/compressed/kaslr.c (https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/kaslr.c) kaynak kodu dosyasından select_random_location fonsiyonunu çağırmaktır. Fonksiyon adından tahmin edebileceğimiz gibi, açılmış çekirdeği yazmak için bir bellek konumu seçer. Sıkıştırılmış çekirdek imajını nerede açacağımızı bulmamız hatta seçmemiz garip gelebilir, ancak Linux çekirdeği, güvenlik nedeniyle çekirdeğin rastgele bir adrese sıkıştırılmasına izin veren kASLR (https://www.wikiwand.com/en/Address_space_layout_randomization)'ı destekler.

Bir sonraki bölümde çekirdeğin yükleme adresinin nasıl randomize edildiğine bakacağız.

Şimdi misc.c (https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/misc.c)'ye dönelim. Çekirdek imajının adresini aldıktan sonra, aldığımız rastgele adresin doğru bir şekilde hizalandığını ve genel olarak yanlış olmadığını kontrol etmeliyiz:

	if ((unsigned long)output & (MIN_KERNEL_ALIGN - 1))
		error("Destination physical address inappropriately aligned");

	if (virt_addr & (MIN_KERNEL_ALIGN - 1))
		error("Destination virtual address inappropriately aligned");

	if (heap > 0x3fffffffffffUL)
		error("Destination address too large");

	if (virt_addr + max(output_len, kernel_total_size) > KERNEL_IMAGE_SIZE)
		error("Destination virtual address is beyond the kernel mapping area");

	if ((unsigned long)output != LOAD_PHYSICAL_ADDR)
	    error("Destination address does not match LOAD_PHYSICAL_ADDR");

	if (virt_addr != LOAD_PHYSICAL_ADDR)
		error("Destination virtual address changed when not relocatable");
		
Tüm bu kontrollerden sonra aşina olduğumuz mesajı göreceğiz,
 
               Decompressing Linux...
Şimdi çekirdeği açmak için _decompress foksiyonunu çağırıyoruz,	

	__decompress(input_data, input_len, NULL, NULL, output, output_len, NULL, error);
_decompress fonksiyonunun implementasyonu, çekirdek derleme sırasında hangi açma algoritmasının seçildiğine bağlıdır:

	#ifdef CONFIG_KERNEL_GZIP
	#include "../../../../lib/decompress_inflate.c"
	#endif

	#ifdef CONFIG_KERNEL_BZIP2
	#include "../../../../lib/decompress_bunzip2.c"
	#endif

	#ifdef CONFIG_KERNEL_LZMA
	#include "../../../../lib/decompress_unlzma.c"
	#endif

	#ifdef CONFIG_KERNEL_XZ
	#include "../../../../lib/decompress_unxz.c"
	#endif

	#ifdef CONFIG_KERNEL_LZO
	#include "../../../../lib/decompress_unlzo.c"
	#endif

	#ifdef CONFIG_KERNEL_LZ4
	#include "../../../../lib/decompress_unlz4.c"
	#endif
	
Çekirdek açıldıktan sonra iki fonksiyon daha çağrılır: parse_elf ve handle_relocations. Bu fonksiyonların ana noktası, sıkıştırılmış çekirdek imajını bellekte doğru yerine taşımaktır. Bunun nedeni, açma işleminin in-place (https://www.wikiwand.com/en/In-place_algorithm) algoritması ile yapılması ve hala çekirdeği doğru adrese taşımamız gerektiğidir. Bildiğimiz gibi, çekirdek imajı bir ELF (https://www.wikiwand.com/en/Executable_and_Linkable_Format) çalıştırılabilir dosyasıdır. parse_elf foksiyonunun temel amacı, yüklenebilir segmentleri doğru adrese taşımaktır. Çekirdeğin yüklenebilir bölümlerini readelf programının çıktısında görebiliriz:

	readelf -l vmlinux

	Elf file type is EXEC (Executable file)
	Entry point 0x1000000
	There are 5 program headers, starting at offset 64

	Program Headers:
	  Type           Offset             VirtAddr           PhysAddr
			 FileSiz            MemSiz              Flags  Align
	  LOAD           0x0000000000200000 0xffffffff81000000 0x0000000001000000
			 0x0000000000893000 0x0000000000893000  R E    200000
	  LOAD           0x0000000000a93000 0xffffffff81893000 0x0000000001893000
			 0x000000000016d000 0x000000000016d000  RW     200000
	  LOAD           0x0000000000c00000 0x0000000000000000 0x0000000001a00000
			 0x00000000000152d8 0x00000000000152d8  RW     200000
	  LOAD           0x0000000000c16000 0xffffffff81a16000 0x0000000001a16000
			 0x0000000000138000 0x000000000029b000  RWE    200000
			 
parse_elf fonksiyonunun amacı, bu segmentleri select_random_location fonksiyonundan aldığımız çıktı adresine yüklemektir. Bu fonksiyon ELF imzasını kontrol ederek başlar:

	Elf64_Ehdr ehdr;
	Elf64_Phdr *phdrs, *phdr;

	memcpy(&ehdr, output, sizeof(ehdr));

	if (ehdr.e_ident[EI_MAG0] != ELFMAG0 ||
	    ehdr.e_ident[EI_MAG1] != ELFMAG1 ||
	    ehdr.e_ident[EI_MAG2] != ELFMAG2 ||
	    ehdr.e_ident[EI_MAG3] != ELFMAG3) {
		error("Kernel is not a valid ELF file");
		return;
	}
	
ELF başlığı geçerli değilse, bir hata mesajı yazdırır ve durur. Geçerli bir ELF dosyanız varsa, verilen ELF dosyasındaki tüm program başlıklarını gözden geçiririz ve doğru 2 megabayt hizalanmış adresleri olan tüm yüklenebilir segmentleri çıktı arabelleğine kopyalarız:

		for (i = 0; i < ehdr.e_phnum; i++) {
			phdr = &phdrs[i];

			switch (phdr->p_type) {
			case PT_LOAD:
	#ifdef CONFIG_X86_64
				if ((phdr->p_align % 0x200000) != 0)
					error("Alignment of LOAD segment isn't multiple of 2MB");
	#endif
	#ifdef CONFIG_RELOCATABLE
				dest = output;
				dest += (phdr->p_paddr - LOAD_PHYSICAL_ADDR);
	#else
				dest = (void *)(phdr->p_paddr);
	#endif
				memmove(dest, output + phdr->p_offset, phdr->p_filesz);
				break;
			default:
				break;
			}
		}
		
Bu kadar.

Bu andan itibaren tüm yüklenebilir segmentler doğru yerdedir.

parse_elf fonksiyonundan sonraki adım handle_relocations fonksiyonunu çağırmaktır. Bu fonksiyonun uygulanması, CONFIG_X86_NEED_RELOCS çekirdek yapılandırma seçeneğine bağlıdır ve etkinleştirilmişse, bu fonksiyon çekirdek imajındaki adresleri ayarlar. Bu fonksiyon yalnızca çekirdek yapılandırması sırasında CONFIG_RANDOMIZE_BASE yapılandırma seçeneği etkinleştirilmişse çağrılır. handle_relocations fonksiyonunun uygulanması yeterince kolaydır. Bu fonksiyon, LOAD_PHYSICAL_ADDR değerini çekirdeğin temel yükleme adresinin değerinden çıkarır ve böylece çekirdeğin yükleme ile bağlandığı yer ile gerçekte yüklendiği yer arasındaki farkı elde ederiz. Bundan sonra, çekirdeğin yüklendiği gerçek adresi, çalıştırılmak için bağlandığı adresi ve çekirdek imajının sonundaki yer değiştirme tablosunu bildiğimiz için çekirdeği yeniden konumlandırabiliriz. 

Çekirdek yer değiştirdikten sonra, extract_kernel fonksiyonundan arch / function to arch/x86/boot/compressed/head_64.S'e geri döneriz.

Çekirdeğin adresi rax registerında olacak ve ona atlayacağız:

	jmp	*%rax
	
Hepsi bu kadar. Şimdi çekirdekteyiz!

Linkler
--------------------------------------------------------------------------------

* [address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization)
* [initrd](https://en.wikipedia.org/wiki/Initrd)
* [long mode](https://en.wikipedia.org/wiki/Long_mode)
* [bzip2](http://www.bzip.org/)
* [RdRand instruction](https://en.wikipedia.org/wiki/RdRand)
* [Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)
* [Programmable Interval Timers](https://en.wikipedia.org/wiki/Intel_8253)
* [Onceki bolum](https://github.com/ayyucedemirbas/linux-insides_Turkish/blob/master/Booting/linux-bootstrap-4.md)
