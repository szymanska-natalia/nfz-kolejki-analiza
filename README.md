# Dostępność specjalistów a kolejki NFZ — analiza w Power BI

Analiza sprawdzająca, czy liczba lekarzy specjalistów na mieszkańca tłumaczy długość kolejek do NFZ w polskich województwach. Wniosek: wbrew intuicji liczba lekarzy nie tłumaczy długości kolejek — zależność okazała się statystycznie nieistotna, co sugeruje, że przyczyn należy szukać gdzie indziej.

---

## Pytanie

Czy liczba specjalistów przypadających na mieszkańca tłumaczy, jak długo pacjent czeka w kolejce NFZ?

Pomysł na projekt wziął się z codziennego doświadczenia — sama czekam na wizyty u specjalistów miesiącami, a wiele osób czeka nawet latami. Mimo opłacania składek pacjenci często rezygnują z publicznej kolejki i płacą za wizyty prywatne. Chciałam sprawdzić, gdzie naprawdę leży problem: czy kolejki wynikają po prostu z braku lekarzy danej specjalności, czy z czegoś innego.

Do analizy wybrałam cztery specjalizacje o najdłuższych kolejkach, dla których dostępne były wiarygodne dane (psychiatria, neurologia, okulistyka, dermatologia), w podziale na 16 województw.

---

## Kluczowy wniosek

Prosta hipoteza „mniej lekarzy = dłuższe kolejki" **nie potwierdza się w danych.**

Aby to sprawdzić rzetelnie, obliczyłam współczynnik korelacji (Pearsona) między liczbą specjalistów na mieszkańca a średnim czasem oczekiwania — osobno dla każdej z czterech specjalizacji:

| Specjalizacja | Korelacja (r) | Interpretacja |
|---|---|---|
| Okulistyka | −0,31 | słaba ujemna |
| Dermatologia | +0,13 | brak |
| Neurologia | −0,07 | brak |
| Psychiatria | +0,25 | słaba dodatnia |

Wszystkie współczynniki mieszczą się w przedziale od −0,31 do +0,25, co oznacza **brak istotnej zależności** — dla żadnej specjalizacji liczba lekarzy nie tłumaczy długości kolejek. Mimo że na wykresach punktowych Power BI wykreślił linie trendu sugerujące pewną zależność, obliczenie współczynnika korelacji pokazało, że jest ona pozorna — linia przebiega przez rozrzucone punkty, ale ich nie opisuje.

**Wniosek: za długość kolejek odpowiada coś innego niż sama liczba specjalistów.** Najprawdopodobniej wpływają na nie czynniki, których ten projekt nie obejmuje: rozmieszczenie placówek, udział sektora prywatnego, sposób finansowania świadczeń przez NFZ, organizacja przyjęć czy migracja pacjentów między województwami.

Potwierdza to analiza liczby placówek: województwo śląskie ma ich najwięcej (423), a mimo to należy do regionów z długimi kolejkami. Więcej placówek nie oznacza krótszego oczekiwania.

