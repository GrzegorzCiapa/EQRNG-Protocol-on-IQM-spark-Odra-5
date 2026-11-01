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

```text
        QB1 (idx 0)           QB2 (idx 1)
         [H, ster.]            [H, ster.]
              \                    /
               \   CX / CZ        /
                v                v
                 QB3 (idx 2, Q3)
               [CENTRALNY CEL CX]
                ^                ^
               /   CX / CZ        \
              /                    \
         [H, ster.]            [H, ster.]
        QB4 (idx 3)           QB5 (idx 4)
