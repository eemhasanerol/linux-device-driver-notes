# Linux Kernel Completions (Tamamlanma Bekleyicileri)

## 1. Giriş: Completion Nedir?

**Completion**, bir veya daha fazla iş parçacığının (thread), çekirdek içindeki bir aktivitenin belirli bir noktaya veya duruma gelmesini **beklemesi** gerektiğinde kullanılan mekanizmadır.

Anlamsal olarak `pthread_barrier()` yapısına benzer.

> **Temel Amaç:** Race Condition riski olmadan, bir işin bitmesini beklemektir.

---

## 2. Neden İhtiyaç Duyarız? (Yanlış ve Doğru Yöntemler)

Linux çekirdek geliştiricileri, bekleme işlemleri için **Locks/Semaphores** veya **Busy-Loops** yanlış kullanılmasını önlemek amacıyla Completion mekanizmasını geliştirmiştir.

### ⚠️ Kaçınılması Gerekenler
Bir kod yazarken aklınıza şunlar geliyorsa, **yanlış yoldasınız** demektir:
* "Buraya bir `while` döngüsü koyayım, iş bitene kadar dönsün." (Busy-Wait)
* "Döngü içine `yield()` veya `msleep(1)` koyayım da diğer iş bitene kadar oyalansın."

Bu yöntemler işlemciyi verimsiz kullanır ve kodun niyetini belirsizleştirir.

### ✅ Doğru Yöntem: Completion
Bunun yerine `wait_for_completion()` ve `complete()` kullanmak çok daha verimlidir çünkü:
1.  **Net Niyet:** Kodu okuyan kişi "Burada bir bekleme/senkronizasyon var" diye hemen anlar.
2.  **Yüksek Verim:** Düşük seviyeli zamanlayıcı (scheduler) altyapısını kullanır. Bekleyen thread, gerçekten sonuç gerekene kadar **uyutulur** (Sleep). İşlemci o sırada başka işleri yapar.

---

## 3. Altyapı ve Çalışma Mantığı

Completion mekanizması, Linux Zamanlayıcısının (Scheduler) **`waitqueue`** ve **`wakeup`** altyapısı üzerine inşa edilmiştir.

Olay, teknik olarak `struct completion` yapısı içindeki basit bir bayrağa indirgenmiştir: **`done`**.

`include/linux/completion.h` içinde tanımlı yapı şöyledir:

```c
struct completion {
    unsigned int done;      // İş bitti mi bayrağı (0: Bitmedi, >0: Bitti)
    wait_queue_head_t wait; // Bekleyenlerin uyuduğu kuyruk
};
```
### Çalışma Süreci (Workflow)

Sistem döngüsü şu 3 temel adımda işler:

1.  **Wait (Bekleme):**
    `wait_for_completion` çağıran thread, `wait_queue` (bekleme kuyruğu) listesine eklenir ve işlemci tarafından **uyutulur** (Context Switch -> Sleep).

2.  **Done (Kontrol):**
    `struct completion` yapısındaki `done` bayrağı kontrol edilir.
    * Eğer `done == 0`: İş bitmemiştir, uyumaya devam eder.
    * Eğer `done > 0`: İş bitmiştir, beklemeden devam eder.

3.  **Signal (Sinyal/Uyandırma):**
    İş bittiğinde (`complete()` çağrıldığında), `done` bayrağı arttırılır (`done++`) ve kuyrukta uyuyan thread **uyandırılır** (Wake Up -> Task Running).

## 4. Completion Initialization ve Hafıza Mantığı 

Bir completion değişkenini nerede tanımlayacağınız, kodunuzun mimarisine ve değişkenin **ömrüne (lifetime)** bağlıdır. Yanlış seçim, birden fazla cihaz taktığınızda sistemin karışmasına veya hafıza hatalarına yol açar.

### A. Statik Tanımlama (Global Değişken)
**Mantık:** "Bu sürücüden sistemde sadece bir tane olacak ve değişken sürücü yüklü olduğu sürece hep orada durmalı."

* **Ne Zaman Kullanılır:** Basit modüllerde veya sistem genelinde tek bir kaynağı beklerken.
* **Hafıza Bölgesi:** `.data` veya `.bss` (Programın statik hafızası).
* **Ömür:** Modül yüklendiği (`insmod`) andan, silindiği (`rmmod`) ana kadar yaşar.

```c
#include <linux/completion.h>

/* Global Scope */
// Sürücü çalıştığı sürece bu değişken sabittir.
static DECLARE_COMPLETION(setup_done);

void my_function(void) {
    wait_for_completion(&setup_done);
}
```
B. Dinamik Tanımlama (Struct İçinde - Heap)
Mantık: "Bilgisayara aynı USB cihazından 5 tane takılabilir. Her cihazın KENDİNE ÖZEL bekleme bayrağı olmalı."

Eğer global değişken kullanırsanız, 1. cihazın işi bittiğinde yanlışlıkla 2. cihazı bekleyen kodu uyandırır. Bu yüzden her cihazın kendi veri yapısı (struct) içinde, kendine özel bir completion olmalıdır.

Ne Zaman Kullanılır: Profesyonel donanım sürücülerinde (USB, PCI, Platform Drivers).

Hafıza Bölgesi: Heap (kmalloc ile ayrılan alan).

Ömür: Cihaz takıldığında (probe) başlar, çıkarıldığında (remove) biter.

