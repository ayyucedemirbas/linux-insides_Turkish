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