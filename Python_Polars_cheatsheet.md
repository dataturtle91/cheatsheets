# Polars – cheatsheet dla analityka BI (SQL / Power Query / R)

> Polars ≥ 1.x. Wszystkie snippety zakładają:
> ```python
> import polars as pl
> import polars.selectors as cs   # selektory kolumn wg typu/nazwy
> ```
> Instalacja: `pip install polars scipy` (SciPy tylko do testów statystycznych, sekcja 7).

## 0. Mapa pojęć: Polars ↔ SQL ↔ Power Query ↔ R

Polars działa jak SQL: **kontekst** (`select`, `filter`, `with_columns`, `group_by().agg()`) przyjmuje **wyrażenia** (`pl.col('x') * 2`) – opis obliczenia, a nie dane. Brak indeksu wierszy (jak w SQL/Power Query, inaczej niż w pandas). Brak wartości = zawsze `null`.

| Polars | SQL | Power Query (M) | R (dplyr) |
| --- | --- | --- | --- |
| `select` | SELECT | Table.SelectColumns | select / transmute |
| `filter` | WHERE / HAVING | Table.SelectRows | filter |
| `with_columns` | SELECT *, expr AS x (lub CTE) | Table.AddColumn | mutate |
| `group_by().agg()` | GROUP BY | Table.Group | group_by + summarise |
| `expr.over()` | OVER (PARTITION BY) | brak wprost (Group + Join) | group_by + mutate |
| `join` | JOIN | Merge Queries | *_join |
| `pl.concat` | UNION ALL | Append Queries | bind_rows |
| `unpivot` / `pivot` | UNPIVOT / PIVOT | Unpivot / Pivot Column | pivot_longer / pivot_wider |
| `sort` | ORDER BY | Table.Sort | arrange |
| `LazyFrame` | plan zapytania + optymalizator | query folding (analogia) | dbplyr (lazy tbl) |

---

## 1. Podstawowe operacje na DataFrame

**Wczytywanie i zapisywanie** (Parquet = kolumnowy, z typami i kompresją; preferuj go zamiast CSV).

```python
df = pl.read_csv('sales.csv', separator=';', try_parse_dates=True,
                 null_values=['', 'NA', 'N/A'], infer_schema_length=10_000)
df = pl.read_csv('sales.csv', schema_overrides={'id': pl.Utf8})  # wymuś typ kolumny
df = pl.read_parquet('sales.parquet')
df = pl.read_excel('sales.xlsx', sheet_name='Dane')              # wymaga: pip install fastexcel

df.write_csv('out.csv')
df.write_parquet('out.parquet', compression='zstd')

# z pamięci (odpowiednik "Enter Data" w PQ / tibble() w R)
df = pl.DataFrame({'id': [1, 2, 3], 'kat': ['A', 'B', 'A'], 'kwota': [10.5, 20.0, None]})
```

**Podgląd i podstawowe informacje**

```python
df.head(5); df.tail(5); df.sample(5)   # sample = losowe wiersze
df.glimpse()                           # jak str() / glimpse() w R
df.shape                               # (wiersze, kolumny)
df.columns; df.dtypes; df.schema       # schema = {kolumna: typ}
df.describe()                          # count, null_count, mean, std, min, kwartyle, max
df.estimated_size('mb')                # rozmiar w pamięci
df.null_count()                        # liczba nulli w każdej kolumnie
df['kwota']                            # Series;  df.get_column('kwota') – to samo
df.n_unique('kat')                     # COUNT(DISTINCT kat)
```

**Pivot / unpivot** (Excel: tabela przestawna / PQ: Unpivot Columns)

```python
long = df.unpivot(index='id', on=['q1', 'q2', 'q3'], variable_name='kwartal', value_name='wartosc')
wide = long.pivot(on='kwartal', index='id', values='wartosc', aggregate_function='sum')
```

---

## 2. Selekcja, filtrowanie, transformacje

### SELECT – wybór kolumn

```python
df.select('id', 'kwota')                              # SELECT id, kwota
df.select(pl.col('id', 'kwota'))                      # to samo, przez wyrażenie
df.select(pl.all().exclude('id'))                     # SELECT * EXCEPT (id)
df.select(cs.numeric())                               # wszystkie kolumny numeryczne
df.select(cs.starts_with('q') | cs.by_name('id'))     # selektory łączysz | & - ~
df.select(netto=pl.col('kwota') / 1.23)               # SELECT kwota/1.23 AS netto
df.rename({'kwota': 'amount'}); df.drop('tmp')
```

### WHERE – filtrowanie wierszy

```python
df.filter(pl.col('kwota') > 100)
df.filter((pl.col('kat') == 'A') & (pl.col('kwota') > 100))     # & | ~  (nawiasy obowiązkowe!)
df.filter(pl.col('kat').is_in(['A', 'B']))                       # IN (...)
df.filter(pl.col('kwota').is_between(10, 50))                    # BETWEEN
df.filter(pl.col('nazwa').str.contains('(?i)abc'))               # LIKE '%abc%' (regex, (?i) = bez wielkości liter)
df.filter(pl.col('kwota').is_not_null())                         # IS NOT NULL
df.filter(pl.col('data').dt.year() == 2025)
df.filter(pl.col('kat') == 'A', pl.col('kwota') > 100)           # kilka warunków po przecinku = AND
```

