# KQL Cheatsheet

> Kusto Query Language — fast reference for Microsoft Fabric Real-Time Intelligence, Azure Data Explorer, Log Analytics, Sentinel, Application Insights.

---

## Mental Model

KQL is **pipe-based, not nested**. Each query starts with a table name, then chains operators with `|`. Read top-to-bottom, like Unix shell.

```kql
TableName
| operator1 args
| operator2 args
| operator3 args
```

> 💡 **Key insight**: No nested subqueries — comment out lines to debug incrementally.

---

## SQL → KQL Translation

| SQL | KQL | Notes |
|---|---|---|
| `SELECT *` | (implicit) | No SELECT keyword |
| `SELECT a, b, c` | `project a, b, c` | |
| `WHERE x > 10` | `where x > 10` | Use `==` for equality |
| `ORDER BY x DESC` | `sort by x desc` | |
| `LIMIT 10` | `take 10` | Or `top N` |
| `GROUP BY x` | `summarize ... by x` | |
| `COUNT(*)` | `count()` | |
| `AS alias` | `alias =` | `summarize Cnt = count() by x` |
| `JOIN ON x = y` | `join kind=inner Y on $left.x == $right.y` | Many join kinds |
| `UNION` | `union T1, T2` | |
| `DISTINCT x` | `distinct x` | |
| `NULL` | `isnull(col)` | Function-based |
| `CAST(x AS int)` | `toint(x)` | Family: `toint`, `tostring`, `todatetime` |
| `LIKE '%foo%'` | `contains 'foo'` | Or `has`, `startswith`, `matches regex` |
| `WITH cte AS (...)` | `let cte = ...;` | Statement-level |
| `BETWEEN a AND b` | `between (a .. b)` | Note `..` operator |

---

## Core Operators (90% of queries)

### `take` / `sample`
```kql
T | take 5            // 5 random rows (cheap; no sort)
T | sample 100        // 100 random rows (statistical sample)
```

### `where` (filter rows)
```kql
T | where State == 'TEXAS'
T | where Damage > 10000 and EventType !contains 'Hail'
T | where StartTime between (datetime(2007-04-01) .. datetime(2007-04-30))
T | where isnotempty(Author)
```

### `project` / `project-away` / `project-rename` / `extend`
```kql
T | project StartTime, State, EventType         // SELECT specific cols
T | project-away EpisodeId, EventId             // SELECT all EXCEPT
T | project-rename When = StartTime             // rename column
T | extend Total = InjuriesDirect + DeathsDirect // ADD computed column
```

> **`project` vs `extend`**: `project` picks columns (drops rest); `extend` adds columns (keeps everything).

### `sort by` / `top`
```kql
T | sort by DamageProperty desc
T | sort by EventType asc, StartTime desc       // multiple keys
T | top 10 by DamageProperty desc               // sort + take in one
```

### `summarize` (GROUP BY)
```kql
T | summarize count() by State

T | summarize EventCount = count() by State, EventType

T | summarize 
    Total = count(),
    Injuries = sum(InjuriesDirect),
    AvgDamage = avg(DamageProperty),
    MaxDamage = max(DamageProperty),
    P95Damage = percentile(DamageProperty, 95)
  by State
```

### `distinct`
```kql
T | distinct State                              // unique states
T | distinct State, EventType                   // unique combinations
```

---

## Time-Series Operations

### `bin()` — bucket time
```kql
T | summarize count() by bin(StartTime, 1d)     // group by day
T | summarize count() by bin(StartTime, 1h)     // by hour
T | summarize count() by bin(StartTime, 5m)     // 5-min buckets
T | summarize count() by bin(StartTime, 7d)     // weekly
```

### `make-series` — time series with gap filling
Like `summarize`, but fills empty buckets with a default value. Essential for accurate charts.

```kql
T
| where StartTime > ago(30d)
| make-series Events = count() default = 0 
    on StartTime step 1d
| render timechart
```