> **Metoda:** korelacja Pearsona, n=16 województw na specjalizację, dane za 2025 r. Wnioski oparte na liczbach, nie na wizualnej ocenie wykresów. Szczegóły i zastrzeżenia w sekcji [Ograniczenia](#ograniczenia).

---

## Dane i źródła

Projekt łączy dane z trzech niezależnych źródeł publicznych:

- **Kolejki NFZ** — czas oczekiwania i liczba oczekujących na świadczenia, pobrane z publicznego API NFZ (Informator o Terminach Leczenia, endpoint `/queues`). Dane pobierane parametrycznie dla 4 specjalizacji × 16 województw. Stan na 2026 r. (snapshot).
- **Liczba specjalistów** — sprawozdania **MZ-89**, zbiór „Lekarze specjaliści zatrudnieni w placówkach ochrony zdrowia według województw", [dane.gov.pl](https://dane.gov.pl/pl/dataset/2107). Zakres: 2015–2025 (bez danych za 2024 r.).
- **Populacja województw** — [GUS, Bank Danych Lokalnych](https://bdl.stat.gov.pl), użyta do przeliczenia liczby specjalistów na 10 tys. mieszkańców. Zakres: 2015–2025.

Zakres analizy: **4 specjalizacje** (psychiatria, neurologia, okulistyka, dermatologia) × **16 województw**.

> **Uwagi o danych:**
> - **Zakres czasowy:** dane o kolejkach to migawka z 2026 r., dane o kadrze obejmują lata 2015–2025. Świadomy kompromis — kolejki NFZ dostępne były jako bieżący stan, nie szereg czasowy. Główna analiza (korelacja kadra–kolejki) używa danych kadrowych za 2025 r.
> - **Luka 2024:** zbiór o specjalistach nie zawiera danych za 2024 r., dlatego wykres trendu kadry pomija ten rok (linia łączy 2023 z 2025).

---

## Jak zbudowane

**Pozyskanie i czyszczenie danych — Power Query**

Dane o kolejkach pobrałam z API NFZ (Informator o Terminach Leczenia) parametrycznie. Napisałam funkcję `fn_pobierz_wszystko(benefit, wojewodztwo)`, którą wywołałam dla każdej z 64 kombinacji (4 specjalizacje × 16 województw). API zwraca wyniki porcjami po maksymalnie 25 rekordów, więc funkcja sama oblicza liczbę potrzebnych stron (na podstawie pola `meta.count` z odpowiedzi API) i pobiera wszystkie — łącznie ok. 300–400 zapytań na jedno odświeżenie danych.

Dodatkowa trudność: nazwy świadczeń w API NFZ są niespójne między specjalizacjami (np. „PORADNIA ZDROWIA PSYCHICZNEGO" dla psychiatrii, ale „ŚWIADCZENIA Z ZAKRESU NEUROLOGII" dla neurologii). Każdą nazwę musiałam zweryfikować ręcznie w przeglądarce, zanim zbudowałam automatyzację.

Czyszczenie obejmowało: rozwinięcie zagnieżdżonych struktur, ujednolicenie typów (kody województw i TERYT jako tekst, by zachować zera wiodące; wartości liczbowe i współrzędne w formacie en-US) oraz **zachowanie pustych wartości jako pustych, a nie zer** — zamiana na zero zaniżyłaby średnie czasy oczekiwania.

**Model danych — schemat gwiazdy**

Dane uporządkowałam w schemat gwiazdy: w centrum tabele z wartościami do analizy (kolejki NFZ, liczba specjalistów, populacja), a wokół tabele opisowe, według których te wartości filtruję — województwa, specjalizacje i lata. Dzięki temu jeden filtr (np. wybór specjalizacji) działa na cały dashboard.

Województwa w trzech źródłach były oznaczone na trzy różne sposoby (nazwa, kod NFZ „01–16", kod TERYT). Aby je połączyć, stworzyłam tabelę łączącą wszystkie trzy oznaczenia — bez niej tabele nie dałyby się ze sobą powiązać.

**Miary — DAX**

- `Średni czas oczekiwania (dni)` — średnia czasu oczekiwania z danych NFZ
- `Specjaliści na 10 tys` — liczba specjalistów podzielona przez populację województwa i przemnożona przez 10 000 (przeliczenie „na mieszkańca", by porównywać województwa różnej wielkości)
- `Liczba placówek` — liczba unikalnych świadczeniodawców

**Wizualizacja — Power BI + Excel**

Dashboard dwustronicowy: strona „Przegląd" (wskaźniki KPI + analiza zależności kadra–kolejki) oraz „Szczegóły" (rankingi województw + trend liczby specjalistów w czasie). Współczynniki korelacji obliczyłam dodatkowo w Excelu (korelacja Pearsona), ponieważ pozwoliło to zweryfikować zależności liczbowo, a nie tylko wizualnie.

---

## Dashboard

Interaktywny dashboard w Power BI, dwie strony połączone wspólnym filtrem specjalizacji.

**Strona 1 — Przegląd**

Wskaźniki KPI (średni czas oczekiwania, liczba specjalistów na 10 tys., liczba placówek) oraz dwa wykresy punktowe pokazujące zależność (a właściwie jej brak) między liczbą specjalistów a czasem oczekiwania — dla wybranej specjalizacji i dla wszystkich czterech naraz.

![Strona 1 — Przegląd](images/przeglad.png)

**Strona 2 — Szczegóły**

Trend liczby specjalistów na przestrzeni lat 2015–2025 oraz rankingi województw: według czasu oczekiwania i według liczby placówek.

![Strona 2 — Szczegóły](images/szczegoly.png)

---

## Ograniczenia

Wnioski z tej analizy należy traktować jako sygnały, nie twarde dowody — z kilku powodów:

- **Mała próba (n=16).** Każda korelacja liczona jest na 16 województwach. To za mało, by wyciągać mocne wnioski statystyczne — wyniki pokazują tendencje, nie pewniki.
- **Tylko cztery specjalizacje.** Brak zależności w czterech dziedzinach nie znaczy, że nie istnieje w innych. Pełniejszy obraz wymagałby zbadania większej liczby specjalizacji.
- **Korelacja ≠ przyczynowość.** Nawet gdyby zależność była silniejsza, współwystępowanie nie oznacza, że jedno powoduje drugie.
- **Definicja „specjalisty".** Liczba specjalistów pochodzi ze sprawozdań MZ-89, których definicja może nie obejmować wszystkich pracujących lekarzy danej specjalności (np. liczby dla psychiatrii wydają się zaniżone).
- **Spadek w 2017 r. — przyczyna niepotwierdzona.** W danych o specjalistach widoczny jest wyraźny spadek między 2016 a 2017 r. Przyczyna wymaga weryfikacji — prawdopodobnie wynika ze zmian metodologicznych w sprawozdawczości (możliwe powiązanie z wprowadzeniem sieci szpitali w 2017 r. lub zmianą definicji „zatrudnionego specjalisty"), ale nie zostało to potwierdzone w notach metodologicznych źródła. Spadku nie należy więc interpretować jako pewnego faktycznego ubytku kadry.
- **Różny zakres czasowy.** Kolejki to stan z 2026 r., kadra z 2015–2025 — zestawienie najnowszych dostępnych danych, ale nie z tego samego momentu.
- **Luka 2024.** Brak danych o specjalistach za 2024 r.
- **Średnia nieważona.** Średni czas oczekiwania to prosta średnia z placówek, nieważona ich wielkością ani liczbą pacjentów.
- **Tylko przypadki stabilne.** Z API NFZ pobrałam wyłącznie przypadki stabilne (`case=1`), pomijając przypadki pilne (`case=2`) jako odrębną kategorię.

---

## Co dalej

Kierunki, w których projekt można rozwinąć:

- **Więcej specjalizacji** — sprawdzenie, czy brak zależności utrzymuje się szerzej, czy dotyczy tylko tych czterech dziedzin.
- **Inne czynniki** — włączenie danych, które mogą realnie tłumaczyć kolejki: rozmieszczenie placówek, udział sektora prywatnego, finansowanie NFZ.
- **Przypadki pilne** — dodanie kategorii `case=2` (przypadki pilne) obok stabilnych, by porównać oba typy kolejek.
- **Zmiana w czasie** — zestawienie kolejek i kadry z tego samego okresu (gdyby dane historyczne o kolejkach były dostępne), zamiast migawki.

---

## Jak uruchomić

1. Pobierz plik `dostepnosc_specjalistow.pbix`.
2. Otwórz w [Power BI Desktop](https://powerbi.microsoft.com/desktop/) (bezpłatny).
3. Dashboard jest interaktywny — użyj filtra specjalizacji, by przełączać między psychiatrią, neurologią, okulistyką i dermatologią.

Dane w pliku to migawka z 2026 r.

---

*Projekt portfolio — analiza danych (Junior Data Analyst). Narzędzia: Power BI (Power Query, DAX), Excel.*

*English version of this README: [README.en.md](README.en.md).*
