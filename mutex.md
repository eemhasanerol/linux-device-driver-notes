# Mutex (Mutual Exclusion) Notları

## 1. Mutex Nedir? (Uyuyan Kilit)

Spinlock'un "dönen" (busy-wait) yapısının aksine, **Mutex (Mutual Exclusion)** "uyuyan" (sleep) bir kilitleme mekanizmasıdır.

Mantığı şudur:

> "Kapı kilitli mi? Tamam, kapı önünde dönüp durmayayım. Beni uyutun, kilit açılınca beni uyandırın. O sırada işlemci (CPU) başka işler yapsın."

Bu özellik sayesinde Mutex, işlemciyi gereksiz yere meşgul etmez.

* **Spinlock:** CPU kilidi tutar (CPU sürekli döner).
* **Mutex:** Task kilidi tutar (Task uykuya geçer).

## 2. Mutex Nasıl Çalışır? (İç Yapıdaki İroni)

Mutex'in iç yapısında ilginç bir detay vardır: **Mutex, kendi içinde bir Spinlock kullanır.**

`struct mutex` yapısına baktığımızda şunu görürüz (Sadece hata ayıklama kodları temizlenmiş, öz hali):

```c
struct mutex {
    atomic_long_t owner;        // Kilidin Sahibi
    spinlock_t wait_lock;       // Bekleme Listesi Kilidi
    struct list_head wait_list; // Bekleme Listesi
};
```
Bu yapıdaki elemanların görevleri kritik önem taşır:

owner: Kilidi fiilen o an elinde tutan işlemi (process) temsil eder.

wait_list: Kilidi o an alamayan ve "yarışan" (contenders) diğer işlemlerin uyutularak sıraya dizildiği listedir.

wait_lock: İşte burası çok önemlidir. wait_list'e yeni bir işlem eklenirken veya oradan çıkarılırken listenin bozulmaması gerekir. Özellikle çok çekirdekli (SMP) sistemlerde bu listenin tutarlı kalmasını sağlamak için, liste erişimi içerideki bu Spinlock ile korunur.

Yani Özetle Süreç Şöyledir:

Bir task mutex_lock çağırır.

Kilit doluysa, task wait_list içine atılacaktır.

Task'ı listeye güvenli bir şekilde eklemek için önce kısa süreliğine wait_lock (spinlock) alınır.

Task listeye eklenir, spinlock bırakılır ve task uyku moduna (Sleep) geçirilir.

Kilit sahibi işini bitirip mutex_unlock dediğinde, listedeki sıradaki task uyandırılır.

## 3. Ne Zaman Kullanılır? (Spinlock vs. Mutex)

Spinlock ve Mutex arasındaki seçim, **"Ne kadar bekleyeceğim?"** ve **"Neredeyim?"** sorularına göre yapılır.

| Özellik | Spinlock | Mutex |
| :--- | :--- | :--- |
| **Bekleme Tipi** | Busy-Wait (Döner) | Sleep (Uyur) |
| **Context Switch** | Yok | Var (Maliyetli) |
| **Kullanım Yeri** | Çok kısa işler (< 1ms) | Uzun süren işler, IO işlemleri |
| **Interrupt (IRQ)** | **KULLANILIR** | **YASAKTIR** (Çünkü uyutur) |
| **CPU Kullanımı** | Yüksek (Dönerken %100) | Düşük (Uyurken %0) |

**Temel Kurallar:**

* **Interrupt (Kesme) İçindeysen:** Kesinlikle **Spinlock** kullanmalısın. Çünkü Interrupt handler uyuyamaz.
* **Uzun İşler Yapıyorsan:** Eğer kilitliyken `copy_from_user` yapacaksan, diskten veri bekleyeceksen veya TCP paketi bekleyeceksen **Mutex** kullanmalısın.



## 4. Mutex Tanımlama (Initialization)

Bir Mutex kullanılmadan önce mutlaka tanımlanmalıdır (Initialize). Kernel bunun için iki yöntem sunar:

