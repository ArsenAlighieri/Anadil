# Anadil Dil Referansi

Bu dosya projenin su anki calisan davranisini ozetler. V0.3 itibariyle dil
native compiler odaklidir; `yorumla` komutu test ve gecis yardimcisi olarak
durur.

## Program Yapisi

Her program parametresiz bir `Ana()` fonksiyonu icermelidir.

```ana
Ana() {
    yazdir(10);
}
```

## Tipler

Desteklenen tipler:

```ana
sayı
mantık
metin
dizi
```

ASCII yazimlarda tip keyword alias'i yoktur; kaynak kodda `sayı` ve `mantık`
kullanilir.

## Degisken Tanimlama

```ana
x: sayı = 10;
durum: mantık = doğru;
mesaj: metin = "Merhaba";
degerler: dizi = {1, "iki", doğru};
```

Degisken taniminda tip zorunludur.

## Atama

```ana
x = 30;
durum = yanlış;
mesaj = "Yeni";
```

Normal atamada atanan deger degiskenin tipiyle ayni olmalidir.

Dizi elemani atamasi ayri bir kuraldir:

```ana
degerler[0] = "bir";
degerler[1] = 2;
```

Diziler heterojen oldugu icin dizi elemanina farkli tipte deger atanabilir.

## Sabit Degerler

```ana
10
-10
doğru
yanlış
"Merhaba"
```

Unary eksi sayilar icin gecerlidir:

```ana
x: sayı = -10;
yazdir(-x);
yazdir(10 + -3);
```

## Yorum Satirlari

`//` satir sonuna kadar yorum kabul edilir.

```ana
// Bu satir calistirilmaz.
yazdir(10);
```

## Aritmetik Operatorler

```ana
+
-
*
/
```

Ornek:

```ana
sonuc: sayı = (10 + 20) * 2;
```

## Karsilastirma Operatorleri

```ana
==
!=
<
>
<=
>=
```

Karsilastirmalar `mantık` degeri uretir.

```ana
yazdir(10 > 5);
```

## Kosul

Kosul parantez icinde yazilir.

```ana
eğer (x > 10) {
    yazdir(x);
} değilse {
    yazdir(0);
}
```

## Donguler

Sonsuz dongu:

```ana
döngü {
    yazdir(1);
}
```

Kosullu dongu:

```ana
döngü (x < 10) {
    x = x + 1;
}
```

Sayacli dongu:

```ana
döngü (i: sayı = 0; i < 10; i = i + 1) {
    yazdir(i);
}
```

Dongu kontrol ifadeleri:

```ana
kır;
devam;
```

`loop` keyword alias'i yoktur. Dilin keyword yuzeyi Turkce tutulur.

## Fonksiyonlar

Fonksiyon tanimlamak icin ayri bir anahtar kelime yoktur.

```ana
Topla(a: sayı, b: sayı) -> sayı {
    dön a + b;
}
```

Donus tipi olmayan fonksiyon:

```ana
YazdirDeger(x: sayı) {
    yazdir(x);
}
```

Donus tipi belirtilirse tum kontrol yollari deger dondurmelidir.

## Diziler

V0.3 ile `dizi` tipi eklenmistir.

```ana
degerler: dizi = {1, "iki", doğru};
bos: dizi = {};
```

Diziler dynamic ve heterojendir:

- Ayni dizide `sayı`, `mantık`, `metin` ve `dizi` degerleri bulunabilir.
- Dizi elemanlari degistirilebilir.
- Diziler referans tiptir; assignment ayni dizi nesnesini paylasir.

```ana
Ana() {
    a: dizi = {1, "iki"};
    b: dizi = a;

    b[0] = "bir";
    yazdir(a[0]); // bir
}
```

Index okuma:

```ana
yazdir(degerler[0]);
```

Index atama:

```ana
degerler[0] = "bir";
```

Dizi uzunlugu:

```ana
yazdir(uzunluk(degerler));
```

Runtime hata kosullari:

- Negatif index hata verir.
- Aralik disi index hata verir.

V0.3 sinirlari:

- `yazdir(dizi)` desteklenmez.
- `yazdir(dizi[i])` desteklenir.
- `dizi[i] + 1` gibi dynamic `deger` aritmetigi desteklenmez.
- Push/pop veya append API'si yoktur.

## Yerlesik Fonksiyonlar

Su anda iki temel yerlesik fonksiyon vardir:

```ana
yazdır(deger);
yazdir(deger);
uzunluk(metin_veya_dizi);
```

`yazdır` deger dondurmez. Geriye uyumluluk icin `yazdir` ASCII alias'i da
kabul edilir. Bu yuzden su gecersizdir:

```ana
x: sayı = yazdir(10);
```

`uzunluk(metin)` metnin byte uzunlugunu `sayı` olarak dondurur.

```ana
x: sayı = uzunluk("Merhaba");
```

`uzunluk(dizi)` dizi eleman sayisini `sayı` olarak dondurur.

```ana
degerler: dizi = {1, 2, 3};
yazdir(uzunluk(degerler));
```

## Hata Kontrolleri

Semantic analiz su durumlari yakalar:

- `Ana()` eksikligi.
- `Ana()` fonksiyonunun parametre almasi.
- `Ana()` fonksiyonunun donus tipi belirtmesi.
- Tip uyumsuzlugu.
- Tanimlanmamis degisken kullanimi.
- Tanimlanmamis fonksiyon cagrisi.
- Yanlis arguman sayisi veya tipi.
- `kır` / `devam` ifadelerinin dongu disinda kullanilmasi.
- `yazdır` sonucunun deger gibi kullanilmasi.
- Dizi index tipinin `sayı` olmamasi.
- Dizi olmayan deger uzerinde index okuma/atama.

Runtime su durumlari yakalar:

- Sifira bolme.
- Dizi index'inin negatif olmasi.
- Dizi index'inin aralik disinda olmasi.

## Native Derleme

Anadil, Windows x64 icin native compiler icerir.

Assembly uretmek:

```powershell
cargo run -- asm examples\topla.ana
```

Assembly dosyasini yazmak:

```powershell
cargo run -- asm-yaz examples\topla.ana
```

Executable derlemek:

```powershell
cargo run -- derle examples\topla.ana
examples\topla.exe
```

Derleyip calistirmak:

```powershell
cargo run -- calistir examples\dizi_v03.ana
```

Native derleme Visual Studio Build Tools C++ araclarini kullanir. Su anki MVP
Windows x64 disinda hedef uretmez.

Daha teknik ayrinti icin: [native_compiler.md](native_compiler.md)

