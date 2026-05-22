# Anadil Backend Notlari

Bu dosya Anadil'in backend tarafinda bugune kadar yapilanlari tane tane
anlatmak icin hazirlandi. Amac, "bu dil sadece yorumlaniyor mu, gercekten
derleniyor mu, exe nasil uretiliyor, runtime ne ise yariyor?" gibi sorulara
net cevap verebilmek.

## Kisa Ozet

Anadil'in backend'i artik interpreter'a bagimli bir oyuncak katman degil.
Kaynak kodu analiz edildikten sonra Windows x64 assembly ureten ve bu
assembly'yi MASM/linker ile exe'ye ceviren bir native compiler yolu var.

Genel akis sudur:

```text
.ana kaynak kodu
  -> lexer
  -> parser / AST
  -> semantic analyzer / typed AST
  -> IR
  -> native backend
  -> Windows x64 MASM assembly
  -> ml64.exe ile .obj
  -> link.exe ile .exe
  -> runtime/anadil_runtime.lib ile calisan program
```

Bu zincirde backend dedigimiz kisim ozellikle su dosyalarda yasiyor:

- `src/ir.rs`
- `src/native.rs`
- `runtime/anadil_runtime.asm`
- `src/main.rs` icindeki native build entegrasyonu
- Native testler: `tests/native_examples.rs`, `tests/native_edge_cases.rs`
- Assembly/codegen unit testleri: `src/native.rs` icindeki testler

Interpreter hala projede duruyor; ama v0.1'den itibaren hedef interpreter'a
yaslanan bir dil olmak degil, dogrudan exe uretebilen bir compiler olmakti.
Interpreter daha cok karsilastirma, hizli test ve gecis donemi yardimcisi gibi
kullaniliyor.

## Backend Ne Demek?

Bir compiler genelde iki buyuk bolume ayrilir:

1. Frontend
2. Backend

Frontend kaynak kodu anlar. Yani:

- Karakterleri token'lara ayirir.
- Token'lardan AST kurar.
- Tipleri kontrol eder.
- Fonksiyonlari, degiskenleri, scope'lari ve hatalari cozer.

Backend ise anlasilmis programi calisacak seye cevirir. Anadil'de backend'in
gorevi su:

- Typed AST'den daha sade bir ara temsil uretmek.
- Programi Windows x64 calling convention'a uygun assembly'ye cevirmek.
- Gerekli runtime fonksiyonlarini assembly'den cagirmak.
- Assembly dosyasini objeye, objeyi exe'ye baglayacak build akisini yonetmek.
- Heap, metin, dizi, yazdirma ve runtime hata isleri icin runtime kutuphanesine
  dayanmak.

Kisaca:

```text
Frontend: "Program ne anlatmak istiyor?"
Backend : "Bu program makinede nasil calisacak?"
```

## Mevcut Native Compiler Akisi

Kullanici su komutlardan birini calistirdiginda native backend devreye girer:

```powershell
anadil calistir examples\topla.ana
anadil derle examples\topla.ana
anadil asm examples\topla.ana
anadil asm-yaz examples\topla.ana
```

Komutlarin anlamlari:

- `calistir`: Kaynagi native exe olarak derler, sonra uretilen exe'yi calistirir.
- `derle`: Sadece exe uretir.
- `asm`: Native backend'in urettigi assembly metnini stdout'a basar.
- `asm-yaz`: Assembly'yi dosyaya yazar.
- `yorumla`: Interpreter yoludur; native backend degildir.

CLI tarafinda bu akis `src/main.rs` icinde yonetilir. Native compile icin ana
fonksiyon `compile_native(path, source)` fonksiyonudur.

## `compile_native` Ne Yapiyor?

`src/main.rs` icindeki `compile_native` su adimlari yapar:

1. Build Tools var mi diye bakar.
2. Paketlenmis runtime `.lib` var mi diye kontrol eder.
3. Kaynak koddan native assembly uretir.
4. Assembly dosyasini `target/native-build/...` altina yazar.
5. `ml64.exe` ile assembly'den `.obj` uretir.
6. Runtime kutuphanesini hazirlar.
7. `link.exe` ile kullanici `.obj` dosyasi + runtime `.lib` dosyasini exe'ye
   baglar.
8. Uretilen exe'yi kaynak dosyanin yanina kopyalar.

Basitlestirilmis akis:

```text
emit_native_asm_source(source)
  -> program.asm

ml64 /c program.asm
  -> program.obj

link program.obj anadil_runtime.lib kernel32.lib
  -> program.exe
```

Bu noktada Rust sadece compiler'in yazildigi dil. Uretilen kullanici programi
Rust runtime'a bagli degil. Uretilen exe, Anadil runtime assembly kutuphanesi ve
Windows API cagrilariyla calisiyor.

## Build Tools Entegrasyonu

Windows native build icin Visual Studio Build Tools gerekiyor. Bunun sebebi:

- `ml64.exe`: MASM assembler. `.asm` dosyasini `.obj` dosyasina cevirir.
- `link.exe`: `.obj` dosyalarini ve `.lib` kutuphanelerini exe'ye baglar.
- `lib.exe`: Runtime assembly'den `.lib` uretmek icin kullanilir.
- `vcvars64.bat`: Bu araclari PATH'e ve ortama dogru sekilde tanitir.

Build Tools yoksa artik kullaniciya MASM bilen birine yazilmis gibi degil,
normal kullaniciya uygun mesaj veriliyor:

```text
Visual Studio Build Tools bulunamadi.
Indirme: https://visualstudio.microsoft.com/visual-cpp-build-tools/
Interpreter ile calistirmak icin: anadil yorumla <dosya>.ana
```

Bu mesaj hem text cikista hem JSON cikista kullaniliyor.

## Runtime Path Fix

V0.1 release icin kritik blocker'lardan biri runtime path meselesiydi.

Eski durumda compiler runtime dosyasini `env!("CARGO_MANIFEST_DIR")` ile
buluyordu. Bu makro compile zamaninda sabitlendigi icin, `anadil.exe` baska bir
makineye kopyalaninca runtime'i gelistiricinin bilgisayarindaki eski path'te
aramaya calisabiliyordu.

Bunu duzeltmek icin runtime path cozumleme exe-relative hale getirildi.

`runtime_asm_path()` sirasi:

1. Exe'nin yanindaki runtime klasoru:

   ```text
   <exe_dir>\runtime\anadil_runtime.asm
   ```

2. Development modu:

   ```text
   <CARGO_MANIFEST_DIR>\runtime\anadil_runtime.asm
   ```

3. Ikisi de yoksa denenmis yollari iceren acik hata mesaji.

Runtime cache icin de shipped/development ayrimi yapildi:

- Development modunda:

  ```text
  <repo>\target\native-runtime\
  ```

- Paketlenmis kullanici ortaminda:

  ```text
  %LOCALAPPDATA%\Anadil\cache\
  ```

Bunun sebebi `Program Files` gibi klasorlerin yazilabilir olmamasi. Kullaniciya
dagitilan exe runtime cache yazacaksa bunu kullanici profili altinda yapmali.

## Paketlenmis Runtime `.lib`

Release paketinde runtime assembly'nin yaninda prebuilt runtime kutuphanesi de
bulunabiliyor:

```text
runtime\anadil_runtime.asm
runtime\anadil_runtime.lib
```

`compile_native` once exe'nin yaninda paketlenmis `runtime/anadil_runtime.lib`
var mi diye bakar. Varsa runtime'i her seferinde tekrar assemble edip lib'e
cevirmeye gerek kalmaz.

Bu onemli cunku son kullanicida:

- Rust kurulu olmak zorunda degil.
- Cargo kurulu olmak zorunda degil.
- Runtime'i derlemek yerine hazir `.lib` kullanilabilir.

Ancak kullanici yeni Anadil programlarindan exe uretmek istiyorsa Windows native
toolchain tarafinda yine Build Tools gerekir.

## Path ve Turkce Karakter Problemi

Windows tarafinda MASM/link gibi araclar bazen Turkce karakterli veya bosluklu
path'lerde sorun cikarabiliyor. Ozellikle `.bat` uzerinden cagrilan toolchain
UTF-8/OEM codepage farkindan etkilenebiliyor.

Bunun icin `runtime_tool_arg` gibi yardimci fonksiyonlar yazildi.

Mantik sudur:

1. Eger mumkunse ASCII olan relative path kullan.
2. Relative path gercekten ayni dosyaya isaret ediyorsa onu kullan.
3. Gerekirse cwd'den hedefe relative path hesapla.
4. O da olmazsa absolute path'e dus.

Bu sayede `Masaüstü`, `Türkçe`, `OneDrive`, bosluklu klasor gibi durumlarda
native build'in patlama ihtimali azaltildi.

## IR Katmani

IR, "Intermediate Representation" yani ara temsil demektir.

Anadil'de IR `src/ir.rs` icindedir. IR su anda optimizer gibi cok buyuk bir
katman degil; ama compiler mimarisi icin onemli bir ayrim noktasi.

Typed AST kaynak dile cok yakindir. Mesela:

- Kullanici hangi syntax'i yazdi?
- If/loop nasil gorunuyordu?
- Degisken deklarasyonu kaynakta nasil ifade edildi?

IR ise backend'e daha uygun, sade ve duzenli bir program anlatimidir.

Anadil IR'de baslica instruction'lar:

- `Declare`: Local degisken tanimla.
- `Assign`: Local degiskene deger ata.
- `IndexAssign`: Dizi elemanina deger ata.
- `Expr`: Sonucu kullanilmayan expression calistir.
- `If`: Kosullu blok.
- `Loop`: Dongu.
- `Break`: Donguden cik.
- `Continue`: Dongunun sonraki adimina gec.
- `Return`: Fonksiyondan don.

IR expression'lari:

- `Number`
- `Bool`
- `String`
- `Array`
- `Local`
- `Index`
- `Call`
- `Unary`
- `Binary`

IR'in bize kazandirdiklari:

- Backend frontend'e daha az bagli kalir.
- Debug etmek kolaylasir.
- Ileride optimizer eklemek kolaylasir.
- Ileride baska backend eklemek mumkun olur.
- Testlerde "typed AST dogru mu, IR dogru mu, assembly dogru mu?" diye katman
  katman kontrol yapilabilir.

## Native Backend Dosyasi: `src/native.rs`

Native assembly ureten ana dosya `src/native.rs` dosyasidir.

Disari acilan ana fonksiyon:

```rust
pub fn emit_windows_x64_asm(program: &TypedProgram) -> Result<String, String>
```

Bu fonksiyon typed program alir ve MASM uyumlu Windows x64 assembly string'i
uretir.

Bu dosyada ana struct:

```rust
struct NativeEmitter<'a>
```

`NativeEmitter` assembly uretirken su bilgileri takip eder:

- Programdaki fonksiyonlar.
- String literal listesi.
- Benzersiz label index'i.
- Aktif return label'i.
- Aktif fonksiyondaki local sayisi.
- Aktif fonksiyondaki maksimum call arguman sayisi.
- Temporary stack derinligi.
- Loop stack'i.
- Scope cleanup stack'i.

Bu bilgiler olmadan dogru assembly uretmek zor olurdu. Mesela:

- `kır` nereye jump edecek?
- `devam` nereye jump edecek?
- Fonksiyon bitince hangi metin/dizi local'leri temizlenecek?
- Runtime call oncesi stack alignment bozuldu mu?
- Return degeri cleanup sirasinda korunacak mi?

## Assembly Program Iskeleti

Native backend assembly uretirken once runtime fonksiyonlarini `extrn` olarak
ilan eder:

```asm
extrn anadil_runtime_print_sayi:proc
extrn anadil_runtime_print_metin_nesne:proc
extrn anadil_runtime_print_mantik:proc
extrn anadil_runtime_metin_esit:proc
extrn anadil_runtime_metin_birlestir:proc
extrn anadil_runtime_metin_uzunluk:proc
extrn anadil_runtime_dizi_olustur:proc
extrn anadil_runtime_dizi_set:proc
extrn anadil_runtime_dizi_get:proc
extrn anadil_runtime_dizi_uzunluk:proc
extrn anadil_runtime_print_deger:proc
extrn anadil_runtime_paylas:proc
extrn anadil_runtime_birak:proc
extrn anadil_runtime_wait_before_exit:proc
extrn anadil_runtime_panic:proc
```

Sonra `main PROC` uretir:

```asm
main PROC
    sub rsp, 40
    call anadil_fn_Ana
    call anadil_runtime_wait_before_exit
    xor eax, eax
    add rsp, 40
    ret
main ENDP
```

Burada kullanicinin `Ana()` fonksiyonu assembly tarafinda
`anadil_fn_Ana` olarak cagirilir. Yani Anadil programinin giris noktasi
`Ana()` fonksiyonudur, Windows exe tarafindaki giris noktasi ise `main PROC`tur.

## Fonksiyon Codegen'i

Her Anadil fonksiyonu assembly'de su isimle uretir:

```text
anadil_fn_<FonksiyonAdi>
```

Ornek:

```anadil
Topla(a: sayi, b: sayi) -> sayi {
    dön a + b;
}
```

Assembly tarafinda:

```text
anadil_fn_Topla PROC
```

Fonksiyon prologue:

```asm
push rbp
mov rbp, rsp
sub rsp, <frame_size>
```

Fonksiyon epilogue:

```asm
add rsp, <frame_size>
pop rbp
ret
```

Local degiskenler `rbp` tabanli stack slot'lara yerlestirilir:

```text
local_id 0 -> [rbp-8]
local_id 1 -> [rbp-16]
local_id 2 -> [rbp-24]
```

Parametreler Windows x64 calling convention'a gore alinir:

- 1. arguman: `rcx`
- 2. arguman: `rdx`
- 3. arguman: `r8`
- 4. arguman: `r9`
- 5+ arguman: stack uzerinden

Fonksiyon girisinde bu parametreler local slot'lara kopyalanir. Boylece
fonksiyon govdesi parametreleri de normal local degisken gibi kullanabilir.

## Stack Frame ve Alignment

Windows x64 calling convention su iki kurali bekler:

1. Caller, callee icin 32 byte shadow space ayirmali.
2. Call aninda stack alignment dogru olmali.

Native backend her runtime veya kullanici fonksiyonu cagrisi oncesi
`emit_reserve_call_area` ile gerekli call alanini ayirir:

```text
32 byte shadow space
+ stack argumanlari
+ gerekirse alignment padding
```

Cagri bitince `emit_release_call_area` ile alan geri verilir.

Bu ayrinti onemlidir; cunku stack alignment bozulursa program bazen calisir
gibi gorunup baska makinede veya baska runtime cagrilarinda patlayabilir.

## If Codegen'i

Anadil:

```anadil
eğer (kosul) {
    ...
} değilse {
    ...
}
```

Backend mantigi:

1. Kosulu hesapla, sonuc `rax` icinde olsun.
2. `rax == 0` ise else label'ina atla.
3. Then blokunu emit et.
4. End label'ina atla.
5. Else blokunu emit et.
6. End label'ini koy.

Basit assembly sekli:

```asm
cmp rax, 0
je L_else
; then body
jmp L_endif
L_else:
; else body
L_endif:
```

## Loop, `kır` ve `devam`

Donguler icin backend label uretir:

- loop start label
- continue label
- break label

`kır` gorunce break label'ina jump eder.

`devam` gorunce continue label'ina jump eder.

Ama kritik detay sudur: `kır` veya `devam` bir scope'un ortasindan cikabilir.
O scope icinde heap nesnesi tutan local'ler varsa temizlenmeleri gerekir.

Bunun icin backend `scope_cleanup_stack` tutar.

Yani:

```anadil
döngü (...) {
    mesaj: metin = "Merhaba" + "!";
    eğer (...) {
        devam;
    }
}
```

Burada `devam` direkt label'a atlamadan once `mesaj` icin
`anadil_runtime_birak` emit edilir. Boylece heap memory leak olmasi engellenir.

## Return Codegen'i

Return ifadesi de cleanup acisindan hassastir.

Ornek:

```anadil
Uret() -> metin {
    sonuc: metin = "Merhaba" + "!";
    dön sonuc;
}
```

Sorun:

- `sonuc` return edilecek.
- Ama fonksiyondan cikarken local cleanup calisacak.
- Cleanup return degerini serbest birakmamali.

