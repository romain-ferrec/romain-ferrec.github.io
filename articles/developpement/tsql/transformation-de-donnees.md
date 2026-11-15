---
title: Transformation de données
---

# Transformation de données

Transact-SQL permet de regrouper et réorganiser les données.

A partir d'un jeu de données de collectes de déchets, nous allons voir comment présenter les poids par type et par mois, puis calculer différents niveaux de totaux.

```sql
DECLARE @WasteCollection TABLE 
(
    Id INT PRIMARY KEY NULL,
    CollectionDate DATETIME NOT NULL,
    WeightKg DECIMAL(8, 2) NOT NULL,
    WasteType VARCHAR(100) NOT NULL
)

INSERT INTO @WasteCollection (Id, CollectionDate, WeightKg, WasteType) VALUES
(1, '2023-01-03', 305.47, 'Glass'), (2, '2023-01-16', 453.35, 'Plastic'), (3, '2023-01-27', 207.29, 'Paper'),
(4, '2023-02-10', 255.07, 'Glass'), (5, '2023-02-12', 303.74, 'Plastic'), (6, '2023-02-19', 251.68, 'Paper'),
(7, '2023-03-03', 409.51, 'Glass'), (8, '2023-03-22', 358.32, 'Plastic'), (9, '2023-03-23', 500.64, 'Paper'),
(10, '2023-01-05', 150.00, 'Glass'), (11, '2023-02-15', 200.00, 'Plastic'), (12, '2023-03-10', 300.00, 'Paper'),
(13, '2023-01-20', 100.00, 'Glass'), (14, '2023-02-25', 150.00, 'Plastic'), (15, '2023-03-15', 250.00, 'Paper'),
(16, '2023-01-30', 75.00, 'Glass'), (17, '2023-02-28', 125.00, 'Plastic'), (18, '2023-03-20', 175.00, 'Paper')
```

## Pivot

Nous souhaitons obtenir le poids total des déchets collectés par types, pour les mois de janvier, février et mars de l'année 2023.

```sql
SELECT 
    WasteType,                     -- colonne non pivotée
    [January], [February], [March] -- colonnes pivotées
FROM 
( 
    -- Requête qui fournit les données
    SELECT WasteType, WeightKg, DATENAME(mm, CollectionDate) AS [Month] 
    FROM @WasteCollection
    WHERE CollectionDate >= '2023-01-01' AND CollectionDate < '2024-04-01'
) AS MonthlyWaste
PIVOT(
    SUM(WeightKg) -- fonction d'aggrégation des données sur une colonne
    FOR [Month] -- colonne contenant les valeurs pour les noms des colonnes pivotées
    IN ([January], [February], [March])
) AS PivotTable
```

| WasteType | January | February | March   |
|-----------|---------|----------|---------|
| Glass     | 630.47  | 255.07   | 409.51  |
| Paper     | 207.29  | 251.68   | 1225.64 |
| Plastic   | 453.35  | 778.74   | 358.32  |

## Unpivot

Nous souhaitons rétablir les colonnes correspondant aux mois comme étant des lignes.

```sql
DECLARE @PivotedWasteCollection TABLE
(
    WasteType VARCHAR(100) NOT NULL,
    January DECIMAL(8, 2) NOT NULL,
    February DECIMAL(8, 2) NOT NULL,
    March DECIMAL(8, 2) NOT NULL
)

INSERT INTO @PivotedWasteCollection (WasteType, January, February, March)
VALUES
    ('Glass', 630.47, 255.07, 409.51),
    ('Paper', 207.29, 251.68, 1225.64),
    ('Plastic', 453.35, 778.74, 358.32);

SELECT CollectionMonth, WeightKg, WasteType 
FROM @PivotedWasteCollection
UNPIVOT (WeightKg FOR CollectionMonth IN ([January], [February], [March])) AS UnpivotedTable
```

| CollectionMonth | WeightKg | WasteType |
|-----------------|----------|-----------|
| January         | 630.47   | Glass     |
| February        | 255.07   | Glass     |
| March           | 409.51   | Glass     |
| January         | 207.29   | Paper     |
| February        | 251.68   | Paper     |
| March           | 1225.64  | Paper     |
| January         | 453.35   | Plastic   |
| February        | 778.74   | Plastic   |
| March           | 358.32   | Plastic   |

