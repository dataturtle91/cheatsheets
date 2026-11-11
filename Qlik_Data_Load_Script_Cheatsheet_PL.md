# Qlik Data Load Script — cheatsheet dla BI Developera

Zakres: wyłącznie skrypt ładowania, ETL i model danych. Przykłady celują w Qlik Sense; szablon Section Access wskazuje konkretnie wariant Windows. Dokumentację sprawdzono 11 listopada 2026 r.

**Jak używać:** bloki są niezależnymi wzorcami, a nie jednym skryptem do uruchomienia w całości. `lib://...`, połączenia i tabele źródłowe wymagają dopasowania. Jeśli przykład używa `Resident`, zakłada istnienie wskazanej tabeli. Kod nie był wykonywany w silniku Qlik; źródła przy istotnych regułach prowadzą do dokumentacji producenta.

## Spis treści

1. [Metody ładowania danych](#1-metody-ładowania-danych)
2. [Czyszczenie danych i NULL](#2-czyszczenie-danych-i-null)
3. [Tekst, liczby, daty i statystyka](#3-tekst-liczby-daty-i-statystyka)
4. [Transformacje i model danych](#4-transformacje-i-model-danych)
5. [Zmienne, stan i sterowanie](#5-zmienne-stan-i-sterowanie)
6. [QVD i incremental load](#6-qvd-i-incremental-load)
7. [Kalendarz, uprawnienia i diagnostyka](#7-kalendarz-uprawnienia-i-diagnostyka)

## Zanim zaczniesz: model wykonania Qlik

| Znasz z SQL / Power Query / R | Odpowiednik lub różnica w Qlik |
|---|---|
| `SELECT`, `mutate()`, `Table.AddColumn` | `LOAD` wybiera pola i oblicza wyrażenia dla rekordów. Zwykle nie potrzebujesz pętli po wierszach. |
| Query folding w Power Query | `SQL SELECT` wykonuje pracę w bazie. Transformacje w Qlik `LOAD` nie są automatycznie składane z powrotem do SQL. |
| CTE / subquery / pipeline `%>%` | Preceding LOAD buduje kolejne warstwy transformacji; czytasz wykonanie od dołu. |
| Tabela tymczasowa / materializowany data frame | `Resident LOAD` czyta tabelę już załadowaną do pamięci Qlik. |
| R: `summarise(group_by(df, CustomerID), Total = sum(Amount, na.rm = TRUE))` | `LOAD CustomerID, Sum(Amount) AS Total Resident Sales GROUP BY CustomerID;` — tworzy tabelę agregatów z jednym wierszem na klienta. |
| `UNION ALL`, `bind_rows()` | `Concatenate`; pola są dopasowywane po nazwach. |
| `LAG() OVER (...)` | `Previous()` lub `Peek()` po zapewnieniu sortowania i obsłużeniu granic grup. |
| Relacje po jawnych warunkach `ON` | W modelu Qlik pola o identycznych nazwach tworzą asocjacje automatycznie. |

„Wektoryzacja” to tu przydatna analogia do deklarowania transformacji całej tabeli, a nie gwarancja konkretnego sposobu wykonania CPU. Qlik ma też funkcje stanowe, więc kolejność rekordów może mieć znaczenie.

Wartość Qlik może być **dual**: jednocześnie reprezentacją liczbową i tekstową. Daty są liczbami dni, a czas częścią ułamkową dnia. Funkcje formatujące, np. `Date()` i `Num()`, zwykle zachowują część liczbową.

```qlik
// Jawne ustawienia ograniczają zależność od konfiguracji serwera.
SET DecimalSep='.';
SET ThousandSep=',';
SET DateFormat='YYYY-MM-DD';
SET TimestampFormat='YYYY-MM-DD hh:mm:ss';
SET FirstWeekDay=0;   // Poniedziałek.
SET BrokenWeeks=0;
SET ReferenceDay=4;  // Reguły numeracji tygodni zgodne z ISO.
SET ErrorMode=1;     // Nie ignoruj błędów skryptu.
```

**Konwencje:** instrukcje kończ średnikiem; tekst zapisuj w `'apostrofach'`, nazwy z odstępami w `[nawiasach]`. Komentarze: `//` i `/* ... */`. Nazwy pól rozróżniają wielkość liter. Wartość logiczna true to zwykle `-1`, false to `0`; do flag sumowanych wygodniej tworzyć jawne `1/0`.

## 1. Metody ładowania danych

### 1.1. LOAD vs SELECT — miejsce wykonania transformacji

`SQL SELECT` jest przekazywany przez konektor do źródła i używa dialektu bazy. `LOAD` wykonuje funkcje Qlik oraz czyta m.in. pliki, QVD, dane inline i tabele rezydentne.

```qlik
LIB CONNECT TO 'DWH_SQL';

Sales:
LOAD
    OrderID,
    CustomerID,
    Date(Floor(OrderTimestamp)) AS OrderDate,
    Quantity * UnitPrice AS NetAmount;
SQL SELECT
    OrderID,
    CustomerID,
    OrderTimestamp,
    Quantity,
    UnitPrice
FROM dbo.OrderLines
WHERE IsCancelled = 0;
// WHERE wykonuje baza; Date/Floor i mnożenie wykonuje Qlik.

Products:
LOAD
    ProductID,
    Trim(ProductName) AS ProductName
FROM [lib://Landing/products.csv]
(txt, utf8, embedded labels, delimiter is ';', msq);
```

**Pro-Tip:** ograniczaj kolumny i rekordy możliwie blisko źródła. Koszt transferu milionów odrzuconych później rekordów zwykle przewyższa koszt prostego `WHERE` w bazie. Nie oczekuj query foldingu od preceding LOAD. Składnię źródeł opisuje [Qlik: LOAD](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/ScriptRegularStatements/Load.htm).

### 1.2. Preceding LOAD — wieloetapowa transformacja

Warstwy wykonują się **od dołu do góry**. To odpowiednik kompozycji projekcji SQL lub kolejnych `mutate()`, ale bez samodzielnej tabeli pośredniej dla każdej warstwy. Alias obliczony w jednej warstwie jest dostępny dla warstwy nad nią; nie używaj go w innym wyrażeniu tego samego `LOAD`.

```qlik
Sales:
LOAD
    OrderID,
    OrderDate,
    NetAmount,
    Round(NetAmount * 1.23, 0.01) AS GrossAmount;
LOAD
    OrderID,
    Date(Date#(OrderDateText, 'YYYY-MM-DD')) AS OrderDate,
    Quantity * UnitPrice AS NetAmount;
SQL SELECT OrderID, OrderDateText, Quantity, UnitPrice
FROM dbo.OrderLines;
```

**Pro-Tip:** preceding LOAD zwykle oszczędza materializację względem kilku tabel `Resident`. Potrzebujesz jednak `Resident`, gdy chcesz użyć `ORDER BY`. Preceding LOAD ma ograniczenia z prefiksami `Join`, `Crosstable` i `IntervalMatch`; nie traktuj go jak dowolnie optymalizowalnego CTE. Jeśli deduplikujesz wynik, umieść `DISTINCT` w najwyższym LOAD. [Qlik: Resident i preceding LOAD](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/load-data-from-previously-loaded-table.htm).

### 1.3. Resident LOAD i GROUP BY — materializacja i agregacja

`Resident` czyta istniejącą tabelę w pamięci. Wszystkie pola używane poza funkcjami agregującymi muszą należeć do klucza grupowania; Qlik nie wywnioskuje zależności funkcyjnej jak „rok wynika z daty”. [Qlik: GROUP BY](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/ScriptRegularStatements/Load.htm).

```qlik
// RawSales: OrderID, CustomerID, OrderDate, NetAmount.
SalesPrepared:
LOAD
    OrderID,
    CustomerID,
    MonthStart(OrderDate) AS MonthKey,
    NetAmount
Resident RawSales;

SalesMonthly:
LOAD
    CustomerID,
    MonthKey,
    Sum(NetAmount) AS MonthlyNet,
    Count(DISTINCT OrderID) AS OrderCount,
    Avg(NetAmount) AS AvgLineNet
Resident SalesPrepared
GROUP BY CustomerID, MonthKey;

DROP TABLE SalesPrepared;
// RawSales usuń tylko wtedy, gdy detal nie jest już potrzebny.
```

**Pro-Tip:** analogią w R jest grupowanie zakończone agregacją przez `summarise()`. W Qliku `GROUP BY` działa w obrębie konkretnego LOAD: wyznacza grupy dla agregacji i powstaje tabela wynikowa; źródło nie otrzymuje trwałego „stanu grupowania”. Filtr odpowiadający SQL `HAVING` wykonaj w następnym `Resident LOAD ... WHERE MonthlyNet > ...`. Nie zostawiaj w modelu detalu i agregatu z kilkoma wspólnymi polami bez zaplanowania ich asocjacji.

### 1.4. INLINE i AUTOGENERATE — słowniki i generatory rekordów

`INLINE` umieszcza mały zbiór bezpośrednio w skrypcie. `AUTOGENERATE n` tworzy rekordy bez zewnętrznego źródła; `RecNo()` numeruje rekordy wejściowe od 1.

```qlik
StatusDictionary:
LOAD * INLINE [
StatusCode, StatusName
N, Nowe
P, W toku
C, Zamkniete
];

Numbers:
LOAD RecNo() AS N
AUTOGENERATE 10;

// Jeden rekord bazowy powtarzany przez WHILE.
LET vStartDate = MakeDate(2026, 1, 1);
LET vEndDate   = MakeDate(2026, 1, 31);

Dates:
LOAD
    Date($(vStartDate) + IterNo() - 1) AS GeneratedDate
AUTOGENERATE 1
WHILE $(vStartDate) + IterNo() - 1 <= $(vEndDate);
```

**Pro-Tip:** rozróżniaj `RecNo()` — pozycję wejścia, `RowNo()` — pozycję wyniku, oraz `IterNo()` — kolejne powtórzenie jednego wejścia przez `WHILE`. Generowanie kalendarza w jednym LOAD jest prostsze od proceduralnego `FOR` po dniach.

## 2. Czyszczenie danych i NULL

### 2.1. Alt() i Coalesce() — istotna różnica względem SQL

**`Alt()` nie jest ogólnym odpowiednikiem SQL `COALESCE`.** Zwraca pierwszy argument mający poprawną reprezentację liczbową; jeśli żaden nie pasuje, zwraca ostatni argument. Nadaje się do liczb i prób interpretacji dat. `Coalesce()` wybiera pierwszą wartość różną od NULL, również tekst. [Qlik: Alt](https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/Scripting/ConditionalFunctions/alt.htm), [Qlik: Coalesce](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/ConditionalFunctions/coalesce.htm).

```qlik
CleanSales:
LOAD
    OrderID,
    Alt(Amount, 0) AS AmountOrZero,
    Coalesce(CustomerName, 'Nieznany') AS CustomerNameClean,
    Date(
        Alt(
            Date#(RawDate, 'YYYY-MM-DD'),
            Date#(RawDate, 'DD.MM.YYYY'),
            Null()
        )
    ) AS ParsedDate
Resident RawSales;

NullExamples:
LOAD
    Alt('ABC', 'Brak') AS AltResult,        // 'Brak'
    Coalesce('ABC', 'Brak') AS TextResult,  // 'ABC'
    Coalesce('', 'Brak') AS EmptyResult    // Pusty tekst, nie 'Brak'.
AUTOGENERATE 1;
```

**Pro-Tip:** `Alt(Amount, 0)` może zamaskować błędny tekst liczbowy tak samo jak brak danych. Zachowuj pole surowe albo flagę błędu parsowania. W starszej wersji bez `Coalesce()` użyj `If(IsNull(x), replacement, x)`.

### 2.2. IsNull() — NULL, pusty tekst i białe znaki

`IsNull(x)` wykrywa rzeczywisty NULL. `''`, `'NULL'` i `'   '` to osobne przypadki. Porównanie `x = Null()` nie zastępuje testu braku.

```qlik
CustomersClean:
LOAD
    CustomerID,
    If(IsNull(Email), 1, 0) AS EmailWasNull,
    If(
        IsNull(Email) OR Len(Trim(Email)) = 0,
        Null(),
        Lower(Trim(Email))
    ) AS EmailClean
Resident CustomersRaw;
```

**Pro-Tip:** jeżeli źródło używa literalnego `'NULL'` lub `'N/A'`, ujednolicaj je jawnie według kontraktu źródła. Nie zastępuj automatycznie wszystkich takich tekstów w całym modelu — mogą być prawidłową wartością biznesową.

### 2.3. NullAsValue i NullValue — materializacja braków

`NullAsValue` włącza konwersję NULL na zwykłą wartość w wybranych polach kolejnych ładowań. `NullValue` określa zastępczy tekst. Samo ustawienie `NullValue` nie uruchamia konwersji. `NullAsNull` wyłącza ją dla następnych ładowań; nie cofa już zapisanych wartości. [Qlik: NullAsValue](https://help.qlik.com/en-US/sense/february2019/Subsystems/Hub/Content/Sense_Hub/Scripting/ScriptRegularStatements/NullAsValue.htm), [Qlik: NullValue](https://help.qlik.com/en-US/sense/May2025/Subsystems/Hub/Content/Sense_Hub/Scripting/ValueHandlingVariables/NullValue.htm).

```qlik
SET NullValue='<BRAK>';
NullAsValue Region;

Customers:
LOAD CustomerID, Region
FROM [lib://Curated/Customers.qvd] (qvd);

NullAsNull Region;  // Przywrócenie domyślnej semantyki na dalszy skrypt.
```

**Pro-Tip:** prawdziwe NULL-e nie tworzą normalnych asocjacji ze sobą. Po zamianie na `'<BRAK>'` stają się wspólnym symbolem i mogą łączyć tabele. Dla kluczy stosuj świadomie zaprojektowany rekord „Unknown”; unikaj globalnego `NullAsValue *`.

## 3. Tekst, liczby, daty i statystyka

### 3.1. Funkcje tekstowe — czyszczenie i ekstrakcja

| Funkcja ([] = argument opcjonalny) | Zastosowanie | Analogia / uwaga |
|---|---|---|
| `Trim(text)` | Usuwa skrajne spacje | SQL `TRIM`, R `trimws()` |
| `LTrim(text)` | Usuwa spacje z początku | Czyszczenie lewego marginesu |
| `RTrim(text)` | Usuwa spacje z końca | Czyszczenie prawego marginesu |
| `Upper(text)` | Zamienia litery na wielkie | Normalizacja kodów |
| `Lower(text)` | Zamienia litery na małe | Normalizacja e-maili |
| `Capitalize(text)` | Zamienia pierwsze litery słów na wielkie, pozostałe na małe | Formatowanie nazw; ostrożnie ze skrótowcami |
| `Len(text)` | Liczy znaki | SQL `LENGTH`; NULL to inny przypadek niż długość 0 |
| `Left(text, count)` | Pobiera pierwsze `count` znaków | Ekstrakcja prefiksu |
| `Right(text, count)` | Pobiera ostatnie `count` znaków | Ekstrakcja sufiksu |
| `Mid(text, start[, count])` | Pobiera fragment od pozycji `start`, numerowanej od **1**; bez `count` — do końca | SQL `SUBSTRING`; Power Query `Text.Middle` zaczyna od **0**. [Dokumentacja Mid](https://help.qlik.com/en-US/sense/November2025/Subsystems/Hub/Content/Sense_Hub/Scripting/StringFunctions/Mid.htm) |
| `PurgeChar(text, chars)` | Usuwa każdy znak należący do zbioru | Filtrowanie znaków, nie podciągów |
| `KeepChar(text, chars)` | Zostawia tylko wskazane znaki | Biała lista znaków |
| `Replace(text, from, to)` | Zamienia podciąg | SQL `REPLACE` |
| `SubField(text, delimiter[, n])` | Zwraca n-ty fragment; ujemne n liczy od końca | Bez n w LOAD generuje wiersze |
| `TextBetween(text, left, right[, n])` | Pobiera n-ty fragment pomiędzy ogranicznikami | Brak pasującego fragmentu daje NULL |
| `Index(text, substring[, n])` | Pozycja n-tego wystąpienia podciągu; brak = 0 | Pozycje od 1; ujemne n szuka od końca. [Dokumentacja Index](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/StringFunctions/Index.htm) |
| `FindOneOf(text, chars[, n])` | Pozycja n-tego napotkanego znaku ze zbioru; brak = 0 | Szuka dowolnego znaku, nie całego podciągu. [Dokumentacja FindOneOf](https://help.qlik.com/en-US/sense/february2019/Subsystems/Hub/Content/Sense_Hub/Scripting/StringFunctions/FindOneOf.htm) |
| `SubStringCount(text, substring)` | Liczy wystąpienia podciągu | Kontrola separatorów |
| `Repeat(text, n)` | Powtarza tekst n razy | Budowanie wypełnienia |
| `Text(expr)` | Wymusza interpretację tekstową | Nie odtworzy zer wiodących utraconych wcześniej |
| `Match(text, value1, value2, ...)` | Zwraca indeks dokładnego dopasowania lub 0 | Rozróżnia wielkość liter |
| `MixMatch(text, value1, value2, ...)` | Zwraca indeks dopasowania lub 0 | Bez rozróżniania wielkości liter |
| `WildMatch(text, pattern1, pattern2, ...)` | Zwraca indeks pasującego wzorca lub 0 | `*` i `?`; bez rozróżniania wielkości liter |
| `MapSubstring('map', text)` | Zamienia podciągi według mapy | Dopasowanie od lewej; rozróżnia wielkość liter, nie przetwarza wyniku ponownie. [Dokumentacja MapSubstring](https://help.qlik.com/en-US/sense/May2025/Subsystems/Hub/Content/Sense_Hub/Scripting/MappingFunctions/MapSubstring.htm) |
| `Hash128(expr1, expr2, ...)` | Tworzy tekstowy hash składników | Klucze złożone |
| `Hash256(expr1, expr2, ...)` | Tworzy tekstowy hash 256-bitowy | Sygnatura atrybutów do wykrywania zmian |
| `Concat([DISTINCT] text[, delimiter[, sort_weight]])` | Agreguje tekst z wielu rekordów | SQL `STRING_AGG`; w LOAD wymaga właściwej agregacji/GROUP BY |
| Dokumentacja katalogu | [Funkcje tekstowe Qlik](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/StringFunctions/string-functions.htm) | [Match, MixMatch i WildMatch](https://help.qlik.com/en-US/qlikview/September2026/Subsystems/Client/Content/QV_QlikView/Scripting/ConditionalFunctions/conditional-functions.htm) |

```qlik
TextClean:
LOAD
    CustomerID,
    Trim(CustomerName) AS CustomerNameClean,
    PurgeChar(Phone, ' +-()') AS PhoneCompact,
    KeepChar(Phone, '0123456789') AS PhoneDigits,
    Replace(ProductCode, 'OLD-', 'NEW-') AS ProductCodeClean,
    SubField(Email, '@', 2) AS EmailDomain,
    SubField(FilePath, '/', -1) AS FileName,
    TextBetween(Payload, '<id>', '</id>') AS PayloadID
Resident RawText;

// Brak trzeciego argumentu SubField generuje WIELE wierszy.
OrderTags:
LOAD
    OrderID,
    Trim(SubField(Tags, '|')) AS Tag
Resident OrdersRaw;
// 'BI|ETL|Qlik' -> trzy rekordy danego OrderID.
```

**Pro-Tip:** `SubField()` bez indeksu odpowiada raczej `separate_rows()` niż zwykłemu `strsplit()` zwracającemu listę. Dwa takie wywołania w jednym LOAD tworzą kombinacje kartezjańskie. Dlatego rozbijaj tagi do tabeli pomostowej, a nie do faktów z kwotą powielaną w każdym wierszu. [Qlik: SubField](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/StringFunctions/SubField.htm).

### 3.2. Funkcje numeryczne — wartość a reprezentacja

`Round(x, step)`, `Ceil(x, step)` i `Floor(x, step)` operują na wielokrotności kroku. **Drugi argument nie oznacza liczby miejsc po przecinku**: dwa miejsca to krok `0.01`. `Fabs()` zwraca wartość bezwzględną, `Num()` formatuje liczbę. [Qlik: funkcje numeryczne](https://help.qlik.com/en-US/sense/May2025/Subsystems/Hub/Content/Sense_Hub/ChartFunctions/GeneralNumericFunctions/general-numeric-functions-charts.htm).

| Funkcja ([] = argument opcjonalny) | Zastosowanie | Uwagi |
|---|---|---|
| `Round(x[, step[, offset]])` | Zaokrągla do najbliższej wielokrotności kroku przesuniętej o offset | Dwa miejsca dziesiętne: `step=0.01`, nie `2` |
| `Ceil(x[, step[, offset]])` | Zaokrągla w górę do granicy kroku | Domyślnie krok 1, offset 0 |
| `Floor(x[, step[, offset]])` | Zaokrągla w dół do granicy kroku | Dla liczb ujemnych to nie obcięcie w kierunku zera |
| `Fabs(x)` | Wartość bezwzględna | SQL/R `abs()` |
| `Sign(x)` | Znak liczby: -1, 0 lub 1 | Klasyfikacja korekt i kierunku zmiany |
| `Frac(x)` | Część ułamkowa jako `x - Floor(x)` | Wyodrębnienie czasu z timestampu |
| `Div(a, b)` | Część całkowita ilorazu | Kontroluj dzielenie przez zero |
| `Mod(a, b)` | Nieujemna reszta z dzielenia całkowitego | Oba argumenty całkowite, `b > 0`; reguły różnią się między dialektami SQL. [Dokumentacja Mod](https://help.qlik.com/en-US/qlikview/September2026/Subsystems/Client/Content/QV_QlikView/ChartFunctions/GeneralNumericFunctions/mod.htm) |
| `Fmod(a, b)` | Reszta z dzielenia również dla liczb rzeczywistych | Nie utożsamiaj z ograniczonym do integerów `Mod()` |
| `Even(x)` / `Odd(x)` | Test parzystości / nieparzystości | Wynik logiczny -1/0; dla niecałkowitego x — NULL |
| `Pow(x, y)` | Potęgowanie x do y | R `x^y` |
| `Sqr(x)` | Kwadrat x | `x*x` |
| `Sqrt(x)` | Pierwiastek kwadratowy | Wymaga `x >= 0` |
| `Exp(x)` | Funkcja wykładnicza e do x | Odwrotność logarytmu naturalnego |
| `Log(x)` | Logarytm naturalny | Wymaga `x > 0`; nie logarytm dziesiętny |
| `Log10(x)` | Logarytm o podstawie 10 | Wymaga `x > 0` |
| `Num(x[, format[, decimal_sep[, thousand_sep]]])` | Formatuje liczbę jako dual | Nie służy do parsowania tekstu ani zmiany precyzji obliczeń |
| `Num#(text[, format[, decimal_sep[, thousand_sep]]])` | Interpretuje tekst liczbowy | Jawne separatory przy niejednorodnych źródłach |
| `Money(x[, format[, decimal_sep[, thousand_sep]]])` | Formatuje kwotę | Nie przelicza waluty |
| `Money#(text[, format[, decimal_sep[, thousand_sep]]])` | Interpretuje tekst kwoty | Dopasuj maskę i symbol waluty do źródła |
| `Dual(text, number)` | Łączy etykietę i wartość liczbową | Np. etykieta kategorii z kluczem sortowania |
| `IsNum(expr)` | Sprawdza możliwość interpretacji liczbowej | Zwraca wynik logiczny, nie sparsowaną liczbę |
| `Alt(expr1, expr2, ..., else)` | Pierwsza poprawna reprezentacja liczbowa lub wartość końcowa | Nie jest ogólnym tekstowym COALESCE |
| `RangeSum(x1, x2, ...)` | Suma listy wartości | NULL i tekst nienumeryczny traktuje jak 0 |
| `RangeAvg(x1, x2, ...)` | Średnia wartości liczbowych z listy | Braków nie zamienia na zera |
| `RangeMin(x1, x2, ...)` / `RangeMax(x1, x2, ...)` | Minimum / maksimum listy | Przydatne do ograniczania zakresu; nie GROUP BY |
| `RangeCount(x1, x2, ...)` | Liczy wartości tekstowe i liczbowe | NULL pomija; nie jest testem typu liczbowego |
| `RangeNumericCount(x1, x2, ...)` | Liczy wartości liczbowe | Kontrola kompletności pól numerycznych |
| `RangeNullCount(x1, x2, ...)` | Liczy rzeczywiste NULL-e | Pusty tekst nie jest NULL-em |
| `Sum(expr)` / `Avg(expr)` / `Min(expr)` / `Max(expr)` | Agregacje rekordów | W skrypcie respektuj ziarno GROUP BY |
| `Median(expr)` / `Stdev(expr)` / `Fractile(expr, fraction)` | Mediana, odchylenie standardowe, kwantyl | Agregacje grup; `fraction` określa kwantyl |
| Dokumentacja katalogu | [Funkcje numeryczne](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/ChartFunctions/GeneralNumericFunctions/general-numeric-functions-charts.htm), [potęgi i logarytmy](https://help.qlik.com/en-us/sense/November2023/Subsystems/Hub/Content/Sense_Hub/Scripting/ExponentialAndLogarithmicFunctions/exponential-logarithmic-functions.htm) | Funkcje agregujące i Range opisano także w punkcie 3.4 |

```qlik
NumericDemo:
LOAD
    Round(123.456, 0.01) AS RoundedAmount,       // 123.46
    Ceil(123, 10) AS NextTen,                   // 130
    Floor(123, 10) AS PreviousTen,              // 120
    Floor(-1.2) AS NegativeFloor,               // -2, nie -1
    Fabs(-15) AS AbsoluteAmount,                // 15
    Num(1234.5, '#,##0.00', ',', ' ') AS Label, // Tekst 1 234,50 + liczba
    Num#('1 234,50', '#,##0.00', ',', ' ') AS ParsedAmount
AUTOGENERATE 1;
```

**Pro-Tip:** `Num()` formatuje, `Num#()` interpretuje tekst. Nie traktuj `Num()` jako odpowiednika SQL `CAST(... AS DECIMAL)`. Zaokrąglenie każdej pozycji i zaokrąglenie sumy mogą dać inny wynik; przy kwotach stosuj uzgodnioną regułę oraz ewentualnie całkowite jednostki groszy.

### 3.3. Daty i czas — parsowanie, formatowanie, granice okresów

| Funkcja ([] = argument opcjonalny) | Znaczenie | Uwagi |
|---|---|---|
| `Date(number[, format])` | Formatuje liczbę jako datę | Nie usuwa części godzinowej; dzienny klucz: `Date(Floor(ts))` |
| `Date#(text[, format])` | Interpretuje tekst jako datę | Jawna maska uniezależnia parser od ustawień serwera |
| `Timestamp(number[, format])` | Formatuje datę i czas | Nie wykonuje konwersji strefy czasowej |
| `Timestamp#(text[, format])` | Interpretuje tekst daty i czasu | Dopasuj też część ułamkową sekund, jeśli występuje |
| `Time(number[, format])` | Formatuje czas doby | Do wyodrębnienia części czasowej użyj `Frac(ts)` |
| `Time#(text[, format])` | Interpretuje tekst czasu | Różnica pomiędzy zegarem a interwałem ma znaczenie |
| `Interval(number[, format])` | Formatuje czas trwania | Liczba wejściowa jest w dniach; używaj dla różnicy timestampów |
| `Interval#(text[, format])` | Interpretuje tekst czasu trwania | Nadaje się do interwałów przekraczających dobę |
| `MakeDate(year[, month[, day]])` | Buduje datę | Pominięte miesiąc i dzień przyjmują 1 |
| `MakeTime(hour[, minute[, second]])` | Buduje czas z części | Sekundy mogą zawierać część ułamkową |
| `MakeWeekDate(weekyear[, week[, weekday]])` | Buduje datę z roku tygodniowego i tygodnia | Skrócona sygnatura; dostępne też parametry reguł tygodnia. [Dokumentacja MakeWeekDate](https://help.qlik.com/en-us/sense/November2023/Subsystems/Hub/Content/Sense_Hub/Scripting/DateAndTimeFunctions/makeweekdate.htm) |
| `Year(date)` | Rok kalendarzowy | Nie zawsze ten sam co rok tygodniowy |
| `Month(date)` | Miesiąc jako dual: nazwa i numer 1–12 | Sama nazwa miesiąca nie identyfikuje roku |
| `Day(date)` | Dzień miesiąca | Numer 1–31 |
| `WeekDay(date[, first_week_day])` | Nazwa dnia i indeks 0–6 | Indeks zależy od wskazanego początku tygodnia |
| `Week(date[, first_week_day[, broken_weeks[, reference_day]]])` | Numer tygodnia | ISO wymaga właściwych ustawień, nie tylko wywołania `Week()`. [Dokumentacja Week](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/DateAndTimeFunctions/week.htm) |
| `WeekYear(date)` | Rok, do którego należy tydzień | Używaj razem z `Week()` |
| `Hour(ts)` / `Minute(ts)` / `Second(ts)` | Składniki czasu | Nie myl ich z całkowitą długością interwału |
| `DayStart(date[, period_no])` / `DayEnd(date[, period_no])` | Początek / ostatnia milisekunda dnia | `period_no` przesuwa o dni |
| `WeekStart(date[, period_no[, first_week_day]])` / `WeekEnd(date[, period_no[, first_week_day]])` | Granice tygodnia | `period_no` przesuwa o tygodnie |
| `MonthStart(date[, period_no])` | Początek miesiąca | Stabilny miesięczny klucz liczbowy |
| `MonthEnd(date[, period_no])` | Ostatnia milisekunda miesiąca | Do dziennego klucza końca miesiąca dodaj `Floor()` |
| `QuarterStart(date[, period_no[, first_month_of_year]])` / `QuarterEnd(date[, period_no[, first_month_of_year]])` | Granice kwartału | Parametr początku roku pozwala uwzględnić rok fiskalny |
| `YearStart(date[, period_no[, first_month_of_year]])` / `YearEnd(date[, period_no[, first_month_of_year]])` | Granice roku | `period_no` przesuwa o lata; początek roku nie musi być w styczniu |
| `MonthName(date[, period_no])` | Etykieta miesiąc–rok z liczbowym początkiem miesiąca | Dual do identyfikacji okresu |
| `QuarterName(date[, period_no[, first_month_of_year]])` | Etykieta kwartału z rokiem | Dual, nie sam numer kwartału |
| `AddMonths(date, n[, mode])` | Przesuwa o miesiące kalendarzowe | `mode=1` zmienia regułę względem końca miesiąca dla dni od 28. wzwyż |
| `AddYears(date, n)` | Przesuwa o lata | Wartość ujemna cofa datę. [Dokumentacja AddYears](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/DateAndTimeFunctions/addyears.htm) |
| `Age(as_of_date, birth_date)` | Wiek w ukończonych latach | Nie jest prostą różnicą numerów lat |
| `InMonth(date, base_date, period_no)` | Test przynależności do miesiąca względem daty bazowej | `0` = miesiąc bazowy, `-1` = poprzedni; wynik -1/0 |
| `InMonthToDate(date, base_date, period_no)` | Test MTD względem daty bazowej | Obejmuje koniec dnia bazowego, nie tylko jego godzinę |
| `InQuarter(date, base_date, period_no[, first_month_of_year])` | Test przynależności do kwartału | Umożliwia przesunięcie i rok fiskalny |
| `InQuarterToDate(date, base_date, period_no[, first_month_of_year])` | Test QTD | Do dnia odniesienia w odpowiednim kwartale |
| `InYear(date, base_date, period_no[, first_month_of_year])` | Test przynależności do roku | Uwzględnia pełny rok, także daty po dniu bazowym |
| `InYearToDate(date, base_date, period_no[, first_month_of_year])` | Test YTD | Zwraca -1/0; do flagi 1/0 użyj `If(..., 1, 0)` |
| `InWeek(date, base_date, period_no[, first_week_day])` / `InWeekToDate(date, base_date, period_no[, first_week_day])` | Test tygodnia / WTD | Sterujesz pierwszym dniem tygodnia |
| `NetworkDays(start, end[, holiday, ...])` | Dni robocze pon.–pt., włącznie z granicami | Święta przekazujesz jako stałe tekstowe; brak konfigurowalnego weekendu |
| `Today([timer_mode])` | Bieżąca data według trybu zegara | W skrypcie `0` = poprzedni zakończony reload, `1` = bieżący reload. [Dokumentacja Today](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/DateAndTimeFunctions/today.htm) |
| `Now([timer_mode])` | Bieżący timestamp według trybu zegara | `Now(1)` do chwili wywołania w skrypcie; stały punkt odniesienia zapisz raz w LET. [Dokumentacja Now](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/DateAndTimeFunctions/now.htm) |
| `UTC()` | Bieżący czas UTC | Wspólna strefa dla metadanych ETL |
| `ConvertToLocalTime(ts[, place[, ignore_dst]])` | Konwertuje timestamp UTC/GMT na czas miejsca docelowego | Dla obsługi DST podaj obsługiwane miejsce, nie sam offset GMT. [Dokumentacja konwersji stref](https://help.qlik.com/en-us/sense/November2023/Subsystems/Hub/Content/Sense_Hub/Scripting/DateAndTimeFunctions/converttolocaltime.htm) |
| Dokumentacja katalogu | [Funkcje daty i czasu Qlik](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/DateAndTimeFunctions/date-time-functions.htm) | `date`, `base_date` i `ts` oznaczają już zinterpretowane wartości daty/czasu; `period_no` jest całkowitym przesunięciem okresu |

```qlik
SET DateFormat='YYYY-MM-DD';

DatesDemo:
LOAD
    Date(Floor(EventTS)) AS EventDate,
    Timestamp(EventTS, 'YYYY-MM-DD hh:mm:ss') AS EventTimestamp,
    MakeDate(2026, 1, 1) AS YearStartDate,
    Date(AddMonths(Floor(EventTS), 1)) AS OneMonthLater,
    Date(Floor(MonthEnd(EventTS))) AS MonthEndDate,
    NetworkDays(
        MakeDate(2026, 11, 9),
        MakeDate(2026, 11, 13),
        '2026-11-11'
    ) AS BusinessDays  // 4: święto przekazane jawnie.
;
LOAD Timestamp#('2026-11-11 14:30:00', 'YYYY-MM-DD hh:mm:ss') AS EventTS
AUTOGENERATE 1;
```

`Date()` nie jest obcięciem czasu: do dziennego klucza stosuj `Date(Floor(ts))`. Argumenty świąt `NetworkDays()` są stałymi tekstowymi; funkcja nie pobiera automatycznie świąt krajowych i nie pozwala zmienić układu weekendu. [Qlik: Date](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/FormattingFunctions/Date.htm), [Qlik: NetworkDays](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/DateAndTimeFunctions/networkdays.htm).

**Pro-Tip:** dla zmian nocnych, różnych krajów lub nietypowych weekendów zbuduj kalendarz roboczy z `IsWorkingDay` i narastającym indeksem dni roboczych. W ETL jawnie ustal strefę czasową przed wyznaczaniem dnia; samo `Timestamp()` nie konwertuje UTC na czas lokalny.

### 3.4. Agregacje statystyczne vs operacje wierszowe

`Sum`, `Avg`, `Min`, `Max`, `Median`, `Stdev` i `Fractile` agregują rekordy grupy. `RangeSum`, `RangeMax` i `RangeAvg` obliczają wynik z listy argumentów; poniżej są to kolumny jednego rekordu.

```qlik
SalesStats:
LOAD
    CustomerID,
    Sum(Amount) AS TotalAmount,
    Avg(Amount) AS AvgAmount,
    Median(Amount) AS MedianAmount,
    Stdev(Amount) AS StdDevAmount,
    Fractile(Amount, 0.95) AS P95Amount,
    Min(Amount) AS MinAmount,
    Max(Amount) AS MaxAmount,
    Count(Amount) AS NonNullAmountCount
Resident SalesRaw
GROUP BY CustomerID;

QuarterMetrics:
LOAD
    ProductID,
    RangeSum(Jan, Feb, Mar) AS QuarterTotal,
    RangeMax(Jan, Feb, Mar) AS BestMonthAmount,
    RangeAvg(Jan, Feb, Mar) AS AvgAvailableMonth
Resident BudgetWide;
```

`RangeSum(10, Null(), 5)` daje `15`, podczas gdy zwykłe dodawanie propaguje NULL. `RangeSum()` traktuje też nienumeryczny tekst jako zero — to wygoda, ale również potencjalne ukrycie błędu danych. `RangeMax()` nie jest odpowiednikiem takiej zamiany braków na zero. [Qlik: RangeSum](https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/ChartFunctions/RangeFunctions/rangesum.htm).

**Pro-Tip:** `RangeSum(Jan, Feb, Mar)` odpowiada idei `rowSums(..., na.rm=TRUE)`, a `Sum(Amount) GROUP BY CustomerID` — `summarise()`. Nie licz globalnej średniej jako średniej z miesięcznych średnich: zachowaj sumę i liczbę obserwacji do ważonej rekonstrukcji.

## 4. Transformacje i model danych

### 4.1. JOIN — fizyczne scalanie tabel

Qlik wykonuje **natural join po wszystkich polach o wspólnych nazwach**. Nie ma tu klauzuli `ON`: warunek kształtujesz przez wybór i nazwy pól. Brak wspólnych pól oznacza iloczyn kartezjański. Sam `JOIN` domyślnie jest outer join. [Qlik: JOIN](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/ScriptPrefixes/Join.htm).

| Prefiks | Wynik względem tabeli docelowej po lewej |
|---|---|
| `Inner Join (Sales)` | Tylko dopasowania z obu stron |
| `Left Join (Sales)` | Wszystkie wiersze Sales i pasujące dane dołączane |
| `Right Join (Sales)` | Wszystkie wiersze danych dołączanych i pasujące Sales |
| `Outer Join (Sales)` | Dopasowania oraz niedopasowane wiersze z obu stron |

```qlik
Sales:
LOAD OrderID, CustomerID, Amount
FROM [lib://Curated/Sales.qvd] (qvd);

// Customers.qvd musi mieć maksymalnie jeden wiersz na CustomerID.
LEFT JOIN (Sales)
LOAD CustomerID, CustomerName, Segment
FROM [lib://Curated/Customers.qvd] (qvd);

// Alternatywne warianty tego SAMEGO kroku — nie uruchamiaj wszystkich:
// INNER JOIN (Sales) LOAD CustomerID, Segment FROM [...] (qvd);
// RIGHT JOIN (Sales) LOAD CustomerID, Segment FROM [...] (qvd);
```

**Pro-Tip:** złączenie 1:N powiela fakty i może zawyżyć sumy. Sprawdź unikalność klucza po stronie wymiaru oraz liczebność przed i po złączeniu. `LOAD DISTINCT CustomerID, Segment` nie gwarantuje jednego wiersza na klienta, jeśli klient ma różne segmenty.

### 4.2. KEEP — redukcja rekordów bez scalania tabel

`KEEP` porównuje wspólne pola tak jak natural join, ale pozostawia **dwie osobne tabele**. Redukuje ich zbiory rekordów. To bliższe semijoinom i filtrowaniu istnieniem niż dołączaniu kolumn. [Qlik: KEEP](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/ScriptPrefixes/Keep.htm).

| Prefiks | Tabela istniejąca A | Nowa tabela B |
|---|---|---|
| `Inner Keep (A)` | Tylko pasujące rekordy | Tylko pasujące rekordy |
| `Left Keep (A)` | Zachowana w całości | Tylko rekordy pasujące do A |
| `Right Keep (A)` | Tylko rekordy pasujące do B | Zachowana w całości |

```qlik
Sales:
LOAD OrderID, CustomerID, Amount
FROM [lib://Curated/Sales.qvd] (qvd);

Customers:
LEFT KEEP (Sales)
LOAD CustomerID, CustomerName, Segment
FROM [lib://Curated/Customers.qvd] (qvd);

// Sales pozostaje bez zmian.
// Customers zawiera tylko klientów występujących w Sales.
// Tabele są nadal osobne; wiąże je CustomerID.
```

**Pro-Tip:** użyj KEEP, jeśli potrzebujesz przyciąć wymiar, zachowując model asocjacyjny. Nie naprawia on duplikatów w wymiarze ani relacji wiele-do-wielu; unika jedynie fizycznego rozmnożenia faktów przez JOIN.

### 4.3. CONCATENATE i NoConcatenate — kontrola dopisywania wierszy

`Concatenate` dopisuje rekordy, dopasowując pola po nazwach; brakujące pola dostają NULL. Nie usuwa duplikatów. Tabele o takim samym zestawie nazw pól i tej samej liczbie pól Qlik może skonkatenować automatycznie, nawet jeśli mają inne etykiety. [Qlik: CONCATENATE](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/ScriptPrefixes/concatenate.htm).

```qlik
FactSales:
LOAD OrderID, CustomerID, Amount
FROM [lib://Curated/Sales_2025.qvd] (qvd);

CONCATENATE (FactSales)
LOAD OrderID, CustomerID, Amount, Channel
FROM [lib://Curated/Sales_2026.qvd] (qvd);
// Rekordy z 2025 mają Channel = NULL.

// Wymuszenie osobnej tabeli przy identycznym schemacie.
SalesPositive:
NoConcatenate
LOAD *
Resident FactSales
WHERE Amount > 0;

DROP TABLE FactSales;
RENAME TABLE SalesPositive TO FactSales;
```

**Pro-Tip:** w SQL `UNION ALL` zwykle kojarzysz z dopasowaniem pozycyjnym; tutaj decydują nazwy pól. `NoConcatenate` blokuje sklejenie, ale **nie blokuje asocjacji** — dwie pozostawione kopie z wieloma wspólnymi polami nadal utworzą klucz syntetyczny.

### 4.4. Mapping LOAD i ApplyMap() — lookup bez mnożenia rekordów

Tabela `MAPPING LOAD` ma dwie kolumny: klucz i wartość. Nie staje się zwykłą tabelą modelu i jest automatycznie usuwana po wykonaniu skryptu. `ApplyMap()` dokłada jedną wartość dla klucza; brak dopasowania obsłuż trzecim argumentem.

```qlik
MapCountry:
MAPPING LOAD
    Upper(Trim(CountryCode)) AS CountryCode,
    CountryName
FROM [lib://Reference/Countries.qvd] (qvd);

Customers:
LOAD
    CustomerID,
    CountryCode,
    ApplyMap(
        'MapCountry',
        Upper(Trim(CountryCode)),
        'Nieznany kraj'
    ) AS CountryName
FROM [lib://Curated/Customers.qvd] (qvd);
```

Bez wartości domyślnej `ApplyMap()` zwraca niezmapowany argument wejściowy, a nie NULL. Nie nadawaj wynikowi tej samej nazwy co pole wejściowe użyte w mapowaniu; zastosuj nowy alias i ewentualną zmianę nazwy w kolejnym kroku. [Qlik: ApplyMap](https://help.qlik.com/en-US/sense/November2025/Subsystems/Hub/Content/Sense_Hub/Scripting/MappingFunctions/ApplyMap.htm).

**Pro-Tip:** przy relacji wiele-do-jednego i pojedynczym atrybucie mapping często jest prostszy i tańszy od JOIN. Zadbaj o jednoznaczność mapy: wiele wartości dla jednego klucza nie utworzy wielu wyników, a kolejność odczytu może zdecydować o przyjętej wartości. Dla wielu atrybutów rozważ pozostawienie osobnego wymiaru zamiast powielania ich w faktach.

### 4.5. Synthetic Keys — kontrolowany klucz złożony

Jeżeli co najmniej dwie tabele współdzielą co najmniej dwa pola, Qlik może utworzyć klucz syntetyczny `$Syn`. Nie każdy taki klucz jest błędem, ale zwykle warto jawnie ustalić relację i ziarno tabel. Zagnieżdżone klucze syntetyczne są szczególnie mocnym sygnałem problemu.

```qlik
// Oba LOAD-y muszą być wykonane w tym samym reloadzie.
FactSales:
LOAD
    OrderID,
    AutoNumber(
        Hash128(CustomerID, ProductID),
        'CustomerProduct'
    ) AS [%CustomerProductKey],
    Amount
Resident RawSales;

DimCustomerProduct:
LOAD DISTINCT
    AutoNumber(
        Hash128(CustomerID, ProductID),
        'CustomerProduct'
    ) AS [%CustomerProductKey],
    CustomerID,
    ProductID
Resident RawSales;

DROP TABLE RawSales;
// Wspólne pole FactSales i DimCustomerProduct to wyłącznie klucz.
```

`AutoNumber()` nadaje zwarte identyfikatory w obrębie wykonania. `AutoNumberHash128(CustomerID, ProductID)` jest wygodną alternatywą. **Wyników autonumeracji nie traktuj jako trwałego klucza pomiędzy niezależnymi reloadami.** Do QVD wymienianych między aplikacjami zapisuj np. `Hash128(CustomerID, ProductID)`, a kompresję do integerów wykonuj wspólnie w aplikacji docelowej. [Qlik: AutoNumberHash128](https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/Scripting/CounterFunctions/autonumberhash128.htm).

**Pro-Tip:** samo dodanie klucza złożonego nie usuwa problemu, jeśli pozostawisz wszystkie jego składowe w obu tabelach. `Hash128(a, b)` ogranicza też ryzyko kolizji wynikających z naiwnego `a & b`, np. `AB+C` kontra `A+BC`; nie oznacza matematycznej niemożliwości kolizji hasha.

### 4.6. Circular References — jednoznaczne ścieżki asocjacji

Pętla oznacza kilka ścieżek pomiędzy elementami modelu. Przykład: Sales łączy się z Customers po `CustomerID`, z Calendar po `Date`, a Customers również z Calendar po polu `Date`, które w rzeczywistości oznacza dzień rejestracji klienta.

```qlik
Sales:
LOAD
    OrderID,
    CustomerID,
    Date(Floor(OrderTS)) AS OrderDate,
    Amount
FROM [lib://Curated/Sales.qvd] (qvd);

Customers:
LOAD
    CustomerID,
    CustomerName,
    Date(Floor(SignupTS)) AS SignupDate
FROM [lib://Curated/Customers.qvd] (qvd);

// Kalendarz sprzedaży będzie miał klucz OrderDate.
// Jeśli potrzebujesz kalendarza rejestracji, użyj SignupDate
// oraz odrębnych nazw atrybutów, np. SignupYear, SignupMonth.
```

**Pro-Tip:** `AutoNumber()` kompresuje klucze, ale nie usuwa cykli w grafie asocjacji. Naprawiaj znaczenie pól i ścieżki: model gwiazdy, świadoma link table, wspólny fakt dla zgodnego ziarna lub osobne role wymiarów. Nie rozwiązuj problemu samym ukrywaniem pól albo akceptacją tabeli luźno powiązanej. [Qlik: praktyki modelowania](https://help.qlik.com/en-US/sense/November2024/pdf/Manage%20data.pdf).

## 5. Zmienne, stan i sterowanie

### 5.1. SET vs LET oraz dollar-sign expansion

`SET` przypisuje tekst. `LET` oblicza wyrażenie i przypisuje wynik. `$(vName)` wstawia zawartość zmiennej do tekstu instrukcji przed jej interpretacją; to mechanizm generowania skryptu, nie parametryzowane zapytanie SQL.

```qlik
SET vExpression = 1 + 2;   // Tekst: 1 + 2
LET vResult = 1 + 2;       // Wynik: 3

SET vQvdRoot = lib://Curated;
LET vReloadDate = Floor(Today(1));

TRACE Expression=$(vExpression);  // Expression=1 + 2
TRACE Result=$(vResult);          // Result=3

Sales:
LOAD OrderID, Amount
FROM [$(vQvdRoot)/Sales.qvd] (qvd);

// Parametryzowany fragment wyrażenia, nie SUB ani funkcja z typami.
SET vSafeRatio = If(($2) = 0, Null(), ($1) / ($2));

Ratios:
LOAD
    ProductID,
    $(vSafeRatio(Profit, Revenue)) AS Margin
Resident ProductTotals;
```

**Pro-Tip:** przy wstawianiu tekstu do SQL sam odpowiadasz za quoting i format dat. Przy liczbach ułamkowych używaj świadomie ekspansji numerycznej `$(#vNumber)` lub jawnego formatowania, aby separator dziesiętny nie zmienił składni. Makra z `$1`/`$2` są tekstowe; złożone argumenty zawierające przecinki wymagają szczególnej ostrożności.

### 5.2. If(), Pick(), Match(), WildMatch() — logika wiersza

Funkcja `If()` wybiera wartość dla rekordu, analogicznie do SQL `CASE` lub R `if_else()`. `Match()` zwraca pozycję dokładnego dopasowania, a `WildMatch()` obsługuje wzorce `*` i `?` oraz dopasowanie bez rozróżniania wielkości liter.

```qlik
OrdersClassified:
LOAD
    OrderID,
    If(Amount < 0, 'Korekta', 'Sprzedaz') AS DocumentClass,
    If(Match(Status, 'CANCELLED', 'REJECTED') > 0, 0, 1) AS IsValid,
    If(WildMatch(ProductCode, 'TEST-*', 'DEMO-*') > 0, 1, 0) AS IsTest
Resident OrdersRaw;
```

**Pick() + Match() zamiast zagnieżdżonych If().** Przy wyborze wyniku na podstawie kodu jednego pola `Match()` wyznacza pozycję dopasowania, a `Pick()` zwraca element z odpowiadającej mu listy wyników. To odpowiednik SQL `CASE Status WHEN ...`, z oddzielną listą kluczy i wyników. `Pick()` przyjmuje indeksy od 1, natomiast `Match()` zwraca 0, gdy nie znajdzie dopasowania. Wzorzec **`Pick(1 + Match(...), default, result1, result2, ...)`** rezerwuje pierwszą pozycję na wynik domyślny. [Qlik: Pick i połączenie z Match](https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/Scripting/ConditionalFunctions/pick.htm).

```qlik
// Porównanie równoważnych zapisów; OrdersRaw zawiera OrderID i Status.
OrderStatusLabels:
LOAD
    OrderID,
    StatusNormalized,

    // Klasyczny zapis: kolejne zagnieżdżone warunki.
    If(StatusNormalized = 'N', 'Nowe',
        If(StatusNormalized = 'P', 'W toku',
            If(StatusNormalized = 'C', 'Zamkniete', 'Nieznany')
        )
    ) AS StatusLabelIf,

    // Ten sam wynik: pozycje kodów odpowiadają pozycjom etykiet.
    Pick(
        1 + Match(StatusNormalized, 'N', 'P', 'C'),
        'Nieznany',  // Match=0 -> Pick=1: wynik domyślny.
        'Nowe',     // Match=1 -> Pick=2: N.
        'W toku',   // Match=2 -> Pick=3: P.
        'Zamkniete' // Match=3 -> Pick=4: C.
    ) AS StatusLabelPick;
LOAD
    OrderID,
    // NULL i puste wartości prowadzą do domyślnego wyniku.
    // Normalizacja obsługuje też małe litery i skrajne spacje.
    Coalesce(Upper(Trim(Status)), '') AS StatusNormalized
Resident OrdersRaw;
```

`Match()` rozróżnia wielkość liter; bez wcześniejszej normalizacji można użyć `MixMatch()`. Dla wzorców wykorzystaj `WildMatch()` jako indeks dla `Pick()`, ustawiając wzorce od najbardziej szczegółowych do ogólnych — pierwsze dopasowanie wyznacza wynik. [Qlik: Match i warianty dopasowania](https://help.qlik.com/en-US/sense/November2024/Subsystems/Hub/Content/Sense_Hub/Scripting/ConditionalFunctions/match.htm).

**Pro-Tip:** utrzymuj zgodną kolejność i liczbę kluczy oraz wyników. `Pick()` + `Match()` poprawia czytelność niewielkiego słownika, ale samo w sobie nie gwarantuje przyspieszenia reloadu. Dla rozbudowanej lub współdzielonej mapy wybierz `Mapping LOAD` + `ApplyMap()`; dla zakresów i złożonych predykatów `If()` nadal bywa czytelniejszy. Reguły zapisane jako dane łatwiej kontrolować i wersjonować.

### 5.3. Peek() vs Previous() — poprzedni wynik a poprzednie wejście

| Funkcja | Co odczytuje | Typowy cel |
|---|---|---|
| `Previous(expr)` | Wyrażenie na poprzednim rekordzie wejściowym danego LOAD | Różnica względem poprzedniego wejścia |
| `Peek('field', row, 'table')` | Pole z już załadowanej tabeli; domyślnie poprzedni wiersz bieżącego wyniku | Akumulacja, odczyt granic, wynik z poprzedniego wiersza |

`Peek()` numeruje wiersze od 0, a `-1` oznacza ostatni. Może czytać wcześniej obliczone pole wynikowe. `Previous()` może odwołać się także do rekordu wejściowego odrzuconego przez `WHERE`, więc nie jest wymienny z Peek na filtrowanym wyniku. [Qlik: Peek](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/InterRecordFunctions/Peek.htm), [Qlik: Previous — przykład z WHERE](https://help.qlik.com/en-US/qlikview/September2026/Subsystems/Client/Content/QV_QlikView/Scripting/InterRecordFunctions/Previous.htm).

```qlik
// SalesRaw ma numeryczny OrderDate i jednoznaczne OrderID.
SalesRunning:
NoConcatenate
LOAD
    CustomerID,
    OrderDate,
    OrderID,
    Amount,
    If(
        CustomerID = Previous(CustomerID),
        Amount - Previous(Amount),
        Null()
    ) AS DeltaAmount,
    If(
        CustomerID = Peek('CustomerID'),
        RangeSum(Peek('RunningAmount'), Amount),
        RangeSum(Amount)
    ) AS RunningAmount
Resident SalesRaw
ORDER BY CustomerID, OrderDate, OrderID;

DROP TABLE SalesRaw;

LET vFirstCustomer = Peek('CustomerID', 0, 'SalesRunning');
```

**Pro-Tip:** odpowiednik SQL `PARTITION BY` implementujesz warunkiem zmiany klucza, a `ORDER BY` — jawnie w Resident LOAD. Dodaj deterministyczny tie-breaker, np. `OrderID`; fizyczna kolejność w QVD nie powinna definiować wyniku biznesowego.

### 5.4. SCD2 — domknięcie okresów obowiązywania

Poniższy wzorzec zakłada historię na poziomie dni: maksymalnie jedną wersję klienta na dzień, numeryczne `ValidFrom`, brak NULL w kluczu. Sortowanie malejące pozwala odczytać początek następnej chronologicznie wersji z poprzedniego załadowanego rekordu.

```qlik
CustomerHistory:
NoConcatenate
LOAD
    CustomerID,
    Segment,
    Date(ValidFrom) AS ValidFrom,
    Date(
        If(
            CustomerID = Peek('CustomerID'),
            Peek('ValidFrom') - 1,
            MakeDate(2999, 12, 31)
        )
    ) AS ValidTo,
    If(CustomerID = Peek('CustomerID'), 0, 1) AS IsCurrent
Resident CustomerChanges
ORDER BY CustomerID, ValidFrom DESC;

DROP TABLE CustomerChanges;
```

**Pro-Tip:** dla timestampów przechowuj przedziały półotwarte `[ValidFrom, ValidToExclusive)`, z końcem równym początkowi kolejnej wersji. Nie odejmuj arbitralnie sekundy. To jeszcze nie kompletny proces SCD2: musisz wykrywać faktyczne zmiany atrybutów i obsługiwać zdarzenia spóźnione. Standardowy `IntervalMatch` używa granic włącznie, więc wymaga świadomego dostosowania przedziałów. [Qlik: IntervalMatch i SCD](https://help.qlik.com/en-US/sense/November2025/Subsystems/Hub/Content/Sense_Hub/LoadData/matching-intervals-to-discrete-data.htm).

### 5.5. FOR/NEXT i IF/THEN — sterowanie zadaniami ETL

Instrukcja `IF ... THEN` steruje wykonaniem całych bloków skryptu. To inne zastosowanie niż funkcja `If()` wewnątrz LOAD. Pętle są przydatne do partycji i plików.

```qlik
FOR vYear = 2024 TO 2026

    IF Alt(FileSize('lib://Landing/Sales_$(vYear).qvd'), 0) > 0 THEN

        YearSales:
        LOAD OrderID, CustomerID, Amount
        FROM [lib://Landing/Sales_$(vYear).qvd] (qvd);

        STORE YearSales
        INTO [lib://Curated/Sales_$(vYear).qvd] (qvd);

        DROP TABLE YearSales;

    ELSE
        TRACE Brak opcjonalnej partycji roku $(vYear);
    END IF

NEXT vYear
```

**Pro-Tip:** pętla po kilku plikach to orkiestracja; pętla po milionie rekordów zwykle oznacza niepotrzebną rezygnację z tabelowego LOAD. Pomijanie brakującej partycji stosuj tylko wtedy, gdy jest opcjonalna — dla obowiązkowego źródła wymagaj błędu zadania.

### 5.6. SUB/CALL — reużywalne procedury

`SUB` definiuje procedurę wykonywaną przez `CALL`; nie jest funkcją wywoływaną dla każdego rekordu w LOAD. Dodatkowe parametry formalne, których nie przekazano przy CALL, mogą pełnić rolę zmiennych lokalnych.

```qlik
SUB LogTableRows(pTable, pRows)
    // pRows nie jest przekazany w CALL, więc służy jako lokalna zmienna.
    LET pRows = NoOfRows('$(pTable)');
    TRACE Tabela=$(pTable), rekordy=$(pRows);
END SUB

// FactSales musi istnieć w tym miejscu skryptu.
CALL LogTableRows('FactSales');

SUB Increment(pValue)
    LET pValue = pValue + 1;
END SUB

LET vCounter = 0;
CALL Increment(vCounter);  // Zmienna po CALL ma wartość 1.
```

**Pro-Tip:** parametry są kopiowane do procedury, a przy przekazaniu nazwy zmiennej jej wynik może zostać skopiowany z powrotem. To inne zachowanie niż domyślne oczekiwania z funkcji R. Ogranicz globalne efekty uboczne; zmienne robocze deklaruj jako nieprzekazane parametry formalne. [Qlik: SUB](https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/Scripting/ScriptControlStatements/Sub.htm).

## 6. QVD i incremental load

### 6.1. STORE i warstwy QVD — trwałe punkty pośrednie

QVD przechowuje jedną tabelę Qlik. Jest przydatny jako warstwa ekstrakcji, wynik wspólnych transformacji i źródło dla wielu aplikacji. Typowy układ: **źródło → Raw QVD → Curated QVD → model aplikacji**. Każda warstwa powinna mieć określone ziarno, klucze i zasady odświeżania.

```qlik
LIB CONNECT TO 'DWH_SQL';

SalesExtract:
SQL SELECT OrderID, CustomerID, OrderTS, Amount, ModifiedAt
FROM dbo.Sales;

STORE SalesExtract INTO [lib://Raw/Sales.qvd] (qvd);
DROP TABLE SalesExtract;

FactSales:
LOAD
    OrderID,
    CustomerID,
    Date(Floor(OrderTS)) AS OrderDate,
    Amount,
    ModifiedAt
FROM [lib://Raw/Sales.qvd] (qvd);

STORE FactSales INTO [lib://Curated/FactSales.qvd] (qvd);
```

**Pro-Tip:** QVD nie jest bazą z `UPDATE` ani automatycznym dopisywaniem przez STORE. Zapis przebudowanego pełnego pliku nadal kosztuje I/O, nawet jeśli ze źródła pobierasz tylko deltę. Przy dużej historii rozważ partycje i przebudowę tylko tych, których dotyczą zmiany. [Qlik: QVD](https://help.qlik.com/en-US/qlikview/May2024/Subsystems/Client/Content/QV_QlikView/QVD_files.htm).

### 6.2. Optimized QVD Load — szybki odczyt bez transformacji

Silnik sam wybiera tryb optimized albo standard. Optimized wykorzystuje wewnętrzną reprezentację QVD bez zwykłego przetwarzania wartości każdego rekordu. To **cecha konkretnego odczytu**, nie specjalny typ pliku i nie gwarancja optymalności całego pipeline'u.

| Operacja podczas odczytu QVD | Praktyczna reguła |
|---|---|
| Odczyt wybranych pól bez przekształceń | Pozwala zachować optimized |
| Zmiana nazwy pola przez `AS` | Co do zasady dozwolona |
| `Date()`, `Upper()`, mnożenie, `ApplyMap()` na odczytywanych danych | Wymaga zwykłego przetwarzania |
| `WHERE Amount > 0` i inne filtry wymagające oceny wartości | Zwykle przerywają optimized |
| Prosty `WHERE Exists(Key)` | Typowy wyjątek pozwalający na optimized |
| `WHERE Not Exists(Key)` | W klasycznym merge incremental wymusza standard |
| Wyrażenia, złożone filtry i warianty `Exists(Key, expr)` | Nie zakładaj optimized; sprawdzaj log |
| Agregacja lub deduplikacja podczas odczytu | Nie projektuj ich jako prostego optimized load |

Reguły ogólne potwierdza [Qlik: wydajność modelu i QVD](https://help.qlik.com/en-US/sense/November2024/pdf/Manage%20data.pdf); zachowanie klasycznego merge opisuje [Qlik: incremental QVD](https://help.qlik.com/en-US/qlikview/May2024/Subsystems/Client/Content/QV_QlikView/QVD_Incremental.htm). Ostateczną weryfikacją konkretnej kombinacji prefiksów i filtrów jest log reloadu w używanej wersji silnika.

```qlik
// Wcześniej załadowany zbiór wartości filtra.
WantedCustomers:
LOAD DISTINCT CustomerID
FROM [lib://Reference/ActiveCustomers.qvd] (qvd);

FactSales:
LOAD
    OrderID,
    CustomerID,
    Amount
FROM [lib://Curated/Sales.qvd] (qvd)
WHERE Exists(CustomerID);

DROP TABLE WantedCustomers;

// Osobny wzorzec: szybki odczyt i transformacja w pamięci.
Stage:
LOAD OrderID, OrderTS, Amount
FROM [lib://Raw/Orders.qvd] (qvd);

Orders:
LOAD
    OrderID,
    Date(Floor(OrderTS)) AS OrderDate,
    Amount
Resident Stage;

DROP TABLE Stage;
```

**Pro-Tip:** optimized staging + Resident oznacza dodatkowy przebieg i pamięć szczytową. Może być wolniejszy od jednego standardowego odczytu przy małych danych lub dużej redukcji rekordów. Mierz czas całego reloadu i RAM, a nie tylko obecność etykiety optimized. Dla prostego `Exists` zachowaj pole filtra w odczycie i unikaj jednoczesnego przemianowywania go.

### 6.3. Exists() i Not Exists() — stan słownika wartości

`Exists(Field, expression)` sprawdza, czy wartość jest już załadowana do wskazanego pola. Bez drugiego argumentu sprawdza wartość tego pola w bieżącym rekordzie. To **nie jest** dowolny SQL `EXISTS (SELECT ...)` skorelowany po tabelach.

Uwzględnia zarówno poprzednie tabele, jak i wartości już przyjęte w bieżącym LOAD. Dlatego `WHERE Not Exists(Key)` może przepuścić tylko pierwszy z nowych rekordów tego samego klucza. [Qlik: Exists](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Scripting/InterRecordFunctions/Exists.htm).

```qlik
ExcludedKeys:
LOAD * INLINE [
ExcludedCustomerID
A
];

// Oddzielne pole wyszukiwania nie jest uzupełniane przez ten LOAD.
// Zachowujemy oba rekordy B i odrzucamy A.
AcceptedRows:
LOAD CustomerID, Amount
INLINE [
CustomerID, Amount
A, 10
B, 20
B, 30
]
WHERE NOT Exists(ExcludedCustomerID, CustomerID);

DROP TABLE ExcludedKeys;
```

**Pro-Tip:** gdy chcesz antyzłączenia względem stałego zbioru z zachowaniem duplikatów, użyj oddzielnej nazwy pola słownika jak powyżej. Gdy chcesz „pierwszy rekord na klucz”, użyj świadomie samouzupełniającego się zbioru i deterministycznego porządku. Nazwa pola jest globalna w modelu — obca tabela z tym samym polem może zmienić wynik filtra.

### 6.4. Incremental load — INSERT i UPDATE bez ponownej ekstrakcji historii

**Insert-only:** pobierz nowe rekordy po trwałym znaczniku postępu i doklej je do QVD. Jest to poprawne wyłącznie, jeśli starsze rekordy nie są zmieniane ani usuwane.

**Insert + update:** pobierz nowe wersje rekordów, następnie dołącz starą zawartość QVD tylko dla kluczy nieobecnych w delcie. Nowa wersja ma pierwszeństwo. Jest to odpowiednik upsert/MERGE odtworzonego na tabelach w pamięci.

Poniższy szablon pokazuje **kolejny reload z istniejącym QVD**. Zakłada osobny proces ETL, unikalne i niepuste `OrderID`, jedno aktualne wystąpienie klucza w `dbo.Sales` oraz w starym QVD, numeryczne timestampy dostarczane przez konektor i brak fizycznych usunięć. Granice są przykładowe; w produkcji `vFrom` pochodzi z ostatniego zatwierdzonego checkpointu, a `vTo` z uzgodnionej granicy po stronie źródła.

```qlik
SET vFrom = 2026-11-10 00:00:00;
SET vTo   = 2026-11-11 00:00:00;

LIB CONNECT TO 'DWH_SQL';

// SQL w stylu SQL Server. Inne bazy mogą wymagać jawnego CAST.
// Okno półotwarte: [vFrom, vTo).
DeltaRaw:
SQL SELECT OrderID, CustomerID, Amount, ModifiedAt
FROM dbo.Sales
WHERE ModifiedAt >= '$(vFrom)'
  AND ModifiedAt <  '$(vTo)';

IF Alt(NoOfRows('DeltaRaw'), 0) > 0 THEN

    FactSales:
    NoConcatenate
    LOAD OrderID, CustomerID, Amount, ModifiedAt
    Resident DeltaRaw;

    DROP TABLE DeltaRaw;

    // OrderID zawiera na tym etapie wyłącznie klucze delty.
    CONCATENATE (FactSales)
    LOAD OrderID, CustomerID, Amount, ModifiedAt
    FROM [lib://Curated/FactSales.qvd] (qvd)
    WHERE NOT Exists(OrderID);

ELSE

    // Pusta delta: odczyt istniejącego zbioru bez antyzłączenia.
    FactSales:
    NoConcatenate
    LOAD OrderID, CustomerID, Amount, ModifiedAt
    FROM [lib://Curated/FactSales.qvd] (qvd);

    IF NOT IsNull(TableNumber('DeltaRaw')) THEN
        DROP TABLE DeltaRaw;
    END IF

END IF

// Kandydat do publikacji. Najpierw walidacja, potem podmiana wersji.
STORE FactSales
INTO [lib://Curated/FactSales_next.qvd] (qvd);

// Checkpoint vTo zatwierdź dopiero po udanej publikacji danych.
// STORE oraz zapis checkpointu nie tworzą jednej transakcji.
```

Ten szablon scala bieżący stan, a nie historię SCD2. Pierwszy reload wymaga pełnej ekstrakcji i inicjalizacji checkpointu. Jeżeli źródłem jest CDC z wieloma zmianami tego samego klucza, najpierw wybierz ostatnią wersję według sekwencji zdarzeń; `DISTINCT` po wszystkich kolumnach tego nie załatwia. [Qlik: wzorce incremental load](https://help.qlik.com/en-US/qlikview/May2024/Subsystems/Client/Content/QV_QlikView/QVD_Incremental.htm).

**Pro-Tip:** znacznik `ModifiedAt` jest bezpieczny tylko przy gwarancjach źródła. Długie transakcje mogą zatwierdzić się później ze starszym timestampem. Preferuj CDC/commit sequence; alternatywnie stosuj uzgodnione nakładanie okien, deduplikację i okresową rekonsyliację. Zapis danych i checkpointu projektuj tak, aby przerwany reload można było powtórzyć idempotentnie. Fizyczne DELETE wymagają tombstone'ów, listy aktualnych kluczy lub odrębnej rekonsyliacji.

## 7. Kalendarz, uprawnienia i diagnostyka

### 7.1. Master Calendar z flagami YTD i MTD

Kalendarz zawiera jeden rekord na dzień, także bez transakcji. Łączy się z faktami przez dzienny klucz. Poniżej `FactSales.OrderDate` jest już wynikiem `Date(Floor(...))`. Flagi są liczone względem daty reloadu i zapisane jako `1/0`.

```qlik
LET vAsOfDate = Floor(Today(1));

CalendarBounds:
LOAD
    Min(OrderDate) AS MinDate,
    Max(OrderDate) AS MaxDate
Resident FactSales;

LET vMinDate = Floor(Peek('MinDate', 0, 'CalendarBounds'));
LET vMaxDate = Floor(Peek('MaxDate', 0, 'CalendarBounds'));
LET vHasDates = If(
    IsNull(Peek('MinDate', 0, 'CalendarBounds'))
    OR IsNull(Peek('MaxDate', 0, 'CalendarBounds')),
    0, 1
);
DROP TABLE CalendarBounds;

// Obsługa zbioru bez poprawnych dat.
IF $(vHasDates) = 0 THEN
    TRACE Brak poprawnych dat do utworzenia Master Calendar;
ELSE

    // Pełne lata: od początku pierwszego roku danych
    // do końca ostatniego roku danych lub roku raportowania.
    LET vCalendarStart = Floor(YearStart($(vMinDate)));
    LET vCalendarEnd = Floor(
        YearEnd(RangeMax($(vMaxDate), $(vAsOfDate)))
    );

    MasterCalendar:
    LOAD
        Date(CalendarDay) AS OrderDate,
        Year(CalendarDay) AS CalendarYear,
        Month(CalendarDay) AS CalendarMonth,
        Date(MonthStart(CalendarDay), 'YYYY-MM') AS CalendarYearMonth,
        'Q' & Ceil(Month(CalendarDay) / 3) AS CalendarQuarter,
        Week(CalendarDay) AS CalendarWeek,
        WeekYear(CalendarDay) AS CalendarWeekYear,
        WeekDay(CalendarDay) AS CalendarWeekDay,
        If(
            CalendarDay >= YearStart($(vAsOfDate))
            AND CalendarDay <= $(vAsOfDate),
            1, 0
        ) AS IsYTD,
        If(
            CalendarDay >= MonthStart($(vAsOfDate))
            AND CalendarDay <= $(vAsOfDate),
            1, 0
        ) AS IsMTD,
        If(CalendarDay = $(vAsOfDate), 1, 0) AS IsAsOfDate,
        If(CalendarDay > $(vAsOfDate), 1, 0) AS IsFuture;
    LOAD
        $(vCalendarStart) + IterNo() - 1 AS CalendarDay
    AUTOGENERATE 1
    WHILE $(vCalendarStart) + IterNo() - 1 <= $(vCalendarEnd);

END IF
```

`CalendarMonth` sam nie identyfikuje miesiąca w wielu latach. `CalendarYearMonth` jest dual z liczbową datą początku miesiąca. Numer tygodnia zestawiaj z `WeekYear()`, bo początek stycznia może należeć do ostatniego tygodnia poprzedniego roku.

**Pro-Tip:** flagi skryptowe zmieniają się dopiero po reloadzie. Dla raportów „stan na zamknięty dzień” ustaw `vAsOfDate` z metadanych zakończonego ładowania, zamiast bezwarunkowego `Today(1)`. Jeśli fakt ma kilka ról daty, ustal osobne kalendarze lub kontrolowany most dat; nie łącz jednego kalendarza kilkoma niezależnymi ścieżkami.

### 7.2. Section Access — autoryzacja wierszowa

Section Access definiuje tożsamości i wartości uprawnień; Section Application zawiera dane biznesowe. Wspólne pole redukcji ogranicza dostępne dane przez asocjacje. Systemowe nazwy pól i wartości w tabeli uprawnień zapisuj wielkimi literami; tekstowe wartości pola redukcji normalizuj również w danych.

Poniżej wariant **Qlik Sense Enterprise on Windows**, z identyfikatorami `DOMENA\LOGIN` i kontem harmonogramu. W Qlik Cloud identyfikację użytkownika oraz tożsamość reloadu trzeba dopasować do IdP i konfiguracji tenant; nie przenoś konta `INTERNAL\SA_SCHEDULER` w ciemno.

```qlik
SECTION ACCESS;

AccessRules:
LOAD * INLINE [
ACCESS, USERID, REDUCTION
ADMIN, CORP\QLIK_ADMIN, PL
ADMIN, CORP\QLIK_ADMIN, DE
ADMIN, INTERNAL\SA_SCHEDULER, PL
ADMIN, INTERNAL\SA_SCHEDULER, DE
USER, CORP\ANNA, PL
USER, CORP\PIOTR, DE
];

SECTION APPLICATION;

FactSales:
LOAD
    OrderID,
    CustomerID,
    Upper(Trim(CountryCode)) AS REDUCTION,
    Amount
FROM [lib://Curated/Sales.qvd] (qvd);
```

Szablon zakłada zbiór regionów PL i DE; przy rozbudowie domeny rozszerz także uprawnienia administratora i reloadu. `ADMIN` nie powinien być traktowany jako uniwersalne obejście redukcji. `*` w polu redukcji oznacza wartości wymienione w Section Access, nie automatycznie wszystkie wartości danych aplikacji. [Qlik: Section Access](https://help.qlik.com/en-us/sense/May2023/Subsystems/Hub/Content/Sense_Hub/Scripting/Security/manage-security-with-section-access.htm).

**Pro-Tip:** przetestuj pozytywne i negatywne przypadki użytkowników oraz propagację redukcji do każdej wrażliwej tabeli. Tabela odłączona od pola redukcji może pozostać nią nieobjęta. Section Access nie zastępuje ochrony samych plików QVD. Przy wczytywaniu uprawnień z QVD wymuś zwykły odczyt np. przez `Upper(USERID)`; optimized load nie jest wspierany dla takiego ładowania Section Access. [Qlik: QVD w Section Access](https://help.qlik.com/en-US/sense/November2025/pdf/Manage%20data.pdf).

### 7.3. TRACE, EXIT SCRIPT, DROP TABLE/FIELD — diagnostyka i pamięć

`TRACE` zapisuje komunikat do logu. `EXIT SCRIPT` kończy wykonanie skryptu; nie jest odpowiednikiem wyjątku ani rollbacku. `DROP TABLE` usuwa tabelę z pamięci, a `DROP FIELD ... FROM ...` wskazane pole z konkretnej tabeli; nie kasuje pliku QVD.

```qlik
LET vStartedAt = Now(1);
TRACE Start ladowania FactSales;

FactSales:
LOAD OrderID, CustomerID, Amount, TechnicalBatchID
FROM [lib://Curated/Sales.qvd] (qvd);

LET vRows = NoOfRows('FactSales');
LET vDistinctIDs = FieldValueCount('OrderID');
LET vElapsedSeconds = Round((Now(1) - vStartedAt) * 86400, 0.1);

TRACE FactSales: rows=$(vRows), distinct_ids=$(vDistinctIDs);
TRACE Czas odczytu: $(vElapsedSeconds) s;

DROP FIELD TechnicalBatchID FROM FactSales;

// Ustaw na 1 tylko podczas świadomego debugowania.
LET vDebugStop = 0;
IF $(vDebugStop) = 1 THEN
    TRACE Kontrolowane zakonczenie skryptu po zaladowaniu faktow;
    EXIT SCRIPT;
END IF
```

**Pro-Tip:** `FieldValueCount('OrderID')` dotyczy globalnego słownika pola, a nie tylko jednej tabeli. Do walidacji ziarna użyj agregacji `Resident` konkretnej tabeli. `EXIT SCRIPT` może pozostawić zadanie jako zakończone poprawnie; przy obowiązkowych walidacjach zadbaj o rzeczywisty status błędu w mechanizmie wykonania. Nie ukrywaj błędów globalnym `ErrorMode=0`.

### 7.4. Kontrola ziarna i wielkości modelu

Zacznij od zdania „jeden rekord tej tabeli oznacza…”. `OrderID` może być unikalne w nagłówkach, ale nie w pozycjach zamówienia. Dla pozycji kluczem może być `OrderID + LineID`.

```qlik
// Zakładamy, że FactSales ma jeden rekord na OrderID.
KeyCounts:
LOAD
    OrderID,
    Count(OrderID) AS KeyRows
Resident FactSales
GROUP BY OrderID;

DuplicateKeys:
NoConcatenate
LOAD OrderID, KeyRows
Resident KeyCounts
WHERE KeyRows > 1;

LET vDuplicateKeys = Alt(NoOfRows('DuplicateKeys'), 0);
TRACE Klucze z duplikatami=$(vDuplicateKeys);

DROP TABLE KeyCounts;
IF NOT IsNull(TableNumber('DuplicateKeys')) THEN
    DROP TABLE DuplicateKeys;
END IF

// NULL w kluczu sprawdzaj osobno — Count(OrderID) go nie policzy.
NullKeyCheck:
LOAD
    Sum(If(IsNull(OrderID), 1, 0)) AS NullKeyRows
Resident FactSales;

LET vNullKeyRows = Peek('NullKeyRows', 0, 'NullKeyCheck');
TRACE Rekordy bez klucza=$(vNullKeyRows);
DROP TABLE NullKeyCheck;
```

**Pro-Tip:** redukuj liczbę pól i kardynalność wartości tam, gdzie biznes nie potrzebuje pełnej precyzji. Pełne timestampy, identyfikatory techniczne i długie teksty powtarzane w faktach zwiększają pamięć. Rozdzielenie daty i czasu może pomóc, ale nie usuwaj precyzji niezbędnej do audytu, kolejności zdarzeń lub incremental load.

## Szybki wybór wzorca

| Potrzeba | Wzorzec |
|---|---|
| Odfiltrować miliony rekordów w bazie | `SQL SELECT ... WHERE ...` |
| Kilka zależnych obliczeń bez sortowania | Preceding LOAD |
| Posortować przed różnicami lub akumulacją | Resident LOAD + ORDER BY |
| Jeden atrybut po jednoznacznym kluczu | Mapping LOAD + ApplyMap |
| Zachować wymiar jako osobną tabelę i przyciąć rekordy | KEEP lub kontrolowany Exists |
| Połączyć partycje o zgodnym znaczeniu pól | Jawny CONCATENATE |
| Antyzłączenie zachowujące wiele wierszy na klucz | Not Exists z oddzielnym polem słownika |
| Trwały klucz między aplikacjami i reloadami | Hash128/Hash256 po znormalizowanych składnikach |
| Zwarty klucz w jednym modelu | AutoNumber/AutoNumberHash128 we wspólnym reloadzie |
| Aktualizacje istniejących rekordów QVD | Delta najpierw, stare rekordy bez kluczy delty |
| Przedziały historycznych atrybutów | SCD2 + dopasowanie czasu do poprawnej wersji |
| YTD/MTD niezależne od danych transakcyjnych każdego dnia | Master Calendar z jawną datą odniesienia |



