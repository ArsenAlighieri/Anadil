# Anadil V0.1 Local Release Validation

Tarih: 2026-05-10
Ortam: Windows, Visual Studio Build Tools mevcut

## Paketleme

- `pwsh -File .\package.ps1` lokal makinede calistirilamadi: `pwsh`
  PATH icinde yok.
- `powershell -NoProfile -ExecutionPolicy Bypass -File .\package.ps1`
  basariyla calisti.
- Uretilen ZIP:
  `target/dist/Anadil-v0.1.0-windows-x64.zip`
- SHA256:
  `71BE933CFDD339047188DC68AA65D1245524B69A641807AC59895C7A75EE256E`

## Icerik Kontrolu

`target/dist/Anadil-v0.1.0-windows-x64/` altinda beklenen dosyalar
dogrulandi:

- `anadil.exe`
- `anadil-ide.exe`
- `runtime/anadil_runtime.lib`
- `runtime/anadil_runtime.asm`
- `examples/*.ana`
- `docs/*.md`
- `KURULUM.txt`
- `CHANGELOG.txt`
- `LICENSE.txt`
- `README.txt`

## ZIP Smoke Test

ZIP ayri bir `target/release-validation-*` klasorune acildi.

- `anadil.exe yardim` basariyla calisti.
- `anadil.exe yorumla examples\topla.ana` basariyla calisti ve `30`
  yazdirdi.

## Installer

`makensis.exe` PATH icinde bulunamadi; lokal `-Installer` dogrulamasi
atlanmistir. CI workflow NSIS'i kendi kurdugu icin bu lokal ortam
eksikligi release blocker olarak isaretlenmedi.

---

# Anadil V0.3 Local Release Validation

Tarih: 2026-05-23
Ortam: Windows, Visual Studio Build Tools mevcut

## Paketleme

- `pwsh -File .\package.ps1` lokal makinede calistirilamadi: `pwsh`
  PATH icinde yok.
- `powershell -ExecutionPolicy Bypass -File .\package.ps1` basariyla calisti.
- Uretilen ZIP:
  `target/dist/Anadil-v0.3.0-windows-x64.zip`
- SHA256:
  `537C22A5BD8A07A5856BDC0323D7E1E46BBAC0819866482C36F9E5EA6172C549`

## Icerik Kontrolu

ZIP `target/smoke-v0.3.0/` klasorune acildi. Beklenen temel dosyalar
dogrulandi:

- `anadil.exe`
- `anadil-ide.exe`
- `runtime/anadil_runtime.lib`
- `runtime/anadil_runtime.asm`
- `examples/dizi_v03.ana`
- `docs/DIL_REFERANSI.md`
- `KURULUM.txt`
- `CHANGELOG.txt`
- `LICENSE.txt`
- `README.txt`

## ZIP Smoke Test

- `anadil.exe yardim` basariyla calisti ve `Anadil 0.3.0` gosterdi.
- `anadil.exe yorumla examples\dizi_v03.ana` basariyla calisti.
- `anadil.exe derle examples\dizi_v03.ana` basariyla `dizi_v03.exe` uretti.
- `examples\dizi_v03.exe` basariyla calisti.

Beklenen/cikan dizi ornegi ciktisi:

```text
3
1
bir
iki
```