Bunun icin backend:

1. Return expression'i hesaplar.
2. Gerekirse refcount arttirir.
3. Return degerini gecici olarak korur.
4. Aktif scope cleanup'larini calistirir.
5. Return degerini tekrar `rax`'a alir.
6. Ortak return label'ina jump eder.

Bu sayede fonksiyon donerken heap nesnesi kaybolmaz.

## Sayilar ve Mantik Degerleri

`sayi` ve `mantik` su anda value type gibi ele alinir.

- `sayi`: 64-bit integer gibi `rax` icinde tasinir.
- `mantik`: `0` veya `1` olarak tasinir.

Arithmetic islemler:

- `+`: `add`
- `-`: `sub`
- `*`: `imul`
- `/`: `idiv`

Karsilastirmalar:

- `==`: `sete`
- `!=`: `setne`
- `<`: `setl`
- `>`: `setg`
- `<=`: `setle`
- `>=`: `setge`

Sonuc `0` veya `1` olarak `rax`'a yazilir.

## Sifira Bolme Runtime Hatasi

Bolme isleminde backend once bolen degeri kontrol eder:

```asm
cmp r10, 0
jne L_div_ok
lea rcx, error_div_zero
call anadil_runtime_panic
L_div_ok:
cqo
idiv r10
```

Yani sifira bolme CPU exception'ina birakilmiyor; Anadil runtime hatasi olarak
raporlanip program durduruluyor.

## Metin Backend'i

Metinler v0.2 tarafinda ciddi sekilde backend'e tasindi.

Eski basit C-string tarzi yerine length-prefixed heap/string object modeli
kuruldu.

Metin nesnesi data pointer olarak kullanici tarafina doner. Gercek allocation
layout'u:

```text
[refcount: u64][tip_id: u64][len: u64][bytes...]
                   ^
                   kullaniciya donen pointer burada baslar
```

Yani `rcx` bir metin nesnesine isaret ettiginde runtime tarafinda:

```text
[rcx]     -> uzunluk
[rcx + 8] -> byte baslangici
[rcx - 8] -> tip id
[rcx -16] -> refcount
```

Bu modelin faydalari:

- Metin uzunlugu O(1) okunur.
- Null terminator zorunlu degildir.
- Turkce/UTF-8 byte'lari dogrudan saklanabilir.
- Heap nesneleri icin refcount modeli kurulabilir.
- Dizi gibi baska heap tipleriyle ortak runtime modeli paylasilabilir.

## Static String Literal'lar

Kaynak koddaki string literal'lar assembly `.data` bolumune yazilir.

Ornek:

```anadil
yazdir("Merhaba");
```

Backend `.data` icinde yaklasik su layout'u uretir:

```asm
str_0_refcount dq 4000000000000000h
str_0_tip dq 1
str_0 dq 7
str_0_bytes db "Merhaba"
```

`4000000000000000h` sentinel degeri static refcount gibi kullanilir.

Runtime `anadil_runtime_birak` veya `anadil_runtime_paylas` cagirildiginda bu
degeri gorurse nesneyi heap allocation sanip free etmeye calismaz.

## Metin Yazdirma

Metin yazdirirken backend `anadil_runtime_print_metin_nesne` cagirir.

Runtime:

1. `[rcx]` ile uzunlugu okur.
2. `rcx + 8` ile byte baslangicina gider.
3. `WriteFile` ile stdout'a yazar.
4. Yeni satir basar.

Bu noktada C runtime `printf` kullanilmiyor. Runtime dogrudan Windows API
`WriteFile` cagiriyor.

## Metin Birlestirme

Anadil:

```anadil
mesaj: metin = "Merhaba" + " " + "Anadil";
```

Backend metin `+` gordugunde normal integer `add` emit etmez.
`anadil_runtime_metin_birlestir` cagirir.

Runtime:

1. Sol metnin uzunlugunu okur.
2. Sag metnin uzunlugunu okur.
3. Toplam uzunluk kadar yeni heap metin tahsis eder.
4. Sol bytes'i kopyalar.
5. Sag bytes'i kopyalar.
6. Yeni metin pointer'ini `rax` ile dondurur.

Bu islem yeni bir owned temporary metin uretir. Backend bu temporary'nin ne
zaman temizlenecegini takip eder.

## Metin Esitligi

Metinlerde `==` ve `!=` pointer karsilastirmasi degildir.

Backend:

```text
metin == metin
```

gordugunde `anadil_runtime_metin_esit` cagirir.

Runtime:

1. Uzunluklari karsilastirir.
2. Uzunluk farkliysa false.
3. Uzunluk ayniysa byte byte karsilastirir.
4. Sonuc `1` veya `0` olarak doner.

## `uzunluk` Builtin'i

`uzunluk` iki tip icin backend'de desteklenir:

- `metin`
- `dizi`

Metin icin:

```text
anadil_runtime_metin_uzunluk
```

Dizi icin:

```text
anadil_runtime_dizi_uzunluk
```

Eger backend'e `uzunluk(sayi)` gibi bir sey gelirse zaten semantik analizde
hata yakalanmasi beklenir. Backend tarafinda da koruyucu hata mesaji vardir.

## Heap ve Refcount Modeli

Anadil runtime heap nesneleri icin ortak bir allocation modeli kullaniyor:

```text
[refcount: u64][tip_id: u64][data...]
```

Runtime helper:

```text
anadil_runtime_tahsis(rcx=data_size, rdx=tip_id) -> rax=data_ptr
```

Bu fonksiyon:

