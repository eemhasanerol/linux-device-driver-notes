## 1. Başlangıç: Paylaşılan Veri Sorunu (Neden Lock Var?)

Bir bilgisayarda birden fazla işlem aynı anda çalışabilir.
Bu işlemler bazen aynı veriye erişmek ister: bir sayaç, bir buffer veya bir global değişken.

Bu durum günlük hayattan şu örneğe benzer:

"Iki kişi aynı kaleme aynı anda uzanıp yazmaya çalışırsa defter karalanır."

Bilgisayarda da aynı şey olur. Örneğin iki işlem aynı anda:

```c
counter = counter + 1;
```

satırını çalıştırmak isterse sonuç bozulabilir. Çünkü bu işlem aslında üç adımdan oluşur:

1. Değeri oku  
2. 1 ekle  
3. Yeni değeri geri yaz  

İki işlem bu adımları aynı anda yürütürse adımlar karışır ve race condition ortaya çıkar.

Bu nedenle paylaşılan veriyi korumak için lock (kilit) mekanizmasına ihtiyaç vardır.
