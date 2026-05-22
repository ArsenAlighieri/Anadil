# Anadil Hoca Gorusme Notlari

Bu dosya, Anadil projesini hocaya anlatmak icin kisa ama teknik olarak dolu bir konusma notudur.

## 1. Kisa Tanitim

Anadil, Turkce sozdizimine sahip kucuk bir programlama dilidir. Projenin hedefi sadece Turkce anahtar kelimeleri olan yorumlanan bir oyuncak dil yapmak degil; kaynak koddan dogrudan Windows native `.exe` uretebilen bir compiler prototipi gelistirmektir.

Ornek program:

```ana
Ana() {
    degerler: dizi = {1, "iki", 3};

    yazdir(uzunluk(degerler));
    yazdir(degerler[0]);

    degerler[0] = "bir";
    yazdir(degerler[0]);
}
```

Bu ornek artik sadece interpreter'da degil, native compiler hattinda da calismaktadir.

## 2. Projenin Amaci

Anadil'in ana amaci, Turkce okunabilirlik ile gercek compiler mimarisini birlestiren bir dil prototipi ortaya koymaktir.

Proje su alanlari gostermeyi hedefler:

- Lexer ve parser tasarimi
- AST ve typed AST uretimi
- Semantic analysis
- Basit IR tasarimi
- Native code generation
- Runtime tasarimi
- Heap nesneleri ve refcount cleanup
- IDE entegrasyonu
- Packaging ve release sureci

Kisa cumleyle:

> Anadil, Turkce sozdizimli, kendi runtime'i ve native compiler backend'i olan erken asama bir programlama dili prototipidir.

## 3. Dil Ozellikleri

Su anda dilde bulunan temel ozellikler:

- Guncel giris noktasi: `Ana()`
- Degisken tanimi:

```ana
x: sayı = 10;
```

- Tipler:

```text
sayı
mantık
metin
dizi
```

- Fonksiyon tanimlama:

```ana
Topla(a: sayı, b: sayı) -> sayı {
    dön a + b;
}
```

- Kosul:

```ana
eğer (x > 0) {
    yazdir(x);
} değilse {
    yazdir(0);
}
```

- Dongu:

```ana
döngü (i: sayı = 0; i < 3; i = i + 1) {
    yazdir(i);
}
```

- Dongu kontrolu:

```ana
kır;
devam;
```

- Metin islemleri:

```ana
mesaj: metin = "Merhaba" + " Anadil";
yazdir(mesaj);
yazdir(uzunluk(mesaj));
yazdir(mesaj == "Merhaba Anadil");
```

- Dizi islemleri:

```ana
degerler: dizi = {1, "iki", 3};
yazdir(uzunluk(degerler));
yazdir(degerler[0]);
degerler[0] = "bir";
yazdir(degerler[0]);
```

## 4. Compiler Mimarisi

Anadil'in genel compile pipeline'i:

```text
.ana kaynak kod
-> lexer
-> parser / AST
-> semantic analyzer
-> typed AST
-> IR
-> Windows x64 assembly
-> MASM / linker
-> .exe
```

Bu akista interpreter ana hedef degildir. Interpreter daha cok referans davranis ve test altyapisi icin tutulur.

## 5. Frontend

Compiler frontend su parcalardan olusur:

- Lexer
- Parser
- AST
- Semantic analyzer
- Typed AST

Lexer kaynak kodu token'lara ayirir.

Parser token'lardan AST uretir.

Semantic analyzer su kontrolleri yapar:

- `Ana()` giris noktasi var mi?
- Tip uyumsuzlugu var mi?
- Fonksiyon cagrisinda arguman sayisi dogru mu?
- Fonksiyon donus tipi dogru mu?
- `kır` ve `devam` dongu disinda kullanilmis mi?
- Ulasilamayan kod var mi?
- `uzunluk` dogru tipe uygulanmis mi?
- Dizi index'i sayi mi?

Semantic analyzer'dan sonra typed AST olusur. Backend artik tipleri bilerek kod uretebilir.

## 6. IR

IR, typed AST ile native assembly arasinda daha sade bir ara temsil olarak kullanilir.

IR sayesinde:

- Fonksiyonlar
- Local degiskenler
- Atamalar
- If/loop yapilari
- Runtime cagri mantigi

