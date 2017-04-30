
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


Video Modunu Ayarlama



Şimdi doğrudan video modu başlatmaya geçebiliriz. Set_video fonksiyonunda RESET_HEAP() çağrısında durduk. Sonraki include/uapi /linux/screen_info.h sodyası içinde tanımlanan boot_params.screen_info yapısında video modu parametrelerini saklayan store_mode_params çağrısıdır.


Store_mode_params fonksiyonuna bakarsak, fonksiyonun store_cursor_position fonksiyonuna yapılan çağrı ile başladığını görebiliriz. Fonksiyon adından anlayabileceğiniz gibi, imleç hakkında bilgi alır ve onu saklar.


Öncelikle store_cursor_position, AH = 0x3 olan biosreg türünde iki değişkeni başlatır ve 0x10 BIOS kesmesini çağırır. Kesme başarıyla çalıştırıldıktan sonra, DL ve DH kayıtlarındaki sıra ve sütunları döndürür. Satır ve sütun, boot_params.screen_info yapısından orig_x ve orig_y alanlarına depolanır.


Store_cursor_position yürütüldükten sonra, store_video_mode fonksiyonu çağrılır. Sadece geçerli video modunu alır ve boot_params.screen_info.orig_video_mode'da saklar.


Bundan sonra, geçerli video modunu kontrol eder ve video_segmentini ayarlar. BIOS, kontrolü boot sector'ına aktardıktan sonra, aşağıdaki adresler video belleği içindir:


       0xB000:0x0000   32 Kb   Monochrome Text Video Memory
       0xB800:0x0000   32 Kb   Color Text Video Memory


Bu nedenle, mevcut video modu tek renkli modda MDA, HGC veya VGA ise video_segment değişkenini 0xB000 olarak, mevcut video modu renkli modda ise 0xB800 olarak ayarlarız. Video bölümünün adresini ayarladıktan sonra font boyutu boot_params.screen_info.orig_video_points dosyasında şu şekilde saklanmalıdır:


         set_fs(0);
         font_size = rdfs16(0x485);
         boot_params.screen_info.orig_video_points = font_size;


Her şeyden önce set_fs fonksiyonuyla FS registerına 0 koyuyoruz. Önceki bölümde set_fs gibi fonksiyonlar gördük. Bunların hepsi boot.h dosyasında tanımlanmıştır. Ardından 0x485 adresinde bulunan değeri okur (bu bellek konumu yazı tipi boyutunu almak için kullanıldı) ve yazı tipi boyutunu boot_params.screen_info.orig_video_points olarak kaydederiz.


             x = rdfs16(0x44a);
             y = (adapter == ADAPTER_CGA) ? 25 : rdfs8(0x484)+1;


Ardından, 0x44a adresine ve 0x484 adresine göre sütun miktarını elde ediyoruz ve bunları boot_params.screen_info.orig_video_cols ve boot_params.screen_info.orig_video_lines adreslerinde saklıyoruz. Bundan sonra, store_mode_params'ın yürütülmesi tamamlanmıştır.



Ardından, ekran içeriğini yalnızca heap olarak kaydeden save_screen fonksiyonunu görebilirsiniz. Bu fonksiyon, satırlar ve sütunlar miktarı vb. gibi önceki fonksiyonlarda elde ettiğimiz tüm verileri toplar ve aşağıdaki gibi tanımlanmış saved_screen yapısında saklar:



             static struct saved_screen {
                 int x, y;
                 int curx, cury;
                 u16 *data;
              } saved;


Daha sonra heap'in bunun için boş alan olup olmadığını denetler:


              if (!heap_free(saved.x*saved.y*sizeof(u16)+512))
           return;


Yeterli ise heap alanını tahsis eder ve saved_screen'i içinde depolar.


Bir sonraki çağrı arch/x86/boot/video-mode.c'den probe_cards (0) 'dır. Tüm video_cards'ı geçer ve cards tarafından sağlanan modların sayısını toplar. İşte ilginç an, döngüyü görebiliyoruz:



             for (card = video_cards; card < video_cards_end; card++) {
                 /* collecting number of modes here */
              }


Ancak video_cards hiçbir yerde bildirilmedi. Cevap basittir: x86 çekirdek kurulum kodunda sunulan her video modu şu şekilde tanımlanmıştır:


            
             static __videocard video_vga = {
             .card_name  = "VGA",
              .probe      = vga_probe,
              .set_mode   = vga_set_mode,
               };


