# FastTree (H#)

Następca OSTree: content-addressable, zorientowany blokowo, bez hardlinków.
**Cała biblioteka i CLI są napisane w H#** — bez Nima, Rusta i C w rdzeniu.
Dla innych języków jest stabilne C ABI (`ft_*`), a dla **Nim** gotowy binding
w [`bindings/nim`](bindings/nim).

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
Bit.hk                  manifest pakietu (bit): lib + CLI, komendy build-lib/test-all/test-nim
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
src/ioengine.h#         wsadowe I/O (API jak w wersji Nim; backend sekwencyjny + `cat`)
src/oci.h#              klient rejestru OCI (transport: curl)
src/composefs.h#        obraz composefs + mount + dm-verity
src/overlay.h#          OverlayFS (ulotny tmpfs / trwały)
src/cli.h#              komendy CLI
src/ft.h#               STABILNE C ABI (symbole ft_*) — jedyna powierzchnia FFI
src/tests/*_test.h#     47 testów jednostkowych H#
bindings/nim/           binding Nim (nimble) nad C ABI + 8 testów
scripts/                build-lib.sh, run-tests.sh, test-nim.sh, fake_registry.py
```

## Budowanie

Wymaga kompilatora `h#` 0.9+ z zainstalowaną biblioteką standardową.

```bash
scripts/build-lib.sh [--release]   # build/libfasttree.so, .a i build/fasttree
scripts/run-tests.sh [wzorzec]     # testy H# (odtwarza harness `bit test`)
scripts/test-nim.sh                # biblioteka + testy bindingu Nim
bit build --emit so                # alternatywnie przez bit (.hlib | so | a)
```

Zależności uruchomieniowe (narzędzia, nie biblioteki): `tar`, `cp`, `curl`,
`find`, `ln`; dla `deploy`/`overlay` dodatkowo `mkcomposefs`, `mount.composefs`,
`veritysetup`, `losetup`, `mount` (composefs trzeba zbudować ze źródeł:
`meson setup build -Dfuse=disabled && ninja -C build`).

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

## Binding Nim

```nim
import fasttree

let s = openStore("/var/lib/fasttree/store")
let h = s.put(@[1'u8, 2, 3])          # hash hex
echo s.get(h)                          # some(@[1, 2, 3])
let m = buildManifest("/srv/rootfs", s)
echo m["root"].getStr
```

Budowanie i linkowanie: patrz [`bindings/nim/README.md`](bindings/nim/README.md).

## C ABI

Eksportuje je moduł `ft` jako `ft_<nazwa>` (`int` → `int64_t`, `string` →
`const char*`). Konwencje: bezstanowe (store/manifest po ścieżce, bez uchwytów),
wyniki złożone jako JSON `{"ok":true,...}` / `{"ok":false,"error":"..."}`,
dane binarne jako hex lub pliki, zwróconych napisów **nie wolno zwalniać**.
Pełna lista: `bindings/nim/src/fasttree/ffi.nim` i `src/ft.h#`.

## Format na dysku (bez zmian, `formatVersion = 1`)

Zgodny z wersją Nim: store i manifest utworzone starą implementacją czyta nowa
i odwrotnie (ten sam `root`, te same granice chunków, ten sam layout).

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
* 47 testów H# i 8 testów Nim (dynamicznie i statycznie).
* ABI wywołane z C i z Nima.

**Nie zweryfikowane** (brak środowiska): prawdziwe `mkcomposefs`/`mount.composefs`,
dm-verity z device-mapper, `mount -t overlay` (w testach `deploy` użyto atrap
narzędzi; ścieżki błędów i wykrywanie możliwości są testowane), `ci.yml` na
prawdziwym GitHub Actions.

## Znane ograniczenia (świadome różnice względem wersji Nim)

* **Brak `io_uring`** — H# 0.9 nie ma async I/O ani FFI do liburing.
  `ioengine` zachowuje API (`IoJob`, `read_batch`, `write_batch`), backend jest
  sekwencyjny; składanie plików z chunków idzie przez `cat`.
* **Pamięć**: runtime H# 0.9 nie ma GC, a tablica `[int]` zajmuje 8 B/bajt.
  Pliki są chunkowane w pamięci — dla wielogigabajtowych plików CLI zużyje
  proporcjonalnie dużo RAM. Zwrócone przez C ABI napisy żyją do końca procesu.
* **Ścieżki z tabulatorem lub nową linią** są odrzucane (`list_tree` zwraca błąd).
* **HTTP przez `curl`** (wbudowany klient H# nie ma TLS ani nagłówków).
  Bloby mają limit 1 h na żądanie (w Nim było 30 s, co przerywało duże warstwy).
* Hash: `FASTTREE_HASH=sha256` zastępuje `-d:fasttreeNoBlake3`.

## Obejścia błędów kompilatora H# 0.9 (patrz komentarze w kodzie)

`u32`/`u64` przesuwają arytmetycznie (stąd `int` + maski), `while i < <literał>
255` bywa obcinane do 8 bitów, `to_string(<wyrażenie>)` i `.len()` w warunkach
potrafią ucinać wartości lub wywalać program, pole struktury w warunku
potrafi trafić w złe pole, literały z `{...}` są interpolowane, `.pop()` i
`std -> strbuf/map` nie mają kodowania w LLVM, `assert_eq` porównuje napisy po
wskaźniku (stąd `src/tests/check.h#`). Kod unika tych konstrukcji; przy
aktualizacji kompilatora warto je ponownie sprawdzić.