### `ago()` and `now()` — time arithmetic
```kql
T | where StartTime > ago(7d)                   // last 7 days
T | where StartTime > ago(1h)                   // last hour
T | where StartTime between (ago(30d) .. now()) // last 30 days
T | extend AgeInDays = (now() - StartTime) / 1d
```

### Time units
| Unit | Examples |
|---|---|
| microseconds | `1tick`, `1microsecond` |
| seconds | `1s`, `30s` |
| minutes | `1m`, `15m` |
| hours | `1h`, `6h` |
| days | `1d`, `7d` |

### Date functions
```kql
datetime(2024-01-15)             // parse string to datetime
datetime(2024-01-15 13:45:00)    // with time
startofday(StartTime)            // truncate to day boundary
startofmonth(StartTime)          // truncate to month
format_datetime(StartTime, 'yyyy-MM-dd')
datepart('hour', StartTime)      // extract hour
dayofweek(StartTime)             // 0=Sun, 6=Sat
```

---

## String Operations

### Matching operators (performance matters!)

| Operator | Use Case | Performance |
|---|---|---|
| `==` | Exact match | Fast |
| `=~` | Case-insensitive exact match | Fast |
| `has` | Whole-word match | **Very fast (indexed)** |
| `has_cs` | Whole-word, case sensitive | **Very fast** |
| `contains` | Substring (case insensitive) | Slow |
| `contains_cs` | Substring, case sensitive | Slow |
| `startswith` | Prefix match | Medium |
| `endswith` | Suffix match | Slow |
| `matches regex` | Regular expression | Slowest |

> ⚡ **Performance tip**: Use `has` instead of `contains` whenever possible. `has` uses indexes; `contains` scans every character. Up to 100x faster on large tables.

### String functions
```kql
strlen(s)                        // length
tolower(s) / toupper(s)          // case conversion
trim(' ', s)                     // remove leading/trailing chars
substring(s, 0, 5)               // 5 chars from index 0
split(s, ',')                    // returns dynamic array
strcat(a, '-', b)                // concatenation
replace_string(s, 'old', 'new')  // string replace
extract('@(\\w+)\\.com', 1, Email)  // regex capture group
```

---

## Aggregation Functions

| Function | Description |
|---|---|
| `count()` | Row count |
| `countif(predicate)` | Count where condition is true |
| `dcount(col)` | Distinct count (approximate, fast) |
| `dcount_hll(col)` | HyperLogLog approximate distinct |
| `sum(col)` | Sum |
| `avg(col)` | Average |
| `min(col)` / `max(col)` | Min/Max |
| `stdev(col)` / `variance(col)` | Std deviation / variance |
| `percentile(col, 95)` | Specific percentile |
| `percentiles(col, 50, 90, 95, 99)` | Multiple percentiles |
| `make_list(col)` | Aggregate values into array |
| `make_set(col)` | Aggregate distinct values into array |
| `arg_max(sortCol, *)` | Row with max value (keeps all cols) |
| `arg_min(sortCol, *)` | Row with min value (keeps all cols) |

### `arg_max` / `arg_min` — secret weapon
Returns the entire row where a column is max/min — saves a self-join.

```kql
// Latest event per state (with all columns of that event)
Weather
| summarize arg_max(StartTime, *) by State

// Costliest event per type
Weather
| summarize arg_max(DamageProperty, *) by EventType
```

---

## Joins

> ⚠️ **GOTCHA**: KQL's default join is `innerunique`, which de-duplicates left side keys. **Always specify `kind=inner` explicitly.**

| kind= | Description | SQL equivalent |
|---|---|---|
| `inner` | Match both sides | INNER JOIN |
| `innerunique` | (DEFAULT!) inner with left-side de-dup | (no equivalent) |
| `leftouter` | All from left + matches | LEFT JOIN |
| `rightouter` | All from right + matches | RIGHT JOIN |
| `fullouter` | All from both sides | FULL OUTER JOIN |
| `leftsemi` | Left rows with match (no right cols) | WHERE EXISTS |
| `leftanti` | Left rows WITHOUT match | WHERE NOT EXISTS |
| `rightsemi` / `rightanti` | Mirrored | |

