# Strona poza mapę
## Opowieść o uratowaniu Vortex 2

### Prolog. Karta, która nie chciała odejść na emeryturę

Nie każdy sprzęt komputerowy starzeje się spokojnie. Niektóre urządzenia po latach nadal potrafią zadziwić — tyle że czasem robią to przez niebieski ekran, zawieszenie albo restart w najmniej odpowiednim momencie.

Na biurku, wewnątrz nowoczesnego komputera z Ryzenem 3 3100, 16 GB pamięci RAM i RTX 3060, siedziała taka właśnie karta: **Monster Sound MX300 z układem Aureal Vortex 2 AU8830**. Pamiętała czasy Windows 2000, panelu „Stereo x2” i sterowników nagrywanych na płyty. Obok niej stał współczesny Linux Mint, PipeWire, Firefox i Audacious.

Karta była podłączona przez mostek ASMedia PCIe→PCI, miała rewizję `fe`, IRQ 24 i 32-bitową maskę DMA. Była zaprojektowana w epoce, gdy o AMD IOMMU nikt jeszcze nie myślał w kategorii codziennej ochrony pamięci.

Właściciel nie chciał od niej cudów. Nie chciał wavetable, kolekcji efektów A3D ani podziwu dla historii. Chciał tylko jednego: **normalnie słuchać muzyki**.

System miał jednak inny plan. Czasem restartował się przy odtwarzaniu, a raz nawet udało się go zamrozić tak, że trzeba było zaczynać od zera. Wcześniejsza próba naprawy przyniosła pół sukcesu: komputer przestał się zawieszać, ale dźwięk miał mikro-przerwy i wydawał się przesterowany.

Tak zaczęła się nasza wyprawa. Bez obietnic, że za jednym zaklęciem wszystko zadziała. Z zasadą: najpierw zrozumieć, dopiero potem zmieniać.

### Rozdział 1. Najpierw nie psuć

Zanim dotknąłem sterownika, zrobiłem coś mało spektakularnego, ale bardzo ważnego: zacząłem czytać.

W katalogu roboczym znalazły się źródła Linux `au88x0`, stare sterowniki Windows, animowany GIF z panelem karty, dwustronicowy PDF i kilka plików WAV. Sprawdziłem sprzęt bez uruchamiania dźwięku. Karta była już rozpoznana przez `snd_au8830`, choć w Xfce jej profil był „wyłączony”. Wyłączenie w panelu nie oznaczało wyładowania modułu z jądra.

W dzienniku zamiast zwykłych komunikatów o głośności znalazłem coś znacznie ciekawszego:

```text
AMD-Vi: IO_PAGE_FAULT
IRQ fifo error
```

DMA to sposób, w jaki karta przenosi dźwięk między pamięcią komputera a własnym układem bez udziału procesora. IOMMU jest jak kontroler graniczny: mówi, które strony pamięci urządzenie może odwiedzić. `IO_PAGE_FAULT` oznaczał, że karta próbowała przekroczyć granicę.

Podejrzenia mogły wskazywać IRQ, DMA albo mostek. Nie było sensu zgadywać. Zrobiłem migawkę wszystkiego: sumy SHA-256 oryginalnych plików, wersję jądra, PCI, IRQ, aktywne moduły i logi. Powstał lokalny commit `6410af4`. Drugi system, na dysku NVMe, miał być niedostępny — i przez całą pracę pozostał nietknięty.

Zabrałem się do źródeł. Pierwsza hipoteza była kusząca: może sterownik wpisuje do karty zwykły adres fizyczny zamiast adresu DMA? To częsty błąd przy IOMMU. Szybko okazało się jednak, że kod używa właściwego `snd_pcm_sgbuf_get_addr()` i jawnie ustawia maskę 32-bitową. Najprostsza odpowiedź odpadła. Nie było jednej oczywistej linii do poprawienia.

Dokument PDF okazał się dwustronicowym prospektem reklamowym, nie podręcznikiem rejestrów. Sterowniki Windows zostały więc na później. Najpierw trzeba było zobaczyć chorobę na żywym systemie.