Burada __videocard bir makro:


            #define __videocard struct card_info __attribute__((used,section(".videocards")))


 card_info yapısı şu anlama gelir:



          struct card_info {
               const char *card_name;
               int (*set_mode)(struct mode_info *mode);
Video Modunu Ayarlama



Şimdi doğrudan video modu başlatmaya geçebiliriz. Set_video fonksiyonunda RESET_HEAP() çağrısında durduk. Sonraki include/uapi /linux/screen_info.h sodyası içinde tanımlanan boot_params.screen_info yapısında video modu parametrelerini saklayan store_mode_params çağrısıdır.


Store_mode_params fonksiyonuna bakarsak, fonksiyonun store_cursor_position fonksiyonuna yapılan çağrı ile başladığını görebiliriz. Fonksiyon adından anlayabileceğiniz gibi, imleç hakkında bilgi alır ve onu saklar.


Öncelikle store_cursor_position, AH = 0x3 olan biosreg türünde iki değişkeni başlatır ve 0x10 BIOS kesmesini çağırır. Kesme başarıyla çalıştırıldıktan sonra, DL ve DH kayıtlarındaki sıra ve sütunları döndürür. Satır ve sütun, boot_params.screen_info yapısından orig_x ve orig_y alanlarına depolanır.


Store_cursor_position yürütüldükten sonra, store_video_mode fonksiyonu çağrılır. Sadece geçerli video modunu alır ve boot_params.screen_info.orig_video_mode'da saklar.


Bundan sonra, geçerli video modunu kontrol eder ve video_segmentini ayarlar. BIOS, kontrolü boot sector'ına aktardıktan sonra, aşağıdaki adresler video belleği içindir:


       0xB000:0x0000   32 Kb   Monochrome Text Video Memory
       0xB800:0x0000   32 Kb   Color Text Video Memory


Bu nedenle, mevcut video modu tek renkli modda MDA, HGC veya VGA ise video_segment değişkenini 0xB000 olarak, mevcut video modu renkli modda ise 0xB800 olarak ayarlarız. Video bölümünün adresini ayarladıktan sonra font boyutu boot_params.screen_info.orig_video_points dosyasında şu şekilde saklanmalıdır:


         set_fs(0);
         font_size = rdfs16(0x485);
         boot_params.screen_info.orig_video_points = font_size;


Her şeyden önce set_fs fonksiyonuyla FS registerına 0 koyuyoruz. Önceki bölümde set_fs gibi fonksiyonlar gördük. Bunların hepsi boot.h dosyasında tanımlanmıştır. Ardından 0x485 adresinde bulunan değeri okur (bu bellek konumu yazı tipi boyutunu almak için kullanıldı) ve yazı tipi boyutunu boot_params.screen_info.orig_video_points olarak kaydederiz.


             x = rdfs16(0x44a);
             y = (adapter == ADAPTER_CGA) ? 25 : rdfs8(0x484)+1;


Ardından, 0x44a adresine ve 0x484 adresine göre sütun miktarını elde ediyoruz ve bunları boot_params.screen_info.orig_video_cols ve boot_params.screen_info.orig_video_lines adreslerinde saklıyoruz. Bundan sonra, store_mode_params'ın yürütülmesi tamamlanmıştır.



Ardından, ekran içeriğini yalnızca heap olarak kaydeden save_screen fonksiyonunu görebilirsiniz. Bu fonksiyon, satırlar ve sütunlar miktarı vb. gibi önceki fonksiyonlarda elde ettiğimiz tüm verileri toplar ve aşağıdaki gibi tanımlanmış saved_screen yapısında saklar:



             static struct saved_screen {
                 int x, y;
                 int curx, cury;
                 u16 *data;
              } saved;


Daha sonra heap'in bunun için boş alan olup olmadığını denetler:


              if (!heap_free(saved.x*saved.y*sizeof(u16)+512))
           return;


Yeterli ise heap alanını tahsis eder ve saved_screen'i içinde depolar.


Bir sonraki çağrı arch/x86/boot/video-mode.c'den probe_cards (0) 'dır. Tüm video_cards'ı geçer ve cards tarafından sağlanan modların sayısını toplar. İşte ilginç an, döngüyü görebiliyoruz:



             for (card = video_cards; card < video_cards_end; card++) {
                 /* collecting number of modes here */
              }


Ancak video_cards hiçbir yerde bildirilmedi. Cevap basittir: x86 çekirdek kurulum kodunda sunulan her video modu şu şekilde tanımlanmıştır:


            
             static __videocard video_vga = {
             .card_name  = "VGA",
              .probe      = vga_probe,
              .set_mode   = vga_set_mode,
               };


Burada __videocard bir makro:


            #define __videocard struct card_info __attribute__((used,section(".videocards")))


 card_info yapısı şu anlama gelir:



          struct card_info {
               const char *card_name;
               int (*set_mode)(struct mode_info *mode);
               int (*probe)(void);
               struct mode_info *modes;
               int nmodes;
               int unsafe;
               u16 xmode_first;
               u16 xmode_n;
           };               int (*probe)(void);
               struct mode_info *modes;
               int nmodes;
               int unsafe;
               u16 xmode_first;
               u16 xmode_n;
           };
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