daha acik sekilde izlenebilir.

IR su anda buyuk bir optimizer altyapisi degil, ama ileride optimizer icin uygun bir basamak olarak duruyor.

## 7. Backend

Backend, typed AST/IR bilgisini alip Windows x64 assembly ureten kisimdir.

Backend akisi:

```text
Typed AST / IR
-> generated .asm
-> ml64.exe ile .obj
-> link.exe ile .exe
```

Kullanilan toolchain:

```text
ml64.exe   assembly -> object file
link.exe   object + runtime lib -> exe
lib.exe    runtime object -> runtime lib
```

Backend'in sorumluluklari:

- Fonksiyonlari assembly procedure olarak emit etmek
- Stack frame kurmak
- Local degiskenleri stack'e yerlestirmek
- Windows x64 calling convention'a uymak
- Aritmetik islemler icin assembly uretmek
- Karsilastirmalar icin assembly uretmek
- If/loop/break/continue/return icin label ve jump uretmek
- Fonksiyon cagrisini emit etmek
- Runtime helper cagrisini emit etmek
- Heap objeleri icin retain/release cleanup cagri noktalarini koymak

Ornek fonksiyon prologue/epilogue mantigi:

```asm
push rbp
mov rbp, rsp
sub rsp, frame_size

; fonksiyon govdesi

add rsp, frame_size
pop rbp
ret
```

Local degiskenler `rbp` tabanli offsetlerle tutulur:

```asm
mov QWORD PTR [rbp-8], rax
```

## 8. Windows x64 Calling Convention

Backend Windows x64 ABI'ye gore kod uretir.

Ilk dort arguman register'larla verilir:

```text
rcx
rdx
r8
r9
```

Daha fazla arguman stack uzerinden verilir.

Ayrica Windows x64 tarafinda function call oncesi shadow space ve stack alignment kurallarina dikkat edilir. Bu onemli, cunku runtime helper'lari ve Windows API cagri mekanizmasi buna baglidir.

## 9. Runtime

Anadil'in native runtime'i assembly dosyasidir:

```text
runtime/anadil_runtime.asm
```

Runtime su servisleri saglar:

- Sayi yazdirma
- Mantik yazdirma
- Metin yazdirma
- Metin uzunlugu
- Metin birlestirme
- Metin esitlik karsilastirmasi
- Heap tahsisi
- Refcount artirma
- Refcount azaltma
- Runtime panic / hata
- Dizi olusturma
- Dizi uzunlugu
- Dizi elemani okuma
- Dizi elemani yazma
- Dinamik deger yazdirma

Backend, bu runtime fonksiyonlarini assembly icinden cagirir.

## 10. Heap ve Refcount

Metin ve dizi gibi degerler heap objesi olarak tutulur.

Genel heap nesnesi mantigi:

```text
[refcount]
[tip_id]
[data...]
```

Kullanici koduna donen pointer data baslangicini gosterir.

Runtime helper'lari:

```text
anadil_runtime_tahsis
anadil_runtime_paylas
anadil_runtime_birak
```

`paylas`, refcount artirir.

`birak`, refcount azaltir. Refcount sifira inerse nesne heap'ten temizlenir.

Dizi icinde metin veya baska dizi gibi refcount edilen eleman varsa, dizi temizlenirken bu elemanlar da birakilir.

## 11. Metin Backend'i

Metinler heap/runtime destekli calisir.

Ornek:

```ana
mesaj: metin = "Merhaba" + " Anadil";
yazdir(mesaj);
```

Backend burada:

1. String literal adreslerini hazirlar.
2. `anadil_runtime_metin_birlestir` cagrisi emit eder.
3. Donen heap metin nesnesini local degiskene koyar.
4. Scope sonunda `anadil_runtime_birak` ile cleanup yapar.

Metin uzunlugu:

```ana
uzunluk(mesaj)
```

Native tarafta:

```text
anadil_runtime_metin_uzunluk
```

helper'ina gider.

## 12. Dizi Backend'i

V0.3 ile dizi native compiler hattina girdi.

Dizi literal:

```ana
degerler: dizi = {1, "iki", 3};
```

Backend burada:

1. `anadil_runtime_dizi_olustur` cagirir.
2. Her eleman icin tag + payload hazirlar.
3. `anadil_runtime_dizi_set` ile elemanlari diziye yazar.
4. Dizi local degiskene kaydedilir.
5. Scope sonunda dizi `anadil_runtime_birak` ile temizlenir.

Dizi eleman layout'u:

```text
[tag: u64]
[payload: u64]
```

Tag degerleri:

```text
1 = sayı
2 = mantık
3 = metin
4 = dizi
```

Okuma:

```ana
yazdir(degerler[0]);
```

Backend:

```text
anadil_runtime_dizi_get
anadil_runtime_print_deger
```

cagrilarini kullanir.

Atama:

```ana
degerler[0] = "bir";
```

Backend:

```text
anadil_runtime_dizi_set
```

cagrisi uretir.

Out-of-range durumunda runtime panic:

```text
Anadil runtime hatasi: Dizi index'i aralik disinda
```

## 13. Dinamik Deger Meselesi

Diziler heterojen oldugu icin `degerler[0]` compile-time'da kesin olarak `sayı` mi `metin` mi bilinmeyebilir.

Bu yuzden dizi elemani native tarafta dinamik deger hucreleriyle temsil edilir:

```text
tag + payload
```

`yazdir(degerler[0])` runtime'da tag'e bakar:

- tag sayi ise sayi yazdirir
- tag mantik ise mantik yazdirir
- tag metin ise metin yazdirir

Bu, heterojen dizi destegini mumkun kilar.

## 14. IDE

Native IDE projenin ana kullanici yuzudur.

IDE ozellikleri:

- `.ana` dosyasi acma
- Kaydetme
- F5 / Yap akisi
- EXE derleme
- Son EXE'yi calistirma
- Output ve Build sekmeleri
- Diagnostic kartlari
- Hata kartindan editore gitme
- Native Build Tools yoksa kullanici dostu uyari
- `dizi` tip renklendirmesi

Web IDE deneysel/ikincil tutuluyor. Ana odak native IDE.

## 15. Release Durumu

V0.1:

- Native compile akisi
- Paketleme
- IDE temel akisi
- Runtime path fix
- Build Tools UX

V0.2:

- Metin runtime'i
- Metin concat
- Metin equality
- `uzunluk(metin)`
- Refcount cleanup

V0.3:

- Dizi tipi
- Dizi literal
- Dizi index okuma
- Dizi eleman atama
- `uzunluk(dizi)`
- Native runtime dizi primitive'leri
- Native IDE dizi highlight

## 16. Test Yaklasimi

Projede testler birkac katmanda tutuluyor:

- Lexer/parser/sema unit testleri
- Interpreter davranis testleri
- Native assembly emission testleri
- CLI diagnostic testleri
- Native edge case testleri
- Native examples parity testleri

En onemli testlerden biri parity testidir:

```text
Interpreter output == Native exe output
```

Yani ayni `.ana` programi interpreter'da ve native exe'de ayni sonucu vermeli.

## 17. Su Anki Sinirlar

Dil hala erken asamadadir.

Mevcut sinirlar:

- Simdilik Windows x64 odakli
- Visual Studio Build Tools gerekiyor
- LLVM kullanilmiyor
- Register allocation basit
- Optimization sinirli
- Standart kutuphane cok kucuk
- Modül sistemi yok
- Paket yoneticisi yok
- Dizi destegi yeni, edge case testleri artiriliyor
- Dynamic deger sistemi henuz sinirli yuzeyde kullaniliyor

## 18. Hocaya Kisa Anlatim

Hocaya su sekilde anlatabilirsin:

> Hocam Anadil, Turkce sozdizimli kucuk bir programlama dili. Amacim sadece interpreter yazmak degildi; kaynak koddan dogrudan Windows native executable uretebilen bir compiler prototipi gelistirmekti. Lexer, parser, semantic analyzer, typed AST, IR ve native x64 assembly backend'i var. Ayrica kendi assembly runtime dosyasi var. Metin ve dizi gibi heap objeleri runtime tarafinda refcount ile yonetiliyor. IDE uzerinden build/run yapilabiliyor. Interpreter hala var ama asil hedef native compiler; interpreter daha cok referans davranis ve test icin kullaniliyor.