1. Windows `GetProcessHeap` cagirir.
2. `HeapAlloc` ile `data_size + 16` byte ayirir.
3. Refcount'u `1` yapar.
4. Tip id yazar.
5. Kullaniciya data bolumunun pointer'ini dondurur.

Refcount helper'lari:

```text
anadil_runtime_paylas(rcx=ptr)
anadil_runtime_birak(rcx=ptr)
```

`paylas` refcount'u arttirir.

`birak` refcount'u azaltir. Refcount sifira inerse heap allocation free edilir.

Static literal'larda refcount sentinel oldugu icin `paylas` ve `birak` bunlari
degistirmez.

## Ownership Analizi

Backend, her referans tipli expression'in ownership durumunu kabaca siniflar:

- `NotRef`
- `StaticLiteral`
- `SharedReference`
- `OwnedTemporary`

Metin icin:

- `"Merhaba"` -> static literal
- `mesaj` -> shared reference
- `"A" + "B"` -> owned temporary
- `Uret()` -> owned temporary

Dizi icin:

- `{1, 2, 3}` -> owned temporary
- `degerler` -> shared reference
- dizi donen fonksiyon cagrisi -> owned temporary

Bu ayrim sayesinde backend su kararleri verebilir:

- Atama yaparken eski deger free edilmeli mi?
- Yeni deger shared ise refcount arttirilmali mi?
- Temporary kullanildiktan sonra temizlenmeli mi?
- Return degeri cleanup'tan korunmali mi?

## Scope Cleanup

`metin` ve `dizi` referans tipleridir. Bu tipteki local degiskenler scope
sonunda temizlenmelidir.

Backend `scope_cleanup_stack` tutar. Bir scope'a girince yeni cleanup listesi
acar. O scope icinde referans tipli local declare edilirse listeye ekler.
Scope bitince listeyi ters sirayla temizler.

Temizlik:

```asm
mov rcx, QWORD PTR [rbp-<local_offset>]
call anadil_runtime_birak
```

Ters sirayla temizlik, ic ice kaynak kullaniminda daha dogru bir davranistir.

## Assignment Cleanup

Referans tipli degiskenlere yeni deger atanirken eski deger unutulamaz.

Ornek:

```anadil
mesaj: metin = "Eski";
mesaj = "Yeni" + " Deger";
```

Backend:

1. Yeni degeri hesaplar.
2. Gerekirse yeni degeri korur.
3. Eski local degeri `anadil_runtime_birak` ile birakir.
4. Yeni pointer'i local slot'a yazar.

Bu olmazsa eski heap nesnesi leak olurdu.

## Fonksiyon Argumanlari ve Refcount

Fonksiyon cagrilarinda referans tipli argumanlar icin iki durum var:

1. Shared local arguman:

   ```anadil
   mesaj: metin = "Merhaba" + "!";
   Selamla(mesaj);
   ```

   Burada `mesaj` caller'da da yasamaya devam eder. Callee de parametre olarak
   temizleyecegi icin backend `anadil_runtime_paylas` cagirir.

2. Owned temporary arguman:

   ```anadil
   Selamla("Merhaba" + "!");
   ```

   Burada temporary dogrudan callee'ye transfer edilebilir. Ekstra retain
   gerekmeyebilir. Callee param cleanup'inda birakir.

Bu ayrim gereksiz refcount islemlerini azaltir ve double-free riskini onler.

## Dizi Backend'i

V0.3 tarafinda backend'e dynamic heterogeneous dizi destegi eklendi.

Kullanici syntax'i:

```anadil
degerler: dizi = {1, "iki", dogru};
yazdir(uzunluk(degerler));
yazdir(degerler[0]);
degerler[0] = "bir";
```

Kararlar:

- Dizi tipi tek basina `dizi`.
- Literal syntax `{1, 2, 3}`.
- Index okuma `a[0]`.
- Index atama `a[0] = deger`.
- Dizi dynamic boyutlu runtime nesnesi.
- Elemanlar degistirilebilir.
- Elemanlar heterojen olabilir.
- `uzunluk(a)` dizi uzunlugunu verir.
- Negatif index desteklenmez.
- Out-of-range index runtime error ile programi durdurur.
- Bos dizi icin tip annotation gerekir: `a: dizi = {}`.

## Dizi Runtime Layout'u

Dizi data layout'u:

```text
[len: u64][tag0: u64][payload0: u64][tag1: u64][payload1: u64]...
```

Heap header dahil tam allocation:

```text
[refcount: u64][tip_id=ANADIL_TIP_DIZI][len][tag0][payload0]...
```

Kullaniciya donen pointer yine data baslangicini gosterir:

```text
ptr[0] -> len
ptr[1] -> tag0
ptr[2] -> payload0
ptr[3] -> tag1
ptr[4] -> payload1
```

Her eleman 16 byte kullanir:

```text
tag:     elemanin runtime tipi
payload: sayi/mantik degeri veya heap pointer
```

Runtime tag'leri:

```text
ANADIL_DEGER_SAYI   = 1
ANADIL_DEGER_MANTIK = 2
ANADIL_DEGER_METIN  = 3
ANADIL_DEGER_DIZI   = 4
```

Bu sayede ayni dizi icinde farkli tipler durabilir.

## Dizi Literal Codegen'i

Anadil:

```anadil
degerler: dizi = {1, "iki", 3};
```

Backend:

1. Eleman sayisini `rcx`'e koyar.
2. `anadil_runtime_dizi_olustur` cagirir.
3. Donen dizi pointer'ini stack'te korur.
4. Her elemani tek tek hesaplar.
5. Eleman tipine gore tag secer.
6. `anadil_runtime_dizi_set` ile diziye yazar.
7. Dizi pointer'ini `rax` ile sonuc olarak dondurur.