### Syntax
```kql
let injuries = T1 | summarize Inj = sum(InjuriesDirect) by State;
let damage   = T2 | summarize Dmg = sum(DamageProperty) by State;

injuries
| join kind=inner damage on State
| project State, Inj, Dmg
```

> ⚡ **Performance**: In KQL, **always put the SMALLER dataset on the LEFT** of the join. Opposite of what SQL engines optimize for.

---

## Let Statements (variables / CTEs)

```kql
let StartDate = datetime(2007-04-01);
let EndDate   = datetime(2007-06-30);
let MinDamage = 10000;
let HighRiskStates = dynamic(['TEXAS', 'KANSAS', 'OKLAHOMA']);

Weather
| where StartTime between (StartDate .. EndDate)
| where DamageProperty > MinDamage
| where State in (HighRiskStates)
```

### Let with a sub-query
```kql
let TopStates = Weather
    | summarize count() by State
    | top 10 by count_
    | project State;

Weather
| where State in (TopStates)
| summarize avg(DamageProperty) by State
```

---

## Render — Inline Visualizations

| Keyword | What it does |
|---|---|
| `timechart` | Line chart for time-series (auto-detects time column) |
| `linechart` | Plain line chart |
| `areachart` | Filled area chart |
| `stackedareachart` | Stacked area |
| `columnchart` | Vertical bars |
| `barchart` | Horizontal bars |
| `piechart` | Pie chart |
| `scatterchart` | Scatter plot (x/y) |
| `anomalychart` | Time chart with anomalies highlighted |
| `table` | Tabular (default) |

```kql
T
| summarize count() by bin(StartTime, 1d)
| render timechart with (title='Daily events', ymin=0)

T
| summarize count() by EventType
| top 5 by count_
| render piechart
```

---

## Window Functions

KQL applies these **in the current sort order** — always sort first.

| Function | Returns |
|---|---|
| `prev(col, N)` | Value from N rows back (default N=1) |
| `next(col, N)` | Value from N rows ahead |
| `row_number()` | Sequential row number (1, 2, 3...) |
| `row_cumsum(col)` | Running total |
| `row_rank(col)` | Rank within partition |
| `row_window_session(col, idle, max, restart)` | Sessionization |

### Sessionization example
```kql
// Group user clicks into sessions (30min idle gap = new session)
Clicks
| sort by UserId asc, Timestamp asc
| extend SessionStart = row_window_session(Timestamp, 30m, 1h, UserId != prev(UserId))
| summarize Events = count(), Start = min(Timestamp), End = max(Timestamp) 
    by UserId, SessionStart
```

---

## Power Operators

### `mv-expand` — unpivot arrays
```kql
// If 'Tags' is dynamic array ['red', 'urgent', 'big']
T
| mv-expand Tag = Tags                     // one row per tag
| summarize count() by tostring(Tag)
```

### `parse` — extract from strings
```kql
Logs
| parse Message with 'User ' UserId ' did ' Action ' at ' * 
// Now UserId and Action are columns
```

### `evaluate` — built-in algorithms (underrated!)
```kql
// Anomaly detection
T
| make-series Cnt=count() default=0 on Timestamp step 1h by Service
| evaluate series_decompose_anomalies(1.5)

// Auto-cluster (clustering / segmentation)
T | evaluate autocluster()

// Basket analysis (frequent itemsets)
T | evaluate basket()

// Diff patterns between cohorts
T | evaluate diffpatterns(Errors, '', Status)
```

> 💎 **Hidden gem**: `evaluate` gives you ML-style functions baked into the query language. `series_decompose_anomalies` in particular is incredible for monitoring.

### `union` — combine multiple tables
```kql
union Logs2023, Logs2024, Logs2025
| where Status == 'Error'

// Union with prefix to track source
union withsource=SourceTable Logs2023, Logs2024
| summarize count() by SourceTable
```

### `materialize` — cache for reuse
```kql
let baseData = materialize(
    Weather
    | where StartTime > ago(30d)
    | where DamageProperty > 0
);

baseData | summarize sum(DamageProperty) by State;
baseData | summarize sum(InjuriesDirect) by State;
```

