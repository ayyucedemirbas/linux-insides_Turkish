
Çekirdek Boot Süreci Bölüm 3.

Video Modunu Başlatma ve Korumalı Moda Geçiş

Bu yazı, çekirdeğin boot süreci yazı dizisinin 3. yazısı. Önceki bölümde, set.video rutininin çağrısından önce main.c dosyasından durduk. Bu bölümde şunu göreceğiz:

- Çekirdek kurulum kodunda video modunu başlatma,
- Korumalı moda geçmeden önce hazırlık,
- Korumalı moda geçiş

NOT: Korumalı mod hakkında hiçbir şey bilmiyorsanız, önceki bölümde bu konuda bazı bilgiler bulabilirsiniz. Ayrıca size yardımcı olabilecek birkaç link var.


Yukarıda yazmış olduğum gibi, arch /x86/boot /video.c kaynak kodu dosyasında tanımlanan set_video fonksiyonundan başlayacağız. İlk olarak, video modunu boot_params.hdr yapısından başlatarak başladığını görebiliriz:



          u16 mode = boot_params.hdr.vid_mode;

Biz bunu copy_boot_params fonksiyonu ile doldurduk (önceki yazıda okuyabilirsiniz). Vid_mode, bootloader tarafından doldurulan zorunlu bir alandır. Bu konudaki bilgiyi, çekirdek boot protokolünde bulabilirsiniz:


           Offset  Proto   Name        Meaning
           /Size
           01FA/2  ALL     vid_mode    Video mode control



Linux çekirdeği boot protokolünü okuyabildiğimiz için:



       vga=<mode>
           <mode> here is either an integer (in C notation, either
           decimal, octal, or hexadecimal) or one of the strings
           "normal" (meaning 0xFFFF), "ext" (meaning 0xFFFE) or "ask"
           (meaning 0xFFFD).  This value should be entered into the
           vid_mode field, as it is used by the kernel before the command
           line is parsed.


Böylece gruba veya başka bir bootloader yapılandırma dosyasına vga seçeneğini ekleyebiliriz ve bu seçeneği çekirdek komut satırına geçireceğiz. Bu seçenek, açıklamada belirtildiği gibi farklı değerlere sahip olabilir. Örneğin, bir tam sayı 0xFFFD olabilir . Vga'ya geçerseniz, şu şekilde bir menü göreceksiniz:


 ![alt tag](https://camo.githubusercontent.com/fc6d3f91001fe97d4036d3d37bbdac89a3e5d3b9/687474703a2f2f6f6935392e74696e797069632e636f6d2f656a637a38312e6a7067)


Bir video modu seçmenizi isteyecektir. Uygulanmasına bakacağız ancak buna  geçmeden önce başka şeylere bakmamız gerekiyor.



Çekirdek Veri Tipleri


Daha önce, çekirdek kurulum kodunda u16 gibi farklı veri türlerinin tanımlarını gördük. Çekirdek tarafından sağlanan birkaç veri türüne bakalım:


              Type      char   short   int  long    u8    u16    u32   u64

              Size       1      2      4      8      1     2      4     8   


Çekirdeğin kaynak kodunu okursanız bunları çok sık görürsünüz ve bu nedenle bunları hatırlamak güzel olur.



Heap API


Set_video fonksiyonundaki boot_params.hdr dosyasından vid_mode'u edintikten sonra, RESET_HEAP ffonksiyonuna yapılan çağrıya bakabiliriz. RESET_HEAP, boot.h'de tanımlanan bir makrodur. Şu şekilde tanımlanmıştır:



           #define RESET_HEAP() ((void *)( HEAP = _end ))



İkinci bölümü okuduysanız, heap'i init_heap fonksiyonu ile başlattığımızı hatırlayacaksınız. Boot.h dosyasında tanımlanan heap için birkaç yardımcı fonsiyonumuz var. Onlar:




            #define RESET_HEAP()



Yukarıda gördüğümüz gibi burada _end sadece extern char _end []; olduğu yerde HEAP değişkenini _end ile eşitleyerek heap'i sıfırlar; 


Bir sonraki GET_HEAP makrosu;




             #define GET_HEAP(type, n) \
                  ((type *)__get_heap(sizeof(type),__alignof__(type),(n)))



Heap ayırma için. dahili fonksiyon  __get_heap'i 3 parametreyle çağırır:



- Tahsis edilmesi gereken bayt cinsinden bir boyut
- __alignof __ (type), bu türdeki değişkenlerin nasıl hizalandığını gösterir
- n ne kadar öğe ayıracağını söyler



__get_heap'in uygulanması:


                
              static inline char *__get_heap(size_t s, size_t a, size_t n)
             {
                 char *tmp;

                 HEAP = (char *)(((size_t)HEAP+(a-1)) & ~(a-1));
                 tmp = HEAP;
                 HEAP += s*n;
                 return tmp;
              }



İleride onun kullanımını göreceğiz:



             saved.data = GET_HEAP(u16, saved.x * saved.y);



__get_heap'in nasıl çalıştığını anlamaya çalışalım. HEAP (RESET_HEAP () sonrasında _end ile aynıdır) a parametresine göre hizalı belleğin adresini görebiliriz. Bundan sonra hafıza adresini HEAP'den tmp değişkenine kaydettik, HEAP'i tahsis edilen bloğun sonuna taşıdık ve tahsis edilen hafızanın başlangıç adresi olan tmp'yi döndürdük.



Ve son fonksiyon:


                   static inline bool heap_free(size_t n)
                  {
                      return (int)(heap_end - HEAP) >= (int)n;
                  }



HEAP değerini heap_end'den (önceki bölümde hesapladık) çıkarır ve n için yeterli bellek varsa 1 döndürür.


Bu kadar. Artık heap için basit bir API sahibiyiz ve video modunu ayarlayabiliyoruz.