######Video Modunu Ayarlama



Şimdi doğrudan video modu başlatmaya geçebiliriz. Set_video fonksiyonunda RESET_HEAP() çağrısında durduk. Sonraki include/uapi /linux/screen_info.h sodyası içinde tanımlanan boot_params.screen_info yapısında video modu parametrelerini saklayan store_mode_params çağrısıdır.


Store_mode_params fonksiyonuna bakarsak, fonksiyonun store_cursor_position fonksiyonuna yapılan çağrı ile başladığını görebiliriz. Fonksiyon adından anlayabileceğiniz gibi, imleç hakkında bilgi alır ve onu saklar.


Öncelikle store_cursor_position, AH = 0x3 olan biosreg türünde iki değişkeni başlatır ve 0x10 BIOS kesmesini çağırır. Kesme başarıyla çalıştırıldıktan sonra, DL ve DH kayıtlarındaki sıra ve sütunları döndürür. Satır ve sütun, boot_params.screen_info yapısından orig_x ve orig_y alanlarına depolanır.


Store_cursor_position yürütüldükten sonra, store_video_mode fonksiyonu çağrılır. Sadece geçerli video modunu alır ve boot_params.screen_info.orig_video_mode'da saklar.


Bundan sonra, geçerli video modunu kontrol eder ve video_segmentini ayarlar. BIOS, kontrolü boot sector'ına aktardıktan sonra, aşağıdaki adresler video belleği içindir:


       0xB000:0x0000   32 Kb   Monochrome Text Video Memory
       0xB800:0x0000   32 Kb   Color Text Video Memory


Bu nedenle, mevcut video modu tek renkli modda MDA, HGC veya VGA ise video_segment değişkenini 0xB000 olarak, mevcut video modu renkli modda ise 0xB800 olarak ayarlarız. Video bölümünün adresini ayarladıktan sonra font boyutu boot_params.screen_info.orig_video_points dosyasında şu şekilde saklanmalıdır:


         set_fs(0);
         font_size = rdfs16(0x485);
         boot_params.screen_info.orig_video_points = font_size;


Her şeyden önce set_fs fonksiyonuyla FS registerına 0 koyuyoruz. Önceki bölümde set_fs gibi fonksiyonlar gördük. Bunların hepsi boot.h dosyasında tanımlanmıştır. Ardından 0x485 adresinde bulunan değeri okur (bu bellek konumu yazı tipi boyutunu almak için kullanıldı) ve yazı tipi boyutunu boot_params.screen_info.orig_video_points olarak kaydederiz.


             x = rdfs16(0x44a);
             y = (adapter == ADAPTER_CGA) ? 25 : rdfs8(0x484)+1;


Ardından, 0x44a adresine ve 0x484 adresine göre sütun miktarını elde ediyoruz ve bunları boot_params.screen_info.orig_video_cols ve boot_params.screen_info.orig_video_lines adreslerinde saklıyoruz. Bundan sonra, store_mode_params'ın yürütülmesi tamamlanmıştır.



Ardından, ekran içeriğini yalnızca heap olarak kaydeden save_screen fonksiyonunu görebilirsiniz. Bu fonksiyon, satırlar ve sütunlar miktarı vb. gibi önceki fonksiyonlarda elde ettiğimiz tüm verileri toplar ve aşağıdaki gibi tanımlanmış saved_screen yapısında saklar:



             static struct saved_screen {
                 int x, y;
                 int curx, cury;
                 u16 *data;
              } saved;



