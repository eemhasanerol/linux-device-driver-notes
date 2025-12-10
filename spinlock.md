# Spinlocks – Öğretici Anlatım

Bu doküman, çekirdek programlamada spinlock kavramını **sıfırdan**, **basit**, **teknik olarak doğru** ve **adım adım öğretici** bir şekilde açıklamak amacıyla hazırlanmıştır.

---

## 1. Başlangıç: Paylaşılan Veri Sorunu (Neden Lock Var?)

Bir bilgisayarda birden fazla işlem aynı anda çalışabilir.
Bu işlemler bazen aynı veriye erişmek ister: bir sayaç, bir buffer veya bir global değişken.

Bu durum günlük hayattan şu örneğe benzer:

"Iki kişi aynı kaleme aynı anda uzanıp yazmaya çalışırsa defter karalanır."

Bilgisayarda da aynı şey olur. Örneğin iki işlem aynı anda:

```c
counter = counter + 1;
```

satırını çalıştırmak isterse sonuç bozulabilir. Çünkü bu işlem aslında birkaç adıma bölünerek yapılır ve bu adımların karışması **race condition** denilen hataya yol açar.

Bu nedenle paylaşılan veriyi korumak için **lock** (kilit) mekanizmasına ihtiyaç vardır.

---

## 2. Critical Region (Kritik Bölge) – Korunması Gereken Bölge

Bir kod parçası, aynı anda iki işlem tarafından yürütüldüğünde bozuluyorsa buna **kritik bölge (critical region)** denir.

Yukarıdaki örnekte olduğu gibi, bazı işlemler tek satır görünse de aslında birden fazla adıma ayrılır ve bu adımların sırası bozulduğunda veri tutarsız hâle gelir.

Bu nedenle:

- Paylaşılan veriyi okuyan,
- Paylaşılan veriyi değiştiren,
- Paylaşılan veriyi geri yazan

kod parçaları **kritik bölgedir**.

Kritik bölge = *"aynı anda erişildiğinde bozulma riski taşıyan kod"*.

---

## 3. Producer–Consumer (Üretici–Tüketici) Problemi

Kritik bölge kavramının en bilinen örneklerinden biri **Producer–Consumer** problemidir.

- **Producer**: Veri üretir → buffer’a yazar  
- **Consumer**: Veri tüketir → buffer’dan okur  

İki taraf da aynı paylaşılan değişkenleri (`in`, `out`) değiştirir.

Eğer kilit kullanılmazsa:

- İki işlem aynı yere veri yazabilir
- Consumer boş buffer’dan okur
- Producer dolu buffer’ın üstüne yazar

Bu nedenle bu yapı **mutlaka bir lock ile korunmalıdır**.

---

## 4. Peki Lock Nasıl Çalışır?

Lock (kilit) fikri çok basittir:

> “Bu bölgeyi aynı anda sadece bir işlem kullansın; diğerleri beklesin.”

Buradaki kritik soru:

- Bekleyen işlem uyuyarak mı bekleyecek?
- Yoksa CPU üzerinde aktif şekilde dönerek mi bekleyecek?

Cevap bizi spinlock’a götürür.

---

## 5. Atomic Operation – Spinlock’un Temeli

Normal bir işlem CPU düzeyinde şu adımlardan oluşur:

```
read → modify → write
```

Bu adımlar **bölünebilir**, araya başka işlemler girebilir. Asıl sorun buradadır.

Bazı özel CPU talimatları ise atomiktir:

- `test_and_set`
- `compare_and_swap`
- `xchg`

Bu komutlar:

- Tek adımda çalışır  
- Bölünemez  
- Kesilemez  
- Yarışma koşulunu engeller  

Spinlock, işte bu atomik talimatlar üzerine kuruludur.

**Spinlock = Donanım destekli atomik kilit**

---

## 6. Spinlock Nedir? (Basit Tanım)

Spinlock, bir kilidi alamayan işlemin **uyumak yerine** CPU üzerinde **aktif olarak dönerek** (busy-wait) beklediği bir lock türüdür.

Örnek bekleme davranışı:

```c
while (!lock_available) {
    // bekle ama uyuma, CPU'yu bırakma
}
```

Spinlock özellikleri:

- Uyku yok  
- Context switch yok  
- Busy-wait var  
- Çok kısa kritik bölgelerde kullanılmak zorundadır  

---

## 7. Spinlock Nerede Kullanılır?

Spinlock yalnızca **sleep etmenin yasak olduğu** yerlerde kullanılabilir:

### • Interrupt context
IRQ handler asla uyuyamaz → mutex kullanılamaz → spinlock gerekir.

### • Preemption'ın kapatılması gereken yerler
CPU context değiştirmesin diye spinlock kullanılır.

### • Multi-core paylaşımlı veri
CPU0 lock’u alır → CPU1 spin eder → veri güvenliği sağlanır.

