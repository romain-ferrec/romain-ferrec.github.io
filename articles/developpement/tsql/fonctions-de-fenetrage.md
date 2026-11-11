---
title: Fonction de fenêtrage
---

# Fonctions de fenêtrage

La clause ```OVER``` définit la fenêtre sur laquelle une fonction est appliquée. Elle peut être utilisée avec :
- ```ORDER BY``` : Trie les lignes
- ```PARTITION BY``` : Groupe les ligne par une ou plusieurs colonnes
- ```ROW``` / ```RANGE``` : Délimite la plage de lignes dans chaque partition

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
    -- Moyenne des points par classes
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
    -- Numérote les lignes séquentiellement à partir de 1
    ROW_NUMBER() OVER (ORDER BY Id) AS "RowNumber",
    -- Attribue le même rang aux ex æquo (avec saut des rangs suivants)
    RANK() OVER (PARTITION BY Class ORDER BY Points DESC) AS "Rank",
    -- Attribue le même rang aux ex æquo sans sauter les lignes
    DENSE_RANK() OVER (ORDER BY Points DESC) AS "DenseRank",
    -- Numérote les groupes de lignes selon leur répartition (ici par 3)
    NTILE(3) OVER (ORDER BY FirstName, LastName DESC) AS "Ntile"
FROM @Group ORDER BY Class
```

| FirstName | LastName | Class | Points | RowNumber | Rank | DenseRank | Ntile |
|-----------|----------|-------|--------|-----------|------|-----------|-------|
| Emily     | Johnson  | A     | 92     | 3         | 2    | 2         | 1     |
| John      | Doe      | A     | 95     | 1         | 1    | 1         | 2     |
| Michael   | Brown    | B     | 85     | 4         | 3    | 4         | 2     |
| Sarah     | Davis    | B     | 88     | 5         | 1    | 3         | 3     |
| Jane      | Smith    | B     | 88     | 2         | 1    | 3         | 1     |

## Analytique

Retournent une valeurs agrégé à partir d'un ensemble de ligne, avec dans certains cas, la possibilité de retourner 

```sql
SELECT FirstName, LastName, Class, Points,
    -- Nombre de points de la personne suivante dans le classement 
    LEAD(Points, 1) OVER (ORDER BY Points DESC) AS "Lead",
    -- Nombre de points de la personne précédente dans le classement
    LAG(Points, 1) OVER (ORDER BY Points DESC) AS "Lag",
    -- Nombre de points de la première ligne dans le classement
    FIRST_VALUE(Points) OVER (PARTITION BY Class ORDER BY Points DESC) AS "FirstVal",
    -- Nombre de points de la dernière ligne dans le classement
    LAST_VALUE(Points) OVER (PARTITION BY Class ORDER BY Points DESC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS "LastVal",
    -- Pourcentage de personnes dans la classe ayant un nombre de points égal ou inferieur à x
    ROUND(CUME_DIST() OVER (PARTITION BY Class ORDER BY Points DESC), 2) AS "CumeDist",
    -- Pourcentage du rang relatif d'une personne dans sa classe
    PERCENT_RANK() OVER (PARTITION BY Class ORDER BY Points DESC) AS "PercentRank"
FROM @Group
```

| FirstName | LastName | Class | Points | Lead | Lag  | FirstVal | LastVal | CumeDist | PercentRank |
|-----------|----------|-------|--------|------|------|----------|---------|----------|-------------|
| John      | Doe      | A     | 95     | 92   | NULL | 95       | 92      | 0.5      | 0           |
| Emily     | Johnson  | A     | 92     | 88   | 95   | 95       | 92      | 1        | 1           |
| Jane      | Smith    | B     | 88     | 88   | 92   | 88       | 85      | 0.67     | 0           |
| Sarah     | Davis    | B     | 88     | 85   | 88   | 88       | 85      | 0.67     | 0           |
| Michael   | Brown    | B     | 85     | NULL | 88   | 88       | 85      | 1        | 1           |


> [!NOTE]
> - ```PERCENT_RANK(x)``` : $\frac{RANK(x)-1}{COUNT() - 1}$
> - ```CUME_DIST(x)```    | $\frac{\text{Number of rows with value} \le{x}}{COUNT()}$

> [!WARNING]
> Par défaut, les fonctions telles que ```LAST_VALUE``` disposent d'une clause
> ```rows_range_clause ``` dont les lignes sont limitées à la plage ```ROWS UNBOUNDED PRECEDING AND CURRENT ROW```.
>
> Sans la clause personnalisée dans l'exemple précedent, nous aurions obtenu 
> respectivement les valeurs : 95, 92, 88, 88, 85.