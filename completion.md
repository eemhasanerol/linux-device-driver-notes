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

C

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
C. Stack Üzerinde Tanımlama (Fonksiyon İçi - Yerel)
Mantık: "Bu çok kısa bir iş. Fonksiyonun içinde birini bekleyeceğim, iş bitince de bu değişkenle işim kalmayacak, hemen silinsin."

Ne Zaman Kullanılır: Hızlı, tek seferlik ve fonksiyon dışına çıkmayacak beklemelerde.

Hafıza Bölgesi: Stack (Yığın).

Ömür: Sadece o fonksiyonun süslü parantezleri { } arasında yaşar.

⚠️ KRİTİK TEHLİKE: Eğer donanım işini bitirmeden (complete çağırmadan) bu fonksiyon return ederse, değişken hafızadan silinir. Donanım daha sonra silinmiş hafızaya yazmaya çalışırsa Kernel Panic oluşur. Bu yüzden _ONSTACK makrosu ile sisteme haber vermek zorunludur.

C

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