### WITH / mutate – nowe kolumny

`with_columns` dodaje lub nadpisuje kolumny i zachowuje pozostałe (jak `mutate()` w R i `Table.AddColumn` w PQ). Wyrażenia w jednym `with_columns` liczą się **równolegle** i nie widzą się nawzajem – zależne kolumny dawaj w kolejnych `with_columns`.

```python
df = (df
  .with_columns(netto=pl.col('kwota') / 1.23,
                rok=pl.col('data').dt.year(),
                kat_up=pl.col('kat').str.to_uppercase())
  .with_columns(vat=pl.col('kwota') - pl.col('netto')))            # zależy od 'netto' → osobny krok

# CASE WHEN
df.with_columns(
    segment=pl.when(pl.col('kwota') >= 1000).then(pl.lit('VIP'))
              .when(pl.col('kwota') >= 100).then(pl.lit('Std'))
              .otherwise(pl.lit('Low')))

# ta sama operacja na wielu kolumnach (jak across() w dplyr)
df.with_columns(pl.col('q1', 'q2', 'q3').round(1))
df.with_columns(cs.numeric().fill_null(0).name.suffix('_clean'))

# sortowanie, duplikaty, top-N
df.sort('kwota', descending=True)                  # ORDER BY kwota DESC
df.sort(['kat', 'kwota'], descending=[False, True])
df.unique(subset=['id'], keep='first', maintain_order=True)  # DISTINCT ON
df.top_k(10, by='kwota')                           # TOP 10
df.pipe(my_func, arg=1)                            # łańcuchowanie własnych funkcji

# stringi i daty
pl.col('s').str.strip_chars().str.replace_all(r'\s+', ' ').str.split(',')
pl.col('s').str.extract(r'(\d+)', 1)
pl.col('d').str.to_date('%Y-%m-%d')                 # tekst → data
pl.col('d').dt.truncate('1mo')                      # początek miesiąca (jak EOMONTH / DATE_TRUNC)
pl.col('d').dt.offset_by('1mo')                     # DATEADD
(pl.col('d2') - pl.col('d1')).dt.total_days()       # DATEDIFF
```

---

## 3. Agregacje i grupowanie

### GROUP BY

```python
(df.group_by('kat')
   .agg(pl.len().alias('n'),                          # COUNT(*)
        pl.col('kwota').sum().alias('suma'),
        pl.col('kwota').mean().alias('srednia'),
        pl.col('klient').n_unique().alias('klienci'), # COUNT(DISTINCT)
        pl.col('kwota').filter(pl.col('rok') == 2025).sum().alias('suma_2025'),  # SUM(CASE WHEN ...)
        pl.col('kwota').max().alias('max'))
   .sort('suma', descending=True))

df.group_by('kat', 'rok', maintain_order=True).agg(pl.col('kwota').sum())   # kilka kluczy
df.group_by('kat').agg(pl.all().sum())                                       # wszystko naraz
df.group_by('kat').agg(pl.col('kwota').sort(descending=True).head(3))        # top-3 per grupa (lista)

# HAVING = filter po agg
df.group_by('kat').agg(pl.len().alias('n')).filter(pl.col('n') > 10)

# grupowanie po czasie (jak GROUP BY DATE_TRUNC('month', d))
df.sort('data').group_by_dynamic('data', every='1mo').agg(pl.col('kwota').sum())
```

### Funkcje okienkowe: `.over()` = `OVER (PARTITION BY ...)`

Wynik ma **tyle samo wierszy co wejście** (jak window function w SQL, a nie jak GROUP BY).

```python
df.with_columns(
    suma_kat=pl.col('kwota').sum().over('kat'),                       # SUM() OVER (PARTITION BY kat)
    udzial=pl.col('kwota') / pl.col('kwota').sum().over('kat'),       # % w grupie
    rank=pl.col('kwota').rank('dense', descending=True).over('kat'),  # DENSE_RANK() (też: 'ordinal' = ROW_NUMBER)
    prev=pl.col('kwota').shift(1).over('klient', order_by='data'),    # LAG(kwota, 1)
    next=pl.col('kwota').shift(-1).over('klient', order_by='data'),   # LEAD
    narast=pl.col('kwota').cum_sum().over('klient', order_by='data'), # running total
    roll3=pl.col('kwota').rolling_mean(window_size=3).over('klient', order_by='data'),  # ruchoma średnia
    first_val=pl.col('kwota').first().over('klient', order_by='data'),
)

# zmiana r/r, % zmiany
df.with_columns(zmiana=pl.col('kwota').pct_change().over('kat', order_by='rok'))

# najnowszy rekord per klient (ROW_NUMBER() = 1)
df.filter(pl.col('data') == pl.col('data').max().over('klient'))
df.sort('data').group_by('klient').last()            # wersja skrócona
```

