# 💍 Marriage Eligibility System
> A family relationship database modeled in both **Neo4j (Graph)** and **MySQL (Relational)** to determine eligible marriage candidates based on Islamic/customary marriage rules.

---

## 📋 Table of Contents
- [Overview](#overview)
- [Marriage Rules](#marriage-rules)
- [Data Schema](#data-schema)
  - [Relational (SQL)](#relational-sql)
  - [Graph (Neo4j)](#graph-neo4j)
- [Family Structure](#family-structure)
- [SQL](#sql)
  - [Insert Data](#insert-data)
  - [Query: Eligible Cousins](#query-eligible-cousins)
  - [Query: Eligible Step-Siblings](#query-eligible-step-siblings)
  - [Query: matchScore for Cousins](#query-matchscore-for-cousins)
  - [Query: matchScore for Step-Siblings](#query-matchscore-for-step-siblings)
- [Cypher (Neo4j)](#cypher-neo4j)
  - [Create Nodes](#create-nodes)
  - [Create Relationships](#create-relationships)
- [Match Score Logic](#match-score-logic)

---

## Overview

This project simulates a **family tree database** and determines who is eligible to marry whom based on defined rules. It is implemented in two paradigms:

| Paradigm | Technology | Purpose |
|---|---|---|
| Relational | MySQL | Structured queries with CTEs |
| Graph | Neo4j | Visual relationship traversal |

---

## Marriage Rules

An individual may marry someone who falls into one of the following categories:

| # | Rule | Description |
|---|---|---|
| 1 | **Father's Brother's Child** | Cross cousin from paternal uncle |
| 2 | **Father's Sister's Child** | Cross cousin from paternal aunt |
| 3 | **Mother's Sister's Child** | Parallel cousin from maternal aunt only |
| 4 | **Step-Sibling** | Child of step-parent with no biological relation |

> ⚠️ **Mother's Brother's Child is NOT eligible** — children of maternal uncles are excluded.

Additional constraint: candidates must be of **opposite gender**.

---

## Data Schema

### Relational (SQL)

**Table: `person`**

| Column | Type | Description |
|---|---|---|
| `id` | char(16) | Unique identifier **(PK)** |
| `name` | varchar(100) | Full name |
| `birthDate` | date | Date of birth |
| `gender` | enum('m','f') | Gender |
| `fatherId` | char(16) | FK → biological father |
| `motherId` | char(16) | FK → biological mother |
| `appearanceScore` | int(11) | Appearance score (1–10) |
| `wealthScore` | int(11) | Wealth score (1–100) |

**Table: `married`**

| Column | Type | Description |
|---|---|---|
| `id` | char(36) | Unique identifier **(PK)** |
| `husbandId` | char(16) | FK → husband |
| `wifeId` | char(16) | FK → wife |
| `status` | enum('married','divorced') | Marriage status |

### Graph (Neo4j)

**Node:** `Person`
```
{name, gender, appearanceScore, wealthScore}
```

**Relationships:**

| Relationship | Description |
|---|---|
| `MARRIED_TO` | Active marriage |
| `DIVORCED_TO` | Past marriage (ended) |
| `FATHER_OF` | Biological father → child |
| `MOTHER_OF` | Biological mother → child |
| `HIGH_POSSIBLE_TO_MARRY` | matchScore > 0.70 |
| `MODERATE_POSSIBLE_TO_MARRY` | matchScore 0.41–0.70 |
| `LOW_POSSIBLE_TO_MARRY` | matchScore ≤ 0.40 |

---

## Family Structure

```
Paternal Side                    Maternal Side
Arthur + Beatrice                William + Yvonne
├── Charles (M)                  ├── Sophia (F) → married Oscar
├── Diana (F) → married Benjamin ├── Victoria (F) → divorced Charles
└── Edward (M) → married Lily    └── Oliver (M) → married Rachel

Charles + Victoria (divorced) → Henry
Benjamin + Diana               → Katherine
Edward + Lily                  → Michael
Oscar + Sophia                 → Quinn
Oliver + Rachel                → Sebastian

Robert + Naomi (divorced)      → Sarah
Charles + Naomi (married)      [2nd marriage, Sarah = step-sibling of Henry]
```

---

## SQL

### Insert Data

<details>
<summary>Click to expand</summary>

```sql
INSERT INTO person (id, name, birthDate, gender, fatherId, motherId, appearanceScore, wealthScore) VALUES
-- Paternal Grandparents
('a1b2c3d4', 'Arthur',    '1950-03-10', 'm', NULL, NULL, 7, 80),
('e5f6g7h8', 'Beatrice',  '1952-06-22', 'f', NULL, NULL, 8, 75),
-- Maternal Grandparents
('u1v2w3x4', 'William',   '1948-07-14', 'm', NULL, NULL, 7, 85),
('y5z6a7b8', 'Yvonne',    '1951-11-30', 'f', NULL, NULL, 8, 80),
-- Arthur & Beatrice's Children
('i9j0k1l2', 'Charles',   '1975-04-15', 'm', 'a1b2c3d4', 'e5f6g7h8', 7, 70),
('m3n4o5p6', 'Diana',     '1977-09-01', 'f', 'a1b2c3d4', 'e5f6g7h8', 8, 65),
('q7r8s9t0', 'Edward',    '1980-12-05', 'm', 'a1b2c3d4', 'e5f6g7h8', 6, 60),
-- William & Yvonne's Children
('c9d0e1f2', 'Sophia',    '1974-02-18', 'f', 'u1v2w3x4', 'y5z6a7b8', 9, 70),
('g3h4i5j6', 'Victoria',  '1976-08-25', 'f', 'u1v2w3x4', 'y5z6a7b8', 8, 68),
('k7l8m9n0', 'Oliver',    '1979-05-11', 'm', 'u1v2w3x4', 'y5z6a7b8', 7, 72),
-- Spouses (Generation 2)
('o1p2q3r4', 'Benjamin',  '1975-03-20', 'm', NULL, NULL, 7, 65),
('s5t6u7v8', 'Lily',      '1981-07-08', 'f', NULL, NULL, 8, 58),
('w9x0y1z2', 'Oscar',     '1973-01-15', 'm', NULL, NULL, 6, 75),
('a3b4c5d6', 'Rachel',    '1980-04-22', 'f', NULL, NULL, 7, 63),
('e7f8g9h0', 'Naomi',     '1977-10-05', 'f', NULL, NULL, 8, 70),
('i1j2k3l4', 'Robert',    '1974-06-18', 'm', NULL, NULL, 6, 55),
-- Generation 3
('m5n6o7p8', 'Henry',     '2000-01-10', 'm', 'i9j0k1l2', 'g3h4i5j6', 9, 90),  -- Charles & Victoria
('q9r0s1t2', 'Katherine', '2001-05-22', 'f', 'o1p2q3r4', 'm3n4o5p6', 8, 38),  -- Benjamin & Diana
('u3v4w5x6', 'Michael',   '2002-09-14', 'm', 'q7r8s9t0', 's5t6u7v8', 3, 15),  -- Edward & Lily
('y7z8a9b0', 'Quinn',     '2000-03-30', 'f', 'w9x0y1z2', 'c9d0e1f2', 9, 85),  -- Oscar & Sophia
('c1d2e3f4', 'Sebastian', '2003-11-07', 'm', 'k7l8m9n0', 'a3b4c5d6', 5, 20),  -- Oliver & Rachel
('g5h6i7j8', 'Sarah',     '1999-08-19', 'f', 'i1j2k3l4', 'e7f8g9h0', 6, 55);  -- Robert & Naomi
```

```sql
INSERT INTO married (id, husbandId, wifeId, status) VALUES
('m1a2b3c4', 'a1b2c3d4', 'e5f6g7h8', 'married'),   -- Arthur & Beatrice
('m2d3e4f5', 'u1v2w3x4', 'y5z6a7b8', 'married'),   -- William & Yvonne
('m3g4h5i6', 'i9j0k1l2', 'g3h4i5j6', 'divorced'),  -- Charles & Victoria
('m4j5k6l7', 'o1p2q3r4', 'm3n4o5p6', 'married'),   -- Benjamin & Diana
('m5m6n7o8', 'q7r8s9t0', 's5t6u7v8', 'married'),   -- Edward & Lily
('m6p7q8r9', 'w9x0y1z2', 'c9d0e1f2', 'married'),   -- Oscar & Sophia
('m7s8t9u0', 'k7l8m9n0', 'a3b4c5d6', 'married'),   -- Oliver & Rachel
('m8v9w0x1', 'i1j2k3l4', 'e7f8g9h0', 'divorced'),  -- Robert & Naomi
('m9y0z1a2', 'i9j0k1l2', 'e7f8g9h0', 'married');   -- Charles & Naomi (2nd marriage)
```

</details>

---

### Query: Eligible Cousins

> Finds all cousins eligible for marriage based on rules 1–3. Does **not** include matchScore — eligibility and scoring are intentionally separated.

```sql
WITH
person_parents AS (
  SELECT
    p.id       AS personId,
    p.name     AS personName,
    p.gender   AS personGender,
    p.fatherId AS fatherId,
    p.motherId AS motherId
  FROM person p
),

-- Find all siblings of the father
father_siblings AS (
  SELECT
    pp.personId,
    sib.id     AS siblingId,
    sib.name   AS siblingName,
    sib.gender AS siblingGender
  FROM person_parents pp
  JOIN person dad ON dad.id = pp.fatherId
  JOIN person sib
    ON sib.fatherId = dad.fatherId
   AND sib.motherId = dad.motherId
   AND sib.id <> pp.fatherId
),

-- Find all siblings of the mother
mother_siblings AS (
  SELECT
    pp.personId,
    sib.id     AS siblingId,
    sib.name   AS siblingName,
    sib.gender AS siblingGender
  FROM person_parents pp
  JOIN person mom ON mom.id = pp.motherId
  JOIN person sib
    ON sib.fatherId = mom.fatherId
   AND sib.motherId = mom.motherId
   AND sib.id <> pp.motherId
),

-- Rule 1: Children of Father's male siblings (paternal uncle)
cousins_father_brother AS (
  SELECT
    pp.personId,
    pp.personName         AS Person,
    pp.personGender       AS PersonGender,
    'Father''s Brother'   AS Relation,
    fs.siblingName        AS UncleAunt,
    c.id                  AS cousinId,
    c.name                AS Cousin,
    c.gender              AS CousinGender
  FROM person_parents pp
  JOIN father_siblings fs ON fs.personId = pp.personId AND fs.siblingGender = 'm'
  JOIN person c ON (c.fatherId = fs.siblingId OR c.motherId = fs.siblingId)
  WHERE c.gender <> pp.personGender
),

-- Rule 2: Children of Father's female siblings (paternal aunt)
cousins_father_sister AS (
  SELECT
    pp.personId,
    pp.personName         AS Person,
    pp.personGender       AS PersonGender,
    'Father''s Sister'    AS Relation,
    fs.siblingName        AS UncleAunt,
    c.id                  AS cousinId,
    c.name                AS Cousin,
    c.gender              AS CousinGender
  FROM person_parents pp
  JOIN father_siblings fs ON fs.personId = pp.personId AND fs.siblingGender = 'f'
  JOIN person c ON (c.fatherId = fs.siblingId OR c.motherId = fs.siblingId)
  WHERE c.gender <> pp.personGender
),

-- Rule 3: Children of Mother's female siblings only (maternal aunt)
-- Mother's BROTHER is excluded
cousins_mother_sister AS (
  SELECT
    pp.personId,
    pp.personName         AS Person,
    pp.personGender       AS PersonGender,
    'Mother''s Sister'    AS Relation,
    ms.siblingName        AS UncleAunt,
    c.id                  AS cousinId,
    c.name                AS Cousin,
    c.gender              AS CousinGender
  FROM person_parents pp
  JOIN mother_siblings ms ON ms.personId = pp.personId AND ms.siblingGender = 'f'
  JOIN person c ON (c.fatherId = ms.siblingId OR c.motherId = ms.siblingId)
  WHERE c.gender <> pp.personGender
),

eligible_cousins AS (
  SELECT * FROM cousins_father_brother
  UNION
  SELECT * FROM cousins_father_sister
  UNION
  SELECT * FROM cousins_mother_sister
)

SELECT
  personId,
  Person,
  PersonGender,
  Relation,
  UncleAunt,
  cousinId,
  Cousin,
  CousinGender
FROM eligible_cousins
ORDER BY Person, Relation;
```

---

### Query: Eligible Step-Siblings

> Finds all step-siblings eligible for marriage (Rule 4). Only traverses **active marriages** (`status = 'married'`).

```sql
WITH
person_parents AS (
  SELECT
    p.id       AS personId,
    p.name     AS personName,
    p.gender   AS personGender,
    p.fatherId AS fatherId,
    p.motherId AS motherId
  FROM person p
),

-- Rule 4a: Father remarried → step-mother's children from another man
stepsiblings_from_father AS (
  SELECT
    pp.personId,
    pp.personName         AS Person,
    pp.personGender       AS PersonGender,
    sm.name               AS StepParent,
    ss.id                 AS stepSibId,
    ss.name               AS StepSibling,
    ss.gender             AS StepSiblingGender
  FROM person_parents pp
  JOIN married m
    ON m.husbandId = pp.fatherId
   AND m.wifeId <> pp.motherId
   AND m.status = 'married'
  JOIN person sm ON sm.id = m.wifeId
  JOIN person ss ON ss.motherId = m.wifeId
  WHERE ss.fatherId <> pp.fatherId
    AND ss.gender <> pp.personGender
),

-- Rule 4b: Mother remarried → step-father's children from another woman
stepsiblings_from_mother AS (
  SELECT
    pp.personId,
    pp.personName         AS Person,
    pp.personGender       AS PersonGender,
    sd.name               AS StepParent,
    ss.id                 AS stepSibId,
    ss.name               AS StepSibling,
    ss.gender             AS StepSiblingGender
  FROM person_parents pp
  JOIN married m
    ON m.wifeId = pp.motherId
   AND m.husbandId <> pp.fatherId
   AND m.status = 'married'
  JOIN person sd ON sd.id = m.husbandId
  JOIN person ss ON ss.fatherId = m.husbandId
  WHERE ss.motherId <> pp.motherId
    AND ss.gender <> pp.personGender
),

eligible_stepsiblings AS (
  SELECT * FROM stepsiblings_from_father
  UNION
  SELECT * FROM stepsiblings_from_mother
)

SELECT
  personId,
  Person,
  PersonGender,
  StepParent,
  stepSibId,
  StepSibling,
  StepSiblingGender
FROM eligible_stepsiblings
ORDER BY Person;
```

---

### Query: matchScore for Cousins

> Joins eligible cousin results with `person` table to calculate compatibility score.

```sql
WITH
person_parents AS (
  SELECT p.id AS personId, p.name AS personName, p.gender AS personGender,
         p.fatherId, p.motherId
  FROM person p
),
father_siblings AS (
  SELECT pp.personId, sib.id AS siblingId, sib.name AS siblingName, sib.gender AS siblingGender
  FROM person_parents pp
  JOIN person dad ON dad.id = pp.fatherId
  JOIN person sib ON sib.fatherId = dad.fatherId AND sib.motherId = dad.motherId AND sib.id <> pp.fatherId
),
mother_siblings AS (
  SELECT pp.personId, sib.id AS siblingId, sib.name AS siblingName, sib.gender AS siblingGender
  FROM person_parents pp
  JOIN person mom ON mom.id = pp.motherId
  JOIN person sib ON sib.fatherId = mom.fatherId AND sib.motherId = mom.motherId AND sib.id <> pp.motherId
),
cousins_father_brother AS (
  SELECT pp.personId, pp.personName AS Person, pp.personGender AS PersonGender,
         'Father''s Brother' AS Relation, fs.siblingName AS UncleAunt,
         c.id AS cousinId, c.name AS Cousin, c.gender AS CousinGender
  FROM person_parents pp
  JOIN father_siblings fs ON fs.personId = pp.personId AND fs.siblingGender = 'm'
  JOIN person c ON (c.fatherId = fs.siblingId OR c.motherId = fs.siblingId)
  WHERE c.gender <> pp.personGender
),
cousins_father_sister AS (
  SELECT pp.personId, pp.personName AS Person, pp.personGender AS PersonGender,
         'Father''s Sister' AS Relation, fs.siblingName AS UncleAunt,
         c.id AS cousinId, c.name AS Cousin, c.gender AS CousinGender
  FROM person_parents pp
  JOIN father_siblings fs ON fs.personId = pp.personId AND fs.siblingGender = 'f'
  JOIN person c ON (c.fatherId = fs.siblingId OR c.motherId = fs.siblingId)
  WHERE c.gender <> pp.personGender
),
cousins_mother_sister AS (
  SELECT pp.personId, pp.personName AS Person, pp.personGender AS PersonGender,
         'Mother''s Sister' AS Relation, ms.siblingName AS UncleAunt,
         c.id AS cousinId, c.name AS Cousin, c.gender AS CousinGender
  FROM person_parents pp
  JOIN mother_siblings ms ON ms.personId = pp.personId AND ms.siblingGender = 'f'
  JOIN person c ON (c.fatherId = ms.siblingId OR c.motherId = ms.siblingId)
  WHERE c.gender <> pp.personGender
),
eligible_cousins AS (
  SELECT * FROM cousins_father_brother
  UNION
  SELECT * FROM cousins_father_sister
  UNION
  SELECT * FROM cousins_mother_sister
)

SELECT
  ec.Person,
  ec.PersonGender,
  ec.Relation,
  ec.UncleAunt,
  ec.Cousin,
  ec.CousinGender,
  ROUND((0.5 * (c.appearanceScore / 10.0)) + (0.5 * (c.wealthScore / 100.0)), 2) AS matchScore,
  CASE
    WHEN (0.5 * (c.appearanceScore / 10.0)) + (0.5 * (c.wealthScore / 100.0)) <= 0.40 THEN 'Low'
    WHEN (0.5 * (c.appearanceScore / 10.0)) + (0.5 * (c.wealthScore / 100.0)) <= 0.70 THEN 'Moderate'
    ELSE 'High'
  END AS matchCategory
FROM eligible_cousins ec
JOIN person c ON c.id = ec.cousinId
ORDER BY Person, matchScore DESC;
```

---

### Query: matchScore for Step-Siblings

> Joins eligible step-sibling results with `person` table to calculate compatibility score.

```sql
WITH
person_parents AS (
  SELECT p.id AS personId, p.name AS personName, p.gender AS personGender,
         p.fatherId, p.motherId
  FROM person p
),
stepsiblings_from_father AS (
  SELECT pp.personId, pp.personName AS Person, pp.personGender AS PersonGender,
         sm.name AS StepParent, ss.id AS stepSibId, ss.name AS StepSibling, ss.gender AS StepSiblingGender
  FROM person_parents pp
  JOIN married m ON m.husbandId = pp.fatherId AND m.wifeId <> pp.motherId AND m.status = 'married'
  JOIN person sm ON sm.id = m.wifeId
  JOIN person ss ON ss.motherId = m.wifeId
  WHERE ss.fatherId <> pp.fatherId AND ss.gender <> pp.personGender
),
stepsiblings_from_mother AS (
  SELECT pp.personId, pp.personName AS Person, pp.personGender AS PersonGender,
         sd.name AS StepParent, ss.id AS stepSibId, ss.name AS StepSibling, ss.gender AS StepSiblingGender
  FROM person_parents pp
  JOIN married m ON m.wifeId = pp.motherId AND m.husbandId <> pp.fatherId AND m.status = 'married'
  JOIN person sd ON sd.id = m.husbandId
  JOIN person ss ON ss.fatherId = m.husbandId
  WHERE ss.motherId <> pp.motherId AND ss.gender <> pp.personGender
),
eligible_stepsiblings AS (
  SELECT * FROM stepsiblings_from_father
  UNION
  SELECT * FROM stepsiblings_from_mother
)

SELECT
  es.Person,
  es.PersonGender,
  es.StepParent,
  es.StepSibling,
  es.StepSiblingGender,
  ROUND((0.5 * (s.appearanceScore / 10.0)) + (0.5 * (s.wealthScore / 100.0)), 2) AS matchScore,
  CASE
    WHEN (0.5 * (s.appearanceScore / 10.0)) + (0.5 * (s.wealthScore / 100.0)) <= 0.40 THEN 'Low'
    WHEN (0.5 * (s.appearanceScore / 10.0)) + (0.5 * (s.wealthScore / 100.0)) <= 0.70 THEN 'Moderate'
    ELSE 'High'
  END AS matchCategory
FROM eligible_stepsiblings es
JOIN person s ON s.id = es.stepSibId
ORDER BY Person, matchScore DESC;
```

---

## Cypher (Neo4j)

### Create Nodes

```cypher
// Reset
MATCH (n) DETACH DELETE n;
```

```cypher
// Paternal Grandparents
CREATE (arthur:Person   {name: 'Arthur',   gender: 'M', appearanceScore: 7, wealthScore: 80})
CREATE (beatrice:Person {name: 'Beatrice', gender: 'F', appearanceScore: 8, wealthScore: 75})
CREATE (arthur)-[:MARRIED_TO]->(beatrice)
CREATE (beatrice)-[:MARRIED_TO]->(arthur)

// Arthur & Beatrice's Children
CREATE (charles:Person {name: 'Charles', gender: 'M', appearanceScore: 7, wealthScore: 70})
CREATE (diana:Person   {name: 'Diana',   gender: 'F', appearanceScore: 8, wealthScore: 65})
CREATE (edward:Person  {name: 'Edward',  gender: 'M', appearanceScore: 6, wealthScore: 60})
CREATE (arthur)-[:FATHER_OF]->(charles)
CREATE (arthur)-[:FATHER_OF]->(diana)
CREATE (arthur)-[:FATHER_OF]->(edward)
CREATE (beatrice)-[:MOTHER_OF]->(charles)
CREATE (beatrice)-[:MOTHER_OF]->(diana)
CREATE (beatrice)-[:MOTHER_OF]->(edward)

// Maternal Grandparents
CREATE (william:Person {name: 'William', gender: 'M', appearanceScore: 7, wealthScore: 85})
CREATE (yvonne:Person  {name: 'Yvonne',  gender: 'F', appearanceScore: 8, wealthScore: 80})
CREATE (william)-[:MARRIED_TO]->(yvonne)
CREATE (yvonne)-[:MARRIED_TO]->(william)

// William & Yvonne's Children
CREATE (sophia:Person   {name: 'Sophia',   gender: 'F', appearanceScore: 9, wealthScore: 70})
CREATE (victoria:Person {name: 'Victoria', gender: 'F', appearanceScore: 8, wealthScore: 68})
CREATE (oliver:Person   {name: 'Oliver',   gender: 'M', appearanceScore: 7, wealthScore: 72})
CREATE (william)-[:FATHER_OF]->(sophia)
CREATE (william)-[:FATHER_OF]->(victoria)
CREATE (william)-[:FATHER_OF]->(oliver)
CREATE (yvonne)-[:MOTHER_OF]->(sophia)
CREATE (yvonne)-[:MOTHER_OF]->(victoria)
CREATE (yvonne)-[:MOTHER_OF]->(oliver)

// Charles & Victoria — divorced, Henry is their child
CREATE (charles)-[:DIVORCED_TO]->(victoria)
CREATE (victoria)-[:DIVORCED_TO]->(charles)
CREATE (henry:Person {name: 'Henry', gender: 'M', appearanceScore: 9, wealthScore: 90})
CREATE (charles)-[:FATHER_OF]->(henry)
CREATE (victoria)-[:MOTHER_OF]->(henry)

// Benjamin + Diana — married, Katherine is their child
CREATE (benjamin:Person  {name: 'Benjamin',  gender: 'M', appearanceScore: 7, wealthScore: 65})
CREATE (katherine:Person {name: 'Katherine', gender: 'F', appearanceScore: 8, wealthScore: 38})
CREATE (diana)-[:MARRIED_TO]->(benjamin)
CREATE (benjamin)-[:MARRIED_TO]->(diana)
CREATE (benjamin)-[:FATHER_OF]->(katherine)
CREATE (diana)-[:MOTHER_OF]->(katherine)

// Edward + Lily — married, Michael is their child
CREATE (lily:Person    {name: 'Lily',    gender: 'F', appearanceScore: 8, wealthScore: 58})
CREATE (michael:Person {name: 'Michael', gender: 'M', appearanceScore: 3, wealthScore: 15})
CREATE (edward)-[:MARRIED_TO]->(lily)
CREATE (lily)-[:MARRIED_TO]->(edward)
CREATE (edward)-[:FATHER_OF]->(michael)
CREATE (lily)-[:MOTHER_OF]->(michael)

// Oscar + Sophia — married, Quinn is their child
CREATE (oscar:Person {name: 'Oscar', gender: 'M', appearanceScore: 6, wealthScore: 75})
CREATE (quinn:Person {name: 'Quinn', gender: 'F', appearanceScore: 9, wealthScore: 85})
CREATE (sophia)-[:MARRIED_TO]->(oscar)
CREATE (oscar)-[:MARRIED_TO]->(sophia)
CREATE (oscar)-[:FATHER_OF]->(quinn)
CREATE (sophia)-[:MOTHER_OF]->(quinn)

// Oliver + Rachel — married, Sebastian is their child
CREATE (rachel:Person    {name: 'Rachel',    gender: 'F', appearanceScore: 7, wealthScore: 63})
CREATE (sebastian:Person {name: 'Sebastian', gender: 'M', appearanceScore: 5, wealthScore: 20})
CREATE (oliver)-[:MARRIED_TO]->(rachel)
CREATE (rachel)-[:MARRIED_TO]->(oliver)
CREATE (oliver)-[:FATHER_OF]->(sebastian)
CREATE (rachel)-[:MOTHER_OF]->(sebastian)

// Robert & Naomi — divorced, Sarah is their child
CREATE (naomi:Person  {name: 'Naomi',  gender: 'F', appearanceScore: 8, wealthScore: 70})
CREATE (robert:Person {name: 'Robert', gender: 'M', appearanceScore: 6, wealthScore: 55})
CREATE (sarah:Person  {name: 'Sarah',  gender: 'F', appearanceScore: 6, wealthScore: 55})
CREATE (robert)-[:DIVORCED_TO]->(naomi)
CREATE (naomi)-[:DIVORCED_TO]->(robert)
CREATE (robert)-[:FATHER_OF]->(sarah)
CREATE (naomi)-[:MOTHER_OF]->(sarah)

// Charles + Naomi — current marriage (2nd marriage for both)
CREATE (charles)-[:MARRIED_TO]->(naomi)
CREATE (naomi)-[:MARRIED_TO]->(charles)
```

---

### Create Relationships

> Run each block separately in Neo4j Browser.

```cypher
// Rule 1: Children of Father's siblings (all genders)
MATCH (anak:Person)<-[:FATHER_OF]-(ayah:Person)
MATCH (ayah)<-[:FATHER_OF|MOTHER_OF]-(gp:Person)-[:FATHER_OF|MOTHER_OF]->(sibling:Person)
WHERE sibling <> ayah
MATCH (sibling)-[:FATHER_OF|MOTHER_OF]->(cousin:Person)
WHERE cousin.gender <> anak.gender
WITH anak, cousin,
  ROUND((0.5 * (cousin.appearanceScore / 10.0)) + (0.5 * (cousin.wealthScore / 100.0)), 2) AS score
FOREACH (_ IN CASE WHEN score <= 0.40 THEN [1] ELSE [] END |
  MERGE (anak)-[:LOW_POSSIBLE_TO_MARRY {matchScore: score}]->(cousin))
FOREACH (_ IN CASE WHEN score > 0.40 AND score <= 0.70 THEN [1] ELSE [] END |
  MERGE (anak)-[:MODERATE_POSSIBLE_TO_MARRY {matchScore: score}]->(cousin))
FOREACH (_ IN CASE WHEN score > 0.70 THEN [1] ELSE [] END |
  MERGE (anak)-[:HIGH_POSSIBLE_TO_MARRY {matchScore: score}]->(cousin));
```

```cypher
// Rule 2: Children of Mother's female siblings only
MATCH (anak:Person)<-[:MOTHER_OF]-(ibu:Person)
MATCH (ibu)<-[:FATHER_OF|MOTHER_OF]-(gp:Person)-[:FATHER_OF|MOTHER_OF]->(sibling:Person)
WHERE sibling <> ibu AND sibling.gender = 'F'
MATCH (sibling)-[:FATHER_OF|MOTHER_OF]->(cousin:Person)
WHERE cousin.gender <> anak.gender
WITH anak, cousin,
  ROUND((0.5 * (cousin.appearanceScore / 10.0)) + (0.5 * (cousin.wealthScore / 100.0)), 2) AS score
FOREACH (_ IN CASE WHEN score <= 0.40 THEN [1] ELSE [] END |
  MERGE (anak)-[:LOW_POSSIBLE_TO_MARRY {matchScore: score}]->(cousin))
FOREACH (_ IN CASE WHEN score > 0.40 AND score <= 0.70 THEN [1] ELSE [] END |
  MERGE (anak)-[:MODERATE_POSSIBLE_TO_MARRY {matchScore: score}]->(cousin))
FOREACH (_ IN CASE WHEN score > 0.70 THEN [1] ELSE [] END |
  MERGE (anak)-[:HIGH_POSSIBLE_TO_MARRY {matchScore: score}]->(cousin));
```

```cypher
// Rule 3: Step-siblings from Father's side (father remarried)
MATCH (anak:Person)<-[:FATHER_OF]-(ayah:Person)-[:MARRIED_TO]->(step_mom:Person)
MATCH (step_mom)-[:MOTHER_OF]->(step_sib:Person)
WHERE NOT (ayah)-[:FATHER_OF]->(step_sib)
  AND step_sib.gender <> anak.gender
WITH anak, step_sib,
  ROUND((0.5 * (step_sib.appearanceScore / 10.0)) + (0.5 * (step_sib.wealthScore / 100.0)), 2) AS score
FOREACH (_ IN CASE WHEN score <= 0.40 THEN [1] ELSE [] END |
  MERGE (anak)-[:LOW_POSSIBLE_TO_MARRY {matchScore: score}]->(step_sib))
FOREACH (_ IN CASE WHEN score > 0.40 AND score <= 0.70 THEN [1] ELSE [] END |
  MERGE (anak)-[:MODERATE_POSSIBLE_TO_MARRY {matchScore: score}]->(step_sib))
FOREACH (_ IN CASE WHEN score > 0.70 THEN [1] ELSE [] END |
  MERGE (anak)-[:HIGH_POSSIBLE_TO_MARRY {matchScore: score}]->(step_sib));
```

```cypher
// Rule 4: Step-siblings from Mother's side (mother remarried)
MATCH (anak:Person)<-[:MOTHER_OF]-(ibu:Person)-[:MARRIED_TO]->(step_dad:Person)
MATCH (step_dad)-[:FATHER_OF]->(step_sib:Person)
WHERE NOT (ibu)-[:MOTHER_OF]->(step_sib)
  AND step_sib.gender <> anak.gender
WITH anak, step_sib,
  ROUND((0.5 * (step_sib.appearanceScore / 10.0)) + (0.5 * (step_sib.wealthScore / 100.0)), 2) AS score
FOREACH (_ IN CASE WHEN score <= 0.40 THEN [1] ELSE [] END |
  MERGE (anak)-[:LOW_POSSIBLE_TO_MARRY {matchScore: score}]->(step_sib))
FOREACH (_ IN CASE WHEN score > 0.40 AND score <= 0.70 THEN [1] ELSE [] END |
  MERGE (anak)-[:MODERATE_POSSIBLE_TO_MARRY {matchScore: score}]->(step_sib))
FOREACH (_ IN CASE WHEN score > 0.70 THEN [1] ELSE [] END |
  MERGE (anak)-[:HIGH_POSSIBLE_TO_MARRY {matchScore: score}]->(step_sib));
```

---

## Match Score Logic

> Eligibility check and match scoring are **intentionally separated** — first determine who is eligible, then calculate how compatible they are.

### Formula

```
matchScore = w1 * (appearanceScore / 10) + w2 * (wealthScore / 100)
```

| Variable | Value | Description |
|---|---|---|
| `w1` | 0.5 | Weight for appearance |
| `w2` | 0.5 | Weight for wealth |
| `appearanceScore` | 1–10 | Normalized by dividing by 10 |
| `wealthScore` | 1–100 | Normalized by dividing by 100 |

### Categories

| Category | Range | Meaning |
|---|---|---|
| 🔴 Low | 0.00 – 0.40 | Poor compatibility |
| 🟡 Moderate | 0.41 – 0.70 | Average compatibility |
| 🟢 High | 0.71 – 1.00 | High compatibility |
