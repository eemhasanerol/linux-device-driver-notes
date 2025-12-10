## 1. Başlangıç: Paylaşılan Veri Sorunu (Neden Lock Var?)

Bir bilgisayarda birden fazla işlem aynı anda çalışabilir.
Bu işlemler bazen aynı veriye erişmek ister: bir sayaç, bir buffer veya bir global değişken.

Bu durum günlük hayattan şu örneğe benzer:

"Iki kişi aynı kaleme aynı anda uzanıp yazmaya çalışırsa defter karalanır."

Bilgisayarda da aynı şey olur. Örneğin iki işlem aynı anda:

```c
counter = counter + 1;
