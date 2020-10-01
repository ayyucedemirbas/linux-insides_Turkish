## Kontrol Grupları

### Giriş

`Cgroups` , işlemci süresi, grup başına işlem sayısı, kontrol grubu başına bellek miktarı veya bir işlem veya işlem kümesi için bu tür kaynakların kombinasyonu 
gibi kaynakları tahsis etmemize olanak tanıyan Linux çekirdeği tarafından sağlanan özel bir mekanizmadır. Gruplar hiyerarşik olarak düzenlenmiştir ve burada bu 
mekanizma, hiyerarşik olduklarından ve alt grupların ebeveynlerinden belirli parametreler kümesini devraldıkları için olağan süreçlere benzer. Ama aslında aynı 
değiller. Normal süreç ağacı her zaman tek iken, kontrol gruplarının birçok farklı hiyerarşisinin aynı anda var olabileceği `Cgroups` ve normal süreçler arasındaki 
temel farklar. Her bir kontrol grubu hiyerarşisi, kontrol grubu alt sistemleri setine eklendiği için bu sıradan bir adım değildi.

Bir 'kontrol grubu alt sistemi' (control group subsystem), bir işlemci süresi veya [pids](https://en.wikipedia.org/wiki/Process_identifier) sayısı  ​​veya başka bir deyişle bir "kontrol grubu" için işlem sayısı gibi bir tür kaynağı temsil eder. Linux çekirdeği, on iki "kontrol grubu alt sistemi" ni takip etmek için destek sağlar:

* `cpuset` - bir gruptaki görevlere ayrı işlemcileri ve bellek düğümlerini atar;
* `cpu` - cgroup görevlerinin işlemci kaynaklarına erişimini sağlamak için zamanlayıcıyı kullanır;
* `cpuacct` - bir grup tarafından işlemci kullanımıyla ilgili raporlar oluşturur;
* `io` - [cihazları engelle] (https://en.wikipedia.org/wiki/Device_file) için okuma / yazma sınırı belirler;
* `memory` - bir gruptaki görev (ler) tarafından bellek kullanımı sınırını belirler;
* `devices` - bir gruptaki görev (ler) ile cihazlara erişime izin verir;
* `freezer` - bir gruptaki görev (ler) i askıya almaya / sürdürmeye izin verir;
* `net_cls` - bir gruptaki görevlerden ağ paketlerini işaretlemeye izin verir;
* `net_prio` - bir grup için ağ arabirimi başına ağ trafiğinin önceliğini dinamik olarak ayarlamanın bir yolunu sağlar;
* `perf_event` - bir gruba [performans olaylarına] (https: //en.wikipedia.org/wiki/Perf_ \ (Linux \)) erişim sağlar;
* `hugetlb` - bir grup için [büyük sayfalar] (https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt) desteğini etkinleştirir;
* `pid` - bir gruptaki işlemlerin sayısını sınırlar.
