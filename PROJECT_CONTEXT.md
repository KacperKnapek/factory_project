# PROJECT_CONTEXT

**Ostatnia aktualizacja:** 2026-07-31

## Cel projektu
Analiza i modelowanie przepływu zleceń produkcyjnych w fabryce obuwia. Główne zadania: rekonstrukcja rzeczywistych ścieżek zleceń, identyfikacja miejsc kumulowania się czasu, analiza terminowości i budowa modelu predykcyjnego opóźnień.

## Dane
- ~30,885 rekordów, 97 kolumn
- 1,816 modeli, 279 kontrahentów
- Okres: koniec 2016 - połowa 2018
- Jeden wiersz = pozycja/partia produkcyjna (nie zawsze całe zamówienie klienta)
- Zawiera: identyfikatory zleceń, kontrahentów, modeli, rozmiarówkę, terminy, timestampy etapów produkcji, statusy, info o szwalni, produkcję zewnętrzną, odrzucenia

## Kluczowe ograniczenia
1. **Brak danych o zasobach** – nie ma informacji o liczbie pracowników, maszyn, zmianach, awariach, przezbrojeniach
2. **Mieszane czasy** – różnica między timestampami zawiera: czekanie w kolejce + transport + operacja + inne przestoje (nie czysty czas pracy)
3. **Brak pełnej historii** – jeden timestamp na etap; wielokrotne powroty na wcześniejszy etap mogą nie być widoczne
4. **Niejasne kolumny** – `plan`, `wstrzymane`, `termin zam` vs `termin prod`, `MAG 2/3`, `ploter` vs `rozkrój` wymagają weryfikacji przez dane

## Główne obserwacje
- `Rozkrój` i `ploter` nigdy nie występują razem – prawdopodobnie alternatywne ścieżki technologiczne
- `Plan` nie jest liczbą par (wartości kilka tysięcy vs. rozmiarówka ~8 par) – prawdopodobnie pozycja w planie
- `Wstrzymane` wypełnione dla większości rekordów, ale mało zleceń ma status wstrzymania
- Szczegółowe etapy zaznaczają się od marca 2017 – wcześniejsze dane mogą być niekompletne
- Daty zostały rozpoznane jako timestampy w formacie `MM/DD/YY HH:MM:SS`; braki są przechowywane jako `NaT`
- Kontrola kolejności wykazała przypadki, w których `termin zam` jest wcześniejszy niż `data zam`; kolumny terminów nie powinny być bezpośrednio traktowane jako kolejne etapy procesu

## Wykonane transformacje w EDA
- Usunięto kolumny całkowicie puste
- Dodano `czas produkcji` jako różnicę `zdane na magazyn - rozkrój`
- Zmieniono nazwę technicznego pola `pole1` na `komplety szwalnia`
- Dodano `liczba par` jako sumę kolumn rozmiarowych w wierszu i ustawiono typ `int`
- Po utworzeniu `liczba par` usunięto kolumny rozmiarowe `size_cols`

## Główne pytania
1. Jakie ścieżki przechodzą zlecenia? Jakie warianty są najczęstsze?
2. Które etapy są pomijane? Między którymi kumuluje się czas?
3. Jak długo trwa realizacja typowego zlecenia? Jaki % opóźnień?
4. Czy ploter vs rozkrój, szwalnie, wielkość partii, pilność, model, kontrahent wpływają na czas?
5. Czy można wcześnie przewidzieć opóźnienie?

## Przebieg projektu
```
Surowe dane → Audyt → Czyszczenie → Rekonstrukcja historii zleceń
→ Analiza wariantów procesu → Analiza czasów → Analiza terminowości
→ Analiza czynników ryzyka → Model predykcyjny → Opcjonalna symulacja
```

