# R Cheatsheet dla analityka BI (SQL / Pandas → R)

> Ściągawka na co dzień: od podstaw języka, przez tidyverse i `data.table`, po `ggplot2`, statystykę i modele.
> Wszystkie przykłady używają **jednego, wspólnego zestawu danych** (e-commerce + logi sieciowe), zdefiniowanego w sekcji 0.
> Komentarze `# SQL:` i `# Pandas:` pokazują odpowiedniki z Twojego obecnego warsztatu.

**Wymagania wersji:** R ≥ 4.1 (pipe `|>`, lambda `\(x)`), R ≥ 4.2 (placeholder `_`), dplyr ≥ 1.1.0 (`.by`, `join_by()`, `reframe()`, `.default`).

---

## Spis treści

0. [Setup i dane przykładowe](#0-setup-i-dane-przykładowe)
1. [Podstawy i struktury danych](#1-podstawy-i-struktury-danych)
2. [Sterowanie przepływem i własne funkcje](#2-sterowanie-przepływem-i-własne-funkcje)
3. [Wejście/wyjście (I/O)](#3-wejściewyjście-io)
4. [Manipulacja danymi: dplyr + tidyr](#4-manipulacja-danymi-dplyr--tidyr)
5. [Czyszczenie danych: braki, duplikaty, tekst, daty](#5-czyszczenie-danych-braki-duplikaty-tekst-daty)
6. [Wysokowydajne przetwarzanie: data.table](#6-wysokowydajne-przetwarzanie-datatable)
7. [Wizualizacja: ggplot2](#7-wizualizacja-ggplot2)
8. [Statystyka i modelowanie](#8-statystyka-i-modelowanie)
9. [Dodatek A: tabela SQL ↔ dplyr ↔ Pandas ↔ data.table](#dodatek-a-tabela-odpowiedników)
10. [Dodatek B: pułapki R (gotchas)](#dodatek-b-pułapki-r-gotchas)
11. [Dodatek C: mini-case cyber (wykrywanie brute force)](#dodatek-c-mini-case-cyber)

---

## 0. Setup i dane przykładowe

```r
# --- Instalacja (jednorazowo) ---------------------------------------------
# install.packages(c("tidyverse", "data.table", "broom", "readxl", "writexl",
#                    "arrow", "here", "slider", "scales", "pROC"))

# --- Ładowanie (na początku każdego skryptu) ------------------------------
library(tidyverse)   # ładuje: dplyr, tidyr, readr, stringr, forcats, purrr, tibble, ggplot2, lubridate
library(data.table)  # uwaga: kolidują nazwy (np. between(), first(), last(), year()), patrz Dodatek B
library(broom)
library(scales)

set.seed(42)         # powtarzalność losowania (jak random_state w Pythonie)
```

### Dane: klienci, zamówienia, logi

```r
# --- Klienci (wymiar) -----------------------------------------------------
customers <- tibble(
  customer_id = 1:10,
  name        = c("Anna Nowak", "Jan Kowalski", "Eva Müller", "Hans Schmidt", "Petr Novák",
                  "Maria Wiśniewska", "Jozef Kováč", "Lena Fischer", "Piotr Zieliński", "Karolina Mazur"),
  country     = c("PL", "PL", "DE", "DE", "CZ", "PL", "SK", "DE", "PL", "PL"),
  segment     = rep(c("B2C", "B2B"), 5),
  signup_date = ymd("2025-01-15") + days(c(0, 30, 45, 90, 120, 150, 200, 240, 300, 330))
)

# --- Zamówienia (fakty) ---------------------------------------------------
n_orders <- 60
orders <- tibble(
  order_id    = 1001:(1000 + n_orders),
  # klienci 11 i 12 NIE istnieją w tabeli customers -> "osierocone" rekordy (demo anti_join)
  customer_id = sample(1:12, n_orders, replace = TRUE),
  order_date  = ymd("2026-01-01") + sample(0:300, n_orders, replace = TRUE),
  category    = sample(c("Electronics", "Home", "Fashion", "Books"), n_orders, replace = TRUE),
  qty         = sample(1:5, n_orders, replace = TRUE),
  revenue     = round(runif(n_orders, 20, 1500), 2),
  status      = sample(c("completed", "returned", "cancelled"), n_orders,
                       replace = TRUE, prob = c(0.8, 0.1, 0.1))
)

# Tylko zrealizowane zamówienia (będziemy używać bardzo często)
orders_ok <- orders |> filter(status == "completed")

# --- Logi sieciowe / bezpieczeństwa --------------------------------------
n_logs <- 300
logs <- tibble(
  ts     = ymd_hms("2026-11-10 08:00:00", tz = "Europe/Warsaw") + sort(sample(0:7200, n_logs, replace = TRUE)),
  src_ip = sample(c("10.0.0.5", "192.168.1.20", "203.0.113.7", "198.51.100.23", "45.33.32.156"),
                  n_logs, replace = TRUE, prob = c(.25, .25, .2, .2, .1)),
  user   = sample(c("admin", "jsmith", "akowalski", "svc_backup", "root"), n_logs, replace = TRUE),
  event  = sample(c("LOGIN_OK", "LOGIN_FAIL", "FW_DROP", "PORT_SCAN"), n_logs,
                  replace = TRUE, prob = c(.45, .35, .15, .05)),
  bytes  = rpois(n_logs, 2000)
)
```

---

## 1. Podstawy i struktury danych

### 1.1 Wektory atomowe

Wektor atomowy to **jednorodny** ciąg wartości jednego typu. W R nie ma skalarów, pojedyncza liczba to wektor długości 1.

```r
# Cztery typy, które spotkasz najczęściej:
x_dbl <- c(10.5, 20, 30)          # double (numeric), domyślny typ liczb
x_int <- c(1L, 2L, 3L)            # integer (sufiks L)
x_chr <- c("PL", "DE", "CZ")      # character
x_lgl <- c(TRUE, FALSE, NA)       # logical (zawsze pełne TRUE/FALSE, unikaj T/F)

typeof(x_dbl); class(x_chr); length(x_lgl)

# --- Koercja (wymuszanie typu): logical < integer < double < character ---
c(1, "a", TRUE)        # "1" "a" "TRUE"  -> wszystko stało się tekstem
c(1, TRUE, FALSE)      # 1 1 0           -> TRUE=1, FALSE=0 (dlatego sum(warunek) liczy spełnienia)

# --- Konstruktory --------------------------------------------------------
1:5                    # 1 2 3 4 5
seq(0, 100, by = 25)   # 0 25 50 75 100
seq_len(5)             # 1..5 (bezpieczne w pętlach, patrz sekcja 2)
rep(c("A", "B"), times = 3)   # A B A B A B
rep(c("A", "B"), each = 3)    # A A A B B B

# --- Indeksowanie: R liczy od 1 (nie od 0 jak Python)! -------------------
daily_rev <- c(Mon = 120.5, Tue = 98, Wed = 143.2, Thu = 87.9)

daily_rev[2]                  # 2. element
daily_rev[-1]                 # wszystko OPRÓCZ 1. (ujemny indeks = wykluczenie)
daily_rev[c(1, 3)]            # kilka pozycji
daily_rev["Wed"]              # po nazwie
daily_rev[daily_rev > 100]    # po warunku logicznym (jak df[df.x > 100] w Pandas)
which(daily_rev > 100)        # indeksy spełniające warunek
daily_rev[order(-daily_rev)]  # sortowanie malejąco
```

### 1.2 Wektoryzacja i recykling (kluczowa cecha R)

Operacje działają **na całych wektorach naraz**, bez pętli. To odpowiednik operacji na kolumnach w SQL/Pandas.

```r
price_net <- c(100, 250, 80, 410)
qty       <- c(2, 1, 5, 3)

price_net * 1.23              # każdy element x 1.23 (skalar rozciągnięty na wektor)
price_net * qty               # element po elemencie: 200 250 400 1230
sum(price_net * qty)          # SUM(price * qty)
price_net > 90                # wektor logiczny: TRUE TRUE FALSE TRUE
sum(price_net > 90)           # ile spełnia warunek (TRUE=1)
mean(price_net > 90)          # jaki ODSETEK spełnia warunek
any(price_net > 400); all(price_net > 50)

# Funkcje matematyczne są wektorowe: round(), log(), sqrt(), abs(), cumsum(), diff(), ...
cumsum(price_net)             # sumy narastające
diff(price_net)               # różnice kolejnych elementów (jak LAG: x - lag(x))

# --- %in% to odpowiednik SQL IN ------------------------------------------
countries <- c("PL", "DE", "US", "CZ")
countries %in% c("PL", "DE", "CZ")   # TRUE TRUE FALSE TRUE
# UWAGA: nie używaj == do porównania z wektorem (recykling zrobi niespodziankę):
countries == c("PL", "DE")           # porównuje parami, krótszy wektor jest powtarzany!

# --- Recykling (recycling): krótszy wektor jest cyklicznie powtarzany ----
c(1, 2, 3, 4) + c(10, 20)     # 11 22 13 24   (10,20,10,20): OK, bo 4 jest wielokrotnością 2
c(1, 2, 3)    + c(10, 20)     # 11 22 13 + WARNING (3 nie jest wielokrotnością 2)
# Reguła: skalar (długość 1) recyklingowany jest zawsze bezpiecznie.
# Recykling wektorów o długości > 1 to częste źródło cichych błędów.

# --- NA: brak danych "zaraża" wyniki ------------------------------------
sum(c(1, NA, 3))                 # NA
sum(c(1, NA, 3), na.rm = TRUE)   # 4
NA > 1                           # NA (nie FALSE!)
NA == NA                         # NA, więc do testu użyj is.na()
```

### 1.3 Faktory (zmienne kategoryczne)

`factor` to wektor liczb całkowitych (kody) + słownik etykiet (`levels`). Kluczowy dla modeli, wykresów i kolejności kategorii. Odpowiednik `pd.Categorical` / dictionary encoding.

```r
status <- c("new", "paid", "shipped", "paid", "new", "returned")

f_status <- factor(status, levels = c("new", "paid", "shipped", "returned"))
levels(f_status)       # etykiety w kolejności poziomów
nlevels(f_status)
as.integer(f_status)   # kody wewnętrzne: 1 2 3 2 1 4
table(f_status)        # tabela częstości (GROUP BY + COUNT); zachowa poziomy z zerem

# --- Faktor uporządkowany (ordinal): można porównywać < > --------------
size <- factor(c("M", "S", "L", "M"), levels = c("S", "M", "L"), ordered = TRUE)
size < "L"             # TRUE TRUE FALSE TRUE

# --- PUŁAPKA: konwersja faktora na liczby -----------------------------
f_num <- factor(c("10", "20", "5"))
as.numeric(f_num)                  # KODY, nie wartości!
as.numeric(as.character(f_num))    # 10 20 5

# --- Binning ciągłej zmiennej (SQL: CASE WHEN BETWEEN) -----------------
cut(orders$revenue,
    breaks = c(0, 100, 500, 1000, Inf),
    labels = c("low", "mid", "high", "premium"),
    right  = FALSE)                # przedziały [0,100), [100,500), ...

# --- forcats (tidyverse): wygodna praca z faktorami --------------------
orders |> mutate(category = factor(category)) |> pull(category) |> fct_infreq() |> levels()  # wg częstości
fct_relevel(f_status, "paid")          # przenieś "paid" na początek (zmienia poziom referencyjny w modelach)
fct_recode(f_status, "refund" = "returned")   # zmiana nazwy etykiety
fct_lump_n(factor(orders$category), n = 2, other_level = "Other")  # TOP-2 + reszta jako "Other"
droplevels(f_status[f_status != "new"])        # usuń nieużywane poziomy
# Najczęstszy użytek na wykresach: fct_reorder(category, revenue) -> sortowanie słupków (sekcja 7)
```

### 1.4 Listy

Lista może zawierać elementy **różnych typów i długości** (także inne listy i ramki). Odpowiednik słownika Pythona / obiektu JSON.

```r
alert <- list(
  id       = 101,
  src_ip   = "203.0.113.7",
  tags     = c("bruteforce", "ssh"),
  meta     = list(severity = "high", score = 8.7)
)

alert$id                  # element po nazwie -> sama wartość
alert[["tags"]]           # j.w. (przydatne, gdy nazwa jest w zmiennej)
alert$meta$severity       # zagnieżdżenie
alert["id"]               # UWAGA: pojedyncze [ ] zwraca LISTĘ (pod-listę), nie wartość
alert$resolved <- FALSE   # dodanie elementu
alert$id <- NULL          # usunięcie elementu
str(alert)                # kompaktowy podgląd struktury (Twój nowy najlepszy przyjaciel)

# [[ ]] wyciąga element, [ ] wycina fragment. Mnemonik "pociąg i wagon":
#   pociąg[1]   = pociąg z pierwszym wagonem
#   pociąg[[1]] = zawartość pierwszego wagonu
```

### 1.5 Macierze

Macierz = wektor atomowy z wymiarami (jeden typ). Używana w algebrze liniowej, korelacjach, tabelach krzyżowych.

```r
sales_m <- matrix(
  c(120, 135, 160,
    90,  110, 105),
  nrow = 2, byrow = TRUE,
  dimnames = list(country = c("PL", "DE"), quarter = c("Q1", "Q2", "Q3"))
)

sales_m["PL", "Q2"]       # 135
dim(sales_m); t(sales_m)  # wymiary; transpozycja
sales_m * 2               # element po elemencie
sales_m %*% t(sales_m)    # mnożenie macierzowe (algebra liniowa)
rowSums(sales_m); colMeans(sales_m)   # zoptymalizowane, szybsze niż apply()
apply(sales_m, 2, max)    # 2 = po kolumnach, 1 = po wierszach (więcej w sekcji 2)
```

### 1.6 `data.frame` (i `tibble`)

`data.frame` to **lista wektorów o równej długości**. Każda kolumna ma własny typ. To tabela SQL / `pd.DataFrame`.

```r
df <- data.frame(
  order_id = 1:4,
  country  = c("PL", "DE", "PL", "CZ"),
  revenue  = c(250.5, 99, 410, 75.25)
)

is.list(df)                  # TRUE: data.frame to lista kolumn
str(df)                      # typy kolumn  | Pandas: df.info()
summary(df)                  # statystyki   | Pandas: df.describe()
nrow(df); ncol(df); names(df); head(df, 3)

df$revenue                   # kolumna jako wektor           | Pandas: df["revenue"]
df[["revenue"]]              # j.w. (nazwa może być zmienną)
df[df$revenue > 100, ]       # filtr wierszy (przecinek!)    | SQL: WHERE revenue > 100
df[df$revenue > 100, c("order_id", "revenue")]   # wiersze + kolumny
df[order(-df$revenue), ]     # sortowanie                    | SQL: ORDER BY revenue DESC
df$margin <- df$revenue * 0.2   # nowa kolumna              | Pandas: df["margin"] = ...

# --- tibble: nowoczesny data.frame z tidyverse --------------------------
# Różnice: ładniejszy print (typy kolumn, tylko 10 wierszy), brak partial matching przez $,
# nigdy nie zamienia znaków na faktory, nie ma nazw wierszy.
orders          # wypisz tibble
glimpse(orders) # poziomy podgląd: typ + pierwsze wartości (świetne do szybkiej inspekcji)
as_tibble(df)   # konwersja
```

---

## 2. Sterowanie przepływem i własne funkcje

### 2.1 Instrukcje warunkowe

```r
# --- Klasyczne if / else (SKALARNE: warunek musi mieć długość 1) ----------
total_revenue <- sum(orders_ok$revenue)

if (total_revenue > 50000) {
  message("Cel osiągnięty")
} else if (total_revenue > 30000) {
  message("Blisko celu")
} else {
  message("Poniżej celu")
}
# && i || sprawdzają tylko pierwszy element i skracają ewaluację (warunki w if).
# & i | są wektorowe, używaj ich w filter()/ifelse()/mutate().
# Od R 4.2 if() z warunkiem dłuższym niż 1 daje BŁĄD (wcześniej tylko ostrzeżenie).

# --- ifelse(): wektorowy IF (SQL: CASE WHEN ... ELSE ... END) -------------
orders$size_flag <- ifelse(orders$revenue > 500, "high", "standard")
# Pułapki ifelse(): gubi klasę Date/factor (zwraca liczby/kody), a typ wyniku zależy od wartości.
# Bezpieczniejsza wersja z dplyr (ścisła kontrola typów):
orders |> mutate(signed_rev = if_else(status == "returned", -revenue, revenue))

# --- case_when(): wiele warunków (SQL: CASE WHEN ... WHEN ... ELSE ...) ---
orders |>
  mutate(
    tier = case_when(
      revenue >= 1000 ~ "A",     # wygrywa PIERWSZY spełniony warunek (kolejność ma znaczenie!)
      revenue >= 300  ~ "B",
      .default        = "C"      # ELSE (dplyr >= 1.1.0); bez tego niedopasowane wiersze to NA
    )
  ) |>
  count(tier)

# --- switch(): wybór wg wartości tekstowej (skalarny) -------------------
severity <- function(event) {
  switch(event,
         PORT_SCAN  = "critical",
         LOGIN_FAIL = "warning",
         FW_DROP    = "info",
         "unknown")               # ostatni, nienazwany argument = wartość domyślna
}
vapply(c("PORT_SCAN", "LOGIN_OK"), severity, character(1))   # switch skalarny -> opakuj w vapply
```

### 2.2 Pętle: `for`, `while` i dlaczego w R lepiej ich unikać

R jest językiem **wektorowym**: pętle w R-kodzie są wolne (interpretowane), a operacje wektorowe działają w skompilowanym C. Pętla ma sens tylko wtedy, gdy każda iteracja zależy od poprzedniej (np. symulacje, paginacja API, retry).

```r
# --- ❌ Pętla for: przeliczenie przychodu brutto (z VAT 23%) --------------
gross <- numeric(nrow(orders))                 # ZAWSZE prealokuj wynik (nie rośnij wektora w pętli)
for (i in seq_len(nrow(orders))) {             # seq_len() zamiast 1:nrow() (patrz niżej)
  gross[i] <- orders$revenue[i] * 1.23
}

# --- ✅ To samo wektorowo: jedna linia, ~100x szybciej --------------------
gross <- orders$revenue * 1.23
# dplyr: orders |> mutate(gross = revenue * 1.23)

# --- Pomiar czasu --------------------------------------------------------
x <- runif(1e6)
system.time({ out <- numeric(length(x)); for (i in seq_along(x)) out[i] <- x[i] * 1.23 })   # pętla
system.time(out <- x * 1.23)                                                                # wektorowo
# Dokładniejsze benchmarki: bench::mark(...) lub microbenchmark::microbenchmark(...)

# --- ❌ Anty-wzorzec: rosnący wektor (kopiuje pamięć przy każdej iteracji, O(n^2)) ---
res <- c()
for (i in 1:10000) res <- c(res, i^2)    # NIE RÓB TEGO

# --- Pułapka 1:length(x) --------------------------------------------------
x <- c()
for (i in 1:length(x)) print(i)          # length = 0 -> 1:0 = c(1, 0): wykona się 2 razy!
for (i in seq_along(x)) print(i)         # poprawnie: 0 iteracji

# --- while: gdy nie znasz liczby iteracji (np. retry / paginacja) ----------
attempt <- 0
repeat {                                 # repeat + break: pętla "do-while"
  attempt <- attempt + 1
  success <- runif(1) > 0.7              # symulacja wywołania API
  if (success || attempt >= 5) break
}
cat("Próby:", attempt, "\n")
```

### 2.3 Alternatywy dla pętli: rodzina `apply`

Funkcja stosowana do **każdego elementu** listy/wektora/kolumny. Gdy nie da się zwektoryzować, wybierz `apply` (lub `purrr::map`) zamiast `for`.

```r
# Zamiast pętli po grupach: split() + lapply()
by_cat <- split(orders_ok, orders_ok$category)    # lista ramek, po jednej na kategorię
names(by_cat)

# lapply(): zawsze zwraca LISTĘ
lapply(by_cat, \(d) sum(d$revenue))               # \(d) = skrót na function(d), R >= 4.1

# sapply(): próbuje uprościć wynik do wektora/macierzy (wygodne, ale typ wyniku bywa nieprzewidywalny)
sapply(by_cat, \(d) sum(d$revenue))

# vapply(): jak sapply, ale z KONTROLĄ typu wyniku (bezpieczne w kodzie produkcyjnym)
vapply(by_cat, \(d) sum(d$revenue), FUN.VALUE = numeric(1))

# tapply(): agregacja wektora po grupach (SQL: GROUP BY)
tapply(orders_ok$revenue, orders_ok$category, sum)

# apply(): po wierszach (1) lub kolumnach (2) macierzy; na macierzach wolisz rowSums()/colMeans()
apply(sales_m, 1, sum)

# Map()/mapply(): iteracja po kilku wektorach równolegle
mapply(\(q, r) q * r, orders$qty, orders$revenue)   # = orders$qty * orders$revenue (wektorowo lepiej)

# Budowanie ramki z listy wyników
do.call(rbind, lapply(by_cat, \(d) data.frame(n = nrow(d), revenue = sum(d$revenue))))

# --- ✅ Wersja tidyverse (docelowo najczytelniejsza) ----------------------
orders_ok |> summarise(n = n(), revenue = sum(revenue), .by = category)
# purrr (typowane odpowiedniki apply): map(), map_dbl(), map_chr(), map_lgl(), map_dfr()/list_rbind()
by_cat |> map_dbl(\(d) sum(d$revenue))

# Pułapka sapply(): dla pustego wejścia zwraca list(), a nie wektor -> w funkcjach używaj vapply()
```

### 2.4 Własne funkcje

```r
# Argumenty domyślne, walidacja, wartość zwracana
calc_margin <- function(revenue, cost_ratio = 0.7, round_to = 2) {
  stopifnot(is.numeric(revenue), cost_ratio >= 0, cost_ratio <= 1)   # asercje: szybkie "fail fast"
  margin <- revenue * (1 - cost_ratio)
  round(margin, round_to)        # OSTATNIE WYRAŻENIE jest zwracane (return() nie jest potrzebne)
}

calc_margin(1000)                          # 300 (cost_ratio = 0.7)
calc_margin(1000, cost_ratio = 0.8)        # argument nazwany
calc_margin(c(100, 250, 80))               # funkcja jest WEKTOROWA, bo korzysta z operacji wektorowych

# --- Wczesny return() i zwracanie wielu wartości (przez listę) -----------
safe_div <- function(a, b) {
  if (b == 0) return(NA_real_)     # return() wyłącznie dla wczesnego wyjścia
  a / b
}

order_kpis <- function(df) {
  list(
    n_orders = nrow(df),
    revenue  = sum(df$revenue),
    aov      = mean(df$revenue)    # Average Order Value
  )
}
kpi <- order_kpis(orders_ok)
kpi$aov

# --- Argument ... (przekazywanie dodatkowych argumentów dalej) -----------
mean_safe <- function(x, ...) mean(x, na.rm = TRUE, ...)
mean_safe(c(1, NA, 3), trim = 0.1)

# --- Funkcje dla dplyr: operator "curly-curly" {{ }} ---------------------
# pozwala przekazać NAZWĘ KOLUMNY jako argument (tzw. tidy evaluation)
top_n_by <- function(df, group_col, value_col, n = 3) {
  df |>
    slice_max({{ value_col }}, n = n, by = {{ group_col }}) |>
    arrange({{ group_col }}, desc({{ value_col }}))
}
top_n_by(orders_ok, category, revenue, n = 2)    # TOP 2 zamówienia w każdej kategorii

# --- Obsługa błędów ------------------------------------------------------
result <- tryCatch(
  { log("tekst") },                                   # kod, który może się wywalić
  error   = function(e) { message("Błąd: ", conditionMessage(e)); NA },
  warning = function(w) { message("Ostrzeżenie: ", conditionMessage(w)); NA },
  finally = message("Zawsze się wykona")              # sprzątanie
)
# purrr::possibly()/safely() opakowują funkcję tak, by nie przerywała map().
```

---

## 3. Wejście/wyjście (I/O)

| Funkcja | Pakiet | Zwraca | Uwagi |
|---|---|---|---|
| `read.csv()` | base | data.frame | wolna, nie polecana do dużych plików |
| `read_csv()` | readr (tidyverse) | tibble | szybka, dobre wykrywanie typów, czytelne komunikaty |
| `fread()` | data.table | data.table | **najszybsza** (wielowątkowa), automatycznie wykrywa separator |
| `readRDS()` | base | dowolny obiekt R | zachowuje typy, faktory i daty 1:1 |
| `read_parquet()` | arrow | tibble | format kolumnowy, idealny do dużych danych (jak w hurtowniach) |

```r
library(readr); library(data.table)

# ============ readr (tidyverse) ==========================================
df <- read_csv("data/orders.csv")                # Pandas: pd.read_csv()

df <- read_csv(
  "data/orders.csv",
  col_types = cols(                               # jawne typy (zalecane w pipeline'ach produkcyjnych)
    order_id   = col_integer(),
    order_date = col_date(format = "%Y-%m-%d"),
    revenue    = col_double(),
    .default   = col_character()
  ),
  na     = c("", "NA", "N/A", "null"),           # co traktować jako brak danych
  n_max  = 1000                                    # tylko pierwsze 1000 wierszy (szybki podgląd)
)
problems(df)                                       # wiersze, które nie sparsowały się zgodnie z typem

# --- Polski Excel / CSV: średnik, przecinek dziesiętny, kodowanie Windows-1250 ---
read_csv2("data/raport_pl.csv")                    # separator ";" + decimal ","
read_delim(
  "data/raport_pl.csv", delim = ";",
  locale = locale(decimal_mark = ",", grouping_mark = " ", encoding = "Windows-1250")
)

# --- Wiele plików naraz (np. dzienne logi) -------------------------------
files    <- list.files("logs/", pattern = "\\.csv$", full.names = TRUE)
all_logs <- read_csv(files, id = "source_file")    # readr przyjmuje wektor plików i dodaje kolumnę ze źródłem
# data.table: rbindlist(lapply(files, fread), idcol = "source_file")

write_csv(orders, "out/orders_clean.csv")

# ============ data.table::fread() (największa wydajność) ================
DT <- fread("data/orders.csv")                     # wynik: data.table (rozszerzony data.frame)
DT <- fread("data/big.csv",
            select = c("order_id", "revenue"),     # wczytaj tylko potrzebne kolumny (oszczędza RAM i czas)
            nrows  = 1e6,
            dec    = ",", encoding = "UTF-8",
            na.strings = c("", "NA"))
fwrite(DT, "out/orders.csv")                       # zapis też jest wielowątkowy i bardzo szybki

# ============ Natywne pliki R: .rds ======================================
saveRDS(orders, "out/orders.rds")                  # jeden obiekt, zachowuje typy kolumn, faktory, daty, atrybuty
orders_back <- readRDS("out/orders.rds")           # przypisujesz do DOWOLNEJ nazwy
identical(orders, orders_back)                     # TRUE
# CSV traci typy (data -> tekst, faktor -> tekst); .rds nie. Dobre do cache'owania wyników etapów.
# .RData (save()/load()) zapisuje wiele obiektów, ale load() "wstrzykuje" nazwy do środowiska, więc unikaj.

# ============ Excel =======================================================
readxl::read_excel("data/budget.xlsx", sheet = "2026", range = "A1:F100", skip = 0)
writexl::write_xlsx(list(Orders = orders, Customers = customers), "out/report.xlsx")   # wiele arkuszy

# ============ Parquet (kolumnowy; Twoja hurtownia też to lubi) ===========
arrow::write_parquet(orders, "out/orders.parquet")
orders_pq <- arrow::read_parquet("out/orders.parquet")
# Dane większe niż RAM: arrow::open_dataset("folder/") |> filter(...) |> collect()   (lazy, jak SQL)

# ============ Bazy SQL: piszesz dplyr, R generuje SQL ====================
# con <- DBI::dbConnect(odbc::odbc(), dsn = "dwh")
# DBI::dbGetQuery(con, "SELECT TOP 10 * FROM orders")       # surowy SQL
# tbl(con, "orders") |> filter(status == "completed") |> summarise(rev = sum(revenue)) |> show_query()
# ... |> collect()     # dopiero collect() ściąga dane do pamięci R (dbplyr tłumaczy dplyr -> SQL)

# ============ Ścieżki =====================================================
here::here("data", "orders.csv")     # ścieżka względem korzenia projektu (.Rproj), niezależna od getwd()
```

---

## 4. Manipulacja danymi: dplyr + tidyr

### 4.1 Operator potoku (`|>` i `%>%`)

Potok przekazuje wynik poprzedniego kroku jako **pierwszy argument** następnego. Łańcuch potoków czyta się jak sekwencję CTE w SQL lub method chaining w Pandas.

```r
# Pandas: df.query(...).groupby(...).agg(...)   |   SQL: WITH step1 AS (...), step2 AS (...) SELECT ...
orders |>
  filter(status == "completed") |>
  summarise(total = sum(revenue))

# Skrót w RStudio: Ctrl+Shift+M. Ustawienie "Use native pipe operator" w Global Options -> Code.
```

- **`|>` (natywny, R ≥ 4.1)**: bez zależności, minimalny narzut, wymaga nawiasów: `x |> f()`. Placeholder `_` (R ≥ 4.2) tylko jako **argument nazwany**.
- **`%>%` (magrittr)**: placeholder `.`, może być użyty w kilku miejscach, pozwala budować funkcje z potoku.

```r
orders %>% lm(revenue ~ qty, data = .)     # magrittr: kropka jako placeholder
orders |>  lm(revenue ~ qty, data = _)     # natywny: podkreślnik, tylko nazwany argument
```

### 4.2 Podstawowe czasowniki: `select`, `filter`, `mutate`, `arrange`

```r
# ---- select: wybór kolumn (SQL: SELECT col1, col2) ------------------------
orders |> select(order_id, revenue)
orders |> select(-status)                      # wszystko oprócz status
orders |> select(order_id:category)            # zakres kolumn
orders |> select(starts_with("order"))         # pomocnicze: starts_with/ends_with/contains/matches
orders |> select(where(is.numeric))            # wg typu kolumny
orders |> select(any_of(c("order_id", "nieistniejaca")))   # ignoruje brakujące (all_of() zgłosi błąd)
orders |> rename(net_revenue = revenue)        # zmiana nazwy (nowa = stara)
orders |> relocate(revenue, .after = order_id) # zmiana kolejności kolumn

# ---- filter: wiersze (SQL: WHERE) -----------------------------------------
orders |> filter(status == "completed", revenue > 500)     # przecinek = AND
orders |> filter(status == "returned" | qty >= 4)          # OR
orders |> filter(category %in% c("Home", "Books"))         # IN
orders |> filter(between(revenue, 100, 500))               # BETWEEN (obustronnie włącznie)
orders |> filter(!is.na(revenue), str_detect(category, "^E"))
# filter() odrzuca wiersze, gdzie warunek = NA (tak samo jak WHERE w SQL)

# ---- mutate: nowe/zmienione kolumny (SQL: wyrażenia w SELECT; Pandas: assign) ---
orders |>
  mutate(
    vat           = revenue * 0.23,
    revenue_gross = revenue + vat,                  # można używać kolumny utworzonej linię wyżej
    order_month   = floor_date(order_date, "month"),
    price_unit    = revenue / qty
  )
orders |> mutate(across(c(revenue, qty), \(x) x * 2))                          # across: ta sama funkcja na wielu kolumnach
orders |> mutate(across(where(is.character), str_to_upper))                    # na wszystkich kolumnach tekstowych
orders |> transmute(order_id, revenue_gross = revenue * 1.23)                  # mutate + zachowaj tylko wymienione

# ---- arrange: sortowanie (SQL: ORDER BY) ---------------------------------
orders |> arrange(desc(revenue))
orders |> arrange(category, desc(revenue))
orders |> slice_max(revenue, n = 5)       # TOP 5 (SQL: ORDER BY ... DESC LIMIT 5)
orders |> slice_head(n = 5)               # pierwsze 5 (LIMIT 5)
orders |> slice_sample(n = 5)             # losowe 5
```

### 4.3 Grupowanie i agregacja

```r
# SQL: SELECT category, COUNT(*) AS n_orders, COUNT(DISTINCT customer_id), SUM(revenue) ... GROUP BY category
# Pandas: df.groupby("category").agg(n_orders=("order_id","count"), ...)
orders_ok |>
  group_by(category) |>
  summarise(
    n_orders      = n(),                      # COUNT(*)
    n_customers   = n_distinct(customer_id),  # COUNT(DISTINCT customer_id)
    total_revenue = sum(revenue),
    aov           = mean(revenue),            # UWAGA: nie nazywaj wyniku "revenue" i potem używaj revenue w kolejnej linii:
                                              # w summarise() nowa kolumna "przysłania" starą i mean(revenue) policzy z agregatu!
    p90_revenue   = quantile(revenue, 0.9),
    .groups = "drop"                          # zdejmij grupowanie po summarise (unikaj niespodzianek)
  ) |>
  arrange(desc(total_revenue))

# Nowsza składnia: .by (zamiast group_by + ungroup), od dplyr 1.1.0
orders_ok |> summarise(total_revenue = sum(revenue), .by = c(category, status))

# HAVING = filter() PO summarise()
orders_ok |>
  summarise(total_revenue = sum(revenue), n = n(), .by = customer_id) |>
  filter(n >= 3)                              # SQL: ... GROUP BY customer_id HAVING COUNT(*) >= 3

# count(): skrót na group_by + summarise(n = n())
orders |> count(status)                       # SQL: SELECT status, COUNT(*) ... GROUP BY status
orders |> count(category, status, sort = TRUE)
orders |> count(category, wt = revenue)       # suma ważona (SUM(revenue)) zamiast liczby wierszy
orders |> add_count(customer_id, name = "n_orders_customer")   # COUNT(*) OVER (PARTITION BY customer_id)

# Wiele kolumn i wiele funkcji naraz
orders_ok |>
  summarise(across(c(revenue, qty), list(sum = sum, avg = mean), .names = "{.col}_{.fn}"),
            .by = category)
```

### 4.4 Łączenie tabel (JOIN-y)

```r
# SQL: SELECT * FROM orders o LEFT JOIN customers c ON o.customer_id = c.customer_id
# Pandas: orders.merge(customers, on="customer_id", how="left")
orders |> left_join(customers, by = "customer_id")

# Klucze o różnych nazwach / wielokolumnowe:
#   by = c("customer_id" = "id")                      (klasycznie)
#   by = join_by(customer_id == id)                   (nowoczesnie, dplyr >= 1.1.0)
#   by = join_by(customer_id, country)                (klucz złożony)

orders |> inner_join(customers, by = join_by(customer_id))    # INNER JOIN: tylko pasujące
orders |> full_join(customers,  by = join_by(customer_id))    # FULL OUTER JOIN
orders |> right_join(customers, by = join_by(customer_id))    # RIGHT JOIN

# ---- Filtrujące joiny (nie dodają kolumn!) ---------------------------------
# anti_join = WHERE NOT EXISTS (...): zamówienia bez klienta w słowniku, czyli kontrola jakości danych (orphans)
orders |> anti_join(customers, by = join_by(customer_id))
# semi_join = WHERE EXISTS (...): zamówienia tylko klientów z PL
orders |> semi_join(filter(customers, country == "PL"), by = join_by(customer_id))

# ---- Zabezpieczenie przed eksplozją wierszy (duplikaty klucza po prawej stronie) ----
orders |> left_join(customers, by = join_by(customer_id), relationship = "many-to-one")
# jeśli klucz w customers nie jest unikalny, dostaniesz błąd zamiast cichego rozmnożenia wierszy.
# Dodatkowo: unmatched = "error" przerwie, gdy któryś wiersz nie ma dopasowania.

# ---- Joiny nierównościowe (SQL: JOIN ... ON o.order_date >= c.signup_date) ----
orders |> inner_join(customers, by = join_by(customer_id, order_date >= signup_date))
# Dostępne też: between(), overlaps(), closest(): np. dopasowanie do najbliższej daty (as-of join).

# ---- Kolizje nazw kolumn ---------------------------------------------------
orders |> left_join(customers, by = "customer_id", suffix = c("_ord", "_cust"))

# ---- Operacje zbiorowe -----------------------------------------------------
bind_rows(orders_ok, filter(orders, status == "returned"), .id = "src")   # UNION ALL (dopasowanie po nazwach kolumn)
union(df1, df2); intersect(df1, df2); setdiff(df1, df2)                    # UNION / INTERSECT / EXCEPT (z deduplikacją)
```

### 4.5 Funkcje okna (window functions)

Odpowiednik `OVER (PARTITION BY ... ORDER BY ...)`: **`group_by()` + `mutate()`** (zamiast `summarise()`, wiersze zostają).

```r
# UWAGA: w odróżnieniu od SQL, kolejność wierszy dla lag()/lead()/cumsum() wynika z KOLEJNOŚCI W RAMCE,
# a nie z ORDER BY w OVER(). Najpierw arrange(), potem funkcje okna (albo argument order_by).

orders_ok |>
  group_by(customer_id) |>                                 # PARTITION BY customer_id
  arrange(order_date, .by_group = TRUE) |>                 # ORDER BY order_date
  mutate(
    order_seq       = row_number(),                        # ROW_NUMBER()
    rank_by_rev     = min_rank(desc(revenue)),             # RANK()        (remisy -> luka w numeracji)
    dense_by_rev    = dense_rank(desc(revenue)),           # DENSE_RANK()  (remisy -> bez luk)
    pct_rank        = percent_rank(revenue),               # PERCENT_RANK()
    quartile        = ntile(revenue, 4),                   # NTILE(4)
    prev_order_date = lag(order_date),                     # LAG(order_date, 1)
    next_revenue    = lead(revenue),                       # LEAD(revenue, 1)
    days_since_prev = as.integer(order_date - prev_order_date),
    cum_revenue     = cumsum(revenue),                     # SUM(revenue) OVER (... ROWS UNBOUNDED PRECEDING)
    share_of_cust   = revenue / sum(revenue),              # SUM(revenue) OVER (PARTITION BY customer_id)
    first_order     = first(order_date)                    # FIRST_VALUE(...)
  ) |>
  ungroup()

# ---- TOP N w grupie (klasyk: ROW_NUMBER() ... WHERE rn <= N) ----------------
orders_ok |> slice_max(revenue, n = 3, by = category)                       # 3 największe zamówienia per kategoria
orders_ok |> slice_max(revenue, n = 1, by = customer_id, with_ties = FALSE) # dokładnie 1 wiersz na grupę

# ---- Szereg czasowy: MoM i narastające YTD ---------------------------------
monthly <- orders_ok |>
  mutate(month = floor_date(order_date, "month")) |>
  summarise(revenue = sum(revenue), .by = month) |>
  arrange(month) |>                                       # KLUCZOWE: bez tego lag() liczy po przypadkowej kolejności
  mutate(
    prev_month = lag(revenue),
    mom_pct    = revenue / prev_month - 1,                # zmiana m/m
    ytd        = cumsum(revenue)                          # SUM() OVER (ORDER BY month)
  )
# Alternatywa niezależna od kolejności wierszy: lag(revenue, order_by = month)

# ---- Średnia krocząca (SQL: AVG() OVER (ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)) ----
monthly |> mutate(ma3 = slider::slide_dbl(revenue, mean, .before = 2, .complete = TRUE))
# Pandas: s.rolling(3).mean() | data.table: frollmean(revenue, 3)
```

### 4.6 Reshaping: `pivot_longer()` i `pivot_wider()`

```r
monthly_cat <- orders_ok |>
  mutate(month = floor_date(order_date, "month")) |>
  summarise(revenue = sum(revenue), .by = c(month, category))     # format "długi" (tidy): 1 wiersz = miesiąc x kategoria

# ---- LONG -> WIDE (SQL: SUM(CASE WHEN category = 'X' THEN revenue END); Pandas: pivot_table) ----
wide <- monthly_cat |>
  pivot_wider(
    names_from  = category,       # wartości z tej kolumny staną się nazwami nowych kolumn
    values_from = revenue,        # z tej kolumny trafią wartości do komórek
    values_fill = 0,              # brakujące kombinacje -> 0 zamiast NA
    values_fn   = sum             # jeśli kombinacji jest wiele (agregacja przy pivocie)
  ) |>
  arrange(month)

# ---- WIDE -> LONG (UNPIVOT; Pandas: melt) ----------------------------------
long <- wide |>
  pivot_longer(
    cols      = -month,           # które kolumny "roztopić" (tu: wszystkie oprócz month)
    names_to  = "category",
    values_to = "revenue"
  )
# Format długi jest wymagany przez ggplot2; szeroki jest czytelniejszy w raportach.

# ---- Przydatne dodatki tidyr -----------------------------------------------
monthly_cat |> complete(month, category, fill = list(revenue = 0))      # uzupełnij brakujące kombinacje (siatka dat x kategorii)
customers |> separate_wider_delim(name, delim = " ", names = c("first_name", "last_name"))   # split kolumny
customers |> unite("country_segment", country, segment, sep = "_")                           # CONCAT kolumn
```

---

## 5. Czyszczenie danych: braki, duplikaty, tekst, daty

### 5.1 Braki danych (`NA`)

```r
df_na <- tibble(
  order_id = 1:6,
  revenue  = c(100, NA, 250, NA, 80, 300),
  country  = c("PL", NA, "DE", "DE", "", "CZ"),
  segment  = c("B2C", "B2B", NA, NA, "B2C", "B2B")
)

# ---- Diagnoza (SQL: WHERE col IS NULL / COUNT(*) - COUNT(col)) --------------
is.na(df_na$revenue)                              # wektor logiczny
sum(is.na(df_na$revenue))                         # ile braków w kolumnie
colSums(is.na(df_na))                             # braki w KAŻDEJ kolumnie
df_na |> summarise(across(everything(), \(x) sum(is.na(x))))   # to samo w tidyverse
df_na |> summarise(across(everything(), \(x) mean(is.na(x))))  # odsetek braków (profil jakości danych)
df_na |> filter(is.na(revenue))                   # wiersze z brakiem

# ---- Usuwanie / uzupełnianie ------------------------------------------------
df_na |> drop_na()                                # usuń wiersze z DOWOLNYM NA (Pandas: dropna())
df_na |> drop_na(revenue)                         # tylko wg wybranych kolumn

df_na |> replace_na(list(revenue = 0, segment = "UNKNOWN"))   # Pandas: fillna({...}) | SQL: COALESCE
df_na |> mutate(revenue = coalesce(revenue, mean(revenue, na.rm = TRUE)))   # imputacja średnią (COALESCE)
df_na |> mutate(country = na_if(country, ""))     # pusty tekst -> NA (odwrotność replace_na)
df_na |> fill(country, .direction = "down")       # LOCF: przenieś ostatnią znaną wartość w dół (jak ffill())

# ---- Semantyka NA (ważne!) --------------------------------------------------
NA == NA            # NA   -> testuj is.na(), nie == NA
NA & FALSE          # FALSE (bo wynik jest znany niezależnie od NA)
NA | TRUE           # TRUE
mean(c(1, NA, 3))   # NA, więc dodaj na.rm = TRUE
# NULL (pusty obiekt, "nic") to NIE to samo co NA (brakująca wartość w wektorze).
```

### 5.2 Duplikaty

```r
customers_raw <- tibble(
  customer_id = c(1, 2, 2, 3, 3, 3),
  email       = c("a@x.pl", "b@x.pl", "b@x.pl", "c@x.de", "c@y.de", "c@z.de"),
  updated_at  = ymd(c("2026-01-01", "2026-02-01", "2026-02-15", "2026-03-01", "2026-04-01", "2026-05-01"))
)

# ---- dplyr: distinct() ------------------------------------------------------
customers_raw |> distinct()                             # usuń w pełni identyczne wiersze (SELECT DISTINCT *)
customers_raw |> distinct(customer_id)                  # zostaje TYLKO kolumna customer_id (SELECT DISTINCT customer_id)
customers_raw |> distinct(customer_id, .keep_all = TRUE)  # unikalne wg klucza, ale zostaw PIERWSZY wiersz ze wszystkimi kolumnami

# ---- "Zostaw najnowszy rekord na klienta" (klasyk: ROW_NUMBER() ... ORDER BY updated_at DESC, rn = 1) ----
customers_raw |>
  arrange(desc(updated_at)) |>                          # najnowsze na górze
  distinct(customer_id, .keep_all = TRUE)               # distinct zachowuje pierwszy napotkany = najnowszy
# lub: slice_max(updated_at, n = 1, by = customer_id)

# ---- Znajdź duplikaty (zamiast je usuwać) -----------------------------------
customers_raw |> add_count(customer_id) |> filter(n > 1)         # wszystkie wiersze z powtórzonym kluczem
customers_raw |> filter(duplicated(customer_id) | duplicated(customer_id, fromLast = TRUE))   # j.w. w base R
anyDuplicated(customers_raw$customer_id) > 0                      # szybki test: czy klucz jest unikalny?
customers_raw |> count(customer_id) |> filter(n > 1)              # SQL: GROUP BY ... HAVING COUNT(*) > 1

# ---- data.table: unique() z argumentem `by` (bardzo wydajne dla dużych tabel) ---
DT <- as.data.table(customers_raw)
unique(DT)                                  # wiersze unikalne w całości
unique(DT, by = "customer_id")              # pierwszy rekord na klucz
unique(DT, by = "customer_id", fromLast = TRUE)   # ostatni rekord na klucz
setorder(DT, -updated_at); unique(DT, by = "customer_id")   # najnowszy na klucz
uniqueN(DT, by = "customer_id")             # COUNT(DISTINCT customer_id)
duplicated(DT, by = "customer_id")          # wektor logiczny: czy wiersz jest powtórką
```

### 5.3 Tekst i RegEx (`stringr`)

Wszystkie funkcje `stringr` zaczynają się od `str_`, **pierwszym argumentem jest wektor tekstu**, a wzorzec to RegEx (silnik ICU). W stringach R backslash trzeba podwajać (`"\\d+"`), a od R 4.0 można użyć **raw string**: `r"(\d+)"`.

```r
emails <- c("  Jan.Kowalski@Firma.PL ", "anna@shop.de", "bad-email", NA)

# ---- Czyszczenie ------------------------------------------------------------
str_trim(emails)                         # obcina spacje na brzegach       | SQL: TRIM()     | Pandas: .str.strip()
str_squish(emails)                       # j.w. + redukuje wielokrotne spacje w środku
str_to_lower(emails)                     # LOWER() | str_to_upper() | str_to_title()
str_length(emails)                       # LEN()
str_sub("PL-2026-0042", 4, 7)            # "2026"  (SUBSTRING)
str_pad("42", width = 6, pad = "0")      # "000042"

# ---- Wykrywanie (SQL: LIKE / REGEXP) ---------------------------------------
str_detect(emails, "@")                  # wektor logiczny (NA dla NA)
str_detect(emails, regex("firma", ignore_case = TRUE))
str_detect(emails, fixed("."))           # fixed() = dosłowny tekst (bez RegEx, szybciej)
str_starts(emails, "anna"); str_ends(emails, "\\.de")

# ---- Zamiana (SQL: REPLACE / REGEXP_REPLACE) -------------------------------
str_replace(emails, "\\.", "_")          # tylko PIERWSZE wystąpienie
str_replace_all(emails, "[aeiou]", "*")  # wszystkie wystąpienia
str_remove(emails, "\\s+")               # usuń dopasowanie (replace z "")
str_replace_all("a-b_c", c("-" = " ", "_" = " "))   # wiele zamian naraz (nazwany wektor)

# ---- Ekstrakcja (SQL: REGEXP_SUBSTR; Pandas: .str.extract) ------------------
emails |> str_trim() |> str_extract("(?<=@).+$")         # domena: tekst po @ (lookbehind)
str_extract_all("a 10.0.0.5 b 8.8.8.8", r"(\b(?:\d{1,3}\.){3}\d{1,3}\b)")   # LISTA wszystkich IP w tekście
str_match("PL-2026-0042", "([A-Z]{2})-(\\d{4})-(\\d+)")  # macierz: pełne dopasowanie + grupy przechwytujące
str_split_fixed("PL-2026-0042", "-", n = 3)               # split na kolumny
str_count("a,b,c,d", ",")                                 # ile wystąpień

# ---- Sklejanie --------------------------------------------------------------
str_c("ORD-", orders$order_id[1:3])                       # CONCAT (NA propaguje się; paste0 zamienia NA na "NA")
str_glue("Zamówienie {orders$order_id[1]} na kwotę {orders$revenue[1]} PLN")   # interpolacja jak f-string
str_flatten(c("a", "b", "c"), collapse = ", ")            # wektor -> jeden tekst (STRING_AGG / ", ".join)
```

#### Case study: parsowanie surowych logów (cyber)

```r
raw_logs <- c(
  '2026-11-10 08:14:02 203.0.113.7 "POST /wp-login.php HTTP/1.1" 401 532',
  '2026-11-10 08:14:05 203.0.113.7 "POST /wp-login.php HTTP/1.1" 401 532',
  '2026-11-10 08:15:11 10.0.0.5 "GET /dashboard HTTP/1.1" 200 10432',
  '2026-11-10 08:16:40 198.51.100.23 "GET /admin/../../etc/passwd HTTP/1.1" 403 211',
  '2026-11-10 08:17:03 45.33.32.156 "GET /search?q=1%27%20OR%201=1-- HTTP/1.1" 500 87'
)

# Raw string r"-(...)-" pozwala pisać RegEx bez podwajania backslashy i bez kolizji z cudzysłowami
log_pattern <- r"-(^(\S+ \S+) (\S+) "(\S+) (\S+) [^"]+" (\d{3}) (\d+)$)-"

m <- str_match(raw_logs, log_pattern)      # kolumna 1 = całość, 2..7 = grupy przechwytujące

parsed <- tibble(
  ts     = ymd_hms(m[, 2], tz = "Europe/Warsaw"),
  src_ip = m[, 3],
  method = m[, 4],
  path   = m[, 5],
  status = as.integer(m[, 6]),
  bytes  = as.integer(m[, 7])
)

# Sygnatury podejrzanych żądań (SQLi, path traversal, brute force na logowaniu)
parsed |>
  mutate(
    threat = case_when(
      str_detect(path, regex("union.+select|%27|--", ignore_case = TRUE)) ~ "SQL injection",
      str_detect(path, r"(\.\./|etc/passwd)")                              ~ "Path traversal",
      str_detect(path, "wp-login|/admin") & status %in% c(401, 403)        ~ "Auth probing",
      .default = "clean"
    )
  )

# Alternatywa: tidyr::extract() robi to samo bezpośrednio na kolumnach ramki
tibble(line = raw_logs) |>
  extract(line, into = c("ts", "src_ip", "method", "path", "status", "bytes"),
          regex = log_pattern, convert = TRUE)
```

### 5.4 Daty i czas (`lubridate`)

```r
# ---- Parsowanie: nazwa funkcji = KOLEJNOŚĆ elementów daty w tekście ---------
ymd("2026-11-10")                    # rok-miesiąc-dzień (separatory dowolne: 2026/11/10, 20261110)
dmy("10.11.2026")                    # dzień-miesiąc-rok (polski format)
mdy("11/10/2026")                    # miesiąc-dzień-rok (US)
ymd_hms("2026-11-10 14:35:20", tz = "Europe/Warsaw")   # data + czas + strefa
ymd_hm("2026-11-10 14:35")
parse_date_time(c("2026-11-10 14:35", "10.11.2026 09:05"), orders = c("ymd HM", "dmy HM"))   # mieszane formaty
as_date(ymd_hms("2026-11-10 14:35:20"))   # POSIXct -> Date
# Bez lubridate (base): as.Date("10.11.2026", format = "%d.%m.%Y")   | SQL: CAST/TO_DATE | Pandas: pd.to_datetime

# ---- Wyciąganie komponentów (SQL: EXTRACT / DATEPART; Pandas: .dt.*) --------
d <- ymd_hms("2026-11-10 14:35:20", tz = "Europe/Warsaw")
year(d); month(d); day(d); hour(d); minute(d)
quarter(d); week(d); isoweek(d); yday(d)
month(d, label = TRUE, abbr = FALSE)        # nazwa miesiąca (zależna od locale systemu)
wday(d, week_start = 1, label = TRUE)       # dzień tygodnia; week_start = 1 -> poniedziałek (standard PL/ISO)

# ---- Zaokrąglanie do okresu (SQL: DATE_TRUNC; Pandas: .dt.to_period / resample) ----
floor_date(d, "month")                      # początek miesiąca (agregacje miesięczne!)
floor_date(d, "week", week_start = 1)       # początek tygodnia (poniedziałek)
floor_date(d, "15 minutes")                 # kubełki 15-minutowe (przydatne w logach)
ceiling_date(d, "month")                    # początek NASTĘPNEGO miesiąca (koniec przedziału)
round_date(d, "hour")

# ---- Arytmetyka: Period (kalendarzowy) vs Duration (sekundy) ---------------
d + days(7)                                 # period: "tydzień kalendarzowy" (uwzględnia zmiany czasu)
d + months(1)                               # period: "1 miesiąc" (31 stycznia + 1 miesiąc = NA; użyj %m+%)
ymd("2026-01-31") %m+% months(1)            # 2026-02-28 (ostatni dzień miesiąca)
d + ddays(1); d + dhours(2)                 # duration: dokładnie 86400 / 7200 sekund

# ---- Różnice i interwały ----------------------------------------------------
d1 <- ymd("2026-01-15"); d2 <- ymd("2026-11-10")
d2 - d1                                      # difftime: Time difference of 299 days   | SQL: DATEDIFF(day, d1, d2)
as.integer(d2 - d1)                          # liczba dni jako integer
as.numeric(difftime(d2, d1, units = "weeks"))
int <- interval(d1, d2)                      # = d1 %--% d2
time_length(int, "months")                   # dokładna długość w miesiącach
time_length(interval(customers$signup_date, ymd("2026-11-10")), "years")   # staż klienta w latach (wektorowo)
ymd("2026-06-01") %within% int               # czy data wpada w przedział
int_overlaps(int, interval(ymd("2026-10-01"), ymd("2026-12-31")))

# ---- Bieżący czas, strefy, sekwencje ---------------------------------------
today(); now()
with_tz(d, "UTC")                            # ten sam MOMENT, inny zapis zegara (konwersja stref)
force_tz(d, "UTC")                           # ten sam ZEGAR, inny moment (poprawa błędnie przypisanej strefy)
seq(ymd("2026-01-01"), by = "month", length.out = 12)   # kalendarz miesięcy (dim_date)

# ---- Praktyka: luka między kolejnymi zdarzeniami (sessionizacja logów) ------
logs |>
  arrange(src_ip, ts) |>
  mutate(gap_sec = as.numeric(ts - lag(ts), units = "secs"), .by = src_ip) |>
  mutate(new_session = is.na(gap_sec) | gap_sec > 30 * 60, session_id = cumsum(new_session), .by = src_ip)
```

---

## 6. Wysokowydajne przetwarzanie: data.table

`data.table` to rozszerzenie `data.frame`: bardzo szybkie, oszczędne pamięciowo, z zwięzłą składnią **`DT[i, j, by]`**. Przy milionach wierszy zwykle wygrywa z dplyr/Pandas.

```
DT[ i ,  j ,  by ]
    │    │    └── GROUP BY
    │    └─────── SELECT / obliczenia / przypisania (:=)
    └──────────── WHERE (i do sortowania/joinów)
```

```r
library(data.table)

DT <- as.data.table(orders)     # kopiuje obiekt
# setDT(orders)                 # konwertuje IN-PLACE (bez kopii, oszczędność RAM): obiekt "orders" staje się data.table

# ---- i: filtrowanie wierszy (SQL: WHERE) ------------------------------------
DT[status == "completed"]                          # bez cudzysłowów i DT$: kolumny widoczne wprost
DT[status == "completed" & revenue > 500]
DT[order(-revenue)][1:5]                           # TOP 5 wg przychodu
DT[category %chin% c("Home", "Books")]             # %chin% = szybki %in% dla tekstu

# ---- j: wybór/obliczenia (SQL: SELECT) --------------------------------------
DT[, revenue]                                      # wektor
DT[, .(order_id, revenue)]                         # .() = list(): zwraca data.table z kolumnami
DT[, .(total = sum(revenue), avg = mean(revenue))] # agregacja do jednego wiersza
DT[, c("order_id", "revenue")]                     # wybór po nazwach (też zmienna z nazwami: DT[, cols, with = FALSE])

# ---- by: grupowanie (SQL: GROUP BY) -----------------------------------------
DT[status == "completed",                          # i  = WHERE
   .(n = .N, total = sum(revenue)),                # j  = SELECT agregaty (.N = liczba wierszy w grupie = COUNT(*))
   by = category]                                  # by = GROUP BY
DT[, .N, by = .(category, status)]                 # COUNT(*) po 2 kolumnach
DT[, .(n_customers = uniqueN(customer_id)), by = category]

# ---- Symbole specjalne ------------------------------------------------------
# .N   = liczba wierszy (w grupie)       .I   = numery wierszy        .GRP = numer grupy
# .SD  = Subset of Data (kolumny/wiersze bieżącej grupy)              .BY  = wartości kluczy grupowania
DT[, lapply(.SD, mean), .SDcols = c("revenue", "qty"), by = category]   # średnia wielu kolumn per grupa (across)
DT[, .SD[which.max(revenue)], by = customer_id]                         # najlepsze zamówienie każdego klienta (TOP 1 per grupa)

# ---- Chaining: DT[...][...] (odpowiednik potoku) ---------------------------
DT[status == "completed", .(rev = sum(revenue)), by = category][order(-rev)]

# ---- Klucze i indeksy: błyskawiczne wyszukiwanie (binary search O(log n)) ---
setkey(DT, customer_id)            # FIZYCZNIE sortuje tabelę po kluczu (in-place) i oznacza go jako klucz
DT[.(5)]                           # szybki lookup po kluczu (wiersze klienta 5)
key(DT)                            # sprawdź klucz
setindex(DT, category)             # indeks wtórny: bez zmiany kolejności wierszy
DT[category == "Home"]             # automatycznie użyje indeksu (bez zmiany składni)
# Analogia: setkey ≈ clustered index, setindex ≈ non-clustered index w SQL

# ---- JOIN: X[Y, on = ...] -> dla KAŻDEGO wiersza Y szukamy dopasowania w X ----
cust <- as.data.table(customers)
cust[DT, on = "customer_id"]                      # LEFT JOIN: DT LEFT JOIN cust (wszystkie wiersze DT)
cust[DT, on = "customer_id", nomatch = NULL]      # INNER JOIN
DT[cust, on = "customer_id"]                      # odwrotnie: wszystkie wiersze cust (RIGHT JOIN z perspektywy DT)
merge(DT, cust, by = "customer_id", all.x = TRUE) # klasyczna składnia (jak w Pandas merge)
DT[cust, on = .(customer_id, order_date >= signup_date), nomatch = NULL]   # join nierównościowy
# Rolling join (as-of join: dopasuj najbliższy poprzedni znacznik czasu), niezwykle przydatne w logach:
# events[alerts, on = .(src_ip, ts), roll = TRUE]
```

### 6.1 Modyfikacja w miejscu: operator `:=` (przez referencję)

`:=` **dodaje, zmienia lub usuwa kolumny bez kopiowania całej tabeli**. To główny powód wydajności `data.table` (brak alokacji nowej kopii przy dużych danych). Wynik **nie jest** przypisywany `<-`.

```r
DT[, vat := revenue * 0.23]                              # nowa kolumna (in-place)
DT[, `:=`(gross = revenue * 1.23,                         # kilka kolumn naraz
          month = floor_date(order_date, "month"))]
DT[status == "returned", revenue := 0]                   # UPDATE ... WHERE status = 'returned' (tylko wybrane wiersze)
DT[, vat := NULL]                                         # usunięcie kolumny (ALTER TABLE DROP COLUMN)

# ---- Funkcje okna w data.table: := + by (zamiast group_by + mutate) ----------
setorder(DT, customer_id, order_date)                     # sortowanie in-place (zamiast order() w i)
DT[, avg_cat   := mean(revenue), by = category]           # AVG() OVER (PARTITION BY category)
DT[, rnk       := frank(-revenue, ties.method = "min"), by = category]   # RANK()
DT[, cum_rev   := cumsum(revenue), by = customer_id]      # narastająco per klient
DT[, `:=`(prev_rev = shift(revenue, 1, type = "lag"),     # LAG()
          next_rev = shift(revenue, 1, type = "lead")),   # LEAD()
   by = customer_id]

# ---- Warunki: fifelse() / fcase() (szybsze odpowiedniki if_else / case_when) --
DT[, tier := fcase(revenue >= 1000, "A",
                   revenue >= 300,  "B",
                   default = "C")]

# ---- Pozostałe funkcje "set*" (wszystkie in-place) ----------------------------
setnames(DT, "revenue", "net_revenue")                    # zmiana nazwy kolumny
setcolorder(DT, c("order_id", "customer_id"))             # kolejność kolumn
setorder(DT, -net_revenue)                                # sortowanie malejąco
set(DT, i = 1L, j = "net_revenue", value = 0)             # najszybsza modyfikacja pojedynczej komórki (do użycia w pętlach)

# ---- ⚠️ PUŁAPKA referencji: przypisanie nie kopiuje -------------------------
DT2 <- DT              # DT2 i DT to TEN SAM obiekt w pamięci
DT2[, test := 1]       # dodało kolumnę "test" także do DT!
DT3 <- copy(DT)        # jawna, głęboka kopia (używaj, gdy chcesz mieć niezależny obiekt)
# To samo dotyczy funkcji: := wewnątrz funkcji modyfikuje oryginał użytkownika. Zrób copy() na wejściu, jeśli tego nie chcesz.

# ---- Reshape: dcast (wide) / melt (long) ------------------------------------
dcast(DT[status == "completed"], month ~ category, value.var = "net_revenue", fun.aggregate = sum)
melt(wide_dt, id.vars = "month", variable.name = "category", value.name = "revenue")

# ---- Benchmark: dlaczego data.table? ----------------------------------------
# GROUP BY na 50 mln wierszy: data.table zwykle kilka razy szybszy niż dplyr i Pandas, zużywa mniej RAM.
# Reguła kciuka: do ~1 mln wierszy używaj tego, co czytelniejsze (dplyr); powyżej: data.table lub arrow/duckdb.
```

---

## 7. Wizualizacja: ggplot2

### 7.1 Gramatyka grafiki

Wykres = **dane** + **mapowanie estetyk** (`aes()`: co jest osią X/Y, kolorem, rozmiarem) + **warstwy geometrii** (`geom_*`) + opcjonalnie skale, fasety, motyw. Warstwy dokładasz operatorem `+`. Analogia z Qlik/Excela: `aes()` to przeciągnięcie wymiaru i miary na osie/kolory, `geom_*` to wybór typu wykresu.

```r
ggplot(data = orders_ok, mapping = aes(x = category, y = revenue)) +   # 1) dane + mapowanie (dziedziczone przez geomy)
  geom_boxplot() +                                                    # 2) geometria (typ wykresu)
  labs(title = "Przychód wg kategorii", x = NULL, y = "PLN") +        # 3) opisy
  theme_minimal()                                                     # 4) motyw
# Wynik przypisz do obiektu: p <- ggplot(...) + ..., aby dokładać warstwy lub zapisać: ggsave("p.png", p)
```

### 7.2 Najczęstsze typy wykresów

```r
# ---- Słupkowy: geom_col() (wysokość = wartość z danych) vs geom_bar() (liczy wiersze) ----
orders_ok |>
  summarise(revenue = sum(revenue), .by = category) |>
  ggplot(aes(x = fct_reorder(category, revenue), y = revenue)) +   # fct_reorder: sortuje słupki wg wartości
  geom_col(fill = "#2C7FB8") +
  coord_flip() +                                                    # słupki poziome (czytelne etykiety)
  scale_y_continuous(labels = label_number(big.mark = " ", suffix = " PLN")) +   # format osi (pakiet scales)
  labs(title = "Przychód wg kategorii", x = NULL, y = NULL) +
  theme_minimal(base_size = 12)

orders |> ggplot(aes(x = status)) + geom_bar()                      # geom_bar sam liczy: SELECT status, COUNT(*)

# ---- Słupki grupowane / skumulowane / 100% (argument position) --------------
monthly_cat |>
  ggplot(aes(x = month, y = revenue, fill = category)) +
  geom_col(position = "stack") +        # "stack" (domyślnie) | "dodge" (obok siebie) | "fill" (100% udziały)
  scale_fill_brewer(palette = "Set2")

# ---- Punktowy (scatter) + linia trendu --------------------------------------
orders_ok |>
  ggplot(aes(x = qty, y = revenue, colour = category)) +
  geom_jitter(width = 0.15, alpha = 0.7, size = 2) +                # jitter: lekkie rozrzucenie nakładających się punktów
  geom_smooth(aes(group = 1), method = "lm", se = TRUE,             # jedna linia regresji dla wszystkich punktów
              colour = "black", linewidth = 0.8)

# ---- Liniowy (szereg czasowy) -----------------------------------------------
monthly |>
  ggplot(aes(x = month, y = revenue)) +
  geom_line(colour = "#D95F0E", linewidth = 1) +
  geom_point(size = 2) +
  scale_x_date(date_labels = "%b %Y", date_breaks = "1 month") +
  scale_y_continuous(labels = label_number(big.mark = " ")) +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))

# ---- Boxplot (rozkład: mediana, kwartyle, wartości odstające) ---------------
orders_ok |>
  ggplot(aes(x = category, y = revenue, fill = category)) +
  geom_boxplot(outlier.colour = "red", show.legend = FALSE) +
  geom_jitter(width = 0.1, alpha = 0.4)                              # surowe punkty na wierzchu

# ---- Histogram / gęstość ----------------------------------------------------
orders_ok |> ggplot(aes(x = revenue)) + geom_histogram(bins = 20, fill = "grey70", colour = "white")
orders_ok |> ggplot(aes(x = revenue, fill = category)) + geom_density(alpha = 0.4)

# ---- Heatmapa zdarzeń (cyber): godzina x typ zdarzenia ----------------------
logs |>
  mutate(hour = hour(ts)) |>
  count(hour, event) |>
  ggplot(aes(x = factor(hour), y = event, fill = n)) +
  geom_tile(colour = "white") +
  scale_fill_viridis_c(option = "magma") +
  labs(x = "Godzina", y = NULL, fill = "Liczba zdarzeń")
```

### 7.3 Fasetowanie (małe wielokrotności)

```r
# facet_wrap: jeden panel na wartość zmiennej (odpowiednik "trellis" / small multiples w Qliku)
monthly_cat |>
  ggplot(aes(x = month, y = revenue)) +
  geom_col(fill = "#2C7FB8") +
  facet_wrap(~ category, ncol = 2, scales = "free_y")     # scales = "free_y": niezależna skala Y w każdym panelu

# facet_grid: siatka wiersze ~ kolumny (dwa wymiary)
orders |>
  left_join(customers, by = "customer_id") |>
  filter(!is.na(segment)) |>
  ggplot(aes(x = revenue)) +
  geom_histogram(bins = 15) +
  facet_grid(segment ~ status)
```

### 7.4 Motywy i szybka stylizacja

```r
theme_set(theme_minimal(base_size = 12))    # globalny motyw dla całej sesji (theme_bw, theme_classic, theme_light, theme_void)

p <- monthly |>
  ggplot(aes(month, revenue)) + geom_line() +
  labs(title = "Przychód miesięczny", subtitle = "Zamówienia zrealizowane, 2026",
       x = NULL, y = "PLN", caption = "Źródło: sklep online") +
  theme(
    legend.position = "bottom",                                  # "none" ukrywa legendę
    plot.title      = element_text(face = "bold", size = 14),
    panel.grid.minor = element_blank(),                          # wyłącz drobną siatkę
    axis.text.x     = element_text(angle = 45, hjust = 1)
  )

# Kolory: własne wartości / palety
#   scale_fill_manual(values = c(Electronics = "#1b9e77", Home = "#d95f02", Fashion = "#7570b3", Books = "#e7298a"))
#   scale_colour_viridis_d()   (paleta przyjazna daltonistom)

# Zapis (rozmiar w calach, dpi = rozdzielczość)
ggsave("out/revenue_monthly.png", plot = p, width = 8, height = 5, dpi = 300)

# Układ wielu wykresów: library(patchwork); p1 + p2 | p3
# Interaktywność (jak w dashboardach BI): plotly::ggplotly(p)
```

---

## 8. Statystyka i modelowanie

Dane syntetyczne z sesjami w sklepie online (500 wierszy), spójne ze scenariuszem biznesowym:

```r
set.seed(123)
n <- 500
sessions <- tibble(
  session_id   = 1:n,
  device       = sample(c("mobile", "desktop"), n, replace = TRUE, prob = c(0.6, 0.4)),
  channel      = sample(c("organic", "paid", "email"), n, replace = TRUE),
  pages        = rpois(n, 5) + 1,                                              # liczba odwiedzonych podstron
  time_on_site = round(rgamma(n, shape = 2, scale = if_else(device == "desktop", 120, 90)))   # sekundy
) |>
  mutate(
    basket_value = round(20 + 8 * pages + if_else(device == "desktop", 25, 0) + rnorm(n, 0, 30), 2),
    converted    = rbinom(n, 1, plogis(-3 + 0.25 * pages + 0.4 * (device == "desktop") + 0.5 * (channel == "email")))
  )
```

### 8.1 Klasyczne testy statystyczne

```r
# ---- Test t-Studenta: czy średni czas na stronie różni się między desktopem a mobile? ----
# H0: średnie w obu grupach są równe. Jeśli p < 0.05, odrzucamy H0 (różnica istotna statystycznie).
t.test(time_on_site ~ device, data = sessions)                    # domyślnie test WELCHA (nie zakłada równych wariancji)
t.test(time_on_site ~ device, data = sessions, var.equal = TRUE)  # klasyczny Student (równe wariancje)
# t.test(przed, po, paired = TRUE)                                # test dla par (pomiary powtórzone, np. przed/po zmianie)
# Założenia: w przybliżeniu normalny rozkład (lub duża próba). Alternatywa nieparametryczna:
wilcox.test(time_on_site ~ device, data = sessions)               # test Manna-Whitneya
shapiro.test(sessions$time_on_site[1:200])                        # test normalności (p < 0.05 -> rozkład nie jest normalny)
# Pandas/SciPy: scipy.stats.ttest_ind(a, b, equal_var=False)

# ---- Korelacja ---------------------------------------------------------------
cor(sessions$pages, sessions$basket_value)                         # Pearson (liniowa), wartość -1..1
cor(sessions$pages, sessions$time_on_site, method = "spearman")    # Spearman (rangowa, odporna na odstające)
cor.test(~ pages + basket_value, data = sessions)                  # + p-value i przedział ufności
sessions |> select(where(is.numeric), -session_id) |> cor(use = "pairwise.complete.obs")   # macierz korelacji
# Pandas: df.corr()

# ---- Chi-kwadrat: czy konwersja zależy od kanału? (zmienne kategoryczne) -----
tab <- table(sessions$channel, sessions$converted)                 # tabela krzyżowa (kontyngencji)
tab
chisq.test(tab)                                                    # H0: zmienne są niezależne
chisq.test(tab)$expected                                           # liczności oczekiwane: jeśli któraś < 5, użyj fisher.test()
prop.table(tab, margin = 1)                                        # odsetki wierszowe (conversion rate per kanał)
# SciPy: scipy.stats.chi2_contingency(tab)

# ---- ANOVA: porównanie średnich więcej niż 2 grup ------------------------------
aov_res <- aov(basket_value ~ channel, data = sessions)
summary(aov_res)
TukeyHSD(aov_res)                                                  # które pary kanałów się różnią (post-hoc)
```

### 8.2 Regresja liniowa: `lm()`

Składnia **formuły**: `y ~ x1 + x2` (zmienna objaśniana ~ objaśniające). Identyczna jak w `statsmodels.formula.api` w Pythonie.

```r
m_lin <- lm(basket_value ~ pages + device + channel, data = sessions)
summary(m_lin)
# Jak czytać summary():
#  - Estimate    : zmiana Y przy wzroście X o 1 jednostkę, przy stałych pozostałych zmiennych (ceteris paribus)
#  - Zmienne kategoryczne: współczynnik = różnica względem POZIOMU REFERENCYJNEGO (pierwszy alfabetycznie)
#  - Pr(>|t|)    : p-value dla współczynnika (p < 0.05 -> istotny)
#  - R-squared   : jaka część wariancji Y jest wyjaśniona; Adjusted R-squared uwzględnia liczbę zmiennych

confint(m_lin)                                              # przedziały ufności współczynników

# Zmiana poziomu referencyjnego faktora
sessions <- sessions |> mutate(channel = fct_relevel(channel, "organic"))

# Interakcje i transformacje
lm(basket_value ~ pages * device, data = sessions)          # pages + device + pages:device (efekt pages zależy od device)
lm(log(basket_value) ~ pages, data = sessions)              # model log-liniowy
lm(basket_value ~ pages + I(pages^2), data = sessions)      # I() chroni operator arytmetyczny wewnątrz formuły

# Predykcja
new_sessions <- tibble(pages = c(3, 8), device = c("mobile", "desktop"), channel = c("paid", "email"))
predict(m_lin, newdata = new_sessions, interval = "prediction")   # przewidywana wartość + przedział

# Diagnostyka reszt (założenia: liniowość, homoskedastyczność, normalność reszt)
par(mfrow = c(2, 2)); plot(m_lin); par(mfrow = c(1, 1))
```

### 8.3 Regresja logistyczna: `glm()`

Do zmiennej binarnej (0/1): konwersja, churn, incydent tak/nie. Model przewiduje **log-szanse** (logit).

```r
m_log <- glm(converted ~ pages + device + channel,
             data = sessions, family = binomial(link = "logit"))
summary(m_log)

# Interpretacja: współczynniki są w skali log-odds. Aby uzyskać ILORAZ SZANS (odds ratio), wykonaj exp():
exp(coef(m_log))
#  OR = 1.28 dla "pages": każda dodatkowa podstrona zwiększa szanse konwersji o ~28%
#  OR = 1    brak efektu | OR > 1 zwiększa szanse | OR < 1 zmniejsza szanse

# Predykcja prawdopodobieństw i klasyfikacja
sessions <- sessions |>
  mutate(
    p_hat      = predict(m_log, type = "response"),     # prawdopodobieństwo 0..1 (bez type = "response" dostaniesz log-odds)
    pred_class = if_else(p_hat > 0.5, 1, 0)             # próg odcięcia (dobierz do kosztów błędów)
  )

table(actual = sessions$converted, predicted = sessions$pred_class)   # macierz pomyłek (confusion matrix)
mean(sessions$converted == sessions$pred_class)                       # accuracy
# AUC: pROC::auc(pROC::roc(sessions$converted, sessions$p_hat))
# Uwaga: przy niezbalansowanych klasach accuracy jest mylące (patrz precision/recall/AUC).
```

### 8.4 Pakiet `broom`: wyniki modeli jako tibble

`summary()` zwraca tekst, którego nie da się łatwo przetwarzać. `broom` zamienia modele i testy na **schludne ramki danych** (do filtrowania, wykresów, eksportu do Excela/Qlika).

```r
library(broom)

tidy(m_lin, conf.int = TRUE)       # tabela współczynników: term, estimate, std.error, statistic, p.value, conf.low, conf.high
glance(m_lin)                      # JEDEN wiersz z metrykami modelu: r.squared, adj.r.squared, sigma, AIC, BIC, nobs ...
augment(m_lin)                     # dane oryginalne + .fitted, .resid, .hat, .cooksd (diagnostyka obserwacji wpływowych)

# Model logistyczny: od razu ilorazy szans z przedziałami ufności
or_tbl <- tidy(m_log, exponentiate = TRUE, conf.int = TRUE)
glance(m_log)                      # null.deviance, deviance, AIC, nobs ...

# Testy statystyczne też obsługiwane
tidy(t.test(time_on_site ~ device, data = sessions))
tidy(cor.test(~ pages + basket_value, data = sessions))
tidy(chisq.test(tab))

# Wykres ilorazów szans (forest plot)
or_tbl |>
  filter(term != "(Intercept)") |>
  ggplot(aes(x = estimate, y = fct_reorder(term, estimate))) +
  geom_pointrange(aes(xmin = conf.low, xmax = conf.high)) +
  geom_vline(xintercept = 1, linetype = "dashed") +            # OR = 1: brak efektu
  labs(x = "Odds ratio (95% CI)", y = NULL)

# Wiele modeli naraz (np. osobna regresja dla każdego kanału), wzorzec "nest + map"
sessions |>
  nest_by(channel) |>
  mutate(model = list(lm(basket_value ~ pages, data = data))) |>
  reframe(tidy(model))
```

---

## Dodatek A: tabela odpowiedników

| Operacja | SQL | dplyr | Pandas | data.table |
|---|---|---|---|---|
| Wybór kolumn | `SELECT a, b` | `select(a, b)` | `df[["a","b"]]` | `DT[, .(a, b)]` |
| Filtr wierszy | `WHERE x > 5` | `filter(x > 5)` | `df[df.x > 5]` / `.query()` | `DT[x > 5]` |
| Nowa kolumna | `SELECT x*2 AS y` | `mutate(y = x * 2)` | `df.assign(y=df.x*2)` | `DT[, y := x * 2]` |
| Sortowanie | `ORDER BY x DESC` | `arrange(desc(x))` | `sort_values("x", ascending=False)` | `DT[order(-x)]` / `setorder(DT, -x)` |
| GROUP BY | `GROUP BY g` | `group_by(g) \|> summarise(...)` lub `.by = g` | `groupby("g").agg(...)` | `DT[, .(...), by = g]` |
| COUNT(*) | `COUNT(*)` | `n()` / `count()` | `size()` / `value_counts()` | `.N` |
| COUNT DISTINCT | `COUNT(DISTINCT x)` | `n_distinct(x)` | `nunique()` | `uniqueN(x)` |
| HAVING | `HAVING COUNT(*) > 1` | `filter()` po `summarise()` | `.query()` po `agg` | `DT[...][n > 1]` |
| LEFT JOIN | `LEFT JOIN b ON ...` | `left_join(b, by = ...)` | `a.merge(b, how="left")` | `b[a, on = ...]` |
| NOT EXISTS | `WHERE NOT EXISTS (...)` | `anti_join()` | `merge(indicator=True)` + filtr | `a[!b, on = ...]` |
| UNION ALL | `UNION ALL` | `bind_rows()` | `pd.concat()` | `rbindlist()` |
| DISTINCT | `SELECT DISTINCT` | `distinct()` | `drop_duplicates()` | `unique()` |
| CASE WHEN | `CASE WHEN ... END` | `case_when()` / `if_else()` | `np.select()` / `np.where()` | `fcase()` / `fifelse()` |
| COALESCE | `COALESCE(x, 0)` | `coalesce(x, 0)` / `replace_na()` | `fillna(0)` | `fcoalesce()` |
| ROW_NUMBER | `ROW_NUMBER() OVER (...)` | `row_number()` | `cumcount() + 1` | `seq_len(.N)` / `rowid()` |
| RANK | `RANK() OVER (...)` | `min_rank()` | `rank(method="min")` | `frank(ties.method="min")` |
| LAG / LEAD | `LAG(x) OVER (...)` | `lag(x)` / `lead(x)` | `shift(1)` / `shift(-1)` | `shift(x, 1)` / `shift(x, -1)` |
| Suma narastająca | `SUM(x) OVER (ORDER BY d)` | `cumsum(x)` | `cumsum()` | `cumsum(x)` |
| TOP N / grupa | `WHERE rn <= N` | `slice_max(x, n = N, by = g)` | `groupby().head(N)` | `DT[order(-x), head(.SD, N), by = g]` |
| PIVOT | `SUM(CASE WHEN ...)` | `pivot_wider()` | `pivot_table()` | `dcast()` |
| UNPIVOT | `UNPIVOT` / `UNION ALL` | `pivot_longer()` | `melt()` | `melt()` |
| DATE_TRUNC | `DATE_TRUNC('month', d)` | `floor_date(d, "month")` | `dt.to_period("M")` | `floor_date()` / `as.IDate` |
| DATEDIFF | `DATEDIFF(day, a, b)` | `as.integer(b - a)` | `(b - a).dt.days` | `as.integer(b - a)` |
| LIKE / REGEX | `LIKE '%x%'` | `str_detect(x, "x")` | `.str.contains("x")` | `x %like% "x"` |
| CTE | `WITH s AS (...)` | pipe `\|>` lub obiekty pośrednie | method chaining | `DT[...][...]` |

---

## Dodatek B: pułapki R (gotchas)

1. **Indeksy zaczynają się od 1**, a przedziały `1:3` są włącznie (inaczej niż `range(0,3)` w Pythonie).
2. **`=` vs `<-` vs `==`:** przypisanie to `<-` (konwencja), `=` służy do argumentów, a porównanie to `==`.
3. **`T`/`F` mogą zostać nadpisane** (`T <- 0`), więc zawsze pisz pełne `TRUE`/`FALSE`.
4. **`1:length(x)`** dla pustego `x` iteruje po `c(1, 0)`, więc używaj `seq_along(x)` / `seq_len(n)`.
5. **Faktor → liczba:** `as.numeric(factor)` zwraca kody. Poprawnie: `as.numeric(as.character(f))`.
6. **`NA`:** `NA == NA` daje `NA`, agregaty z `NA` zwracają `NA` (dodaj `na.rm = TRUE`), a `filter()` po cichu usuwa wiersze z `NA`.
7. **Recykling** wektorów o niedopasowanych długościach (ostrzeżenie, ale bywa cichy), uważaj z `==` na wektorze (użyj `%in%`).
8. **`&` vs `&&`:** `&` jest wektorowe (do `filter`), `&&` skalarne (do `if`).
9. **`ifelse()` gubi klasę `Date`**, więc używaj `dplyr::if_else()`.
10. **`summarise()` przysłania nazwy:** `summarise(revenue = sum(revenue), aov = mean(revenue))` policzy `aov` z już zsumowanej wartości.
11. **`lag()`/`lead()`/`cumsum()` zależą od kolejności wierszy**, więc zawsze `arrange()` przed lub użyj `order_by =`.
12. **`group_by()` zostaje do `ungroup()`**, a zapomniane grupowanie psuje kolejne kroki (stąd `.by` lub `.groups = "drop"`).
13. **`sapply()` jest niestabilne typologicznie** (raz wektor, raz lista, raz macierz), w kodzie produkcyjnym użyj `vapply()` lub `purrr::map_*`.
14. **`data.table` modyfikuje przez referencję:** `DT2 <- DT` nie kopiuje, użyj `copy()`.
15. **Konflikty nazw pakietów** (`dplyr::between` vs `data.table::between`, `lubridate::year` vs `data.table::year`, `dplyr::first`...), diagnozuj przez `conflicts()` i wołaj jawnie: `dplyr::filter()`, `data.table::first()`.
16. **Liczby zmiennoprzecinkowe:** `0.1 + 0.2 == 0.3` daje `FALSE`, porównuj przez `isTRUE(all.equal(a, b))` lub `dplyr::near(a, b)`.
17. **`$` robi partial matching** na data.frame (`df$rev` zadziała, jeśli jest jedna kolumna `revenue`), tibble tego nie robi (bezpieczniej).
18. **Zapis CSV traci typy**, a do wiernego cache'owania pośrednich wyników używaj `.rds` lub `.parquet`.
19. **Locale:** nazwy miesięcy/dni w `lubridate` zależą od ustawień systemowych (`Sys.setlocale("LC_TIME", "pl_PL.UTF-8")`).
20. **Praca z projektem:** używaj RStudio Projects (`.Rproj`) + `here::here()`, nigdy `setwd()` na sztywno w skryptach.

---

## Dodatek C: mini-case cyber

Wykrywanie podejrzanej aktywności logowania. Pokazuje połączenie: daty (`lubridate`), agregacje, funkcje okna i filtry.

```r
# ---- 1) Brute force: >= 3 nieudane logowania z jednego IP w oknie 10 minut ---
bruteforce <- logs |>
  filter(event == "LOGIN_FAIL") |>
  mutate(bucket = floor_date(ts, "10 minutes")) |>                  # kubełek czasowy (tumbling window)
  summarise(fails = n(), users_tried = n_distinct(user), .by = c(src_ip, bucket)) |>
  filter(fails >= 3) |>
  arrange(desc(fails))
# SQL: GROUP BY src_ip, DATE_BIN('10 minutes', ts) HAVING COUNT(*) >= 3

# ---- 2) Udane logowanie bezpośrednio po serii porażek (potencjalne przejęcie konta) ----
takeover_candidates <- logs |>
  filter(event %in% c("LOGIN_FAIL", "LOGIN_OK")) |>
  arrange(src_ip, user, ts) |>
  group_by(src_ip, user) |>
  mutate(
    prev_event  = lag(event),
    prev2_event = lag(event, 2),
    gap_sec     = as.numeric(ts - lag(ts), units = "secs")
  ) |>
  ungroup() |>
  filter(event == "LOGIN_OK", prev_event == "LOGIN_FAIL", prev2_event == "LOGIN_FAIL", gap_sec < 120)

# ---- 3) Anomalie wolumenu danych (exfiltration?): z-score bajtów per IP -------
anomalies <- logs |>
  mutate(z = (bytes - mean(bytes)) / sd(bytes), .by = src_ip) |>   # standaryzacja w obrębie IP
  filter(abs(z) > 3) |>                                            # reguła "3 sigm"
  arrange(desc(abs(z)))

# ---- 4) Profil ryzyka per IP: ranking + udział zdarzeń ------------------------
ip_risk <- logs |>
  summarise(
    events     = n(),
    fail_rate  = mean(event == "LOGIN_FAIL"),                      # średnia z logicznego = odsetek
    scans      = sum(event == "PORT_SCAN"),
    drops      = sum(event == "FW_DROP"),
    .by = src_ip
  ) |>
  mutate(risk_score = round(100 * (0.6 * fail_rate + 0.3 * pmin(scans / 5, 1) + 0.1 * pmin(drops / 20, 1)), 1)) |>
  arrange(desc(risk_score))

# ---- 5) Wykres: liczba nieudanych logowań w czasie (kubełki 10 minut) ---------
logs |>
  filter(event == "LOGIN_FAIL") |>
  count(bucket = floor_date(ts, "10 minutes"), src_ip) |>
  ggplot(aes(x = bucket, y = n, fill = src_ip)) +
  geom_col() +
  scale_x_datetime(date_labels = "%H:%M") +
  labs(title = "Nieudane logowania wg IP", x = "Czas", y = "Liczba prób", fill = "Źródłowy IP")
```

---

*Wskazówka na koniec: najszybszy sposób nauki to tłumaczenie własnych zapytań SQL i skryptów Pandas 1:1 na dplyr/data.table z pomocą Dodatku A, a potem upraszczanie ich do idiomatycznego R.*