---

## 4. Łączenie danych

### Joins

```python
a.join(b, on='id', how='inner')                       # INNER JOIN
a.join(b, on='id', how='left')                        # LEFT JOIN
a.join(b, on='id', how='right')
a.join(b, on='id', how='full', coalesce=True)         # FULL OUTER JOIN (coalesce=True scala klucze)
a.join(b, on='id', how='semi')                        # WHERE EXISTS / IN – tylko kolumny z 'a'
a.join(b, on='id', how='anti')                        # WHERE NOT EXISTS – wiersze z 'a' bez pary w 'b'
a.join(b, how='cross')                                # CROSS JOIN

a.join(b, left_on='id', right_on='klient_id', how='left')   # różne nazwy kluczy
a.join(b, on=['id', 'rok'], how='left', suffix='_b')         # klucz złożony, sufiks dla duplikatów nazw
a.join(b, on='id', how='left', validate='m:1')               # kontrola kardynalności (1:1, m:1, 1:m, m:m)
a.join_asof(b, on='data', by='klient', strategy='backward')  # dopasowanie do najbliższej wcześniejszej daty
```

| `how` | Wynik | Odpowiednik |
| --- | --- | --- |
| `inner` | wiersze z parą po obu stronach | INNER JOIN |
| `left` | wszystkie z lewej + pasujące z prawej | LEFT JOIN |
| `full` | wszystkie z obu stron | FULL OUTER JOIN |
| `semi` | wiersze z lewej, które mają parę (bez kolumn z prawej) | EXISTS / `semi_join` w dplyr |
| `anti` | wiersze z lewej, które **nie** mają pary | NOT EXISTS / `anti_join` w dplyr |
| `cross` | iloczyn kartezjański | CROSS JOIN |

### Concatenation / UNION

```python
pl.concat([df1, df2])                         # UNION ALL – takie same kolumny i typy (how='vertical')
pl.concat([df1, df2], how='vertical_relaxed') # automatycznie rzutuje typy do wspólnego
pl.concat([df1, df2], how='diagonal')         # różne zestawy kolumn → brakujące = null (jak bind_rows)
pl.concat([df1, df2], how='horizontal')       # sklejenie kolumn obok siebie (bez klucza!)
pl.concat([df1, df2]).unique()                # UNION (bez duplikatów)

# EXCEPT / INTERSECT przez joiny
df1.join(df2, on=df1.columns, how='anti')     # EXCEPT
df1.join(df2, on=df1.columns, how='semi')     # INTERSECT

# wczytanie wielu plików naraz (Append Queries z folderu w PQ)
pl.read_parquet('dane/*.parquet')
pl.scan_csv('dane/2025-*.csv')                # lazy
```

---

## 5. Braki danych (nulls) i typy danych

**Null ≠ NaN.** `null` = brak wartości (dla każdego typu), `NaN` to specjalna wartość liczbowa float. Logika trójwartościowa jak w SQL: `null == null` daje `null`, a nie `True`.

```python
pl.col('x').is_null(); pl.col('x').is_not_null()   # IS NULL / IS NOT NULL
pl.col('x').is_nan()                               # tylko dla float
df.null_count()                                    # nulle per kolumna
df.filter(pl.any_horizontal(pl.all().is_null()))   # wiersze z jakimkolwiek nullem

# uzupełnianie / usuwanie
df.drop_nulls()                                    # usuń wiersze z nullem w dowolnej kolumnie
df.drop_nulls(subset=['kwota'])
df.with_columns(pl.col('kwota').fill_null(0))                       # COALESCE(kwota, 0)
df.with_columns(pl.col('kwota').fill_null(strategy='forward'))      # LOCF (jak fill() w tidyr)
df.with_columns(pl.col('kwota').fill_null(pl.col('kwota').mean()))  # średnią
df.with_columns(pl.col('kwota').fill_null(pl.col('kwota').median().over('kat')))  # medianą grupy
df.with_columns(pl.col('x').fill_nan(None))                         # NaN → null
df.with_columns(pl.coalesce('a', 'b', pl.lit(0)).alias('wartosc'))  # COALESCE(a, b, 0)
df.with_columns(pl.col('x').replace(-999, None))                    # wartość-wskaźnik → null (NULLIF)
```

**Typy danych**

| Polars | Odpowiednik SQL | Uwagi |
| --- | --- | --- |
| `Int8/16/32/64`, `UInt*` | TINYINT…BIGINT | domyślnie `Int64` |
| `Float32/64` | REAL / DOUBLE | ma NaN |
| `Decimal` | DECIMAL | eksperymentalny, do finansów |
| `Utf8` / `String` | VARCHAR | |
| `Categorical`, `Enum` | – | oszczędność pamięci, szybsze group_by / join |
| `Boolean` | BIT | |
| `Date`, `Datetime`, `Duration`, `Time` | DATE, TIMESTAMP, INTERVAL, TIME | `Datetime('us', 'Europe/Warsaw')` |
| `List`, `Array`, `Struct` | ARRAY / ROW | zagnieżdżone, bez odpowiednika w klasycznym SQL |

