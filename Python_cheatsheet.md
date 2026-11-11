# Python Cheatsheet dla analityka BI (SQL / R → Python)

> Codzienna ściągawka: od podstaw języka, przez Pandas, wizualizację, po statystykę i modelowanie.
> Wszystkie przykłady używają **jednego, wspólnego zestawu danych** (e-commerce + logi sieciowe + sesje WWW), zdefiniowanego w sekcji 0. Jest to ten sam scenariusz co w ściądze z R, więc łatwo porównasz oba ekosystemy.
> Komentarze `# SQL:`, `# dplyr:` i `# Power Query:` pokazują odpowiedniki z Twojego obecnego warsztatu.

**Zasady kodu w tym pliku:** operacje wektorowe i wbudowane metody Pandas, method chaining (`.assign()`, `.query()`, `.agg()`), brak pętli po wierszach i brak `.apply()` tam, gdzie istnieje metoda wbudowana. Pętle pojawiają się tylko po *kolekcjach pythonowych* lub *chunkach* plików.

**Wersje:** Python ≥ 3.10 (`match`), pandas ≥ 2.2 (kod zgodny również z 3.x), seaborn ≥ 0.13, statsmodels ≥ 0.14. W pandas 3.x Copy-on-Write jest domyślny, w 2.x włącz: `pd.options.mode.copy_on_write = True`.

---

## Spis treści

