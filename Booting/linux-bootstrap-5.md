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
//to be continued :)






Linkler
--------------------------------------------------------------------------------

* [address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization)
* [initrd](https://en.wikipedia.org/wiki/Initrd)
* [long mode](https://en.wikipedia.org/wiki/Long_mode)
* [bzip2](http://www.bzip.org/)
* [RdRand instruction](https://en.wikipedia.org/wiki/RdRand)
* [Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)
* [Programmable Interval Timers](https://en.wikipedia.org/wiki/Intel_8253)
* [Previous part](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-4.md)