```python
# rzutowanie (CAST)
df.with_columns(pl.col('id').cast(pl.Utf8))
df.with_columns(pl.col('kwota').cast(pl.Float64, strict=False))     # TRY_CAST: błędne → null (domyślnie strict=True = błąd)
df.with_columns(pl.col('kat').cast(pl.Categorical))
df.with_columns(cs.string().cast(pl.Categorical))                   # hurtowo
df.cast({'id': pl.Int32, 'kwota': pl.Float32})                      # słownik typów dla DataFrame

# tekst → data/czas
pl.col('d').str.to_date('%d.%m.%Y', strict=False)
pl.col('ts').str.to_datetime('%Y-%m-%d %H:%M:%S').dt.replace_time_zone('Europe/Warsaw')

# sprawdzenie typów i ich zmiana przy wczytywaniu
df.schema
pl.read_csv('f.csv', schema_overrides={'id': pl.Utf8, 'data': pl.Date})

# optymalizacja pamięci
df.shrink_to_fit(); df.select(cs.integer().shrink_dtype())
```

---

## 6. Lazy Evaluation (`LazyFrame`)

**Eager** (`DataFrame`) wykonuje każdy krok od razu. **Lazy** (`LazyFrame`) tylko buduje **plan zapytania**; dopiero `.collect()` go optymalizuje i wykonuje (jak SQL: piszesz zapytanie, a silnik wybiera plan; w Power Query podobny efekt daje query folding).

**Dlaczego lazy:**
- **Predicate pushdown** – filtry spychane do skanowania pliku (czyta się mniej wierszy).
- **Projection pushdown** – czytane są tylko użyte kolumny (świetne dla Parquet).
- Eliminacja wspólnych podwyrażeń, równoległe wykonanie, optymalizacja kolejności operacji.
- Praca na danych większych niż RAM (`streaming`).

**Kiedy `.collect()`:** na końcu pipeline'u, gdy potrzebujesz wyniku jako `DataFrame` (do podglądu, wykresu, SciPy, eksportu do Excela). Dla zapisu do pliku bez ładowania do pamięci użyj `sink_*`.

```python
lf = pl.scan_parquet('dane/*.parquet')            # też scan_csv, scan_ndjson; lub df.lazy()

result = (lf
  .filter(pl.col('rok') >= 2024)                  # zostanie "wepchnięty" do skanowania
  .select('kat', 'kwota', 'klient')               # zostaną wczytane tylko te kolumny
  .group_by('kat')
  .agg(pl.col('kwota').sum().alias('suma'),
       pl.col('klient').n_unique().alias('klienci'))
  .sort('suma', descending=True)
  .collect())                                     # ← tu dopiero obliczenia

lf.explain()                                      # tekstowy plan zoptymalizowany (jak EXPLAIN w SQL)
lf.explain(optimized=False)                       # plan przed optymalizacją
lf.collect_schema()                               # schema bez wykonywania zapytania (lf.schema też działa, ale ostrzega)
lf.head(100).collect()                            # szybki podgląd

lf.collect(engine='streaming')                    # przetwarzanie kawałkami (dane > RAM)
lf.sink_parquet('out.parquet')                    # zapis strumieniowy bez zbierania do RAM

# kilka zapytań naraz ze wspólnym skanem
pl.collect_all([lf.group_by('kat').agg(pl.len()), lf.group_by('rok').agg(pl.len())])
```

**Wzorzec pracy:** `scan_*` → transformacje → `collect()` → ewentualnie `.to_pandas()` / `.to_numpy()` / `.write_excel()` do dalszych etapów.

---

## 7. Statystyka opisowa i inferencyjna

### Statystyka opisowa (wbudowana w Polars)