```c

struct my_device_data {
    int id;
    struct completion op_done; // Her cihazın kendi özel bayrağı
};

static int my_probe(struct platform_device *pdev)
{
    // 1. Her yeni cihaz için hafızadan taze yer ayır
    struct my_device_data *data;
    data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);

    // 2. O taze yerdeki completion'ı başlat (done = 0)
    // Bunu yapmazsanız içindeki çöp değerler yüzünden kod çalışmaz.
    init_completion(&data->op_done);

    return 0;
}
```
C. Stack Üzerinde Tanımlama (Fonksiyon İçi - Yerel)
Mantık: "Bu çok kısa bir iş. Fonksiyonun içinde birini bekleyeceğim, iş bitince de bu değişkenle işim kalmayacak, hemen silinsin."

Ne Zaman Kullanılır: Hızlı, tek seferlik ve fonksiyon dışına çıkmayacak beklemelerde.

Hafıza Bölgesi: Stack (Yığın).

Ömür: Sadece o fonksiyonun süslü parantezleri { } arasında yaşar.

⚠️ KRİTİK TEHLİKE: Eğer donanım işini bitirmeden (complete çağırmadan) bu fonksiyon return ederse, değişken hafızadan silinir. Donanım daha sonra silinmiş hafızaya yazmaya çalışırsa Kernel Panic oluşur. Bu yüzden _ONSTACK makrosu ile sisteme haber vermek zorunludur.

```c
void my_temp_function(void)
{
    // Stack üzerinde geçici tanımlama
    // Kernel'e "Bu stack üzerinde, dikkat et" diyoruz.
    DECLARE_COMPLETION_ONSTACK(gecici_comp);

    baslat_islem(&gecici_comp);
    
    // Fonksiyon bitmeden, işin %100 bittiğinden emin olmalıyız!
    wait_for_completion(&gecici_comp);
    
    // Fonksiyon bitince "gecici_comp" yok olur.
}
```


## 5. Temel API Fonksiyonları (The Arsenal)

Bir completion nesnesi ile yapabileceğiniz 4 temel işlem vardır: **Bekleme**, **Haber Verme**, **Sıfırlama** ve **Kontrol Etme**.

### A. Bekleme Fonksiyonları (Waiting)
Bu fonksiyonları çağıran kod (Thread), sinyal gelene kadar işlemciyi bırakır ve **UYUR** (Block).

| Fonksiyon | Açıklama ve Kullanım Yeri |
| :--- | :--- |
| `wait_for_completion(&x)` | **⚠️ Riskli:** Sonsuza kadar bekler. Eğer donanım bozulur ve cevap vermezse, bu process "Zombi" olur (`kill -9` ile bile ölmez). Sadece kesinlikle güvenilen kısa işlerde kullanılmalı. |
| `wait_for_completion_interruptible(&x)` | **Kullanıcı Dostu:** Beklerken `Ctrl+C` gibi sinyallerle uyandırılabilir. Eğer sinyal ile bölünürse `-ERESTARTSYS` döner. |
| `wait_for_completion_timeout(&x, timeout)` | **✅ En Güvenlisi:** Belirtilen süre (`timeout`) dolarsa, iş bitmese bile uyanır. Donanım sürücülerinde (I2C, SPI, DMA) **mutlaka bu kullanılmalıdır.** |
| `wait_for_completion_killable(&x)` | Sadece süreci öldüren (fatal) sinyallerle uyanır. |

> **Timeout Kullanımı İçin Örnek:**
> `timeout` parametresi "Jiffies" (Zaman dilimi) cinsindendir. Saniye cinsinden vermek için `msecs_to_jiffies()` kullanın.

```c
unsigned long kalan_zaman;
// 3 Saniye bekle
kalan_zaman = wait_for_completion_timeout(&dev->done, msecs_to_jiffies(3000));

if (kalan_zaman == 0) {
    printk("HATA: Donanım cevap vermedi (Timeout)!\n");
    return -ETIMEDOUT;
}
// Başarılı: kalan_zaman > 0
```

### B. Haber Verme Fonksiyonları (Signaling)
Bu fonksiyonlar, kuyrukta uyuyan kodları UYANDIRIR. Genellikle Interrupt (IRQ) Handler içinden çağrılır.
complete(&x): Kuyrukta bekleyen sadece bir task'ı uyandırır. done sayacını 1 artırır. %99 durumda bunu kullanacaksınız.
complete_all(&x): Bu olayı bekleyen herkesi (tüm threadleri) aynı anda uyandırır. done sayacını UINT_MAX yapar. ### C. Sıfırlama (Re-Initialization)
Completion mekanizması varsayılan olarak "tek atımlık" (One-shot) çalışır. İşlem bir kez complete() edildiğinde done sayacı artar (Örn: 1 olur).Eğer aynı değişkeni bir döngü içinde tekrar kullanacaksanız (örneğin sürekli veri paketi yolluyorsanız), her turda sayacı sıfırlamanız gerekir. Aksi takdirde wait fonksiyonu "Bu zaten bitmiş" der ve beklemeden geçer.C// done bayrağını tekrar 0 yapar.

```c
// DİKKAT: Bunu yaparken başka bir thread'in beklemediğinden emin olun (Race Condition).
reinit_completion(&dev->done);
```

### D. Beklemesiz Kontrol (Non-Blocking Check)Bazen "Uyuma lüksüm yok, sadece bitmiş mi diye bakıp çıkacağım" dersinizC// Eğer iş bittiyse (done > 0) true döner ve hakkı kullanır (done--).
```c
// Eğer iş bitmediyse, UYUMAZ, hemen false döner.
if (try_wait_for_completion(&dev->done)) {
    // İş bitmiş, devam et
} else {
    // İş bitmemiş, bekleyemem, başka işe bakayım
}
```
```c
C// Sadece durumu kontrol eder, hiçbir şeyi değiştirmez (done değerine dokunmaz).
bool bitti_mi = completion_done(&dev->done);
```
