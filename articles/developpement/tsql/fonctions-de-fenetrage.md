---
title: Fonction de fenêtrage
---

# Fonctions de fenêtrage

La fonction ```OVER``` définit les lignes sur lesquelles une fonction de fenêtrage est appliqué. Elle peut-être utilisée avec :
- ```PARTITION BY``` : Groupe les ligne par colonnes
- ```ORDER BY``` : Trie les lignes
- ```ROW``` / ```RANGE``` : Délimite les lignes

```sql
DECLARE @Group TABLE 
(
    Id INT PRIMARY KEY NOT NULL,
    FirstName VARCHAR(100) NOT NULL,
    LastName VARCHAR(100) NOT NULL,
    Class VARCHAR(50) NOT NULL,
    Points INT NOT NULL
)

INSERT INTO @Group (Id, FirstName, LastName, Class, Points) VALUES
(1, 'John', 'Doe', 'A', 95),
(2, 'Jane', 'Smith', 'B', 88),
(3, 'Emily', 'Johnson', 'A', 92),
(4, 'Michael', 'Brown', 'B', 85),
(5, 'Sarah', 'Davis', 'B', 88);
```

## Agrégation

```sql
SELECT FirstName, LastName, Class, Points, 
    AVG(Points) OVER (PARTITION BY Class) AS "AveragePointsByClass"
FROM @Group
```

| FirstName | LastName | Class | Points | AveragePointsByClass |
|-----------|----------|-------|--------|----------------------|
| John      | Doe      | A     | 95     | 93                   |
| Emily     | Johnson  | A     | 92     | 93                   |
| Michael   | Brown    | B     | 85     | 87                   |
| Sarah     | Davis    | B     | 88     | 87                   |
| Jane      | Smith    | B     | 88     | 87                   |


## Classement

```sql
SELECT FirstName, LastName, Class, Points, 
    ROW_NUMBER() OVER (ORDER BY Id) AS "RowNumberById",
    RANK() OVER (PARTITION BY Class ORDER BY Points DESC) AS "RankByPointsByClass",
    DENSE_RANK() OVER (ORDER BY Points DESC) AS "DenseRankByPoints",
    NTILE(3) OVER (ORDER BY FirstName, LastName DESC) AS "TercilesByFullname"
FROM @Group
```

| FirstName | LastName | Class | Points | RowNumberById | RankByPointsByClass | DenseRankByPoints | TercilesByFullname |
|-----------|----------|-------|--------|---------------|---------------------|-------------------|--------------------|
| Emily     | Johnson  | A     | 92     | 3             | 2                   | 2                 | 1                  |
| Jane      | Smith    | B     | 88     | 2             | 1                   | 3                 | 1                  |
| John      | Doe      | A     | 95     | 1             | 1                   | 1                 | 2                  |
| Michael   | Brown    | B     | 85     | 4             | 3                   | 4                 | 2                  |
| Sarah     | Davis    | B     | 88     | 5             | 1                   | 3                 | 3                  |