# FastTree (H#)

Następca OSTree: content-addressable, zorientowany blokowo, bez hardlinków.
**Cała biblioteka i CLI są napisane w H#.** Dla innych języków jest stabilne
C ABI (`ft_*`).

## Dlaczego nie OSTree

| Cecha | OSTree | FastTree |
|---|---|---|
| Struktura repo | zbiór plików + hardlinki | jeden obraz (composefs/EROFS) + CAS na chunkach |
| Haszowanie | SHA-256, jednowątkowe | BLAKE3, drzewo Merkle |
| Weryfikacja integralności | przy przełączaniu commitów | dm-verity + digest composefs |
| Modyfikacje lokalne | stateroot | natywne warstwy OverlayFS (ulotne/trwałe) |
| Deduplikacja | całe pliki (hardlink) | chunki FastCDC o zmiennym rozmiarze |
| GC | `ostree prune` (po commitach) | `fasttree gc` (po chunkach, live-set z `CURRENT_TAG` + pins) |
| Autoryzacja rejestru | n/d | pełny OCI Bearer flow (401 → token → retry) |

## Układ repozytorium

```
Bit.hk                  manifest pakietu (bit): lib + CLI, komendy build-lib/test-all/fsck
src/lib.h#              wejście biblioteki (zbiera moduły)
src/main.h#             binarka `fasttree`
src/blake3.h#           BLAKE3 w czystym H# (zweryfikowany względem referencji)
src/hashing.h#          hashe jako hex, algorytm (blake3 | sha256-fallback)
src/chunker.h#          FastCDC (gear hash)
src/json.h#             parser/serializator JSON (std -> json w H# 0.9 jest płaski)
src/sys.h#              narzędzia zewnętrzne z kodem wyjścia, listowanie drzewa, stderr
src/store.h#            lokalny CAS (shardowany, wersjonowany plikiem FORMAT)
src/manifest.h#         manifest = drzewo Merkle rootfs; build / JSON / diff
src/layers.h#           warstwy OCI: tar + whiteout + opaque + hardlinki + xattr
src/gc.h#               mark-and-sweep (CURRENT_TAG + pins.json)
src/repo.h#             tagi, pins/unpin, rollback, usuwanie tagów
src/derive.h#           derive: nowy tag = baza + zmiany (layering)
src/fsck.h#             weryfikacja integralności: store, manifest, całe repo
src/stats.h#            statystyki: rozmiary, deduplikacja, sieroty, szacunek GC
src/ioengine.h#         wsadowe I/O (backend sekwencyjny + `cat`)
src/oci.h#              klient rejestru OCI (transport: curl)
src/composefs.h#        obraz composefs + mount + dm-verity
src/overlay.h#          OverlayFS (ulotny tmpfs / trwały)
src/cli.h#              komendy CLI
src/ft.h#               STABILNE C ABI (symbole ft_*) — jedyna powierzchnia FFI
src/tests/*_test.h#     94 testy jednostkowe H#
scripts/                build-lib.sh, run-tests.sh
```

## Budowanie

Wymaga kompilatora `h#` 0.9+ z zainstalowaną biblioteką standardową.

```bash
scripts/build-lib.sh [--release]   # build/libfasttree.so, .a i build/fasttree
scripts/run-tests.sh [wzorzec]     # testy H# (odtwarza harness `bit test`)
bit build --emit so                # alternatywnie przez bit (.hlib | so | a)
```

Zależności uruchomieniowe (narzędzia, nie biblioteki): `tar`, `cp`, `curl`,
`find`, `ln`; dla `deploy`/`overlay` dodatkowo `mkcomposefs`, `mount.composefs`,
`veritysetup`, `losetup`, `mount` (composefs trzeba zbudować ze źródeł:
`meson setup build -Dfuse=disabled && ninja -C build`).

## Model systemu

Jak FastTree ma się do Fedora Silverblue, skąd biorą się aktualizacje i pakiety
Fedory oraz czego jeszcze brakuje do pełnego systemu: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

## Komendy CLI

```
fasttree pull <registry>/<repo>:<tag> [--as <nazwa>]   pobierz obraz OCI -> manifest + chunki w CAS
fasttree snapshot <katalog> <tag>         manifest z lokalnego katalogu (bez rejestru)
fasttree list                             tagi z oznaczeniami [current, previous, pinned]
fasttree stats [tag]                      rozmiary, deduplikacja, sieroty, ile zwolni gc
fasttree fsck [--repair]                  weryfikacja store + manifestów (kod 1 przy problemach)
fasttree verify <tag>                     weryfikacja jednego manifestu względem store
fasttree diff <tag-a> <tag-b>             różnice na poziomie plików i chunków
fasttree checkout <tag> <katalog>         odtwórz drzewo ze store (katalog nie może istnieć)
fasttree deploy <tag> [--atomic]          obraz composefs + dm-verity + symlink current
fasttree derive <baza> <katalog-zmian> <nowy-tag>   nowy tag = baza + zmiany (katalog, upperdir overlayfs, warstwa OCI)
fasttree overlay commit <nazwa> <nowy-tag>   zapisz zmiany overlaya na aktywnym wdrożeniu jako tag
fasttree rollback                         wróć do PREVIOUS_TAG (atomowa podmiana symlinku)
fasttree pin <tag> "<notatka>" | pins | unpin <tag>
fasttree rm <tag>                         usuń manifest/artefakty (nie aktywny, nie przypięty)
fasttree gc [--dry-run]                   mark-and-sweep po chunkach
fasttree status [--diff] | overlay create|remove|list|diff | version
```

