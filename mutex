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