```python
df.describe(percentiles=[0.05, 0.25, 0.5, 0.75, 0.95])

df.select(
    pl.col('kwota').mean().alias('mean'),
    pl.col('kwota').median().alias('median'),
    pl.col('kwota').std().alias('std'),              # domyślnie ddof=1 (próbkowe); ddof=0 = populacyjne
    pl.col('kwota').var().alias('var'),
    pl.col('kwota').min().alias('min'), pl.col('kwota').max().alias('max'),
    pl.col('kwota').quantile(0.9).alias('p90'),
    (pl.col('kwota').quantile(0.75) - pl.col('kwota').quantile(0.25)).alias('IQR'),
    pl.col('kwota').skew().alias('skew'),
    pl.col('kwota').kurtosis().alias('kurt'),
    pl.col('kwota').mode().first().alias('moda'),
    (pl.col('kwota').std() / pl.col('kwota').mean()).alias('CV'),   # współczynnik zmienności
)

# per grupa
df.group_by('kat').agg(pl.col('kwota').mean().alias('mean'), pl.col('kwota').std().alias('std'),
                       pl.col('kwota').quantile(0.5).alias('median'), pl.len().alias('n'))

# rozkład / tabela częstości
df['kat'].value_counts(sort=True, normalize=True)    # liczność i udział (jak COUNT(*) ... GROUP BY)
df.select(pl.col('kwota').hist(bin_count=10))        # histogram (Series.hist)

# korelacje
df.select(cs.numeric()).corr()                       # macierz korelacji (Pearson)
df.select(pl.corr('x', 'y'))                         # pojedyncza korelacja
df.select(pl.corr('x', 'y', method='spearman'))      # Spearman

# wartości odstające (reguła 1.5*IQR) i z-score
q1, q3 = df['kwota'].quantile(0.25), df['kwota'].quantile(0.75)
df.filter(~pl.col('kwota').is_between(q1 - 1.5 * (q3 - q1), q3 + 1.5 * (q3 - q1)))
df.with_columns(z=(pl.col('kwota') - pl.col('kwota').mean()) / pl.col('kwota').std())
```

### Statystyka inferencyjna

Polars **nie ma wbudowanych testów statystycznych**. Wzorzec: przygotuj dane w Polars → `.to_numpy()` → **SciPy / statsmodels**.

```python
from scipy import stats

a = df.filter(pl.col('grupa') == 'A')['kwota'].drop_nulls().to_numpy()
b = df.filter(pl.col('grupa') == 'B')['kwota'].drop_nulls().to_numpy()

stats.shapiro(a)                                  # normalność (H0: rozkład normalny)
stats.levene(a, b)                                # równość wariancji
stats.ttest_ind(a, b, equal_var=False)            # test t Welcha (2 grupy niezależne)
stats.mannwhitneyu(a, b)                          # nieparametryczny odpowiednik
stats.f_oneway(a, b, c)                           # ANOVA (>2 grupy); nieparam.: stats.kruskal
stats.ttest_rel(przed, po)                        # próby zależne

# test niezależności chi² dla dwóch zmiennych kategorycznych
ct = df.pivot(on='kat2', index='kat1', values='id', aggregate_function='len').fill_null(0)
chi2, p, dof, expected = stats.chi2_contingency(ct.drop('kat1').to_numpy())

# korelacja z p-value
stats.pearsonr(df['x'], df['y'])                  # r, p-value (Series konwertuje się do numpy)
stats.spearmanr(df['x'], df['y'])

# przedział ufności dla średniej (t-Studenta)
m, se = a.mean(), stats.sem(a)
stats.t.interval(0.95, df=len(a) - 1, loc=m, scale=se)

# bootstrap CI dla mediany
res = stats.bootstrap((a,), np.median, confidence_level=0.95, n_resamples=5000)
res.confidence_interval

# regresja liniowa (statsmodels, formuła jak w R: lm(y ~ x1 + x2))
import statsmodels.formula.api as smf
smf.ols('kwota ~ cena + C(kat)', data=df.to_pandas()).fit().summary()
```

| Pytanie | Test | Funkcja |
| --- | --- | --- |
| Czy średnie 2 grup różnią się? | t Welcha / Mann-Whitney | `ttest_ind(equal_var=False)` / `mannwhitneyu` |
| Czy średnie >2 grup różnią się? | ANOVA / Kruskal-Wallis | `f_oneway` / `kruskal` |
| Zmiana przed/po (te same obiekty)? | t sparowany / Wilcoxon | `ttest_rel` / `wilcoxon` |
| Zależność dwóch zmiennych kategorycznych? | chi² | `chi2_contingency` |
| Zależność dwóch zmiennych liczbowych? | Pearson / Spearman | `pearsonr` / `spearmanr` |
| Czy rozkład jest normalny? | Shapiro-Wilk | `shapiro` |

> Przy wielu testach naraz pamiętaj o korekcie na wielokrotne porównania (np. Bonferroni, Benjamini-Hochberg: `stats.false_discovery_control`). P-value ≠ wielkość efektu – raportuj też różnicę średnich i przedział ufności.

---

## 8. Funkcje tekstowe, numeryczne i daty (rozszerzenie)

Funkcje tekstowe żyją w przestrzeni nazw `.str`, daty w `.dt`, listy w `.list`, a struktury w `.struct`. Wszystko to są wyrażenia, więc działają w `select`, `with_columns`, `filter`, `agg` i `over`.

### 8.1 Funkcje tekstowe (`.str`)

