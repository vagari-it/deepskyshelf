# DeepSkyShelf

Natywna aplikacja macOS do porządkowania biblioteki astrofotografii: katalog klatek FITS,
przygotowanie materiału do stackowania (Siril, PixInsight WBPP, AstroWizard), archiwizacja
i odzyskiwanie miejsca. Wzorowana na [FitsInsight](https://fitsinsight.com/guide), ale działa
w całości lokalnie.

## Zasady

- **Oryginały są nietykalne.** Skaner tylko czyta nagłówki. Przygotowanie do stackowania będzie
  tworzyć foldery z hard linków (zero dodatkowego miejsca).
- **Nic nie znika bez podglądu.** Usuwanie zawsze najpierw pokazuje listę i rozmiar, domyślnie
  przenosi do Kosza, a do archiwum kopiuje z weryfikacją sum kontrolnych.
- **Nazwa obiektu to nie wszystko.** Projekty łączą się po współrzędnych (np. `1318-kombi` = `NGC 6910`).

## Budowanie

Wymaga Xcode 16+ (Swift 6). Bez projektu Xcode, wszystko przez Swift Package Manager:

```bash
./scripts/build-app.sh          # build/DeepSkyShelf.app + build/deepskyshelf (CLI)
open build/DeepSkyShelf.app
swift test
```

Pakiet można też otworzyć w Xcode (`open Package.swift`) i uruchamiać schemat `DeepSkyShelfApp`.

## Narzędzie CLI

Korzysta z tego samego katalogu co aplikacja (`~/Library/Application Support/DeepSkyShelf/catalog.sqlite`).

```bash
build/deepskyshelf scan ~/Pictures/Asiair   # dodaj folder i zeskanuj
build/deepskyshelf import                   # wykryte urządzenia (ASIAIR, Seestar, pendrive)
build/deepskyshelf import "/Volumes/EMMC Images" --dry-run   # co jest nowe i gdzie trafi
build/deepskyshelf scan                     # ponowny skan (tylko nowe/zmienione pliki)
build/deepskyshelf targets                  # obiekty i integracja per filtr
build/deepskyshelf sessions                 # noce
build/deepskyshelf calibration --verbose    # dopasowanie darków, flatów i biasów, lista braków
build/deepskyshelf calibration --library    # czego brakuje w CalibrationLibrary
build/deepskyshelf quality --target 1318      # ocena jakości (mierzy nowe klatki) i podsumowanie nocy
build/deepskyshelf stage "1318" --program astrobbq --dry-run   # plan folderu do stackowania
build/deepskyshelf stage "M 33" --program deepskyshelf --run --astrowizard
build/deepskyshelf storage                  # zajętość wg roli plików, duplikaty
build/deepskyshelf cleanup                  # podgląd porządków (niczego nie zmienia)
build/deepskyshelf destinations add-folder NAS /Volumes/astro/Archiwum
DEEPSKYSHELF_S3_SECRET=… build/deepskyshelf destinations add-s3 B2 --endpoint https://s3.eu-central-003.backblazeb2.com \
    --region eu-central-003 --bucket moj-bucket --key-id …
build/deepskyshelf archive 1318-kombi --to NAS --dry-run
```

Ikona aplikacji: zapisz kwadratowy PNG (najlepiej 1024×1024) jako `Resources/AppIcon.png`, skrypt zbuduje z niego `.icns`.

## Struktura

| Moduł | Zawartość |
|---|---|
| `DeepSkyShelfCore/FITS` | Parser nagłówków FITS (bloki 2880 B, HIERARCH, daty z mikrosekundami, RA/Dec sześćdziesiątkowo) |
| `DeepSkyShelfCore/Scan` | Skaner przyrostowy (mtime + rozmiar), klasyfikacja typu (light/dark/flat/bias) i roli pliku |
| `DeepSkyShelfCore/Catalog` | Baza SQLite (GRDB), podsumowania: obiekty, noce, zajętość, duplikaty |
| `DeepSkyShelfCore/Calibration` | Zestawy kalibracyjne, serie lightów, dopasowanie z uwagami, braki, CalibrationLibrary |
| `DeepSkyShelfCore/Quality` | Odczyt pikseli FITS, pomiar gwiazd (FWHM, eliptyczność, liczba, tło), ocena 0–10, decyzje |
| `DeepSkyShelfCore/Import` | Wykrywanie urządzeń, plan importu z nagłówków, kopia z SHA-256, cofanie importu |
| `DeepSkyShelfCore/Staging` | Plany folderów dla Siril/WBPP/AstroWizard, hard linki, skrypty `.ssf`, uruchamianie `siril-cli` |
| `DeepSkyShelfCore/Cleanup` | Plan porządków, Kosz z dziennikiem i cofaniem, zamiana duplikatów na hard linki |
| `DeepSkyShelfCore/Archive` | Archiwizacja do folderu/NAS i S3/B2 (własny klient SigV4), manifest, przywracanie, pęk kluczy |
| `DeepSkyShelfApp` | Interfejs SwiftUI |
| `deepskyshelf-cli` | Narzędzie wiersza poleceń |

**Role plików:** *Surowe* (pojedyncze klatki z kamery; nazwa `Light_`/`Dark_`/… chroni plik w każdym folderze),
*Pośrednie* (wszystko w `process/` i `cache/` oraz niestackowane `pp_*`/`r_*`, do odtworzenia),
*Wyniki* (`result_*`, `starless_*`, stacki z `STACKCNT`, obrazy RGB, pliki bez typu klatki), *Mastery* (master dark/flat/bias).
Ręcznie robione foldery (`work/`, `backup/`) nigdy nie są uznawane za pośrednie.

## Porządki i archiwum: zasady bezpieczeństwa

- Usuwanie zawsze trafia do **Kosza** i jest zapisane w Historii operacji z przyciskiem *Przywróć*.
- Plik zmieniony od skanu (inny rozmiar) jest pomijany.
- Domyślnie zaznaczone są tylko foldery robocze projektów nieruszanych od 14 dni, w których każdy plik ma
  oryginał (dopasowanie po kamerze, DATE-OBS i czasie ekspozycji).
- Duplikaty zamieniane są na **hard linki** dopiero po porównaniu bajt po bajcie; wszystkie ścieżki działają dalej.
- Archiwum: NAS po skopiowaniu jest odczytywany ponownie i sprawdzany SHA-256; w S3/B2 serwer odrzuca plik,
  jeśli suma SHA-256 lub Content-MD5 się nie zgadza. Manifest `deepskyshelf-manifest.json` zapisywany jest na końcu.
- Lokalne kopie można usunąć (do Kosza) dopiero po bezbłędnej archiwizacji; klatki zostają w katalogu jako
  „tylko w archiwum”, więc godziny integracji się nie zmieniają. Przywracanie sprawdza każdy plik sumą SHA-256.

## Plan

- [x] **Etap 1: katalog.** Skanowanie, obiekty, noce, wszystkie pliki, zajętość, duplikaty, uszkodzone pliki.
- [x] **Etap 2: kalibracja.** Dopasowanie darków (kamera, gain, offset, binning, czas ±1 s, temp. ±3,5 °C),
      flatów (filtr, ogniskowa ±5 %, kąt kamery ±5° modulo 180°, najbliższa noc) i biasów/dark flatów do flatów.
      Pokrycie per seria, lista „Do zrobienia”, profile kamer, uzupełnianie `CalibrationLibrary` hard linkami.
- [x] **Etap 3: przygotowanie do stackowania.** Foldery z hard linków w `<biblioteka>/Stacking/<obiekt>/`:
      Siril ze skryptami AstroBBQ (SHO/LRGB/RGB, Twój HOO, Mono/OSC; osobne foldery dla różnych darków, jak `sho/` i `rgb/`),
      Siril ze skryptem DeepSkyShelf (każda noc z własnymi flatami, `merge` przed `register`, wynik `result_<filtr>.fit`),
      WBPP (`LIGHT/NIGHT_…/FILTER_…` + README), AstroWizard (mastery wyrównane `register -2pass` + `seqapplyreg -framing=min`
      jako `AstroWizard/H.fit, O.fit, S.fit…`). Siril uruchamiany z aplikacji (`siril-cli`), z logiem i przerywaniem.
- [x] **Etap 4: import nowych nocy.** Z ASIAIR (udział SMB), pendrive'a i Seestara (USB), wykrywanych po podłączeniu.
      Lighty do `Obiekt/Noc/Filtr`, flaty do `Obiekt/Noc/Flat/Filtr` (do każdego obiektu z tej nocy, hard linki),
      darki i biasy do `CalibrationLibrary/<Kamera>/<Typ>/<wariant>`. Pomija to, co już jest w bibliotece, stacki Live
      i ucięte pliki. Każda kopia sprawdzana SHA-256; pliki na urządzeniu zostają; import można cofnąć.
- [x] **Etap 5: jakość klatek.** Własny pomiar gwiazd (tło i szum z histogramu, detekcja z odrzuceniem gorących pikseli,
      FWHM i eliptyczność z momentów, OSC łączone 2×2). Ocena 0–10 względem mediany obiektu/kamery/filtra:
      ostrość 35 %, przejrzystość 35 %, eliptyczność 15 %, tło 15 %, z limitami dla utraty gwiazd i FWHM > 1,45× mediany.
      Na ręcznie odrzuconych klatkach 1318-kombi: 97 % obstrukcji i 71 % mgły poniżej progu 5, 2 % fałszywych odrzuceń.
      Próg do ustawienia, ręczne przyjęcie/odrzucenie, foldery `rejected/` honorowane, odrzucone klatki pomijane w stackowaniu.
      Podsumowanie nocy: przydatne klatki i godziny, ocena słowna i epizody z przyczyną (mgła/chmury, przeszkoda/rosa,
      gorsza ostrość, smugi, jaśniejsze niebo); w tabeli wartości względem typowej klatki serii i dymki z wyjaśnieniem.
- [x] **Porządki: darki i biasy w projektach.** Kopie z `CalibrationLibrary` leżące w folderach projektów; do Kosza tylko
      po potwierdzeniu, że kopia w bibliotece jest tym samym plikiem albo identyczna bajt po bajcie.
- [x] **Etap 6: archiwizacja.** NAS (SMB) i chmura (S3/B2), manifest z sumami, pliki w archiwum widoczne jako offline,
      przywracanie. Do zrobienia: opcjonalna bezstratna kompresja `fpack`, pliki > 5 GB (multipart).
- [x] **Etap 7: odzyskiwanie miejsca.** Pliki pośrednie, duplikaty (hard linki), uszkodzone pliki. Podgląd, Kosz, historia z cofaniem.

## Licencja

DeepSkyShelf © 2026 vagari.it. Wszelkie prawa zastrzeżone.

Program jest darmowy (freeware) i ma zamknięty kod: wolno go używać i przekazywać dalej
bezpłatnie, w niezmienionej postaci; nie wolno go zmieniać ani sprzedawać. Pełne warunki są
w pliku [LICENSE](LICENSE). Kodu źródłowego nie publikujemy.

Korzysta z [GRDB.swift](https://github.com/groue/GRDB.swift) (MIT) i
[Swift Argument Parser](https://github.com/apple/swift-argument-parser) (Apache 2.0);
ich licencje są dołączone do aplikacji w `Contents/Resources/Licenses`.