> [!WARNING]
> ```UNPIVOT``` n'est pas capable de restituer les données originelles dés lors qu'une fonction agrégation a été appliquée lors du ```PIVOT```.

## Grouping sets

Nous souhaitons connaitre les poids totaux des déchets par **mois** ou par **type**, ainsi que le total global.

```sql
SELECT WasteType, DATENAME(mm, CollectionDate) AS CollectionMonth, SUM(WeightKg) AS TotalWeight
FROM @WasteCollection
WHERE CollectionDate >= '2023-01-01' AND CollectionDate < '2024-04-01'
GROUP BY GROUPING SETS ((WasteType), (DATENAME(mm, CollectionDate)), ())
```

| WasteType | CollectionMonth | TotalWeight |
|-----------|-----------------|-------------|
| *NULL*    | February        | 1285.49     |
| *NULL*    | January         | 1291.11     |
| *NULL*    | March           | 1993.47     |
| *NULL*    | *NULL*          | 4570.07     |
| Glass     | *NULL*          | 1295.05     |
| Paper     | *NULL*          | 1684.61     |
| Plastic   | *NULL*          | 1590.41     |

## Cube

Nous souhaitons connaitre les poids totaux des déchets :
1. par **mois** et par **type**.
2. par **mois** ou par **type**
3. dans leur ensemble

```sql
SELECT WasteType, DATENAME(mm, CollectionDate) AS CollectionMonth, SUM(WeightKg) AS TotalWeight
FROM @WasteCollection
WHERE CollectionDate >= '2023-01-01' AND CollectionDate < '2024-04-01'
GROUP BY CUBE (WasteType, DATENAME(mm, CollectionDate))
```

| WasteType | CollectionMonth | TotalWeight |
|-----------|-----------------|-------------|
| Glass     | February        | 255.07      |
| Paper     | February        | 251.68      |
| Plastic   | February        | 778.74      |
| *NULL*    | February        | 1285.49     |
| Glass     | January         | 630.47      |
| Paper     | January         | 207.29      |
| Plastic   | January         | 453.35      |
| *NULL*    | January         | 1291.11     |
| Glass     | March           | 409.51      |
| Paper     | March           | 1225.64     |
| Plastic   | March           | 358.32      |
| *NULL*    | March           | 1993.47     |
| *NULL*    | *NULL*          | 4570.07     |
| Glass     | *NULL*          | 1295.05     |
| Paper     | *NULL*          | 1684.61     |
| Plastic   | *NULL*          | 1590.41     |

> [!NOTE]
> La clause ```CUBE``` dans cet exemple est équivalente à : ```GROUPING SETS ((WasteType, DATENAME(mm, CollectionDate)), (WasteType), (DATENAME(mm, CollectionDate)), ())```.

## Rollup

Nous souhaitons connaitre les poids totaux des déchets :
1. par **type** et par **mois**
2. par **type**
3. dans leur ensemble

```sql
SELECT WasteType, DATENAME(mm, CollectionDate) AS CollectionMonth, SUM(WeightKg) AS TotalWeight
FROM @WasteCollection
WHERE YEAR(CollectionDate) = 2023
GROUP BY ROLLUP (WasteType, DATENAME(mm, CollectionDate))
```

| WasteType | CollectionMonth | TotalWeight |
|-----------|-----------------|-------------|
| Glass     | February        | 255.07      |
| Glass     | January         | 630.47      |
| Glass     | March           | 409.51      |
| Glass     | *NULL*          | 1295.05     |
| Paper     | February        | 251.68      |
| Paper     | January         | 207.29      |
| Paper     | March           | 1225.64     |
| Paper     | *NULL*          | 1684.61     |
| Plastic   | February        | 778.74      |
| Plastic   | January         | 453.35      |
| Plastic   | March           | 358.32      |
| Plastic   | *NULL*          | 1590.41     |
| *NULL*    | *NULL*          | 4570.07     |

> [!NOTE]
> La clause ```ROLLUP``` dans cet exemple est équivalente à : ```GROUPING SETS ((WasteType, DATENAME(mm, CollectionDate)), (WasteType), ())```