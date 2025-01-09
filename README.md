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