Daha sonra heap'in bunun için boş alan olup olmadığını denetler:



              if (!heap_free(saved.x*saved.y*sizeof(u16)+512))
           return;


Yeterli ise heap alanını tahsis eder ve saved_screen'i içinde depolar.


Bir sonraki çağrı arch/x86/boot/video-mode.c'den probe_cards (0) 'dır. Tüm video_cards'ı geçer ve cards tarafından sağlanan modların sayısını toplar. İşte ilginç an, döngüyü görebiliyoruz:



             for (card = video_cards; card < video_cards_end; card++) {
                 /* collecting number of modes here */
              }


Ancak video_cards hiçbir yerde bildirilmedi. Cevap basittir: x86 çekirdek kurulum kodunda sunulan her video modu şu şekilde tanımlanmıştır:


            
             static __videocard video_vga = {
             .card_name  = "VGA",
              .probe      = vga_probe,
              .set_mode   = vga_set_mode,
               };


Burada __videocard bir makro:


            #define __videocard struct card_info __attribute__((used,section(".videocards")))


 card_info yapısı şu anlama gelir:



          struct card_info {
               const char *card_name;
               int (*set_mode)(struct mode_info *mode);
               int (*probe)(void);
               struct mode_info *modes;
               int nmodes;
               int unsafe;
               u16 xmode_first;
               u16 xmode_n;
           };
           
           
.videocards bölümündedir.Şimdi arch/ x86/ boot/setup.ld linker komut dosyasına bakalım:


           .videocards : {
             video_cards = .;
              *(.videocards)
              video_cards_end = .;
             }

Bu, video_cards'ın sadece bir bellek adresi olduğu ve tüm card_info yapılarının bu segmente yerleştirildiği anlamına gelir. Bu, tüm card_info yapılarının video_cards ve video_cards_end arasına yerleştirildiği anlamına gelir; dolayısıyla, hepsini dolaşmak için bir döngü kullanabilirsiniz. Probe_cards'ın çalıştırılmasından sonra doldurulmuş nmodes'la static __videocard video_vga (video modlarının sayısı) gibi tüm yapılarımız var.

set_mode fonksiyonu video-mode.c'de tanımlanır ve video modlarının sayısı olan tek bir parametre olan mode'u alır. (Menüden veya setup_video'nun başlangıcında çekirdek setup kodunun header dosyasından alırız).


Set_mode fonksiyonu modu kontrol eder ve raw_set_mode fonksiyonunu çağırır. raw_set_mode,  seçilen kart, yani card-> set_mode (struct mode_info *) için set_mode fonksiyonunu çağırır. Bu fonksiyona card_info yapısından erişebiliriz. Her video modu, video moduna bağlı olarak doldurulan değerlerle bu yapıyı tanımlar (örneğin vga için, video_vga.set_mode fonksiyonu. Yukarıdaki vga için card_info yapısının örneğine bakın). Video_vga.set_mode, vga modunu kontrol eden ve ilgili fonksiyonu çağıran vga_set_mode'dur:

          static int vga_set_mode(struct mode_info *mode)
{
	vga_set_basic_mode();

	force_x = mode->x;
	force_y = mode->y;

	switch (mode->mode) {
	case VIDEO_80x25:
		break;
	case VIDEO_8POINT:
		vga_set_8font();
		break;
	case VIDEO_80x43:
		vga_set_80x43();
		break;
	case VIDEO_80x28:
		vga_set_14font();
		break;
	case VIDEO_80x30:
		vga_set_80x30();
		break;
	case VIDEO_80x34:
		vga_set_80x34();
		break;
	case VIDEO_80x60:
		vga_set_80x60();
		break;
	}
	return 0;
}


Video modunu ayarlayan her fonksiyon sadece AH register'ında belirli bir değere sahip 0x10 BIOS interrupt'ını çağırır.


Video modunu ayarladıktan sonra onu boot_params.hdr.vid_mode'a geçiririrz.


Sonraki vesa_store_edid çağrıldı. Bu fonksiyon, basitçe çekirdek kullanımı için EDID (Genişletilmiş Görüntü Tanımlama Verileri) bilgilerini saklar. Bu işlemden sonra store_mode_params tekrar çağrılır. Son olarak, do_restore ayarlıysa, ekran önceki bir duruma geri yüklenir.

