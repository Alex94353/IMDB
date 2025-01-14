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
- **`dim_genre`**: Obsahuje žánre filmov, čo umožňuje analýzu popularity podľa žánrov.
- **`dim_date`**: obsahuje informácie o dátumoch, ktoré sú kľúčové pre časovú analýzu.

Štruktúra hviezdicového modelu je znázornená na diagrame nižšie. Tento model zjednodušuje prepojenie faktovej tabuľky s dimenziami, čím zlepšuje efektivitu analýzy a pochopenie vzťahov medzi údajmi.

<p align="center">
  <img src="https://github.com/Alex94353/IMDB/blob/main/star_schema.jpg?raw=true">
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

Dimenzia dim_names bola vytvorená na poskytovanie kontextu pre faktovú tabuľku a zahŕňa informácie o menách, dátumoch narodenia, výške a kategóriách osôb podľa ich pohlavia. Dáta boli očistené od záznamov s chýbajúcimi hodnotami. Kategorizácia pohlaví bola zabezpečená pomocou podmienok, kde boli kategórie "actor" a "actress" prevedené na hodnoty "actor_male" a "actor_female" a pre ostatné prípady bola definovaná hodnota "director".

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

Podobne, dimenzia dim_movie obsahuje informácie o filmoch, vrátane názvu, dátumu vydania, trvania, krajiny pôvodu, jazykov, produkčnej spoločnosti a celosvetového príjmu. Táto dimenzia je typu SCD Typ 0, pretože informácie o filmoch, ako názov, dátum vydania, trvanie a produkčná spoločnosť, sú považované za nemenné.



Faktová tabuľka fact_ratings obsahuje záznamy o hodnoteniach filmov a prepojenia na všetky dimenzie. Obsahuje kľúčové metriky, ako sú priemerné hodnotenie, celkový počet hlasov a medián hodnotení. Tieto hodnoty sú spojované s dimenziami dim_movie a dim_date na základe príslušných kľúčov dimenzií.
```sql
DROP TABLE IF EXISTS fact_ratings;
CREATE TABLE fact_ratings AS
SELECT
    DISTINCT ROW_NUMBER() OVER (ORDER BY movie_id) AS fact_rating_id,
    dm.dim_movie_id,
    r.avg_rating,
    r.total_votes,
    r.median_rating,
    dd.dim_date_id
FROM 
    ratings r
JOIN dim_movie dm 
    ON r.movie_id = dm.dim_movie_id
JOIN dim_date dd
    ON dm.date_published = dd.date  
WHERE 
    r.avg_rating IS NOT NULL 
    AND r.total_votes IS NOT NULL 
    AND r.median_rating IS NOT NULL;
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

Graf 1: Najviac hodnotené žánre (Top 5 žánrov)
Táto vizualizácia zobrazuje 5 najčastejších žánrov filmov. Umožňuje identifikovať, ktoré žánre sú najpopulárnejšie medzi filmami v databáze. Zistíme napríklad, že žáner Drama má najväčší počet filmov. Tieto informácie môžu byť užitočné pri analýze trendov v preferenciách používateľov.
```sql
SELECT
  g.genre,
  COUNT(DISTINCT b.dim_movie_id) AS film_count
FROM
  bridge AS b
  JOIN sdim_genre AS g ON b.sdim_genre_id = g.sdim_genre_id
GROUP BY
  g.genre
ORDER BY
  film_count DESC
LIMIT
  5;

```
![image](https://github.com/user-attachments/assets/00ac9618-6d0b-4676-83e5-b75c84cffed5)

Graf 2: Najviac hodnotené filmy (Top 10 filmov s najvyšším hodnotením)
Táto vizualizácia ukazuje 10 filmov s najvyšším priemerným hodnotením. Tieto filmy sú obľúbené medzi divákmi a získali najlepšie hodnotenie. Zistíme, ktoré filmy sú najlepšie hodnotené a môžeme ich použiť ako odporúčania pre používateľov.
```sql
SELECT
  dm.title,
  MAX(fr.avg_rating) AS max_avg_rating
FROM
  fact_ratings AS fr
  JOIN dim_movie AS dm ON fr.dim_movie_id = dm.dim_movie_id
GROUP BY
  fr.dim_movie_id,
  dm.title
ORDER BY
  max_avg_rating DESC
LIMIT
  10;

```
![image](https://github.com/user-attachments/assets/16497178-1e7c-41ab-ad61-f55eb783c572)

Graf 3: Počet filmov podľa roku a krajiny
Tento graf zobrazuje počet filmov vydaných v jednotlivých rokoch a krajinách. Umožňuje identifikovať trendy vo filmovej produkcii v rôznych geografických oblastiach a časových obdobiach.
```sql
SELECT
  DATE_PART (YEAR, dim_movie.date_published) AS year,
  dim_movie.country,
  COUNT(DISTINCT dim_movie.dim_movie_id) AS film_count
FROM
  dim_movie
GROUP BY
  year,
  country
ORDER BY
  year,
  film_count DESC;
```
![image](https://github.com/user-attachments/assets/b3398fbd-649b-41fc-8551-9a47113898dc)


Graf 4: Priemerný počet hlasov podľa roku vydania
Tento graf ukazuje priemerný počet hlasov na film v závislosti od roku vydania. Ukazuje, či existuje rastúci trend v počte hlasov, ktorý by mohol súvisieť s nárastom počtu používateľov hodnotiacich filmy.
```sql
SELECT
  DATE_PART(YEAR, dm.date_published) AS year,
  AVG(fr.total_votes) AS avg_votes  
FROM
  dim_movie AS dm
JOIN
  fact_ratings AS fr ON dm.dim_movie_id = fr.dim_movie_id
GROUP BY
  year
ORDER BY
  year;
```
![image](https://github.com/user-attachments/assets/5f3f13a4-95df-44d3-9656-b2164f570a85)


Graf 5: Tri najlepšie žánre podľa celosvetových príjmov
Tento graf zobrazuje tri žánre, ktoré generujú najvyššie celosvetové príjmy. Cieľom je identifikovať, ktoré žánre dosahujú najvyššie príjmy a porovnať ich výkonnosť v rámci celosvetového trhu.
```sql
SELECT
  dg.genre,
  SUM(CAST(REGEXP_REPLACE(dm.worldwide_gross_income, '^[^0-9]*', '') AS DECIMAL(18,2))) AS total_worldwide_income
FROM
  dim_genre AS dg
JOIN
  bridge_genre_movie AS bgm ON dg.dim_genre_id = bgm.dim_genre_id  -- исправлено имя столбца
JOIN
  dim_movie AS dm ON bgm.dim_movie_id = dm.dim_movie_id
GROUP BY
  dg.genre
ORDER BY
  total_worldwide_income DESC
LIMIT 3;
```
![image](https://github.com/user-attachments/assets/85c2fbba-fc9a-4388-963a-2703929f25f5)


---
Dashboard poskytuje komplexný pohľad na dáta, pričom zodpovedá dôležité otázky týkajúce sa filmových preferencií a správania používateľov. Vizualizácie umožňujú jednoduchú interpretáciu dát a môžu byť využité na optimalizáciu odporúčacích systémov, marketingových stratégií a filmových služieb.

author: Aleksei Bykov