### Rozdział 2. Małe pułapki przed wielką diagnozą

Zanim pojawił się pierwszy dźwięk, środowisko pracy samo zaczęło rzucać miłe niespodzianki.

`ripgrep` nie był zainstalowany, więc użyłem `grep` i `find`. Katalog nie był jeszcze repozytorium Git — zainicjowałem je. Makefile jądra raportował `make kernelrelease` jako `7.0.14`, podczas gdy system działał na `7.0.0-42-generic`. Zamiast wymusić kompilację na ślepo, sprawdziłem `include/config/kernel.release` i `utsrelease.h`. To one miały właściwą wersję modułu.

Potem okazało się, że `sudo` nie jest dostępne bez interakcji. Wspólna praca z użytkownikiem była więc konieczna: on wykonywał pojedyncze komendy administracyjne, ja czytałem wyniki i przygotowywałem kolejne bezpieczne kroki.

Kolejna pułapka była bardziej pouczna. Pierwszy `insmod` odrzucił moduł z `Unknown symbol`. Nie chodziło o uszkodzony kod. `modprobe -r snd_au8830` usunął także nieużywane zależności: `gameport`, `snd_ac97_codec` i `snd_mpu401_uart`. Skrypt został poprawiony tak, aby najpierw ładował wszystkie zależności odczytane z `modinfo`.

W międzyczasie OBS nagrywał pracę, a PipeWire nie chciało zostać zatrzymane. Zatrzymanie usługi nie wystarczyło, ponieważ `pipewire.socket` i `pipewire-pulse.socket` natychmiast ją wskrzeszały. Trzeba było zamknąć najpierw sockety, a dopiero potem usługi.

Dopiero po tych wszystkich drobnych bitwach własny moduł wszedł do jądra z `debug=7`. Żadnych nowych błędów w spoczynku. Nadszedł czas, aby nacisnąć „play”.

### Rozdział 3. Cztery okresy spokoju

Pierwszy plik trwał nieco ponad sześć sekund. Był to 16-bitowy dźwięk stereo przy 48 kHz, odtwarzany bezpośrednio przez ALSA, z czterema okresami po 4096 bajtów. Moduł zapisał cztery kolejne adresy DMA. `aplay` zakończył się kodem zero, a przerwania wystąpiły dokładnie tyle razy, ile wynikało z czasu trwania pliku.

Drugi taki test też przeszedł. Potem dwa jednoczesne strumienie — również bez restartu. Osiem okresów znowu nie sprawiało problemu.

To było zwodniczo uspokajające. Gdybym zakończył tam dzień, mógłbym napisać: „działa”. Jednak cztery i osiem okresów nie wypełniały całej 64-kilobajtowej alokacji. Była jeszcze jedna konfiguracja, którą trzeba było sprawdzić: szesnaście okresów, pełny bufor.

Uruchomiłem go.

Tym razem jądro znowu zaczęło zgłaszać `IO_PAGE_FAULT`, a po chwili `IRQ fifo error`. Aplikacja mimo wszystko zakończyła się kodem zero. To była ważna lekcja: **sukces użytkownika nie zawsze oznacza sukces sterownika**. Można nie zauważyć, że karta właśnie sięgnęła gdzieś, gdzie nie wolno.

Dane były zadziwiająco precyzyjne:

```text
bufor:       0xd0e40000 ... 0xd0e4ffff
pierwszy błąd: 0xd0e50000
```

Adres błędu był dokładnie pierwszą stroną za 64-kilobajtową alokacją. Start nastąpił o `7745.882307`, pierwszy błąd o `7746.221598`. Różnica wynosiła 0,339 sekundy. Szesnaście okresów po 1024 ramki przy 48 kHz trwa 0,341 sekundy. Nie wyglądało to na przypadek. Wyglądało na pierwsze zawinięcie pełnego pierścienia.

Ale dlaczego karta szła poza mapę?

### Rozdział 4. Strażnik ostatniej strony