| Zadanie | Polars | SQL / Excel |
| --- | --- | --- |
| Długość | `str.len_chars()` | LEN |
| Małe / duże / Title | `str.to_lowercase()` / `to_uppercase()` / `to_titlecase()` | LOWER / UPPER / PROPER |
| Przycięcie spacji | `str.strip_chars()` (też `strip_chars_start/_end`) | TRIM |
| Podciąg | `str.slice(0, 3)` (od 0, długość 3) | LEFT / MID |
| Pierwsze / ostatnie N | `str.head(3)` / `str.tail(3)` | LEFT / RIGHT |
| Zamiana | `str.replace('a', 'b')` / `replace_all` | SUBSTITUTE / REPLACE |
| Zawiera / zaczyna / kończy | `str.contains` / `starts_with` / `ends_with` | LIKE / SEARCH |
| Pozycja | `str.find('x')` (0-based, null gdy brak) | FIND / CHARINDEX |
| Dopełnienie | `str.pad_start(8, '0')` / `str.zfill(8)` | RIGHT('00000000'+x, 8) |
| Sklejanie kolumn | `pl.concat_str(['a','b'], separator=' ')` | CONCAT / & |

```python
df.with_columns(
    # czyszczenie i normalizacja
    nazwa_clean=pl.col('nazwa').str.strip_chars().str.replace_all(r'\s+', ' ').str.to_titlecase(),
    kod=pl.col('kod').cast(pl.Utf8).str.zfill(6),                       # 123 → '000123'
    bez_prefiksu=pl.col('sku').str.strip_prefix('SKU-'),                # też strip_suffix
    # regex
    nr=pl.col('opis').str.extract(r'(\d+)', 1).cast(pl.Int64, strict=False),   # pierwsza grupa
    wszystkie=pl.col('opis').str.extract_all(r'\d+'),                   # lista dopasowań
    ile_cyfr=pl.col('opis').str.count_matches(r'\d'),
    # sklejanie i formatowanie
    pelne=pl.concat_str([pl.col('imie'), pl.col('nazwisko')], separator=' ', ignore_nulls=True),
    etykieta=pl.format('{} ({})', pl.col('nazwa'), pl.col('kat')),
)

# warunki tekstowe
df.filter(pl.col('mail').str.ends_with('@firma.pl'))
df.filter(pl.col('nazwa').str.contains('(?i)polars'))                     # (?i) = ignoruj wielkość liter
df.filter(pl.col('nazwa').str.contains('a.c', literal=True))              # tekst dosłowny, bez regexu
df.filter(pl.col('kat').str.contains_any(['A', 'B'], ascii_case_insensitive=True))

# rozbijanie tekstu (Text to Columns / Split Column w PQ)
df.with_columns(czesci=pl.col('adres').str.split(','))                    # kolumna typu List[str]
df.with_columns(miasto=pl.col('adres').str.split(',').list.get(-1).str.strip_chars())
df.with_columns(pl.col('pelne').str.split_exact(' ', 1)                   # Struct z polami field_0, field_1
                  .struct.rename_fields(['imie', 'nazwisko']).alias('osoba')).unnest('osoba')
df.with_columns(g=pl.col('s').str.extract_groups(r'(?<rok>\d{4})-(?<mies>\d{2})')).unnest('g')  # nazwane grupy regexu

# rozwijanie list do wierszy (jak Expand w PQ / unnest w tidyr / CROSS APPLY STRING_SPLIT)
df.with_columns(tag=pl.col('tagi').str.split(';')).explode('tag')
df.group_by('id').agg(pl.col('tag').str.join(', '))                       # STRING_AGG / paste(collapse=)

# inne
pl.col('s').str.reverse(); pl.col('s').str.len_bytes()
pl.col('s').str.to_integer(base=16)                                       # hex → int
pl.col('s').str.json_decode()                                             # JSON w tekście → Struct
pl.col('s').str.replace_many(['ą', 'ę'], ['a', 'e'])                      # wiele zamian naraz
pl.col('s').str.normalize('NFKD')                                         # normalizacja Unicode (np. do usuwania diakrytyków)
pl.col('s').hash()                                                        # szybki hash (do anonimizacji/kluczy, nie do kryptografii)
```

### 8.2 Funkcje numeryczne

| Zadanie | Polars | SQL / Excel |
| --- | --- | --- |
| Zaokrąglenie | `round(2)`, `floor()`, `ceil()` | ROUND / FLOOR / CEILING |
| Wartość bezwzględna, znak | `abs()`, `sign()` | ABS / SIGN |
| Potęga, pierwiastek | `pow(2)` lub `** 2`, `sqrt()` | POWER / SQRT |
| Logarytm, exp | `log()`, `log10()`, `log1p()`, `exp()` | LN / LOG10 / EXP |
| Modulo, dzielenie całk. | `% 3`, `// 3` | MOD / DIV |
| Ograniczenie zakresu | `clip(0, 100)` | MIN(MAX(x, 0), 100) |
| Min/max z kolumn w wierszu | `pl.min_horizontal`, `pl.max_horizontal` | LEAST / GREATEST |

