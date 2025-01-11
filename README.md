# **ETL proces datasetu IMDb**

Tento repozitár obsahuje implementáciu ETL procesu v Snowflake pre analýzu dát z IMDb datasetu. Projekt sa zameriava na preskúmanie informácií o filmoch, ich hodnoteniach a demografických údajov o súvisiacich osobách. Výsledný dátový model umožňuje multidimenzionálnu analýzu a vizualizáciu kľúčových metrik.

---
## **1. Úvod a popis zdrojových dát**
Cieľom semestrálneho projektu je analyzovať dáta týkajúce sa filmov, ich režisérov, žánrov a hodnotení. Táto analýza umožňuje identifikovať trendy vo filmovej produkcii, obľúbené filmy a správanie divákov.

Zdrojové dáta sú štruktúrované v relačnej databáze a obsahujú nasledujúce hlavné tabuľky:

- `movie`
- `genre`
- `director_mapping`
- `role_mapping`
- `names`
- `ratings`

Účelom ETL procesu bolo tieto dáta pripraviť, transformovať a sprístupniť pre viacdimenzionálnu analýzu.
---
### **1.1 Dátová architektúra**

### **ERD diagram**
Surové dáta sú usporiadané v relačnom modeli, ktorý je znázornený na **entitno-relačnom diagrame (ERD)**:

<p align="center">
  <img src=https://github.com/Alex94353/IMDB/blob/main/IMDB_ERD.png>
  <br>
  <em>Obrázok 1 Entitno-relačná schéma IMDb</em>
</p>

---

## **2 Dimenzionálny model**

Navrhnutý bol **hviezdicový model (star schema)**,ktorý je efektívny pre analytické spracovanie údajov. Centrálne miesto zaujíma faktová tabuľka **`fact_ratings`**, ktorá je prepojená s nasledujúcimi dimenziami:
- **`dim_movie`**: Obsahuje podrobné informácie o knihách (názov, autor, rok vydania, vydavateľ).
- **`dim_names`**: Obsahuje demografické údaje o používateľoch, ako sú vekové kategórie, pohlavie, povolanie a vzdelanie.
- **`dim_date`**: Zahrňuje informácie o dátumoch hodnotení (deň, mesiac, rok, štvrťrok).
- **`dim_time`**: Obsahuje podrobné časové údaje (hodina, AM/PM).
- **`sdim_genre`**: Špecifikuje žánre filmov, čo umožňuje analýzu popularity podľa žánrov.
- **`bridge`**: Prepojuje filmy a žánre, aby sa umožnilo mapovanie viacerých žánrov na jeden film.

Štruktúra hviezdicového modelu je znázornená na diagrame nižšie. Tento model zjednodušuje prepojenie faktovej tabuľky s dimenziami, čím zlepšuje efektivitu analýzy a pochopenie vzťahov medzi údajmi.

<p align="center">
  <img src="https://github.com/Alex94353/IMDB/blob/main/star_schema.png">
  <br>
  <em>Obrázok 2 Schéma hviezdy pre IMDb</em>
</p>

---

## **3. ETL proces v Snowflake**
ETL proces pozostával z troch hlavných fáz: `extrahovanie` (Extract), `transformácia` (Transform) a `načítanie` (Load). Tento proces bol implementovaný v Snowflake s cieľom pripraviť zdrojové dáta zo staging vrstvy do viacdimenzionálneho modelu vhodného na analýzu a vizualizáciu.

---
### **3.1 Extract (Extrahovanie dát)**
Dáta zo zdrojového datasetu (formát `.csv`) boli najprv nahraté do Snowflake prostredníctvom interného stage úložiska s názvom `imdb_stage`. Stage v Snowflake slúži ako dočasné úložisko na import alebo export dát. Vytvorenie stage bolo zabezpečené príkazom:

#### Príklad kódu:
```sql
CREATE OR REPLACE STAGE imdb_stage;
```
Do stage boli následne nahraté súbory obsahujúce údaje o filmoch, režiséroch, žánroch, hercoch, hodnoteniach a ďalších entitách. Dáta boli importované do staging tabuliek pomocou príkazu `COPY INTO`. Pre každú tabuľku sa použil podobný príkaz:

```sql
COPY INTO movie
FROM @imdb_stage/movie.csv
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1);
```

V prípade nekonzistentných záznamov bol použitý parameter `ON_ERROR = 'CONTINUE'`, ktorý zabezpečil pokračovanie procesu bez prerušenia pri chybách.

---
### **3.2 Transfor (Transformácia dát)**

V tejto fáze boli údaje z medzitabuliek vyčistené, transformované a obohatené. Hlavným cieľom bolo pripraviť dimenzie a faktovú tabuľku, ktoré umožňujú jednoduchú a efektívnu analýzu dát.