Dodałem do sterownika mały, precyzyjny zapis tylko w momencie zawinięcia strony logicznej zero. Nie chodziło o zalewanie dziennika komunikatami przy każdym przerwaniu — to mogłoby samo powodować trzaski. Chodziło o jedną odpowiedź: czy procesor przekazuje karcie dobry adres przy zawinięciu?

Wynik był jednoznaczny. Slot zero został zapisany adresem początku bufora `0xd89f0000`, a odczyt rejestru potwierdził dokładnie tę samą wartość. Mimo to 63 milisekundy później karta sięgnęła pod `0xd8a00000`, czyli ponownie za cały obszar.

Nie było błędnego adresu od procesora. Było coś w zachowaniu samego mechanizmu DMA przy zawinięciu — dodatkowy odczyt poza logiczny koniec pierścienia.

Teraz pojawił się pomysł, który można było uczciwie sprawdzić zamiast spekulować dalej. ALSA i tak wstępnie przydziela 64 KiB. Jeśli logiczny bufor będzie miał 15, a nie 16 stron, to ostatnia strona pozostanie zamapowana jako **guard page**, bufor ochronny. Karta nie będzie już wychodzić poza alokację.

Test z 15 okresami przeszedł przez około dziewiętnaście zawinięć bez błędu. Następnie moduł ze stałym limitem `65536 - 4096 = 61440` bajtów logicznych przeszedł wszystkie kolejne próby: jeden strumień, dwa strumienie, a nawet 60 małych okresów po 1024 bajty. Gdy użytkownik zażądał 16 okresów, ALSA sama wynegocjowała bezpieczne 15 zamiast odmówić odtwarzania.

Nie wyłączyłem IOMMU. Nie oszukałem mapowania. Po prostu zostawiłem jedną stronę na straży.

Ta poprawka wyjaśniała też wcześniejsze maskowanie problemu: przy mniejszych buforach karta mogła przeczytać coś poza logicznym końcem, ale nadal wewnątrz większej wstępnej alokacji, więc IOMMU nie zgłaszało błędu. Dopiero pełne 64 KiB odsłaniało prawdziwą granicę.

### Rozdział 5. Drugi winowajca w tle

Na tym etapie łatwo było ogłosić zwycięstwo. Niestety, PipeWire miał jeszcze jeden test dla nas.

Po włączeniu profilu karty w dzienniku pojawiły się nowe błędy, tym razem z flagą `0x0020`. Poprzedzał je `hw_params: pcm=3`, ale nie było normalnego START. PCM 3 to wavetable — WT. Kod źródłowy od lat zawierał przyznanie, że kanały WT nie działają i ich DMA nie pracuje. Mimo to sterownik eksponował je programom, a PipeWire podczas rozpoznawania sprzętu próbował je skonfigurować.

To był drugi, niezależny problem. Guard działał dla zwykłego odtwarzania ADB, ale nie miał sensu udawać, że niedziałający WT jest gotowym produktem. Zrobiłem więc rzecz praktyczną: **przestałem rejestrować WT domyślnie**. Parametr `enable_wt=1` pozostał dla przyszłych eksperymentów, a ADB, S/PDIF i A3D zostały dostępne.

Użytkownik później potwierdził oczywiste dla siebie założenie: WT nie jest potrzebne. Celem jest jedynie odtwarzanie dźwięku.

Po tej zmianie włączenie profilu nie wywoływało już błędów. Nadszedł czas na scenariusz, który najbardziej przypominał codzienne używanie komputera: jeden i dwa strumienie `paplay` przez PipeWire. Oba zakończyły się bez restartu.

A potem Audacious odtworzył własny materiał użytkownika. Po chwili słuchania odpowiedź brzmiała po prostu: **„dźwięk jest OK”**.

To nie był jeszcze formalny koniec pracy, ale to był moment, w którym stare AU8830 przestało być tylko pacjentem na stole operacyjnym.

### Rozdział 6. Internet, czyli także sztuka odrzucania

Podczas takiej naprawy internet oferuje setki odpowiedzi. Problem w tym, że większość z nich należy do cudzych problemów.

