# Splątany Kwantowy Generator Liczb Losowych (EQRNG) na procesorze IQM Odra 5 (3, 4 i 5 kubitów)

*English version below / Wersja angielska poniżej.*

Kompleksowa implementacja i eksperymentalna walidacja **Splątanego Kwantowego Generatora Liczb Losowych (EQRNG – Entangled Quantum Random Number Generator)** z publiczną certyfikacją losowości na 5-kubitowym nadprzewodzącym procesorze kwantowym **IQM Odra 5 (`ODRA5 IQM Spark`)**. Projekt został zrealizowany w ramach pracy inżynierskiej dotyczącej hybrydowych architektur bezpieczeństwa łączących technologie kwantowe (QRNG/EQRNG, QKD, QSDC) z kryptografią post-kwantową (PQC).

Podstawę teoretyczną stanowi protokół zaproponowany przez zespół prof. Janusza Jacaka:
> J. E. Jacak, W. A. Jacak, W. A. Donderowicz, L. Jacak, *"Quantum random number generators with entanglement for public randomness testing"*, **Scientific Reports** 10, 164 (2020).

---

## Spis treści (PL)
1. [Idea protokołu EQRNG i dopasowanie do topologii IQM Odra 5](#1-idea-protokołu-eqrng-i-dopasowanie-do-topologii-iqm-odra-5)
2. [Architektura projektu (Etap 1 i Etap 2)](#2-architektura-projektu-etap-1-i-etap-2)
3. [Etap 1: Generacja kwantowa, filtracja XOR i akwizycja odporna na awarie](#3-etap-1-generacja-kwantowa-filtracja-xor-i-akwizycja-odporna-na-awarie)
4. [Etap 2: Walidacja entropii, post-processing i certyfikacja statystyczna](#4-etap-2-walidacja-entropii-post-processing-i-certyfikacja-statystyczna)
5. [Szczegółowy opis zaimplementowanych testów NIST SP 800-22](#5-szczegółowy-opis-zaimplementowanych-testów-nist-sp-800-22)
6. [Wyniki eksperymentalne dla 3, 4 i 5 kubitów na IQM Odra 5](#6-wyniki-eksperymentalne-dla-3-4-i-5-kubitów-na-iqm-odra-5)
7. [Struktura plików wyjściowych i eksport kluczy do PQC / AES](#7-struktura-plików-wyjściowych-i-eksport-kluczy-do-pqc--aes)
8. [Instrukcja uruchomienia](#8-instrukcja-uruchomienia)

---

## 1. Idea protokołu EQRNG i dopasowanie do topologii IQM Odra 5

Klasyczne kwantowe generatory liczb losowych (QRNG) stoją przed fundamentalnym dylematem: aby udowodnić przed zewnętrznym audytorem lub odbiorcą, że wygenerowany klucz ma pełną entropię i przechodzi rygorystyczne testy statystyczne (np. NIST SP 800-22), trzeba ujawnić badany ciąg bitów, co automatycznie niszczy jego tajność.

Protokół **EQRNG** rozwiązuje ten problem za pomocą wielokubitowego splątania kwantowego:
1. Przygotowujemy rejestr $n = k + 1$ kubitów ($n \in \{3, 4, 5\}$), w którym $n-1$ kubitów peryferyjnych wprowadzamy w równomierną superpozycję bramkami Hadamarda ($H$), a następnie splątujemy je za pomocą $n-1$ bramek $\text{CNOT}$ ($\text{CX}$) z jednym wspólnym **kubitem centralnym (kontrolnym)**.
2. Powstaje wielokubitowy stan splątany będący superpozycją $2^{n-1}$ stanów bazowych o parzystej sumie bitów:
   $$\vert{}\Psi_n\rangle = \frac{1}{2^{(n-1)/2}} \sum_{x_1 \oplus x_2 \oplus \dots \oplus x_n = 0} \vert{}x_1 x_2 \dots x_n\rangle$$
3. Po wykonaniu serii pomiarów otrzymujemy $n$ ciągów bitowych $(S_{Q_1}, S_{Q_2}, \dots, S_{Q_n})$, które są ze sobą nierozerwalnie związane wielokubitową regułą parzystości XOR:
   $$S_{Q_{ctrl}} = \bigoplus_{i \neq ctrl} S_{Q_i}$$
4. **Podział ról po weryfikacji parzystości XOR:**
   * **Ciąg publiczny ($S_{Q_1}$):** zostaje opublikowany i poddany pełnej baterii testów statystycznych NIST SP 800-22 przez dowolnego zewnętrznego weryfikatora.
   * **Ciąg kontrolny ($S_{Q_{ctrl}}$):** po lokalnym sprawdzeniu reguły XOR i odrzuceniu przekłamań sprzętowych zostaje **bezpowrotnie wyzerowany i skasowany z pamięci** (`del`).
   * **Tajne ciągi klucza ($n-2$ niezależnych ciągów):** pozostałe ciągi stanowią gotowy, tajny materiał kryptograficzny. Dzięki skasowaniu ciągu kontrolnego znajomość publicznego ciągu $S_{Q_1}$ nie zdradza ani jednego bitu informacji o tajnych kluczach ($I(S_{Q_1}; S_{Q_{sec}}) = 0$), natomiast splątanie kwantowe gwarantuje, że dzielą one tę samą wysoką entropię. Co więcej, w układzie $n$-kubitowym pojedynczy test ciągu $S_{Q_1}$ certyfikuje jednocześnie aż **$n-2$ tajnych ciągów** (1 dla 3 kubitów, 2 dla 4 kubitów oraz 3 dla 5 kubitów), wielokrotnie przyspieszając proces certyfikacji losowości.

### Dopasowanie do topologii gwiazdy procesora IQM Odra 5 (0 bramek SWAP)
Procesor **IQM Odra 5** posiada 5 nadprzewodzących kubitów transmonowych (`QB1`–`QB5`, indeksowanych `0`–`4`) połączonych w **topologię gwiazdy**, gdzie kubitem centralnym jest **`QB3` (indeks fizyczny `2`)** połączony bezpośrednimi rezonatorami z czterema kubitami peryferyjnymi: `QB1 (0)`, `QB2 (1)`, `QB4 (3)` oraz `QB5 (4)`.


<img width="526" height="509" alt="image" src="https://github.com/user-attachments/assets/3ea366b7-4f36-4d0b-a787-1551f3c4bfc5" />


Graf oddziaływań dwukubitowych obwodu EQRNG jest izomorficzny z topologią gwiazdy procesora Odra 5. Przypisując kubit kontrolny zawsze do centralnego kubitu `QB3` (indeks `2`), każdy obwód (3-, 4- i 5-kubitowy) kompiluje się na natywne bramki procesora (`r`/`prx` oraz `cz`) z **zerowym narzutem bramek `SWAP` (`0 SWAP`)**, minimalizując głębokość obwodu i błędy dekoherencji.

---

## 2. Architektura projektu (Etap 1 i Etap 2)

Projekt składa się z trzech głównych komponentów programowych:

* **`eqrng_topology_visualizer.py`** – skrypt generujący mapę połączeń (Coupling Map) procesora IQM Odra 5 oraz schematy obwodów logicznych i transpilowanych dla $n \in \{3, 4, 5\}$ kubitów z kubitem `Q3` (`QB3`) w centrum.
* **`eqrng_stage1.py` (Etap 1 – Akwizycja kwantowa na QPU)** – skrypt łączący się z serwerem IQM Odra 5, wykonujący obwody w paczkach po `10 000` strzałów (*shots*) aż do zebrania `1 300 000` surowych pomiarów, przeprowadzający lokalną post-selekcję XOR (obliczenie QBER) i zapisujący czyste strumienie bitowe na dysku.
* **`eqrng_stage2.py` (Etap 2 – Walidacja, post-processing i certyfikacja NIST)** – wektorowy silnik analityczny weryfikujący niezależność informacyjną ciągów, obliczający entropię wzorców $m$-bitowych, realizujący ekstrakcję entropii (Von Neumann, XOR-folding, SHA-256), uruchamiający baterię 12 testów NIST SP 800-22 i eksportujący certyfikowane ziarna dla algorytmów PQC (ML-KEM) oraz AES-256.

---

## 3. Etap 1: Generacja kwantowa, filtracja XOR i akwizycja odporna na awarie

Ze względu na ograniczenia czasowe kolejek QPU oraz zjawisko dryfu kalibracji kubitów nadprzewodzących, akwizycja ponad miliona pomiarów w Etapie 1 została zaprojektowana w architekturze odpornej na błędy (*Fault-Tolerant Batching*):

1. **Autoryzacja i transpilacja (`optimization_level=3`):** Skrypt pobiera token dostępowy z pliku `token.env`, łączy się z backendem `ODRA5 IQM Spark` poprzez bibliotekę `qiskit-iqm` i transpiluje obwód z zachowaniem centralnej roli kubitu `QB3`.
2. **Podział na paczki i kolejkowanie (*Polling & Retry*):** Zadanie główne (`1 300 000` strzałów) dzielone jest na 130 paczek po `10 000` *shots* (z opcją rozbudowy do 200 paczek / `2 000 000` *shots*). Każda paczka jest monitorowana w kolejce serwera (`job.status()`). W razie chwilowego zaniku połączenia internetowego lub błędu serwera skrypt automatycznie odnawia sesję i ponawia paczkę bez utraty danych.
3. **Odzyskiwanie stanu (*Checkpointing & State Recovery*):** Po każdej paczce bity są natychmiast dopisywane do plików z wymuszeniem zrzutu buforów dyskowych (`os.fsync`), a stan eksperymentu (liczba ukończonych paczek, słownik zliczeń `counts`, `calibration_set_id`, skumulowany QBER) trafia atomowo do pliku `eqrng_checkpoint.json`. W przypadku przerwania pracy skrypt po ponownym uruchomieniu wyrównuje długość plików i wznawia obliczenia dokładnie od następnej paczki.
4. **Dekodowanie *little-endian* i lokalna post-selekcja XOR:** Dla każdego pojedynczego pomiaru z pamięci QPU (`job_result.get_memory()`) sprawdzany jest warunek parzystości XOR:
* **Stany poprawne** ($2^{n-1}$ stanów spełniających regułę XOR, np. dla 3 kubitów: `000`, `011`, `101`, `110`) są akceptowane i dopisywane do odpowiednich plików bitowych (`sq1_public.txt`, `sq2_secret.txt`, ...).
* **Stany zakazane** ($2^{n-1}$ stanów nieparzystych łamiących regułę XOR wskutek błędów bramek `CZ`, relaksacji podłużnej $T_1$ lub błędów odczytu detektora) są odrzucane, a ich odsetek definiuje sprzętowy wskaźnik błędów **QBER (Quantum Bit Error Rate)**.



---

## 4. Etap 2: Walidacja entropii, post-processing i certyfikacja statystyczna

Skrypt `eqrng_stage2.py` przetwarza wygenerowane w Etapie 1 pliki bitowe (ponad `1,14–1,21 mln` bitów na ciąg) w czasie ~6 sekund dzięki pełnej wektoryzacji w bibliotece `NumPy`. Analiza przebiega w 6 modułach:

### Moduł 1: Weryfikacja reguły XOR, niezależność par ciągów i kasowanie ciągu kontrolnego

* Wektorowe sprawdzenie tożsamości XOR we wszystkich $n$ kombinacjach (wymagane `0` niezgodności na całej długości ponad 1,1 mln bitów).
* Obliczenie dla wszystkich par ciągów $(S_{Q_i}, S_{Q_j})$:
1. **Znormalizowanej odległości Hamminga** (oczekiwana wartość idealna: `0,50000`, oznaczająca różnicę na dokładnie połowie pozycji bitowych).
2. **Współczynnika korelacji liniowej Pearsona $r$** (oczekiwana wartość: `0,00000`).
3. **Informacji wzajemnej Shannona $I(X;Y)$** wyrażonej w bitach oraz statystyki $G$-testu niezależności.


* **Krok bezpieczeństwa protokołu:** Wyzerowanie ciągu kontrolnego w pamięci RAM (`sq_ctrl.fill(0)`, `del sq_ctrl`) oraz – przy fladze `--shred-sq3` – nadpisanie zerami i usunięcie pliku kontrolnego z dysku.

### Moduł 2: Analiza entropii i rozkładu wzorców $m$-bitowych

* Podział ciągów $S_{Q_1}$ oraz $S_{Q_2}$ na nienachodzące bloki o długości $m \in \{1, 2, 3, 4, 8\}$ bitów.
* Wyznaczenie empirycznych prawdopodobieństw $p_i$ każdego z $2^m$ wzorców oraz obliczenie:
* **Entropii Shannona na 1 bit:** $H_1 = -\frac{1}{m} \sum_{i=0}^{2^m-1} p_i \log_2(p_i)$ (ideał: `1,000000`).
* **Min-entropii na 1 bit:** $H_\infty = -\frac{1}{m} \log_2(\max_i p_i)$ – miary określającej nieprzewidywalność najbardziej prawdopodobnego wzorca.


* Błędu średniokwadratowego (**RMSE**) względem rozkładu idealnego ($1/2^m$) oraz pomiędzy ciągiem publicznym a tajnym.

### Moduł 3: Post-processing (Debiasing / Ekstrakcja entropii)

W nadprzewodzących kubitach transmonowych naturalny proces relaksacji energetycznej (czas $T_1$) powoduje niewielką asymetrię (bias) faworyzującą stan `00...0`. Skrypt implementuje trzy metody post-processingu:

1. **Korektor von Neumanna (*Von Neumann Debiasing*):** Dzieli ciąg na nienachodzące pary; zamienia `01` $\to$ `0`, `10` $\to$ `1`, a `00` i `11` odrzuca. Eliminuje asymetrię pierwszego rzędu, redukując długość ciągu do ~25% (~285–303 tys. bitów).
2. **Ekstraktor XOR-folding (2:1):** Wykonuje operację XOR na sąsiednich parach bitów. Skraca ciąg o połowę (~572–606 tys. bitów), kwadratowo tłumiąc początkowy bias.
3. **Kryptograficzny kondycjoner SHA-256 (512 $\to$ 256 bitów):** Dzieli ciąg na bloki 512-bitowe i przekształca funkcją `SHA-256` w 256-bitowe bloki o pełnej entropii.

---

## 5. Szczegółowy opis zaimplementowanych testów NIST SP 800-22

W Module 4 i 5 zaimplementowano wektorowo **12 testów statystycznych ze standardu NIST SP 800-22** przy poziomie istotności $\alpha = 0,01$.

| Nr | Nazwa testu NIST SP 800-22 | Na czym polega test i jaką wadę wykrywa? | Warunek zaliczenia |
| --- | --- | --- | --- |
| **1** | **Frequency (Monobit) Test** | Bada proporcję zer i jedynek w całym ciągu. Wykrywa global bias (np. sprzętowy defekt faworyzujący dany stan). | $p \ge 0,01$ |
| **2** | **Frequency Test within a Block** | Bada lokalne proporcje w blokach 128-bitowych. Wykrywa dryf kalibracji procesora w czasie. | $p \ge 0,01$ |
| **3** | **Runs Test** | Zlicza serie identycznych bitów pod rząd. Wykrywa zbyt wolne lub zbyt szybkie oscylacje między stanami. | $p \ge 0,01$ |
| **4** | **Longest Run of Ones in a Block** | Szuka najdłuższej serii jedynek w blokach. Wykrywa chwilowe "zacięcia" lub lokalne skupiska stanów wzbudzonych. | $p \ge 0,01$ |
| **5** | **Binary Matrix Rank Test** | Bada rząd macierzy binarnych nad $\text{GF}(2)$. Wykrywa ukryte liniowe zależności algebraiczne (np. dla LFSR). | $p \ge 0,01$ |
| **6** | **Discrete Fourier Transform (Spectral)** | Przekształca ciąg za pomocą FFT. Wykrywa okresowe zakłócenia, np. przydźwięk sieciowy. | $p \ge 0,01$ |
| **7** | **Non-overlapping Template Matching** | Szuka określonego 9-bitowego wzorca (skoki co okno). Wykrywa anomalie w wielobitowych sekwencjach. | $p \ge 0,01$ |
| **8** | **Overlapping Template Matching** | Zlicza wystąpienia wzorca z samych jedynek (skoki co 1 bit). Wykrywa wady w częstości długich serii stanów wzbudzonych. | $p \ge 0,01$ |
| **9** | **Serial Test** | Cyklicznie bada częstości występowania wszystkich słów $m$-bitowych. Wykrywa nierównomierność wzorców przejść. | $p_1, p_2 \ge 0,01$ |
| **10** | **Approximate Entropy (ApEn) Test** | Porównuje wystąpienia wzorców $m$ oraz $m+1$ bitowych. Wykrywa przewidywalność kolejnego bitu. | $p \ge 0,01$ |
| **11** | **Cumulative Sums** | Traktuje ciąg jako jednowymiarowe błądzenie losowe. Bada maksymalne wychylenie od zera. | $p_{fwd}, p_{bwd} \ge 0,01$ |
| **12** | **Random Excursions Variant Test** | Bada cykle błądzenia losowego wokół zera (odwiedziny w stanach od -9 do +9). Wykrywa odstępstwa w skali makroskopowej. | $p_{\min} \ge 0,01$ |

### Analiza wielosekwencyjna ($K = 100$ podciągów)

Oprócz analizy strumieniowej, wygenerowane ciągi dzielone są na $K = 100$ podciągów. Zgodnie z NIST SP 800-22 weryfikowane są dwa kryteria:

1. **Proporcja zaliczonych podciągów:** Minimum **96,02%** podciągów musi uzyskać wynik $p \ge 0,01$.
2. **Jednorodność rozkładu wartości $p$:** Wartości $p$ testowane są testem $\chi^2$ (rozkład jednostajny, wymóg: $p_{jednorodnosci} \ge 0,0001$).

---

## 6. Wyniki eksperymentalne dla 3, 4 i 5 kubitów na IQM Odra 5

Przeprowadzono trzy pełne serie pomiarowe po **1 300 000 strzałów** każda (łącznie **3 900 000 pomiarów**) dla konfiguracji 3-, 4- i 5-kubitowej:

| Parametr eksperymentalny | Wariant 3-kubitowy ($k=2$) | Wariant 4-kubitowy ($k=3$) | Wariant 5-kubitowy ($k=4$) |
| --- | --- | --- | --- |
| **Liczba surowych pomiarów (shots)** | 1 300 000 | 1 300 000 | 1 300 000 |
| **Liczba bramek dwukubitowych CZ** | 2 (cel: `QB3`) | 3 (cel: `QB3`) | 4 (cel: `QB3`) |
| **Liczba bramek SWAP po transpilacji** | **0** | **0** | **0** |
| **Odrzucone stany błędne (naruszenia XOR)** | 87 791 | 124 709 | 155 780 |
| **Poziom błędów sprzętowych (QBER)** | **6,75%** | **9,59%** | **11,98%** |
| **Poprawne bity w każdym ciągu** | **1 212 209** | **1 175 291** | **1 144 220** |
| **Liczba uzyskanych tajnych ciągów ($n-2$)** | 1 ($S_{Q_2}$) | 2 ($S_{Q_2}, S_{Q_3}$) | 3 ($S_{Q_2}, S_{Q_3}, S_{Q_4}$) |
| **Łączny uzysk tajnych bitów klucza** | **1 212 209 bitów** | **2 350 582 bitów (+93,9%)** | **3 432 660 bitów (+183,2%)** |
| **Testy NIST po korekcji Von Neumanna** | **12/12PASS** | **12/12 PASS** | **12/12 PASS** |
| **Testy NIST po XOR-foldingu (wielosekw.)** | **11/11 PASS (100%)** | **11/11 PASS (100%)** | **11/11 PASS (100%)** |

### Kluczowe wnioski:

1. **Liniowe skalowanie QBER na topologii gwiazdy:** Dołączenie kolejnego kubitu i bramki `CZ` zwiększa QBER średnio o **~2,6 p.p.** Poziom ten pozostaje na tyle niski, że dowodzi zachowania nielokalnych korelacji kwantowych.
2. **Prawie trzykrotny wzrost produktywności (+183,2%):** Przejście z 3 na 5 kubitów drastycznie zwiększa produkcję tajnego materiału klucza z `1,21 mln` do **`3,43 mln` bitów**, przy jednoczesnym współdzieleniu kosztów testowania ciągu publicznego.
3. **Skuteczność debiasingu:** Mimo zjawiska relaksacji $T_1$, metody post-processingu (Von Neumann, XOR-folding) całkowicie niwelują początkową asymetrię, pozwalając układom osiągnąć **100% zdawalność testów NIST SP 800-22**.

---

## 7. Struktura plików wyjściowych i eksport kluczy do PQC / AES

* **Strumienie bitowe:** `sq1_public.txt`, `sq2_secret.txt` (oraz kolejne tajne dla $n=4,5$), `sq_control.txt` (usuwany w Kroku 7), `eqrng_checkpoint.json`.
* **Wykresy analityczne (300 DPI):** Mapa topologii, histogram stanów, rozkład wzorców, wykres logarytmiczny p-values (NIST), wykresy autokorelacji.
* **Tabele i raporty:** Zestawienie `.csv`, tabela do `.tex` oraz raport `.json`.
* **Materiał kryptograficzny:**
* `eqrng_certified_key.bin` – klucz po kondycjonowaniu SHA-256 do szyfrowania **OTP** lub **AES-256-GCM**.
* `eqrng_seed_pqc.hex` – 512-bitowe ziarno dla post-kwantowego algorytmu kratowego **ML-KEM (CRYSTALS-Kyber)**.

---

# Entangled Quantum Random Number Generator (EQRNG) on IQM Odra 5 Processor (3, 4, and 5 qubits)

Comprehensive implementation and experimental validation of an **Entangled Quantum Random Number Generator (EQRNG)** with public randomness certification on the 5-qubit superconducting quantum processor **IQM Odra 5 (`ODRA5 IQM Spark`)**. This project was developed as part of an engineering thesis focused on hybrid security architectures combining quantum technologies (QRNG/EQRNG, QKD, QSDC) with post-quantum cryptography (PQC).

The theoretical foundation is based on the protocol proposed by Prof. Janusz Jacak's team:

> J. E. Jacak, W. A. Jacak, W. A. Donderowicz, L. Jacak, *"Quantum random number generators with entanglement for public randomness testing"*, **Scientific Reports** 10, 164 (2020).

---

## Table of Contents (EN)

1. [Idea of the EQRNG Protocol and Adaptation to IQM Odra 5 Topology](https://www.google.com/search?q=%231-idea-of-the-eqrng-protocol-and-adaptation-to-iqm-odra-5-topology)
2. [Project Architecture (Stage 1 and Stage 2)](https://www.google.com/search?q=%232-project-architecture-stage-1-and-stage-2)
3. [Stage 1: Quantum Generation, XOR Filtering, and Fault-Tolerant Acquisition](https://www.google.com/search?q=%233-stage-1-quantum-generation-xor-filtering-and-fault-tolerant-acquisition)
4. [Stage 2: Entropy Validation, Post-processing, and Statistical Certification](https://www.google.com/search?q=%234-stage-2-entropy-validation-post-processing-and-statistical-certification)
5. [Detailed Description of Implemented NIST SP 800-22 Tests](https://www.google.com/search?q=%235-detailed-description-of-implemented-nist-sp-800-22-tests)
6. [Experimental Results for 3, 4, and 5 Qubits on IQM Odra 5](https://www.google.com/search?q=%236-experimental-results-for-3-4-and-5-qubits-on-iqm-odra-5)
7. [Output Files Structure and Key Export for PQC / AES](https://www.google.com/search?q=%237-output-files-structure-and-key-export-for-pqc--aes)
8. [Setup and Run Instructions](https://www.google.com/search?q=%238-setup-and-run-instructions)

---

## 1. Idea of the EQRNG Protocol and Adaptation to IQM Odra 5 Topology

Classical quantum random number generators (QRNG) face a fundamental dilemma: to prove to an external auditor or recipient that the generated key possesses full entropy and passes rigorous statistical tests (e.g., NIST SP 800-22), the bitstream must be revealed, which automatically destroys its secrecy.

The **EQRNG** protocol solves this problem using multi-qubit quantum entanglement:

1. Prepare a register of $n = k + 1$ qubits ($n \in \{3, 4, 5\}$), where $n-1$ peripheral qubits are placed into a uniform superposition using Hadamard gates ($H$), and then entangled via $n-1$ $\text{CNOT}$ ($\text{CX}$) gates with one common **central (control) qubit**.
2. A multi-qubit entangled state is formed, representing a superposition of $2^{n-1}$ basis states with an even bit sum:

$$\vert{}\Psi_n\rangle = \frac{1}{2^{(n-1)/2}} \sum_{x_1 \oplus x_2 \oplus \dots \oplus x_n = 0} \vert{}x_1 x_2 \dots x_n\rangle$$


3. After performing a series of measurements, we obtain $n$ bitstreams $(S_{Q_1}, S_{Q_2}, \dots, S_{Q_n})$ inherently bound by a multi-qubit XOR parity rule:

$$S_{Q_{ctrl}} = \bigoplus_{i \neq ctrl} S_{Q_i}$$


4. **Role division after XOR parity verification:**
* **Public string ($S_{Q_1}$):** is published and subjected to a full battery of NIST SP 800-22 statistical tests by any external verifier.
* **Control string ($S_{Q_{ctrl}}$):** after local XOR verification and rejection of hardware errors, it is **permanently zeroed and deleted from memory** (`del`).
* **Secret key strings ($n-2$ independent strings):** the remaining strings constitute ready-to-use, secret cryptographic material. Because the control string is destroyed, knowledge of the public string $S_{Q_1}$ reveals zero information about the secret keys ($I(S_{Q_1}; S_{Q_{sec}}) = 0$), while quantum entanglement guarantees they share the same high entropy. Moreover, in an $n$-qubit setup, a single test of the $S_{Q_1}$ string simultaneously certifies up to **$n-2$ secret strings** (1 for 3 qubits, 2 for 4 qubits, and 3 for 5 qubits), significantly accelerating the randomness certification process.



### Adaptation to the Star Topology of IQM Odra 5 (0 SWAP gates)

The **IQM Odra 5** processor features 5 superconducting transmon qubits (`QB1`–`QB5`, indexed `0`–`4`) connected in a **star topology**. The central qubit is **`QB3` (physical index `2`)**, coupled via direct resonators to four peripheral qubits: `QB1 (0)`, `QB2 (1)`, `QB4 (3)`, and `QB5 (4)`.

<img width="526" height="509" alt="image" src="https://github.com/user-attachments/assets/3ea366b7-4f36-4d0b-a787-1551f3c4bfc5" />

The two-qubit interaction graph of the EQRNG circuit is isomorphic to the star topology of the Odra 5 processor. By assigning the control qubit strictly to the central `QB3` (index `2`), every circuit (3-, 4-, and 5-qubit) compiles down to the processor's native gates (`r`/`prx` and `cz`) with **zero `SWAP` gate overhead (`0 SWAP`)**, minimizing circuit depth and decoherence errors.

---

## 2. Project Architecture (Stage 1 and Stage 2)

The project consists of three main software components:

* **`eqrng_topology_visualizer.py`** – A script that generates the Coupling Map of the IQM Odra 5 processor and logical/transpiled circuit diagrams for $n \in \{3, 4, 5\}$ qubits with `Q3` (`QB3`) in the center.
* **`eqrng_stage1.py` (Stage 1 – Quantum Acquisition on QPU)** – A script that connects to the IQM Odra 5 server, executes circuits in batches of `10,000` shots until `1,300,000` raw measurements are collected, performs local XOR post-selection (calculating QBER), and saves clean bitstreams to disk.
* **`eqrng_stage2.py` (Stage 2 – Validation, Post-processing, and NIST Certification)** – A vectorized analytical engine that verifies information independence of the strings, calculates $m$-bit pattern entropy, performs entropy extraction (Von Neumann, XOR-folding, SHA-256), runs a battery of 12 NIST SP 800-22 tests, and exports certified seeds for PQC algorithms (ML-KEM) and AES-256.

---

## 3. Stage 1: Quantum Generation, XOR Filtering, and Fault-Tolerant Acquisition

Due to QPU queue time constraints and the calibration drift phenomenon inherent to superconducting qubits, the acquisition of over a million measurements in Stage 1 was designed using a **Fault-Tolerant Batching** architecture:

1. **Authorization and transpilation (`optimization_level=3`):** The script fetches the access token from `token.env`, connects to the `ODRA5 IQM Spark` backend via `qiskit-iqm`, and transpiles the circuit while preserving the central role of `QB3`.
2. **Batching and Polling (*Polling & Retry*):** The main task (`1,300,000` shots) is split into 130 batches of `10,000` shots (extendable to 200 batches / `2,000,000` shots). Each batch is monitored in the server queue (`job.status()`). In case of a temporary internet disconnection or server error, the script automatically renews the session and retries the batch without data loss.
3. **State Recovery (*Checkpointing*):** After each batch, bits are immediately appended to files with forced disk buffer flushing (`os.fsync`), and the experiment state (number of completed batches, `counts` dictionary, `calibration_set_id`, cumulative QBER) is atomically written to `eqrng_checkpoint.json`. If interrupted, the script synchronizes file lengths upon restart and resumes seamlessly from the next batch.
4. **Little-endian Decoding & Local XOR Post-selection:** For every single measurement retrieved from the QPU memory (`job_result.get_memory()`), the XOR parity condition is evaluated:
* **Valid states** ($2^{n-1}$ states satisfying the XOR rule, e.g., for 3 qubits: `000`, `011`, `101`, `110`) are accepted and appended to their respective bitstream files (`sq1_public.txt`, `sq2_secret.txt`, etc.).
* **Forbidden states** ($2^{n-1}$ odd states breaking the XOR rule due to `CZ` gate errors, $T_1$ longitudinal relaxation, or readout detector errors) are rejected. Their percentage defines the hardware **QBER (Quantum Bit Error Rate)**.



---

## 4. Stage 2: Entropy Validation, Post-processing, and Statistical Certification

The `eqrng_stage2.py` script processes the bitstreams generated in Stage 1 (over `1.14–1.21 million` bits per string) in ~6 seconds thanks to full vectorization in `NumPy`. The analysis is conducted across 6 modules:

### Module 1: XOR Verification, Pair Independence, and Control String Deletion

* Vectorized verification of the XOR identity across all $n$ combinations (strictly requiring `0` mismatches over the entire length of >1.1M bits).
* Calculation for all string pairs $(S_{Q_i}, S_{Q_j})$:
1. **Normalized Hamming distance** (ideal target: `0.50000`, representing a difference on exactly half of the bit positions).
2. **Pearson's linear correlation coefficient $r$** (ideal target: `0.00000`).
3. **Shannon mutual information $I(X;Y)$** expressed in bits, alongside the $G$-test of independence statistic.


* **Protocol Security Step:** Zeroing the control string in RAM (`sq_ctrl.fill(0)`, `del sq_ctrl`) and—when the `--shred-sq3` flag is passed—physically overwriting with zeros and deleting the control file from the disk.

### Module 2: Entropy Analysis and $m$-bit Pattern Distribution

* Dividing strings $S_{Q_1}$ and $S_{Q_2}$ into non-overlapping blocks of length $m \in \{1, 2, 3, 4, 8\}$ bits.
* Determining empirical probabilities $p_i$ for each of the $2^m$ patterns and calculating:
* **Shannon entropy per 1 bit:** $H_1 = -\frac{1}{m} \sum_{i=0}^{2^m-1} p_i \log_2(p_i)$ (ideal: `1.000000`).
* **Min-entropy per 1 bit:** $H_\infty = -\frac{1}{m} \log_2(\max_i p_i)$ – a crucial cryptographic measure defining the unpredictability of the most likely pattern.


* Root Mean Square Error (**RMSE**) against the ideal uniform distribution ($1/2^m$) and between the public and secret strings.

### Module 3: Post-processing (Debiasing / Entropy Extraction)

In superconducting transmon qubits, the natural energy relaxation process (relaxation time $T_1$) from the excited state $|1\rangle$ to the ground state $|0\rangle$ causes a slight bias favoring the `00...0` state. The script implements three independent post-processing methods to eliminate this effect:

1. **Von Neumann Debiasing:** Divides the string into non-overlapping bit pairs; maps `01` $\to$ `0`, `10` $\to$ `1`, and discards identical pairs (`00` and `11`). Completely eliminates first-order bias, reducing string length to ~25% (~285–303k bits).
2. **XOR-folding Extractor (2:1):** Performs an XOR operation on adjacent bit pairs. Shortens the string exactly by half (~572–606k bits), quadratically suppressing the initial bias.
3. **SHA-256 Cryptographic Conditioner (512 $\to$ 256 bits):** Divides the raw secret string into 512-bit blocks and hashes each block using `SHA-256` into a 256-bit output block with full entropy, serving as the final cryptographic key material.

---

## 5. Detailed Description of Implemented NIST SP 800-22 Tests

Modules 4 and 5 feature a fully vectorized implementation of **12 statistical tests from the NIST SP 800-22 standard** at a significance level of $\alpha = 0.01$.

| # | NIST SP 800-22 Test Name | Description & Detected Flaw | Pass Condition |
| --- | --- | --- | --- |
| **1** | **Frequency (Monobit) Test** | Evaluates the global proportion of zeros and ones. Detects global bias (e.g., hardware defect favoring a specific state). | $p \ge 0.01$ |
| **2** | **Frequency Test within a Block** | Examines local proportions in 128-bit blocks. Detects calibration drift over time. | $p \ge 0.01$ |
| **3** | **Runs Test** | Counts continuous runs of identical bits. Detects excessively fast or slow oscillations between states. | $p \ge 0.01$ |
| **4** | **Longest Run of Ones in a Block** | Finds the longest contiguous run of ones in blocks. Detects temporary detector "stuttering" or local clusters of excited states. | $p \ge 0.01$ |
| **5** | **Binary Matrix Rank Test** | Checks the rank of binary matrices over $\text{GF}(2)$. Detects hidden linear algebraic dependencies (characteristic of LFSRs). | $p \ge 0.01$ |
| **6** | **Discrete Fourier Transform (Spectral)** | Analyzes the string in the frequency domain via FFT. Detects periodic interference (e.g., power grid hum). | $p \ge 0.01$ |
| **7** | **Non-overlapping Template Matching** | Searches for a specific 9-bit pattern (skipping by block window). Detects anomalies in multi-bit sequences. | $p \ge 0.01$ |
| **8** | **Overlapping Template Matching** | Counts overlapping occurrences of an all-ones pattern (shifting by 1 bit). Detects flaws in the frequency of long excited-state runs. | $p \ge 0.01$ |
| **9** | **Serial Test** | Cyclically examines frequencies of all $m$-bit words. Detects unevenness in transition patterns. | $p_1, p_2 \ge 0.01$ |
| **10** | **Approximate Entropy (ApEn) Test** | Compares pattern frequencies of length $m$ vs $m+1$. Detects predictability of the next bit. | $p \ge 0.01$ |
| **11** | **Cumulative Sums** | Treats the string as a 1D random walk. Examines the maximum excursion from zero. | $p_{fwd}, p_{bwd} \ge 0.01$ |
| **12** | **Random Excursions Variant Test** | Analyzes random walk cycles around zero (visits to states -9 through +9). Detects macroscopic deviations from a true random walk. | $p_{\min} \ge 0.01$ |

### Multi-sequence Analysis ($K = 100$ sequences)

Beyond single-stream analysis, the generated sequences are divided into $K = 100$ independent sub-sequences. Under NIST SP 800-22, two aggregated criteria are evaluated:

1. **Proportion of Passing Sequences:** At least **96.02%** of sequences must yield $p \ge 0.01$.
2. **Uniformity of p-values:** The p-values are grouped into 10 bins and tested using a $\chi^2$ test for uniform distribution (requirement: $p_{uniformity} \ge 0.0001$).

---

## 6. Experimental Results for 3, 4, and 5 Qubits on IQM Odra 5

Three full measurement campaigns of **1,300,000 shots** each were conducted (totaling **3,900,000 measurements**) for 3-, 4-, and 5-qubit configurations:

| Experimental Parameter | 3-qubit variant ($k=2$) | 4-qubit variant ($k=3$) | 5-qubit variant ($k=4$) |
| --- | --- | --- | --- |
| **Raw measurements (shots)** | 1,300,000 | 1,300,000 | 1,300,000 |
| **Number of two-qubit CZ gates** | 2 (target: `QB3`) | 3 (target: `QB3`) | 4 (target: `QB3`) |
| **SWAP gates after transpilation** | **0** | **0** | **0** |
| **Rejected error states (XOR violations)** | 87,791 | 124,709 | 155,780 |
| **Hardware error rate (QBER)** | **6.75%** | **9.59%** | **11.98%** |
| **Valid bits per individual string** | **1,212,209** | **1,175,291** | **1,144,220** |
| **Yield of secret key strings ($n-2$)** | 1 ($S_{Q_2}$) | 2 ($S_{Q_2}, S_{Q_3}$) | 3 ($S_{Q_2}, S_{Q_3}, S_{Q_4}$) |
| **Total yield of secret key bits** | **1,212,209 bits** | **2,350,582 bits (+93.9%)** | **3,432,660 bits (+183.2%)** |
| **NIST Tests post Von Neumann debiasing** | **12/12 PASS** | **12/12 PASS** | **12/12 PASS** |
| **NIST Tests post XOR-folding (multi-seq)** | **11/11 PASS (100%)** | **11/11 PASS (100%)** | **11/11 PASS (100%)** |

### Key Takeaways:

1. **Linear QBER scaling on Star Topology:** Appending each successive peripheral qubit and `CZ` gate increases the XOR error rate by **~2.6 percentage points**. This level remains low enough to prove the preservation of non-local quantum correlations.
2. **Nearly threefold throughput increase (+183.2%):** Expanding from a 3-qubit to a 5-qubit setup drastically boosts secret key generation from `1.21 million` to **`3.43 million` bits**, while sharing the exact same public test cost.
3. **Debiasing Efficiency:** Despite the $T_1$ relaxation phenomenon, post-processing methods (Von Neumann, XOR-folding) completely neutralize initial bias, allowing the outputs to achieve a **100% pass rate in the NIST SP 800-22 battery**.

---

## 7. Output Files Structure and Key Export for PQC / AES

* **Bitstreams:** `sq1_public.txt`, `sq2_secret.txt` (and subsequent secret files for $n=4,5$), `sq_control.txt` (deleted in Step 7), `eqrng_checkpoint.json`.
* **Analytical Plots (300 DPI):** Topology map, state histogram, pattern distribution, p-value logarithmic chart (NIST), autocorrelation plots.
* **Tables and Reports:** `.csv` dataset, LaTeX `.tex` table, and a comprehensive `.json` report.
* **Cryptographic Material:**
* `eqrng_certified_key.bin` – SHA-256 conditioned binary key ready for **OTP** or **AES-256-GCM** encryption.
* `eqrng_seed_pqc.hex` – A certified 512-bit entropy seed formatted in hex for the post-quantum lattice algorithm **ML-KEM (CRYSTALS-Kyber)**.



---

## 8. Setup and Run Instructions

### Environment Requirements

```bash
pip install qiskit qiskit-iqm python-dotenv numpy scipy matplotlib pylatexenc

```

### Connection Setup

Create a `token.env` file in the root directory:

```env
IQM_SERVER_URL=[https://odra5.e-science.pl/cocos](https://odra5.e-science.pl/cocos)
IQM_TOKEN=your_secret_iqm_access_token

```

### Step 1: Topology Visualization

```bash
python eqrng_topology_visualizer.py

```

### Step 2: Quantum Data Acquisition on QPU (Stage 1)

```bash
python eqrng_stage1.py

```

*(If the connection is interrupted, rerunning the script safely resumes the process from the last checkpoint without data loss).*

### Step 3: Validation, Post-processing, and NIST Tests (Stage 2)

```bash
# Standard analysis:
python eqrng_stage2.py

# Analysis with secure disk wiping of the control file:
python eqrng_stage2.py --shred-sq3

# Fast demo test on synthetic data:
python eqrng_stage2.py --demo

```

```

```
