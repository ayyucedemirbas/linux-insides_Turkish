Çekirdek Önyükleme Süreci Bölüm 2

Çekirdek Kurulumunun Ýlk Adýmlarý

Önceki bölümde linux çekirdeði içerisine dalmaya baþladýk ve çekirdek kurulum kodunun ilk bölümünü gördük. Arch/x86/boot/main.c'den main fonksiyonuna (C'de yazýlan ilk fonksiyon) ilk çaðrýda durduk.
Bu bölümde, çekirdek kurulum kodunu araþtýrmaya ve 
- korumalý modun ne olduðunu, 
- geçiþ için bazý hazýrlýklarý, 
- heap ve konsol baþlatma, 
- bellek algýlama, CPU doðrulama, klavye baþlatma 
- ve çok daha fazlasýný göreceðiz. 

Öyleyse, devam edelim.

Korumalý Mod

Yerel Intel64 Long Mode'a geçmeden önce, çekirdek CPU'yu korumalý moda geçirmelidir.

Korumalý mod nedir? Korumalý mod, ilk olarak 1982'de x86 mimarisine eklendi ve 80286 iþlemciden Intel 64'e ve Long Mode'a gelene kadar ana iþlemci modlarýydý.

 Real Mode'dan uzaklaþmanýn baþlýca nedeni RAM'e çok sýnýrlý eriþim olmasýdýr. Önceki bölümden hatýrladýðýnýz gibi, Real Mode'da yalnýzca 2 üzeri 20 byte veya 1 Megabayt, bazen sadece 640 Kilobayt RAM mevcut.

 Korumalý mod birçok deðiþiklik getirdi, ancak ana farklardan biri bellek yönetimindeki farktýr. 20 bit adres veriyolu yerine 32 bitlik adres veri yolu getirildi. 4 Gigabyte bellek ve 1 Megabayt gerçek moda eriþime izin verildi. Ayrýca, sonraki bölümlerde okuyabileceðiniz sayfalama desteði eklendi.

Korumalý modda bellek yönetimi, neredeyse baðýmsýz iki parçaya bölünür:

- Bölütleme
- Sayfalama
Burada sadece bölütlemeyi göreceðiz. Disk belleði bir sonraki bölümde ele alýnacaktýr. 

Önceki bölümde okuyabileceðiniz gibi, adresler gerçek modda iki kýsýmdan oluþur: 

- Segmentin taban adresi
- Segment tabanýndan ofset

Ve eðer bu iki parçayý biliyorsak fiziksel adresi bulabiliriz:

            PhysicalAddress = Segment Selector * 16 + Offset


Bellek bölümlemesi korumalý modda tamamen yeniden yapýlandýrýldý. 64 Kilobayt sabit boyutlu bölüm yok. Bunun yerine, her bir bölümün boyutu ve konumu, Bölüm Tanýmlayýcýlarý adý verilen iliþkili bir veri yapýsý tarafýndan tanýmlanýr. Bölüm tanýmlayýcýlarý, Genel Tanýmlayýcý Tablosu (GDT) adý verilen bir veri yapýsýnda saklanýr.

GDT, hafýzada bulunan bir yapýdýr. Belleðinde sabit bir yeri yoktur, böylece adresi GDTR özel kaydýnda saklanýr. Daha sonra, GDT'nin Linux çekirdeði kodunda yüklendiðini göreceðiz. Belleðe yüklemek aþaðýdaki gibi bir iþlem olacak:

            lgdt gdt

Burada lgdt komutu global tanýmlayýcý tablosunun taban adresini ve sýnýrýný (büyüklüðü) GDTR kaydýna yükler. GDTR, 48 bitlik bir kayýttýr ve iki bölümden oluþur:

- Genel tanýmlayýcý tablosunun boyutu (16-bit);
- Küresel tanýmlayýcý tablosunun adresi (32-bit).

Yukarýda belirtildiði gibi GDT, bellek bölümlerini tanýmlayan bölüm tanýmlayýcýlarý içerir. Her tanýmlayýcý 64-bit boyutundadýr. Bir tanýmlayýcýnýn genel þemasý þöyledir:

         31          24        19      16              7            0
         ------------------------------------------------------------
         |             | |B| |A|       | |   | |0|E|W|A|            |
         | BASE 31:24  |G|/|L|V| LIMIT |P|DPL|S|  TYPE | BASE 23:16 | 4
         |             | |D| |L| 19:16 | |   | |1|C|R|A|            |
         ------------------------------------------------------------
         |                             |                            |
         |        BASE 15:0            |       LIMIT 15:0           | 0
         |                             |                            |
         ------------------------------------------------------------

Endiþelenmeyin, Real Mode'dan sonra biraz korkutucu olduðunu biliyorum, ama kolay. Örneðin LIMIT 15: 0, Tanýmlayýcýnýn 0-15 bitinin limit deðeri içerdiði anlamýna gelir. Geri kalan kýsmý LIMIT 19: 16'da. Yani Limit boyutu 0-19, yani 20 bittir. Þimdi ona bir göz atalým:

   Sýnýr [20 bit] 0-15,16-19 bit. Bu, length_of_segment - 1'i tanýmlar. G (Granularity) bitine baðlýdýr.
   - G (bit 55) 0 ise ve segment sýnýrý 0 ise, segmentin boyutu 1 Byte'dir
   - G; 1 ve segment sýnýrý 0 ise, segmentin boyutu 4096 Byte'dir
   - G; 0 ve segment sýnýrý 0xfffff ise, segmentin boyutu 1 Megabayt
   - G; 1, segment sýnýrý 0xfffff ise, segmentin boyutu 4 Gigabyte'dýr

Bu demektir ki, eðer



