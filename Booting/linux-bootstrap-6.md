
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
    
Gördüğünüz gibi dört global sembol içeriyor. İlk ikisi, z_input_len ve z_output_len sıkıştırılmış ve sıkıştırılmamış vmlinux.bin.gz arşivinin boyutlarıdır. Üçüncüsü, linux çekirdek görüntüsünün row binary'sine point eden input_data parametremizdir (tüm hata ayıklama sembolleri, yorumlar ve yer değiştirme bilgisinden arındırılmıştır). Son parametre olan input_data_end, sıkıştırılmış linux görüntüsünün sonuna point eder.

Bu nedenle, select_random_location fonksiyonunun ilk parametresi, piggy.o object dosyasına gömülü sıkıştırılmış çekirdek görüntüsünün pointerıdır.

select_random_location fonksiyonunun ikinci parametresi z_input_len'dir.

select_random_location fonksiyonunun üçüncü ve dördüncü parametreleri, sırasıyla sıkıştırılmış çekirdek görüntüsünün adresi ve uzunluğudur. Sıkıştırılmış çekirdeğin adresi arch/x86/boot/compressed/head_64.S kaynak kodu dosyasından geldi ve startup_32 fonksiyonunun adresi 2 megabaytlık bir sınıra hizalandı. Sıkıştırılmış çekirdeğin boyutu, yine piggy.S'te bulunan z_output_len tarafından verilir.

select_random_location fonksiyonunun son parametresi, çekirdek yükleme adresinin sanal adresidir. Görüldüğü gibi, varsayılan olarak, varsayılan fiziksel yükleme adresiyle denk gelir:

    unsigned long virt_addr = LOAD_PHYSICAL_ADDR;
    
Fiziksel yükleme adresi konfigürasyon seçenekleriyle tanımlanır:

    #define LOAD_PHYSICAL_ADDR ((CONFIG_PHYSICAL_START \
                    + (CONFIG_PHYSICAL_ALIGN - 1)) \
                    & ~(CONFIG_PHYSICAL_ALIGN - 1))
                    
 select_random_location parametrelerini ele aldık, bu yüzden uygulanmasına bakalım. Bu fonksiyon, çekirdek komut satırındaki nokaslr seçeneğini kontrol ederek başlar:
 
     if (cmdline_find_option_bool("nokaslr")) {
        warn("KASLR disabled: 'nokaslr' on cmdline.");
        return;
    }
 
 Seçenek belirtilirse, select_random_location öğesinden çıkar ve çekirdek yükleme adresini rastgele bırakırız. Bununla ilgili bilgiler çekirdeğin belgelerinde (https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/kernel-parameters.rst) bulunabilir:
 
     kaslr/nokaslr [X86]

    Enable/disable kernel and module base offset ASLR
    (Address Space Layout Randomization) if built into
    the kernel. When CONFIG_HIBERNATION is selected,
    kASLR is disabled by default. When kASLR is enabled,
    hibernation will be disabled.
    
 Diyelim ki çekirdek komut satırına nokaslr geçirmedik ve CONFIG_RANDOMIZE_BASE çekirdek yapılandırma seçeneği etkinleştirildi. Bu durumda çekirdek yükleme bayraklarına (flag) kASLR bayrağı ekleriz:
 
     boot_params->hdr.loadflags |= KASLR_FLAG;
     
 Şimdi başka bir fonksiyonu çağırıyoruz:
 
     initialize_identity_maps();
     
initialize_identity_maps fonksiyonu rch/x86/boot/compressed/kaslr_64.c (https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/kaslr_64.c) kaynak kodu dosyasında tanımlanmıştır. Bu fonksiyon, mapping_info adlı x86_mapping_info structureının bir örneğinin başlatılmasıyla başlar:

    mapping_info.alloc_pgt_page = alloc_pgt_page;
    mapping_info.context = &pgt_data;
    mapping_info.page_flag = __PAGE_KERNEL_LARGE_EXEC | sev_me_mask;
    mapping_info.kernpg_flag = _KERNPG_TABLE;
