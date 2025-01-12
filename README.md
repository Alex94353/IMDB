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

Dimenzia dim_names bola vytvorená na poskytovanie kontextu pre faktovú tabuľku a zahŕňa informácie o menách, dátumoch narodenia a kategóriách osôb podľa ich pohlavia. Dáta boli očistené od záznamov s chýbajúcimi hodnotami. Kategorizácia pohlaví bola zabezpečená pomocou podmienok, kde boli jednotlivé kategórie actor a actress prevedené na hodnoty male a female. Pre ostatné prípady bola definovaná hodnota another.

Táto dimenzia je typu SCD 1, čo znamená, že aktuálne dáta sa v prípade zmien jednoducho aktualizujú bez zachovania historických záznamov.
```sql
DROP TABLE IF EXISTS dim_names;
CREATE TABLE dim_names AS
SELECT
  n.id AS dim_names_id,
  n.name,
  n.date_of_birth,
  COALESCE(r.category, 'director') AS category,
  SPLIT(n.known_for_movies, ',') AS known_for_movies
FROM names n
LEFT JOIN role_mapping r ON n.id = r.name_id
WHERE n.id IS NOT NULL
  AND n.name IS NOT NULL
  AND n.date_of_birth IS NOT NULL
  AND n.known_for_movies IS NOT NULL;
```

Podobne, dim_movie obsahuje informácie o filmoch, vrátane názvu, dátumu vydania, trvania, krajiny pôvodu, jazykov a produkčnej spoločnosti. Táto dimenzia je typu SCD Typ 0, pretože informácie o filmoch, ako názov, dátum vydania, trvanie a produkčná spoločnosť, sú považované za nemenné.

Faktová tabuľka fact_ratings obsahuje záznamy o hodnoteniach a prepojenia na všetky dimenzie. Obsahuje kľúčové metriky, ako sú priemerné hodnotenie, celkový počet hlasov a medián hodnotení, pričom tieto hodnoty sú spojované s dimenziami podľa príslušných kľúčov.
```sql
DROP TABLE IF EXISTS fact_ratings;
CREATE
OR REPLACE TABLE fact_ratings AS
SELECT
  dm.dim_movie_id,
  r.avg_rating,
  r.total_votes,
  r.median_rating,
  n.dim_names_id
FROM
  ratings AS r
  JOIN dim_movie AS dm ON r.movie_id = dm.dim_movie_id
  JOIN bridge AS b ON dm.dim_movie_id = b.dim_movie_id
  JOIN sdim_genre AS sg ON b.sdim_genre_id = sg.sdim_genre_id
  JOIN dim_names AS n ON ARRAY_TO_STRING (n.known_for_movies, ',') LIKE '%' || dm.dim_movie_id || '%'
WHERE
  NOT r.movie_id IS NULL
  AND NOT r.avg_rating IS NULL
  AND NOT r.total_votes IS NULL
  AND NOT r.median_rating IS NULL
  AND NOT n.dim_names_id IS NULL;
```

---
### **3.3 Load (Načítanie dát)**

Po úspešnom vytvorení dimenzií, faktovej tabuľky a mostovej tabuľky boli dáta nahraté do finálnej štruktúry.
Pre optimalizáciu využitia úložiska boli dočasné staging tabuľky odstránené:

```sql
DROP TABLE IF EXISTS director_mapping;
DROP TABLE IF EXISTS genre;
DROP TABLE IF EXISTS movie;
DROP TABLE IF EXISTS names;
DROP TABLE IF EXISTS ratings;
DROP TABLE IF EXISTS role_mapping;
```

ETL proces umožnil transformáciu pôvodných dát z .csv formátov do viacdimenzionálneho modelu typu hviezda. Proces zahŕňal čistenie, obohacovanie a reorganizáciu údajov z filmovej databázy. Výsledná štruktúra umožňuje efektívnu analýzu filmových hodnotení, rolí režisérov, hereckého obsadenia a ďalších kľúčových metadát. Tento model poskytuje spoľahlivý základ pre tvorbu vizualizácií a reportov, ktoré podporujú lepšie pochopenie trendov a preferencií divákov.

---
## **4 Vizualizácia dát**
Dashboard obsahuje **5 vizualizácií**, ktoré poskytujú základný prehľad o kľúčových metrikách a trendoch týkajúcich sa filmov, používateľov a hodnotení. Tieto vizualizácie odpovedajú na dôležité otázky a umožňujú lepšie pochopiť správanie používateľov a ich preferencie.

<p align="center">
  <img src="https://github.com/Alex94353/IMDB/blob/main/db.png?raw=true">
  <br>
  <em>Obrázok 3 Dashboard AmazonBooks datasetu</em>
</p>