Dimenzie boli navrhnuté na poskytovanie kontextu pre faktovú tabuľku. `dim_names` Zahŕňa informácie o osobách spojených s filmami (herci, režiséri a pod.), ako aj kategórie ich účasti. Dáta boli očistené od záznamov s chýbajúcimi hodnotami. Táto dimenzia je typu SCD 2, čo umožňuje sledovať historické zmeny v kategóriách účasti osôb (zmena hereckých rolí) umožňuje uchovávanie histórie.
```sql
CREATE TABLE dim_names AS
SELECT DISTINCT
    n.id AS dim_names_id,
    n.name,
    n.date_of_birth,
    r.category
FROM names n
JOIN role_mapping r ON n.id = r.name_id
WHERE n.id IS NOT NULL
  AND n.name IS NOT NULL
  AND n.date_of_birth IS NOT NULL
  AND r.category IS NOT NULL;
```
Dimenzia `dim_date` je navrhnutá tak, aby uchováva údaje o dátumoch vydania filmov a obsahuje odvodené údaje, ako sú deň, mesiac, štvrťrok a rok. Pre jednoduchšiu analýzu sú pridané textové reprezentácie dní v týždni a mesiacov. Táto dimenzia je klasifikovaná ako SCD Typ 0. To znamená, že existujúce záznamy v tejto dimenzii sú nemenné a uchovávajú statické informácie.

V prípade, že by bolo potrebné sledovať zmeny súvisiace s odvodenými atribútmi (napr. pracovné dni vs. sviatky), bolo by možné prehodnotiť klasifikáciu na SCD Typ 1 (aktualizácia hodnôt) alebo SCD Typ 2 (uchovávanie histórie zmien). V aktuálnom modeli však táto potreba neexistuje, preto je `dim_date` navrhnutá ako SCD Typ 0 s rozširovaním o nové záznamy podľa potreby.

```sql
CREATE TABLE dim_date AS
SELECT DISTINCT
    ROW_NUMBER() OVER (ORDER BY CAST(m.date_published AS DATE)) AS dim_date_id,
    EXTRACT(DAY FROM m.date_published) AS day,
    EXTRACT(DOW FROM m.date_published) + 1 AS dayOfWeek,
    DATE_PART('week', m.date_published) AS week,
    EXTRACT(MONTH FROM m.date_published) AS month,
    EXTRACT(QUARTER FROM m.date_published) AS quarter,
    EXTRACT(YEAR FROM m.date_published) AS year,
    m.date_published AS timestamp,
    CASE EXTRACT(DOW FROM m.date_published) + 1
        WHEN 1 THEN 'Monday'
        WHEN 2 THEN 'Tuesday'
        WHEN 3 THEN 'Wednesday'
        WHEN 4 THEN 'Thursday'
        WHEN 5 THEN 'Friday'
        WHEN 6 THEN 'Saturday'
        WHEN 7 THEN 'Sunday'
    END AS dayOfWeekAsString,
    CASE EXTRACT(MONTH FROM m.date_published)
        WHEN 1 THEN 'January'
        WHEN 2 THEN 'February'
        WHEN 3 THEN 'March'
        WHEN 4 THEN 'April'
        WHEN 5 THEN 'May'
        WHEN 6 THEN 'June'
        WHEN 7 THEN 'July'
        WHEN 8 THEN 'August'
        WHEN 9 THEN 'September'
        WHEN 10 THEN 'October'
        WHEN 11 THEN 'November'
        WHEN 12 THEN 'December'
    END AS monthAsString
FROM movie m
WHERE m.date_published IS NOT NULL;
```
Podobne `dim_movie` obsahuje informácie o filmoch, vrátane názvu, dátumu vydania, trvania, krajiny pôvodu, jazykov a produkčnej spoločnosti. Táto dimenzia je typu SCD Typ 0, pretože informácie o filmoch, ako názov, dátum vydania, trvanie a produkčná spoločnosť, sú považované za nemenné.

Faktová tabuľka `fact_ratings` obsahuje záznamy o hodnoteniach a prepojenia na všetky dimenzie. Obsahuje kľúčové metriky, ako je hodnota hodnotenia a časový údaj.
```sql
CREATE TABLE fact_ratings AS
SELECT
  ROW_NUMBER() OVER (
    ORDER BY
      r.movie_id
  ) AS fact_rating_id,
  r.movie_id AS dim_movie_id,
  n.dim_names_id,
  d.dim_date_id,
  t.dim_time_id
FROM
  ratings AS r
  JOIN dim_movie AS m ON r.movie_id = m.dim_movie_id
  LEFT JOIN dim_names AS n ON r.movie_id = n.dim_names_id
  JOIN dim_date AS d ON CAST(m.date_published AS DATE) = d.timestamp
  LEFT JOIN dim_time AS t ON TO_CHAR(m.date_published, 'HH24:MI:SS') = t.timestamp
WHERE
  NOT r.movie_id IS NULL
  AND NOT n.dim_names_id IS NULL
  AND NOT d.dim_date_id IS NULL
  AND NOT t.dim_time_id IS NULL;

```

---