0. [Setup i dane przykładowe](#0-setup-i-dane-przykładowe)
1. [Podstawy Pythona pod analizę danych](#1-podstawy-pythona-pod-analizę-danych)
2. [Fundamenty Pandas: wczytywanie, inspekcja, selekcja, filtrowanie](#2-fundamenty-pandas)
3. [Czyszczenie danych: braki, duplikaty, typy, tekst, daty](#3-czyszczenie-danych)
4. [Zaawansowana manipulacja: JOIN, groupby, okna, reshaping, wydajność](#4-zaawansowana-manipulacja-danych)
5. [Wizualizacja: Pandas, Seaborn, Matplotlib](#5-wizualizacja)
6. [Statystyka i modelowanie: SciPy, Statsmodels](#6-statystyka-i-modelowanie)
7. [Dodatek A: tabela SQL ↔ Pandas ↔ dplyr ↔ Power Query](#dodatek-a-tabela-odpowiedników)
8. [Dodatek B: pułapki Pythona i Pandas](#dodatek-b-pułapki-gotchas)
9. [Dodatek C: mini-case cyber (wykrywanie brute force)](#dodatek-c-mini-case-cyber)

---

## 0. Setup i dane przykładowe

```python
# --- Instalacja (jednorazowo) ------------------------------------------------
# pip install pandas numpy matplotlib seaborn scipy statsmodels openpyxl pyarrow duckdb sqlalchemy

import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
import scipy.stats as st
import statsmodels.api as sm
import statsmodels.formula.api as smf

# --- Wygodne ustawienia wyświetlania ---------------------------------------
pd.set_option("display.max_columns", None)        # pokaż wszystkie kolumny
pd.set_option("display.width", 140)
pd.options.display.float_format = "{:,.2f}".format
pd.options.mode.copy_on_write = True              # Copy-on-Write: bezpieczna semantyka kopii (domyślne w pandas 3.x)

rng = np.random.default_rng(42)                   # nowoczesny generator losowy (zamiast np.random.seed); R: set.seed(42)
```

### Dane: klienci, zamówienia, logi

```python
# --- Klienci (wymiar) --------------------------------------------------------
customers = pd.DataFrame({
    "customer_id": np.arange(1, 11),
    "name": ["Anna Nowak", "Jan Kowalski", "Eva Müller", "Hans Schmidt", "Petr Novák",
             "Maria Wiśniewska", "Jozef Kováč", "Lena Fischer", "Piotr Zieliński", "Karolina Mazur"],
    "country": ["PL", "PL", "DE", "DE", "CZ", "PL", "SK", "DE", "PL", "PL"],
    "segment": ["B2C", "B2B"] * 5,
    "signup_date": pd.to_datetime("2025-01-15")
                   + pd.to_timedelta([0, 30, 45, 90, 120, 150, 200, 240, 300, 330], unit="D"),
})

# --- Zamówienia (fakty) -------------------------------------------------------
n_orders = 60
orders = pd.DataFrame({
    "order_id": np.arange(1001, 1001 + n_orders),
    "customer_id": rng.integers(1, 13, n_orders),    # klienci 11 i 12 NIE istnieją w customers -> "osierocone" rekordy (demo anti-join)
    "order_date": pd.to_datetime("2026-01-01") + pd.to_timedelta(rng.integers(0, 301, n_orders), unit="D"),
    "category": rng.choice(["Electronics", "Home", "Fashion", "Books"], n_orders),
    "qty": rng.integers(1, 6, n_orders),
    "revenue": rng.uniform(20, 1500, n_orders).round(2),
    "status": rng.choice(["completed", "returned", "cancelled"], n_orders, p=[0.8, 0.1, 0.1]),
})

# Tylko zrealizowane zamówienia (będziemy używać bardzo często)
orders_ok = orders.query("status == 'completed'")

# --- Logi sieciowe / bezpieczeństwa -------------------------------------------
n_logs = 300
logs = pd.DataFrame({
    "ts": pd.Timestamp("2026-11-10 08:00:00", tz="Europe/Warsaw")
          + pd.to_timedelta(np.sort(rng.integers(0, 7201, n_logs)), unit="s"),   # posortowane znaczniki czasu
    "src_ip": rng.choice(["10.0.0.5", "192.168.1.20", "203.0.113.7", "198.51.100.23", "45.33.32.156"],
                         n_logs, p=[0.25, 0.25, 0.20, 0.20, 0.10]),
    "user": rng.choice(["admin", "jsmith", "akowalski", "svc_backup", "root"], n_logs),
    "event": rng.choice(["LOGIN_OK", "LOGIN_FAIL", "FW_DROP", "PORT_SCAN"], n_logs, p=[0.45, 0.35, 0.15, 0.05]),
    "bytes": rng.poisson(2000, n_logs),
})
```

---

## 1. Podstawy Pythona pod analizę danych

### 1.1 Wbudowane struktury danych

| Struktura | Składnia | Kolejność | Modyfikowalna | Duplikaty | Wyszukiwanie `in` | Typowe zastosowanie |
|---|---|---|---|---|---|---|
| **list** | `[1, 2, 3]` | tak (indeksy) | tak | tak | O(n) | sekwencja wartości, wyniki zapytań |
| **dict** | `{"k": v}` | tak (kolejność wstawiania) | tak | unikalne klucze | O(1) po kluczu | mapowania, słowniki wymiarów, rekordy JSON |
| **set** | `{1, 2, 3}` | nie | tak | **nie** | O(1) | unikalne wartości, operacje zbiorowe (blocklisty IP) |
| **tuple** | `(1, 2)` | tak | **nie** | tak | O(n) | stałe rekordy, klucze słowników, zwracanie wielu wartości |

```python
# ============ LIST: uporządkowana, modyfikowalna, dopuszcza duplikaty =============
amounts = [120.5, 99.0, 410.0, 75.25]
amounts.append(310.0)                 # dodaj na koniec
amounts.extend([20.0, 15.5])          # dodaj wiele
amounts.insert(0, 5.0)                # wstaw na pozycję 0
amounts.remove(99.0)                  # usuń pierwszą napotkaną wartość 99.0
last = amounts.pop()                  # zdejmij i zwróć ostatni element

amounts[0]                            # pierwszy (indeksy od 0! w R od 1)
amounts[-1]                           # ostatni
amounts[1:3]                          # wycinek [start:stop): stop jest WYŁĄCZNY
amounts[::-1]                         # odwrócona kolejność ([start:stop:krok])
sorted(amounts, reverse=True)         # nowa posortowana lista (oryginał bez zmian)
sum(amounts) / len(amounts)           # średnia "ręcznie" (w Pandas: .mean())

# ============ DICT: mapowanie klucz -> wartość (jak tabela słownikowa) =============
margin_by_category = {"Electronics": 0.12, "Home": 0.25, "Fashion": 0.40, "Books": 0.18}

margin_by_category["Home"]                    # 0.25 (KeyError, gdy brak klucza)
margin_by_category.get("Toys", 0.0)           # get() z wartością domyślną: brak wyjątku
margin_by_category["Toys"] = 0.30             # dodanie / nadpisanie
for category, margin in margin_by_category.items():   # pętla po parach (kolekcja pythonowa: OK)
    pass
margin_by_category | {"Garden": 0.22}         # scalanie słowników (Python 3.9+); {**a, **b} w starszych
# Użycie w Pandas: orders["category"].map(margin_by_category)  -> odpowiednik LEFT JOIN do słownika

# ============ SET: unikalne elementy + algebra zbiorów =============================
seen_ips = {"10.0.0.5", "203.0.113.7", "45.33.32.156", "203.0.113.7"}   # duplikat zniknie
blocklist = {"203.0.113.7", "198.51.100.23", "185.220.101.1"}

seen_ips & blocklist        # część wspólna    | SQL: INTERSECT
seen_ips | blocklist        # suma             | SQL: UNION
seen_ips - blocklist        # różnica          | SQL: EXCEPT
seen_ips ^ blocklist        # różnica symetryczna (w jednym, ale nie w obu)
"203.0.113.7" in blocklist  # O(1): błyskawiczne, w liście byłoby O(n)
# Deduplikacja z zachowaniem kolejności: list(dict.fromkeys(items))

# ============ TUPLE: niemodyfikowalna, hashowalna =================================
endpoint = ("203.0.113.7", 443)
ip, port = endpoint                  # rozpakowanie (unpacking)
first, *rest = [10, 20, 30, 40]      # first=10, rest=[20, 30, 40]
visits = {("10.0.0.5", 22): 14}      # krotka jako klucz słownika (lista by nie przeszła: niehashowalna)
# Funkcja zwracająca wiele wartości faktycznie zwraca krotkę (patrz 1.4)

# ============ Przydatne z biblioteki standardowej =================================
from collections import Counter, defaultdict
Counter(["LOGIN_FAIL", "LOGIN_OK", "LOGIN_FAIL", "FW_DROP"]).most_common(2)   # [('LOGIN_FAIL', 2), ('LOGIN_OK', 1)]
by_ip = defaultdict(list)            # słownik z automatyczną wartością domyślną (pusta lista)
by_ip["10.0.0.5"].append("LOGIN_OK") # bez sprawdzania, czy klucz istnieje
```

### 1.2 Instrukcje warunkowe

```python
revenue = 1250.0

# --- if / elif / else (wcięcia 4 spacje są składnią, nie stylem) -------------------
if revenue >= 1000:
    tier = "A"
elif revenue >= 300:            # kolejność ma znaczenie: pierwszy spełniony warunek wygrywa (jak CASE WHEN)
    tier = "B"
else:
    tier = "C"

# --- Operator warunkowy (ternary) ----------------------------------------------------
tier = "A" if revenue >= 1000 else "B" if revenue >= 300 else "C"

# --- Przydatne konstrukcje -------------------------------------------------------------
300 <= revenue < 1000           # porównania łańcuchowe (BETWEEN-like, ale z prawym końcem wyłączonym)
revenue > 100 and tier == "A"   # and / or / not (słowa, nie symbole!)
tier in {"A", "B"}              # test członkostwa (SQL: IN)
amounts = []
if not amounts:                 # "falsy": puste kolekcje, 0, "", None zachowują się jak False
    print("brak danych")
value = None
if value is None:               # None porównuj przez `is`, nie ==
    pass

# --- match (Python 3.10+): "switch" ze wzorcami -----------------------------------------
def severity(event: str) -> str:
    match event:
        case "PORT_SCAN":
            return "critical"
        case "LOGIN_FAIL" | "FW_DROP":      # kilka wartości w jednym przypadku
            return "warning"
        case _:                              # default
            return "info"

# ⚠️ W Pandas NIE używaj if/elif na całej kolumnie (ValueError: truth value of a Series is ambiguous).
#    Warunki wektorowe -> np.where / np.select / pd.cut (patrz sekcja 3.6).
```

### 1.3 Comprehensions ("zamiast pętli")

Zwięzły, szybki sposób budowania list/słowników/zbiorów z innych kolekcji **pythonowych**. Dla kolumn Pandas używaj operacji wektorowych, nie comprehensions.

```python
amounts = [120.5, 99.0, 410.0, 75.25, 310.0]

# --- List comprehension: [wyrażenie for element in kolekcja if warunek] -----------------
gross = [round(x * 1.23, 2) for x in amounts]                  # transformacja (map)
big_orders = [x for x in amounts if x > 100]                   # filtrowanie (WHERE)
labels = ["high" if x > 200 else "low" for x in amounts]       # warunek w wyrażeniu (CASE WHEN)
pairs = [(i, x) for i, x in enumerate(amounts, start=1)]       # enumerate: indeks + wartość
matrix = [[1, 2], [3, 4], [5, 6]]
flat = [x for row in matrix for x in row]                      # spłaszczenie (kolejność pętli jak w zagnieżdżonym for)

# --- Dictionary comprehension: {klucz: wartość for ...} --------------------------------
revenue_by_cat = {"Electronics": 5200.0, "Home": 3100.0, "Fashion": 2300.0, "Books": 900.0}
total = sum(revenue_by_cat.values())
share = {cat: round(v / total, 3) for cat, v in revenue_by_cat.items()}     # udział kategorii
inverse = {v: k for k, v in revenue_by_cat.items()}                          # odwrócenie mapowania
status_codes = dict(zip(["ok", "fail"], [200, 401]))                         # zip: sklejanie dwóch sekwencji

# --- Set comprehension: unikalne wartości ----------------------------------------------
emails = ["Jan@Firma.pl", "anna@shop.de", "piotr@firma.pl"]
domains = {e.split("@")[1].lower() for e in emails}            # {'firma.pl', 'shop.de'}

# --- Generator expression: LENIWE (nie buduje listy w pamięci), idealne do agregatów ---
total_gross = sum(x * 1.23 for x in amounts)                   # nawiasy () zamiast []
has_blocked = any(ip in blocklist for ip in seen_ips)          # any() przerywa przy pierwszym True

# --- Liczenie bez zagnieżdżonych pętli: Counter zamiast comprehension O(n*m) ------------
events = [{"src_ip": "10.0.0.5", "event": "LOGIN_FAIL"}, {"src_ip": "10.0.0.5", "event": "LOGIN_FAIL"},
          {"src_ip": "203.0.113.7", "event": "PORT_SCAN"}]
fails_by_ip = Counter(e["src_ip"] for e in events if e["event"] == "LOGIN_FAIL")

# Zasada: comprehension dla kolekcji pythonowych, wektoryzacja Pandas/NumPy dla kolumn.
# W R odpowiednik: lapply()/sapply()/purrr::map().
```

### 1.4 Funkcje: `def` i `lambda`

```python
# --- Klasyczne def: argumenty domyślne, type hints, docstring, return --------------------
def calc_margin(revenue, cost_ratio: float = 0.7, round_to: int = 2):
    """Marża = przychód * (1 - cost_ratio). Działa na liczbie ORAZ na kolumnie Pandas."""
    if not 0 <= cost_ratio <= 1:
        raise ValueError("cost_ratio musi być w przedziale [0, 1]")      # fail fast
    return np.round(revenue * (1 - cost_ratio), round_to)                 # np.round: skalar i Series (wbudowany round() tylko skalar)

calc_margin(1000)                                       # 300.0
calc_margin(orders["revenue"], cost_ratio=0.8)          # funkcja jest WEKTOROWA, bo składa się z operacji wektorowych

# --- Wiele wartości zwrotnych (krotka) i argumenty zmienne -----------------------------------
def order_kpis(df: pd.DataFrame) -> tuple[int, float, float]:
    return len(df), df["revenue"].sum(), df["revenue"].mean()
n, total, aov = order_kpis(orders_ok)                   # rozpakowanie krotki

def log_event(event: str, *tags, **meta):               # *tags -> krotka, **meta -> słownik
    return {"event": event, "tags": tags, **meta}
log_event("LOGIN_FAIL", "ssh", "bruteforce", src_ip="203.0.113.7")

# --- ⚠️ Pułapka: modyfikowalny argument domyślny ---------------------------------------------
def add_tag_bad(tag, tags=[]):        # ŹLE: lista tworzona RAZ i współdzielona między wywołaniami
    tags.append(tag); return tags
def add_tag(tag, tags=None):          # DOBRZE: None jako wartownik
    tags = [] if tags is None else tags
    tags.append(tag); return tags

# --- lambda: mała funkcja anonimowa (jedno wyrażenie) -------------------------------------------
sorted(events, key=lambda e: e["src_ip"])                           # klucz sortowania dla list pythonowych
max(events, key=lambda e: len(e["event"]))
customers.sort_values("name", key=lambda s: s.str.lower())          # key= dostaje CAŁĄ kolumnę (wektorowo)

# W method chaining lambda d: oznacza "bieżący DataFrame w łańcuchu" (a NIE iterację po wierszach):
orders.assign(gross=lambda d: d["revenue"] * 1.23)                  # R: mutate(gross = revenue * 1.23)

# --- Kompozycja łańcucha: .pipe() pozwala wpiąć własną funkcję ---------------------------------
def add_vat(df: pd.DataFrame, rate: float = 0.23) -> pd.DataFrame:
    return df.assign(gross=df["revenue"] * (1 + rate))
orders_ok.pipe(add_vat, rate=0.08).head()                           # R: orders |> add_vat(rate = 0.08)

from functools import partial
add_vat_8 = partial(add_vat, rate=0.08)                             # "zamrożony" argument

# --- Obsługa błędów i zasoby ------------------------------------------------------------------------
from pathlib import Path
try:
    df = pd.read_csv(Path("data") / "orders.csv")                   # pathlib: ścieżki niezależne od systemu
except FileNotFoundError as exc:
    print(f"Brak pliku: {exc}")                                     # f-string: interpolacja zmiennych
else:
    print(f"Wczytano {len(df):,} wierszy")                          # :, = separator tysięcy
finally:
    pass                                                            # wykona się zawsze (sprzątanie)
# with open("x.txt") as f: ...  -> menedżer kontekstu sam zamyka plik
```

---

## 2. Fundamenty Pandas

### 2.1 Wczytywanie danych

```python
from pathlib import Path

# ============ CSV ==============================================================================
orders_csv = pd.read_csv(
    "data/orders.csv",
    usecols=["order_id", "customer_id", "order_date", "category", "qty", "revenue", "status"],  # tylko potrzebne kolumny: mniej RAM i szybciej
    dtype={"order_id": "int32", "customer_id": "int32", "qty": "int8", "revenue": "float32",
           "category": "category", "status": "category"},       # jawne typy: kontrola jakości i oszczędność pamięci
    parse_dates=["order_date"],                                 # od razu datetime
    na_values=["", "N/A", "null", "-"],                         # co traktować jako NaN
    # nrows=1000,                                               # szybki podgląd dużego pliku
    # engine="pyarrow",                                         # wielowątkowe parsowanie (szybsze na dużych plikach)
)

# Polski eksport (Excel PL): średnik, przecinek dziesiętny, spacja jako separator tysięcy, Windows-1250
raport_pl = pd.read_csv("data/raport_pl.csv", sep=";", decimal=",", thousands=" ",
                        encoding="cp1250", parse_dates=["data"], dayfirst=True)

# Wiele plików naraz (np. dzienne logi) - generator + jedno concat (szybciej niż doklejanie w pętli)
all_logs = pd.concat(
    (pd.read_csv(f).assign(source_file=f.name) for f in sorted(Path("logs").glob("*.csv"))),
    ignore_index=True,
)
orders.to_csv("out/orders_clean.csv", index=False)              # index=False: bez zapisu indeksu jako kolumny

# ============ EXCEL (wymaga openpyxl) ============================================================
budget = pd.read_excel("data/budget.xlsx", sheet_name="2026", usecols="A:F", skiprows=2, header=0)
all_sheets = pd.read_excel("data/budget.xlsx", sheet_name=None)        # sheet_name=None -> słownik {arkusz: DataFrame}
with pd.ExcelWriter("out/report.xlsx", engine="openpyxl") as xw:      # wiele arkuszy w jednym pliku
    orders.to_excel(xw, sheet_name="Orders", index=False)
    customers.to_excel(xw, sheet_name="Customers", index=False)
# Power Query: "From Workbook" / "Combine Files"

# ============ JSON / NDJSON (logi, API) ===========================================================
events = pd.read_json("logs/events.ndjson", lines=True)                # lines=True: jeden obiekt JSON na linię
api_df = pd.json_normalize([{"id": 1, "geo": {"country": "PL", "city": "Kraków"}}])   # spłaszcza zagnieżdżenia -> geo.country

# ============ PARQUET (format kolumnowy, standard w hurtowniach) ======================================
orders.to_parquet("out/orders.parquet", index=False)                   # zachowuje typy; mały rozmiar; szybki odczyt
orders_pq = pd.read_parquet("out/orders.parquet", columns=["order_id", "revenue"])   # czyta tylko wybrane kolumny

# ============ SQLite / bazy SQL ======================================================================
import sqlite3
con = sqlite3.connect(":memory:")                                     # lub sqlite3.connect("dwh.db")
orders.to_sql("orders", con, index=False, if_exists="replace")        # zapis DataFrame do tabeli
pd.read_sql("SELECT category, SUM(revenue) AS rev FROM orders GROUP BY category", con)
pd.read_sql("SELECT * FROM orders WHERE status = ? AND revenue > ?", con,
            params=("completed", 500))                                # ZAWSZE parametryzuj (ochrona przed SQL injection)
# Inne bazy: from sqlalchemy import create_engine; pd.read_sql(sql, create_engine("postgresql+psycopg://..."))

# ============ DuckDB: SQL bezpośrednio na DataFrame'ach i plikach (świetne dla znających SQL) ==========
import duckdb

duckdb.sql("""
    WITH monthly AS (                                                    -- CTE działa normalnie
        SELECT date_trunc('month', order_date) AS month, category, SUM(revenue) AS revenue
        FROM orders                                                      -- DuckDB widzi zmienną Pythona `orders`
        WHERE status = 'completed'
        GROUP BY ALL                                                     -- GROUP BY po wszystkich kolumnach niebędących agregatami
    )
    SELECT *, LAG(revenue) OVER (PARTITION BY category ORDER BY month) AS prev_revenue
    FROM monthly
    ORDER BY category, month
""").df()                                                                # .df() -> wynik jako pandas DataFrame

duckdb.sql("SELECT src_ip, COUNT(*) AS n FROM 'logs/*.parquet' WHERE event = 'LOGIN_FAIL' GROUP BY 1").df()
# ↑ zapytanie wprost na wielu plikach Parquet/CSV bez ładowania ich do RAM; Pandas read_sql oficjalnie wspiera tylko
#   sqlite3 i SQLAlchemy, więc dla DuckDB używaj duckdb.sql(...).df()
```

### 2.2 Inspekcja struktury

```python
orders.head()                         # pierwsze 5 wierszy | tail(3) = ostatnie 3 | R: head()
orders.sample(5, random_state=42)     # losowe 5 wierszy (powtarzalnie)
orders.shape                          # (wiersze, kolumny)
orders.columns.tolist()               # lista nazw kolumn
orders.dtypes                         # typy kolumn
orders.info()                         # typy + liczba NIE-null + zużycie pamięci (pierwsze, co robisz po wczytaniu!)

orders.describe()                                     # statystyki kolumn numerycznych (R: summary())
orders.describe(include="all")                        # + kolumny tekstowe/kategorii (unique, top, freq)
orders["revenue"].describe(percentiles=[.5, .9, .99]) # własne percentyle

orders["category"].value_counts()                     # SQL: GROUP BY + COUNT(*) ORDER BY DESC | R: count(category, sort = TRUE)
orders["status"].value_counts(normalize=True)         # udziały zamiast liczności
orders["status"].value_counts(dropna=False)           # uwzględnij NaN jako osobną kategorię
orders["customer_id"].nunique()                       # SQL: COUNT(DISTINCT customer_id)
orders.isna().sum()                                   # liczba braków w każdej kolumnie
orders.select_dtypes("number").columns                # wybór kolumn wg typu (R: select(where(is.numeric)))
orders.memory_usage(deep=True).sum() / 1e6            # zużycie pamięci w MB
pd.crosstab(orders["category"], orders["status"])     # tabela krzyżowa (liczności)
```

### 2.3 Selekcja i indeksowanie: `.loc[]` vs `.iloc[]`

| | `.loc[]` | `.iloc[]` |
|---|---|---|
| Wybór po | **etykietach** (nazwy wierszy/kolumn, maski logiczne) | **pozycjach** (numery od 0) |
| Wycinek `a:b` | **włącznie** z końcem | **bez** końca (jak w liście) |
| Typowe użycie | filtr + wybór kolumn, przypisania | "pierwsze N wierszy", dostęp pozycyjny |

```python
o = orders.set_index("order_id")      # etykietą wierszy staje się order_id (jak PRIMARY KEY)

# --- .loc: etykiety ---------------------------------------------------------------------
o.loc[1001]                           # jeden wiersz (Series) po etykiecie
o.loc[1001:1005]                      # etykiety 1001..1005 WŁĄCZNIE (wymaga posortowanego indeksu)
o.loc[1001:1005, ["category", "revenue"]]           # wiersze + kolumny
orders.loc[orders["revenue"] > 500, ["order_id", "revenue"]]    # maska logiczna + kolumny (SQL: WHERE + SELECT)
orders.loc[:, "order_id":"category"]                # zakres kolumn po nazwach (włącznie)

# --- .iloc: pozycje -----------------------------------------------------------------------
orders.iloc[0]                        # pierwszy wiersz
orders.iloc[:5]                       # pierwsze 5 (stop wyłączny) | R: slice(1:5) / head(5)
orders.iloc[:5, [0, 5]]               # 5 wierszy, kolumny nr 0 i 5
orders.iloc[-3:]                      # ostatnie 3

# --- Przypisania: zawsze przez .loc (nigdy łańcuch orders[mask]["col"] = ...) ------------------
orders.loc[orders["status"] == "returned", "revenue"] = 0       # SQL: UPDATE ... WHERE status = 'returned'
# ⚠️ Chained assignment (df[mask]["col"] = x) nie modyfikuje oryginału (przy Copy-on-Write nigdy nie zadziała).

# --- Dostęp do jednej komórki: .at (etykiety) / .iat (pozycje) są szybsze niż .loc/.iloc -----------
o.at[1001, "revenue"]

# --- Wybór kolumn ------------------------------------------------------------------------------------
orders["revenue"]                     # Series (jedna kolumna)
orders[["order_id", "revenue"]]       # DataFrame (lista nazw; podwójne nawiasy!)
orders.drop(columns=["status"])       # wszystko oprócz | R: select(-status)
orders.filter(like="order")           # kolumny po fragmencie nazwy | filter(regex=...) | R: select(contains("order"))
```

### 2.4 Filtrowanie wierszy (Boolean Indexing)

```python
# Maska logiczna = Series True/False; df[maska] zwraca wiersze spełniające warunek
orders[orders["revenue"] > 500]                                       # SQL: WHERE revenue > 500

# --- Wiele warunków: & (AND), | (OR), ~ (NOT); KAŻDY warunek w nawiasach! ------------------------
orders[(orders["status"] == "completed") & (orders["revenue"] > 500)]
orders[(orders["status"] == "returned") | (orders["qty"] >= 4)]
orders[~(orders["status"] == "cancelled")]                           # NOT
# ⚠️ Słowa `and`/`or` nie działają na Series (ValueError). Brak nawiasów -> błąd priorytetu operatorów.

# --- isin: SQL IN / NOT IN ----------------------------------------------------------------------------
orders[orders["category"].isin(["Home", "Books"])]                    # IN ('Home','Books')
orders[~orders["category"].isin(["Home", "Books"])]                   # NOT IN

# --- between: przedział obustronnie włączony (SQL: BETWEEN) ----------------------------------------------
orders[orders["revenue"].between(100, 500)]

# --- .query(): czytelniejsza składnia tekstowa, bez powtarzania nazwy DataFrame -----------------------------
orders.query("status == 'completed' and revenue > 500")               # R: filter(status == "completed", revenue > 500)
threshold = 800
orders.query("revenue > @threshold and category in ['Home', 'Books']")   # @zmienna: odwołanie do zmiennej Pythona

# --- Warunki na tekście i brakach ------------------------------------------------------------------------------
customers[customers["name"].str.contains("ski$", case=False, regex=True)]    # SQL: LIKE '%ski'
orders[orders["revenue"].isna()]                                       # IS NULL (==np.nan nigdy nie zadziała)
orders[orders["revenue"].notna()]                                      # IS NOT NULL

# --- Maska zapisana do zmiennej: przydatna do czytelności i wielokrotnego użycia ------------------------------
is_big = orders["revenue"] > 1000
is_ok = orders["status"].eq("completed")
orders.loc[is_big & is_ok, ["order_id", "revenue"]]
is_big.sum(), is_big.mean()                                            # ile / jaki odsetek spełnia warunek (True = 1)
```

---

## 3. Czyszczenie danych

### 3.1 Braki danych

```python
df_na = pd.DataFrame({
    "order_id": [1, 2, 3, 4, 5, 6],
    "revenue":  [100.0, np.nan, 250.0, np.nan, 80.0, 300.0],
    "country":  ["PL", None, "DE", "DE", "", "CZ"],
    "segment":  ["B2C", "B2B", None, None, "B2C", "B2B"],
})

# --- Diagnoza (SQL: IS NULL / COUNT(*) - COUNT(col)) -----------------------------------------------------
df_na.isna()                         # maska True/False dla każdej komórki (notna() = odwrotność)
df_na.isna().sum()                   # liczba braków w kolumnie
df_na.isna().mean().mul(100).round(1)   # % braków w kolumnie (profil jakości danych)
df_na[df_na["revenue"].isna()]       # wiersze z brakiem

# --- Usuwanie: dropna ----------------------------------------------------------------------------------------
df_na.dropna()                                # wiersze z JAKIMKOLWIEK NaN | R: drop_na()
df_na.dropna(subset=["revenue"])              # tylko wg wybranych kolumn
df_na.dropna(how="all")                       # tylko wiersze całkowicie puste
df_na.dropna(thresh=3)                        # zostaw wiersze mające >= 3 wartości niepuste
df_na.dropna(axis=1, thresh=5)                # usuń KOLUMNY z zbyt wieloma brakami

# --- Uzupełnianie: fillna ------------------------------------------------------------------------------------
df_na["revenue"].fillna(0)                                       # stała | SQL: COALESCE(revenue, 0) | R: replace_na()
df_na.fillna({"revenue": 0, "segment": "UNKNOWN"})               # słownik: różne wartości dla różnych kolumn
df_na["revenue"].fillna(df_na["revenue"].median())               # imputacja medianą (odporna na odstające)
df_na["revenue"].ffill()                                          # przenieś ostatnią znaną wartość w dół (LOCF) | R: tidyr::fill()
df_na["revenue"].bfill()                                          # w górę
df_na["revenue"].interpolate()                                    # interpolacja liniowa (szeregi czasowe)

# Imputacja medianą WEWNĄTRZ grupy (transform zwraca serię o długości oryginału: wzorzec "okno")
df_na.assign(revenue=lambda d: d["revenue"].fillna(d.groupby("country")["revenue"].transform("median")))

# --- Puste teksty i wartości zastępcze -> prawdziwe NaN ----------------------------------------------------------
df_na.replace({"country": {"": np.nan}})                          # "" -> NaN | R: na_if(country, "")
df_na.replace(["N/A", "null", "-"], np.nan)                       # kilka placeholderów naraz
# Najlepiej już przy wczytaniu: pd.read_csv(..., na_values=[...])

# --- Typy nullable (Int64, string, boolean): liczby całkowite z brakami ----------------------------------------------
pd.Series([1, None, 3]).dtype                                     # float64 (NaN wymusza float)
pd.Series([1, None, 3], dtype="Int64")                            # Int64: zachowuje typ całkowity, brak = <NA>
df_na.convert_dtypes()                                            # automatycznie wybierz typy nullable

# Semantyka: np.nan == np.nan -> False (testuj isna()); agregaty domyślnie POMIJAJĄ NaN (skipna=True), odwrotnie niż w R (na.rm = FALSE)
```

### 3.2 Duplikaty

```python
customers_raw = pd.DataFrame({
    "customer_id": [1, 2, 2, 3, 3, 3],
    "email": ["a@x.pl", "b@x.pl", "b@x.pl", "c@x.de", "c@y.de", "c@z.de"],
    "updated_at": pd.to_datetime(["2026-01-01", "2026-02-01", "2026-02-15", "2026-03-01", "2026-04-01", "2026-05-01"]),
})

# --- Wykrywanie ---------------------------------------------------------------------------------------------------------------
customers_raw.duplicated().sum()                                 # ile wierszy to pełne duplikaty
customers_raw.duplicated(subset="customer_id")                   # maska: True dla DRUGIEGO i kolejnych wystąpień klucza (keep="first")
customers_raw[customers_raw.duplicated("customer_id", keep=False)]    # keep=False: zaznacz WSZYSTKIE wiersze z powtórzonym kluczem
customers_raw["customer_id"].is_unique                           # test unikalności klucza (asercja jakości)
# SQL: GROUP BY customer_id HAVING COUNT(*) > 1

# --- Usuwanie: drop_duplicates (parametr keep) ----------------------------------------------------------------------------------
customers_raw.drop_duplicates()                                  # pełne duplikaty wierszy | SELECT DISTINCT *
customers_raw.drop_duplicates(subset="customer_id")                         # keep="first" (domyślnie): zostaw pierwszy
customers_raw.drop_duplicates(subset="customer_id", keep="last")            # zostaw ostatni
customers_raw.drop_duplicates(subset="customer_id", keep=False)             # usuń WSZYSTKIE wiersze, które mają duplikat

# --- Wzorzec "zostaw najnowszy rekord na klienta" (SQL: ROW_NUMBER() OVER (PARTITION BY id ORDER BY updated_at DESC) = 1) ----------
latest = (customers_raw
          .sort_values("updated_at")                             # najpierw sortuj po dacie rosnąco
          .drop_duplicates(subset="customer_id", keep="last"))   # ostatni = najnowszy
# dplyr: arrange(desc(updated_at)) |> distinct(customer_id, .keep_all = TRUE)
# Alternatywa: customers_raw.loc[customers_raw.groupby("customer_id")["updated_at"].idxmax()]
```

### 3.3 Transformacje typów

```python
orders.astype({"order_id": "int32", "qty": "int8", "category": "category"})   # kilka kolumn naraz (słownik)

# --- Tekst -> liczba: to_numeric z errors="coerce" (błędne wartości stają się NaN zamiast wyjątku) ------------------------------
raw = pd.Series(["1 234,50 zł", "99,90 zł", "brak", "2 000 zł"])
cleaned = (raw.str.replace(r"[^\d,]", "", regex=True)      # usuń wszystko poza cyframi i przecinkiem
              .str.replace(",", ".", regex=False))         # przecinek dziesiętny -> kropka
pd.to_numeric(cleaned, errors="coerce")                    # "" po czyszczeniu ("brak") -> NaN; 1234.5, 99.9, NaN, 2000.0

# --- Kategorie: oszczędność pamięci przy kolumnach o niskiej liczbie unikalnych wartości ------------------------------------------
before = orders["category"].memory_usage(deep=True)
cat = orders["category"].astype("category")
after = cat.memory_usage(deep=True)                        # typowo kilkukrotnie mniej
sizes = pd.CategoricalDtype(["S", "M", "L"], ordered=True) # kategoria uporządkowana (porównania <, >, sortowanie wg poziomów)
pd.Series(["M", "S", "L"], dtype=sizes) < "L"              # R: factor(..., ordered = TRUE)

# --- Inne konwersje ----------------------------------------------------------------------------------------------------------------
orders["order_id"].astype("string")                        # nullable string dtype
pd.to_numeric(orders["qty"], downcast="integer")           # automatycznie najmniejszy typ całkowity
orders["flag"] = orders["status"].eq("completed").astype("int8")    # bool -> 0/1
```

### 3.4 Czyszczenie tekstu: akcesor `.str`

Metody `.str.*` działają na **całej kolumnie naraz** (bez pętli), a brakujące wartości (NaN) przechodzą bez błędu.

```python
emails = pd.Series(["  Jan.Kowalski@Firma.PL ", "anna@shop.de", "bad-email", None])

emails.str.strip()                                   # obcina spacje na brzegach  | SQL: TRIM  | R: str_trim
emails.str.lower()                                   # LOWER | upper(), title()
emails.str.strip().str.lower()                       # łańcuch metod
emails.str.replace(r"\s+", " ", regex=True)          # RegEx: wielokrotne spacje -> jedna  | R: str_squish
emails.str.replace(".", "_", regex=False)            # regex=False: dosłowny tekst (kropka to nie "dowolny znak")
emails.str.contains("@", na=False)                   # maska; na=False: NaN traktuj jako False (inaczej błąd w filtrze)
emails.str.contains("firma", case=False, na=False)   # bez rozróżniania wielkości liter | SQL: ILIKE '%firma%'
emails.str.startswith("anna", na=False)
emails.str.len()                                     # długość
emails.str.slice(0, 4)                               # podciąg | emails.str[:4]  (SQL: SUBSTRING)
emails.str.zfill(25)                                 # dopełnienie zerami do długości (np. kody)

# --- Ekstrakcja RegEx -------------------------------------------------------------------------------------------------
emails.str.strip().str.extract(r"@(.+)$", expand=False)      # domena: pierwsza grupa przechwytująca -> Series
emails.str.split("@", expand=True)                            # podział na kolumny (expand=True -> DataFrame)
emails.str.findall(r"[a-z]+")                                 # wszystkie dopasowania (lista na wiersz)
emails.str.count(r"\.")                                       # liczba wystąpień wzorca

# Sklejanie: customers["country"].str.cat(customers["segment"], sep="_")   |  R: str_c() / unite()
```

#### Case study: parsowanie surowych logów (cyber)

```python
raw_logs = pd.DataFrame({"line": [
    '2026-11-10 08:14:02 203.0.113.7 "POST /wp-login.php HTTP/1.1" 401 532',
    '2026-11-10 08:14:05 203.0.113.7 "POST /wp-login.php HTTP/1.1" 401 532',
    '2026-11-10 08:15:11 10.0.0.5 "GET /dashboard HTTP/1.1" 200 10432',
    '2026-11-10 08:16:40 198.51.100.23 "GET /admin/../../etc/passwd HTTP/1.1" 403 211',
    '2026-11-10 08:17:03 45.33.32.156 "GET /search?q=1%27%20OR%201=1-- HTTP/1.1" 500 87',
]})

# str.extract z NAZWANYMI grupami (?P<nazwa>...) od razu zwraca DataFrame z kolumnami o tych nazwach
pattern = r'^(?P<ts>\S+ \S+) (?P<src_ip>\S+) "(?P<method>\S+) (?P<path>\S+) [^"]+" (?P<status>\d{3}) (?P<bytes>\d+)$'

parsed = (raw_logs["line"].str.extract(pattern)
          .assign(ts=lambda d: pd.to_datetime(d["ts"]))
          .astype({"status": "int16", "bytes": "int64"}))

# Klasyfikacja zagrożeń: str.contains + np.select (wektorowe CASE WHEN; pierwszy spełniony warunek wygrywa)
conditions = [
    parsed["path"].str.contains(r"union.+select|%27|--", case=False, regex=True),
    parsed["path"].str.contains(r"\.\./|etc/passwd", regex=True),
    parsed["path"].str.contains(r"wp-login|/admin", regex=True) & parsed["status"].isin([401, 403]),
]
parsed["threat"] = np.select(conditions, ["SQL injection", "Path traversal", "Auth probing"], default="clean")
# R: str_match() + case_when() | Power Query: Split Column by Delimiter + Conditional Column
```

### 3.5 Daty i czas: akcesor `.dt`

```python
# --- Parsowanie: to_datetime ----------------------------------------------------------------------------------------------
pd.to_datetime(pd.Series(["2026-11-10", "2026-11-11"]))                                  # ISO: zadziała bez formatu
pd.to_datetime(pd.Series(["10.11.2026", "11.11.2026"]), format="%d.%m.%Y")               # jawny format: SZYBSZY i jednoznaczny
pd.to_datetime(pd.Series(["10.11.2026", "błąd"]), format="%d.%m.%Y", errors="coerce")    # błędne -> NaT (odpowiednik NaN dla dat)
pd.to_datetime(pd.Series(["2026-11-10 14:35:20"]), utc=True)                             # od razu strefa UTC
pd.to_datetime(pd.Series(["2026-11-10", "10 Nov 2026"]), format="mixed")                 # mieszane formaty (wolniej)
# R: ymd()/dmy() | SQL: TO_DATE / CAST

# --- Komponenty daty: .dt.* (wektorowo) ---------------------------------------------------------------------------------------
d = orders["order_date"].dt
orders.assign(
    year=d.year, month=d.month, day=d.day,
    quarter=d.quarter, weekday=d.dayofweek,               # dayofweek: poniedziałek = 0
    weekday_name=d.day_name(),                             # nazwa (po polsku: day_name(locale="pl_PL.UTF-8"), wymaga locale w systemie)
    iso_week=d.isocalendar().week,
    month_start=d.to_period("M").dt.to_timestamp(),       # DATE_TRUNC('month', ...) | R: floor_date(x, "month")
)
logs["ts"].dt.hour                                         # godzina | EXTRACT(HOUR FROM ts)
logs["ts"].dt.floor("15min")                               # kubełki 15-minutowe (agregacje logów)
orders["order_date"].dt.normalize()                        # zerowanie czasu (data bez godziny)

# --- Strefy czasowe -----------------------------------------------------------------------------------------------------------------
logs["ts"].dt.tz_convert("UTC")                            # konwersja (ten sam MOMENT, inny zapis zegara)
# naive -> aware: .dt.tz_localize("Europe/Warsaw") (przypisanie strefy; dla czasu letniego: ambiguous=, nonexistent=)

# --- Różnice i arytmetyka -------------------------------------------------------------------------------------------------------------
(orders["order_date"] - pd.Timestamp("2026-01-01")).dt.days            # SQL: DATEDIFF(day, ...)  | różnica = Timedelta
(logs["ts"].diff()).dt.total_seconds()                                 # odstęp między kolejnymi zdarzeniami w sekundach
orders["order_date"] + pd.Timedelta(days=7)                            # dodanie 7 dni
orders["order_date"] + pd.offsets.MonthEnd(0)                          # koniec miesiąca (offsety kalendarzowe)
((pd.Timestamp("2026-11-10") - customers["signup_date"]).dt.days / 365.25).round(1)   # staż klienta w latach

# --- Generowanie kalendarza (dim_date) i resampling -------------------------------------------------------------------------------------
pd.date_range("2026-01-01", periods=12, freq="MS")                     # początki miesięcy (MS = Month Start; ME = Month End)
orders_ok.resample("MS", on="order_date")["revenue"].sum()             # agregacja do miesięcy (GROUP BY date_trunc('month'))
orders_ok.groupby(pd.Grouper(key="order_date", freq="W-MON"))["revenue"].sum()   # tygodnie (liczone od poniedziałku)

# --- Praktyka: sessionizacja logów (nowa sesja po przerwie > 30 min) ----------------------------------------------------------------------
sessions_log = (logs.sort_values(["src_ip", "ts"])
                .assign(gap_sec=lambda d: d.groupby("src_ip")["ts"].diff().dt.total_seconds())
                .assign(new_session=lambda d: d["gap_sec"].isna() | (d["gap_sec"] > 30 * 60))
                .assign(session_id=lambda d: d.groupby("src_ip")["new_session"].cumsum()))
```

### 3.6 Kolumny warunkowe i binning (SQL: `CASE WHEN`)

```python
# --- np.where: IF / ELSE wektorowo -----------------------------------------------------------------------
orders["signed_rev"] = np.where(orders["status"] == "returned", -orders["revenue"], orders["revenue"])

# --- np.select: wiele warunków (CASE WHEN ... WHEN ... ELSE) ----------------------------------------------
orders["tier"] = np.select(
    condlist=[orders["revenue"] >= 1000, orders["revenue"] >= 300],     # sprawdzane w KOLEJNOŚCI, pierwszy True wygrywa
    choicelist=["A", "B"],
    default="C",                                                         # ELSE
)                                                                        # dplyr: case_when()

# --- Binning: pd.cut (przedziały zadane) i pd.qcut (kwantyle, równoliczne grupy; SQL: NTILE) ------------------
orders["rev_band"] = pd.cut(orders["revenue"], bins=[0, 100, 500, 1000, np.inf],
                            labels=["low", "mid", "high", "premium"], right=False)     # [0,100), [100,500), ...
orders["rev_quartile"] = pd.qcut(orders["revenue"], q=4, labels=["Q1", "Q2", "Q3", "Q4"])

# --- Mapowanie słownikiem (lookup bez JOIN-a) i przycinanie wartości --------------------------------------------
margin_by_category = {"Electronics": 0.12, "Home": 0.25, "Fashion": 0.40, "Books": 0.18}
orders["margin_pct"] = orders["category"].map(margin_by_category)        # Series.map(dict): wbudowane, wektorowe
orders["revenue_capped"] = orders["revenue"].clip(upper=1200)            # winsoryzacja górna
orders["revenue"].mask(orders["status"] == "cancelled", 0)               # mask: zamień na 0 tam, gdzie warunek True
```

---

## 4. Zaawansowana manipulacja danych

### 4.1 Łączenie tabel: `merge()` i `concat()`

```python
# SQL: SELECT * FROM orders o LEFT JOIN customers c ON o.customer_id = c.customer_id
# dplyr: left_join(customers, by = "customer_id")
orders.merge(customers, on="customer_id", how="left", validate="m:1")
# validate="m:1": kontrola kardynalności (wiele zamówień : 1 klient). Jeśli klucz po prawej NIE jest unikalny,
# dostaniesz błąd zamiast cichego rozmnożenia wierszy (najczęstszy błąd JOIN-ów!)

# --- Parametr how: typ JOIN-a ------------------------------------------------------------------------------------------
orders.merge(customers, on="customer_id", how="inner")    # INNER JOIN: tylko pasujące
orders.merge(customers, on="customer_id", how="left")     # LEFT JOIN: wszystkie zamówienia
orders.merge(customers, on="customer_id", how="right")    # RIGHT JOIN
orders.merge(customers, on="customer_id", how="outer")    # FULL OUTER JOIN
orders.head(3).merge(customers.head(3), how="cross")      # CROSS JOIN (iloczyn kartezjański)

# --- Klucze o różnych nazwach (left_on / right_on) -------------------------------------------------------------------------
dim = customers.rename(columns={"customer_id": "id"})
(orders.merge(dim, left_on="customer_id", right_on="id", how="left")
       .drop(columns="id"))                                # kolumna klucza z prawej tabeli zostaje, więc ją usuń
# Klucz złożony: on=["customer_id", "country"]; kolizje nazw kolumn: suffixes=("_ord", "_cust")

# --- Anti-join / semi-join (SQL: NOT EXISTS / EXISTS) ------------------------------------------------------------------------
orphans = orders[~orders["customer_id"].isin(customers["customer_id"])]          # ANTI JOIN: zamówienia bez klienta (kontrola jakości)
orphans_alt = (orders.merge(customers[["customer_id"]], on="customer_id", how="left", indicator=True)
                     .query("_merge == 'left_only'").drop(columns="_merge"))     # indicator=True dodaje kolumnę _merge: both/left_only/right_only
pl_orders = orders[orders["customer_id"].isin(customers.loc[customers["country"] == "PL", "customer_id"])]   # SEMI JOIN
# dplyr: anti_join() / semi_join() | Power Query: Merge Queries -> Left Anti / Inner

# --- Join po indeksie (szybki; DataFrame.join) ----------------------------------------------------------------------------------
orders.join(customers.set_index("customer_id"), on="customer_id")                # join: lewa kolumna <-> prawy indeks

# --- merge_asof: join "najbliższy poprzedni" (AS-OF; SQL: LATERAL JOIN / ROW_NUMBER) -----------------------------------------------
# Przykład: do każdego zdarzenia dołącz wersję blocklisty obowiązującą w danej chwili
rules = pd.DataFrame({"effective_from": logs["ts"].iloc[[0, 100, 200]].reset_index(drop=True),   # ten sam dtype i strefa po obu stronach!
                      "blocklist_ver": ["v1", "v2", "v3"]})
pd.merge_asof(logs.sort_values("ts"), rules.sort_values("effective_from"),                        # OBA zbiory muszą być posortowane po kluczu
              left_on="ts", right_on="effective_from", direction="backward")                       # tolerance=pd.Timedelta("1h"), by="src_ip" też możliwe

# --- concat: łączenie pionowe (UNION ALL) i poziome ---------------------------------------------------------------------------------------
part1, part2 = orders.iloc[:30], orders.iloc[30:]
pd.concat([part1, part2], ignore_index=True)                       # UNION ALL; ignore_index=True: nowy ciągły indeks
pd.concat([part1, part2], keys=["H1", "H2"], names=["part"])       # keys: dodaje poziom indeksu z etykietą źródła
pd.concat([part1, part2]).drop_duplicates()                        # UNION (z deduplikacją)
# concat(axis=1): sklejanie kolumn wg INDEKSU (uważaj na niedopasowane indeksy). R: bind_rows() / bind_cols()
```

### 4.2 Agregacje i grupowanie: `groupby()` + `.agg()`

```python
# SQL: SELECT category, COUNT(*) AS n_orders, COUNT(DISTINCT customer_id) AS n_customers, SUM(revenue), AVG(revenue)
#      FROM orders WHERE status = 'completed' GROUP BY category
# dplyr: group_by(category) |> summarise(n = n(), ...)
by_cat = (orders_ok
          .groupby("category", as_index=False)            # as_index=False: klucz zostaje kolumną (nie indeksem)
          .agg(                                            # NAMED AGGREGATION: nazwa_wyniku=("kolumna", "funkcja")
              n_orders=("order_id", "size"),               # COUNT(*): size liczy wiersze (count() liczy tylko NIE-NaN)
              n_customers=("customer_id", "nunique"),      # COUNT(DISTINCT ...)
              total_revenue=("revenue", "sum"),
              aov=("revenue", "mean"),                     # Average Order Value
              max_order=("revenue", "max"),
          )
          .sort_values("total_revenue", ascending=False))

# Percentyl per grupa (nie ma go w named aggregation bez lambdy): policz osobno i dołącz
p90 = orders_ok.groupby("category")["revenue"].quantile(0.9).rename("p90_revenue")
by_cat.merge(p90, on="category")

# --- HAVING = filtr PO agregacji -----------------------------------------------------------------------------------------------
(orders_ok.groupby("customer_id", as_index=False)
          .agg(n=("order_id", "size"), total=("revenue", "sum"))
          .query("n >= 3"))                                # SQL: HAVING COUNT(*) >= 3

# --- Wiele kluczy, daty, brakujące klucze -------------------------------------------------------------------------------------------
orders_ok.groupby(["category", "status"]).size()                           # liczności; wynik z MultiIndex -> .unstack() lub .reset_index(name="n")
orders_ok.groupby(pd.Grouper(key="order_date", freq="MS"))["revenue"].sum()    # grupowanie po miesiącu
orders.groupby("category", dropna=False)["revenue"].sum()                  # dropna=False: NaN w kluczu jako osobna grupa (domyślnie wiersze z NaN w kluczu są POMIJANE!)
# Dla kolumn kategorycznych: observed=True pomija puste kombinacje (inaczej wiersze z zerami dla nieistniejących par)

# --- Wiele funkcji na wielu kolumnach (R: across) --------------------------------------------------------------------------------------
orders_ok.groupby("category")[["revenue", "qty"]].agg(["sum", "mean"])   # kolumny z MultiIndex (poziom 0 = zmienna, poziom 1 = funkcja)

# --- transform: wynik agregatu "rozlany" na każdy wiersz (SQL: SUM(x) OVER (PARTITION BY g)) ----------------------------------------------
orders_ok.assign(
    cat_total=lambda d: d.groupby("category")["revenue"].transform("sum"),
    share_in_cat=lambda d: d["revenue"] / d.groupby("category")["revenue"].transform("sum"),
)                                                                          # dplyr: group_by(category) |> mutate(share = revenue / sum(revenue))

# --- Filtr na poziomie grupy bez .filter(lambda): maska z transform ------------------------------------------------------------------------
orders_ok[orders_ok.groupby("customer_id")["order_id"].transform("size") >= 3]    # tylko klienci z >= 3 zamówieniami

# --- TOP N na grupę (SQL: ROW_NUMBER() ... WHERE rn <= N) ------------------------------------------------------------------------------------
top2_per_cat = (orders_ok.sort_values("revenue", ascending=False)
                         .groupby("category")
                         .head(2))                                         # dplyr: slice_max(revenue, n = 2, by = category)
best_per_customer = orders_ok.loc[orders_ok.groupby("customer_id")["revenue"].idxmax()]   # dokładnie 1 wiersz (największe zamówienie) na klienta
orders_ok.nlargest(5, "revenue")                                           # globalny TOP 5 (SQL: ORDER BY ... DESC LIMIT 5)

# --- Tabela krzyżowa: crosstab (z sumami brzegowymi i procentami) -----------------------------------------------------------------------------
pd.crosstab(orders["category"], orders["status"], margins=True, normalize="index")   # udziały wierszowe; margins=True: wiersz/kolumna "All"
```

### 4.3 Funkcje okna (Window Functions)

Mapowanie `OVER (PARTITION BY ... ORDER BY ...)` → **`groupby()` + `transform` / `shift` / `rank` / `cumsum` / `rolling`**. Zasada: najpierw **sortuj**, potem licz (kolejność wierszy = ORDER BY).

| SQL | Pandas | dplyr |
|---|---|---|
| `ROW_NUMBER() OVER (PARTITION BY g ORDER BY d)` | `groupby("g").cumcount() + 1` (po `sort_values`) | `row_number()` |
| `RANK()` / `DENSE_RANK()` | `groupby("g")["x"].rank(method="min"/"dense", ascending=False)` | `min_rank()` / `dense_rank()` |
| `LAG(x)` / `LEAD(x)` | `groupby("g")["x"].shift(1)` / `.shift(-1)` | `lag()` / `lead()` |
| `SUM(x) OVER (PARTITION BY g)` | `groupby("g")["x"].transform("sum")` | `group_by + mutate(sum(x))` |
| `SUM(x) OVER (ORDER BY d)` (running) | `groupby("g")["x"].cumsum()` | `cumsum()` |
| `AVG(x) OVER (ROWS 2 PRECEDING)` | `rolling(3).mean()` | `slider::slide_dbl()` |
| `NTILE(4)` | `pd.qcut(x, 4)` | `ntile(x, 4)` |

```python
# ---- Sekwencja zamówień klienta: numeracja, rankingi, LAG/LEAD, sumy narastające -------------------------------------------------
o = orders_ok.sort_values(["customer_id", "order_date"])            # ORDER BY customer_id, order_date
g = o.groupby("customer_id")                                        # PARTITION BY customer_id

o = o.assign(
    order_seq=g.cumcount() + 1,                                     # ROW_NUMBER()
    rank_by_rev=g["revenue"].rank(method="min", ascending=False),   # RANK() (remisy -> ta sama pozycja, luka w numeracji)
    dense_by_rev=g["revenue"].rank(method="dense", ascending=False),# DENSE_RANK()
    prev_order_date=g["order_date"].shift(1),                       # LAG(order_date, 1)
    next_revenue=g["revenue"].shift(-1),                            # LEAD(revenue, 1)
    cum_revenue=g["revenue"].cumsum(),                              # suma narastająca w obrębie klienta
    share_of_customer=o["revenue"] / g["revenue"].transform("sum"), # udział w sumie klienta
)
o["days_since_prev"] = (o["order_date"] - o["prev_order_date"]).dt.days      # różnica w dniach (NaN dla pierwszego zamówienia)

# ---- Szereg czasowy: MoM, YTD, średnia krocząca ------------------------------------------------------------------------------------
monthly = (orders_ok
           .assign(month=lambda d: d["order_date"].dt.to_period("M").dt.to_timestamp())
           .groupby("month", as_index=False).agg(revenue=("revenue", "sum"))
           .sort_values("month")                                    # KLUCZOWE przed shift()/cumsum()/rolling()
           .assign(
               prev_month=lambda d: d["revenue"].shift(1),          # LAG(revenue)
               mom_pct=lambda d: d["revenue"].pct_change(),         # = revenue / lag(revenue) - 1
               diff=lambda d: d["revenue"].diff(),                  # = revenue - lag(revenue)
               ytd=lambda d: d["revenue"].cumsum(),                 # SUM() OVER (ORDER BY month)
               ma3=lambda d: d["revenue"].rolling(3).mean(),        # średnia krocząca z 3 miesięcy (pierwsze 2 = NaN)
               ma3_min1=lambda d: d["revenue"].rolling(3, min_periods=1).mean(),   # min_periods=1: licz od pierwszego okna
           ))
# Warianty okien: .expanding().mean() (od początku do bieżącego), .ewm(span=3).mean() (średnia wykładnicza),
#                 .rolling(3, center=True) (okno wyśrodkowane)

# ---- Okno kroczące PER GRUPA (np. 3-miesięczna średnia osobno dla każdej kategorii) ------------------------------------------------------
monthly_cat = (orders_ok
               .assign(month=lambda d: d["order_date"].dt.to_period("M").dt.to_timestamp())
               .groupby(["category", "month"], as_index=False)["revenue"].sum()
               .sort_values(["category", "month"]))
ma3_by_cat = (monthly_cat.groupby("category")["revenue"]
              .rolling(3, min_periods=1).mean()
              .reset_index(level=0, drop=True))                     # rolling po groupby daje MultiIndex; zdejmij poziom grupy, by wyrównać z oryginałem
monthly_cat["ma3"] = ma3_by_cat                                      # przypisanie wyrównuje się po indeksie

# ---- Okno CZASOWE: rolling("10min") liczy po zegarze, nie po liczbie wierszy (patrz Dodatek C) --------------------------------------------------
```

### 4.4 Reshaping danych: `melt()` i `pivot_table()`

```python
# Dane w formacie WĄSKIM / długim (tidy): 1 wiersz = miesiąc x kategoria
monthly_cat = (orders_ok
               .assign(month=lambda d: d["order_date"].dt.to_period("M").dt.to_timestamp())
               .groupby(["month", "category"], as_index=False)["revenue"].sum())

# ---- LONG -> WIDE: pivot_table (SQL: SUM(CASE WHEN ...) | Power Query: Pivot Column | dplyr: pivot_wider) ---------------------------
wide = (monthly_cat
        .pivot_table(index="month",            # wiersze
                     columns="category",       # wartości z tej kolumny staną się nazwami kolumn
                     values="revenue",
                     aggfunc="sum",            # agreguje, gdy kombinacja się powtarza (pivot() w takiej sytuacji rzuca błąd)
                     fill_value=0)             # puste kombinacje -> 0 zamiast NaN
        .rename_axis(columns=None)             # usuń napis "category" nad nagłówkami
        .reset_index())                        # month z indeksu wraca do kolumny

# Tabela z sumami brzegowymi (jak tabela przestawna w Excelu)
orders.pivot_table(index="category", columns="status", values="revenue",
                   aggfunc="sum", fill_value=0, margins=True, margins_name="Total")

# ---- WIDE -> LONG: melt (SQL: UNPIVOT | Power Query: Unpivot Other Columns | dplyr: pivot_longer) ----------------------------------------
long = wide.melt(id_vars="month",              # kolumny, które ZOSTAJĄ jako identyfikatory
                 var_name="category",          # nazwa nowej kolumny na nazwy "roztopionych" kolumn
                 value_name="revenue")         # nazwa nowej kolumny na wartości
# Format długi jest wymagany przez Seaborn (hue=, col=); format szeroki jest czytelniejszy w raportach.

# ---- Dodatki: stack/unstack (przesuwanie poziomów indeksu), explode (lista -> wiersze) ------------------------------------------------------
orders_ok.groupby(["category", "status"])["revenue"].sum().unstack(fill_value=0)       # MultiIndex -> kolumny (jak pivot)
pd.DataFrame({"user": ["a", "b"], "ips": [["1.1.1.1", "2.2.2.2"], ["3.3.3.3"]]}).explode("ips")   # każdy element listy -> osobny wiersz
```

### 4.5 Wydajność: jak pisać szybki Pandas

```python
# 1) Wektoryzacja zamiast pętli po wierszach: operacje na całych kolumnach działają w skompilowanym C/NumPy.
#    Poniższa tabela pokazuje, czym zastąpić typowe wzorce "pętli po wierszach":
```

| Chcesz zrobić | Użyj (wektorowo / wbudowanie) |
|---|---|
| Warunek na każdym wierszu (IF/CASE) | `np.where`, `np.select`, `pd.cut`, `Series.map(dict)` |
| Suma / licznik narastająco | `cumsum()`, `cumcount()`, `cummax()` |
| Wartość z poprzedniego wiersza | `shift()`, `diff()`, `pct_change()` |
| Operacja tekstowa | akcesor `.str.*` |
| Operacja na datach | akcesor `.dt.*`, `pd.to_datetime` |
| Wynik zależny od grupy | `groupby().transform()` / `.agg()` |
| Zamiana wartości wg słownika | `Series.map(dict)` / `Series.replace(dict)` |
| Własna logika niewyrażalna inaczej | najpierw sprawdź `eval`/`query`, potem NumPy `np.vectorize` (tylko ostateczność) |

```python
# 2) Typy i pamięć ---------------------------------------------------------------------------------------------------
#    - kolumny tekstowe o niskiej kardynalności -> category (wielokrotnie mniej RAM, szybszy groupby)
#    - liczby całkowite -> int32/int16/int8 (downcast), flagi -> bool
#    - przy wczytywaniu: usecols=, dtype=, parse_dates=; pliki > RAM -> chunksize / Parquet / DuckDB
big = pd.DataFrame({
    "revenue": rng.uniform(20, 1500, 1_000_000),
    "status": rng.choice(["completed", "returned", "cancelled"], 1_000_000),
})
big["status"] = big["status"].astype("category")                 # groupby po kategorii jest znacznie szybszy
big["signed"] = np.where(big["status"] == "returned", -big["revenue"], big["revenue"])   # 1 mln wierszy: milisekundy

# 3) query() / eval(): szybkie wyrażenia na dużych ramkach (bez tworzenia tymczasowych tablic) ------------------------------
big.query("revenue > 1000 and status == 'completed'")
big.eval("gross = revenue * 1.23")                               # zwraca nowy DataFrame z kolumną (inplace tylko, gdy musisz)

# 4) Przetwarzanie dużych plików porcjami (pętla po CHUNKACH, nie po wierszach) ----------------------------------------------------
parts = []
for chunk in pd.read_csv("big_logs.csv", chunksize=500_000, usecols=["src_ip", "event"]):
    parts.append(chunk.query("event == 'LOGIN_FAIL'").groupby("src_ip").size())     # agregacja częściowa na porcji
fails_by_ip = pd.concat(parts).groupby(level=0).sum()                                # scalenie wyników częściowych

# 5) Pomiar: w Jupyterze %timeit wyrażenie; w skrypcie: import timeit / time.perf_counter() ------------------------------------------------
# 6) Gdy Pandas to za mało: DuckDB (SQL, out-of-core), Polars (lazy, wielowątkowy), dtype_backend="pyarrow".
# 7) Method chaining + .pipe(): czytelny łańcuch bez zmiennych tymczasowych; unikaj inplace=True (nie daje zysku wydajności).
```

---

## 5. Wizualizacja

Dane pomocnicze używane w tej sekcji (jak w 4.3 i 4.4):

```python
monthly = (orders_ok.assign(month=lambda d: d["order_date"].dt.to_period("M").dt.to_timestamp())
                    .groupby("month")["revenue"].sum())                           # Series z indeksem czasowym

monthly_cat = (orders_ok.assign(month=lambda d: d["order_date"].dt.to_period("M").dt.to_timestamp())
                        .groupby(["month", "category"], as_index=False)["revenue"].sum())

wide = monthly_cat.pivot_table(index="month", columns="category", values="revenue", aggfunc="sum", fill_value=0)

sns.set_theme(style="whitegrid", context="notebook", palette="deep")              # globalny motyw Seaborn (dotyczy także wykresów Pandas)
```

### 5.1 Szybkie wykresy w Pandas (`.plot.*`)

`DataFrame.plot` / `Series.plot` to nakładka na Matplotlib: najszybsza droga do wykresu eksploracyjnego. Indeks = oś X, kolumny = serie.

```python
# ---- Liniowy: szereg czasowy ---------------------------------------------------------------------------------------------
monthly.plot.line(marker="o", figsize=(9, 4), title="Przychód miesięczny", ylabel="PLN", grid=True)
plt.tight_layout(); plt.show()                         # tight_layout: dopasowanie marginesów; show(): wyświetl

# ---- Słupkowy: poziomy, posortowany (czytelne etykiety kategorii) ------------------------------------------------------------
(orders_ok.groupby("category")["revenue"].sum()
          .sort_values()                               # rosnąco => na wykresie poziomym największy słupek będzie u góry
          .plot.barh(color="#2C7FB8", figsize=(7, 4), title="Przychód wg kategorii"))

# ---- Słupkowy skumulowany (stacked) z tabeli szerokiej ------------------------------------------------------------------------
wide_plot = wide.copy()
wide_plot.index = wide_plot.index.strftime("%Y-%m")    # czytelne etykiety osi X zamiast pełnych timestampów
wide_plot.plot.bar(stacked=True, figsize=(10, 4), rot=45, title="Przychód miesięczny wg kategorii")
plt.legend(title="Kategoria", bbox_to_anchor=(1.02, 1), loc="upper left")    # legenda poza wykresem

# ---- Kołowy: udział statusów (używaj oszczędnie: tylko kilka kategorii) ---------------------------------------------------------
orders["status"].value_counts().plot.pie(autopct="%.1f%%", startangle=90, ylabel="", figsize=(5, 5))

# ---- Inne: histogram, boxplot, scatter ---------------------------------------------------------------------------------------------
orders_ok["revenue"].plot.hist(bins=20, edgecolor="white", title="Rozkład wartości zamówień")
orders_ok.boxplot(column="revenue", by="category", figsize=(7, 4))                    # rozkład wg kategorii
orders_ok.plot.scatter(x="qty", y="revenue", alpha=0.6, figsize=(6, 4))
```

### 5.2 Wizualizacje statystyczne w Seaborn

Seaborn pracuje na **długim formacie** (tidy) i sam agreguje dane. Parametr `hue=` dodaje wymiar koloru (jak `colour = ` w ggplot2).

```python
# ---- barplot: AUTOMATYCZNA agregacja (domyślnie ŚREDNIA z przedziałem ufności 95%) ----------------------------------------------
sns.barplot(data=orders_ok, x="category", y="revenue")                              # średni przychód + słupki błędów (bootstrap CI)
sns.barplot(data=orders_ok, x="category", y="revenue",
            estimator="sum",                      # zmień agregat: "sum", "median", np.max ... (SQL: SUM)
            errorbar=None,                        # wyłącz słupki błędów (seaborn >= 0.12; wcześniej ci=None)
            order=orders_ok.groupby("category")["revenue"].sum().sort_values(ascending=False).index)   # kolejność słupków
plt.title("Przychód wg kategorii"); plt.ylabel("PLN"); plt.show()

# ---- countplot: liczność kategorii (jak GROUP BY + COUNT(*)); dplyr: geom_bar() ---------------------------------------------------
sns.countplot(data=orders, x="status", hue="category",
              order=orders["status"].value_counts().index)

# ---- lineplot: wiele serii; duplikaty X są agregowane (domyślnie średnia + CI) -----------------------------------------------------
sns.lineplot(data=monthly_cat, x="month", y="revenue", hue="category", marker="o", errorbar=None)
plt.xticks(rotation=45); plt.tight_layout(); plt.show()
sns.lineplot(data=orders_ok, x="order_date", y="revenue", estimator="sum", errorbar=None)   # suma dzienna (agregacja w locie)

# ---- Rozkłady i zależności ---------------------------------------------------------------------------------------------------------------
sns.histplot(data=orders_ok, x="revenue", bins=15, kde=True)                          # histogram + krzywa gęstości
sns.boxplot(data=orders_ok, x="category", y="revenue")                                # mediana, kwartyle, odstające
sns.scatterplot(data=orders_ok, x="qty", y="revenue", hue="category")
sns.regplot(data=orders_ok, x="qty", y="revenue")                                     # scatter + linia regresji z przedziałem ufności

# ---- Heatmapa (cyber): liczba zdarzeń wg godziny i typu --------------------------------------------------------------------------------
hour_event = (logs.assign(hour=logs["ts"].dt.hour)
                  .pivot_table(index="event", columns="hour", values="user", aggfunc="count", fill_value=0))
sns.heatmap(hour_event, annot=True, fmt="d", cmap="mako", cbar_kws={"label": "Liczba zdarzeń"})   # annot: liczby w komórkach
plt.xlabel("Godzina"); plt.ylabel(""); plt.show()

# Macierz korelacji: sns.heatmap(df.corr(numeric_only=True), annot=True, fmt=".2f", cmap="coolwarm", vmin=-1, vmax=1)

# ---- Fasety (małe wielokrotności; ggplot2: facet_wrap / facet_grid) -----------------------------------------------------------------------
sns.relplot(data=monthly_cat, x="month", y="revenue", col="category", col_wrap=2,
            kind="line", height=3, aspect=1.4, facet_kws={"sharey": False})            # panel na każdą kategorię, niezależne osie Y
sns.catplot(data=orders, x="status", y="revenue", col="category", kind="box", col_wrap=2)
```

### 5.3 Kontrola nad kompozycją w Matplotlib: `subplots`

`fig` = całe płótno, `ax` = pojedynczy wykres. Kluczowy wzorzec: **tworzysz siatkę, a potem przekazujesz `ax=` do Pandas/Seaborn**.

```python
fig, axes = plt.subplots(2, 2, figsize=(14, 9), constrained_layout=True)   # siatka 2x2; constrained_layout: automatyczne odstępy
# axes to tablica 2x2: axes[wiersz, kolumna]

# Lewy górny: szereg czasowy (Pandas)
monthly.plot.line(ax=axes[0, 0], marker="o", title="Przychód miesięczny", ylabel="PLN")

# Prawy górny: słupki (Seaborn)
sns.barplot(data=orders_ok, x="category", y="revenue", estimator="sum", errorbar=None, ax=axes[0, 1])
axes[0, 1].set_title("Przychód wg kategorii")
axes[0, 1].yaxis.set_major_formatter("{x:,.0f}")                             # formatowanie osi: separator tysięcy

# Lewy dolny: struktura statusów (Pandas)
orders["status"].value_counts().plot.pie(ax=axes[1, 0], autopct="%.0f%%", ylabel="", title="Statusy zamówień")

# Prawy dolny: rozkład (Seaborn)
sns.histplot(orders_ok["revenue"], bins=15, kde=True, ax=axes[1, 1])
axes[1, 1].axvline(orders_ok["revenue"].mean(), color="red", linestyle="--", label="średnia")   # linia pomocnicza
axes[1, 1].legend()

fig.suptitle("Dashboard sprzedaży 2026", fontsize=16, fontweight="bold")     # tytuł całej figury
fig.savefig("out/dashboard.png", dpi=200, bbox_inches="tight")               # zapis: dpi = rozdzielczość, bbox_inches="tight" przycina marginesy
plt.show()

# ---- Inne układy -------------------------------------------------------------------------------------------------------------------------------
fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 4), sharey=True)          # dwa wykresy obok siebie, wspólna oś Y
fig, axes = plt.subplots(2, 1, figsize=(10, 6), sharex=True, gridspec_kw={"height_ratios": [3, 1]})   # nierówne wysokości
fig, axd = plt.subplot_mosaic("AB;CC", figsize=(10, 6))                      # układ mozaikowy: C zajmuje całą dolną szerokość
# Iteracja po panelach (pętla po osiach to OK - to nie wiersze danych):
for ax, cat in zip(axes.flat, ["Electronics", "Home", "Fashion", "Books"]):  # axes.flat: spłaszcza siatkę
    pass
# Daty na osi: import matplotlib.dates as mdates; ax.xaxis.set_major_formatter(mdates.DateFormatter("%b %Y"))
```

---

## 6. Statystyka i modelowanie

Dane syntetyczne z sesji w sklepie online (500 wierszy), spójne z wersją w R:

```python
n = 500
sessions = pd.DataFrame({
    "session_id": np.arange(1, n + 1),
    "device": rng.choice(["mobile", "desktop"], n, p=[0.6, 0.4]),
    "channel": rng.choice(["organic", "paid", "email"], n),
    "pages": rng.poisson(5, n) + 1,                                       # liczba odwiedzonych podstron
})
is_desktop = sessions["device"].eq("desktop")
sessions["time_on_site"] = rng.gamma(shape=2, scale=np.where(is_desktop, 120, 90)).round()          # sekundy
sessions["basket_value"] = (20 + 8 * sessions["pages"] + 25 * is_desktop + rng.normal(0, 30, n)).round(2)
logit = -3 + 0.25 * sessions["pages"] + 0.4 * is_desktop + 0.5 * sessions["channel"].eq("email")    # log-szanse konwersji
sessions["converted"] = rng.binomial(1, 1 / (1 + np.exp(-logit)))                                    # 1 / (1 + e^-x) = funkcja logistyczna
```

### 6.1 Podstawowe miary i korelacje

```python
s = sessions["basket_value"]

# ---- Miary opisowe ---------------------------------------------------------------------------------------------------------
s.mean(); s.median(); s.mode().iloc[0]            # średnia, mediana, dominanta
s.std(); s.var()                                  # ⚠️ Pandas: ddof=1 (odchylenie próbkowe); NumPy np.std(): ddof=0 (populacyjne)!
s.quantile([0.25, 0.5, 0.75, 0.9])                # percentyle
s.skew(); s.kurt()                                # skośność, kurtoza
s.agg(["count", "mean", "median", "std", "min", "max"])   # kilka miar naraz
q1, q3 = s.quantile([0.25, 0.75])
iqr = q3 - q1
outliers = sessions[(s < q1 - 1.5 * iqr) | (s > q3 + 1.5 * iqr)]       # reguła IQR: wartości odstające
sessions.assign(z=(s - s.mean()) / s.std()).query("abs(z) > 3")         # z-score: reguła "3 sigm"
sessions.groupby("device")["time_on_site"].agg(["count", "mean", "median", "std"])   # statystyki wg grup

# ---- Korelacje: Pearson (liniowa) vs Spearman (rangowa, odporna na odstające i zależności monotoniczne) --------------------------
num_cols = ["pages", "time_on_site", "basket_value"]
sessions[num_cols].corr()                                   # macierz korelacji Pearsona | R: cor(df)
sessions[num_cols].corr(method="spearman")                  # Spearman
sessions["pages"].corr(sessions["basket_value"])            # jedna para

r, p = st.pearsonr(sessions["pages"], sessions["basket_value"])         # współczynnik + p-value (H0: brak korelacji)
rho, p_s = st.spearmanr(sessions["pages"], sessions["time_on_site"])
# Interpretacja |r|: <0.3 słaba, 0.3-0.7 umiarkowana, >0.7 silna. Korelacja NIE oznacza przyczynowości.
# R: cor.test()

# ---- Testy istotności -------------------------------------------------------------------------------------------------------------------
desktop = sessions.loc[is_desktop, "time_on_site"]
mobile = sessions.loc[~is_desktop, "time_on_site"]
st.ttest_ind(desktop, mobile, equal_var=False)    # test t Welcha (nie zakłada równych wariancji); H0: równe średnie | R: t.test()
st.mannwhitneyu(desktop, mobile)                  # alternatywa nieparametryczna (gdy rozkład daleki od normalnego)
st.shapiro(s.sample(200, random_state=1))         # test normalności (p < 0.05 -> rozkład nie jest normalny)

tab = pd.crosstab(sessions["channel"], sessions["converted"])           # tabela kontyngencji
chi2, p, dof, expected = st.chi2_contingency(tab)                       # H0: konwersja niezależna od kanału | R: chisq.test()
# expected: liczności oczekiwane; jeśli któraś < 5, użyj st.fisher_exact (tabele 2x2)
pd.crosstab(sessions["channel"], sessions["converted"], normalize="index")   # conversion rate per kanał

st.f_oneway(*[g["basket_value"] for _, g in sessions.groupby("channel")])    # ANOVA: porównanie średnich > 2 grup
# Post-hoc: from statsmodels.stats.multicomp import pairwise_tukeyhsd; pairwise_tukeyhsd(sessions["basket_value"], sessions["channel"])
```

### 6.2 Regresja liniowa (OLS) w `statsmodels`

Formuły identyczne jak w R: `y ~ x1 + x2`. `C(zmienna)` oznacza zmienną kategoryczną (R robi to automatycznie dla faktorów).

```python
m_ols = smf.ols(
    "basket_value ~ pages + C(device) + C(channel, Treatment(reference='organic'))",   # poziom referencyjny wskazany jawnie
    data=sessions,
).fit()
print(m_ols.summary())                              # R: summary(lm(...))
```

**Jak czytać `summary()`:**

```
Górny blok (jakość modelu)
  R-squared / Adj. R-squared : jaka część wariancji Y jest wyjaśniona (Adj. karze za liczbę zmiennych)
  F-statistic / Prob (F)     : test łącznej istotności modelu (p < 0.05 -> model jako całość ma sens)
  AIC / BIC                  : kryteria porównywania modeli (niższe = lepiej)
  No. Observations / Df Resid: liczba obserwacji i stopni swobody reszt
Środkowy blok (współczynniki)
  coef       : zmiana Y przy wzroście X o 1 jednostkę, przy stałych pozostałych zmiennych (ceteris paribus)
               dla zmiennych kategorycznych: różnica względem POZIOMU REFERENCYJNEGO
  std err    : błąd standardowy oszacowania
  t, P>|t|   : statystyka i p-value (p < 0.05 -> współczynnik istotnie różny od 0)
  [0.025 0.975] : 95% przedział ufności
Dolny blok (diagnostyka reszt)
  Omnibus / Jarque-Bera : normalność reszt (p < 0.05 -> reszty nienormalne)
  Durbin-Watson         : autokorelacja reszt (ok. 2 = brak)
  Cond. No.             : bardzo duża wartość -> podejrzenie współliniowości
```

```python
# ---- Wyniki jako DataFrame (odpowiednik broom::tidy w R) ------------------------------------------------------------------------------
coef_tbl = (m_ols.params.rename("coef").to_frame()
            .join(m_ols.bse.rename("std_err"))
            .join(m_ols.pvalues.rename("p_value"))
            .join(m_ols.conf_int().set_axis(["ci_low", "ci_high"], axis=1)))
m_ols.rsquared, m_ols.rsquared_adj, m_ols.aic          # metryki (R: broom::glance)
m_ols.resid; m_ols.fittedvalues                         # reszty i wartości dopasowane (R: broom::augment)

# ---- Predykcja z przedziałami ----------------------------------------------------------------------------------------------------------------
new = pd.DataFrame({"pages": [3, 8], "device": ["mobile", "desktop"], "channel": ["paid", "email"]})
m_ols.get_prediction(new).summary_frame(alpha=0.05)
# mean_ci_*: przedział dla ŚREDNIEJ wartości; obs_ci_*: przedział predykcji dla POJEDYNCZEJ obserwacji (szerszy)

# ---- Interakcje, transformacje, odporne błędy standardowe -------------------------------------------------------------------------------------
smf.ols("basket_value ~ pages * C(device)", data=sessions).fit()          # pages + device + pages:device
smf.ols("np.log(basket_value) ~ pages + np.power(pages, 2)", data=sessions).fit()   # funkcje NumPy wprost w formule
m_robust = smf.ols("basket_value ~ pages + C(device)", data=sessions).fit(cov_type="HC3")   # błędy odporne na heteroskedastyczność

# ---- Diagnostyka ---------------------------------------------------------------------------------------------------------------------------------
sm.qqplot(m_ols.resid, line="s"); plt.show()                                           # normalność reszt (punkty wzdłuż linii)
sns.scatterplot(x=m_ols.fittedvalues, y=m_ols.resid); plt.axhline(0, color="red")      # homoskedastyczność: brak "lejka"
from statsmodels.stats.outliers_influence import variance_inflation_factor
X = m_ols.model.exog
vif = pd.Series([variance_inflation_factor(X, i) for i in range(X.shape[1])], index=m_ols.model.exog_names)   # VIF > 5-10: współliniowość
```

### 6.3 Regresja logistyczna (Logit / GLM)

```python
m_logit = smf.logit("converted ~ pages + C(device) + C(channel)", data=sessions).fit()
print(m_logit.summary())                  # zwróć uwagę: Pseudo R-squ. (McFadden), LLR p-value (łączna istotność)
# Równoważnie (R: glm(..., family = binomial)):
# smf.glm("converted ~ pages + C(device) + C(channel)", data=sessions, family=sm.families.Binomial()).fit()

# ---- Interpretacja: współczynniki są w skali log-odds; exp() daje ILORAZ SZANS (odds ratio) ---------------------------------------------------------
or_tbl = (m_logit.params.rename("odds_ratio").to_frame()
          .join(m_logit.conf_int().set_axis(["ci_low", "ci_high"], axis=1))
          .pipe(np.exp)                                       # exp na estymacie i obu końcach przedziału
          .assign(p_value=m_logit.pvalues))
# OR = 1.28 dla "pages": każda kolejna podstrona zwiększa SZANSE konwersji o ~28%; OR = 1: brak efektu; OR < 1: spadek szans

m_logit.get_margeff().summary()                               # efekty krańcowe: zmiana PRAWDOPODOBIEŃSTWA (bardziej intuicyjne niż OR)

# ---- Predykcja i klasyfikacja -----------------------------------------------------------------------------------------------------------------------
scored = sessions.assign(p_hat=m_logit.predict(sessions)).assign(pred=lambda d: (d["p_hat"] > 0.5).astype(int))   # próg 0.5 dobierz do kosztów błędów
pd.crosstab(scored["converted"], scored["pred"], rownames=["actual"], colnames=["predicted"])    # macierz pomyłek
(scored["converted"] == scored["pred"]).mean()                # accuracy (mylące przy niezbalansowanych klasach)
# AUC / precision / recall: from sklearn.metrics import roc_auc_score, classification_report
```

---

## Dodatek A: tabela odpowiedników

| Operacja | SQL | Pandas | dplyr (R) | Power Query |
|---|---|---|---|---|
| Wybór kolumn | `SELECT a, b` | `df[["a", "b"]]` | `select(a, b)` | Choose Columns |
| Filtr wierszy | `WHERE x > 5` | `df.query("x > 5")` | `filter(x > 5)` | Filter Rows |
| Nowa kolumna | `SELECT x*2 AS y` | `df.assign(y=df["x"] * 2)` | `mutate(y = x * 2)` | Add Custom Column |
| Sortowanie | `ORDER BY x DESC` | `sort_values("x", ascending=False)` | `arrange(desc(x))` | Sort Descending |
| GROUP BY | `GROUP BY g` | `groupby("g").agg(...)` | `group_by(g) \|> summarise()` | Group By |
| COUNT DISTINCT | `COUNT(DISTINCT x)` | `nunique()` | `n_distinct(x)` | Count Distinct Rows |
| HAVING | `HAVING COUNT(*) > 1` | `.agg(...).query("n > 1")` | `filter()` po `summarise()` | Filter po Group By |
| LEFT JOIN | `LEFT JOIN b ON ...` | `a.merge(b, on=..., how="left")` | `left_join(b, by = ...)` | Merge Queries (Left Outer) |
| Anti-join | `NOT EXISTS (...)` | `~a["k"].isin(b["k"])` / `indicator=True` | `anti_join()` | Merge (Left Anti) |
| UNION ALL | `UNION ALL` | `pd.concat([a, b])` | `bind_rows()` | Append Queries |
| DISTINCT | `SELECT DISTINCT` | `drop_duplicates()` | `distinct()` | Remove Duplicates |
| CASE WHEN | `CASE WHEN ... END` | `np.select()` / `np.where()` | `case_when()` | Conditional Column |
| COALESCE | `COALESCE(x, 0)` | `fillna(0)` | `coalesce()` / `replace_na()` | Replace Values / Fill Down |
| ROW_NUMBER | `ROW_NUMBER() OVER (...)` | `groupby().cumcount() + 1` | `row_number()` | Add Index Column |
| RANK | `RANK() OVER (...)` | `rank(method="min")` | `min_rank()` | n/d |
| LAG / LEAD | `LAG(x) OVER (...)` | `shift(1)` / `shift(-1)` | `lag()` / `lead()` | n/d |
| Suma narastająca | `SUM(x) OVER (ORDER BY d)` | `cumsum()` | `cumsum()` | n/d |
| SUM OVER PARTITION | `SUM(x) OVER (PARTITION BY g)` | `groupby("g")["x"].transform("sum")` | `group_by + mutate` | n/d |
| Średnia krocząca | `AVG(x) OVER (ROWS ...)` | `rolling(3).mean()` | `slider::slide_dbl()` | n/d |
| TOP N / grupa | `WHERE rn <= N` | `sort_values().groupby().head(N)` | `slice_max(n = N, by = g)` | Keep Top Rows |
| PIVOT | `SUM(CASE WHEN ...)` | `pivot_table()` | `pivot_wider()` | Pivot Column |
| UNPIVOT | `UNPIVOT` | `melt()` | `pivot_longer()` | Unpivot Other Columns |
| DATE_TRUNC | `DATE_TRUNC('month', d)` | `dt.to_period("M").dt.to_timestamp()` | `floor_date(d, "month")` | Start of Month |
| DATEDIFF | `DATEDIFF(day, a, b)` | `(b - a).dt.days` | `as.integer(b - a)` | Subtract Days |
| LIKE / REGEX | `LIKE '%x%'` | `.str.contains("x")` | `str_detect(x, "x")` | Text.Contains |
| TRIM / LOWER | `TRIM(LOWER(x))` | `.str.strip().str.lower()` | `str_trim() \|> str_to_lower()` | Format > Trim / lowercase |
| CTE / kroki | `WITH s AS (...)` | method chaining / `.pipe()` | pipe `\|>` | Applied Steps |

---

## Dodatek B: pułapki (gotchas)

1. **Indeksy od 0** i wycinki `[a:b]` z końcem wyłącznym (w R od 1 i włącznie); wyjątek: `.loc[a:b]` jest włącznie.
2. **`and`/`or`/`not` na Series** rzucają `ValueError`, używaj `&`, `|`, `~` i **nawiasów** wokół każdego warunku.
3. **Chained assignment:** `df[mask]["col"] = x` nie modyfikuje oryginału (SettingWithCopy). Zawsze `df.loc[mask, "col"] = x` lub `.assign()`.
4. **`inplace=True` nie daje wydajności** i jest wycofywane, więc przypisuj wynik: `df = df.drop(columns=...)`.
5. **`NaN != NaN`:** testuj `isna()`, nigdy `== np.nan`. Agregaty domyślnie **pomijają** NaN (R: nie pomija bez `na.rm = TRUE`).
6. **`groupby` domyślnie odrzuca wiersze z NaN w kluczu**, dodaj `dropna=False`, jeśli ich potrzebujesz.
7. **`count()` vs `size()`:** `count` liczy tylko wartości nie-NaN, `size` liczy wszystkie wiersze (SQL: `COUNT(col)` vs `COUNT(*)`).
8. **`std()`:** Pandas `ddof=1`, NumPy `ddof=0`: ten sam wektor, inne wyniki.
9. **`merge` bez `validate=`** może po cichu rozmnożyć wiersze przy zduplikowanym kluczu; `indicator=True` pomaga w audycie.
10. **Kolumna z NaN jest `float`:** liczby całkowite z brakami wymagają typu `Int64`.
11. **Typy dat:** mieszanie dat naive i tz-aware kończy się błędem; w `merge_asof` oba klucze muszą mieć ten sam dtype i być posortowane.
12. **`df["a"]` to Series, `df[["a"]]` to DataFrame**, a to częsta przyczyna błędów kształtu.
13. **Modyfikowalny argument domyślny** (`def f(x, acc=[])`) jest współdzielony między wywołaniami, używaj `None`.
14. **`is` vs `==`:** `is` tylko dla `None`/`True`/`False`.
15. **Wyniki `rolling()` po `groupby()`** mają MultiIndex: `reset_index(level=0, drop=True)` przed przypisaniem do kolumny.
16. **`shift()`/`cumsum()`/`rolling()` zależą od kolejności wierszy**, więc zawsze `sort_values()` wcześniej.
17. **Porównanie liczb zmiennoprzecinkowych:** `0.1 + 0.2 == 0.3` to `False`, użyj `np.isclose()`.
18. **Ostrzeżenie o kopii przy `df2 = df`:** przypisanie nie kopiuje, to ten sam obiekt (przy Copy-on-Write jest bezpieczniej, ale nadal użyj `.copy()` gdy potrzebujesz niezależności).
19. **Kolumny kategoryczne w `groupby`:** bez `observed=True` dostaniesz wiersze także dla nieistniejących kombinacji.
20. **`str.contains` z `regex=True` (domyślnie)** traktuje `.`, `(`, `+` jako RegEx; dla dosłownego tekstu `regex=False`; zawsze `na=False` w filtrach.

---

## Dodatek C: mini-case cyber

Wykrywanie podejrzanej aktywności logowania: daty, agregacje, okna i filtry w jednym łańcuchu.

```python
# ---- 1) Brute force: >= 3 nieudane logowania z jednego IP w kubełku 10 minut ---------------------------------------
bruteforce = (logs.query("event == 'LOGIN_FAIL'")
              .assign(bucket=lambda d: d["ts"].dt.floor("10min"))           # kubełek czasowy (tumbling window)
              .groupby(["src_ip", "bucket"], as_index=False)
              .agg(fails=("event", "size"), users_tried=("user", "nunique"))
              .query("fails >= 3")
              .sort_values("fails", ascending=False))
# SQL: GROUP BY src_ip, time_bucket('10 minutes', ts) HAVING COUNT(*) >= 3

# ---- 2) Okno KROCZĄCE po czasie: liczba porażek w ostatnich 10 minutach dla każdego zdarzenia --------------------------
fails_10m = (logs.query("event == 'LOGIN_FAIL'")
             .assign(one=1)
             .set_index("ts")                          # rolling po czasie wymaga indeksu czasowego, rosnącego
             .groupby("src_ip")["one"]
             .rolling("10min")                         # okno = ostatnie 10 minut zegarowych (włącznie z bieżącym zdarzeniem)
             .sum()
             .rename("fails_10m")
             .reset_index())                           # kolumny: src_ip, ts, fails_10m
alerts = (fails_10m.query("fails_10m >= 5")
          .groupby("src_ip", as_index=False)
          .agg(max_fails=("fails_10m", "max"), first_alert=("ts", "min")))   # kiedy IP pierwszy raz przekroczył próg
# SQL: COUNT(*) OVER (PARTITION BY src_ip ORDER BY ts RANGE BETWEEN INTERVAL '10 minutes' PRECEDING AND CURRENT ROW)

# ---- 3) Udane logowanie po dwóch porażkach tego samego użytkownika z tego samego IP (możliwe przejęcie konta) ---------
auth = (logs.query("event in ['LOGIN_FAIL', 'LOGIN_OK']")
        .sort_values(["src_ip", "user", "ts"]))
g = auth.groupby(["src_ip", "user"])
suspects = (auth.assign(prev1=g["event"].shift(1),                              # LAG(event, 1)
                        prev2=g["event"].shift(2),                              # LAG(event, 2)
                        gap_sec=g["ts"].diff().dt.total_seconds())              # czas od poprzedniego zdarzenia
            .query("event == 'LOGIN_OK' and prev1 == 'LOGIN_FAIL' and prev2 == 'LOGIN_FAIL' and gap_sec < 120"))

# ---- 4) Anomalie wolumenu danych (exfiltration?): z-score bajtów w obrębie IP ---------------------------------------------
by_ip_bytes = logs.groupby("src_ip")["bytes"]
anomalies = (logs.assign(z=(logs["bytes"] - by_ip_bytes.transform("mean")) / by_ip_bytes.transform("std"))
             .query("abs(z) > 3")                                               # reguła "3 sigm"
             .sort_values("z", key=abs, ascending=False))                       # key=abs: sortuj po wartości bezwzględnej

# ---- 5) Profil ryzyka per IP: flagi logiczne + agregaty (bez lambd w .agg) ---------------------------------------------------
ip_risk = (logs
           .assign(is_fail=logs["event"].eq("LOGIN_FAIL"),
                   is_scan=logs["event"].eq("PORT_SCAN"),
                   is_drop=logs["event"].eq("FW_DROP"))
           .groupby("src_ip", as_index=False)
           .agg(events=("event", "size"),
                fail_rate=("is_fail", "mean"),         # średnia z bool = odsetek
                scans=("is_scan", "sum"),
                drops=("is_drop", "sum"))
           .assign(risk_score=lambda d: (100 * (0.6 * d["fail_rate"]
                                                + 0.3 * np.minimum(d["scans"] / 5, 1)
                                                + 0.1 * np.minimum(d["drops"] / 20, 1))).round(1))
           .sort_values("risk_score", ascending=False))

# ---- 6) Wykres: nieudane logowania w kubełkach 10 min wg IP ---------------------------------------------------------------------
fail_ts = (logs.query("event == 'LOGIN_FAIL'")
           .assign(bucket=lambda d: d["ts"].dt.floor("10min"))
           .groupby(["bucket", "src_ip"], as_index=False).size())
sns.barplot(data=fail_ts, x="bucket", y="size", hue="src_ip")                 # kolumna "size" powstaje z .size() z as_index=False
plt.xticks(rotation=45); plt.ylabel("Liczba prób"); plt.title("Nieudane logowania wg IP"); plt.tight_layout(); plt.show()
```

---

### Checklista szybkiego EDA (po wczytaniu każdego zbioru)

1. `df.shape`, `df.info()`, `df.head()`: rozmiar, typy, pierwsze wiersze.
2. `df.isna().mean()`: profil braków; `df.duplicated().sum()`: duplikaty.
3. `df.describe(include="all")`: zakresy, wartości odstające, kardynalność.
4. Klucze: `df["id"].is_unique`; kategorie: `value_counts(dropna=False)`.
5. Daty: `df["date"].min()`, `.max()`, luki w ciągłości (`date_range` + `reindex`).
6. Dopiero potem: transformacje, joiny (z `validate=`), agregacje, wykresy.

*Wskazówka na koniec: tłumacz własne zapytania SQL z CTE i funkcjami okna 1:1 na łańcuchy Pandas (a gdy łatwiej w SQL, użyj DuckDB na DataFrame'ach), a Dodatek A pomoże przejść między SQL, Pandas, dplyr i Power Query.*