**A. Statik Tanımlama (Derleme zamanı):**
Genellikle global değişkenler için kullanılır.

```c
static DEFINE_MUTEX(my_mutex);
```
**B. Dinamik Tanımlama (Çalışma zamanı):**
Genellikle kmalloc ile ayrılan veri yapıları içindeki mutexler için kullanılır (örneğin bir probe fonksiyonu içinde).

```c
struct fake_data {
    struct mutex mutex;
};
// ... probe fonksiyonu içinde ...
mutex_init(&data->mutex);
```

## 5. Kilitleme Fonksiyonları ve Farkları (Locking)
Mutex edinmek (kilitlemek) için 3 temel fonksiyon vardır. Hangi fonksiyonu seçtiğiniz, task'ın bekleme esnasında nasıl davranacağını belirler.

**A. mutex_lock(&lock)**

Task'ı kesintiye uğratılamaz (uninterruptible) bir uykuya sokar (TASK_UNINTERRUPTIBLE).Risk: Kilit bir sebepten asla açılmazsa, task sonsuza kadar donar. KILL sinyali bile işe yaramaz. Sadece kilit kısa süre tutulacaksa önerilir.

**B. mutex_lock_interruptible(&lock) (Önerilen ✅)**

Task'ı kesintiye uğratılabilir (interruptible) bir uykuya sokar.Task uyurken bir sinyal (örneğin kullanıcının Ctrl+C yapması) gelirse uyanır.Dönüş Değeri: Fonksiyon -EINTR dönerse, kilit alınamamış ve işlem bölünmüş demektir.

**C. mutex_lock_killable(&lock)**

Sadece task'ı gerçekten öldüren (fatal) sinyaller gelirse uyanır.Özet Tablo:FonksiyonUyku TürüSinyal Gelirse Ne Olur?mutex_lockUninterruptibleUyanmaz (Donar)mutex_lock_interruptibleInterruptibleUyanır (-EINTR döner)mutex_lock_killableKillableSadece ölürse uyanır

## 6. Kilit Açma ve Durum KontrolüKilidi açmak için tek bir fonksiyon vardır ve sadece kilidi alan (sahibi) çağırabilir:Cvoid mutex_unlock(struct mutex *lock);
Mutex'in o an kilitli olup olmadığını kontrol etmek için (örneğin debug amaçlı):C// Kilitliyse true, değilse false döner
static bool mutex_is_locked(struct mutex *lock);

## 7. Mutex Altın Kuralları (YASAKLAR)

Linux kernel’in `include/linux/mutex.h` dosyasında belirtilen zorunlu kurallar:

- **Tek Sahip:**  
  Aynı anda sadece bir task mutex’i tutabilir.

- **Sadece Sahibi Açabilir:**  
  Mutex’i kim aldıysa yalnızca o bırakabilir.

- **Recursive Kilit Yasak:**  
  Bir task mutex’i tutarken aynı mutex’i tekrar almaya çalışamaz (deadlock oluşur).

- **IRQ İçinde Kullanım Yasak:**  
  Mutex, IRQ handler, softirq, tasklet veya timer context içinde kullanılamaz.  
  Çünkü interrupt context **uyuyamaz**, mutex ise beklerken uykuya geçer.

- **Exit Yasağı:**  
  Bir task mutex’i tutarken exit edemez.

- **Bellek Serbest Bırakma Yasağı:**  
  Mutex tutuluyorken mutex'in bulunduğu bellek (`kfree`) serbest bırakılamaz.


```c
if (!mutex_trylock(&bar_mutex)) {
    /* Başarısız! Mutex zaten kilitli.
     * Burada uyumuyoruz, hemen başka işlere bakıyoruz.
     */
    return -EBUSY;
}
```

/* Buraya geldiysek kilidi başarıyla aldık demektir */
// ... kritik işlemler ...
mutex_unlock(&bar_mutex);