**Önemli:**  
Spinlock **çok kısa** kritik bölgeler içindir.

---

## 8. Spinlock Nasıl Çalışır?

Senaryo:

1. CPU0 lock’u aldı.  
2. CPU1 lock’u almak istiyor.  
3. CPU1 sürekli lock’un boşalmasını kontrol eder:

```
Boş mu?
Boş mu?
Boş mu?
```

CPU0 lock’u bırakınca CPU1 lock’u anında alır.

Bu davranış SMP (çok çekirdekli) sistemlerde anlamlıdır.

---

## 9. Spinlock CPU Tarafından Tutulur – Mutex TASK Tarafından

| Spinlock | Mutex |
|----------|--------|
| CPU kilidi tutar | Task kilidi tutar |
| Sleep yasaktır | Sleep serbesttir |
| Preemption kapalıdır | Preemption açık olabilir |
| IRQ içinde kullanılabilir | Kullanılamaz |
| Çok kısa işler | Uzun işler |

Özet:

- **Mutex:** Uyutarak bekletir.  
- **Spinlock:** Uyutmadan döndürerek bekletir.

---

## 10. En Büyük Tehlike: IRQ Gelirse Deadlock

Senaryo:

1. Task `spin_lock()` aldı  
2. Interrupt geldi  
3. IRQ handler aynı lock’u almaya çalıştı  
4. Lock task’tadır → IRQ sonsuza kadar spin eder  
5. IRQ sürekli çalıştığı için task’a geri dönülemez  
6. Task `spin_unlock()` çağıramaz  
7. Sistem tamamen kitlenir

Özet:

```
Task lock aldı → IRQ geldi → IRQ lock ister → alamaz
→ CPU task'a dönemez → task lock'u bırakamaz → sistem durur
```

---

# 10.1. Yanlış Kullanım Örneği

### Task context:

```c
void task_function(void)
{
    spin_lock(&lock);      // Preemption kapalı, IRQ açık
    counter++;             // Ortak veri
    spin_unlock(&lock);
}
```

### IRQ handler:

```c
irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    spin_lock(&lock);      // Aynı lock'u almak istiyor → tehlikeli
    counter++;
    spin_unlock(&lock);
    return IRQ_HANDLED;
}
```

### Deadlock’a Giden Süreç

1. Task lock'u aldı  
2. IRQ geldi → handler çalıştı  
3. IRQ lock’u almaya çalıştı → alamadı  
4. Sonsuz spin  
5. IRQ bitmez → task çalışamaz  
6. Task lock’u bırakamaz → **deadlock**

---

# 10.2. Doğru Kullanım: `spin_lock_irqsave()`

Task context mutlaka interrupt’ları kapatarak lock almalıdır.

### Task context:

```c
void task_function(void)
{
    unsigned long flags;

    spin_lock_irqsave(&lock, flags);   // IRQ'ları kapat + preemption kapat
    counter++;
    spin_unlock_irqrestore(&lock, flags);
}
```

### IRQ handler:

```c
irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    spin_lock(&lock);      // IRQ zaten kapalı olduğu için güvenli
    counter++;
    spin_unlock(&lock);
    return IRQ_HANDLED;
}
```

### Neden Doğru?

- IRQ’lar kapalı → IRQ handler lock yarışı yapamaz  
- Task lock’u güvenle kullanır  
- Lock bırakıldığında IRQ eski hâline döner  
- Deadlock tamamen engellenir  

**Kritik kural:**  
Aynı lock hem task hem IRQ tarafından kullanılıyorsa:

❌ `spin_lock()` → ASLA kullanılmaz  
✔ `spin_lock_irqsave()` → HER ZAMAN kullanılır  

---

## 13. Spinlock Altında Yasak Olan İşlemler

- `sleep()`, `msleep()`, `schedule()`  
- `mutex_lock()`  
- `semaphore` işlemleri  
- `waitqueue`  
- `copy_to_user()` / `copy_from_user()`  
- `printk()` (bazı durumlarda)

Sebep: Spinlock altında preemption kapalıdır → sleep edilirse sistem donar.

---

## 14. Spinlock Neden Çok Kısa Olmalı?

Spinlock tutulurken:

- CPU boşa döner  
- IRQ’lar kapalı olabilir  
- Preemption kapalıdır  

Bu nedenle kritik bölge:

> **Birkaç CPU talimatından uzun olmamalıdır.**

---

## 15. Spinlock’un Gerçek Değeri: Multi-core Sistemler

Tek çekirdekli sistemlerde:

- Paralel yarışma yoktur  
- Spinlock’un faydası sınırlıdır  

Ancak çok çekirdekte:

- CPU0 + CPU1 aynı veri üzerinde yarışabilir  
- Spinlock bu yarışmayı doğru şekilde engeller  

Spinlock = SMP sistemlerde temel güvenlik mekanizmasıdır.

---

