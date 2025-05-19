# 2025/2/26

## 环境配置

```gitexclude
git clone https://github.com/berkeley-cs186/sp25-proj1.git
```
其余配置见[cs186.gitbook](https://cs186.gitbook.io/project/assignments/proj1/getting-started)

## 一些简单尝试

```commandline
(base) PS F:\cs186\sp25-proj1> python test.py
FAIL q0 see diffs/q0.txt
FAIL q1i see diffs/q1i.txt
FAIL q1ii see diffs/q1ii.txt
FAIL q1iii see diffs/q1iii.txt
FAIL q1iv see diffs/q1iv.txt
FAIL q2i see diffs/q2i.txt
FAIL q2ii see diffs/q2ii.txt
FAIL q2iii see diffs/q2iii.txt
FAIL q3i see diffs/q3i.txt
FAIL q3ii see diffs/q3ii.txt
FAIL q3iii see diffs/q3iii.txt
FAIL q4i see diffs/q4i.txt
FAIL q4ii_bins_0_to_8 see diffs/q4ii_bins_0_to_8.txt
FAIL q4ii_bin_9 see diffs/q4ii_bin_9.txt
FAIL q4iii see diffs/q4iii.txt
FAIL q4iv see diffs/q4iv.txt
FAIL q4v see diffs/q4v.txt

(base) PS F:\cs186\sp25-proj1> sqlite3 -header lahman.db
SQLite version 3.45.3 2024-04-15 13:34:05 (UTF-16 console I/O)
Enter ".help" for usage hints.

sqlite> .schema people
CREATE TABLE IF NOT EXISTS "people" (
        "playerID" VARCHAR(9) NOT NULL,
        "birthYear" INTEGER NULL,
        "birthMonth" INTEGER NULL,
        "birthDay" INTEGER NULL,
        "birthCountry" VARCHAR(255) NULL,
        "birthState" VARCHAR(255) NULL,
        "birthCity" VARCHAR(255) NULL,
        "deathYear" INTEGER NULL,
        "deathMonth" INTEGER NULL,
        "deathDay" INTEGER NULL,
        "deathCountry" VARCHAR(255) NULL,
        "deathState" VARCHAR(255) NULL,
        "deathCity" VARCHAR(255) NULL,
        "nameFirst" VARCHAR(255) NULL,
        "nameLast" VARCHAR(255) NULL,
        "nameGiven" VARCHAR(255) NULL,
        "weight" INTEGER NULL,
        "height" INTEGER NULL,
        "bats" VARCHAR(255) NULL,
        "throws" VARCHAR(255) NULL,
        "debut" VARCHAR(255) NULL,
        "finalGame" VARCHAR(255) NULL,
        "retroID" VARCHAR(255) NULL,
        "bbrefID" VARCHAR(255) NULL,
        "birth_date" DATE NULL,
        "debut_date" DATE NULL,
        "finalgame_date" DATE NULL,
        "death_date" DATE NULL,
        PRIMARY KEY ("playerID")
);

sqlite> SELECT playerid, namefirst, namelast FROM people;
此处省略···

sqlite> SELECT COUNT(*) FROM fielding;                   
COUNT(*)
143046
```

## 一个问题

按照`gitbook`的步骤修改`proj1.sql`后，运行`test.py`可以看到`PASS`。

```sql
CREATE VIEW q0(era)
AS
  SELECT MAX(ERA) As era
  FROM pitching
;
```

按照教程中使用`python test.py -q 0`命令通过

```commandline
(base) PS F:\cs186\sp25-proj1> python test.py -q 0 
PASS q0
SUCCESS: Your queries passed tests on this dataset
```

检查`/your_output/q0.txt`中的输出也同样正确。

## Task1:Basics

- i. In the `people` table, find the `namefirst`, `namelast` and 
`birthyear` for all players with `weight` greater than 300 pounds.

```sql
CREATE VIEW q1i(namefirst, namelast, birthyear)
AS
  SELECT namefirst, namelast, birthyear
  FROM people
  WHERE weight > 300;
;
```

- ii. Find the `namefirst`, `namelast` and `birthyear` of all players 
whose `namefirst` field contains a space. Order the results by `namefirst`, 
breaking ties with `namelast` both in ascending order.

```sql
CREATE VIEW q1ii(namefirst, namelast, birthyear)
AS
  SELECT namefirst, namelast, birthyear
  FROM people
  WHERE namefirst LIKE '% %'
  ORDER BY namefirst ASC, namelast ASC;
;
```

- iii. From the `people` table, group together players with the same 
`birthyear`, and report the `birthyear`, average `height`, and number of 
players for each `birthyear`. Order the results by `birthyear` in 
ascending order.

```sql
CREATE VIEW q1iii(birthyear, avgheight, count)
AS
  SELECT birthyear, AVG(height) AS avgheight, COUNT(*) AS count
  FROM people
  GROUP BY birthyear
  ORDER BY birthyear ASC;
;
```

- iv. Following the results of part iii, now only include groups 
with an average height > `70`. Again order the results by `birthyear` 
in ascending order.

```sql
CREATE VIEW q1iv(birthyear, avgheight, count)
AS
  SELECT birthyear, avgheight, count
  FROM q1iii
  WHERE avgheight > 70
  ORDER BY birthyear ASC;
;
```

## Task 2 准备

在开始任务2的代码前，发现`gitbook`中并没有提供关于名人堂这个表的相关信息，则首先需要
在数据库中查找。

```commandline
sqlite> . tables
allstarfull          homegames            q2iii              
appearances          leagues              q3i
awardsmanagers       managers             q3ii
awardsplayers        managershalf         q3iii
awardssharemanagers  parks                q4i
awardsshareplayers   people               q4ii
batting              pitching             q4iii
battingpost          pitchingpost         q4iv
collegeplaying       q0                   q4v
divisions            q1i                  salaries
fielding             q1ii                 schools
fieldingof           q1iii                seriespost
fieldingofsplit      q1iv                 teams
fieldingpost         q2i                  teamsfranchises
halloffame           q2ii                 teamshalf
```

发现`halloffame`应该是我们这个任务的主表,再查看一下该表的结构。

```commandline
sqlite> .schema halloffame
CREATE TABLE IF NOT EXISTS "halloffame" (
        "ID" INTEGER NOT NULL,
        "playerID" VARCHAR(10) NOT NULL,
        "yearid" SMALLINT NOT NULL,
        "votedBy" VARCHAR(64) NOT NULL,
        "ballots" SMALLINT NULL,
        "needed" SMALLINT NULL,
        "votes" SMALLINT NULL,
        "inducted" VARCHAR(1) NULL,
        "category" VARCHAR(20) NULL,
        "needed_note" VARCHAR(25) NULL,
        PRIMARY KEY ("ID"),
        FOREIGN KEY("playerID") REFERENCES "people" ("playerID") ON UPDATE NO ACTION ON DELETE NO ACTION
);
CREATE UNIQUE INDEX "halloffame_playerID" ON "halloffame" ("playerID", "yearid", "votedBy");
```

现在可以继续了。

## Task 2: Hall of Fame Schools

- i. Find the `namefirst`, `namelast`, `playerid` and `yearid` of all people 
who were successfully inducted into the Hall of Fame in descending 
order of `yearid`. Break ties on `yearid` by `playerid` (ascending).

```commandline
CREATE VIEW q2i(namefirst, namelast, playerid, yearid)
AS
  SELECT p.namefirst, p.namelast, p.playerid, h.yearid
  FROM halloffame h JOIN people p ON h.playerID = p.playerID
  WHERE h.inducted = 'Y'
  ORDER BY h.yearid DESC, p.playerID ASC;
;
```

- ii. Find the people who were successfully inducted into the Hall 
of Fame and played in college at a school located in the state of 
California. For each person, return their `namefirst`, `namelast`, 
`playerid`, `schoolid`, and `yearid` in descending order of `yearid`. 
Break ties on `yearid` by `schoolid`, `playerid` (ascending). For this 
question, `yearid` refers to the year of induction into the Hall of Fame.

首先检查一下需要的表的结构

```commandline
(base) PS F:\cs186\sp25-proj1> sqlite3 -header lahman.db
SQLite version 3.45.3 2024-04-15 13:34:05 (UTF-16 console I/O)
Enter ".help" for usage hints.
sqlite> SELECT * FROM collegeplaying LIMIT 10;
ID|playerID|schoolID|yearID
1|aardsda01|pennst|2001
2|aardsda01|rice|2002
3|aardsda01|rice|2003
4|abadan01|gamiddl|1992
5|abadan01|gamiddl|1993
6|abbeybe01|vermont|1889
7|abbeybe01|vermont|1890
8|abbeybe01|vermont|1891
9|abbeybe01|vermont|1892
10|abbotje01|kentucky|1991
sqlite> SELECT * FROM schools LIMIT 10;
schoolID|name_full|city|state|country
abilchrist|Abilene Christian University|Abilene|TX|USA
adelphi|Adelphi University|Garden City|NY|USA
adrianmi|Adrian College|Adrian|MI|USA
akron|University of Akron|Akron|OH|USA
alabama|University of Alabama|Tuscaloosa|AL|USA
alabamaam|Alabama A&M University|Normal|AL|USA
alabamast|Alabama State University|Montgomery|AL|USA
albanyst|Albany State University|Albany|GA|USA
albertsnid|Albertson College|Caldwell|ID|USA
albevil|Bevill State Community College|Sumiton|AL|USA
```

初次尝试使用以下代码，出现问题

```sql
CREATE VIEW q2ii(namefirst, namelast, playerid, schoolid, yearid)
AS
  SELECT p.namefirst, p.namelast, p.playerID, cp.schoolID, h.yearid
  FROM halloffame h
      JOIN people p ON h.playerID = p.playerID
      JOIN collegeplaying cp ON p.playerID = cp.playerID
      JOIN schools s ON cp.schoolID = s.schoolID
  WHERE h.inducted = 'Y' AND s.state = 'California'
  ORDER BY h.yearid DESC, cp.schoolID, p.playerID;
;
```

发现`q2ii`中没有输出，进行排查，逐步增加条件。

````sql
SELECT   
    p.nameFirst,   
    p.nameLast,   
    p.playerID,   
    cp.schoolID   
FROM   
    halloffame h  
JOIN   
    people p ON h.playerID = p.playerID  
JOIN   
    collegeplaying cp ON p.playerID = cp.playerID  
WHERE   
    h.inducted = 'Y';
````

这时`q2ii`中正常输出结果，再添加学校筛选即`s.state = 'California'`无输出。发现
是因为表中州使用的是简写😢，被自己蠢哭🤦‍♂️

```sql
CREATE VIEW q2ii(namefirst, namelast, playerid, schoolid, yearid)
AS
  SELECT p.namefirst, p.namelast, p.playerID, cp.schoolID, h.yearid
  FROM halloffame h
      JOIN people p ON h.playerID = p.playerID
      JOIN collegeplaying cp ON p.playerID = cp.playerID
      JOIN schools s ON cp.schoolID = s.schoolID
  WHERE h.inducted = 'Y' AND s.state = 'CA'
  ORDER BY h.yearid DESC, cp.schoolID, p.playerID;
;
```

终于通过了（长呼一口气）。。。。。

- iii. Find the `playerid`, `namefirst`, `namelast` and `schoolid` of 
all people who were successfully inducted into the Hall of Fame -- 
whether or not they played in college. Return people in descending 
order of `playerid`. Break ties on `playerid` by `schoolid` (ascending). 
(Note: `schoolid` should be NULL if they did not play in college.)

```sql
CREATE VIEW q2iii(playerid, namefirst, namelast, schoolid)
AS
  SELECT p.playerID, p.namefirst, p.namelast, cp.schoolID
  FROM halloffame h
      JOIN people p ON h.playerID = p.playerID
      LEFT JOIN collegeplaying cp ON p.playerID = cp.playerID
  WHERE h.inducted = 'Y'
  ORDER BY p.playerID DESC, cp.schoolID;
;
```

一开始没有用`LEFT JOIN`没通过。

1. INNER JOIN(JOIN)
- 只有当两个表中都有匹配的记录时，查询才会返回结果。
2. LEFT JOIN
- 即使`collegeplaying`表中没有匹配的记录，仍然会返回`halloffame`中的全部记录，
并将没有匹配的`schoolID`设置为`NULL`。

## Task 3: SaberMetrics

- i. Find the `playerid`, `namefirst`, `namelast`, `yearid` and 
single-year `slg` (Slugging Percentage) of the players with the 10 
best annual Slugging Percentage recorded over all time. A player 
can appear multiple times in the output. For example, if Babe Ruth’s 
`slg` in 2000 and 2001 both landed in the top 10 best annual Slugging 
Percentage of all time, then we should include Babe Ruth twice in 
the output. For statistical significance, only include players with 
more than 50 at-bats in the season. Order the results by `slg` descending, 
and break ties by `yearid`, `playerid` (ascending).

```sql
CREATE VIEW q3i(playerid, namefirst, namelast, yearid, slg)
AS
  SELECT p.playerid, p.nameFirst, p.nameLast, b.yearID,
         CAST((b.H + b.H2B + 2 * b.H3B + 3 * b.HR) AS FLOAT) / nullif(b.AB, 0) AS slg
  FROM batting b JOIN people p ON b.playerID = p.playerID
  WHERE b.AB > 50
  ORDER BY slg DESC, b.yearID, b.playerID
  LIMIT 10;
;
```

- ii. Following the results from Part i, find the `playerid`, 
`namefirst`, `namelast` and `lslg` (Lifetime Slugging Percentage) for 
the players with the top 10 Lifetime Slugging Percentage. Lifetime Slugging
Percentage (LSLG) uses the same formula as Slugging Percentage (SLG), 
but it uses the number of singles, doubles, triples, home runs, and at 
bats each player has over their entire career, rather than just over a 
single season.

```sql
CREATE VIEW q3ii(playerid, namefirst, namelast, lslg)
AS
  SELECT b.playerID, p.nameFirst, p.nameLast,
         CAST((SUM(b.H) + SUM(b.H2B) + 2 * SUM(b.H3B) + 3 * SUM(b.HR)) AS FLOAT) / nullif(SUM(b.AB), 0) AS lslg
  FROM batting b JOIN people p ON b.playerID = p.playerID
  GROUP BY b.playerID, p.nameFirst, p.nameLast
  HAVING SUM(b.AB) > 50
  ORDER BY lslg DESC, b.playerID
  LIMIT 10;
;
```

- iii. Find the `namefirst`, `namelast` and Lifetime Slugging Percentage 
(`lslg`) of batters whose lifetime slugging percentage is higher than 
that of San Francisco favorite Willie Mays.

```sql
CREATE VIEW q3iii(namefirst, namelast, lslg)
AS
WITH willie_mays_lslg AS (
    SELECT CAST((SUM(bm.H) + SUM(bm.H2B) + 2 * SUM(bm.H3B) + 3 * SUM(bm.HR)) AS FLOAT) / nullif(SUM(bm.AB), 0) AS lslg
    FROM batting bm
    WHERE bm.playerID = 'mayswi01'
--     HAVING SUM(bm.AB) > 50
)
  SELECT p.nameFirst, p.nameLast,
         CAST((SUM(b.H) + SUM(b.H2B) + 2 * SUM(b.H3B) + 3 * SUM(b.HR)) AS FLOAT) / nullif(SUM(b.AB), 0) AS lslg
  FROM batting b JOIN people p ON b.playerID = p.playerID
  GROUP BY p.playerID, p.nameFirst, p.nameLast
  HAVING
      SUM(b.AB) > 50
     AND (CAST((SUM(b.H) + SUM(b.H2B) + 2 * SUM(b.H3B) + 3 * SUM(b.HR)) AS FLOAT) / NULLIF(SUM(b.AB), 0) >
          (SELECT lslg FROM willie_mays_lslg))
;
```

这里使用`WITH`语句定义了一个`CTE`（公共表达式），注释掉的`HAVING`有语法错误，需要跟
`GROUP BY`一起使用。

## Task 4: Salaries

- i. Find the `yearid`, `min`, `max` and `average` of all player salaries 
for each year recorded, ordered by `yearid` in ascending order.

```sql
CREATE VIEW q4i(yearid, min, max, avg)
AS
  SELECT yearID, MIN(salary) AS min, MAX(salary) AS max, AVG(salary) AS avg
  FROM salaries
  GROUP BY yearID
  ORDER BY yearID
;
```

- ii. For salaries in 2016, compute a histogram. Divide the salary 
range into 10 equal bins from `min` to `max`, with `binids` 0 through 9, 
and count the salaries in each bin. Return the `binid`, `low` and `high` 
boundaries for each bin, as well as the number of salaries in each bin, 
with results sorted from smallest bin to largest.

```sql
CREATE VIEW q4ii(binid, low, high, count)
AS
    WITH salary_range AS (
    SELECT
        MIN(salary) AS min_salary,
        MAX(salary) AS max_salary
    FROM
        salaries
    WHERE
        yearid = 2016
),
    bins AS (
    SELECT
        0 AS binid,
        min_salary AS low,
        min_salary + (max_salary - min_salary) / 10.0 AS high
    FROM
        salary_range
    UNION ALL
    SELECT
        1,
        min_salary + (max_salary - min_salary) / 10.0,
        min_salary + 2 * (max_salary - min_salary) / 10.0
    FROM
        salary_range
    UNION ALL
    SELECT
        2,
        min_salary + 2 * (max_salary - min_salary) / 10.0,
        min_salary + 3 * (max_salary - min_salary) / 10.0
    FROM
        salary_range
    UNION ALL
    SELECT
        3,
        min_salary + 3 * (max_salary - min_salary) / 10.0,
        min_salary + 4 * (max_salary - min_salary) / 10.0
    FROM
        salary_range
    UNION ALL
    SELECT
        4,
        min_salary + 4 * (max_salary - min_salary) / 10.0,
        min_salary + 5 * (max_salary - min_salary) / 10.0
    FROM
        salary_range
    UNION ALL
    SELECT
        5,
        min_salary + 5 * (max_salary - min_salary) / 10.0,
        min_salary + 6 * (max_salary - min_salary) / 10.0
    FROM
        salary_range
    UNION ALL
    SELECT
        6,
        min_salary + 6 * (max_salary - min_salary) / 10.0,
        min_salary + 7 * (max_salary - min_salary) / 10.0
    FROM
        salary_range
    UNION ALL
    SELECT
        7,
        min_salary + 7 * (max_salary - min_salary) / 10.0,
        min_salary + 8 * (max_salary - min_salary) / 10.0
    FROM
        salary_range
    UNION ALL
    SELECT
        8,
        min_salary + 8 * (max_salary - min_salary) / 10.0,
        min_salary + 9 * (max_salary - min_salary) / 10.0
    FROM
        salary_range
    UNION ALL
    SELECT
        9,
        min_salary + 9 * (max_salary - min_salary) / 10.0,
        max_salary + 1  -- 这里加1以确保最大值包含在最后一个区间中
    FROM
        salary_range
)
SELECT
    b.binid,
    b.low,
    b.high,
    COUNT(s.salary) AS count
FROM
    bins b
LEFT JOIN
    salaries s ON s.salary >= b.low AND s.salary < b.high AND s.yearid = 2016
GROUP BY
    b.binid, b.low, b.high
ORDER BY
    b.binid
;
```

- iii. Now let's compute the Year-over-Year change in min, max and 
average player salary. For each year with recorded salaries after 
the first, return the `yearid`, `mindiff`, `maxdiff`, and `avgdiff` with 
respect to the previous year. Order the output by `yearid` in ascending 
order. (You should omit the very first year of recorded salaries from 
the result.)

```sql
CREATE VIEW q4iii(yearid, mindiff, maxdiff, avgdiff)
AS
    WITH yearly_salaries AS (
        SELECT yearid, MIN(salary) AS min_salary, MAX(salary) AS max_salary, AVG(salary) AS avg_salary
        FROM salaries
        GROUP BY yearid
    ),
    salary_differences AS (
        SELECT current.yearid,
               current.min_salary - previous.min_salary AS mindiff,
               current.max_salary - previous.max_salary AS maxdiff,
               current.avg_salary - previous.avg_salary AS avgdiff
        FROM yearly_salaries current JOIN yearly_salaries previous ON current.yearID = previous.yearID + 1
    )
  SELECT yearid, mindiff, maxdiff, avgdiff
  FROM salary_differences
  ORDER BY yearid;
;
```

- iv. In 2001, the max salary went up by over $6 million. Write a query 
to find the players that had the max salary in 2000 and 2001. Return 
the `playerid`, `namefirst`, `namelast`, `salary` and `yearid` for 
those two years. If multiple players tied for the max salary in a year, 
return all of them.

```sql
CREATE VIEW q4iv(playerid, namefirst, namelast, salary, yearid)
AS
    WITH max_salaries AS ( 
        SELECT yearID, MAX(salary) AS max_salary
        FROM salaries
        WHERE yearID IN (2000,2001)
        GROUP BY yearID
    )
  SELECT s.playerID, p.namefirst, p.namelast, s.salary, s.yearID
  FROM salaries s 
      JOIN max_salaries m ON s.yearID = m.yearID AND s.salary = m.max_salary 
      JOIN people p ON s.playerID = p.playerID
  WHERE s.yearID IN (2000,2001)
  ORDER BY s.yearID, s.playerID
;
```

- v. Each team has at least 1 All Star and may have multiple. For each 
team in the year 2016, give the `teamid` and `diffAvg` (the difference 
between the team's highest paid all-star's salary and the team's lowest 
paid all-star's salary).

```sql
CREATE VIEW q4v(team, diffAvg) AS
  SELECT a.teamid, MAX(s.salary) - MIN(s.salary) AS diffAvg
  FROM allstarfull a JOIN salaries s ON a.playerID = s.playerID AND a.yearID = 2016 AND s.yearID = 2016
  GROUP BY a.teamid
;
```

完结😊

```commandline
(base) PS F:\cs186\sp25-proj1> python test.py
PASS q0
PASS q1i
PASS q1ii
PASS q1iii
PASS q1iv
PASS q2i
PASS q2ii
PASS q2iii
PASS q3i
PASS q3ii
PASS q3iii
PASS q4i
PASS q4ii_bins_0_to_8
PASS q4ii_bin_9
PASS q4iii
PASS q4iv
PASS q4v
SUCCESS: Your queries passed tests on this dataset
```