
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