`fsck` wykrywa uszkodzone obiekty (hash != nazwa), brakujące chunki, niezgodne
rozmiary i root, `CURRENT_TAG`/piny bez manifestu. `--repair` usuwa uszkodzone
obiekty i porzucone `*.tmp.*`; manifestów nie naprawia — brakujące chunki trzeba
dociągnąć ponownym `pull`/`snapshot`.

## Użycie z H#

```h#
mod store
mod manifest

fn main() is
    let o: store::OpenResult = store::open("/var/lib/fasttree/store")
    let b: manifest::BuildResult = manifest::build_default("/srv/rootfs", o.store)
    write(b.manifest.root)
end
```

## C ABI

Eksportuje je moduł `ft` jako `ft_<nazwa>` (`int` → `int64_t`, `string` →
`const char*`). Konwencje: bezstanowe (store/manifest po ścieżce, bez uchwytów),
wyniki złożone jako JSON `{"ok":true,...}` / `{"ok":false,"error":"..."}`,
dane binarne jako hex lub pliki, zwróconych napisów **nie wolno zwalniać**.
Pełna lista: `src/ft.h#`. Rozszerzenia 0.3 (addytywne, `abi_version` nadal 1,
`abi_minor` = 1; `root` = katalog repo jak w `gc_run`):
`repo_tags`, `repo_pins`, `repo_unpin`, `repo_rollback`, `repo_remove_tag`,
`repo_derive` (`abi_minor` = 2),
`fsck_store`, `fsck_manifest`, `fsck_repo`, `stats_manifest`, `stats_repo`.

## Format na dysku (bez zmian, `formatVersion = 1`)

Format jest stabilny: store i manifest zapisane przez dowolną wersję 0.x czyta
każda inna (ten sam `root`, te same granice chunków, ten sam layout).

* **Manifest** `manifests/<tag>.json`: `formatVersion`, `hashAlgo`, `chunker`
  (min/avg/max), `root`, `entries[]` posortowane po `path` (bajtowo).
  Czytelnik z nowszym `formatVersion` **musi** odrzucić plik.
* **Store** `store/objects/<hex[0:2]>/<hex[2:]>` + plik `FORMAT` (dziś `1`).

## Status — co zweryfikowano, a co nie

Zweryfikowane przez uruchomienie (kompilator h# 0.9.0, Linux x86_64):

* BLAKE3: 14 wektorów 0–100 000 B zgodnych z oficjalnym `blake3` (Python).
* FastCDC: granice i hashe chunków identyczne z modelem referencyjnym.
* Root Merkle manifestu przeliczony niezależnie w Pythonie — zgodny.
* Pełny `pull` z fikcyjnego rejestru z **Bearer auth**, whiteoutem, opaque
  whiteoutem, hardlinkiem, symlinkiem i bitem wykonania; rekonstrukcja plików
  ze store bajt w bajt; `pin`, `gc` (usuwanie sierot, abort przy braku manifestu).
* 94 testy H# (0.3: fsck, repo, stats, ABI na poziomie H#, CLI end-to-end; 0.4: derive).
* 0.3: pełny przepływ CLI (snapshot, diff, checkout, fsck z uszkodzeniem i `--repair`, pin/rm/rollback) na prawdziwych plikach.

**Nie zweryfikowane** (brak środowiska): symbole C `ft_*` rozszerzeń 0.3 (sprawdzone tylko z poziomu H#; potrzebny backend LLVM — z backendem C++ kompilator eksportuje nazwy zmanglowane), `.github/workflows/ci.yml`, prawdziwe `mkcomposefs`/`mount.composefs`, `overlay commit` na zamontowanym overlayu (logika `derive` jest testowana na katalogach),
dm-verity z device-mapper, `mount -t overlay` (w testach `deploy` użyto atrap
narzędzi; ścieżki błędów i wykrywanie możliwości są testowane), `ci.yml` na
prawdziwym GitHub Actions.

## Znane ograniczenia

* **Brak `io_uring`** — H# 0.9 nie ma async I/O ani FFI do liburing.
  `ioengine` zachowuje API (`IoJob`, `read_batch`, `write_batch`), backend jest
  sekwencyjny; składanie plików z chunków idzie przez `cat`.
* **Pamięć**: runtime H# 0.9 nie ma GC, a tablica `[int]` zajmuje 8 B/bajt.
  Pliki są chunkowane w pamięci — dla wielogigabajtowych plików CLI zużyje
  proporcjonalnie dużo RAM. Zwrócone przez C ABI napisy żyją do końca procesu.
* **Ścieżki z tabulatorem lub nową linią** są odrzucane (`list_tree` zwraca błąd).
* **HTTP przez `curl`** (wbudowany klient H# nie ma TLS ani nagłówków).
  Bloby mają limit 1 h na żądanie.
* Hash: `FASTTREE_HASH=sha256` przełącza z BLAKE3 na SHA-256 (fallback).

## Obejścia błędów kompilatora H# 0.9 (patrz komentarze w kodzie)

`u32`/`u64` przesuwają arytmetycznie (stąd `int` + maski), `while i < <literał>
255` bywa obcinane do 8 bitów, `to_string(<wyrażenie>)` i `.len()` w warunkach
potrafią ucinać wartości lub wywalać program, pole struktury w warunku
potrafi trafić w złe pole, literały z `{...}` są interpolowane, `.pop()` i
`std -> strbuf/map` nie mają kodowania w LLVM, `assert_eq` porównuje napisy po
wskaźniku (stąd `src/tests/check.h#`). Kod unika tych konstrukcji; przy
aktualizacji kompilatora warto je ponownie sprawdzić.