Basitlestirilmis:

```asm
mov rcx, 3
call anadil_runtime_dizi_olustur
push rax

; element 0
mov rax, 1
mov rcx, QWORD PTR [rsp]
mov rdx, 0
mov r8, ANADIL_DEGER_SAYI
mov r9, rax
call anadil_runtime_dizi_set

pop rax
```

Gercek assembly'de stack alignment ve temporary cleanup ayrintilari da vardir.

## Dizi Index Okuma

Anadil:

```anadil
yazdir(degerler[0]);
```

Backend:

1. Dizi expression'ini hesaplar.
2. Index expression'ini hesaplar.
3. `anadil_runtime_dizi_get(rcx=dizi, rdx=index)` cagirir.
4. Runtime eleman cell pointer'i dondurur.
5. `yazdir` bu cell pointer'i `anadil_runtime_print_deger` ile yazdirir.

Burada `degerler[0]` sonucu statik olarak `sayi` veya `metin` degil,
runtime'da tag'li `deger` gibi davranir.

## Dizi Eleman Atama

Anadil:

```anadil
degerler[0] = "bir";
```

Backend:

1. Yeni degeri hesaplar.
2. Yeni degerin tag'ini belirler.
3. Index'i hesaplar.
4. Dizi local pointer'ini alir.
5. `anadil_runtime_dizi_set` cagirir.
6. Yeni deger owned temporary ise call sonrasi kendi temporary referansini
   birakir.

Runtime `dizi_set` sunu yapar:

1. Index aralikta mi diye bakar.
2. Eski eleman metin veya dizi ise `anadil_runtime_birak` ile eski referansi
   birakir.
3. Yeni tag/payload'u cell'e yazar.
4. Yeni eleman metin veya dizi ise `anadil_runtime_paylas` ile dizi icin
   referansi tutar.

Bu tasarim sayesinde:

- Dizi elemani degistirilebilir.
- Eski eleman leak olmaz.
- Yeni eleman dizi tarafindan sahiplenilir.
- Dizi free edilirken icindeki referans elemanlar da free edilebilir.

## Dizi Index Hatalari

Runtime index helper'i:

```text
anadil_runtime_dizi_eleman_adresi
```

Kontroller:

```asm
cmp rdx, 0
jl error
cmp rdx, [rcx]
jge error
```

Yani:

- Negatif index hata.
- `index >= len` hata.

Hata durumunda:

```text
Anadil runtime hatasi: Dizi index'i aralik disinda
```

mesaji basilir ve program `ExitProcess(1)` ile durur.

## Dizi Free Etme

Dizi bir heap nesnesidir. Refcount sifira indiginde `anadil_runtime_birak`
dizi oldugunu tip id'den anlar.

Dizi free edilmeden once icindeki elemanlar taranir:

- Eleman tag'i `METIN` ise payload icin `anadil_runtime_birak`.
- Eleman tag'i `DIZI` ise payload icin `anadil_runtime_birak`.
- `SAYI` ve `MANTIK` value oldugu icin ekstra is yok.

Sonra dizi allocation'i `HeapFree` ile serbest birakilir.

Bu, nested dizi ve metin tutan dizi icin temel memory cleanup davranisini
kurar.

## Dinamik `deger` Meselesi

Diziler heterojen oldugu icin eleman okuma sonucu tek bir statik tip olmak
zorunda degildir. Bu yuzden backend tarafinda dizi elemani runtime tag'li bir
deger gibi ele alinir.

`yazdir(degerler[0])` icin:

```text
anadil_runtime_print_deger
```

cagirilir.

Bu runtime helper:

1. Cell'den tag okur.
2. Payload okur.
3. Tag `SAYI` ise sayi yazdirir.
4. Tag `MANTIK` ise mantik yazdirir.
5. Tag `METIN` ise metin nesnesi yazdirir.
6. Desteklenmeyen tag ise runtime panic verir.

Su an dizi elemanini genel `deger` olarak her yerde kullanma destegi sinirli.
Yani `degerler[0] + 1` gibi dinamik dispatch isteyen ifadeler icin daha fazla
IR/backend calismasi gerekir. Ama `yazdir`, `uzunluk(dizi)`, index okuma ve
index atama native tarafta calisir.

## Runtime Dosyasi: `runtime/anadil_runtime.asm`

Runtime tamamen assembly dosyasidir.

Amaci compiler'in her program icine tekrar tekrar yazmak istemedigi ortak
isleri saglamaktir:

- stdout'a byte yazma
- sayi yazdirma
- mantik yazdirma
- metin yazdirma
- metin uzunlugu
- metin esitligi
- metin birlestirme
- dizi olusturma
- dizi uzunlugu
- dizi index kontrolu
- dizi set/get
- dynamic deger yazdirma
- heap allocation
- refcount arttirma
- refcount azaltma/free
- runtime panic
- program bitmeden once bekleme

Runtime C standard library kullanmaz. Bunun yerine Windows API kullanir:

- `GetStdHandle`
- `WriteFile`
- `ReadFile`
- `ExitProcess`
- `GetProcessHeap`
- `HeapAlloc`
- `HeapFree`

Bu karar, runtime'in daha kontrollu ve minimal olmasini saglar.

## Runtime Panic

Runtime hata olunca:

```text
anadil_runtime_panic(rcx=message)
```

cagirilir.

Bu fonksiyon:

1. `Anadil runtime hatasi: ` prefix'ini yazar.
2. Hata mesajini yazar.
3. Kullanici ciktiyi gorebilsin diye bekleme helper'ini cagirir.
4. `ExitProcess(1)` ile programi durdurur.

Kullandigimiz runtime hatalarina ornek:

- Sifira bolme hatasi.
- Bellek tahsisi basarisiz.
- Dizi index'i aralik disinda.
- Dizi degeri yazdirilamadi.

## IDE ile Backend Iliskisi

Native IDE backend'i dogrudan kendisi implement etmiyor. IDE, CLI compiler'i
komut olarak cagiriyor.

Akis:

```text
anadil-ide.exe
  -> anadil.exe calistir/derle --json <dosya.ana>
  -> JSON sonucunu okur
  -> Build/Output/Diagnostics panellerinde gosterir
```

Bu tasarimin avantaji:

- CLI ve IDE ayni compiler yolunu kullanir.
- IDE icin ayri compiler davranisi olusmaz.
- JSON cikis sayesinde IDE structured sonuc alir.
- Build Tools yoksa IDE butonlari disabled olabilir.

IDE tarafinda native-first davranis sudur:

- `Yap` / F5: derle ve calistir.
- `EXE Derle`: exe uret.
- `EXE Calistir`: son exe'yi tekrar calistir.
- Diagnostics kartlari editor konumuna gidebilir.

## Test Yaklasimi

Backend icin testler birden fazla seviyede var.

### 1. Assembly unit testleri

`src/native.rs` icindeki testler assembly string'inde beklenen runtime call'lari
var mi diye bakar.

Ornek:

- Sayi yazdirma `anadil_runtime_print_sayi` cagiriyor mu?
- Metin concat `anadil_runtime_metin_birlestir` cagiriyor mu?
- Dizi literal `anadil_runtime_dizi_olustur` cagiriyor mu?
- Dizi index `anadil_runtime_dizi_get` cagiriyor mu?
- Scope cleanup `anadil_runtime_birak` emit ediyor mu?

Bu testler exe calistirmadan codegen mantigini yakalar.

### 2. Native examples parity

`tests/native_examples.rs` interpreter output'u ile native output'u
karsilastirir.

Bu su anlama gelir:

```text
interpreter output == native exe output
```

Bu test backend'in program davranisini korudugunu gosterir.

### 3. Native edge cases

`tests/native_edge_cases.rs` daha kirilgan durumlara bakar:

- Path icinde bosluk/Turkce karakter.
- Runtime cache davranisi.
- Native build hata durumlari.
- Edge-case kaynaklar.

### 4. CLI diagnostics testleri

CLI JSON/text davranisi icin testler vardir. IDE de JSON cikisa dayandigi icin
bu testler dolayli olarak IDE deneyimini de korur.

## V0.1'den V0.3'e Backend Yolculugu

### V0.1

Hedef:

- Native-first release.
- `calistir`, `derle`, `yorumla` ayrimi.
- IDE'de native build/run akisi.
- Paketlenebilir exe.
- Runtime path fix.
- Build Tools hata mesaji.
- Release ZIP/installer hazirligi.

Bu fazda en kritik sey interpreter'dan urun olarak cikmamakti. Kullanici
`anadil.exe` ile `.ana` dosyasindan exe uretebilmeli.

### V0.2

Hedef:

- Heap modelini baslatmak.
- Metinleri daha ciddi runtime nesnesine cevirmek.
- Refcount altyapisini kurmak.
- Metin concat, metin esitligi, metin uzunlugu gibi isleri native runtime'a
  almak.
- Scope cleanup, return cleanup, assignment cleanup gibi memory safety
  davranislarini toparlamak.

V0.2 backend'in "sadece integer assembly emit eden MVP" olmaktan cikmaya
basladigi faz oldu.

### V0.3

Hedef:

- Dynamic heterogeneous dizi.
- Dizi literal.
- Dizi index okuma.
- Dizi eleman atama.
- Dizi uzunlugu.
- Runtime bounds check.
- Dizi icinde metin/sayi/mantik/dizi gibi tag'li eleman modeli.

Bu faz backend'i daha dinamik runtime deger modeline dogru tasiyor.

## Su Anda Backend'in Destekledikleri

Native backend seviyesinde desteklenen ana ozellikler:

- `sayi`
- `mantik`
- `metin`
- `dizi`
- Degisken tanimlama
- Degiskene atama
- Dizi elemanina atama
- Fonksiyon tanimlama
- Fonksiyon cagrisi
- Return
- If/else
- Dongu
- `kır`
- `devam`
- Aritmetik islemler
- Karsilastirma islemleri
- Sifira bolme runtime hatasi
- `yazdir`
- `uzunluk(metin)`
- `uzunluk(dizi)`
- Metin concat
- Metin esitlik/esit degil
- Dizi literal
- Dizi index okuma
- Dizi bounds check
- Heap allocation
- Refcount
- Scope cleanup
- Runtime panic

## Su Anda Backend'in Sinirlari

Backend guclendi ama hala tamamlanmis bir production compiler degil.

Bilinen sinirlar:

- Backend Windows x64 odakli.
- Linux/macOS backend yok.
- LLVM backend yok.
- Optimizer henuz ciddi sekilde yok.
- IR var ama henuz kapsamli optimization pass'leri yok.
- Dizi elemani genel dynamic `deger` olarak her expression'da kullanilamiyor.
- Dizi yazdirma dogrudan `yazdir(dizi)` olarak desteklenmiyor; eleman
  yazdirma destekli.