---

## Management Commands (start with `.`)

```kql
// Schema
.show tables                              // list tables
.show table T schema as json              // detailed schema
.create table T (Col1: string, Col2: int) // create table
.drop table T                             // delete table
.create-merge table T (...)               // create if missing, alter if existing

// Ingestion
.ingest into table T ('https://...csv')   // ingest from URL
.ingest inline into table T <| 1,2,3      // ingest inline values
.set-or-append T <| <query>               // ingest from query result

// Monitoring
.show ingestion failures | top 10 by FailedOn desc
.show operations | where State != 'Completed'
.show database details

// Retention / housekeeping
.alter table T policy retention softdelete = 30d
.clear table T data                        // empty without dropping
```

---

## Practical Query Patterns

### Top N per category
```kql
// 3 costliest events per state
Weather
| sort by State, DamageProperty desc
| extend Rank = row_number(1, State != prev(State))
| where Rank <= 3
```

### Funnel analysis
```kql
Events
| where Timestamp > ago(7d)
| summarize
    Visited = countif(Action == 'PageView'),
    AddedToCart = countif(Action == 'AddToCart'),
    Checkout = countif(Action == 'Checkout'),
    Purchased = countif(Action == 'Purchase')
    by UserId
```

### Rolling 7-day average
```kql
T
| make-series Daily=count() default=0 on Timestamp step 1d
| extend Rolling7d = series_fir(Daily, dynamic([1,1,1,1,1,1,1]), false, false)
| render timechart
```

### Detect missing data / gaps
```kql
Heartbeats
| sort by Timestamp asc
| extend GapSeconds = (Timestamp - prev(Timestamp)) / 1s
| where GapSeconds > 60
| project Timestamp, GapSeconds
```

### Hourly anomaly detection
```kql
Metrics
| where Timestamp > ago(7d)
| make-series Count=count() default=0 on Timestamp step 1h
| extend Anomaly = series_decompose_anomalies(Count, 1.5)
| render anomalychart
```

---

## Common Gotchas

| Gotcha | Fix |
|---|---|
| `=` instead of `==` in where | KQL uses `==` for equality, never `=` |
| `join` without `kind=` defaults to `innerunique` | Always specify `kind=inner` explicitly |
| Case-sensitive table/column names | `TableName` ≠ `tablename` — match exactly |
| `contains` is slow | Use `has` (whole-word) or `has_cs` when possible |
| `bin()` vs `make-series` | `bin` = ignore empty buckets; `make-series` = fill with default |
| Mixing summarize and project | `summarize` REPLACES the schema; reference only what you defined |
| String quoting | KQL uses `' '` single quotes (double quotes also work) |
| Time without timezone | `datetime()` is UTC. Convert at display time |
| join: small ← big | Always put SMALLER table on left of join |

---

## Keyboard Shortcuts (Fabric KQL Queryset)

| Shortcut | Action |
|---|---|
| `Shift + Enter` | Run selected query (or current cursor query) |
| `Ctrl + Space` | IntelliSense / autocomplete |
| `Ctrl + /` | Toggle line comment |
| `Ctrl + K, Ctrl + F` | Format selection |
| `F1` | Command palette |
| `Alt + Up / Down` | Move line up/down |

---

## Where KQL is used

Same syntax, multiple products — massively transferable skill:

- **Microsoft Fabric Real-Time Intelligence** (Eventhouse, KQL Database)
- **Azure Data Explorer** (Kusto — the original)
- **Azure Monitor / Log Analytics** (infrastructure logs)
- **Microsoft Sentinel** (SIEM, security)
- **Application Insights** (app telemetry)
- **Microsoft Defender for Endpoint** (security threat hunting)
- **Microsoft Defender for Cloud**
- **Microsoft 365 Defender**
- **Azure Resource Graph** (Azure inventory queries)

Learning KQL once unlocks all of these.

---

*Cheatsheet by Mustafa Abdeali — built during Microsoft Fabric deep-dive, May 2026*
