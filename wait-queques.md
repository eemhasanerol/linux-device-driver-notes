# Linux Kernel: Wait Queue (Bekleme Kuyruğu)

### 1. Wait Queue Nedir?

Wait Queue, Linux çekirdeğinde bir işlemin beklediği olay (örneğin veri gelmesi) gerçekleşene kadar **uyutularak** bekletildiği yapıdır.

Bu mekanizmanın temel amacı **işlemciyi (CPU) boşuna yormamaktır.**

Bir veriyi beklerken işlemciye sürekli *"Geldi mi?"* diye sormak (Polling) yerine; sisteme *"Veri gelince beni uyandır"* komutu verilir ve işlem uyku moduna alınır. Böylece işlemci, bekleme süresince boşa dönmez ve diğer işlerle ilgilenebilir.

### 2. Nasıl Çalışır? (Arka Plan)

1.  **Uykuya Geçiş:** Sürücü kodu `wait_event` fonksiyonunu çağırdığında, işletim sistemi bu görevi "Çalışanlar" listesinden çıkarıp "Uyuyanlar" listesine taşır.
2.  **Donma Anı:** Görev, işlemci üzerindeki hakkından vazgeçer (`schedule` çağrısı). Kod, tam olarak `wait_event` satırında donar ve asılı kalır.
3.  **Uyandırma:** Beklenen olay (örneğin donanım kesmesi/interrupt) gerçekleştiğinde, sürücünün diğer yarısı `wake_up` fonksiyonunu çağırır.
4.  **Devam Etme:** Uyanan görev kaldığı yerden değil, **şartı tekrar kontrol ederek** çalışmaya devam eder.

### 3. Kod Şablonu

Bu mekanizma, veriyi bekleyen (Process Context) ve veriyi sağlayan (Interrupt Context) iki tarafın işbirliğiyle çalışır.

**Tanımlama:**
```c
DECLARE_WAIT_QUEUE_HEAD(my_queue); // Bekleme odası
int data_ready = 0;                // Şart bayrağı (Flag)
```

#### A. Uyuyan Taraf (Driver Read Fonksiyonu)

```c
/* Veri 1 olana kadar uyu. CPU harcama. */
/* interruptible: Sinyal gelirse (CTRL+C) uyanabilmesini sağlar. */
if (wait_event_interruptible(my_queue, (data_ready == 1))) {
    return -ERESTARTSYS; // Zorla uyandırıldık (Sinyal ile), işlemi iptal et.
}

// Buraya kod indiyse; uyanmışız ve veri hazır demektir.
data_ready = 0; // Veriyi al ve bayrağı indir.
```

#### B. Uyuyan Taraf (Driver Read Fonksiyonu)

```c
/* Donanım işini bitirdi, veriyi koydu. */
data_ready = 1;                   // 1. Önce şartı sağla
wake_up_interruptible(&my_queue); // 2. Sonra uyuyanları dürt
```

## 4. Kritik Notlar ⚠️
Interruptible Kullanımı: Her zaman wait_event_interruptible kullanmaya özen gösterin. Eğer donanım bozulur ve veri asla gelmezse, "interruptible" olmayan bir süreç sonsuza kadar uyur ve program kill -9 ile bile kapatılamaz (Zombi süreç).

Koşul Kontrolü: Wait Queue mekanizması, uyanır uyanmaz koşulu (data_ready == 1) bir kez daha kontrol eder. Eğer koşul sağlanmamışsa (başka bir thread veriyi kaptıysa), kod tekrar uykuya yatar.

Temel Yapı: Kullandığımız mutex ve semaphore gibi daha üst seviye yapıların temelinde de aslında bu Wait Queue mimarisi yatar.

5. İleri Seviye: Mimari Problemler ve Çözümleri 🚀
Mülakatlarda veya yüksek performanslı sistem tasarımlarında karşımıza çıkan 3 kritik senaryo:

A. The Lost Wake-Up Problem (Kayıp Uyandırma)
Sorun: Bir işlem tam "uyumaya" hazırlanırken (henüz listeye girmeden), donanımın wake_up sinyali göndermesi durumudur. Sinyal "boşluğa" gider ve işlemci daha sonra uyku moduna geçince sonsuza kadar uyur. Çözüm: Linux Kernel bu yüzden önce süreci "uyuyor" olarak işaretler (set_current_state), sonra şartı kontrol eder. Böylece sinyal arada kaybolmaz.

B. The Thundering Herd Problem (Gürleyen Sürü)
Sorun: Bir kaynak serbest kaldığında, onu bekleyen 100 işlemin aynı anda uyandırılmasıdır (wake_up_all). Hepsi işlemciye saldırır ama sadece 1'i kaynağı alır, 99'u geri yatar. Bu durum sistemi anlık olarak kilitler (Context Switch Storm). Çözüm: WQ_FLAG_EXCLUSIVE bayrağı kullanılır. Bu bayrak sayesinde Kernel, kuyruğun başındaki sadece 1 işlemi uyandırır, diğerlerini rahatsız etmez.

C. D State (Uninterruptible Sleep / Zombi Süreç)
Sorun: Donanım hatası nedeniyle asla gelmeyen bir sinyali UNINTERRUPTIBLE (kesilemez) modda bekleyen süreçtir. kill -9 komutuna bile cevap vermez çünkü sinyalleri işleyecek koda dönemez. Yüksek Load Average yaratır. Çözüm: Kritik donanım beklemelerinde mutlaka wait_event_timeout (zaman aşımı) kullanılmalıdır.