```python
df.with_columns(
    kw=pl.col('kwota').round(2),
    w_gore=pl.col('kwota').ceil(), w_dol=pl.col('kwota').floor(),
    tys=(pl.col('kwota') / 1000).round(1),
    sig=pl.col('kwota').round_sig_figs(3),                       # 3 cyfry znaczące
    ln=pl.col('kwota').log(),                                    # logarytm naturalny; .log(10) = log10
    bez_outlierow=pl.col('kwota').clip(lower_bound=0, upper_bound=10_000),   # winsoryzacja
    zn=pl.col('saldo').sign(),
    reszta=pl.col('n') % 7, calk=pl.col('n') // 7,
    dzielenie_bezp=pl.col('a') / pl.col('b').replace(0, None),   # unikanie dzielenia przez zero → null
)

# funkcje "po wierszu" (zamiast SUM(A1:C1) w Excelu)
df.with_columns(
    suma=pl.sum_horizontal('q1', 'q2', 'q3'),
    srednia=pl.mean_horizontal('q1', 'q2', 'q3'),
    maks=pl.max_horizontal('q1', 'q2', 'q3'),
    ile_nie_null=pl.sum_horizontal(pl.col('q1', 'q2', 'q3').is_not_null()),
    cala_ok=pl.all_horizontal(pl.col('q1', 'q2', 'q3') > 0),        # też any_horizontal
)

# narastające / różnice / ruchome (dla szeregów)
df.sort('data').with_columns(
    narast=pl.col('x').cum_sum(), max_do_tej_pory=pl.col('x').cum_max(),
    roznica=pl.col('x').diff(), zmiana_pct=pl.col('x').pct_change(),
    srednia_7=pl.col('x').rolling_mean(window_size=7),
    suma_30=pl.col('x').rolling_sum(window_size=30, min_samples=1),
    ewma=pl.col('x').ewm_mean(span=10),                              # wykładnicza średnia ruchoma
)

# dyskretyzacja (buckety, jak BIN w Qlik / cut() w R)
df.with_columns(
    przedzial=pl.col('wiek').cut([18, 35, 60], labels=['<18', '18-35', '36-60', '60+']),
    kwartyl=pl.col('kwota').qcut(4, labels=['Q1', 'Q2', 'Q3', 'Q4']),
)

# argmax/argmin, rangi, losowość
df.select(pl.col('kwota').arg_max(), pl.col('kwota').arg_min())      # indeks maksimum/minimum
pl.col('kwota').rank('average')                                       # też 'min', 'max', 'dense', 'ordinal'
pl.int_range(pl.len()).alias('rn')                                    # numer wiersza (ROW_NUMBER bez sortowania)
pl.int_range(pl.len()).over('kat').alias('rn_w_grupie')
df.sample(fraction=0.1, seed=42)                                      # 10% próbki z ziarnem
```

### 8.3 Funkcje daty i czasu (`.dt`)

| Zadanie | Polars | SQL / Excel |
| --- | --- | --- |
| Rok / kwartał / miesiąc / dzień | `dt.year()`, `quarter()`, `month()`, `day()` | YEAR / DATEPART |
| Tydzień ISO, rok ISO | `dt.week()`, `dt.iso_year()` | ISOWEEK / WEEKNUM(,21) |
| Dzień tygodnia | `dt.weekday()` (**1 = pon … 7 = niedz**) | WEEKDAY(,2) |
| Dzień roku | `dt.ordinal_day()` | DATEPART(dayofyear) |
| Początek okresu | `dt.truncate('1mo')`, `'1w'`, `'1q'`, `'1y'` | DATE_TRUNC |
| Koniec / początek miesiąca | `dt.month_end()`, `dt.month_start()` | EOMONTH |
| Dodaj okres | `dt.offset_by('3mo')`, `'-1y'`, `'7d'` | DATEADD / EDATE |
| Różnica | `(d2 - d1).dt.total_days()` | DATEDIFF |
| Godzina, minuta, sekunda | `dt.hour()`, `minute()`, `second()` | DATEPART |
| Format tekstowy | `dt.to_string('%Y-%m')` / `dt.strftime(...)` | FORMAT / TEXT |