## 19. Sorulabilecek Sorular ve Cevaplar

### Bu sadece Turkce Python gibi bir sey mi?

Hayir. Python gibi yorumlanan bir dil hedeflenmiyor. Anadil'in hedefi native `.exe` uretebilen compiler olmak.

### Interpreter mi compiler mi?

Compiler. Interpreter sadece referans davranis ve test altyapisi icin duruyor.

### Neden Turkce sozdizimi?

Programlama kavramlarini Turkce ifade edebilen, egitim ve okunabilirlik acisindan daha dogal bir dil denemesi yapmak icin. Ama teknik tarafta gercek compiler mimarisi kuruluyor.

### Hangi platformda calisiyor?

Su an Windows x64 hedefleniyor. Native compile icin Visual Studio Build Tools gerekiyor.

### Rust gerekli mi?

Projeyi gelistirmek icin Rust gerekiyor. Ancak paketlenmis release kullanilirsa son kullaniciya Rust gerekmemesi hedefleniyor. Kullanici `anadil.exe`, `anadil-ide.exe` ve runtime dosyalariyla calisabilir.

### Program nasil derleniyor?

Ornek:

```powershell
anadil derle examples\dizi_v03.ana
examples\dizi_v03.exe
```

IDE uzerinden de F5 veya EXE derle akisi kullanilabilir.

### Backend ne uretiyor?

Backend Windows x64 assembly uretiyor. Sonra MASM `ml64.exe` ile object file, `link.exe` ile `.exe` uretiliyor.

### LLVM kullaniyor musun?

Hayir. Su an assembly dogrudan uretiliyor. Bu hem daha basit bir MVP sagliyor hem de calling convention, stack frame ve runtime mantigini daha net gosteriyor.

### Tip sistemi var mi?

Evet. `sayı`, `mantık`, `metin`, `dizi` tipleri var. Semantic analyzer tip hatalarini compile oncesinde yakaliyor.

### Diziler homojen mi?

Hayir. V0.3 tasariminda diziler heterojen. Ayni dizi icinde sayi, metin, mantik gibi farkli degerler bulunabilir.

### Heterojen dizi native tarafta nasil calisiyor?

Her dizi elemani tag + payload olarak tutuluyor. Tag, elemanin hangi tipte oldugunu belirtir.

### `a[0]` sonucu hangi tip?

Heterojen dizi nedeniyle `a[0]` sonucu dinamik deger olarak dusunulur. Runtime tag'e bakarak yazdirma gibi islemleri yapar.

### Out-of-range index olursa ne oluyor?

Runtime error verip programi durduruyor:

```text
Anadil runtime hatasi: Dizi index'i aralik disinda
```

### Memory management nasil?

Basit refcount sistemi var. Metin ve dizi heap objeleri `paylas` ve `birak` mantigiyla yonetiliyor.

### Garbage collector var mi?

Hayir. Su an GC yok. Refcount tabanli daha basit bir runtime cleanup var.

### En guclu tarafi ne?

Interpreter seviyesinde kalmamasi. Native compiler, runtime, IDE, packaging ve test altyapisi birlikte ilerliyor.

### En zayif tarafi ne?

Dil erken asamada. Standart kutuphane, optimizasyon, modül sistemi ve cross-platform destek sinirli.

### Bundan sonra ne yapilacak?

Yakin hedefler:

- V0.3 edge case testlerini artirmak
- V0.3 RC checklist hazirlamak
- Paketleme/release testi yapmak
- Sonra optimizer, daha guclu runtime veya yeni dil ozelliklerine gecmek

### Bu proje ders acisindan ne gosteriyor?

Sadece syntax tasarimi degil; compiler pipeline, semantic analysis, native code generation, runtime, memory management, IDE entegrasyonu ve release engineering konularini gosteriyor.

## 20. Kapanis Cumlesi

> Projede amacim Turkce soz dizimi olan basit bir dil yazmaktan ziyade, uctan uca calisan bir compiler prototipi cikarmakti. Su anda dil kaynak koddan native Windows exe uretebiliyor, kendi runtime'ini kullaniyor, IDE ile calisiyor ve testlerde interpreter-native parity kontroluyle dogrulaniyor.
