
## Çekirdek Önyükleme Süreci - Bölüm 6

### Giriş
Bir önceki bölümde Linux çekirdek önyükleme sürecinin son aşamalarına bir göz attık. Ancak bazı önemli, daha gelişmiş bölümleri atladık.

Hatırlayacağınız gibi, Linux çekirdeğinin giriş noktası, main.c kaynak kodu dosyasında tanımlanan start_kernel fonksiyonudur. Bu fonksiyon LOAD_PHYSICAL_ADDR içinde saklanan adreste yürütülür. ve varsayılan olarak 0x1000000 olan CONFIG_PHYSICAL_START çekirdek yapılandırma seçeneğine bağlıdır:

    config PHYSICAL_START
      hex "Physical address where the kernel is loaded" if (EXPERT || CRASH_DUMP)
      default "0x1000000"
      ---help---
        This gives the physical address where the kernel is loaded.
          ...
          ...
          ...
          
Bu değer çekirdek yapılandırması sırasında değiştirilebilir, ancak yükleme adresi rastgele bir değer olarak da yapılandırılabilir. Bu amaçla, çekirdek yapılandırması sırasında CONFIG_RANDOMIZE_BASE çekirdek yapılandırma seçeneği etkinleştirilmelidir.

Şimdi, Linux çekirdek imajının açılacağı ve yükleneceği fiziksel adres rastgele hale getirilecek. Bu bölüm, CONFIG_RANDOMIZE_BASE seçeneğinin etkinleştirildiği ve çekirdek imajının yükleme adresinin güvenlik nedeniyle (https://www.wikiwand.com/en/Address_space_layout_randomization) rasgele seçildiği durumu dikkate alır.

### Page Table'ın Başlatılması

Çekirdek açıcısının (decompress) çekirdeği açmak ve yüklemek için rastgele bir bellek aralığı arayabilmesi için, kimlik eşlemeli page tablelar başlatılmalıdır. Önyükleyici 16 bit veya 32 bit önyükleme protokolünü kullandıysa, zaten page tablelarımız vardır. Ancak, çekirdek açıcısı yalnızca 64 bit bağlamda geçerli bir bellek aralığı seçerse sorunlar olabilir. Bu yüzden yeni kimlik eşlemeli page tablelar oluşturmamız gerekiyor.

Gerçekten de, çekirdek yükleme adresini rastgele belirlemenin ilk adımı yeni kimlik eşlemeli page tablelar oluşturmaktır. Ama önce, bu noktaya nasıl geldiğimizi düşünelim.

Önceki bölümde, long moda geçişi takip ettik ve çekirdek açıcısı giriş noktasına - extract_kernel fonksiyonuna atladık. Rasgeleleştirme, bu fonksiyona yapılan bir çağrı ile başlar:

    void choose_random_location(unsigned long input,
                                unsigned long input_size,
                                unsigned long *output,
                                unsigned long output_size,
                                unsigned long *virt_addr)
    {}
    
Bu fonksiyon beş parametre alır:


  * `input`;
  * `input_size`;
  * `output`;
  * `output_isze`;
  * `virt_addr`.

Bu parametrelerin ne olduğunu anlamaya çalışalım. İlk parametre olan input, unsigned long'a dönüştürülmüş arch/x86/boot/compressed/misc.c (https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/misc.c) kaynak kodu dosyasından extract_kernel fonksiyonunun input_data parametresidir:

    asmlinkage __visible void *extract_kernel(void *rmode, memptr heap,
                                              unsigned char *input_data,
                                              unsigned long input_len,
                                              unsigned char *output,
                                              unsigned long output_len)
    {
      ...
      ...
      ...
      choose_random_location((unsigned long)input_data, input_len,
                             (unsigned long *)&output,
                             max(output_len, kernel_total_size),
                             &virt_addr);
      ...
      ...
      ...
      
Bu parametre, arch/x86/boot/compressed/head_64.S (https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S) kaynak kodu dosyasından assembly ile geçilir:

    leaq	input_data(%rip), %rdx 
    
input_data küçük mkpiggy programı tarafından oluşturulur. Linux çekirdeğini kendiniz derlemeyi denediyseniz, bu program tarafından üretilen çıktıyı linux/arch/x86/boot/compressed/piggy.S (https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/mkpiggy.c) kaynak kodu dosyasında bulabilirsiniz. Benim durumumda bu dosya şöyle görünüyor:

    .section ".rodata..compressed","a",@progbits
    .globl z_input_len
    z_input_len = 6988196
    .globl z_output_len
    z_output_len = 29207032
    .globl input_data, input_data_end
    input_data:
    .incbin "arch/x86/boot/compressed/vmlinux.bin.gz"
    input_data_end:
    
