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
  <https://github.com/Alex94353/IMDB/blob/main/IMDB_ERD.png alt="ERD Schema">
  <br>
  <em>Obrázok 1 Entitno-relačná schéma IMDb</em>
</p>

---