## Etapy (wykonane/plan)
1. Audyt danych – słownik kolumn, braki, zakresy dat
2. Kontrola jakości – logika sekwencji, nieprawidłowe daty, reguły
3. Oczyszczenie – nowe cechy (liczba par, lead time, opóźnienie, typ ścieżki)
4. Event log – format zdarzeń case_id/activity/timestamp
5. Warianty procesu – najczęstsze ścieżki, macierz przejść
6. Czasy przejścia – statystyki między każdą parą etapów
7. Terminowość – % opóźnień, mediany, percentyle
8. Czynniki ryzyka – regresja, drzewa, feature importance
9. Model predykcyjny – logistyka, drzewo, random forest, gradient boosting
10. Symulacja (opcjonalnie) – scenariusze dla wybranego fragmentu

## Ograniczenia interpretacji
- Projekt nie obejmuje: pełnego cyfrowego bliźniaka, modelowania wszystkich zasobów, obliczania optymalnego zatrudnienia
- Zależności statystyczne ≠ przyczyny; czasami bywają związane ze zmiennymi ukrytymi (np. trudne modele trafiają do określonej szwalni)
- Dane procesowe wiarygodne od marca 2017 roku

## Zasady pracy
- Najpierw analiza danych i hipotezy samodzielnie
- Pytania do eksperta (tata) tylko o rzeczy niemożliwe do rozstrzygnięcia z danych
- Najpilniejsze pytania: znaczenie pojedynczego wiersza, timestamp = wejście/wyjście/zmiana statusu?, rozkrój ↔ ploter alternatywne?

## Tryb edukacyjny i standard pracy z AI

Projekt ma charakter edukacyjny. Każdy etap powinien być prowadzony tak, aby można było zrozumieć nie tylko wynik, ale także sposób dojścia do niego.

- Każda decyzja, transformacja i metryka musi być wyjaśniona prostym językiem.
- Przy każdym etapie należy określić: cel, dane wejściowe, oczekiwany wynik oraz kryterium zakończenia.
- Należy wyraźnie rozdzielać fakty potwierdzone przez dane od hipotez i interpretacji wymagających weryfikacji.
- Nie wolno usuwać rekordów ani kolumn bez wyraźnego uzasadnienia, opisania skutków i zachowania informacji o wykonanej zmianie.
- Przy czyszczeniu danych należy pokazywać przykładowe rekordy przed i po transformacji.
- Po każdej istotnej transformacji należy wykonać walidację: sprawdzić rozmiar danych, braki, typy, zakresy oraz zgodność z regułami biznesowymi.
- Najpierw pracujemy w notebookach, aby proces był widoczny i możliwy do omówienia. Moduły `.py` tworzymy dopiero wtedy, gdy rozwiązanie jest zrozumiane, powtarzalne i wymaga uporządkowania lub ponownego użycia.
- Każda analiza powinna zawierać: pytanie, hipotezę, kod, wynik, interpretację, ograniczenia oraz wniosek.
- Na końcu każdego etapu należy podsumować: czego się nauczyliśmy, co potwierdziliśmy, czego nadal nie wiemy i jaki jest następny krok.

## Słownik terminów

- **case_id** – identyfikator pojedynczego przypadku analizowanego jako całość, np. zlecenia, pozycji zamówienia albo partii. Jego wybór musi wynikać ze struktury danych i być jawnie uzasadniony.
- **event log** – tabela zdarzeń procesu, w której każdy wiersz opisuje zdarzenie dla danego `case_id`; minimalne pola to identyfikator przypadku, aktywność i timestamp.
- **lead time** – całkowity czas od przyjętego początku do przyjętego końca procesu. Początek i koniec muszą być określone przed obliczeniem tej metryki.
- **wariant procesu** – uporządkowana sekwencja etapów, przez które przechodzi przypadek, np. `przyjęte → rozkrój → montaż → zdane na magazyn`.
- **opóźnienie** – przekroczenie ustalonego terminu, np. różnica między datą zakończenia a `termin prod`. Sposób liczenia i traktowanie brakujących dat należy zawsze opisać.
- **cecha** – zmienna używana w analizie lub modelu, np. liczba par, pilność, model, kontrahent albo wybrany wariant ścieżki.