```python
from datetime import date, datetime

df = df.with_columns(
    rok=pl.col('data').dt.year(),
    kwartal=pl.col('data').dt.quarter(),
    mies=pl.col('data').dt.month(),
    tydz=pl.col('data').dt.week(),
    dzien_tyg=pl.col('data').dt.weekday(),                          # 1..7
    nazwa_mies=pl.col('data').dt.to_string('%B'),                   # nazwa miesiąca (zależna od locale)
    rok_mies=pl.col('data').dt.to_string('%Y-%m'),                  # klucz do grupowania/wykresów
    pocz_mies=pl.col('data').dt.truncate('1mo'),
    kon_mies=pl.col('data').dt.month_end(),                         # EOMONTH
    pocz_tyg=pl.col('data').dt.truncate('1w'),                      # poniedziałek
    pocz_roku=pl.col('data').dt.truncate('1y'),
    plus_3m=pl.col('data').dt.offset_by('3mo'),                     # EDATE(+3)
    minus_1r=pl.col('data').dt.offset_by('-1y'),                    # ta sama data rok wcześniej
    dni_od=(pl.lit(date.today()) - pl.col('data')).dt.total_days(), # DATEDIFF(day, data, dziś)
    weekend=pl.col('data').dt.weekday() >= 6,
)

# wiek / staż w pełnych latach
df.with_columns(wiek=((pl.lit(date.today()) - pl.col('ur')).dt.total_days() / 365.25).floor().cast(pl.Int32))

# składanie daty z części
df.with_columns(d=pl.date(pl.col('rok'), pl.col('mies'), pl.lit(1)))                 # DATEFROMPARTS
df.with_columns(ts=pl.datetime(pl.col('rok'), pl.col('mies'), pl.col('dzien'), pl.col('godz')))
df.with_columns(pl.col('data').dt.replace(day=1))                                    # pierwszy dzień miesiąca
df.with_columns(pl.col('data').dt.combine(pl.col('czas')))                           # Date + Time → Datetime
df.with_columns(pl.col('ts').dt.date(), czas=pl.col('ts').dt.time())                 # Datetime → Date / Time

# filtrowanie po dacie
df.filter(pl.col('data').is_between(date(2025, 1, 1), date(2025, 12, 31)))
df.filter(pl.col('data') >= pl.lit(date.today()).dt.offset_by('-30d'))               # ostatnie 30 dni
df.filter(pl.col('data').dt.year() == pl.col('data').max().dt.year())                # bieżący rok w danych

# strefy czasowe
df.with_columns(
    pl.col('ts').dt.replace_time_zone('UTC').dt.convert_time_zone('Europe/Warsaw').alias('ts_pl'))
# replace_time_zone = "ten czas JEST w tej strefie", convert_time_zone = "przelicz do strefy"

# dni robocze (z listą świąt)
sv = [date(2025, 1, 1), date(2025, 1, 6)]
df.with_columns(
    robocze=pl.business_day_count('start', 'koniec', holidays=sv),                   # NETWORKDAYS
    termin=pl.col('start').dt.add_business_days(5, holidays=sv))                     # WORKDAY

# tabela kalendarza (dim_date) – podstawa modelu danych w BI
cal = (pl.date_range(date(2024, 1, 1), date(2026, 12, 31), '1d', eager=True)
         .alias('data').to_frame()
         .with_columns(rok=pl.col('data').dt.year(), kwartal=pl.col('data').dt.quarter(),
                       mies=pl.col('data').dt.month(), tydz=pl.col('data').dt.week(),
                       dzien_tyg=pl.col('data').dt.weekday(),
                       rok_mies=pl.col('data').dt.to_string('%Y-%m'),
                       pocz_mies=pl.col('data').dt.truncate('1mo')))

# uzupełnianie luk w szeregu czasowym (dni bez rekordów)
(df.sort('data').upsample('data', every='1d')
   .with_columns(pl.col('kwota').fill_null(0)))

# okna czasowe w grupach
df.sort('data').rolling('data', period='7d').agg(pl.col('kwota').sum())              # ruchome 7 dni (wg daty)
df.sort('data').group_by_dynamic('data', every='1mo', group_by='kat').agg(pl.col('kwota').sum())
```

**Parsowanie dat z tekstu – najczęstsze formaty**

```python
pl.col('d').str.to_date('%Y-%m-%d')                    # 2025-03-14
pl.col('d').str.to_date('%d.%m.%Y')                    # 14.03.2025 (format polski)
pl.col('d').str.to_date('%d/%m/%y', strict=False)      # dwucyfrowy rok; błędne → null
pl.col('d').str.to_datetime('%Y-%m-%d %H:%M:%S')       # z czasem
pl.col('d').str.to_date()                              # bez formatu: Polars próbuje ISO / typowe formaty
# Excelowy numer seryjny daty (liczba dni od 1899-12-30):
pl.date(1899, 12, 30) + pl.duration(days=pl.col('serial_excel'))
```

**Kody formatu (`strftime`):** `%Y` rok 4 cyfry, `%y` rok 2 cyfry, `%m` miesiąc, `%d` dzień, `%H:%M:%S` czas, `%B` / `%b` nazwa miesiąca pełna / skrócona, `%A` / `%a` nazwa dnia, `%j` dzień roku, `%V` tydzień ISO.

---

## Szybkie pułapki (z perspektywy SQL / PQ / R)

- `&`, `|`, `~` zamiast `and`/`or`/`not`, a warunki w nawiasach: `(a > 1) & (b < 2)`.
- `group_by` (nie `groupby`), `pl.len()` (nie `pl.count()`), `unpivot` (dawniej `melt`), `how='full'` (dawniej `'outer'`).
- Wyrażenia w jednym `with_columns` nie widzą nawzajem swoich wyników; użyj kolejnego `with_columns`.
- `.over()` nie zmienia liczby wierszy, a `group_by().agg()` ją redukuje.
- Kolejność wierszy w `group_by` nie jest gwarantowana; dodaj `maintain_order=True` albo `.sort()`.
- Rzutowanie jest domyślnie **strict** (błąd przy złej wartości); `strict=False` działa jak `TRY_CAST`.
- Dużo danych → `scan_*` + `collect()`, nie `read_*`.