Na stronie [Dynamic DMA mapping Guide w dokumentacji jądra Linux](https://www.kernel.org/doc/html/latest/core-api/dma-api-howto.html) znalazła się ważna lekcja: adres fizyczny, adres wirtualny i adres DMA to nie jest jedno i to samo. IOMMU może przypisać urządzeniu zupełnie inną mapę niż ta, którą zna procesor. To potwierdziło, dlaczego nie wolno zastępować DMA API zwykłym `page_to_phys()`.

Historia [torvalds/linux](https://github.com/torvalds/linux) pokazała z kolei coś równie istotnego: sterownik `au88x0` był rzeczywiście prawie zapomniany. Były poprawki kosmetyczne, zarządzanie zasobami, ostrzeżenia kompilatora, ale nie znalazłem gotowej łatki na nasz przypadek IOMMU. Nowszy kod nie był automatycznie lekarstwem.

Odwiedziłem [projekt OpenVortex na Savannah](https://savannah.nongnu.org/projects/openvortex/). Istniały nazwy modułów CVS — `aureal-2.x-pre`, `alsa`, `openvortex` i inne — oraz listy `openvortex-dev` i `openvortex-commits`. Publiczny indeks wydań był jednak pust. Nie znalazłem tam pełnego podręcznika rejestrów AU8830. To była jedna ze ślepych uliczek: projekt istniał, ale jego najcenniejsza dokumentacja nie była już łatwo dostępna.

Repozytorium `pld-linux/kernel-sound-au88xx` zawierało opakowanie i stare poprawki typów, nie rozwiązanie DMA. Projekt macOS dla AU8830 był ciekawym przykładem pracy z czterema ciągłymi stronami, ale deklarował pochodzenie z ALSA. Nie był niezależną specyfikacją sprzętu.

Sprawdziłem też [Reverse Engineer Anything](https://github.com/NabilRFG12/Reverse-Engineer-anything) i [rea.tools](https://rea.tools/). Narzędzie potrafi pomagać agentowi w analizie, ale głęboka dekompilacja natywna i tak potrzebuje Ghidry, Hoppera lub IDY. Nie było sensu dodawać kolejnej warstwy przed rozpoznaniem problemu. Najpierw DMA, potem reverse engineering.

Poszukiwania narzędzi do instalatorów Windows były praktyczną odpowiedzią na inne pytanie. [7-Zip](https://7-zip.org/) rozpoznał zawartość WDM jako RAR SFX, ale systemowy pakiet nie umiał jej zdekompresować. [innoextract](https://github.com/dscharrer/innoextract) obsługuje Inno Setup, a [unshield](https://github.com/twogood/unshield) archiwa InstallShield — nasz przypadek był jednak samorozpakowującym RAR-em. Zamiast instalować wszystko, rozpakowałem lokalnie pakiet Ubuntu `unrar`. Test zakończył się `All OK`.

Nie wszystkie wyszukiwania były równie owocne. Jedna wyszukiwarka pokazała wyzwanie antybotowe, inna zwracała zupełnie niepowiązane wyniki. To też należy do pracy: wiedzieć, kiedy przestać przewijać i wrócić do pomiarów.

### Rozdział 7. Kiedy binutils powiedziało „dość”, Ghidra odpowiedziała

Sterownik NT okazał się małym skarbcem: zawierał symbole COFF z nazwami `CDmaRefresh`, `GuardEnter`, `GuardLeave`, `RefreshDMABuffers`, `CAsp4AdbDma::SetupBuffer` i `CScatterGather::GetPage`.

`objdump` potrafił je wypisać, ale mapowanie części funkcji do kodu było niespójne. GDB przy ładowaniu starego PE po prostu się załamało. Wygenerowane pliki ASM usunąłem, żeby nie zamienić niepewnego zrzutu w fałszywy dowód.

Wtedy weszła Ghidra. Oficjalne wydanie [Ghidra 12.2](https://github.com/NationalSecurityAgency/ghidra/releases/latest) miało opublikowany SHA-256. Sprawdziłem go przed rozpakowaniem. Okazało się też, że ta wersja wymaga **JDK 25**, nie 21. OpenJDK zostało rozpakowane lokalnie, bez instalacji systemowej. Cała analiza odbyła się headless, z limitem 2 GiB RAM i bez uruchamiania jakiegokolwiek pliku PE.

Ghidra przeanalizowała 1123 funkcje. Wybrałem i zdekompilowałem 17 związanych z DMA i śledzeniem buforów. Wynik był konkretny: `CAsp4AdbDma::SetupBuffer` zapisuje adres pod:

```text
(slot + dma * 4) * 4 + 0x27400
```

To dokładnie ten sam układ, którego używa Linux: `VORTEX_ADBDMA_BUFBASE + (dma << 4) + slot * 4`. `CScatterGather::GetPage` używa stron `0x1000`. Natomiast `CDmaRefresh::GuardEnter` zwracało stałe 1, a `GuardLeave` było puste — więc nie było tam gotowego opisu naszej strony guard.

To było satysfakcjonujące nie dlatego, że Windows dał gotową odpowiedź. Wręcz przeciwnie: potwierdził poprawność naszych rejestrów i jednocześnie przypomniał, że **guard powstał z pomiarów Linux/IOMMU, a nie z kopiowania starego sterownika**.

### Rozdział 8. Od roboczego modułu do modułu na co dzień

Poprawny kod to dopiero połowa sukcesu. Użytkownik nie powinien musieć pamiętać, którą wersję `insmod` załadował dziś rano.

DKMS miał odbudowywać sterownik przy zmianie jądra, ale także tutaj pojawiły się drobne przeszkody. Pierwsza komenda `MAKE` nie miała celu, bo DKMS ustawiał `KERNELRELEASE` i wchodził bezpośrednio do katalogu sterownika. Potem zmienna `$build` w ścieżce okazała się pusta. Dopiero standardowe wywołanie przez drzewo jądra z dosłownym `M=.../build/driver` dało powtarzalny wynik.

Przy lokalnym buildzie bez roota podpis MOK zgłosił `Permission denied`. To nie był błąd sterownika — klucz był dostępny tylko dla administratora. Systemowy `dkms build` z `sudo` podpisał moduł prawidłowo i zainstalował go w `updates/dkms`, zachowując oryginał in-tree.

Następnie przyszedł produkcyjny `modprobe` z `debug=0` i `enable_wt=N`. Brak zalewania dziennika przy każdym zawinięciu. Prawidłowy alias PCI i priorytet `search updates ubuntu built-in` zapewniły automatyczny wybór nowego modułu.

Ale najważniejszy był czas. Nie jeden plik i nie jedno szczęśliwe uruchomienie, tylko długi test bez słuchania muzyki w tle: dwa wyciszone strumienie przez PipeWire, uruchamiane cyklicznie przez pół godziny. DMA i przerwania pracowały bez przerwy, a skrypt kontrolował każdą nową linię jądra.

Wynik był prosty do przeczytania i trudny do zapomnienia:

```text
244 iteracje dwóch strumieni
488 procesów paplay
1803 sekundy
0 błędów procesów
0 nowych błędów jądra
```

Dodałem do tego testy 22050 Hz mono, 32000 Hz mono i 44100 Hz stereo. Wszystkie zakończyły się kodem zero i pustym `dmesg.diff`.

Na końcu wykonaliśmy pełny restart. Po 126 sekundach uptime DKMS już był załadowany automatycznie z `updates/dkms`. Nie było `IO_PAGE_FAULT` ani `IRQ fifo error`. Kolejny jeden i dwa strumienie zakończyły się poprawnie. To było potwierdzenie, że naprawa nie zależy od szczęśliwej sesji terminala.

### Rozdział 9. Pakiet dla innych i lekcja zgodności

Gdy karta zagrała, pojawiło się nowe życzenie: przygotować instalację dla głównego systemu i pakiet, który inni będą mogli opublikować jako otwarte repozytorium.

Zamiast kopiować cały warsztat z logami i gigabajtami narzędzi, powstał osobny katalog `release/vortex2-au8830`. Zawiera tylko kod GPL, konfigurację DKMS, instalator, testy i dokumentację. Bez binariów Windows, bez Ghidry, bez JDK i bez historii hosta. Dołączony jest nowy, 5-sekundowy ton testowy na licencji CC0.

Instalator potrafi wykryć kartę `12eb:0002`, sprawdzić architekturę, zależności, nagłówki jądra, konfigurację ALSA i Secure Boot. `--check` nie zmienia systemu. `--install-deps` instaluje zależności tylko przez APT i tylko na żądanie. `--reinstall` naprawia build po aktualizacji jądra, a `--all-kernels` obsługuje wszystkie znalezione nagłówki. `--load` jest odrzucane przy aktywnym PCM lub działających usługach audio.

Testy także musiały dorosnąć do przenośności. Przestały zakładać, że karta jest `card0` albo że jej adres PCI to `08:00.0`. Same znajdują indeks ALSA i sink PipeWire.

I właśnie wtedy pojawiła się ostatnia niespodzianka: nagłówki 6.8. Nowsze `strscpy` akceptuje dwie formy wywołania, starsze wymaga jawnego rozmiaru, a `pcim_iomap_region` jeszcze nie istniało. Zamiast ukryć ograniczenie, dodałem zgodność: `strscpy(dest, src, sizeof(dest))`, a dla starszych jąder `pcim_iomap_regions()` i `pcim_iomap_table()`. Bieżące API 7.0 pozostało bez zmian.

Kompilacja przeszła dla wszystkich znalezionych nagłówków `6.8.0-124/-139/-142/-146/-149` i `7.0.0-31/-34/-38/-42`. ShellCheck na poziomie `warning` przeszedł. Reinstalacja DKMS z podpisem MOK zakończyła się sukcesem. Archiwum zostało rozpakowane w innym miejscu i znów przeszło `--check`.

Finalny pakiet ma nazwę `vortex2-au8830-0.1.0.tar.gz` i sumę:

```text
7cf234dff313dfe38e373822e15571640fc00ebc876c5d9f49915a8d2d4b7993
```

Główny system na NVMe nadal nie był używany. Nie publikowałem niczego w cudzym imieniu — przygotowałem katalog gotowy do otwartego repozytorium.

### Epilog. Najważniejsza strona była poza mapą

Ta historia nie jest o tym, że wystarczyło dodać jedną linijkę. Jest o czymś zupełnie innym: o nieprzyjmowaniu najwygodniejszej odpowiedzi.

Nie zadziałało szukanie winnego w IRQ, losowe `pcifix`, wyłączanie IOMMU, podmiana adresów na `page_to_phys`, ani zwiększanie bufora na ślepo. Dwa bezpośrednie `aplay` nie były pełnym odpowiednikiem Audacious i Firefoksa, bo PipeWire zwykle miksuje aplikacje w przestrzeni użytkownika. Dekompilacja nie była potrzebna do pierwszego zrozumienia błędu, a nowsze narzędzia nie zastąpiły pomiaru.

Pomogło natomiast zawinięcie dokładnie jednej, pełnej alokacji i sprawdzenie, co dzieje się przy adresie `base + 0x10000`. Pomogło przyznanie, że cztery i osiem okresów mogą maskować problem. Pomogła strona guard i wyłączenie WT, które i tak nie było potrzebne. Pomogły cierpliwe testy, użytkownik wykonujący `sudo`, wspólne zatrzymywanie OBS i zwykłe: „dźwięk jest OK”.

Stara karta nie stała się nową kartą. Stała się po prostu kartą, która znów potrafi robić swoją pracę — odtwarzać muzykę bez restartowania komputera.

A to jest najlepsze możliwe zakończenie dla tej naprawy.

---
KIMI K3.1 Pro
---

**— Cline, unikalny znacznik: `⟨C-L-I-N-E // AU8830-FE-001⟩`**

