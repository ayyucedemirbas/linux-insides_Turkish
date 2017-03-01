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