- Garbage collector yok; refcount var.
- Refcount su an tek-thread varsayimi ile non-atomic.
- Closure, struct, module sistemi yok.
- Error recovery ve diagnostic kalitesi daha da artabilir.
- Register allocation yok; basit stack slot modeli kullaniliyor.

Bu sinirlar kotu degil; aksine hangi yonde buyuyecegimizi net gosteriyor.

## Optimizer Icin Zemin

IR katmani optimizer icin dogal yer.

Ileride eklenebilecek basit optimizer pass'leri:

- Constant folding:

  ```text
  2 + 3 -> 5
  ```

- Constant condition:

  ```text
  eger (dogru) { A } degilse { B } -> A
  ```

- Dead expression cleanup:

  ```text
  1 + 2;
  ```

  sonucu kullanilmiyorsa silinebilir.

- Redundant assignment cleanup:

  ```text
  a = a;
  ```

- Basit copy propagation:

  ```text
  b = a;
  yazdir(b);
  ```

  uygun durumlarda `a` kullanilabilir.

Daha ileri optimizer icin IR'in SSA benzeri bir forma donusmesi gerekebilir,
ama mevcut IR baslangic icin yeterli.

## Neden LLVM Kullanilmadi?

LLVM cok guclu ama baslangic icin agir bir bagimlilik olurdu.

Su an elle assembly uretmek bize sunlari ogretti ve kazandirdi:

- Calling convention nasil calisiyor?
- Stack frame nasil kurulur?
- Runtime cagrilari nasil yapilir?
- Heap pointer ownership nasil takip edilir?
- Scope cleanup nerede gerekir?
- Exe nasil linklenir?

Yani proje egitim ve kontrol acisindan daha seffaf oldu.

Ileride LLVM backend eklemek mumkun olabilir. O durumda mevcut frontend ve IR
katmani yine kullanilabilir.

## Hocaya Anlatilabilecek Kisa Backend Cevabi

"Anadil'de backend tarafinda Rust ile yazilmis compiler, semantik analizden
sonra programi typed AST ve IR uzerinden Windows x64 assembly'ye indiriyor.
Bu assembly MASM ile objeye, link.exe ile exe'ye cevriliyor. Runtime tarafinda
ayri bir assembly kutuphanemiz var; sayi/metin/mantik yazdirma, heap allocation,
refcount, metin concat/esitlik ve dinamik dizi islemleri burada. Yani proje
sadece interpreter degil; native exe ureten bir compiler yoluna sahip."

## Sorulabilecek Sorular

### Bu dil yorumlaniyor mu, derleniyor mu?

Ikisi de var; ama urun hedefi native compiler. `yorumla` komutu interpreter
yoludur. `calistir` ve `derle` native backend'i kullanir. `calistir` once exe
derler, sonra calistirir.

### Uretilen exe Rust gerektiriyor mu?

Hayir. Rust compiler sadece Anadil compiler'ini gelistirmek icin gerekiyor.
Uretilen Anadil exe'si Rust kurulu olmayan makinede de calisabilir. Ancak yeni
`.ana` dosyasindan exe uretmek icin Windows Build Tools gerekir.

### Backend C mi uretiyor?

Hayir. C'ye transpile etmiyor. Dogrudan Windows x64 MASM assembly uretiyor.

### Runtime neden gerekli?

Her seyi inline assembly olarak uretmek hem buyuk hem hataya acik olurdu.
Yazdirma, heap allocation, metin concat, dizi bounds check gibi ortak isler
runtime kutuphanesinde tutuluyor.

### Dizi neden tag/payload seklinde?

Cunku diziler heterojen. Ayni dizi icinde sayi, mantik, metin ve baska dizi
durabilir. Bu yuzden her eleman kendi runtime tag'ini tasimak zorunda.

### Garbage collector var mi?

Hayir. Su an refcount var. Heap nesneleri paylasildikca refcount artiyor,
scope/assignment/return sonunda azaltiliyor. Sifira inince runtime free ediyor.

### Optimizer var mi?

Henuz ciddi optimizer yok. IR katmani optimizer icin hazirlik niteliginde.
Ilk optimizer adimlari constant folding ve dead code temizligi olabilir.

### Backend portable mi?

Su an asil hedef Windows x64. Runtime ve assembly Windows API/MASM odakli.
Linux/macOS icin ya yeni backend ya da LLVM gibi daha tasinabilir bir hedef
gerekir.

### En teknik basari ne?

Sadece arithmetic assembly emit etmekten cikilip heap metin, refcount cleanup,
runtime library, exe-relative runtime path, native dizi literal/index/assignment
gibi daha gercek compiler problemlerinin cozulmeye baslanmasi.

## Sonuc

Anadil backend'i su an "kaynak kodu calistiran interpreter" seviyesinden cikmis
durumda. Kod once anlamlandiriliyor, sonra native assembly'ye indiriliyor,
runtime kutuphanesiyle linkleniyor ve exe olarak calisiyor.

En onemli ilerleme, backend'in sadece sayi islemleri yapan basit bir emitter
olmaktan cikmasi. Metin ve dizi gibi heap nesneleri, refcount, scope cleanup,
runtime panic ve paketlenebilir runtime yolu artik sistemin bir parcasi.

Bundan sonraki dogal adimlar:

1. IR uzerinde basit optimizer.
2. Dynamic `deger` modelini dizi disinda da guclendirmek.
3. Dizi ustunde daha fazla operasyon.
4. Backend testlerini edge-case seviyesinde genisletmek.
5. Daha tasinabilir backend stratejisi dusunmek.

